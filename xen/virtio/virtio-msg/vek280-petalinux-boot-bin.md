# Howto generate virtio-msg-demo BOOT.BIN for VEK280 using PetaLinux

1. Install PetaLinux 2024.1 and the PetaLinux VEK280 2024.1 BSP.
2. Source PetaLinux
```console
$ . /opt/Xilinx/PetaLinux/2024.1/settings.sh

```

3. Create a PetaLinux VEK280 project from the BSP

```console
$ petalinux-create project -s xilinx-vek280-v2024.1-final.bsp -n vek280
[INFO] Create project: vek280
[INFO] New project successfully created in /home/edgar/pl/vek280
```

4. Configure the HW based on the VEK280 virtio-msg enabled XSA.

```consolea
$ petalinux-config --get-hw-description design_1_wrapper.xsa
```
And save the configuration unmodified.

5. Fix pl.dtsi

PetaLinux is unable to generate a proper dts from the design_1_wrapper.xsa.
The PL dts fragment is broken so we need to fix it. I removed most of the
pl.dtsi since we're not going to use the logic in our first demo.

I've included a working pl.dtsi fragment into this repo, you can download
it here [pl.dtsi](pl.dtsi).

Copy pl.dtsi into the PetaLinux project:
```console
$ cp pl.dtsi ./components/plnx_workspace/device-tree/device-tree/pl.dtsi
```

6. Build PetaLinux images

```console
$ petalinux-build
```

7. Build BOOT.BIN

```console
$ petalinux-package boot --u-boot --force
[INFO] Getting Default pdi file
[INFO] File in BOOT BIN: "/home/edgar/pl/vek280/project-spec/hw-description/design_1_wrapper.pdi"
[INFO] File in BOOT BIN: "/home/edgar/pl/vek280/images/linux/plm.elf"
[INFO] File in BOOT BIN: "/home/edgar/pl/vek280/images/linux/psmfw.elf"
[INFO] File in BOOT BIN: "/home/edgar/pl/vek280/images/linux/system.dtb"
[INFO] File in BOOT BIN: "/home/edgar/pl/vek280/images/linux/bl31.elf"
[INFO] File in BOOT BIN: "/home/edgar/pl/vek280/images/linux/u-boot.elf"
[INFO] Generating versal binary package BOOT.BIN...
[INFO] 

****** Bootgen v2024.1
  **** Build date : Apr 29 2024-12:18:25
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.
    ** Copyright 2022-2024 Advanced Micro Devices, Inc. All Rights Reserved.


[INFO]   : Bootimage generated successfully


[INFO] Generating QEMU boot images...
[INFO] File in qemu_boot.img: /home/edgar/pl/vek280/images/linux/BOOT.BIN
[INFO] File in qemu_boot.img: /home/edgar/pl/vek280/images/linux/boot.scr
[INFO] File in qemu_boot.img: /home/edgar/pl/vek280/images/linux/ramdisk.cpio.gz.u-boot
[INFO] Binary is ready.
[INFO] Successfully Generated BIN File
```

