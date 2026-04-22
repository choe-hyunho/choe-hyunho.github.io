---
layout: post
title:  "Bringing an old board back to life - 4"
date:   2026-04-22 13:05:54 +0900
categories: embedded system
comments: true
---
Continuing from the previous post, I will register more platform devices to my board. As before, all the related files on [here](https://github.com/choe-hyunho/miscellaneous/tree/main/Board/SMDK2410).

# LCD

The next platform device to be enabled is LCD display. SMDK2410 board has 240*320 LCD display sub-board. I expected that this board was already implemented and can easily be enabled, but surprisingly, there is no exact implementation. So, I need to gather information about display module to enable it.

Nowadays, display parts for embedded/mobile processor is quite be standardized, MIPI-DSI is the one. Most of processors has MIPI-DSI interface, and if I know characteristics of display module, like pixel clock, horizontal front/back porch, vertical front/back porch, horizontal/vertical sync length, etc., the physical interface itself is almost the same across platforms, and can easily adapt it by applying correct values.

But, for this chip and board, it was made quite before this standard, so proprietary LCD controller which can support both STN & TFT LCD is included. And, it was only after I finally took sub-board apart that LCD module part no. is `LTS350Q1-PD1`. I managed to find its datasheet from Internet, but need to spend some time to figure out how to apply settings properly to the LCD controller driver. Anyway, the result is quite a simple as follows.

```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -28,6 +28,10 @@
 #include <asm/irq.h>
 #include <asm/mach-types.h>
 
+#include <mach/regs-gpio.h>
+#include <mach/regs-lcd.h>
+
+#include <mach/fb.h>
 #include <linux/platform_data/i2c-s3c2410.h>
 
 #include <plat/devs.h>
@@ -69,6 +73,38 @@
 	}
 };
 
+static struct s3c2410fb_display smdk2410_lcd_cfg __initdata = {
+	.lcdcon5	= S3C2410_LCDCON5_FRM565 |
+			  S3C2410_LCDCON5_INVVLINE |
+			  S3C2410_LCDCON5_INVVFRAME |
+			  S3C2410_LCDCON5_PWREN |
+			  S3C2410_LCDCON5_HWSWP,
+
+	.type		= S3C2410_LCDCON1_TFT,
+
+	.width		= 240,
+	.height		= 320,
+
+	.pixclock	= 200000,//141088,
+	.xres		= 240,
+	.yres		= 320,
+	.bpp		= 16,
+	.left_margin	= 3,
+	.right_margin	= 7,
+	.hsync_len	= 5,
+	.upper_margin	= 2,
+	.lower_margin	= 3,
+	.vsync_len	= 2,
+};
+
+static struct s3c2410fb_mach_info smdk2410_fb_info __initdata = {
+	.displays	= &smdk2410_lcd_cfg,
+	.num_displays	= 1,
+	.default_display = 0,
+
+	.lpcsel = 0x03,
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
 	&s3c_device_lcd,
@@ -92,6 +129,7 @@
 
 static void __init smdk2410_init(void)
 {
+	s3c24xx_fb_set_platdata(&smdk2410_fb_info);
 	s3c_i2c0_set_platdata(NULL);
 	platform_add_devices(smdk2410_devices, ARRAY_SIZE(smdk2410_devices));
 	smdk_machine_init();
```

Vanilla kernel already registered LCD device, but omitted framebuffer registration, so I added it and display setting data. That's it. Now I have color display working.

# Touch Screen

Enabling touch screen device is quite simple. S3C2410 has its dedicated touchscreen driver, so enable in kernel config, and set platform data as belows.

```
CONFIG_TOUCHSCREEN_S3C2410=y
```
```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -33,6 +33,7 @@
 #include <asm/mach-types.h>
 
 #include <linux/platform_data/i2c-s3c2410.h>
+#include <linux/platform_data/touchscreen-s3c2410.h>
 
 #include <plat/devs.h>
 #include <plat/cpu.h>
@@ -74,6 +75,8 @@
 	&s3c_device_wdt,
 	&s3c_device_i2c0,
 	&s3c_device_iis,
+	&s3c_device_adc,
+	&s3c_device_ts,
 };
 
 static void __init smdk2410_map_io(void)
@@ -94,6 +97,7 @@
 
 static void __init smdk2410_init(void)
 {
+	s3c24xx_ts_set_platdata(&smdk2410_ts_info);
 	s3c_i2c0_set_platdata(NULL);
 	platform_add_devices(smdk2410_devices, ARRAY_SIZE(smdk2410_devices));
 	smdk_machine_init();
```

Another thing should be noted is ADC device & driver should also be registered together for touchscreen driver work properly. In my case, touchscreen can be accessed by `/dev/input/event0`(this will vary by the order of registering input devices), and can check with `cat` whether input event occurs.

# SD/MMC Controller

Another typical platform device registration. The following code will add SD/MMC controller to my board. The same as before, set platform data for it, and register as platform device.

```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -19,6 +19,7 @@
 #include <linux/serial_s3c.h>
 #include <linux/platform_device.h>
 #include <linux/io.h>
+#include <linux/mmc/host.h>
 
 #include <asm/mach/arch.h>
 #include <asm/mach/map.h>
@@ -28,6 +29,7 @@
 #include <asm/irq.h>
 #include <asm/mach-types.h>
 
+#include <linux/platform_data/mmc-s3cmci.h>
 
 #include <plat/devs.h>
 #include <plat/cpu.h>
@@ -69,6 +71,13 @@
 	}
 };
 
+static struct s3c24xx_mci_pdata smdk2410_mmc_cfg __initdata = {
+	.no_wprotect	= 1,
+	.no_detect		= 1,
+	.set_power		= NULL,
+	.ocr_avail		= MMC_VDD_32_33|MMC_VDD_33_34,
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
 	&s3c_device_lcd,
@@ -75,6 +83,7 @@
 	&s3c_device_wdt,
 	&s3c_device_i2c0,
 	&s3c_device_iis,
+	&s3c_device_sdi,
 };
 
 static void __init smdk2410_map_io(void)
@@ -92,6 +101,7 @@
 
 static void __init smdk2410_init(void)
 {
+	s3c24xx_mci_set_platdata(&smdk2410_mmc_cfg);
 	s3c_i2c0_set_platdata(NULL);
 	platform_add_devices(smdk2410_devices, ARRAY_SIZE(smdk2410_devices));
 	smdk_machine_init();
diff -uNr linux-5.1.21.orig/drivers/mmc/host/s3cmci.c linux-5.1.21/drivers/mmc/host/s3cmci.c
--- linux-5.1.21.orig/drivers/mmc/host/s3cmci.c	2019-07-28 06:28:39.000000000 +0000
+++ linux-5.1.21/drivers/mmc/host/s3cmci.c	2026-04-08 14:27:33.512903708 +0000
@@ -1762,7 +1762,7 @@
 	struct mmc_host	*mmc = platform_get_drvdata(pdev);
 	struct s3cmci_host *host = mmc_priv(mmc);
 
-	if (host->irq_cd >= 0)
+	if (host->irq_cd > 0)
 		free_irq(host->irq_cd, host);
 
 	s3cmci_debugfs_remove(host);
```

The last modification for `drivers/mmc/hosts/s3cmci.c` is to remove annoying kernel bug messages.

# Mic & Speaker

This board has UDA1341TS codec chip and built-in ECM microphone & speaker jack. To enable audio feature, I enabled the following items in kernel config first.

```
CONFIG_SND_SOC_SAMSUNG=y
CONFIG_SND_SOC_SAMSUNG_S3C24XX_UDA134X=y
```

I2S driver(`CONFIG_SND_S3C24XX_I2S`) & UDA134X codec driver(`CONFIG_SND_SOC_UDA134X`) are also required, but they are already enabled or auto-enabled by dependency.

Most of modern audio codec chip is controlled by I2C bus. The real audio data is delivered through I2S or PDM, etc., but the control signal to handle or configure codec is also required, and I2C is widely used. But, in my case, UDA1341 uses L3, which is pre-standard proprietary bus. It is another legacy feature for this board.

I need to figure out how to adapt this codec to newer kernel, because since kernel 3.x, L3 bus driver support was removed from official kernel source. Because UDA134X codec support is still in kernel, which means, in one way or another, it may be possible to enable this codec chip, even the bus driver was removed. As I expected, can register this codec to platform device and the patch results as follows.

```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -19,6 +19,7 @@
 #include <linux/serial_s3c.h>
 #include <linux/platform_device.h>
 #include <linux/io.h>
+#include <linux/gpio.h>
 
 #include <asm/mach/arch.h>
 #include <asm/mach/map.h>
@@ -29,6 +30,8 @@
 #include <asm/mach-types.h>
 
 #include <linux/platform_data/i2c-s3c2410.h>
+
+#include <sound/s3c24xx_uda134x.h>
 
 #include <plat/devs.h>
 #include <plat/cpu.h>
@@ -69,6 +72,66 @@
 	}
 };
 
+static void smdk2410_l3_setdat(struct l3_pins *l3, int val)
+{
+    gpio_set_value(l3->gpio_data, val);
+}
+
+static void smdk2410_l3_setclk(struct l3_pins *l3, int val)
+{
+    gpio_set_value(l3->gpio_clk, val);
+}
+
+static void smdk2410_l3_setmode(struct l3_pins *l3, int val)
+{
+    gpio_set_value(l3->gpio_mode, val);
+}
+
+static struct uda134x_platform_data uda134x_data = {
+	.l3 = {
+		.setdat = smdk2410_l3_setdat,
+		.setclk = smdk2410_l3_setclk,
+		.setmode = smdk2410_l3_setmode,
+
+		.gpio_data = S3C2410_GPB(3),
+		.gpio_clk  = S3C2410_GPB(4),
+		.gpio_mode = S3C2410_GPB(2),
+
+		.use_gpios = 1,
+
+		// timing (safe defaults)
+		.data_hold  = 1,
+		.data_setup = 1,
+		.clock_high = 1,
+		.mode_hold  = 1,
+		.mode_setup = 1,
+	},
+	.model = UDA134X_UDA1341,
+};
+
+static struct platform_device s3c_device_codec = {
+    .name = "uda134x-codec",
+    .id   = -1,
+    .dev  = {
+        .platform_data = &uda134x_data,
+    },
+};
+
+static struct s3c24xx_uda134x_platform_data smdk2410_audio_pins = {
+    .l3_clk   = S3C2410_GPB(4),  /* L3 clock GPIO */
+    .l3_mode  = S3C2410_GPB(2),  /* L3 mode GPIO  */
+    .l3_data  = S3C2410_GPB(3),  /* L3 data GPIO  */
+    .model    = UDA134X_UDA1341,
+};
+
+static struct platform_device s3c_device_audio = {
+    .name = "s3c24xx_uda134x",
+    .id   = -1,
+	.dev  = {
+        .platform_data = &smdk2410_audio_pins,
+    },
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
 	&s3c_device_lcd,
@@ -75,6 +138,9 @@
 	&s3c_device_wdt,
 	&s3c_device_i2c0,
 	&s3c_device_iis,
+	&s3c2410_device_dma,
+	&s3c_device_codec,
+	&s3c_device_audio,
 };
 
 static void __init smdk2410_map_io(void)
```

OK. Now `/proc/asound/cards` shows UDA1341 is succefully registered as ALSA card, and inside `/dev/snd`, `PCMC0D0c` & `PCMC0D0p` devices exist, which is audio capture device and playback device each. Further test may be required after userspace ALSA tool prepared, it is good for now.
