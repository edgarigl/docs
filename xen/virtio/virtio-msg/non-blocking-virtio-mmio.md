# Non-blocking virtio-mmio

Non-blocking virtio-mmio is a non-standard extension to virtio-mmio
allowing guests to issue indirect register accesses to the virtio-mmio
register space in a non-blocking manner.

To try it out, you'll need to use the following branches of QEMU, Linux
and Xen:

QEMU:
https://github.com/edgarigl/qemu/tree/edgar/virtio-msg

Linux:
https://github.com/edgarigl/linux/tree/edgar/virtio-mmio-nb

Xen:
https://github.com/edgarigl/xen/tree/edgar/vmp

## Linux

Build Linux from the mentioned branch and use that for the domU guest.
You'll need to enable:
CONFIG_VIRTIO_MMIO=y


## Xen

In Xen, you'll need to enable the following:
```
CONFIG_VIRTIO_MMIO_NON_BLOCKING=y
CONFIG_VIRTIO_MSG_BUS_XEN=y
CONFIG_NR_VIRTIO_MSG_BUSSES=8
```

## Running on ARM64 dom0less

Add the following fdt property to your dom0less domU description to
instantiate 2 virtio-mmio-nonblocking nodes:
```
  virtio@2001000 {
        compatible = "xen,virtio-mmio-nonblocking";
        reg = <0x0 0x2001000 0x0 0x1000>;
        irq = <0x22>;

        bus@0 {
                compatible = "xen,virtio-msg-bus-xen";
                device-domid = < 0 >;
                bus-id = <0>;
        };
  };

  virtio@2002000 {
        compatible = "xen,virtio-mmio-nonblocking";
        reg = <0x0 0x2002000 0x0 0x1000>;
        irq = <0x23>;

        bus@1 {
                compatible = "xen,virtio-msg-bus-xen";
                device-domid = < 0 >;
                bus-id = <1>;
        };
  };

```

Also, add a passthrough device-tree fragment containing the virtio-mmio nodes:

Here's an example of a pt.dts:
```
/dts-v1/;

/ {
    #address-cells = <2>;
    #size-cells = <2>;

    gic: gic {
            /* Dummy gic node, only used for phandle reference.
             * Xen will populate real vGIC node. */
            #interrupt-cells = <3>;
            interrupt-controller;
    };

    passthrough {
            compatible = "simple-bus";
            #address-cells = <2>;
            #size-cells = <2>;
            ranges;

            virtio@2001000 {
                    dma-coherent;
                    compatible = "virtio,mmio";
                    interrupts = <0x0 0x02 0xf01>;
                    interrupt-parent = <&gic>;
                    reg = <0x0 0x2001000 0x0 0x200>;

                    // Enable grants towards dom0
                    iommus = < &grants 0 >;
            };

            virtio@2002000 {
                    dma-coherent;
                    compatible = "virtio,mmio";
                    interrupts = <0x0 0x03 0xf01>;
                    interrupt-parent = <&gic>;
                    reg = <0x0 0x2002000 0x0 0x200>;
                    // No grants, this will use foreign mappings
            };

            grants: xen_iommu {
                    #iommu-cells = <0x01>;
                    compatible = "xen,grant-dma";
            };

    };
};
```

During boot, domU will get stuck at boot spinning waiting for virtio-mmio
non-blocking to respond. Once dom0 is done booting, run for the following
(adjust memory, maxcpus etc to fit your setup):
```console
qemu-system-aarch64 -M xenpvh \
        -xen-domid 1 \
        -xen-attach -name 1 \
        -no-shutdown \
        -display none \
        -device virtio-msg-bus-xen,bus-id=0,bus=virtio-msg-bus.0 \
        -device virtio-net-device,netdev=n0,iommu_platform=on,bus=virtio-msg-proxy-bus.0 \
        -netdev user,id=n0 \
        -device virtio-msg-bus-xen,bus-id=1,bus=virtio-msg-bus.1 \
        -device virtio-net-device,netdev=n1,iommu_platform=on,bus=virtio-msg-proxy-bus.1 \
        -netdev user,id=n1 \
        -smp 1,maxcpus=1 \
        -serial mon:stdio \
        -m 1024M
```

## Running on x86/hyperlaunch

To run on x86/hyperlaunch, the dom0less fdt bindings are the same as for ARM.

On x86, we need to pass along an ACPI table entry describing this node to the
guest kernel, otherwise virtio-mmio won't be found.

This can be done by creating a virtio-pvh.asl file containing the following:

```
DefinitionBlock ("virtio-pvh.aml", "SSDT", 2, "Xen", "HVM", 0)
{

    Device (VR00) {
        Name (_HID, "LNRO0005")
        Name (_UID, 00)
        Name (_CRS, ResourceTemplate() {
            Memory32Fixed (ReadWrite, 0xfe000000, 0x0200)
            Interrupt (ResourceConsumer, Level, ActiveHigh, Exclusive)  { 20 }
        })
    }

    Device (VR01) {
        Name (_HID, "LNRO0005")
        Name (_UID, 01)
        Name (_CRS, ResourceTemplate() {
            Memory32Fixed (ReadWrite, 0xfe001000, 0x0200)
            Interrupt (ResourceConsumer, Level, ActiveHigh, Exclusive)  { 21 }
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

We need to inject this ssdt into domU. We're going to use the method described here:
https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html

You'll need to create an initrd, here's an example Makerule I use to add
it to my initrd:

initrd: ${ROOTFS} virtio-pvh.aml
        # Inject ssdt.aml
        mkdir -p kernel/firmware/acpi
        cp virtio-pvh.aml kernel/firmware/acpi/
        find kernel | cpio -H newc --create >initrd
        cat ${ROOTFS} >>initrd

Once domU boots, you'll see it gets stuck waiting for virtio-mmio. So launch QEMU:

```console
qemu-system-i386 -M xenpvh,ram-low-base=0x0,ram-low-size=0xf0000000 \
        -machine ram-high-base=0x100000000,ram-low-size=0x20fffffff \
        -xen-domid ${DOMID} \
        -xen-attach -name ${DOMID} \
        -m 8G   \
        -smp 1,maxcpus=1 \
        -display none \
        -device virtio-msg-bus-xen,bus-id=0,bus=virtio-msg-bus.0 \
        -device virtio-blk-device,drive=d0,bus=virtio-msg-proxy-bus.0 \
        -drive if=none,id=d0,file=/dev/sdd1,format=raw \
        -device virtio-msg-bus-xen,bus-id=1,bus=virtio-msg-bus.1 \
        -device virtio-net-device,netdev=n0,iommu_platform=on,mac=52:54:00:12:34:59,bus=virtio-msg-proxy-bus.1 \
        -netdev user,id=n0
```

You'll see a virtio block device and a virtio-net in domU.

