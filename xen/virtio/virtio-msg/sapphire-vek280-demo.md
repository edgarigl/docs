# Sapphire/VEK280 virtio-msg demo for CES

## Running with pre-built binaries

The demo requires booting things in the right order to work.
The steps are:

1. Power off both boards.
2. Power on the VEK 280.
3. Boot Linux/Xen on Versal and run the pci-flr-monitor script.
4. Boot Linux/Xen on the Sapphire board and load the virtio-msg modules.
5. Run the QEMU backend on Versal.
6. The virtio-net network across the boards should now work.

Now we'll describe the steps on the board connected to xsjjaewookb50.

### Package

The package contains the following files:
```console
load.tcl
images-vek280/
images-vek280/xen-image-minimal-qemuarm64.rootfs.cpio.gz.u-boot
images-vek280/BOOT.BIN
images-vek280/system.dtb
images-vek280/boot.scr
images-vek280/boot.script
images-vek280/design_1_wrapper.xsa
images-vek280/Image
images-vek280/kernelconfig
images-sapphire/
images-sapphire/xen-image-minimal-qemux86-64.rootfs.cpio.gz
images-sapphire/bzImage
images-sapphire/xen
images-sapphire/grub.cfg
images-sapphire/kernelconfig
```

The contents of images-vek280 should be copied to /scratch/gitlab-runner/tftp/versal/
The contents of images-sapphire should be copied to /scratch/gitlab-runner/tftp/sapphire/

### Boot of Versal

First, ssh into xsjjaewookb50.

Connect to VEK 280's System Controller UART:
```console
$ sudo picocom -b 115200 /dev/serial/by-id/usb-Xilinx_VEK280_512701B03088-if03-port0
```

Connect to VEK 280's Versal UART:
```console
$ sudo picocom -b 115200 /dev/serial/by-id/usb-Xilinx_VEK280_512701B03088-if01-port0
```

Connect to Sapphire's UART:
```console
$ sudo picocom -b 115200 /dev/serial/by-id/usb-FTDI_USB_Serial_Converter_FTDO6WMH-if00-port0
```

Power off both boards, make sure the VEK 280 is in JTAG boot mode and then power on VEK 280.
```console
# Power off Sapphire
$ /scratch/gitlab-runner/cyberpower-pdu 10.0.6.10 3 2
# Power off VEK 280
$ /scratch/gitlab-runner/cyberpower-pdu 10.0.6.10 4 2
$ sleep 2
# Power on VEK 280
$ /scratch/gitlab-runner/cyberpower-pdu 10.0.6.10 4 1
```

Login to the VEK 280 System Controller's Linux system, on our board the username/password is petalinux/petalinux.
Then reset the Versal SoC.
```console
Username: petalinux
Password: 
eval-brd-sc-zynqmp:~$ sc_app -c  reset
```

Run xsdb and connect to the Versal target:
```console
$ xsdb
rlwrap: warning: your $TERM is 'screen' but rlwrap couldn't find it in the terminfo database. Expect some problems.
                                                                                                                                               
****** System Debugger (XSDB) v2024.1
  **** Build date : May 22 2024-19:19:01
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.
    ** Copyright 2022-2024 Advanced Micro Devices, Inc. All Rights Reserved.


xsdb% connect                                                                                                                                  
tcfchan#0                                                                                                                                      
xsdb% ta 1                                                                                                                                     
xsdb% ta                                                                                                                                       
  1* Versal xcve2802
     2  RPU
        3  Cortex-R5 #0 (Halted)
        4  Cortex-R5 #1 (Lock Step Mode)
     5  APU
        6  Cortex-A72 #0 (Running)
        7  Cortex-A72 #1 (Running)
     8  PPU
        9  MicroBlaze PPU (Sleeping)
    10  PSM
       11  MicroBlaze PSM (Sleeping)
    12  PMC
    13  PL
 14  whole scan chain (board power off)
 15  DPC
xsdb%
```

And then load the BOOT.BIN file:
```
xsdb% source load.tcl                                                                                                                          
INFO: Downloading BIN file: BOOT.BIN to the target.
100%   10MB   1.4MB/s  00:07                                                                                                                   
Info: Cortex-A72 #0 (target 6) Stopped at 0xfffe0700 (External Debug Request)                                                                  
100%    0MB   0.2MB/s  00:00                                                                                                                   
Successfully downloaded /home/edgari/mac-dev/images/system.dtb
INFO: Loading image: boot.scr at 0x20000000
100%    0MB   0.0MB/s  00:00    
Successfully downloaded /home/edgari/mac-dev/boot.scr
Info: Cortex-A72 #0 (target 6) Running
xsdb%
```

You should see the Versal Linux/dom0/Xen system booting on the Versal UART.
Login as root (no password) and we'll run the pci-flr-monitor.sh script.
```console
Starting syslogd/klogd: done
Starting domain watchdog daemon: xenwatchdogd startup


Yocto on Xen development distro 2024.02.10 qemuarm64 /dev/ttyAMA0

qemuarm64 login: root
root@qemuarm64:~# cd /usr/share/virtio-msg-demo/
root@qemuarm64:/usr/share/virtio-msg-demo# ./pci-flr-monitor.sh >/dev/null 
```

Now we're ready to start the Sapphire system.

```console
# Power on Sapphire
$ /scratch/gitlab-runner/cyberpower-pdu 10.0.6.10 3 1
```

After a while, you should see Xen and Dom0 Linux boot on the sapphire UART.

Login as root (no password) and run the run-x86.sh script to load the virtio-msg kernel modules:
```console
Starting xenconsoled...
Starting QEMU as disk backend for dom0
(XEN) [   15.690385] common/grant_table.c:1909:d0v3 Expanding d0 grant table from 1 to 2 frames
Starting domain watchdog daemon: xenwatchdogd startup

[done]
INIT: Id "S0" respawning too fast: disabled for 5 minutes

Yocto on Xen development distro 2024.02.10 qemux86-64 /dev/hvc0

qemux86-64 login: root
root@qemux86-64:~# cd /usr/share/virtio-msg-demo/
root@qemux86-64:/usr/share/virtio-msg-demo# ./run-x86.sh 
+ insmod /usr/share/virtio-msg-demo/virtio_msg_amp.ko
+ insmod /usr/share/virtio-msg-demo/virtio_msg_sapphire.ko
[   92.199688] sapphire_probe
[   92.199721] virtio_msg_sapphire 0000:01:00.0: device_name=0000:01:00.0
[   92.199752] virtio_msg_sapphire 0000:01:00.0: mmr (BAR0) at 0x00000000fca80000, size 0x0000000000080000
[   92.199772] virtio_msg_sapphire 0000:01:00.0: msix (BAR1) at 0x00000000fca00000, size 0x0000000000080000
[   92.199792] virtio_msg_sapphire 0000:01:00.0: shmem (BAR2) at 0x00000000fcb00000, size 0x0000000000010000
[   92.199812] vectors 8
[   92.200744] sapphire_probe: enable bus mastering queue dma 0x0
[   92.200770] sapphire_probe: shmem=000000004f4d65bc a210000
[   92.200783] virtio_msg_sapphire 0000:01:00.0: SHMEM @ 0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 00 
```

At this point, the x86 kernel is waiting for the virtio-msg backend to respond.
On the VEK 280 Versal UART, we're now going to start the backend.

```console
root@qemuarm64:~# cd /usr/share/virtio-msg-demo/
root@qemuarm64:/usr/share/virtio-msg-demo# ./run-arm64.sh 
+ insmod /usr/share/virtio-msg-demo/uio_xilinx_versal_virtio_msg.ko
[  280.612712] uio_dmem_genirq_probe:174
[  280.616410] uio_dmem_genirq_probe:188
[  280.620080] uio_dmem_genirq_probe:194
[  280.623742] uio_dmem_genirq_probe:201
[  280.627405] uio_dmem_genirq_probe:208
[  280.631067] uio_dmem_genirq_probe:215
[  280.634729] uio_dmem_genirq_probe:221
[  280.638468] uio_dmem_genirq_probe:232
[  280.642137] uio_dmem_genirq_probe:250
[  280.645797] uio_dmem_genirq_probe:253
[  280.649453] uio_dmem_genirq_probe: addr 0x4a000000000 size 860000000
[  280.655808] uio_dmem_genirq_probe:275
[  280.659477] uio_dmem_genirq_probe:290
[  280.663148] uio_dmem_genirq_probe:304
[  280.666810] uio_dmem_genirq_probe:309
+ /usr/share/virtio-msg-demo/qemu-system-aarch64 -M x-virtio-msg -m 2G -serial null -display none -daemonize -device virtio-msg-bus-vek280-hexcam,dev=/dev/uio0,spsc-base=0xa210000 -device virtio-net-device,mq=on,netdev=net0,iommu_platform=on -netdev tap,id=net0,ifname=tap0,script=no,downscript=no
[  280.720771] qemu-system-aar[2375]: memfd_create() called without MFD_EXEC or MFD_NOEXEC_SEAL set
ftruncate: Invalid argument
host=0xfff70ddb6000
virtio_set_status: val 0
virtio_set_status: val 0
+ sleep 2
+ echo 'iface tap0 inet dhcp'
+ ifup tap0
udhcpc: started, v1.36.1
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting select for 10.0.6.114, server 10.0.6.1
udhcpc: lease of 10.0.6.114 obtained from 10.0.6.1, lease time 43200
ip: RTNETLINK answers: File exists
/etc/udhcpc.d/50default: Adding DNS 10.0.6.1
```

Now should have IP addresses on Versal's tap0 interface and a different IP address on Sapphire Linux eth3 interface.

Both system's can now reach the network using Sapphire's physical eth1.

