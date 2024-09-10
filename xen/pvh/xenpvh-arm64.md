# arm64 Xen PVH guests with QEMU

Basic documentation on how to build and run Xen PVH machines for ARM with
QEMU virtio backends.


## Xen

To run arm64 Xen PVH with the pending upstream patches you'll need to build
Xen and the Xen tools from the following branch:


The branch contains PVH support for Xen (by Xenia Ragiadakou) with a few
minor changes to align with the submitted QEMU patches.

You can build Xen and the Xen-tools using your usual flows. The way I do it
is with Yocto. I've published my personal Yocto setup (Yoxen) on github:
[yoxen](https://github.com/edgarigl/yoxen)

## QEMU

You'll need a QEMU build from the following branch:
https://github.com/edgarigl/qemu/tree/edgar/xenpvh-qom

Make sure to build and install the qemu-system-i386 binary into the rootfs
of dom0.
You can follow any flow to build QEMU, the way I do it is by building a Xen
enabled toolchain (Yocto SDK) with Yoxen and using that to cross-compile QEMU
for the target.

## Linux

For my test runs, I use a vanilla upstream Linux kernel built from the
following commit:

```
commit 09e5c48fea173b72f1c763776136eeb379b1bc47 (HEAD -> master, linus/master)
Merge: 10d48d70e82d 321e3c3de53c
Author: Linus Torvalds <torvalds@linux-foundation.org>
Date:   Fri Mar 8 18:05:21 2024 -0800
```

## Launch a DomU PVH

I've provided an example xl config file:
[guest-arm-pvh.cfg](guest-arm-pvh.cfg)

```console
$ xl create -cd guest-arm64-pvh.cfg
```
