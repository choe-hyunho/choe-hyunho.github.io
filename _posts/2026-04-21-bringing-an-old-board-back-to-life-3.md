---
layout: post
title:  "Bringing an old board back to life - 3"
date:   2026-04-21 13:07:14 +0900
categories: embedded system
comments: true
---
Now, I have a working embedded Linux board. Until now, don't make any patch to make this board working. But to make it useful, can't help but to modify vanillas. In this article, will show how to modify kernel to bring up all the possible peripherals in SMDK2410 board. Again, all the related files on [here](https://github.com/choe-hyunho/miscellaneous/tree/main/Board/SMDK2410).

In modern embedded Linux system, device tree is widely used. Each device has its own device tree, and all the peripherals and detailed settings are stored in device tree. Device tree will be compiled as binary and can be attached behind kernel binary or managed independently. If some peripherals in device should be enabled/disabled, we can easily do it by just modifying its device tree, without modifying kernel source. Besides this, device tree gives us much more flexibilities to embedded Linux system. Especially, every embedded Linux system has its own unique peripherals, and it is called 'platform device'. In SMDK2410 days, each platform device should be registered in 'machine architecture' or 'board' source file. S3C2410/SMDK2410 supports are succeeded to newer kernels, but the legacy platform device management is not changed. So, in this article, I will add platform devices one by one.

# CS8900A Ethernet Controller

Though CS8900A ethernet controller is already working in U-Boot, kernel default configuration does not include it. So, add the following line to  `Additional configuration fragment files` in Buildroot.
```
CONFIG_CS89x0=y
```
And then, modify `arch/arm/mach-s3c24xx/mach-smdk2410.c` as the following patch.
```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -69,8 +69,22 @@
 	}
 };
 
+#define CS8900_BASE (S3C2410_CS3 + 0x01000300)
+
+static struct resource smdk2410_cs89x0_resources[] = {
+	[0] = DEFINE_RES_MEM(CS8900_BASE, 17),
+	[1] = DEFINE_RES_NAMED(IRQ_EINT9, 1, NULL, IORESOURCE_IRQ | IORESOURCE_IRQ_HIGHLEVEL),
+};
+
+static struct platform_device smdk2410_device_eth = {
+	.name		= "cs89x0",
+	.num_resources	= ARRAY_SIZE(smdk2410_cs89x0_resources),
+	.resource	= smdk2410_cs89x0_resources,
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
+	&smdk2410_device_eth,
 	&s3c_device_lcd,
 	&s3c_device_wdt,
 	&s3c_device_i2c0,
```
I can directly modify codes in `buildroot-2019.05.3/output/build/`, but don't do `make clean` stuffs. It will discard all my changes. Just `make linux-rebuild` and then `make all`. In this manner, can continue patching & debugging kernel. After all modification is done and works well, I can make a patch and feed to Buildroot config.

To make ethernet working, need some extra works. Let's test networking with the following commands.
```
ifconfig eth0 down
ifconfig eth0 hw ether 00:0E:3A:24:10:01
ifconfig eth0 192.168.1.250 netmask 255.255.255.0 up
route add default gw 192.168.1.1
```
The second line is required because this board does not have EEPROM for CS8900A, so cannot save its MAC address. Maybe I can modify codes for this, but decided to use config stuffs. If the above commands are working, can ping to my other host, or internet hosts. But, I met some media auto-detection failure messages, and it is because CS8900A is only supporting 10M Ethernet, but my network hub is for 100M. In U-Boot driver, auto-detection not happens and just used default settings, but in Linux driver, media detection is mandatory and this prevents `eth0` up. The same again, I can modify codes or put a config item for it. In this case, media type can be assigned by module parameter or kernel command line. So, I added `cs89x0_media=rj45` to my kernel command line. Then, the `bootargs` in U-Boot will be as below.
```
setenv bootargs console=ttySAC0,115200 cs89x0_media=rj45 ubi.mtd=6 root=ubi0:rootfs rootfstype=ubifs rw
```
OK, now it works. So, to configure network on every boot, add the following lines to `/etc/network/interfaces`.
```
auto eth0
iface eth0 inet dhcp
    hwaddress ether 00:0E:3A:24:10:01
```
That's it. This is the typical platform device registration procedure. Most of platform devices ared registered just like this.

# Button Keys

Another simple & easy additions. SMDK2410 board has 4 GPIO buttons each is assigned to external interrupt pins. The following patch shows defining platform data for GPIO keys and registering as platform device.
```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -19,6 +19,8 @@
 #include <linux/serial_s3c.h>
 #include <linux/platform_device.h>
 #include <linux/io.h>
+#include <linux/input.h>
+#include <linux/gpio_keys.h>
 
 #include <asm/mach/arch.h>
 #include <asm/mach/map.h>
@@ -28,6 +30,9 @@
 #include <asm/irq.h>
 #include <asm/mach-types.h>
 
+#include <mach/regs-gpio.h>
+#include <mach/gpio-samsung.h>
+
 #include <linux/platform_data/i2c-s3c2410.h>
 
 #include <plat/devs.h>
@@ -69,6 +74,46 @@
 	}
 };
 
+static struct gpio_keys_button s3c_btns[] = {
+    {
+        .gpio = S3C2410_GPF(0),   // EINT0
+        .code = KEY_ENTER,
+        .desc = "EINT0",
+        .active_low = 1,
+    },
+    {
+        .gpio = S3C2410_GPF(2),   // EINT2
+        .code = KEY_ESC,
+        .desc = "EINT2",
+        .active_low = 1,
+    },
+    {
+        .gpio = S3C2410_GPG(3),   // EINT11
+        .code = KEY_UP,
+        .desc = "EINT11",
+        .active_low = 1,
+    },
+    {
+        .gpio = S3C2410_GPG(11),  // EINT19
+        .code = KEY_DOWN,
+        .desc = "EINT19",
+        .active_low = 1,
+    },
+};
+
+static struct gpio_keys_platform_data s3c_btn_data = {
+    .buttons = s3c_btns,
+    .nbuttons = ARRAY_SIZE(s3c_btns),
+};
+
+static struct platform_device s3c_device_btn = {
+    .name = "gpio-keys",
+    .id = -1,
+    .dev = {
+        .platform_data = &s3c_btn_data,
+    },
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
 	&s3c_device_lcd,
@@ -75,6 +120,7 @@
 	&s3c_device_wdt,
 	&s3c_device_i2c0,
 	&s3c_device_iis,
+	&s3c_device_btn,
 };
 
 static void __init smdk2410_map_io(void)
```
And, the following should be added to kernel configuration.
```
CONFIG_KEYBOARD_GPIO=y
```

# Real Time Clock (RTC)

The next device is Real Time Clock(RTC). It is not so very useful for SMDK2410 board, because this board does not have coin-cell battery socket. This means if I recycles power, the date and time stored in RTC is lost. But during examining & modifying the driver codes, I found this is fairly good example of implementing platform device, so will introduce here.

The current driver codes in Linux 5.x is modified since it is originally written. Maybe it is because there are many other chips which are successors of S3C2410, and this RTC code is shared between them, so is rewritten to match up with device tree mechanism. Unfortunately, during that process, the original platform device mechanism was disappeared by accident, or on purpose. So, I need to modify driver code to work with legacy way. The following code shows this.
```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -73,6 +73,7 @@
 	&s3c_device_ohci,
 	&s3c_device_lcd,
 	&s3c_device_wdt,
+	&s3c_device_rtc,
 	&s3c_device_i2c0,
 	&s3c_device_iis,
 };
--- linux-5.1.21.orig/drivers/rtc/rtc-s3c.c
+++ linux-5.1.21/drivers/rtc/rtc-s3c.c
@@ -443,6 +443,24 @@
 	return 0;
 }
 
+static const struct of_device_id s3c_rtc_dt_match[];
+
+static const struct s3c_rtc_data *s3c_rtc_get_data(struct platform_device *pdev)
+{
+	const struct of_device_id *match;
+	const struct platform_device_id *id;
+
+	match = of_match_node(s3c_rtc_dt_match, pdev->dev.of_node);
+	if (match)
+		return match->data;
+
+	id = platform_get_device_id(pdev);
+	if (id)
+		return (const struct s3c_rtc_data *)id->driver_data;
+
+	return NULL;
+}
+
 static int s3c_rtc_probe(struct platform_device *pdev)
 {
 	struct s3c_rtc *info = NULL;
@@ -462,7 +480,7 @@
 	}
 
 	info->dev = &pdev->dev;
-	info->data = of_device_get_match_data(&pdev->dev);
+	info->data = s3c_rtc_get_data(pdev);
 	if (!info->data) {
 		dev_err(&pdev->dev, "failed getting s3c_rtc_data\n");
 		return -EINVAL;
@@ -805,6 +823,27 @@
 	.disable		= s3c6410_rtc_disable,
 };
 
+static const struct platform_device_id s3c_rtc_driver_ids[] = {
+	{
+		.name	= "s3c2410-rtc",
+		.driver_data = (kernel_ulong_t)&s3c2410_rtc_data,
+	}, {
+		.name	= "s3c2416-rtc",
+		.driver_data = (kernel_ulong_t)&s3c2416_rtc_data,
+	}, {
+		.name	= "s3c2443-rtc",
+		.driver_data = (kernel_ulong_t)&s3c2443_rtc_data,
+	}, {
+		.name	= "s3c6410-rtc",
+		.driver_data = (kernel_ulong_t)&s3c6410_rtc_data,
+	}, {
+		.name	= "exynos3250-rtc",
+		.driver_data = (kernel_ulong_t)&s3c6410_rtc_data,
+	},
+	{ /* sentinel */ },
+};
+MODULE_DEVICE_TABLE(platform, s3c_rtc_driver_ids);
+
 static const struct of_device_id s3c_rtc_dt_match[] = {
 	{
 		.compatible = "samsung,s3c2410-rtc",
@@ -829,6 +868,7 @@
 static struct platform_driver s3c_rtc_driver = {
 	.probe		= s3c_rtc_probe,
 	.remove		= s3c_rtc_remove,
+	.id_table	= s3c_rtc_driver_ids,
 	.driver		= {
 		.name	= "s3c-rtc",
 		.pm	= &s3c_rtc_pm_ops,
```

Tha above codes register platform device first. Platform device is not defined additionally because it is already defined in architecture code and shared. But even after device register, RTC device is not discovered in boot process. It is because the driver code was designed to use with device tree mechanism. `s3c_rtc_dt_match` structure array represents compatible device lists for this driver, and works against compatible name of device tree entries. There is also similar scan mechanism for legacy platform devices, but is not implemented for this driver. So, I added it. `s3c_rtc_driver_ids` structure array represents device lists which this driver can handle, and can use this mechanism by adding `.id_table` member of driver structure, and calling `platform_get_device_id()` function.

Actually, this code is not made by me, but I re-applied removed codes from old kernel. Anyway, this patch is fairly good example for platform device scanning, and now RTC device is successfully registered during boot process.