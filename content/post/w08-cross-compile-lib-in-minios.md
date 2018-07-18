+++
title =  "How to Cross Compile a Static Library in Xen Mini-OS"
date = 2018-07-09T18:05:55-04:00
tags = [ "debug", "dependences" ]
featured_image = ""
description = ""
+++


Xen [Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS) is a minimal operating system designed to running on top of Xen hypervisor. To keep the kernel small, there are only few libraries shipped with it: **newlib** for C language library, Xen related library such as **libxc** to communicate with Xen hypervisor, and **lwip** for basic networking. To port LibVMI to Mini-OS, more libraries are needed. These include JSON libraries to parse Rekall profiles, and library with some utility data structures such as [GLib](https://wiki.gnome.org/Projects/GLib).

In this post, we will introduce how to cross compile a new library to Mini-OS in Xen: which part of Xen source code needs to be changed and why.



