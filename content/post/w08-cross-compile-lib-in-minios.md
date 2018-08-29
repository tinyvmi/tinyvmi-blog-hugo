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
{{<highlight make>}} 
# link json lib
# enable this need to set CONFIG_JSON=y in minios.cfg

ifeq ($(CONFIG_JSON),y)
APP_LDLIBS += -ljansson
APP_LDLIBS += -ljson-c
endif

# link gcc lib
# enable this need to set CONFIG_JSON=y in minios.cfg

ifeq ($(CONFIG_GCC),y)
#############################################
# LIBs for g++ gcc cross compiled in stubdom/Makefile.
# libgcc
# todo: the version number should be a var
APP_LDLIBS += -L$(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/lib/gcc/$(MINIOS_TARGET_ARCH)-xen-elf/5.4.0 -whole-archive -lgcc -lgcov -no-whole-archive
LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/lib/gcc/$(MINIOS_TARGET_ARCH)-xen-elf/5.4.0/libgcc.a
LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/lib/gcc/$(MINIOS_TARGET_ARCH)-xen-elf/5.4.0/libgcov.a

# libstdc++ libs
APP_LDLIBS += -L$(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib -whole-archive -lstdc++ -lssp_nonshared -no-whole-archive
LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libstdc++.a
# LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libssp.a
LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libssp_nonshared.a
# LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libsupc++.a
# LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libquadmath.a

APP_LDLIBS += -L$(XEN_ROOT)/stubdom/gcc-newlib-patch -whole-archive -lgcc_newlib_patch -no-whole-archive
LIBS += $(XEN_ROOT)/stubdom/gcc-newlib-patch/libgcc_newlib_patch.a
endif # end CONFIG_GCC,y
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
If errors occur, you might need to tweak the configurations and building options of the library by reading the documentation shipped with the library source code.


## Example 2: cross compile GCC C++ standard library to Mini-OS. 

#### Example app code

Changed file: xen-src/stubdom/tinycpp/main.cpp

{{<highlight C++>}}    

#include <iostream>

int main(void){

    sleep(2);
    std::cout << "hello in cpp !!! \n" << std::endl;

    return 0;
}





{{</hightlight>}}

#### Makefile of stubdom/
Changed file: xen-src/stubdom/Makefile. Changes have three parts: a) cross compile binutils; b) a patch to newlib in Mini-OS. c) cross compile C++ libraries. 
    {{<highlight make>}}    

    ##############
    # Cross-binutils
    ##############

    BINUTILS_VERSION=2.31
    BINUTILS_URL="http://ftp.gnu.org/gnu/binutils"
    binutils-$(BINUTILS_VERSION).tar.gz:
        $(FETCHER) $@ $(BINUTILS_URL)/$@

    binutils-$(BINUTILS_VERSION): binutils-$(BINUTILS_VERSION).tar.gz
        tar xzf $<
        touch $@

    BINUTILS_STAMPFILE=$(CROSS_ROOT)/$(GNU_TARGET_ARCH)-xen-elf/bin/ar
    .PHONY: cross-binutils
    cross-binutils: $(BINUTILS_STAMPFILE)
    $(BINUTILS_STAMPFILE): mk-headers-$(XEN_TARGET_ARCH) binutils-$(BINUTILS_VERSION) $(NEWLIB_STAMPFILE)
        mkdir -p binutils-$(XEN_TARGET_ARCH)
        ( cd binutils-$(XEN_TARGET_ARCH) && \
        CC_FOR_TARGET="$(CC) $(TARGET_CPPFLAGS) $(TARGET_CFLAGS) $(NEWLIB_CFLAGS)" \
        AR_FOR_TARGET=$(AR) \
        LD_FOR_TARGET=$(LD) \
        RANLIB_FOR_TARGET=$(RANLIB) \
        ../binutils-$(BINUTILS_VERSION)/configure \
            --prefix=$(CROSS_PREFIX) \
            --verbose \
            --target=$(GNU_TARGET_ARCH)-xen-elf \
            --enable-interwork \
            --enable-multilib \
            --disable-nls --disable-shared --disable-threads \
            --with-sysroot=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf \
            --with-gcc --with-gun-as --with-gnu-ld && \
        $(MAKE) DESTDIR= && \
        $(MAKE) DESTDIR= install )

    ###########
    # gcc_newlib.patch
    NEWLIB_PATCH_FOR_GCC=$(CURDIR)/gcc_newlib.patch/libgcc_newlib_patch.a

    $(CURDIR)/gcc_newlib.patch/libgcc_newlib_patch.o: $(CURDIR)/gcc_newlib.patch/patch.c
        $(CC) $(TARGET_CPPFLAGS) $(TARGET_CFLAGS) $(NEWLIB_CFLAGS) -c -o $@ $<

    $(NEWLIB_PATCH_FOR_GCC): $(CURDIR)/gcc_newlib.patch/libgcc_newlib_patch.o
        $(AR) cr $@ $^

    ##############
    # Cross-gcc
    ##############

    #gmp: https://gmplib.org/download/gmp/gmp-6.1.2.tar.xz
    #mpfr: https://ftp.gnu.org/gnu/mpfr/mpfr-4.0.1.tar.xz
    #mpc: https://ftp.gnu.org/gnu/mpc/mpc-1.1.0.tar.gz

    #gcc: ftp://ftp.gnu.org/gnu/gcc/gcc-5.4.0/gcc-5.4.0.tar.gz

    GCC_VERSION=5.4.0
    GCC_URL="http://ftp.gnu.org/gnu/gcc/gcc-$(GCC_VERSION)"
    gcc-$(GCC_VERSION).tar.gz:
        $(FETCHER) $@ $(GCC_URL)/$@

    gcc-$(GCC_VERSION): gcc-$(GCC_VERSION).tar.gz
        tar xzf $<
        cd $@ && ./contrib/download_prerequisites && cd -
        touch $@

    # TODO: this stampfile is not valid for make, 
    # even without any changes to gcc, gcc will be 'recompiled' instead of keep the old build.
    # why?
    GCC_STAMPFILE=$(CROSS_ROOT)/lib/gcc/$(GNU_TARGET_ARCH)-xen-elf/$(GCC_VERSION)/libgcc.a
    .PHONY: cross-gcc
    cross-gcc: $(GCC_STAMPFILE)

    $(GCC_STAMPFILE): mk-headers-$(XEN_TARGET_ARCH) gcc-$(GCC_VERSION) $(NEWLIB_STAMPFILE) $(BINUTILS_STAMPFILE)
        mkdir -p gcc-$(XEN_TARGET_ARCH)
        ( cd gcc-$(XEN_TARGET_ARCH) && \
        PATH=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/bin:$(CROSS_PREFIX)/bin:$(PATH) \
        CC_FOR_TARGET="$(CC) $(TARGET_CPPFLAGS) $(TARGET_CFLAGS) $(NEWLIB_CFLAGS)" \
        AR=$(AR) \
        AR_FOR_TARGET=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/bin/ar \
        LD_FOR_TARGET=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/bin/ld  \
        RANLIB_FOR_TARGET=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/bin/ranlib \
        ../gcc-$(GCC_VERSION)/configure \
            --prefix=$(CROSS_PREFIX) \
            --verbose \
            --target=$(GNU_TARGET_ARCH)-xen-elf \
            --enable-multilib \
            --enable-languages=c,c++ \
            --with-newlib \
            --disable-nls \
            --disable-shared --disable-threads \
            --with-gnu-as \
            --with-as=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/bin/as \
            --with-gnu-ld \
            --with-ld=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf/bin/ld \
            --with-sysroot=$(CROSS_PREFIX)/$(GNU_TARGET_ARCH)-xen-elf && \
        $(MAKE) DESTDIR= all-gcc && \
        $(MAKE) DESTDIR= all-target-libgcc && \
        $(MAKE) DESTDIR= install-gcc && \
        $(MAKE) DESTDIR= install-target-libgcc && \
        $(MAKE) DESTDIR= && \
        $(MAKE) DESTDIR= install )

    {{</highlight>}}

#### Makefile of Mini-OS
Changed file: xen-src/extras/mini-os/Makefile. Changes are mainly to add static library path to the linker:

    {{<highlight make>}} 
    # link gcc lib
    # enable this need to set CONFIG_JSON=y in minios.cfg

    ifeq ($(CONFIG_GCC),y)
    #############################################
    # LIBs for g++ gcc cross compiled in stubdom/Makefile.
    # libgcc
    # todo: the version number should be a var
    APP_LDLIBS += -L$(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/lib/gcc/$(MINIOS_TARGET_ARCH)-xen-elf/5.4.0 -whole-archive -lgcc -lgcov -no-whole-archive
    LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/lib/gcc/$(MINIOS_TARGET_ARCH)-xen-elf/5.4.0/libgcc.a
    LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/lib/gcc/$(MINIOS_TARGET_ARCH)-xen-elf/5.4.0/libgcov.a

    # libstdc++ libs
    APP_LDLIBS += -L$(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib -whole-archive -lstdc++ -lssp_nonshared -no-whole-archive
    LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libstdc++.a
    # LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libssp.a
    LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libssp_nonshared.a
    # LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libsupc++.a
    # LIBS += $(XEN_ROOT)/stubdom/cross-root-$(MINIOS_TARGET_ARCH)/$(MINIOS_TARGET_ARCH)-xen-elf/lib/libquadmath.a

    APP_LDLIBS += -L$(XEN_ROOT)/stubdom/gcc-newlib-patch -whole-archive -lgcc_newlib_patch -no-whole-archive
    LIBS += $(XEN_ROOT)/stubdom/gcc-newlib-patch/libgcc_newlib_patch.a
    endif # end CONFIG_GCC,y
    {{</highlight>}}

You also need to add a line of ``CONFIG_GCC=y`` to the minios config file under your app source code. such as ``xen-src/stubdom/tinycpp/minios.cfg``.