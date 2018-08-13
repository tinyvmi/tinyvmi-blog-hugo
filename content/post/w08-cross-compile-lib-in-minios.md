+++
title =  "How to Cross Compile a Static Library in Xen Mini-OS"
date = 2018-07-09T18:05:55-04:00
tags = [ "debug", "dependences" ]
featured_image = ""
description = ""
+++


Xen [Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS) is a minimal operating system designed to running on top of Xen hypervisor. To keep the kernel small, there are only few libraries shipped with it: **newlib** for C language library, Xen related library such as **libxc** to communicate with Xen hypervisor, and **lwip** for basic networking. 
To port LibVMI to Mini-OS, more libraries are needed. These include JSON libraries to parse Rekall profiles, and library with some utility data structures such as [GLib](https://wiki.gnome.org/Projects/GLib). 

The tiny kernel of Mini-OS has limited support for most high level libraries. Some of the libraries could be directly compiled into MiniOS without any changes, such as libjasson, while others might need to be manually customized for MiniOS by eliminating the unsupported portions or only keeping the necessary parts of the library, such as [the reduced GLib for LibVMI](https://github.com/libvmi/glib_lite).

In this post, we will introduce how to directly compile a new library to Mini-OS in Xen: which part of Xen source code needs to be changed and why. Keep in mind that not all libraries could be directly compiled to Mini-OS, and even if it supports the compilation, much efforts still need to be done to make it work as expected. If errors occur by following the instructions below, you will need more efforts to debug the project. This will need some hand on experience of GNU make and C language.

Basically, you will need three parts of changes to make the library successfully compiled into Mini-OS:

1. Application that calls library functions. Needless to say, this is a must-have to make sure the library APIs could indeed be used in a Mini-OS app. For example, in order to cross compile libjson-c in Mini-OS, you could add the following source code in Mini-OS applicaiton with library header **json-c/json.h** and one of the API **json_tokener_parse(char *str)**, as following:

    {{<highlight c>}}
    // tinyvmi/tiny-vmi/rekall.c
    #include <json-c/json.h>

    status_t rekall_profile_symbol_to_rva(... )
    {
        ...
       
        json_object *root = json_tokener_parse(linux_rekall_string);

        ...
    }

    {{</highlight>}}

2. Makefile in stubdom ([xen-src/stubdom/Makefile](https://github.com/tinyvmi/xen/blob/xen-4.10.0-tinyvmi/stubdom/Makefile)). In this Makefile, the library will be downloaded from Internet, configured, statically compiled, and finally installed to a cross compile root directory. There are already some examples (such as **[newlib](https://github.com/tinyvmi/xen/blob/44ce23c0d811c08bb559c46a171b234c3ff714a2/stubdom/Makefile#L78)**) in the original Makefile, so we can just craft a new commands similarly for a new library. Among those commands, we will need to define a cross compile target (**cross-json-c**), along with its dependent targets. For example, for libjson-c, the following lines are added ([code](https://github.com/tinyvmi/xen/blob/1811c836d9e7d999fbdcf0c470502b8f1fe3f388/stubdom/Makefile#L104)):
   {{< highlight make >}}
    # file: xen-src/stubdom/Makefile

    ##############
    # Cross-json-c
    ##############
    
    JSON_C_URL="https://s3.amazonaws.com/json-c_releases/releases"
    JSON_C_VERSION=0.13

    TARGET_CFLAGS += -Wno-error

    json-c-$(JSON_C_VERSION).tar.gz:
        $(FETCHER) $@ $(JSON_C_URL)/$@

    json-c-$(XEN_TARGET_ARCH): json-c-$(JSON_C_VERSION).tar.gz
        tar xzf $<
        mv json-c-$(JSON_C_VERSION) $@

    JSON_C_STAMPFILE=$(CROSS_ROOT)/$(GNU_TARGET_ARCH)-xen-elf/lib/libjson-c.a
    .PHONY: cross-json-c
    cross-json-c: $(JSON_C_STAMPFILE)
    $(JSON_C_STAMPFILE): json-c-$(XEN_TARGET_ARCH) $(NEWLIB_STAMPFILE)
        ( cd json-c-$(XEN_TARGET_ARCH) && \
        sh autogen.sh && \
        CFLAGS="$(TARGET_CPPFLAGS) $(TARGET_CFLAGS)" CC=$(CC) \
        ./configure --disable-shared --disable-threading \
            --prefix=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
            --exec-prefix=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
            --host=$(GNU_TARGET_ARCH)-xen-elf && \
        $(MAKE) DESTDIR= && \
        $(MAKE) DESTDIR= install )

        ...

        .PHONY: tinyvmi
        tinyvmi: cross-json-c cross-newlib $(CROSS_ROOT) tinyvmi-minios-config.mk

    {{< / highlight >}} In this example, we first define the URL and version number of the library. Then the target **json-c-$(JSON_C_VERSION).tar.gz** will download the source code. **json-c-$(XEN_TARGET_ARCH)** will extract the source and rename the folder. Then, the target **$(JSON_C_STAMPFILE)** or **cross-json-c** will go to the source code directory and configure, compile, and install the library in a cross-compile manner. Those cross compile setups are usually given via configure options. This should be kept consistent with the documentation of the specific library we are compiling. Running **./configure --help** would show the available options.  In general, we will need to give the **CFLAGS**, **--prefix**, and **--host** to specify necessary build environment variables for the target machine (here the machine is the Mini-OS VM recognized as $(GNU_TARGET_ARCH)-xen-elf).
    Finally, we need to update the dependent list of target **tinyvmi**, so that every time we rebuild **tinyvmi**, the library will be built when necessary. 

3. Makefile of Mini-OS ([xen-src/extras/mini-os/Makefile](https://github.com/tinyvmi/mini-os/blob/xen-4.10.0/Makefile)). In this Makefile, the library load flags **APP_LDLIBS** (for example, **-ljson-c**) will be given to finally link the static library into the Mini-OS application. To link libjson-c, the following line will do the trick([code](https://github.com/tinyvmi/mini-os/blob/c5cf2aa01a5cc7bc66a6d519bbc5933daa857411/Makefile#L147)):
{{<highlight make>}} APP_LDLIBS += -ljson-c
{{</highlight>}}

Now we are ready to compile the library. You can either run make with the cross compile target:

{{<highlight bash>}}
cd stubdom/
make cross-json-c 
{{</highlight>}}

or compile TinyVMI by:

{{<highlight bash>}}
cd stubdom/
make tinyvmi-stubdom
{{</highlight>}}

If succeed, you will find the static library file (such as libjson-c.a) under the directory **$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf** (will be  /xen-src/stubdom/cross-root-x86_64/x86_64-xen-elf/lib/libjson-c.a for libjson-c)
If errors occur, you might need to tweak the configurations and building options of the library by reading the documentation shipped with the library source code. That's all. Thank you!