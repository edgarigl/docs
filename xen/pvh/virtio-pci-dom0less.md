# dom0less Xen PVH on ARM64 with Virtio PCI using QEMU backends

In preparation for adding PCI support to the Xen PVH ARM machine in QEMU,
I've published some instructions on how to run everything.

## Overview

These are some basic instructions on how to run Xen booting 2 guests
dom0lessly on top of QEMU's virt machine.
The guests are a dom0 and a domU using Virtio-PCI.

Virtio PCI is an emulated PCI controller exposed to domU. It's emulated by QEMU
in dom0 providing virtio-pci devices and potentially other emulated PCI devs.

In this example, Xen will be booted via u-boot in EFI mode.

These instructions assume that you are already able to build Xen, Xen tools and
QEMU from source for ARM.

I build my images with Yocto. I've published my personal Yocto setup (Yoxen) on github:
[yoxen](https://github.com/edgarigl/yoxen)

### Boot race

When booting dom0lessly, dom0 and domU will race towards finishing the boot.
At some point domU will try to scan the Virtio PCI bus and there won't be
anything there. Thanks to a background mmio region in Xen, it won't crash but
it won't find any PCI devices.

Once dom0 has booted, ran QEMU and QEMU has registered an IOREQ server we'll
trigger a PCI bus rescan in domU to discover the newly added devices.

## Repos and branches

Since we're prepping for upstream, we're using either upstream or upstream with
a minimal set of patches for this to work and avoiding vendor trees.

### U-boot

I used u-boot from upstream, specifically the following commit but any recent u-boot should work:
```
commit a481740629af5e51732b86b6ead61dc5aa5a440d (HEAD -> master, origin/master, origin/HEAD)
Merge: 0b06e052fb e49eeb6575
Author: Tom Rini <trini@konsulko.com>
Date:   Thu Aug 22 08:14:48 2024 -0600
```

### QEMU

This QEMU is based on upstream master + a few patches to disable buffered
IOREQ's for ARM and enable PCI for ARM PVH.

https://github.com/edgarigl/qemu/tree/edgar/xenpvh-arm-pci

### Xen

This Xen branch is based on upstream master at the time of writing with
a few patches to enable virtio-pci. The most important ones:
```
xen/arm: dom0less: Add a background PCI ECAM mmio region
xen/arm: create dom0less virtio-pci DT node
xen/arm: Add an ioreq-ready mmio region
```

## Linux

I used a vanilla Linux kernel from this commit:
```console
$ git show
commit 8d8d276ba2fb5f9ac4984f5c10ae60858090babc (HEAD -> master, linus/master)
Merge: bc83b4d1f086 4e378158e5c1
Author: Linus Torvalds <torvalds@linux-foundation.org>
Date:   Tue Sep 10 09:05:20 2024 -0700
```

## dom0less DTS setup

Assuming you start with a device-tree file that works for Xen, let's call it xen.dtb.

By applying the following overlay, we'll be describing 2 guests, dom0 and domU.
Here's en example overlay to go on top of your device-tree for Xen.

```code
        chosen {
                #address-cells = <2>;
                #size-cells = <2>;

                stdout-path = "/pl011@9000000";
                kaslr-seed = <0xd68ecccf 0x8bdd7e5a>;
                bootargs = "root=/dev/vdb1 console=ttyAMA0,115200n8 earlyprintk=serial,ttyAMA0";

                xen,xen-bootargs = "dom0_mem=1G bootscrub=0";
                xen,dom0-bootargs = "rw root=/dev/vdb1 earlyprintk=serial,ttyAMA0 console=hvc0 earlycon=xenboot pcie_ports=native maxcpus=2";

                module@0 {
                        compatible = "xen,linux-zimage", "multiboot,module";
                        xen,uefi-binary = "Image";
                };

                domU1 {
                        compatible = "xen,domain";
                        #address-cells = <2>;
                        #size-cells = <2>;
                        // 1G
                        memory = <0x0 1048576>;
                        cpus = <1>;
                        vpl011;
                        virtio-pci = "grants";

                        module@0 {
                                compatible = "multiboot,kernel", "multiboot,module";
                                bootargs = "rw root=/dev/ram earlyprintk=serial,ttyAMA0 console=hvc0 earlycon=xenboot";
                                xen,uefi-binary = "Image";
                        };
                        module@1 {
                                compatible = "multiboot,ramdisk", "multiboot,module";
                                xen,uefi-binary = "xen-image-minimal-qemuarm64.rootfs.cpio.gz";
                        };
                };

        };
```

## Build images

Build a yocto xen-image-minimal including an sdk for it. We'll need the toolchain with the
xen libraries to build QEMU.

You can build it on your own or follow the instructions in 
[yoxen](https://github.com/edgarigl/yoxen)


## Build QEMU for dom0

This assumes you've got a xen-image-minimal SDK to cross-compile for the target.

```console
$ git clone -b edgar/xenpvh-arm-pci https://github.com/edgarigl/qemu
Cloning into 'qemu'...
remote: Enumerating objects: 751684, done.
remote: Counting objects: 100% (124/124), done.
remote: Compressing objects: 100% (58/58), done.
Receiving objects: 100% (751684/751684), 540.46 MiB | 584.00 KiB/s, done.
remote: Total 751684 (delta 79), reused 88 (delta 66), pack-reused 751560 (from 1)
Resolving deltas: 100% (613714/613714), done.
Updating files: 100% (10189/10189), done.
```

Create a build directory
```console
$ mkdir build-qemu
```

Enable the Yocto xen-image-minimal SDK:
```console
$ source /opt/yoxen/arm64/environment-setup-cortexa57-poky-linux
```

Configure QEMU:
```bash
$ cd build-qemu
$ PKG_CONFIG=pkg-config ../qemu/configure \
        --enable-debug-info \
        --prefix=/opt/qemu/master/ \
        --target-list=aarch64-softmmu \
        --disable-tcg \
        --enable-xen \
        --disable-libusb \
        --disable-libudev \
        --disable-bzip2 \
        --disable-snappy \
        --disable-lzo \
        --disable-kvm \
        --disable-gtk \
        --disable-curl \
        --disable-sdl \
        --disable-curses \
        --disable-vhdx \
        --disable-hv-balloon \
        --disable-virtfs \
        --disable-vhost-crypto \
        --disable-vhost-kernel \
        --disable-vhost-net \
        --disable-vhost-user \
        --disable-vhost-user-blk-server --disable-selinux \
        --disable-gnutls \
        --disable-gcrypt \
        --disable-nettle \
        --disable-crypto-afalg \
        --disable-vte \
        --disable-vmdk \
        --disable-alsa \
        --disable-oss \
        --disable-sndio \
        --disable-coreaudio \
        --disable-pa \
        --disable-qed \
        --disable-bochs \
        --disable-cloop \
        --disable-replication \
        --disable-l2tpv3 \
        --disable-vvfat \
        --disable-glusterfs \
        --disable-png \
        --disable-parallels \
        --disable-vpc \
        --disable-qcow1 --disable-multiprocess \
        --disable-dmg --disable-vdi \
        --disable-vhost-vdpa \
        --disable-rdma \
        --disable-dbus-display \
        --disable-qom-cast-debug \
        --disable-attr \
        --disable-keyring \
        --disable-libvduse \
        --disable-vduse-blk-export \
        --extra-cflags="-Os" \
        --disable-linux-aio \
        --disable-linux-io-uring \
        --enable-trace-backends=log \
        --enable-slirp \
        $*
```

Now build:
```console
$ make
```

The target qemu should now be in build-qemu/qemu-system-aarch64.

## Build boot.scr

Create boot.script:
```code
kernel_addr=0x42000000
fdt_addr=0x44000000

setenv loaddtb "load virtio 0:1 ${fdt_addr} xen.dtb"
setenv bootxen "run loaddtb; load virtio 0:1 ${kernel_addr} xen.efi; bootefi ${kernel_addr} ${fdt_addr}"

setenv bootlinux "run loaddtb; load virtio 0:1 ${kernel_addr} Image; booti ${kernel_addr} - ${fdt_addr}"

run bootxen
```

```console
$ mkimage -T script -d boot.script boot.scr
```


## Build disk-images if running on QEMU

Once you have the images, we'll create some qcow2 images for QEMU. If you're running on
real HW, these steps will differ.

## Create boot partition

Assuming you've got your boot images in the current directory (xen.efi, Image, xen.dtb):

```bash
# Start fresh
rm -f boot-arm.qcow2

# Create a QEMU qcow2 image
qemu-img create -f qcow2 boot-arm.qcow2 512M

# Use nbd to expose the qcow2 image on your dev host
sudo qemu-nbd --connect=/dev/nbd0 boot-arm.qcow2

# Parition the boot disk
/bin/echo -e 'n\np\n\n\n\nw\n' | sudo fdisk /dev/nbd0

# Create a filesystem. Choose whatever fs you need, I used ext4.
sudo mkfs.ext4 /dev/nbd0p1

# Copy files into the boot partition
mkdir boot-arm
sudo mount /dev/nbd0p1 boot-arm

sudo cp Image boot-arm/
sudo cp xen.efi boot-arm/
sudo cp xen.dtb boot-arm/
sudo cp boot.scr boot-arm/

# Cleanup
sudo umount boot-arm
sudo qemu-nbd -d /dev/nbd0
rm -fr boot-arm 
```

Now you should have a boot-arm.qcow2 populated with all the boot images.

## Create rootfs disk image for dom0

This assumes you've built a xen-image-minimal for dom0 and a core-image-minimal for domU.
We also expect qemu-system-aarch64 for dom0 to be in the current directory.

```bash
# Start fresh
rm -f rootfs.qcow2

# Create a QEMU qcow2 image
qemu-img create -f qcow2 rootfs.qcow2 4G

# Use nbd to expose the qcow2 image on your dev host
sudo qemu-nbd --connect=/dev/nbd0 rootfs.qcow2

# Parition the boot disk
/bin/echo -e 'n\np\n\n\n\nw\n' | sudo fdisk /dev/nbd0

# Create a filesystem. Choose whatever fs you need, I used ext4.
sudo mkfs.ext4 /dev/nbd0p1

sudo cp xen-image-minimal-qemuarm64.rootfs.ext4 /dev/nbd0p1
sudo mkdir -p rootfs-mount
sudo mount /dev/nbd0p1 rootfs-mount

# Modify fstab and add our boot partition so the guest can self-updgrade.
echo '/dev/vda1 /boot auto defaults 1 1' | sudo tee -a rootfs-mount/etc/fstab
sudo cp core-image-minimal-qemuarm64.rootfs.cpio.gz rootfs-mount/home/root/
sudo cp Image rootfs-mount/home/root/
sudo cp qemu-system-aarch64 rootfs-mount/usr/bin/

sudo umount rootfs-mount
sudo rmdir rootfs-mount
sudo qemu-nbd -d /dev/nbd0
```

Now you have a rootfs for dom0.

# Run host QEMU virt

Run the outer QEMU with user networking and map port 2222 to port 22 allowing
the host to ssh into dom0.

```bash
$ qemu-system-aarch64 \
    -machine virt,gic_version=3,iommu=smmuv3,virtualization=true \
    -cpu cortex-a57 -m 5G -smp 2 \
    -bios u-boot/u-boot.bin -no-reboot \
    -device virtio-net-pci,netdev=net0,romfile= \
    -netdev type=user,id=net0,hostfwd=tcp::2222-:22 \
    -serial mon:stdio \
    -drive format=qcow2,file=boot-arm.qcow2,if=none,id=drv0 \
    -device virtio-blk-pci,drive=drv0 \
    -drive format=qcow2,file=rootfs.qcow2,if=none,id=drv1 \
    -device virtio-blk-pci,drive=drv1 \
    -display none

U-Boot 2024.10-rc3-00007-ga481740629af (Aug 30 2024 - 16:06:29 +0200)

DRAM:  5 GiB
Core:  51 devices, 14 uclasses, devicetree: board
Flash: 64 MiB
Loading Environment from Flash... *** Warning - bad CRC, using default environment

...

** Booting bootflow 'virtio-blk#33.bootdev.part_1' with script
8728 bytes read in 2 ms (4.2 MiB/s)
1245968 bytes read in 3 ms (396.1 MiB/s)
Booting /xen.efi
Using modules provided by bootloader in FDT
Xen 4.20-unstable (c/s Mon Sep 9 20:47:01 2024 +0200 git:aea400e10b) EFI loader
Image: 0x000000017a94b000-0x000000017d3c1200
xen-image-minimal-qemuarm64.rootfs.cpio.gz: 0x00000001745c3000-0x000000017a949cfe
- UART enabled -
- Boot CPU booting -
- Current EL 0000000000000008 -
- Initialize CPU -
- Turning on paging -
- Paging turned on -
- Ready -
(XEN) Checking for initrd in /chosen
(XEN) Checking for "xen,static-mem" in domain node
(XEN) Region: [0x0000017a94b000, 0x0000017d3c1200) overlapping with mod[2]: [0x0000017a94b000, 0x0000017d3c1200)
(XEN) RAM: 0000000040000000 - 00000001745c2fff
(XEN) RAM: 00000001745c3000 - 000000017a949fff
(XEN) RAM: 000000017a94a000 - 000000017a94afff
(XEN) RAM: 000000017a94b000 - 000000017d3c1fff
(XEN) RAM: 000000017d3c2000 - 000000017d3c2fff
(XEN) RAM: 000000017d3c3000 - 000000017d3c7fff
(XEN) RAM: 000000017d3c8000 - 000000017d543fff
(XEN) RAM: 000000017d544000 - 000000017d546fff
(XEN) RAM: 000000017d54d000 - 000000017d554fff
(XEN) RAM: 000000017d556000 - 000000017d556fff
(XEN) RAM: 000000017d579000 - 000000017d583fff
(XEN) RAM: 000000017d584000 - 000000017e6a1fff
(XEN) RAM: 000000017e6a3000 - 000000017f6bffff
(XEN) RAM: 000000017f6d0000 - 000000017fffffff
(XEN) 
(XEN) MODULE[0]: 000000017d3c8000 - 000000017d544000 Xen         
(XEN) MODULE[1]: 000000017d3c4000 - 000000017d3c8000 Device Tree 
(XEN) MODULE[2]: 000000017a94b000 - 000000017d3c1200 Kernel      
(XEN) MODULE[3]: 00000001745c3000 - 000000017a949cfe Ramdisk     
(XEN) 
(XEN) CMDLINE[000000017a94b000]:domU1 rw root=/dev/ram console=hvc0 earlycon=xenboot
(XEN) 
(XEN) Command line: dom0_mem=1G bootscrub=0
(XEN) Domain heap initialised
(XEN) Booting using Device Tree
(XEN) Platform: Generic System
(XEN) Taking dtuart configuration from /chosen/stdout-path
(XEN) Looking for dtuart at "/pl011@9000000", options ""
 __  __            _  _    ____   ___                     _        _     _
 \ \/ /___ _ __   | || |  |___ \ / _ \    _   _ _ __  ___| |_ __ _| |__ | | ___
  \  // _ \ '_ \  | || |_   __) | | | |__| | | | '_ \/ __| __/ _` | '_ \| |/ _ \
  /  \  __/ | | | |__   _| / __/| |_| |__| |_| | | | \__ \ || (_| | |_) | |  __/
 /_/\_\___|_| |_|    |_|(_)_____|\___/    \__,_|_| |_|___/\__\__,_|_.__/|_|\___|

(XEN) Xen version 4.20-unstable (edgar@) (gcc (Ubuntu 13.2.0-23ubuntu4) 13.2.0) debug=y Tue Sep 10 20:58:19 CEST 2024
```

You shold see both guests boot into login prompts.
Press ctrl+a 6 times in a row and you should see the terminal input switching to domU:
```console
(XEN) *** Serial input to DOM1 (type 'CTRL-a' three times to switch input)
```

Login to domU and issue lspci, there won't be any devices.

Now, In another terminal, SSH into dom0:
```console
$ ssh -p 2222 root@localhost
```

Run the virtio backend QEMU. This QEMU will register an IOREQ server for domU
and provide emulation of the virtio PCI controller.

```bash
$ qemu-system-aarch64 \
    -M xenpvh,pci-ecam-base=0x11000000000,pci-ecam-size=0x10000000,pci-mmio-base=0x11010000000,pci-mmio-size=0x2000000,pci-mmio-high-base=0x10000000000,pci-mmio-high-size=0x1000000000 \
    -xen-domid 1 \
    -xen-attach \
    -no-shutdown \
    -display none \
    -smp 1,maxcpus=1 \
    -m 1024M \
    -netdev user,id=n0 \
    -device virtio-net,netdev=n0,romfile=""
```

Now, in domU, trig a PCI rescan:
```console
$ echo 1 >/sys/bus/pci/rescan 
(d1) [  516.604517] pci 0001:00:00.0: [1b36:0008] type 00 class 0x060000 conventional PCI endpoint
(d1) [  516.703719] pci 0001:00:01.0: [1af4:1000] type 00 class 0x020000 conventional PCI endpoint
(d1) [  516.738427] pci 0001:00:01.0: BAR 0 [io  0x0000-0x001f]
(d1) [  516.768665] pci 0001:00:01.0: BAR 4 [mem 0x00000000-0x00003fff 64bit pref]
(d1) [  516.861555] pci 0001:00:01.0: BAR 4 [mem 0x10012000000-0x10012003fff 64bit pref]: assigned
(d1) [  516.881550] pci 0001:00:01.0: BAR 0 [io  size 0x0020]: can't assign; no space
(d1) [  516.883016] pci 0001:00:01.0: BAR 0 [io  size 0x0020]: failed to assign
(d1) [  516.887370] OF: /pcie@10000000000: no iommu-map translation for id 0x8 on (null)
(d1) [  516.900937] virtio-pci 0001:00:01.0: enabling device (0000 -> 0002)
```

lspci should now show the virtio-net device.
```console
$ lspci -v

0001:00:00.0 Host bridge: Red Hat, Inc. QEMU PCIe Host bridge
(XEN)   Subsystem: Red Hat, Inc. Device 1100
(XEN)   Flags: fast devsel
(XEN) lspci: Unable to load libkmod resources: error -2
(XEN) 
(XEN) 0001:00:01.0 Ethernet controller: Red Hat, Inc. Virtio network device
(XEN)   Subsystem: Red Hat, Inc. Device 0001
(XEN)   Flags: bus master, fast devsel, latency 0, IRQ 13
(XEN)   Memory at 10012000000 (64-bit, prefetchable) [size=16K]
(XEN)   Capabilities: [84] Vendor Specific Information: VirtIO: <unknown>
(XEN)   Capabilities: [70] Vendor Specific Information: VirtIO: Notify
(XEN)   Capabilities: [60] Vendor Specific Information: VirtIO: DeviceCfg
(XEN)   Capabilities: [50] Vendor Specific Information: VirtIO: ISR
(XEN)   Capabilities: [40] Vendor Specific Information: VirtIO: CommonCfg
(XEN)   Kernel driver in use: virtio-pci
```

Bring up the network and ping dom0:
```console
$ ip link set up dev eth0
$ ip a a 10.0.2.16/24 dev eth0
$ ping -i 0.1 10.0.2.2
(XEN) PING 10.0.2.2 (10.0.2.2): 56 data bytes
(XEN) 64 bytes from 10.0.2.2: seq=0 ttl=255 time=45.928 ms
(XEN) 64 bytes from 10.0.2.2: seq=1 ttl=255 time=17.952 ms

```

