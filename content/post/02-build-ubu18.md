+++
title =  "Build Xen From Source Code with Ubuntu 18.04: need to update `qemu-xen`"
date = 2018-05-26T22:46:06-04:00
tags = ["debug"]
featured_image = ""
description = ""
+++

**Problem:** When compiling Xen from source with Ubuntu 18.04, yields `error: static declaration of ‘memfd_create’ follows non-static declaration`:

```
$ cd xen-4.10.0
$ ./configure --enable-systemd --enable-stubdom
$ make -C xen menuconfig 
$ make dist-xen
$ make dist-tools
...... (looks good)
  CC      util/memfd.o
/xen-4.10.0/tools/qemu-xen/util/memfd.c:40:12: error: static declaration of ‘memfd_create’ follows non-static declaration
 static int memfd_create(const char *name, unsigned int flags)
            ^~~~~~~~~~~~
In file included from /usr/include/x86_64-linux-gnu/bits/mman-linux.h:115:0,
                 from /usr/include/x86_64-linux-gnu/bits/mman.h:45,
                 from /usr/include/x86_64-linux-gnu/sys/mman.h:41,
                 from /xen-4.10.0/tools/qemu-xen/include/sysemu/os-posix.h:29,
                 from /xen-4.10.0/tools/qemu-xen/include/qemu/osdep.h:104,
                 from /xen-4.10.0/tools/qemu-xen/util/memfd.c:28:
/usr/include/x86_64-linux-gnu/bits/mman-shared.h:46:5: note: previous declaration of ‘memfd_create’ was here
 int memfd_create (const char *__name, unsigned int __flags) __THROW;
     ^~~~~~~~~~~~
/xen-4.10.0/tools/qemu-xen/rules.mak:66: recipe for target 'util/memfd.o' failed
make: *** [util/memfd.o] Error 1
make: Leaving directory '/xen-4.10.0/tools/qemu-xen-build'
Makefile:220: recipe for target 'subdir-all-qemu-xen-dir' failed
make[3]: *** [subdir-all-qemu-xen-dir] Error 2
make[3]: Leaving directory '/xen-4.10.0/tools'
/xen-4.10.0/tools/../tools/Rules.mk:241: recipe for target 'subdirs-install' failed
make[2]: *** [subdirs-install] Error 2
make[2]: Leaving directory '/xen-4.10.0/tools'
Makefile:68: recipe for target 'install' failed
make[1]: *** [install] Error 2
make[1]: Leaving directory '/xen-4.10.0/tools'
Makefile:127: recipe for target 'install-tools' failed
make: *** [install-tools] Error 2
```

This is due to a bug [here](https://patchwork.ozlabs.org/patch/896059/). Since ubuntu 18.04 shipped with a newer glibc (version ). And `recent glibc added memfd_create in sys/mman.h, which conflicts with the definition in util/memfd.c`. Fortunately, this is fixed at lastest source tree of [Xen: qemu-xen](https://xenbits.xen.org/gitweb/?p=qemu-xen.git;a=commit;h=5c3fdee026a204a59cb392e43a313ab558de9682). Therefore, we need to update the source of `Xen` with the patched `qemu-xen`. 

**Solution:**  

```
cd xen-4.10.0/tools/
rm -r qemu-xen/
git clone https://xenbits.xen.org/git-http/qemu-xen.git
git checkout master # commit: 43139135a8938de44f66333831d3a8655d07663a
make dist-tools
    # success
make dist
sudo make install
    # success
```
