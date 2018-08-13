+++
title =  "Porting LibVMI to Mini-OS on Xen Hypervisor"
date = 2018-08-13T03:21:22-04:00
tags = [ "progress" ]
featured_image = ''
description = ""

+++

This post introduces the project I worked on with Honeynet Project at Google Summer of Code this year. The project is to port a library ([LibVMI](http://libvmi.com)) into a tiny operating system ([Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS)). After porting, LibVMI will have all its functionalities running inside a tiny virtual machine, which has much smaller size as well as higher performance compared to the same library running on a Linux OS.

### Introduction

##### What is Mini-OS 

[Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS) is a tiny operating system demo distributed with the Xen hypervisor sources. It has been a basis for development of serveral Unikernels, such as including ClickOS and Rump kernels. 
Mini-OS can be viewed as a minimal operating system by the following features of it:
- No ring0/ring3, or kernel/user mode separation. Traditional operating systems, like Linux, separate programs into kernel mode and user mode to protect malicious users (applications) from accessing kernel memory. However, in Mini-OS, there is only one mode, ring0, or kernel mode. This eliminates the burden of maintaining the context-switching between two modes. The code size of the kernel and runtime overhead are both reduced. 
- Minimal set of libraries. C libraries

##### What is LibVMI

##### Why porting LibVMI to MiniOS

- Single binary. The entire software stack of system libraries, language runtime, and applications is compiled into a single bootable VM image that runs directly on the hypervisor.
  
- As a single purpose OS. Specialised, sealed, single-purpose libOS VMs that run
directly on the hypervisor. A libOS is structured very differently from
a conventional OS: all services, from the scheduler to the device
drivers to the network stack, are implemented as libraries linked
directly with the application

  
All benefits of Dom0 disaggregation 

- Small TCB
- Small memory footprints
- Higher performance

### Challenges

##### VMI in MiniOS

- xsm permissions
- xenstore permissions
- libraries
- C++ applications or C only?

### Progress & Results

##### Functions added to Mini-OS

    - LibVMI.
    - customized GLib.
    - libjson-c and libjansson.
    - C++ language support. C++ language support was compiled into static libraries, such as libgcc, libstdc++, etc. Now in Mini-OS, we can program with C++ ! not only C. 

##### Security Analysis

<figure>
<img src="images/code-size.png" />
<figcaption>
<h4>Steve Francia</h4>
</figcaption>

![This is an image2](/images/esmeralda.jpg)
![This is an image](/images/code-size.png)

{{%figure src="/images/code-size.png" caption="Code Size of LibVMI and Different Kernels" %}}

- Thoughts about Xen architecture. Dom 0 is too big, possible drawbacks: 1, general purposed os, not necessary, might misused; 2, security risk; 3, performance overhead. Possible solution: a). constraint functions of Dom0 by XSM policies; b) divide Dom0 into two parts: server side with core functionalities, and client side with easy-to-use interfaces for management. For sever side, we can use a minimized unikernel to minimize its size and keep the overall functionalities be as specific as possible. 

##### Performance Analysis



##### Lessons Learned

- Hand crafted code vs. modular porting. 
- Hardcoding vs. configuration scripts. 
- Project management with Makefiles. 


### Summary of GSoC 2018

### Future Directions

- DRAKVUF integration
- Dom0 Introspection
    - xsm module in stubdom?
    - minimal dom0 ?



### Acknowledgement

Tamas

GSoC

### Reference
