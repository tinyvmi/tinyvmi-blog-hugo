+++
title =  "TinyVMI: Porting LibVMI to Mini-OS on Xen Project Hypervisor"
date = 2018-08-13T03:21:22-04:00
tags = [ "progress" ]
featured_image = ''
description = ""

draft = "false"

+++

This post introduces the project I worked on with Honeynet Project at Google Summer of Code this year. The project of [TinyVMI](https://github.com/tinyvmi/tinyvmi) is to port a library ([LibVMI](http://libvmi.com)) into a tiny operating system ([Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS)). After porting, LibVMI will have all its functionalities running inside a tiny virtual machine, which has a much smaller size as well as higher performance compared to the same library running on a Linux OS.

# Introduction

### Mini-OS & Unikernels

[Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS) is a tiny operating system demo distributed with the source of _Xen Project Hypervisor_ (abbreviated as _Xen_ below). It has been a basis for development of serveral unikernels, such as ClickOS and Rump kernels. 
Unikernels can be viewed as a minimized operating system with following features:

- No ring0/ring3, or kernel/user mode separation. Traditional operating systems, like Linux, separate programs into kernel mode and user mode to protect malicious users (applications) from accessing kernel memory. However, in unikernels like Mini-OS, there is only one mode, ring0, or kernel mode. This eliminates the burden of maintaining the context-switching between two modes. The code size of the kernel and runtime overhead are both reduced. 

- A minimal set of libraries. Instead of shipping with many system/application libraries to provide a general purpose computing environment, a unikernel aims to be configured with a minimal set of libraries that are only necessary for the application that runs in it, thus also called a library operating system. For example, in Mini-OS, users can configure with libc to write applications in C language. 

<figure>
<img src="/gsoc-blog/images/unikernel_minios.png" style="width:350px;"/>
<figcaption >
<h4 class="center">Fig.1 General Purpose OS vs. Mini-OS Unikernel</h4>
</figcaption> 
</figure>


As shown in Fig.1, a unikernel is much smaller in size and eliminates all unnecessary tools and libraries, and even file systems from the OS, keeping only the application code and a tiny OS kernel. Such unikernels can be more efficient than traditional operating systems, especially for cloud platforms where each specialized application is usually managed in a standalone VM. The unikernels are supposed to be the next generation of cloud platforms because they can achieve efficiency from several aspects. These include but not limited to:

1. Less memory footprint. A unikernel requires significantly less memory than a traditional operating system. For example, a Mini-OS VM with LibVMI application only requires 64MB of main memory. However, a Linux VM would occupy 4GB of main memory to get average performance for a 64-bit Linux. The reduced memory footprints would allow a single physical machine to host more VMs and reduce the average cost per service. 
2. Faster booting. Since the memory footprint is small and no redundant libraries or kernel modules, a tiny OS would require significant less time to boot than a traditional OS. Booting a tinyOS is just like to start the application itself. 
3. No kernel mode switching. OS kernels and applications are in a same chunk of memory region. CPU context switches caused by system calls are eliminated in unikernels. Therefore, the runtime performance of the unikernels can be much better than a traditional OS.
4. More secure. Each unikernel's VM runs only one application. Isolation between  applications is enforced by the hypervisor, instead of a shared OS kernel. Compared to process isolation or container isolation in a shared Linux, the unikernel is more secure from the lower level isolation.
5. Easy deployment, easy to use. Unikernel applications are build into a single binary to run directly as a VM image, which simplifies the deployment of the service. Unikernel applications are designed to be _single click and run_. All functionalities are customized at building time. Once deployed, the binary package requires no human modifications except the whole binary package being replaced. 

In brief, Mini-OS is a tiny OS originated from the Xen Project hypervisor. Like other unikernels, Mini-OS provides higher performance and a more secure computing environment than a traditional operating system on the cloud.

### Why port LibVMI to MiniOS

LibVMI is a secure critical library that could be used to view a target VM's raw memory from another guest VM, thus gaining a whole view of almost all the activities on the target VM. 

Traditionally, LibVMI runs in Dom0 on the hypervisor. However, Dom0 is already very big even without LibVMI in it. I got the idea of separating LibVMI from Dom0 from the the following observations:

1. Dom0 is a general purpose OS hosting many daily use applications, such as administractor tools. However, LibVMI is a special purpose library and usually not for daily use. Furthermore, there are almost no direct communication between LibVMI and other applications. Thus it is not necessary to install LibVMI in Dom0.

2. Security risk. Dom0 is a critical domain for the hypervisor platform. Introducing a new code base to the kernel would also introduce new security risks. Other applications on Dom0 could leverage kernel vulnerability to compromise LibVMI, and vice versa, a bug in LibVMI could crash other applications or even the entire Dom0 kernel.

3. Performance overhead. As introduced above, a general purpose OS is large and inefficient to run a special purpose application. CPU mode switching, large memory footprints, and process scheduling all introduce overheads for Dom0.
  

Therefore, we propose to port LibVMI to the tiny Mini-OS, named as [TinyVMI](https://github.com/tinyvmi/tinyvmi.git), to explore whether we can achieve the above benefits. 

# Challenges

First, the hypervisor isolates each guest VM from reading other VM's memory pages. A guest VM should get enough permission before it can be used to introspect other VM's memory.  Second, LibVMI depends on serveral libraries that are not supported in the original Mini-OS. Therefore, in this project, we want solutions to overcome these two challenges.

### Permissions of Accessing Other VM's Memory

To introspect a VM's memory from another guest VM, the first thing is to get permissions from the hypervisor. By default, memory pages of each VM are strictly isolated with each other -- they are not allowed to access the memory pages of other VMs. Although the hypervisor allow programmers to share memory pages between two VMs by grant tables, it requires the target VM to explicitly `offer` the page for sharing. Since entire target VM is not trusted and no changes should be made to the target VM. LibVMI uses foreign memory mapping hypercalls to remap memory pages from target VM to its own memory space. The permission of mapping a foreign page (target VM's page) to its own address space for a guest VM (or Dom0) are controlled by Xen Security Module (XSM), which will be introduced below. 

Furthermore, Xen event channels allow guest VM to monitor its memory status in real time under help of hardware interrupts. A ring buffer is shared between hypervisor and the guest kernel to transfer event information. To access the ring buffer, XSM permission should also be granted. 

[Xen Security Module](https://wiki.xenproject.org/wiki/Xen_Security_Modules_:_XSM-FLASK) (XSM) uses FLASK policies as in SELinux, to enforce Mandatory Access Control(MAC) between different domains. Each permission is default to be denied unless explicitly being allowed in the policy. Permissions are granted according to multiple categries the guest domain belongs to, such as the types, roles, users, and attributes of the guest domain ([more](https://wiki.xenproject.org/wiki/Xen_Security_Modules_:_XSM-FLASK#Types.2C_roles.2C_users_and_attributes)). 


The category of a VM is labeled in the configuration file we use to create it via ``xl create <config_file>``. For example: 

```
seclabel='system_u:system_r:domU_t1'
```

will label the VM as type **domU_t1**, under role of **system_r**, and user of **system_u**, user **system_u**.  Type is the lowest level of category. Multiple types can be defined as one role multiple roles as one user.

Permissions are granted based on the types of a VM. For example, the permission of ``map_read`` allow a domain to map other domain's memory with read only permission. The policy:

```
allow domU_t1 domU_t2:mmu {map_read};
```

will allow a VM with type **domU_t1** to read the memory of another VM with type **domU_t2**.


In addition to the permissions granted from XSM, we also need to the permission the read information from [Xenstore](https://wiki.xen.org/wiki/XenStore), which is used to get meta data of the target VM, such as getting the Domain ID from the domain's string name. Xenstore permission can be read via command
``xenstore-ls -p``:

```
# xenstore-ls -p
local = "" . . . . . . . . . . . .   (n0)
 domain = "" . . . . . . . . . . .   (n0)
  0 = "" . . . . . . . . . . . . .   (n0)
   domid = "0" . . . . . . . . . .   (n0)
   name = "Domain-0" . . . . . . .   (n0)
   device-model = "" . . . . . . .   (n0)
    0 = "" . . . . . . . . . . . .   (n0)
     backends = "" . . . . . . . .   (n0)
      console = "" . . . . . . . .   (n0,n0)
      vkbd = ""  . . . . . . . . .   (n0,n0)
      qdisk = "" . . . . . . . . .   (n0,n0)
      9pfs = ""  . . . . . . . . .   (n0,n0)
      qusb = ""  . . . . . . . . .   (n0,n0)
      vfb = "" . . . . . . . . . .   (n0,n0)
      qnic = ""  . . . . . . . . .   (n0,n0)
     state = "running" . . . . . .   (n0)

```
The meaning of permission could be found from the [manual](https://xenbits.xen.org/docs/4.6-testing/man/xenstore-ls.1.html). Command [xenstore-chmod](https://xenbits.xen.org/docs/4.6-testing/man/xenstore-chmod.1.html) can be used to grant reading permissions to certain VM. For example, to enable VM with ID ``8`` to read Xenstore directory ``/local/domain``, you can run : 

```bash
xenstore-chmod -r '/local/domain' r8
```

### Build New Libraries into Mini-OS

The next challenge is building new libraries into Mini-OS. 
Mini-OS is a examplar minimal operating system designed to running on Xen. To keep the kernel small, there are only few libraries can be built in it: **newlib** for C language library, Xen related library such as **libxc** to communicate with the hypervisor, and **lwip** for basic networking. 

To port LibVMI to Mini-OS, 2 more libraries are needed. These include one JSON library to parse Rekall profiles, [libjson-c](https://github.com/json-c/json-c), and one library with utility data structures, [GLib](https://wiki.gnome.org/Projects/GLib). 

In theory, most libraries written in C language can be built into Mini-OS with the help of **newlib**, such as **libjson-c**. [This post](https://tinyvmi.github.io/gsoc-blog/post/w08-cross-compile-lib-in-minios) introduces how to build new libraries. However, some of them might need to be manually customized for MiniOS by eliminating the unsupported portions, such as [GLib](https://developer.gnome.org/glib/).

Furthermore, security applications written in C++ programs can also be ported into Mini-OS. For example, [DRAKVUF](https://drakvuf.com/) is a binary analysis system built ontop of LibVMI and Xen. Portion of its code are in C++ language. To built those code in Mini-OS, we need to cross compile C++ standard libraries into the tiny kernel.

# Project Status & Results

### Functions added to Mini-OS

- Support of LibVMI functions to introspect Linux and Windows guest on x86 architecture. Both **memory access** and **event support** are implemented. ARM architecture and other OS kernels (such as FreeBSD) have not been explored yet.
- A customized [GLib](https://github.com/tinyvmi/tinyvmi/tree/master/tiny-vmi/tiny_glib), a statically compiled [libjson-c](https://github.com/json-c/json-c) is cross compiled into Mini-OS.
- C++ language support. C++ standard library from GCC was cross compiled into static libraries, such as libgcc, libstdc++, etc. Now in Mini-OS, we can program with C++ ! Not only C. Detailed steps can be found in [this post](https://tinyvmi.github.io/gsoc-blog/post/w08-cross-compile-lib-in-minios/).
- A github site of [Documentations](https://tinyvmi.github.io) and a [Blog](https://tinyvmi.github.io/gsoc-blog) are maintained to document the manuals of how to build and run TinyVMI, as well as track the progress of each proceeded step during the summer. 

### Performance Analysis

In order to evaluate the TinyVMI system, we conduct a simple analysis and experiment to show its efficiency.
We build two VM domains with LibVMI on the same hypervisor for comparison. One guest VM running Mini-OS with LibVMI and another VM, Dom0, running Linux (Ubuntu 16.04) with LibVMI. The target VM being introspected is a 64-bit Linux (Ubuntu 16.04).
Results are shown in Fig.2 and Fig.3.

<figure>
<img src="/gsoc-blog/images/code_size_noxl.png" style="width:350px;" />
<figcaption >
<h4 class="center">Fig.2 Code Size of LibVMI and Different Kernels</h4>
</figcaption> 
</figure>
<figure>
<img src="/gsoc-blog/images/perf.png" style="width:350px;"/>
<figcaption >
<h4 class="center">Fig.3 Time in Walking Through Page Table</h4>
</figcaption> 
</figure>

Fig.2 shows the overall code size of the OS with LibVMI in it. LibVMI with MiniOS totaled 83K Lines of Code (LoC) while LibVMI with Linux kernel had 177K LoC, reducing more then 50% percent of code size. Note that the LoC of Linux kernel does not include any driver codes, which only reflects the possible minimal size of a Linux kernel. If drivers included, it could be 15M+ LoC for Linux system. 

Fig.3 shows the time elapsed of reading one page by walking thought the 4 levels of the page table while introspecting a 64-bit Linux guest VM. The time is an average of reading 500 consecutive pages. LibVMI in Mini-OS took 3.7 microseconds, while LibVMI in Linux took 5.7 microseconds, saving more than 30% of time.


# Conclusion
To briefly conclude the project, we have successfully ported the core functionalities of LibVMI into the tiny OS on Xen, Mini-OS. By customizing the XSM policy specifications and Xenstore permissions, a guest VM has been granted with permissions to introspect other guest VM via VMI technique. By customizing and cross compiling static libraries into Mini-OS, we have built LibVMI in a tiny OS, enabling a tiny VM to introspect both Linux and Windows guest VM. Evaluations show the code size is reduced by more than 50% and performance is improved by more than 30% compared to VMI operations on Dom0 of hypervisor.

# Future Directions

- DRAKVUF integration. After the last week of GSoC, C++ language support was added to TinyVMI under the help of [this post from Notes to self](http://lnotestoself.blogspot.com/2010/06/dmini-os-x8632-cmini-os.html). Next step would be cross compiling the DRAKVUF system into TinyVMI. This will enable much more interesting applications that can take full advantage of LibVMI interfaces already provided in the Mini-OS.
  
- Dom0 Introspection. We all know Dom0 is huge. Although much work has been done to disaggregate it, it is still huge. TinyVMI itself has small trusted computing base (TCB). However, we still need to trust Dom0 to enforce the XSM policies. This enlarges the TCB of the system significantly. Since we have to trust Dom0, it will be useless to monitor the main memory of Dom0 from TinyVMI. A further step to disaggregate Dom0 would be separate the XSM module management interface into another stubdom, or just to the same domain as TinyVMI. Taking this apart would make it possible to eliminate Dom0 from the trusted computing base, and allow TinyVMI to monitor Dom0 via VMI techniques. 

# Acknowledgement

Thanks to my mentors, Steven Maresca and Tamas K Lengyel, for accepting me as a student in GSoC this year. This is my first time at GSoC and this exciting project cannot be proceeded so far without your prompt, helpful instructions and graceful patience. Thanks to Zibby Keaton for the grammar checkings on this post. Thanks to all Google Summer of Code committees to provide such a great opportunity for us to explore the world of open source!
