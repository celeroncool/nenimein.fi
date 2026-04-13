---
title: "How to add boot options in PopOS! 22.04"
date: 2023-08-10T08:51:33+03:00
draft: false
tags: [linux, lang-english]
---

Most of the guides tell you to just add a grub option. Well, PopOS! 22.04 does not use GRUB, if you have a UEFI motherboard and installed PopOS! with UEFI (no CSM enabled)...

# kernelstub

Official documentation from System76:
[https://support.system76.com/articles/kernelstub/](https://support.system76.com/articles/kernelstub/)


## sudo kernelstub -p

This will print out your current settings:

```
kernelstub.Config    : INFO     Looking for configuration...
kernelstub           : INFO     System information: 

    OS:..................Pop!_OS 22.04
    Root partition:....../dev/nvme0n1p2
    Root FS UUID:........
    ESP Path:............/boot/efi
    ESP Partition:......./dev/nvme0n1p3
    ESP Partition #:.....3
    NVRAM entry #:.......-1
    Boot Variable #:.....0000
    Kernel Boot Options:.quiet splash amdgpu.ppfeaturemask=0xfff7ffff acpi_enforce_resources=lax
    Kernel Image Path:.../boot/vmlinuz-6.4.6-76060406-generic
    Initrd Image Path:.../boot/initrd.img-6.4.6-76060406-generic
    Force-overwrite:.....False

kernelstub           : INFO     Configuration details: 

   ESP Location:................../boot/efi
   Management Mode:...............True
   Install Loader configuration:..True
   Configuration version:.........3
```

## sudo kernelstub -o "options"

This will modify current kernel options. Beware, this will not ADD, but EDIT, so you will loose your current options...
In this example, I'm adding a custom resolution. For some reason my display only shows up as 120Hz, but it is a 144Hz Freesync Premium monitor (works fine in Windows).

```
sudo kernelstub -o "quiet splash amdgpu.ppfeaturemask=0xfff7ffff acpi_enforce_resources=lax video=DP-3:3440x1440@144"
```

## Finishing up

Now check again with kernelstub -p that your options have been made.

```
sudo kernelstub -p
kernelstub.Config    : INFO     Looking for configuration...
kernelstub           : INFO     System information: 

    OS:..................Pop!_OS 22.04
    Root partition:....../dev/nvme0n1p2
    Root FS UUID:........
    ESP Path:............/boot/efi
    ESP Partition:......./dev/nvme0n1p3
    ESP Partition #:.....3
    NVRAM entry #:.......-1
    Boot Variable #:.....0000
    Kernel Boot Options:.quiet splash amdgpu.ppfeaturemask=0xfff7ffff acpi_enforce_resources=lax video=DP-3:3440x1440@144
    Kernel Image Path:.../boot/vmlinuz-6.4.6-76060406-generic
    Initrd Image Path:.../boot/initrd.img-6.4.6-76060406-generic
    Force-overwrite:.....False

kernelstub           : INFO     Configuration details: 

   ESP Location:................../boot/efi
   Management Mode:...............True
   Install Loader configuration:..True
   Configuration version:.........3
```