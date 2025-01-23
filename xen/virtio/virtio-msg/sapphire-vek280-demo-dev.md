# Developers documentation for the virtio-msg demo

This will outline how to build the various components from source such that they can integrate into other environments than the one used to build the binary drop.

## Repos

Linux:
https://github.com/edgarigl/linux/tree/edgari/virtio-msg-sapphire
https://github.com/edgarigl/linux/tree/edgari/virtio-msg-versal

QEMU:
https://github.com/edgarigl/qemu/tree/edgar/virtio-msg-new

## Linux for Sapphire

To create the demo we used a Linux kernel version based on Bill Mill's tree (Linaro).
This tree has in-kernel drivers with support for the virtio-msg transport.
The relevant options to select are:
CONFIG_VIRTIO_MSG=y
CONFIG_VIRTIO_MSG_AMP=m
CONFIG_VIRTIO_MSG_SAPPHIRE=m

## Linux for Versal

The demo uses the version of Linux released with PetaLinux 2024.1 + some patches to add the UIO driver for QEMU to access PCI host memory and to receive interrupts.
The relevant kernel options are:
CONFIG_UIO_XILINX_VERSAL_VIRTIO_MSG=m

## QEMU

QEMU was built from a tree taken from upstream with a series of patches that add virtio-msg support.
Since the rootfs for Versal was from a custom Yocto, an SDK (toolchain) was created for that same Yocto setup.
With that toolchain environment sourced/enabled, the following confiure options were used for QEMU:
```console
PKG_CONFIG=pkg-config ../qemu/configure \
        --enable-debug-info \
        --prefix=/opt/qemu/master/ \
        --target-list=aarch64-softmmu \
        --enable-tcg \
        --enable-kvm \
        --enable-xen \
        --enable-libusb \
        --disable-libusb \
        --disable-libudev \
        --disable-bzip2 \
        --disable-snappy \
        --disable-lzo \
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
        --enable-trace-backends=nop \
        --extra-cflags="-Og" \
        --disable-linux-aio \
        --disable-linux-io-uring \
        --enable-trace-backends=log \
        --enable-slirp \
        --disable-werror
```

The exact set of options doesn't matter but you'll need --target-list=aarch64-softmmu and --disable-werror at minimum.

