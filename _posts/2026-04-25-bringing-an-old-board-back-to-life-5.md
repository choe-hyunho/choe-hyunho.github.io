---
layout: post
title:  "Bringing an old board back to life - 5"
date:   2026-04-25 21:02:55 +0900
categories: embedded system
comments: true
---
Maybe this article will be the last one for registering peripherals to SMDK2410 board. As always, all the related files on [here](https://github.com/choe-hyunho/miscellaneous/tree/main/Board/SMDK2410) and will be constantly updated accordingly.

# LED

I omitted primitive, but useful device. It's LED. SMDK2410 has several LEDs, and some of them is directly connected to hardware signal and no way to control by software. But the others are connected to GPIOs, and can be controlled by kernel. They are already registered as platform devices in `arch/arm/mach-s3c24xx/common-smdk.c`, but not activated. It is because required kernel config item is omitted or defined as module. So, I enabled these items, and some useful LED triggers, too.

```
CONFIG_LEDS_GPIO=y
CONFIG_LEDS_S3C24XX=y
CONFIG_LEDS_TRIGGER_TIMER=y
CONFIG_LEDS_TRIGGER_HEARTBEAT=y
CONFIG_LEDS_TRIGGER_GPIO=y
CONFIG_LEDS_TRIGGER_MTD=y
CONFIG_LEDS_TRIGGER_DEFAULT_ON=y
```

Two default trigger is defined already, one is `timer`, which make LED blinks periodically, and the other is `nand-disk`, which is indicating NAND activity. You can change LED action by changing `/sys/class/leds/led*/trigger`.

# KS24C080 EEPROM (I2C)

There exists small KS24C080 EEPROM in this board. It may be helpful to save device-specific information. Because this EEPROM uses I2C bus, it is quite useful to show how I2C devices are registered.
To support this EEPROM, `CONFIG_EEPROM_AT24` and `CONFIG_I2C_S3C2410` kernel config item is required, which is already enabled in default config. But, to use I2C devices as Linux character devices, extra config is needed as below.
```
CONFIG_I2C_CHARDEV=y
```

And after applying the following patch, EEPROM is recognized successfully.

```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -19,6 +19,7 @@
 #include <linux/serial_s3c.h>
 #include <linux/platform_device.h>
 #include <linux/io.h>
+#include <linux/i2c.h>
 
 #include <asm/mach/arch.h>
 #include <asm/mach/map.h>
@@ -69,6 +70,19 @@
 	}
 };
 
+static const struct property_entry ks24c080_props[] = {
+    PROPERTY_ENTRY_U32("size", 1024),
+    PROPERTY_ENTRY_U32("pagesize", 16),
+    { }
+};
+
+static struct i2c_board_info smdk2410_i2c_devices[] = {
+	{
+		I2C_BOARD_INFO("24c08", 0x50),
+		.properties = ks24c080_props,
+	},
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
 	&s3c_device_lcd,
@@ -92,6 +106,7 @@
 
 static void __init smdk2410_init(void)
 {
+    i2c_register_board_info(0, smdk2410_i2c_devices, ARRAY_SIZE(smdk2410_i2c_devices));
 	s3c_i2c0_set_platdata(NULL);
 	platform_add_devices(smdk2410_devices, ARRAY_SIZE(smdk2410_devices));
 	smdk_machine_init();
```

The test results are as follows. Let's see how to utilize it later.

```
# i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: UU UU UU UU -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         

# hexdump -C /sys/bus/i2c/devices/0-0050/eeprom 
00000000  00 01 02 03 04 05 06 07  08 09 0a 0b 0c 0d 0e 0f  |................|
00000010  10 11 12 13 14 15 16 17  18 19 1a 1b 1c 1d 1e 1f  |................|
00000020  20 21 22 23 24 25 26 27  28 29 2a 2b 2c 2d 2e 2f  | !"#$%&'()*+,-./|
00000030  30 31 32 33 34 35 36 37  38 39 3a 3b 3c 3d 3e 3f  |0123456789:;<=>?|
00000040  40 41 42 43 44 45 46 47  48 49 4a 4b 4c 4d 4e 4f  |@ABCDEFGHIJKLMNO|
00000050  50 51 52 53 54 55 56 57  58 59 5a 5b 5c 5d 5e 5f  |PQRSTUVWXYZ[\]^_|
00000060  60 61 62 63 64 65 66 67  68 69 6a 6b 6c 6d 6e 6f  |`abcdefghijklmno|
00000070  70 71 72 73 74 75 76 77  78 79 7a 7b 7c 7d 7e 7f  |pqrstuvwxyz{|}~.|
00000080  80 81 82 83 84 85 86 87  88 89 8a 8b 8c 8d 8e 8f  |................|
00000090  90 91 92 93 94 95 96 97  98 99 9a 9b 9c 9d 9e 9f  |................|
000000a0  a0 a1 a2 a3 a4 a5 a6 a7  a8 a9 aa ab ac ad ae af  |................|
000000b0  b0 b1 b2 b3 b4 b5 b6 b7  b8 b9 ba bb bc bd be bf  |................|
000000c0  c0 c1 c2 c3 c4 c5 c6 c7  c8 c9 ca cb cc cd ce cf  |................|
000000d0  d0 d1 d2 d3 d4 d5 d6 d7  d8 d9 da db dc dd de df  |................|
000000e0  e0 e1 e2 e3 e4 e5 e6 e7  e8 e9 ea eb ec ed ee ef  |................|
000000f0  f0 f1 f2 f3 f4 f5 f6 f7  f8 f9 fa fb fc fd fe ff  |................|
00000100  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00000400
```

# Keyboard (SPI)

To show enabling SPI bus, I tried to enable keyboard attached to this board. Surprisingly, there is no implementation for this keyboard until now. So, with some help from AI agents, decided to make my own version. Because I already have a Windows CE version driver and UR5HCSSPI chip datasheet, feed them to AI and get a rough code from it. Then spend some time to debugging, and now I get a working version of this keyboard driver!

First, add kernel config item for SPI & my own keyboard driver.

```
CONFIG_SPI_S3C24XX=y
CONFIG_KEYBOARD_UR5HCSPI=y
```

And next, add SPI bus to platform devices as below.

```C
--- linux-5.1.21.orig/arch/arm/mach-s3c24xx/mach-smdk2410.c
+++ linux-5.1.21/arch/arm/mach-s3c24xx/mach-smdk2410.c
@@ -19,6 +19,8 @@
 #include <linux/serial_s3c.h>
 #include <linux/platform_device.h>
 #include <linux/io.h>
+#include <linux/spi/spi.h>
+#include <linux/spi/s3c24xx.h>
 
 #include <asm/mach/arch.h>
 #include <asm/mach/map.h>
@@ -69,11 +71,58 @@
 	}
 };
 
+static void smdk2410_spi0_cs(struct s3c2410_spi_info *spi, int cs, int pol)
+{
+    gpio_set_value(spi->pin_cs, pol);
+}
+
+static struct s3c2410_spi_info smdk2410_spi0_data = {
+    .pin_cs   = S3C2410_GPG(2),
+    .num_cs   = 1,
+    .bus_num  = 0,
+    .set_cs   = smdk2410_spi0_cs,
+};
+
+static struct spi_board_info smdk2410_spi0_devices[] = {
+	{
+        .modalias       = "dummy",
+        .max_speed_hz   = 1000000,
+        .bus_num        = 0,
+        .chip_select    = 0,
+        .mode           = SPI_MODE_0,
+    },
+};
+
+static void smdk2410_spi1_cs(struct s3c2410_spi_info *spi, int cs, int pol)
+{
+    gpio_set_value(spi->pin_cs, pol);
+}
+
+static struct s3c2410_spi_info smdk2410_spi1_data = {
+    .pin_cs   = S3C2410_GPB(6),
+    .num_cs   = 1,
+    .bus_num  = 1,
+    .set_cs   = smdk2410_spi1_cs,
+};
+
+static struct spi_board_info smdk2410_spi1_devices[] = {
+    {
+        .modalias       = "ur5hcspi",
+        .max_speed_hz   = 500000,
+        .bus_num        = 1,
+        .chip_select    = 0,
+        .mode           = SPI_MODE_1,
+		.irq            = IRQ_EINT1,
+    },
+};
+
 static struct platform_device *smdk2410_devices[] __initdata = {
 	&s3c_device_ohci,
 	&s3c_device_lcd,
 	&s3c_device_wdt,
 	&s3c_device_i2c0,
+	&s3c_device_spi0,
+	&s3c_device_spi1,
 	&s3c_device_iis,
 };
 
 static void __init smdk2410_map_io(void)
@@ -93,6 +142,19 @@
 static void __init smdk2410_init(void)
 {
 	s3c_i2c0_set_platdata(NULL);
+	gpio_request(S3C2410_GPG(2), "spi0-cs");
+	gpio_direction_output(S3C2410_GPG(2), 1);
+	s3c_set_platdata(&smdk2410_spi0_data, sizeof(struct s3c2410_spi_info), &s3c_device_spi0);
+	spi_register_board_info(smdk2410_spi0_devices, ARRAY_SIZE(smdk2410_spi0_devices));
+	gpio_request(S3C2410_GPB(0), "kbd-pwr");
+	gpio_direction_output(S3C2410_GPB(0), 0);
+	gpio_request(S3C2410_GPB(6), "spi1-cs");
+	gpio_direction_output(S3C2410_GPB(6), 1);
+	s3c_gpio_cfgpin(S3C2410_GPF(1), S3C2410_GPF1_EINT1);
+	s3c_gpio_setpull(S3C2410_GPF(1), S3C_GPIO_PULL_UP);
+	irq_set_irq_type(IRQ_EINT1, IRQ_TYPE_LEVEL_LOW);
+	s3c_set_platdata(&smdk2410_spi1_data, sizeof(struct s3c2410_spi_info), &s3c_device_spi1);
+	spi_register_board_info(smdk2410_spi1_devices, ARRAY_SIZE(smdk2410_spi1_devices));
 	platform_add_devices(smdk2410_devices, ARRAY_SIZE(smdk2410_devices));
 	smdk_machine_init();
 }
```

Now, I have registered SPI bus and peripherals to platform, but need some extra patchs to remove some glitches and nasty messages as below.

```C
--- linux-5.1.21.orig/drivers/clk/samsung/clk-s3c2410.c
+++ linux-5.1.21/drivers/clk/samsung/clk-s3c2410.c
@@ -96,6 +96,8 @@
 
 /* should be added _after_ the soc-specific clocks are created */
 static struct samsung_clock_alias s3c2410_common_aliases[] __initdata = {
+    ALIAS(PCLK_SPI, "s3c2410-spi.0", "spi"),
+    ALIAS(PCLK_SPI, "s3c2410-spi.1", "spi"),
 	ALIAS(PCLK_I2C, "s3c2410-i2c.0", "i2c"),
 	ALIAS(PCLK_ADC, NULL, "adc"),
 	ALIAS(PCLK_RTC, NULL, "rtc"),
--- linux-5.1.21.orig/drivers/spi/spi-s3c24xx.c
+++ linux-5.1.21/drivers/spi/spi-s3c24xx.c
@@ -182,9 +182,7 @@
 
 	/* allocate settings on the first call */
 	if (!cs) {
-		cs = devm_kzalloc(&spi->dev,
-				  sizeof(struct s3c24xx_spi_devstate),
-				  GFP_KERNEL);
+		cs = kzalloc(sizeof(struct s3c24xx_spi_devstate), GFP_KERNEL);
 		if (!cs)
 			return -ENOMEM;
 
@@ -208,6 +206,12 @@
 	return 0;
 }
 
+static void s3c24xx_spi_cleanup(struct spi_device *spi)
+{
+	kfree(spi->controller_state);
+	spi->controller_state = NULL;
+}
+
 static inline unsigned int hw_txbyte(struct s3c24xx_spi *hw, int count)
 {
 	return hw->tx ? hw->tx[count] : 0;
@@ -469,7 +473,7 @@
 {
 	/* for the moment, permanently enable the clock */
 
-	clk_enable(hw->clk);
+	clk_prepare_enable(hw->clk);
 
 	/* program defaults into the registers */
 
@@ -536,6 +540,7 @@
 	hw->bitbang.txrx_bufs      = s3c24xx_spi_txrx;
 
 	hw->master->setup  = s3c24xx_spi_setup;
+	hw->master->cleanup = s3c24xx_spi_cleanup;
 
 	dev_dbg(hw->dev, "bitbang at %p\n", &hw->bitbang);
 
@@ -602,7 +607,7 @@
 	return 0;
 
  err_register:
-	clk_disable(hw->clk);
+	clk_disable_unprepare(hw->clk);
 
  err_no_pdata:
 	spi_master_put(hw->master);
@@ -614,7 +619,7 @@
 	struct s3c24xx_spi *hw = platform_get_drvdata(dev);
 
 	spi_bitbang_stop(&hw->bitbang);
-	clk_disable(hw->clk);
+	clk_disable_unprepare(hw->clk);
 	spi_master_put(hw->master);
 	return 0;
 }
@@ -634,7 +639,7 @@
 	if (hw->pdata && hw->pdata->gpio_setup)
 		hw->pdata->gpio_setup(hw->pdata, 0);
 
-	clk_disable(hw->clk);
+	clk_disable_unprepare(hw->clk);
 	return 0;
 }
 
```

Finally, patch `Kconfig` & `Makefile` in `drivers/input/keyboard`, and added my new `ur4hcspi.c` in that directory. Though I made this in a little bit hurry, but it seems to work quite well, at first glance.

```C
--- linux-5.1.21.orig/drivers/input/keyboard/Kconfig
+++ linux-5.1.21/drivers/input/keyboard/Kconfig
@@ -756,4 +756,10 @@
 	  To compile this driver as a module, choose M here: the
 	  module will be called pmic-keys.
 
+config KEYBOARD_UR5HCSPI
+    tristate "UR5HCSPI keyboard"
+    depends on SPI
+    help
+      Support for UR5HCSPI keyboard controller
+
 endif
--- linux-5.1.21.orig/drivers/input/keyboard/Makefile
+++ linux-5.1.21/drivers/input/keyboard/Makefile
@@ -67,3 +67,4 @@
 obj-$(CONFIG_KEYBOARD_TWL4030)		+= twl4030_keypad.o
 obj-$(CONFIG_KEYBOARD_XTKBD)		+= xtkbd.o
 obj-$(CONFIG_KEYBOARD_W90P910)		+= w90p910_keypad.o
+obj-$(CONFIG_KEYBOARD_UR5HCSPI)		+= ur5hcspi.o
```
```C
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/spi/spi.h>
#include <linux/input.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/slab.h>
#include <linux/delay.h>

#ifndef KEY_FN
#define KEY_FN  0x1d0
#endif

/* ===== Protocol constants from datasheet ===== */
#define UR5HCSPI_CTRL           0x80   /* Control prefix: chip->host */
#define UR5HCSPI_ESC            0x1B   /* ESC prefix: host->chip    */

/* Commands chip->host */
#define UR5HCSPI_CMD_INIT_REQ		0xA0
#define UR5HCSPI_CMD_INIT_DONE		0xA1
#define UR5HCSPI_CMD_HEARTBEAT		0xA2
#define UR5HCSPI_CMD_IDENTIFY		0xF2
#define UR5HCSPI_CMD_LED_STATUS		0xA3
#define UR5HCSPI_CMD_RESEND			0xA5
#define UR5HCSPI_CMD_IOMODE_STATUS	0xA7
#define UR5HCSPI_CMD_OUTPUT_DATA	0xA8

/* Commands host->chip */
#define UR5HCSPI_HOST_INIT      	0xA0   /* Reset chip to power-on state */
#define UR5HCSPI_HOST_INIT_DONE 	0xA1   /* Enable keyboard transmission  */
#define UR5HCSPI_HOST_HEARTBEAT 	0xA2
#define UR5HCSPI_HOST_IDENTIFY		0xF2
#define UR5HCSPI_HOST_LED_STATUS	0xA3
#define UR5HCSPI_HOST_LED_MODIFY	0xA6
#define UR5HCSPI_HOST_RESEND		0xA5
#define UR5HCSPI_HOST_IOMODE_MODIFY	0xA7
#define UR5HCSPI_HOST_OUTPUT_DATA	0xA8
#define UR5HCSPI_HOST_WAKEUP_KEY	0xA9

/* Max key code: col*8 + row + 1, max col=13 row=7 -> 13*8+7+1 = 112 */
#define UR5HCSPI_MAX_KEYCODE    0x73
/* Switch codes */
#define UR5HCSPI_CODE_XSW       0x71
#define UR5HCSPI_CODE_SW0       0x72
#define UR5HCSPI_CODE_GIO0      0x73

/* SPI max clock per datasheet */
#define UR5HCSPI_MAX_SPEED_HZ   500000

struct ur5hcspi {
	struct spi_device *spi;
	struct input_dev  *input;
	struct work_struct work;
	int fn_pressed;
	bool numlock_on;
};

/* ===== LRC calculation per datasheet page 13/14 ===== */
static u8 ur5hcspi_lrc(const u8 *buf, size_t len)
{
	u8 lrc = 0;
	size_t i;
	for (i = 0; i < len; i++)
		lrc ^= buf[i];
	if (lrc & 0x80)
		lrc ^= 0xC0;
	return lrc;
}

static int ur5hcspi_write_byte(struct ur5hcspi *ur, u8 val)
{
	struct spi_transfer t = {
		.tx_buf = &val,
		.len = 1,
	};
	struct spi_message m;

	spi_message_init(&m);
	spi_message_add_tail(&t, &m);

	return spi_sync(ur->spi, &m);
}

/* ===== Read one byte from chip (per-byte _SS toggle) ===== */
static int ur5hcspi_read_byte(struct ur5hcspi *ur, u8 *val)
{
	u8 tx = 0xFF;
	struct spi_transfer t = {
		.tx_buf = &tx,
		.rx_buf = val,
		.len = 1,
	};
	struct spi_message m;

	spi_message_init(&m);
	spi_message_add_tail(&t, &m);

	return spi_sync(ur->spi, &m);
}

/* ===== Send one framed command to chip ===== */
/* Format: <ESC=1B> <CMD> <LRC> */
static int ur5hcspi_send_cmd(struct ur5hcspi *ur, u8 cmd)
{
	u8 pkt[3];

	pkt[0] = UR5HCSPI_ESC;
	pkt[1] = cmd;
	pkt[2] = ur5hcspi_lrc(pkt, 2);

	ur5hcspi_write_byte(ur, pkt[0]);
	ur5hcspi_write_byte(ur, pkt[1]);
	ur5hcspi_write_byte(ur, pkt[2]);

	return 0;
}

/* ===== Keymap (Make codes 0x01–0x70 from datasheet p.9) ===== */
static const unsigned short ur5hcspi_keymap[0x74] = {
	[0x01] = KEY_LEFTALT,
	[0x09] = KEY_GRAVE,
	[0x0A] = KEY_BACKSLASH,
	[0x0B] = KEY_TAB,
	[0x0C] = KEY_Z,
	[0x0D] = KEY_A,
	[0x0E] = KEY_X,
	[0x12] = KEY_LEFTSHIFT,
	[0x19] = KEY_LEFTCTRL,
	[0x21] = KEY_FN,
	[0x29] = KEY_ESC,
	[0x2A] = KEY_DELETE,
	[0x2B] = KEY_Q,
	[0x2C] = KEY_CAPSLOCK,
	[0x2D] = KEY_S,
	[0x2E] = KEY_C,
	[0x2F] = KEY_3,
	[0x31] = KEY_1,
	[0x33] = KEY_W,
	[0x35] = KEY_D,
	[0x36] = KEY_V,
	[0x37] = KEY_4,
	[0x39] = KEY_2,
	[0x3A] = KEY_T,
	[0x3B] = KEY_E,
	[0x3D] = KEY_F,
	[0x3E] = KEY_B,
	[0x3F] = KEY_5,
	[0x41] = KEY_9,
	[0x42] = KEY_Y,
	[0x43] = KEY_R,
	[0x44] = KEY_K,
	[0x45] = KEY_G,
	[0x46] = KEY_N,
	[0x47] = KEY_6,
	[0x49] = KEY_0,
	[0x4A] = KEY_U,
	[0x4B] = KEY_O,
	[0x4C] = KEY_L,
	[0x4D] = KEY_H,
	[0x4E] = KEY_M,
	[0x4F] = KEY_7,
	[0x51] = KEY_MINUS,
	[0x52] = KEY_I,
	[0x53] = KEY_P,
	[0x54] = KEY_SEMICOLON,
	[0x55] = KEY_J,
	[0x56] = KEY_COMMA,
	[0x57] = KEY_8,
	[0x59] = KEY_EQUAL,
	[0x5A] = KEY_ENTER,
	[0x5B] = KEY_LEFTBRACE,
	[0x5C] = KEY_APOSTROPHE,
	[0x5D] = KEY_SLASH,
	[0x5E] = KEY_DOT,
	[0x5F] = KEY_RIGHTMETA,
	[0x62] = KEY_RIGHTSHIFT,
	[0x69] = KEY_BACKSPACE,
	[0x6A] = KEY_DOWN,
	[0x6B] = KEY_RIGHTBRACE,
	[0x6C] = KEY_UP,
	[0x6D] = KEY_LEFT,
	[0x6E] = KEY_SPACE,
	[0x6F] = KEY_RIGHT,
	/* Switch codes */
	[0x71] = KEY_POWER,    /* XSW */
	[0x72] = KEY_SLEEP,    /* SW0 */
};

struct vk_map {
	int from;
	int to;
};

static const struct vk_map fn_map[] = {
	{ KEY_1, KEY_F1 },
	{ KEY_2, KEY_F2 },
	{ KEY_3, KEY_F3 },
	{ KEY_4, KEY_F4 },
	{ KEY_5, KEY_F5 },
	{ KEY_6, KEY_F6 },
	{ KEY_7, KEY_F7 },
	{ KEY_8, KEY_F8 },
	{ KEY_9, KEY_F9 },
	{ KEY_0, KEY_F10 },

	{ KEY_MINUS, KEY_NUMLOCK },
	{ KEY_EQUAL, KEY_CANCEL },

	{ KEY_P, KEY_INSERT },
	{ KEY_LEFTBRACE, KEY_PAUSE },
	{ KEY_RIGHTBRACE, KEY_SCROLLLOCK },

	{ KEY_SEMICOLON, KEY_SYSRQ },
	{ KEY_APOSTROPHE, KEY_SYSRQ },

	{ KEY_LEFT, KEY_HOME },
	{ KEY_UP, KEY_PAGEUP },
	{ KEY_DOWN, KEY_PAGEDOWN },
	{ KEY_RIGHT, KEY_END },
};

static const struct vk_map numlock_map[] = {
	{ KEY_7, KEY_KP7 },
	{ KEY_8, KEY_KP8 },
	{ KEY_9, KEY_KP9 },
	{ KEY_0, KEY_KPASTERISK },

	{ KEY_U, KEY_KP4 },
	{ KEY_I, KEY_KP5 },
	{ KEY_O, KEY_KP6 },
	{ KEY_P, KEY_KPMINUS },

	{ KEY_J, KEY_KP1 },
	{ KEY_K, KEY_KP2 },
	{ KEY_L, KEY_KP3 },
	{ KEY_SEMICOLON, KEY_KPPLUS },

	{ KEY_M, KEY_KP0 },
	{ KEY_DOT, KEY_KPDOT },
	{ KEY_SLASH, KEY_KPSLASH },
};

static int find_remap(int key, const struct vk_map *map, int size)
{
	int i;
	for (i = 0; i < size; i++) {
		if (map[i].from == key)
			return map[i].to;
	}
	return key;
}

static void ur5hcspi_process_packet(struct ur5hcspi *ur)
{
	u8 byte, data, lrc_got, lrc_calc;
	int ret, pressed, code, key;

	/* Read first byte */
	ret = ur5hcspi_read_byte(ur, &byte);
	if (ret < 0) {
		dev_err(&ur->spi->dev, "read first byte failed\n");
		return;
	}

	/* ---------------------------------------------------- */
	/* Case 1: CONTROL PACKET (0x80 + data + LRC)           */
	/* ---------------------------------------------------- */
	if (byte == UR5HCSPI_CTRL) {
		/* Second byte */
		ret = ur5hcspi_read_byte(ur, &data);
		if (ret < 0) {
			dev_err(&ur->spi->dev, "missing control data byte\n");
			return;
		}

		/* Third byte (LRC) */
		ret = ur5hcspi_read_byte(ur, &lrc_got);
		if (ret < 0) {
			dev_err(&ur->spi->dev, "missing LRC byte\n");
			return;
		}

		/* Verify LRC */
		{
			u8 tmp[2] = { UR5HCSPI_CTRL, data };
			lrc_calc = ur5hcspi_lrc(tmp, 2);
		}

		if (lrc_got != lrc_calc) {
			dev_warn(&ur->spi->dev, "LRC mismatch: got 0x%02x expected 0x%02x\n", lrc_got, lrc_calc);
			/* Request resend */
			ur5hcspi_send_cmd(ur, UR5HCSPI_HOST_RESEND);
			return;
		}

		/* Handle control responses */
		switch (data) {
		case UR5HCSPI_CMD_INIT_REQ:
			dev_info(&ur->spi->dev, "chip requested re-init\n");
			ur5hcspi_send_cmd(ur, UR5HCSPI_HOST_INIT_DONE);
			break;

		case UR5HCSPI_CMD_INIT_DONE:
		case UR5HCSPI_CMD_HEARTBEAT:
			dev_info(&ur->spi->dev, "control response: 0x%02x\n", data);
			break;

		default:
			dev_err(&ur->spi->dev, "unknown control: 0x%02x\n", data);
			break;
		}

		return;
	}

	/* ---------------------------------------------------- */
	/* Case 2: NORMAL KEYCODE (1 byte stream mode)           */
	/* ---------------------------------------------------- */

	data = byte;

	pressed = !(data & 0x80);   /* bit7=0 → press */
	code    =  (data & 0x7F);

	if (code == 0 || code > UR5HCSPI_MAX_KEYCODE) {
		dev_err(&ur->spi->dev, "unknown keycode: 0x%02x\n", code);
		return;
	}

	key = ur5hcspi_keymap[code];
	if (!key) {
		dev_err(&ur->spi->dev, "unmapped keycode: 0x%02x\n", code);
		return;
	}

	/* FN key handling */
	if (key == KEY_FN) {
		ur->fn_pressed = pressed;
		return;
	}

	/* Track NumLock (toggle on press) */
	if (key == KEY_NUMLOCK && pressed) {
		ur->numlock_on = !ur->numlock_on;
	}

	/* Apply remap */
	if (ur->fn_pressed) {
		if (ur->numlock_on)
			key = find_remap(key, numlock_map, ARRAY_SIZE(numlock_map));

		key = find_remap(key, fn_map, ARRAY_SIZE(fn_map));
	}

	input_report_key(ur->input, key, pressed);
	input_sync(ur->input);
}

static void ur5hcspi_work(struct work_struct *work)
{
	struct ur5hcspi *ur = container_of(work, struct ur5hcspi, work);
	ur5hcspi_process_packet(ur);
}

static irqreturn_t ur5hcspi_irq(int irq, void *dev_id)
{
	struct ur5hcspi *ur = dev_id;
	schedule_work(&ur->work);
	return IRQ_HANDLED;
}

static int ur5hcspi_probe(struct spi_device *spi)
{
	struct ur5hcspi *ur;
	int ret, i;

	ur = devm_kzalloc(&spi->dev, sizeof(*ur), GFP_KERNEL);
	if (!ur) {
		dev_err(&spi->dev, "devm_kzalloc failed\n");
		return -ENOMEM;
	}

	ur->spi = spi;
	spi_set_drvdata(spi, ur);

	/* SPI setup — CPOL=0 CPHA=0, max 500KHz per datasheet */
	spi->mode           = SPI_MODE_1;
	spi->bits_per_word  = 8;
	spi->max_speed_hz   = UR5HCSPI_MAX_SPEED_HZ;  /* ← critical */
	ret = spi_setup(spi);
	if (ret) {
		dev_err(&spi->dev, "spi_setup failed: %d\n", ret);
		return ret;
	}

	/* Input device */
	ur->input = devm_input_allocate_device(&spi->dev);
	if (!ur->input) {
		dev_err(&spi->dev, "input allocate failed\n");
		return -ENOMEM;
	}

	ur->input->name = "UR5HCSPI Keyboard";
	ur->input->id.bustype = BUS_SPI;

	__set_bit(EV_KEY, ur->input->evbit);

	for (i = 0; i < ARRAY_SIZE(ur5hcspi_keymap); i++) {
		if (ur5hcspi_keymap[i])
			__set_bit(ur5hcspi_keymap[i], ur->input->keybit);
	}

	ret = input_register_device(ur->input);
	if (ret) {
		dev_err(&spi->dev, "input_register_device failed ret=%d\n", ret);
		return ret;
	}

	INIT_WORK(&ur->work, ur5hcspi_work);

	/* IRQ on _ATN falling edge */
	if (spi->irq) {
		ret = devm_request_irq(&spi->dev, spi->irq,
							   ur5hcspi_irq,
							   IRQF_TRIGGER_FALLING,
							   "ur5hcspi", ur);
		if (ret) {
			dev_err(&spi->dev, "request_irq failed: %d\n", ret);
			return ret;
		}
	}

	/*
	 * Send "Initialize" to clears all buffers and returns to the power-on state.
	 * Per datasheet p.15: <ESC=1B> <A0> <LRC=7B>
	 */
	ret = ur5hcspi_send_cmd(ur, UR5HCSPI_HOST_INIT);
	if (ret)
		dev_warn(&spi->dev, "init command failed: %d\n", ret);

	dev_info(&spi->dev, "UR5HCSPI keyboard initialized\n");
	return 0;
}

static int ur5hcspi_remove(struct spi_device *spi)
{
	struct ur5hcspi *ur = spi_get_drvdata(spi);
	cancel_work_sync(&ur->work);
	return 0;
}

static const struct spi_device_id ur5hcspi_ids[] = {
	{ "ur5hcspi", 0 },
	{ }
};
MODULE_DEVICE_TABLE(spi, ur5hcspi_ids);

static struct spi_driver ur5hcspi_driver = {
	.driver = {
		.name = "ur5hcspi",
	},
	.probe = ur5hcspi_probe,
	.remove = ur5hcspi_remove,
	.id_table = ur5hcspi_ids,
};

module_spi_driver(ur5hcspi_driver);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("UR5HCSPI Keyboard Driver");
```