# Non-blocking virtio-mmio

Non-blocking virtio-mmio is a non-standard extension to virtio-mmio
allowing guests to issue indirect register accesses to the virtio-mmio
register space in a non-blocking manner.

To try it out, you'll need to use the following branches of QEMU, Linux
and Xen:

QEMU:
https://github.com/edgarigl/qemu/tree/edgar/virtio-msg

Linux:
https://github.com/edgarigl/linux/tree/edgar/virtio-indirect

Xen:
https://github.com/edgarigl/xen/tree/edgar/vmp

## Linux

Build Linux from the mentioned branch and use that for the domU guest.
You'll need to enable:
CONFIG_VIRTIO_MMIO=y


## Xen

In Xen, you'll need to enable the following:
CONFIG_VIRTIO_MMIO_NON_BLOCKING=y
CONFIG_VIRTIO_MSG_BUS_XEN=y

## Running on ARM64 dom0less

Add the following fdt property to your dom0less domU description to
instantiate 3 virtio-mmio-nonblocking nodes:
```
       virtio-mmio-non-blocking = <
                                   0x0 0x2000000 0x0 0x39004000 0 33 0x0
                                   0x0 0x2001000 0x0 0x39005000 0 34 0x0
                                   0x0 0x2002000 0x0 0x39006000 0 35 0x1
                                   >;
```

During boot, domU will get stuck at boot spinning waiting for virtio-mmio
non-blocking to respond. Once dom0 is done booting, run for the following
(adjust memory, maxcpus etc to fit your setup):
```console
${QEMU} -M xenpvh \
        -xen-domid 0 \
        -xen-attach -name 0 \
        -no-shutdown \
        -display none \
        -device virtio-msg-bus-xen,shm-base=0x39004000,bus=virtio-msg-bus.0 \
        -device virtio-msg-bus-xen,shm-base=0x39005000,bus=virtio-msg-bus.1 \
        -device virtio-msg-bus-xen,shm-base=0x39006000,bus=virtio-msg-bus.2 \
        -device virtio-net-device,netdev=net0,iommu_platform=on,bus=virtio-msg-proxy-bus.0 \
        -device virtio-net-device,netdev=net1,iommu_platform=on,bus=virtio-msg-proxy-bus.1 \
        -device virtio-net-device,netdev=net2,iommu_platform=on,bus=virtio-msg-proxy-bus.2 \
        -netdev type=user,id=net0 \
        -netdev type=user,id=net1 \
        -netdev type=user,id=net2 \
        -smp 1,maxcpus=1 \
        -serial mon:stdio \
        -m 1152M
```

## Running on x86 xl

x86 hyperlaunch flows do not yet have support for configuring non-blocking virtio-mmio.
The edgar/vmp branch has a hack that statically creates a single non-blocking
virtio-mmio device for x86 pvh guests.

The hack creates a node with the following configuration:
```
rc = vmp_init(&d->vmp[0], d, 0, 0xfe000000, 0xfe001000, 20);
```

On x86, we need to pass along an ACPI table entry describing this node to the
guest kernel, otherwise virtio-mmio won't be found.

This can be done by creating a virtio-pvh.asl file containing the following:

```
DefinitionBlock ("virtio-pvh.aml", "SSDT", 2, "Xen", "HVM", 0)
{
    Device (VR28) {
        Name (_HID, "LNRO0005")
        Name (_UID, 28)
        Name (_CRS, ResourceTemplate() {
            Memory32Fixed (ReadWrite, 0xfe000000, 0x0200)
            Interrupt (ResourceConsumer, Level, ActiveHigh, Exclusive)  { 20 }
        })
    }
}
```

Compile it into binary aml:
```console
$ iasl virtio-pvh.asl 

Intel ACPI Component Architecture
ASL+ Optimizing Compiler/Disassembler version 20200925
Copyright (c) 2000 - 2020 Intel Corporation

ASL Input:     virtio-pvh.asl -     338 bytes      4 keywords     13 source lines
AML Output:    virtio-pvh.aml -      97 bytes      0 opcodes       4 named objects

Compilation successful. 0 Errors, 0 Warnings, 0 Remarks, 0 Optimizations
```

Now, in your domain xl configuration add the following:
```
acpi_firmware="virtio-pvh.aml"
device_model_version="qemu-xen"
device_model_override="/usr/bin/qemu-system-i386"

device_model_args=[
'-device', 'virtio-msg-bus-xen,shm-base=0xfe001000,bus=virtio-msg-bus.0',
'-device', 'virtio-net-device,netdev=n0,iommu_platform=on,bus=virtio-msg-proxy-b
us.0',
'-netdev', 'type=user,id=n0',
"-global", "virtio-mmio.force-legacy=false",
"-global", "virtio-pci.disable-legacy=on",
"-global", "virtio-pci.disable-modern=off",
'-global', 'virtio-net-device.iommu_platform=on',
'-global', 'virtio-blk-device.iommu_platform=on',
]

```

