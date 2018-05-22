+++
title =  "First Week: A primary test with TinyVMI and build up website"
date = 2018-05-21T09:20:19-04:00
tags = ["progress", "framework"]
featured_image = ""
description = ""
+++

## The problem being addressed:

The previous TinyVMI project[1] was a minimal portion of LibVMI with capability of reading a target VM's memory pages when a kernel virtual address was given. Now we need to extend TinyVMI with events support, and other capabilities of LibVMI, such as support for both 32 & 64 bit systems, both Windows & Linux OS, etc.. Additionally, during the development, documentations need to be carefully written and progresses will be reported in a public blog site.



## What has been done:

**Cache Data Structure Implementation.**  Cache in LibVMI highly depends on Glib functions. However, glib is dynamically linked library and Xen MiniOS cannot be linked dynamically. Thus no glib support is available in TinyVMI. In order to enable cache in TinyVMI, the implementation of GLib data structure and functions are manually ported from GLib or implemented from scratch. Serveral glib data structures are implemented to keep compatible with the original LibVMI as much as possible. Such data structures include: GHashTable, and GSList. During implementation of the glib related functions, only the necessary part of the source code are ported from GLib or manually crafted. More data structures are needed to enable higher performance, such as GQueue, etc. 

**Input:** to keep problem simple, the input file ``/etc/libvmi.conf`` and kernel symbol file in LibVMI are both manually compiled into strings. 


**Partial tested with function map_addr and event_example.** Now TinyVMI is able to support Xen Events, with Xen version of 4.10.0, and target VM running Ubuntu 16.04 64-bit, with kernel version 4.4.0.

**Blog via Hugo:** Two websites were built with individual repo: one for [documentation of TinyVMI](https://tinyvmi.github.io), one for [GSoC blog progress posts](https://tinyvmi.github.io/gsoc-blog). 

## What to do next:

**Cache implementation.** GHashTable and GQueue is not ported from GLib directly. This might affect the performance of TinyVMI.

**Comparison between TinyVMI with LibVMI:** Now we have two primary examples of ``map_addr`` and ``event_example`` that can be used to evaluate the performance of TinyVMI and LibVMI. Primary results shown that TinyVMI is better than LibVMI at ``map_addr`` example, but worse than LibVMI at ``event_example``. However, theoretically, TinyVMI should not be worse than LibVMI. More test should be done to figure out the problem. 

**Debug MiniOS with Xenstore:** Now TinyVMI cannot access xenstore information, i.e. cannot read Domain information from xenstore and convert a domain string name to domain id. The error seems to be a denied access. Need more efforts to figure out the problem. 

**Portability:**  Can we port LibVMI to Xen MiniOS with no changes to the source code of LibVMI? or with minimal changes ? So that we can keep up to date with the new changes of LibVMI.

**More TODOs:** Windows support; network support (with switch option); support more Xen versions instead of only xen-4.10.0 ...



###### [1] [Lele Ma, Xiaomeng Yue, Yuqing Wang, and Qiusong Yang. (2015). Virtual machine introspecting and memory security monitoring based on the light-weight operating system. 计算机应用, 35(6), 1555-1559.](http://www.joca.cn/EN/abstract/abstract18030.shtml)