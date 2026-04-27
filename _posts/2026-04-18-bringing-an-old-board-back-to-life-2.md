---
layout: post
title:  "Bringing an old board back to life - 2"
date:   2026-04-18 20:59:01 +0900
categories: embedded system
comments: true
---
The implementation status and default configuration in U-Boot official release for SMDK2410 is fairly good. First, NAND read & write is working perfectly, and various filesystem supports work, including UBIFS, YAFFS and even FAT. Besides, CS8900A Ethernet port is working, and USB function also included. Now is the time to build Linux kernel and basic root filesystem! And still, all the related files on [here](https://github.com/choe-hyunho/miscellaneous/tree/main/Board/SMDK2410).

# Buildroot

As I said in previous article, will use `buildroot` to make toolchain, kernel, root filesystem. There are many build systems for embedded linux, like [Open Embedded](https://www.openembedded.org/)/[Yocto Project](https://www.yoctoproject.org/) or ancient [uCLinux](https://sourceforge.net/projects/uclinux/)[^1]. Each of them has its on Cons and Pros, but my personal favorite is `buildroot`, because it is clean, neat and fast.

Now, let's start `buildroot`. The first thing to do is choosing which version to use. Each `buildroot` version is somewhat linked with toolchain version and Linux kernel version. Because I want to build recent version as close as possible, I browsed several versions to check their linkage. Followings are my checklist.

- versions which can build gcc 4.9.x[^2]
- versions which can linux kernel compatible to S3C2410/SMDK2410 (the newer the better)
- versions which was supported for long term[^3]

So, I chose `buildroot-2019.05.3` and started configuring. Unfortunately, at the very start, it causes errors related to `libncurses` compatibility. It is quite well-known in modern distros. So, I just decided to start building in docker. There is official docker image prepared to build `buildroot`. Use it as follows.
```
docker run -it -h buildroot -v $(pwd):/work -w /work buildroot/base
```
This command will start docker environment and mount current directory as /work. Now I can `make menuconfig`.

First, just tries to build toolchain and bootloader. Select `arm920t` in `Target options`, and `gcc 4.9.x` in `Toolchain`. And for the `C library` I choose `musl`[^4]. Then, select `U-Boot`, version to `2012.04.01`, `board name` to `smdk2410`. Now I can `make`, and got toolchain and `u-boot.bin` in `output/images`. Write it to NOR flash with JTAG or previous working `u-boot`. OK, it works! And looks the same as the previous Linaro-built one.

# Linux Kernel

Now is the time to build Linux kernel. I checked kernel sources and confirmed that the latest version S3C2410 is supported is `6.2.16`, but decided to choose easy path. The default kernel which my `buildroot` supports are `5.1.x`, and will use it. Run `make menuconfig` again, and select `Linux Kernel` in `Kernel`, check version, and set `Defconfig name` as `s3c2410`. And finally, set `Kernel binary image format` to `uImage`. Let't try building, succeeded... OK, now I have built kernel successfully.

Before testing on real board, I need to build root filesystem, too. As long as I remembered, the original rootfs might use CRAMFS or JFFS2, but both are not preferable. CRAMFS is read-only compressed RAM filesystem, which is only readable and cannot save settings in /etc on its own. JFFS2 was popular in that days, but was developed for NOR flash system and extremely slow on NAND-based MTD. So, possible choices are YAFFS and UBI, but I remembered YAFFS was never been included in vanilla kernel, which means I need patch kernel for YAFFS. Then the only choice is UBI/UBIFS. UBI is quite immature in that days, but it is used widely on AOSP in various devices, so may be good choice as long as it can be working. I added the following lines to `Additional configuration fragment files`.
```
CONFIG_MTD_UBI=y
CONFIG_UBIFS_FS=y
```
OK, it is built.

# Root Filesystem

Now select `Filesystem images`. Choose `ubi image containing ubifs root filesystem`. SMDK2410 board has [SmartMedia Card(SMC)](https://en.wikipedia.org/wiki/SmartMedia) slot, and I have a 64MB SMC card. With information reported by `U-Boot`, page size is 512 bytes, OOB size is 16 bytes, and erase block size is 16 KBytes. Now, I can calculate `buildroot` required UBI/UBIFS parameters. `physical eraseblock size` is the same as above, `0x4000`, `sub-page size` is normally equal to NAND page size or a half of it, then I set to `256` (if it works, will use this, but not working, change to 512). And for `logical eraseblock size`, it is `physical eraseblock size` minus two header pages(EC header & VID header). Because `minimum I/O size` should be the same as page size, it should be `0x200`, and so, `logical eraseblock size` is 0x4000 minus two sub-pages(0x200), equal to `0x3e00`.

Now, I have `rootfs.ubi` & `rootfs.ubifs`, and `zImage` with previous kernel build. It's time to write them to NAND flash. Because I didn't change partition table, this kernel build will follow the default partition table for the machine. It can be found somewhere in `arch/arm/mach-s3c24xx` in kernel tree, or can be shown in UART console, if UART works.
```
  0x000000000000-0x000000004000 : "Boot Agent"
  0x000000000000-0x000000200000 : "S3C2410 flash partition 1" (2M)
  0x000000400000-0x000000800000 : "S3C2410 flash partition 2" (4M)
  0x000000800000-0x000000a00000 : "S3C2410 flash partition 3" (2M)
  0x000000a00000-0x000000e00000 : "S3C2410 flash partition 4" (4M)
  0x000000e00000-0x000001800000 : "S3C2410 flash partition 5" (10M)
  0x000001800000-0x000003000000 : "S3C2410 flash partition 6" (24M)
  0x000003000000-0x000004000000 : "S3C2410 flash partition 7" (16M)
```
But, to use full capacity of NAND, we can define partions in kernel command line rather than modifying default partition table in code. The following commad line will use first 2MB for `u-boot`, next 6MB for `kernel`, and remains for `rootfs`.
```
mtdparts=NAND:2m(u-boot),6m(kernel),-(rootfs)
```
If this command line is properly parsed in kernel, the kernel message will be changed as follows.
```
3 cmdlinepart partitions found on MTD device NAND
Creating 3 MTD partitions on "NAND":
0x000000000000-0x000000200000 : "u-boot"
0x000000200000-0x000000800000 : "kernel"
0x000000800000-0x000004000000 : "rootfs"
```
So, write each binaries to proper location as follows.
```
tftpboot 0x30000000 uImage
nand erase 0x200000 0x600000
nand write 0x30000000 0x200000 ${filesize}

tftpboot 0x30000000 rootfs.ubi
nand erase 0x800000 0x3800000
nand write 0x30000000 0x800000 ${filesize}
```
Before loading images from network, I configured TFTP server[^5] to my PC, put proper files to service directory. and set up the below variables to U-Boot.
```
setenv ethaddr 00:0E:3A:24:10:01
setenv ipaddr 192.168.1.251
setenv serverip 192.168.1.200
setenv netmask 255.255.255.0
setenv gatewayip 192.168.1.1
```
This is my configurations, and should be changed properly to match your network environment. If network access is not available, you can use serial port to download images. Use `loadb` or `loady`, but it will be extremely sloooow. And if you want to store these variables, run `saveenv`.

Then, it's time to boot board to Linux! Run the below commands and let's see what happens.
```
setenv mtdids nand0=NAND
setenv mtdparts mtdparts=NAND:2m(u-boot),6m(kernel),-(rootfs)
setenv bootargs console=ttySAC0,115200 ${mtdparts} ubi.mtd=2 root=ubi0:rootfs rootfstype=ubifs rw
setenv bootcmd nand read 0x30000000 0x200000 0x600000\; bootm 0x30000000
```
OK, good. It is booted to login prompt. Entering `root`, I can get shell prompt. I have listed `/bin`,`/sbin`,`/usr/bin`, and `/usr/sbin`. Wow, what a neat!!! There is only one binary, `busybox` exists, and all the others are symbolic links. I have clean Embedded Linux System in SMDK2410!

---

[^1]: uCLinux is originally made for non-MMU systems, but was quite popular and broadly used for even MMU systems, too. It is still archived.
[^2]: This is because of S3C2410 architecture issue. ARM920T is deprecated quite long ago. Of course, recent gcc still supports this architecture with proper compile options, but ABI or several compatibility problems occur on >= gcc 5.x.
[^3]: LTS support policies and rules are made quite recently, but in the past release, revision number after build year and month means similar to LTS.
[^4]: Let's try this. I could choose among `uClibc-ng`,`glibc`, and `musl`. `glibc` is also used for PC Linux, but quite big, and `uClibc-ng` was used in popular for embedded, but quite legacy.
[^5]: I used [Tftpd64](https://pjo2.github.io/tftpd64/).