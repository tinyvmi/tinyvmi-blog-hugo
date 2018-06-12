+++
title =  "Milestone 01: Main Framework of LibVMI in MiniOS"
date = 2018-06-11T12:49:53-04:00
tags = [ "framework", "progress" ]
featured_image = ""
description = "First Milestone Achieved by Porting Main Framework of LibVMI into MiniOS"
+++


### 1. Milestone Goal: "Port the skeleton of LibVMI and system library dependencies"
The goal of the first milestone is described as following in the proposal to GSoC 2018:

> Since the previous work of TinyVMI already proved the feasibility of porting LibVMI into MiniOS, we can take further steps to port as many modules as possible into MiniOS. We should a) try to keep the original LibVMI file/folder structure unless we have to change it to adopt the MiniOS features. b) deleting all the dynamic library dependencies and converting them into static library functions. These include but not limited to some libxenstore, libxencontrol functions. c) implement all the functions that have dependence on the libraries that are unavailable in MiniOS. These include but not limited to some glib function/structures like: GHashTable, GSList, etc..
> 
> During this stage, we only test the main framework of memory mapping functionality and postpone others to future steps. We will hardcoding some parameters and disabling some modules of LibVMI, so that we can debug and port them one by one in the future. For example, we will disable the function of input module (which reads libvmi.conf) and hardcode our simplified information into the framework for testing, such as domain ID (hardcode instead of converting from domain name), virtual addresses (hardcode instead of reading it from system map file). We will also disable the os support (such as linux_init, windoes_init functions), events support, and different architecture supports. 


### 2. What Has Been Done

- Documentation of Getting Started with TinyVMI: https://tinyvmi.github.io/get-started/

- Xenstore Permission setup in Dom0 on Xen. Currently, TinyVMI is able to read from Xenstore directories to convert the target domain name into domain ID. This requires the permission of reading ``/local/domain`` in Xenstore to be granted to TinyVMI. For simplicity, the privilege is currently set in Dom0 via ``xenstore-chmod`` as shown [here](https://tinyvmi.github.io/get-started/run-tinyvmi/).  

- Hardcoded Input of Target OS Information. The directory ``./tiny-vmi/config/target_conf/`` is added to store hardcoded strings in corresponding to information in ``~/etc/libvmi.conf`` in LibVMI.
    * Strings are encoded using Linux command ``xdd -i <input_file>``
    * Examples are in directory ``./tiny-vmi/config/target_conf_examples``. 
    * Files in ``./tiny-vmi/config/target_conf/*`` are symlinked to the example directory for easier exchange between different target OSes.

- Library dependencies on GLib functionalities are manually implemented or ported into MiniOS. 
    * Data structures include GHashTable, GList, GSList, GQueue. 
    * For each data structure, a minimal subset of the functions are ported: only the functions used by LibVMI are ported to MiniOS. 
    * Implementations can be found at ``./tiny-vmi/tiny_glib/``.

- Skeleton of LibVMI is ported into TinyVMI by manually porting the source code file by file, or function by function. Most code structures of LibVMI are kept and necessary changes are made to make it compatible in MiniOS. In the [latest commit](https://github.com/tinyvmi/tinyvmi/commit/a317bdf72e006cd31612088c676e838d9dcb963e), major changes related to LibVMI source code include:

    * Subdirectory '``./libvmi/``' in LibVMI is changed to '``./tiny-vmi/``' in TinyVMI.
    * In directory ``./tiny-vmi/config/`` only C source files (``.h`` and ``.c``) are used in TinyVMI. Others (``lexicon.l``, ``grammar.y``, ``Makefile.am``, etc.) are ignored.
    * Configure file ``./include/tiny-config.h`` is based on the file in LibVMI ``./config.h``.
    * Xenstore and Xencontrl Interface changed to static library in ``./tiny-vmi/driver/xen/libxs_wrapper.c`` and ``./tiny-vmi/driver/xen/libxc_wrapper.c``. Currently only support Xen versions >= 4.8.
    * A minimized subset of GLib for TinyVMI is defined in ``./tiny-vmi/tiny_glib/``.
    * Test cases in LibVMI is ported to ``./tests/`` directory, where all ``main`` functions are renamed according to their functionalities.
    * Rekall files are not ported yet. 

### 3. Lessons Learned

Manually porting the source code in the granularity of files/functions requires a lot human efforts and will be not compatible with future release of LibVMI. Therefore, this strategy should be avoided in the future development. As an alternative, TinyVMI can include the whole LibVMI source code as a module with a minimal patch. To keep pace with the future updates of LibVMI, we can only update the minimal patch.

### 4. Future Work

#### Long term goals: 
- Full support of all LibVMI functions.
- TinyVMI as a stubdom on Xen with secure booting and remote control interface. 

#### Next steps: 
- Rekall file support: json file parsing in TinyVMI.
- If time allowed, find out an efficient way to keep pace with the latest updates of LibVMI.