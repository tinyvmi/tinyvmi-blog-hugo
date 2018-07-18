+++
title =  "Milestone 02: Enabling Rekall profile, OS support, Xen events support in TinyVMI"
date = 2018-07-16T03:56:10-04:00
tags = [ "progress" ]
featured_image = ""
description = ""
+++


### 1. Milestone Goal: "Port input module and os support, event support, architecture support, and all examples of LibVMi into MiniOS"

The goal of the second milestone is described in section [3.1.2](https://docs.google.com/document/d/1Y0qOjpXog0DjJoBygMCIGwZXvNaYgaADfuQB1mEJ9mI/edit#heading=h.f5mua3ns94g4) ~ [3.1.6](https://docs.google.com/document/d/1Y0qOjpXog0DjJoBygMCIGwZXvNaYgaADfuQB1mEJ9mI/edit#heading=h.szlnxj4xdmjy) in the [proposal to GSoC 2018](https://docs.google.com/document/d/1Y0qOjpXog0DjJoBygMCIGwZXvNaYgaADfuQB1mEJ9mI/edit?usp=sharing). In brief, it includes **a)** reading configurations of target VM (libvmi.conf); **b)** parsing json files containing target VM; **c)** support introspecting both Linux and Windows virtual machines; **d)** architecture support for both x86 and arm; **e)** testing all examples of LibVMI in TinyVMI.

### 2. What Has Been Done

- Configuration file of target VM has been hardcoded as C strings in TinyVMI. Examples could be found [here](https://github.com/tinyvmi/tinyvmi/tree/master/tiny-vmi/config/target_conf_examples). This will not be flexible when it comes to multiple target VMs. Even if the VM name is changed, it needs the configuration and then the TinyVMI re-compiled. Luckily, this drawback could be partially mitigated by using Rekall profiles. 

- JSON support. Rekall profiles are json files. In order to parse json file in TinyVMI, two more libraries are cross compiled into its kernel: **libjson-c** for json profile of Linux, and **libjansson** for json profile of Windows. Unlike the hand coded 'glib' previously, we now cross compile third party libraries directly to Xen Mini-OS in order to simplify the design of the kernel and allow easier maintainance. Here is a post about [how to cross compile static library to MiniOS in Xen]({{< relref "post/w08-cross-compile-lib-in-minios.md" >}}).

- In addition to Linux, Windows support is enabled in TinyVMI. Successfully tested to introspect Windows 7 (64-bit) with LibVMI examples such as process-list and interrupt-event-example, showing feasibility to correctly intropsect the main memory and hardware events of the target Windows VM.

- To simplifiy the building process, [a repo of Xen source](https://github.com/tinyvmi/xen) is forked and updated with integration of TinyVMI. The building and running of TinyVMI has been simplified to several commands. A page of [Quick Start](https://tinyvmi.github.io/quick-start) is updated in the documentation.

- As for testing, having both Windows and Linux as target VM, TinyVMI could support memory mapping and event callbacks with no problem. [Examples of LibVMI](https://github.com/libvmi/libvmi/tree/master/examples), such as **process-list**, **module-list**, and **interrupt-event-example** are tested. However, we still need more testing work to make TinyVMI as rock solid as possible.

### 3. Lessons Learned

- Cross compiling a library to Mini-OS can be easily achieved by [updating the Makefiles in MiniOS]({{< relref "post/w08-cross-compile-lib-in-minios.md">}}).
- Windows VM images. Images could be downloaded from Microsoft: [Free Virtual Machines from IE8 to MS Edge - Microsoft Edge Development](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/). Images could be converted from VirtualBox to Xen.
- In LibVMI, two libraries are used for different OS, but for same purpose: parsing json strings. Maybe we could use just one library to keep it simple and consistent.


### 4. Future Work

#### Long term goals: 

- Network interface I/O of TinyVMI. Through network, TinyVMI would get the domain ID of target VM for introspection. As output, TinyVMI will send out the VMI information through network.

- Enrich TinyVMI with more applications by integrating [DRAKVUF](https://drakvuf.com/).

- TinyVMI as a stubdom on Xen with secure booting and remote control interface. 

- Architecture support to ARM should be addressed.

#### Next steps: 

- Establish network connections to TinyVMI over TCP. TinyVMI should be waiting for TCP connections before initialize LibVMI functionalities.
 
- If time allowed, find out an efficient way to keep pace with the latest updates of LibVMI.