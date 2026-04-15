---
layout: post
title:  "Bringing an old board back to life - 1"
date:   2026-04-15 12:53:58 +0900
categories: embedded system
comments: true
---
While cleaning out some old clutter, I found SMDK2410 development board. It was made almost 20 years ago. Rather ancient than old, and was my first board which could run Embedded Linux. I had lots of memories, mostly painful, with this board. So, decided to bring it back to life! I will describe the process on this blog, and put all the related files on [here](https://github.com/choe-hyunho/miscellaneous/tree/main/Board/SMDK2410).

<div align="center">
  <figure>
    <img src="https://image.ec21.com/image/aijisystem/oimg_GC00150259_CA00282306/High%2Dend-Mobile-System-Reference-Board-SMDK2410-.jpg?v=171053">
    <figcaption>Aiji System SMDK2410 development board</figcaption>
  </figure>
</div>

# First Objective

My first objective is building recent Linux system on this board, and making all the peripherals working, with minimal efforts. And then, try to find something funny stuffs that this board is capable of.

Soon after I search the internet, realized that this board is ancient one. Most of resources are gone, or fragmented. As far as I remember, even in that days, most of works to make this board running existed as series of patches, or source tarballs. And now, almost impossible to find them in the internet. So, I decided to start from finding vanilla software bundles which still supports this hardware.

# Toolchain

Because I plan to use `buildroot` to build my root filesystem, toolchain will be built together with `buildroot`. But, first, I need to build bootloader, searched for prebuilt toolchain. Though I am a big fan of [crosstool-ng](https://crosstool-ng.github.io/), this time, will use proven prebuilt toolchain.

SMDK2410 uses Samsung S3C2410 chip, and it is ARM920T architecture, which is in modern terms, `armv4t`. This is quite legacy architecture, so modern compiler/linker no longer supports fully for that. Actually, when I tried to build `u-boot-2016.11`, gcc 5.x based toolchain causes linker errors. So, my pick is gcc 4.x, more specifically gcc 4.9 toolchain.

It was quite surprise that many of toolchain providing sites were vanished. CodeSourcery was disappeared after it merged to some graphic company, Linaro toolchains are now being legacy and archived only some parts of them [here](https://developer.arm.com/Downloads/-/Legacy%20Linaro%20GNU%20Toolchains). (BTW, recent ARM toolchain can be downloaded at [ARM official site](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads).) Another good site to get prebuilt toolchain for various architecture is [bootlin](https://toolchains.bootlin.com/). They are actually made using `buildroot`, but by providing toolchains for various architectures and `libc` variants, this site can save your time.

Anyway, I selected gcc 4.9 toolchain from Legacy Linaro collections, and will use it until I get working bootloader.

> [!NOTE]
> In that era, most hardware vendors provide toolchain besides BSP. And further, toolchain components like `gcc` or `binutils` are not mature enough, so the quality of toolchain varies. This is why proven toolchain has importance. For example, some toolchain works only in the directory where it originally `make install`ed. i.e. If I move toolchain from designated directory to another directory, it may not work properly.

# JTAG

At the very early stage of embedded development, the only tool we can rely on is JTAG. It is originally made for hardware debugging, but the most common use is to write first binary to flash memory. Of course it has very powerful hardware debugging features like hardware breakpoint and [DCC](https://developer.arm.com/documentation/ddi0406/b/Debug-Architecture/Debug-Register-Interfaces/About-the-debug-register-interfaces/The-Debug-Communications-Channel--DCC-), but in the real debugging world, UART or ADB is far more useful than JTAG. First of all, how often will I need to make breakpoint and step into or over through each instructions? In many cases, it has better to put proper messages for logging and check the log than to spend time tracking error-prawn logic one by one. Certainly I can use DCC for the same purpose, but DCC is not so reliable as I expected, and it is slow! More and more cores included in chip, less and less used the 'break and step' debugging strategy.

At present, I have two [Olimex ARM-USB-OCD](https://www.olimex.com/Products/ARM/JTAG/ARM-USB-OCD/) series and one [Segger J-Trace](https://www.segger.com/products/debug-probes/j-trace/) in hands. J-Trace is fairly feasible with vendor-supplied softwares, but ARM-USB-OCD is rather lack of convenience. ARM-USB-OCD itself is also old hardware, so I decided to revive it, too. I put ARM-USB-OCD resources [here](https://github.com/choe-hyunho/miscellaneous/tree/main/JTAG/Olimex_ARM-USB-OCD), and [OpenOCD](https://openocd.org/) informations [here](https://github.com/choe-hyunho/miscellaneous/tree/main/JTAG/OpenOCD).

> [!NOTE]
> Because I build all my embedded projects in [WSL2](https://github.com/microsoft/WSL) and need to share many files between Windows and WSL, the above resources are based on Windows.

Eventually, I did followings and now can connect to target for debugging and writing to NOR flash.

- Apply some tweaks to install JTAG and serial port driver to Windows.
- Modify [S3C2410 chip configuration file for OpenOCD](https://github.com/choe-hyunho/miscellaneous/blob/main/JTAG/OpenOCD/samsung_s3c2410.cfg).
- Write [SMDK2410 board configuration file for OpenOCD](https://github.com/choe-hyunho/miscellaneous/blob/main/JTAG/OpenOCD/smdk2410.cfg).

# Bootloader

Almost always, [U-Boot](https://u-boot.org/) is the popular choice for embedded system bootloader. Of course there are choices like [LK](https://github.com/littlekernel/lk) or others, but U-Boot is the all-time first choice for embedded systems. Since recent versions drop old hardware supports, I should browse sources to get the last version supported. In conclusion, the last version supports SMDK2410 is `2016.11`, but after some experimenting, I chose `2012.04.01` in the end.

To explain why I choose that version, `u-boot.bin` binary should not exceed 458KB. Maximum size of u-boot block size is limited to 512KB for SMDK2410 board, and the last 64KB should be reserved for environment variable storage, the actual binary size maximum is 458KB. If I modify the config header, certainly can extend the size, but it requires code change or patch, so I select a little bit older version to fit size.

Fortunately, U-Boot is built well with the following commands:
```
CROSS_COMPILE=arm-eabi- make smdk2410_config
CROSS_COMPILE=arm-eabi- make
```
You need to set `PATH` to include Linaro toolchain bin, or put the full path of binaries to `CROSS_COMPILE` variable.

Now, I have `u-boot.bin`, and write it to NOR flash. It boots!

> [!NOTE]
> S3C2410 chip has NAND "steppingstone" feature. i.e. The first 4KB of NAND flash copied to SRAM on boot, then jump. But official U-Boot source never adapted this feature, and only supports NOR booting. I'll try to handle this in the future.