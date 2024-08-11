# x86 Xen PVH guests with QEMU

This is some basic documentation on how to run x86 Xen PVH machines
using the work in progress patches that AMD is in the progress of
trying to upstream.

These instructions assume you already have Xen running on an x86
systems and that you already know howto compile your own Xen,
Xen-tools and QEMU for dom0 and that you are able to run HVM guests.

## Xen

You'll need to build Xen and the Xen tools from the following branch:
https://github.com/edgarigl/xen/tree/edgar/virtio-pvh-upstream

The branch contains PVH support for Xen (by Xenia Ragiadakou) with a few
minor changes to align with the submitted QEMU patches.

You can build Xen and the Xen-tools using your usual flows. The way I
personally do it is with Yocto.
I've published my personal Yocto setup (Yoxen) to github:
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

For reference, I've provided my [kernel-config](kernelconfig-xenpvh-x86.config)

## Launch a DomU PVH

I've provided an example xl config file:
[guest-xenpvh-x86.cfg](guest-xenpvh-x86.cfg)

```console
$ xl create -cd guest-xenpvh-x86.cfg
...
Starting syslogd/klogd: done
Starting domain watchdog daemon: xenwatchdogd startup

INIT: Id "S1" respawning too fast: disabled for 5 minutes
INIT: Id "S0" respawning too fast: disabled for 5 minutes

Yocto on Xen development distro 2024.02.10 qemux86-64 /dev/hvc0

qemux86-64 login: root
root@qemux86-64:~# lspci -v
00:00.0 Host bridge: Red Hat, Inc. QEMU PCIe Host bridge
        Subsystem: Red Hat, Inc. Device 1100
        Flags: fast devsel
lspci: Unable to load libkmod resources: error -2

00:01.0 Ethernet controller: Red Hat, Inc. Virtio network device
        Subsystem: Red Hat, Inc. Device 0001
        Flags: bus master, fast devsel, latency 0, IRQ 17
        Memory at f0000000 (32-bit, non-prefetchable) [size=4K]
        Memory at c010000000 (64-bit, prefetchable) [size=16K]
        Capabilities: [98] MSI-X: Enable+ Count=4 Masked-
        Capabilities: [84] Vendor Specific Information: VirtIO: <unknown>
        Capabilities: [70] Vendor Specific Information: VirtIO: Notify
        Capabilities: [60] Vendor Specific Information: VirtIO: DeviceCfg
        Capabilities: [50] Vendor Specific Information: VirtIO: ISR
        Capabilities: [40] Vendor Specific Information: VirtIO: CommonCfg
        Kernel driver in use: virtio-pci

00:02.0 Keyboard controller: Red Hat, Inc. Virtio 1.0 input (rev 01)
        Subsystem: Red Hat, Inc. Device 1100
        Flags: bus master, fast devsel, latency 0, IRQ 18
        Memory at f0001000 (32-bit, non-prefetchable) [size=4K]
        Memory at c010004000 (64-bit, prefetchable) [size=16K]
        Capabilities: [98] MSI-X: Enable+ Count=2 Masked-
        Capabilities: [84] Vendor Specific Information: VirtIO: <unknown>
        Capabilities: [70] Vendor Specific Information: VirtIO: Notify
        Capabilities: [60] Vendor Specific Information: VirtIO: DeviceCfg
        Capabilities: [50] Vendor Specific Information: VirtIO: ISR
        Capabilities: [40] Vendor Specific Information: VirtIO: CommonCfg
        Kernel driver in use: virtio-pci
```
