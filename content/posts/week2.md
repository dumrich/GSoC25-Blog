+++
title = "Week 2: Project Setup Details and APIC"
date = "2025-06-12T14:48:12-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["QEMU", "vmm"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
+++

## QEMU configuration script
```C
$ cd qemu/build
$ ../configure \
  --target-list=x86_64-softmmu \
  --enable-debug \
  --extra-cflags="-I$HOME/freebsd-src/lib/libvmmapi" \
  --extra-ldflags="-L$HOME/vmmtest_root/usr/lib" \
  --disable-werror
$ gmake -j4
```

This ensures that qemu will fetch the custom `vmmapi` located in $HOME/vmmtest_root/usr/lib.

## `libvmmapi`

```C
$ cd $HOME/freebsd-src/lib/libvmmapi/
$ make DEBUG_FLAGS="-g -O1 -fPIC" WITHOUT_MAN=yes all
$ doas cp libvmmapi.a $HOME/vmmtest_root/usr/lib
```

## `vmm`
```C
# kldunload vmm
# cd freebsd-src/sys/modules/vmm
# kldload ./vmm.ko
```
