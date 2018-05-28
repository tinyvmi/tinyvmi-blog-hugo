+++
title =  "GQueue Ported to TinyVMI"
date = 2018-05-28T13:21:22-04:00
tags = ["progress", "dependences"]
featured_image = ""
description = "add support for LRU cache list in LibVMI"
+++


**Port GQueue to TinyVMI. Ongoing** LibVMI use multiple caches to temporarily store the fetched information (or reconstructed information) from the target virtual machine. This week a cache called `memory_cache_lru` is re-implemented in order to keep consistent with the original LibVMI code. 

`memory_cache_lru` is conceptually similar to TLB in an operating system, which stores the virtual address to phisical address mapping in an order of `latest recent unused`(LRU). LibVMI uses [GQueue](https://github.com/GNOME/glib/blob/master/glib/gqueue.h) in [GLib](https://github.com/GNOME/glib) to manage the memory cache LRU list, and TinyVMI previously used a hand-crafted double linked list to store the LRU list. However, it might be not so efficient as the GQueue implementations. Therefore, in order to find out which one is more efficient, and keep TinyVMI with optimal performance, a GQueue version of LRU list are ported. 

Main work is to port the implementations of `GQueue` from [GLib](https://github.com/GNOME/glib/blob/master/glib/gqueue.c). To keep it minimal, only the functions that LibVMI uses are ported. The code includes [queue.h](https://github.com/tinyvmi/tinyvmi/blob/master/tiny-vmi/tiny_glib/queue.h) and [queue.c](https://github.com/tinyvmi/tinyvmi/blob/master/tiny-vmi/tiny_glib/queue.c). Most of them are directly copied from GLib source code, while all the unused parts are commented out. Commented codes are left in the source file in case these functions could be useful in the future developments.

The code has possed compilation but encountered crash at last during `vmi_destroy`. More debugs effort is needed:

**TODO Debug:**
    tiny_slice_new0: allocate memory region and initialize values to 0; it seems not effective.

    