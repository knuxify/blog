---
layout: post
author: knuxify
title: (Mis)adventures in getting a working display on mainline Linux
description: or, how I spent nearly 3 weeks trying to figure out why I wasn't getting an image on my Galaxy Tab 3 8.0.
image: /images/tab3-display/tablet.jpg
tags: linux mainline exynos mipi display
excerpt_separator: <!--more-->
---

A while back, I tried to mainline my [Samsung Galaxy Tab 3 8.0 (lt01)](https://wiki.postmarketos.org/wiki/Samsung_Galaxy_Tab_3_8.0_(SM-T310)_(samsung-lt01wifi)). It's a tablet from 2013 with the Exynos 4212 chipset - the dual-core variant of the already well-supported Exynos 4412. When support for the Exynos 4 mainline kernel was added to postmarketOS, I figured it was the perfect time to try and get it running on mainline.

The initial task was re-introducing support for the Exynos 4212, as [it was dropped a few years back](https://github.com/torvalds/linux/commit/bca9085e0ae93253bc93ce218c85ac7d7e7f1831). This was fairly straightforward - just moving some things into a separate .dtsi and modifying some drivers was enough to get it booting. Unfortunately, I hit a pretty big hurdle very quickly - **the moment I booted into mainline, the display would completely shut off.**

<!--more-->

I wanted to figure this out quick, because the kernel moves fast, and I didn't feel like rebasing the Exynos 4212 patches forever. But I wasn't just going to try to merge a tablet without a working screeen into mainline!

And so began a multiple week journey, as I tried to figure out the entire problem one step at a time.

## Overview

The average phone/tablet screen comprises of three "layers":

* **The touchscreen/digitizer**, which handles touch inputs; sometimes this also includes touchkeys (menu/back buttons next to the home button, from *ye old times* when phones still had physical buttons);
* **The display panel/LCD**, which is what displays the image;
* **The backlight**, which controls the panel's brightness.

In some cases, the display panel handles the backlight itself. However, in the case of the Galaxy Tab 3 8.0, they are handled separately. The Galaxy Tab 3 8.0 uses an **LSL080AL02** panel[^1] managed by the **S6D7AA0** IC, and a separate backlight controller - the **TI LP8556TMX**.

The panel isn't supported in mainline, but the MSM8916 kernel fork has a driver for [a very similar panel](https://github.com/msm8916-mainline/linux/blob/msm8916/6.2-rc5/drivers/gpu/drm/panel/msm8916-generated/panel-samsung-s6d7aa0-lsl080al03.c), used in the [Samsung Galaxy Tab A 8.0 2015](https://wiki.postmarketos.org/wiki/Samsung_Galaxy_Tab_A_8.0_2015_(samsung-gt58)), which I used as a base for my driver. The LP8556 chip, however, is supported as part of the [lp855x driver](https://github.com/torvalds/linux/blob/master/drivers/video/backlight/lp855x_bl.c).

After adding the driver for the backlight controller into the DTSI, however, nothing seemed to change. Adding the panel driver didn't seem to fix anything either (but we'll talk about that later). I also thought that maybe the touchscreen or the HDMI output/MHL are related, but enabling the drivers for these didn't change anything either.

So, the next logical step was to open up the device and see if we can spot anything obvious in there.

A quick look at the service manual reveals some helpful hints for figuring out why the display isn't working. It also shows the schematics for two chips and the display connector:

* The chip marked as U602 is the LP8556 chip. It outputs a 19V signal going directly to the connector. It's connected to two supplies: V_BATT (an always-on 5v line) for the VDD, and a line managed by the LED_BACKLIGHT_RESET GPIO pin for the EN/VDDIO.
* The chip marked as U600 seems to be some kind of signal booster as well, outputing a 3.3v signal that goes to the display connector. It's enabled by setting the LCD_EN pin signal to high.
* Finally, the connector itself - this has a few standard MIPI connection lines, a connection to the 19V signal from the LP8556, the 3.3V signal from the booster and VMIPI_1.8V from the PMIC. It's also connected to an interrupt line, and the 3 LED output pins from the LP8556 chip are connected to it as well.

I went through the troubleshooting steps and quickly found the culprit - the 19V line from the LP8556 chip was, in fact, at **14.35V**. I verified that the correct voltage, as seen when booting into the recovery (downstream kernel) was about **19.32V**. Nearly 5V missing - strange...

## Figuring out the backlight

But wait - 3 LED output pins? 19V signal? What is any of this about? Ah, if only we had the datasheet for the LP8556...

...well, turns out, [we do](https://www.ti.com/lit/ds/symlink/lp8556.pdf)! TI was kind enough to make the datasheet for this chip publically available.

<!--![LP8556 schematic from the datasheet.](/blog/images/tab3-display/lp8556.png)-->
{% include image url="/blog/images/tab3-display/lp8556.png" description="LP8556 schematic from the datasheet." %}

The LP8556 is described as a "High-Efficiency LED Backlight Driver For Tablets". The brightness is controlled by a PWM signal from 25 to 75 kHz. It also has various configuration options which can be set through an I2C interface.

The pin functions are explained on page 5 of the datasheet. The important parts are as follows:

* **VDD** is the power supply.
* **EN/VDDIO** is both the chip enable and the power supply reference for the PWM, SDA and SCL pins.
* **PWM** is the input PWM signal.
* Pins **LED1** through **LED6** are **LED drivers**.
* **VBOOST** is the boosted output voltage.

In the downstream kernel, the parameters for the backlight are defined at the very end of the `arch/arm/mach-exynos/board-lcd-mipi.c` file.

```c
static struct lp855x_platform_data lp8856_bl_pdata = {
	.mode		= PWM_BASED,
	.device_control	= DEVICE_CONTROL_VAL,
	.initial_brightness = 120,
	.load_new_rom_data = 1,
	.size_program	= ARRAY_SIZE(lp8556_eprom_arr),
	.rom_data	= lp8556_eprom_arr,
	.use_gpio_en	= 1,
	.gpio_en	= GPIO_LED_BACKLIGHT_RESET,
	.power_on_udelay = 1000,
};
```

There are two structs for the I2C configuration data: one for 4 drivers, and one for 3 drivers.

```c
static struct lp855x_rom_data lp8556_eprom_arr[] = {
	{EPROM_CFG3_ADDR, 0x5E, 0x00},
   ...
};

static struct lp855x_rom_data lp8556_eprom_3drv_arr[] = {
	{EPROM_CFG3_ADDR, 0x5E, 0x00},
   ...
};
```

As we've already estabilished by looking at the service manual, we only use 3 drivers. This is also confirmed by the check in `lcd_bl_init`, which checks for the lcdtype variable:

```c
static int lcd_bl_init(void)
{
#ifdef CONFIG_FB_S5P_S6D7AA0
	if (0x800 == (lcdtype & 0xff00)) {
		lp8856_bl_pdata.rom_data = lp8556_eprom_3drv_arr;
		lp8856_bl_pdata.size_program = ARRAY_SIZE(lp8556_eprom_3drv_arr);
	}
#endif

   ...
}
```

`lcdtype` is taken from the cmdline parameters, and on my tablet it was set to `2049`. Sure enough, the if condition is met, and the 3 driver configuration options are selected.

Three other things to take note of here are:

* `mode` in the platform data takes two options: `REGISTER_BASED` or `PWM_BASED`. Here, it's set to `PWM_BASED`.
* `device_control` is a device-specific value. Ours is defined in the driver as `(PWM_CONFIG(LP8556) | 0x80)`, which ends up evaluating as 0x80. (`PWM_CONFIG(LP8556)` evaluates to `LP8556_PWM_CONFIG` which in turn evaluates to `LP8556_PWM_ONLY` from the `lp8556_brightness_source` enum, and equals 0.)
* The `gpio_en` value is, indeed, set to the LED_BACKLIGHT_RESET GPIO.

This seems like enough information to give to the mainline driver, right? Well, not quite.

See, to actually be able to enable PWM mode in the mainline driver, we needed two important pieces of information:

* The PWM device to use. This was simple enough to figure out - `GPIO_LED_BACKLIGHT_PWM` in `arch/arm/mach-exynos/include/mach/gpio-rev00-tab3.h` is defined as `GPD0(1)`, which corresponds to PWM 1.
* The PWM period. This one, however, wasn't anywhere near the lp855x driver.

I went through quite a lot of debugging to figure this part out. Here are some of the more interesting discoveries I made along the way:

* In downstream, two backlight devices appeared under `/sys/class`. One was the `lp855x` device, but it seemed to do nothing... the other was an `mdnie` device, that actually changed the brightness...? Searching for the source was inconslusive, but I ended up finding `drivers/video/samsung/mdnie.c`, which is the driver for mDNIe - "Mobile Digital Natural Image Engine", a Samsung-proprietary color correction system.
* At some point while digging through the kernel for clues, I found `arch/arm/plat-samsung/dev-backlight.c`, which seemed to contain the PWM period value I was looking for - `78770`.
* I copied the period and set up PWM just like the mainline `lp855x` driver docs told me to... but I still got nothing. Turns out, the driver was broken, and PWM mode didn't work at all!

I figured out that I could export out the PWM pin using `echo "1" > /sys/class/pwm/export`, then set the duty cycle and period values manually following the lp855x driver code that *should* have worked. And indeed, I saw the voltage rise from 14.25V to about 16V. Not yet enough, but we were getting close!

Since I now knew that PWM controlled the backlight, and we didn't need the `lp855x` driver for it, I decided to use the generic `pwm-backlight` driver instead. I copied the pwm-backlight node using the generic pwm-backlight driver from another Exynos device, the Galaxy Note 10.1 (p4note):

```c
	backlight: backlight {
		compatible = "pwm-backlight";
		pinctrl-0 = <&backlight_reset>;
		pinctrl-names = "default";
		enable-gpios = <&gpm0 1 GPIO_ACTIVE_HIGH>; /* BACKLIGHT_RESET pin */
		pwms = <&pwm 1 78770 0>;
		brightness-levels = <0 48 128 255>;
		num-interpolated-steps = <8>;
		default-brightness-level = <12>;
	};
```

And sure enough, it worked, and the backlight was now turning on!

## Figuring out the panel connection

Of course, getting the backlight working was *half* of the success. The other half was getting anything to display on the screen at all. And currently, that did not seem to be happening.

The display logo would look like there was a grid overlayed on it, and would fade out. Occasionally, there would be a colored line somewhere on the screen, and would fade out equally as quickly. But what was the reason?

### FIMD and DSI

While trying to figure this out, I wrote an e-mail to [the guy who worked on the p4note](https://viciouss.github.io), and asked if he had any similar issues when working on the display. He replied with some fairly interesting advice, which I was able to combine with what I saw in the p4note dtsi.

Unfortunately, it turns out that there are **two** ways to attach a display, depending on the way it's controlled: directly through the FIMD (usually for LVDS panels), or through the DSI master/DSIM (for, who would've guessed, DSI panels). The Note 10.1 used the FIMD method, and the Tab 3 8.0 used DSI. Thus, the advice wasn't all that useful in my case. But let's start at the beginning.

In the case of panels connected directly to **FIMD** (Fully Interactive Mobile Display), the initialization looks a little something like this:

The panel is dunked somewhere into the `/` node...

```c
/ {
	// [...]
	panel {
		compatible = "samsung,ltl101al01";
		pinctrl-0 = <&lvds_nshdn>;
		pinctrl-names = "default";
		power-supply = <&panel_vdd>;
		enable-gpios = <&gpm0 5 GPIO_ACTIVE_HIGH>;
		backlight = <&backlight>;

		port {
			lcd_ep: endpoint {
				remote-endpoint = <&fimd_ep>;
			};
		};
	};
};
```

The `port` node is crucial here - it acts as a way to tell the FIMD node, located a bit lower, about the connected display. Here's what the FIMD node looks like:

```c
&fimd {
	pinctrl-0 = <&lcd_clk &lcd_data24>;
	pinctrl-names = "default";
	#address-cells = <1>;
	#size-cells = <0>;
	status = "okay";

	samsung,invert-vclk;

	port@3 {
		reg = <3>;

		fimd_ep: endpoint {
			remote-endpoint = <&lcd_ep>;
		};
	};
};
```

If in doubt, [read the manual](https://github.com/torvalds/linux/blob/v6.2/Documentation/devicetree/bindings/display/samsung/samsung%2Cfimd.yaml). The documentation also explains some options I haven't mentioned here (such as `samsung,invert-vclk`).

> **Note**: An interesting thing of note here is the `pinctrl` setup. LVDS pannels connected directly to the FIMD will usually be connected through the LCD data pins. You can check this in downstream - check the `gpio` definitions file for your device, and see if the comments mention any LCD data pins of any kind. Hint - the pins you're looking for are located in the pinctrl definitions in `exynos4x12-pinctrl.dtsi` in mainline.

As for **DSI** panels - the [documentation](https://github.com/torvalds/linux/blob/v6.2/Documentation/devicetree/bindings/display/exynos/exynos_dsim.txt) for DSIM also mentions a port-based setup, but from what I've seen by grepping around the kernel, no DTSIs use it. Instead, the panel is simply dropped directly into the `dsi` node - here, an example from the `galaxy-s3` DTSI:

```c
&dsi_0 {
	vddcore-supply = <&ldo8_reg>;
	vddio-supply = <&ldo10_reg>;
	samsung,burst-clock-frequency = <500000000>;
	samsung,esc-clock-frequency = <20000000>;
	samsung,pll-clock-frequency = <24000000>;
	status = "okay";

	panel@0 {
		compatible = "samsung,s6e8aa0";
		reg = <0>;
		vdd3-supply = <&lcd_vdd3_reg>;
		vci-supply = <&ldo25_reg>;
		reset-gpios = <&gpf2 1 GPIO_ACTIVE_HIGH>;

		// [...]
	};
};
```

Again, read the linked docs for more information. One thing of note here, and in the FIMD panels, is the `reg` value - it dictates the exact way the display is connected. All of that is explained in the docs.

In the case of DSI panels, the `fimd` node **must still be enabled**, as it is used for the framebuffer.

### An aside on the DSI node's clock frequency values

I partially managed to figure out how these values are calculated, and I figured I'd include my research here, as it might come in handy to someone else.

* I found the value of `samsung,esc-clock-frequency` in `arch/arm/plat-s5p/dev-dsim.c` in downstream.
* The PLL clock seems to be the same for all devices in mainline - `24000000`, so it's a pretty safe bet. Not 100% sure where to find it.
* The burst clock (or the high speed/hs clock, as downstream refers to it) is calculated by taking the PLL clock, multiplying it by multiplier `m`, then dividing by `p << s`. The `pms` values, as they are called, are stored in the PLLCTRL register (found in downstream: `((p & 0x3f) << 13) | ((m & 0x1ff) << 4) | ((s & 0x7) << 1);`); their values also seem to be present at the very beginning of `arch/arm/mach-exynos/board-lcd-mipi.c` in downstream.
	* It's also possible to approximate the value, by getting the value of bits `0xffff` from the CLKCTRL register of the DSI master, then multiplying it by the escape clock and by 8. This, however, seems to be somewhat inaccurate, due to the nature of C doing integer divide operations, meaning that anything after the floating point gets lost, which is enough to completely throw off the calculation.

The values of the CLKCTRL and PLLCTRL can be dumped by running `cat /sys/devices/platform/s5p-dsim.0/dsim_dump` in downstream, then cross-referencing the offsets with this table, taken from `drivers/gpu/drm/exynos/exynos_drm_dsi.c` (note: this is the exynos4 table):

```c
static const unsigned int exynos_reg_ofs[] = {
	// [...]
	[DSIM_CLKCTRL_REG] =  0x08,
	// [...]
	[DSIM_PLLCTRL_REG] =  0x4c,
	// [...]
};
```

Also of note are `cat /sys/class/graphics/fb0/device/fimd_dump` and `cat /sys/class/graphics/fb0/device/ielcd_dump`.

### Pin control

Most displays are also going to have an enable pin and/or a reset pin. You can find these by checking the service manual for your device, then checking the files in `arch/arm/mach-exynos/include/mach/` for your device.

For some pins, you might need to set up pin control entries. To figure out whether that's the case, check the GPIO initialization tables for your device (but do **not** mistake them for the sleep tables!), and if there's nothing for that GPIO there, check if some driver down the line uses the GPIO definitions and sets them up manually.

If you can't find any code in downstream that sets up pin control for an otherwise used pin, **don't make a pin control entry yourself!** It's more likely that the pin is already set up as it needs to be, and you'll just end up breaking something when you try to override that.

Pin control entries are defined in the `pinctrl` nodes. Note that depending on the pin you're trying to access, you might need to select a different pinctrl node - for example, `GPC0-1` is under `pinctrl_0`, but `GPM0-1` is under `pinctrl_1`.

Here's an example pin control entry:

```c
	lcd_en: lcd-en {
		samsung,pins = "gpc0-1";
		samsung,pin-function = <EXYNOS_PIN_FUNC_OUTPUT>;
		samsung,pin-pud = <EXYNOS_PIN_PULL_NONE>;
	};
```

Once you add your pin control entries, hook them up to the panel by adding the following lines to the panel node:

```
	pinctrl-0 = <&lcd_en>; // for multiple entries, this will be <&pin1 &pin2> or <&pin1>, <&pin2>
	pinctrl-names = "default";
```

This part is really important - if you don't do this, the kernel won't apply the pin control entries.

In some cases, panel drivers will not accept a GPIO, but a regulator; in that case, you'll need to create a fixed regulator like so:

```c
	lcd_enable_supply: voltage-regulator-3 {
		compatible = "regulator-fixed";
		regulator-name = "LCD_VDD_2.2V";
		regulator-min-microvolt = <2200000>;
		regulator-max-microvolt = <2200000>;
		pinctrl-names = "default";
		pinctrl-0 = <&lcd_en>;
		gpio = <&gpc0 1 GPIO_ACTIVE_HIGH>; /* LCD_EN */
		enable-active-high;
	};
```

**Make sure that you have the fixed regulator driver enabled in the kernel config!** Otherwise, you're going to have a "fun" time figuring out why you're getting mysterious deferred probes.

## Figuring out the panel parameters

Of course, to connect a panel, we need a driver not just for the connector, but for the panel itself as well. Luckily for me, some similar panels were supported by the MSM8916 kernel fork, so I copied [one of the drivers](https://github.com/msm8916-mainline/linux/blob/msm8916/6.2-rc5/drivers/gpu/drm/panel/msm8916-generated/panel-samsung-s6d7aa0-lsl080al03.c) and based my own driver on it.

While the controller - the S6D7AA0 - was the same as the one in the Tab 3 8.0, the panel - LSL080AL03 - was not. Unlike the Tab 3 8.0 panel, it was 1024x768, and not 1280x800. Thus, to get it working on my panel, I had to modify the **timing values**.

For DSI DRM panels, timing values are usually stored in a `drm_display_mode` struct. Here's that struct from the linked driver:

```c
static const struct drm_display_mode s6d7aa0_lsl080al03_mode = {
	.clock = (768 + 18 + 16 + 126) * (1024 + 8 + 2 + 6) * 60 / 1000,
	.hdisplay = 768,
	.hsync_start = 768 + 18,
	.hsync_end = 768 + 18 + 16,
	.htotal = 768 + 18 + 16 + 126,
	.vdisplay = 1024,
	.vsync_start = 1024 + 8,
	.vsync_end = 1024 + 8 + 2,
	.vtotal = 1024 + 8 + 2 + 6,
	.width_mm = 122,
	.height_mm = 163,
};
```

In the downstream Exynos tree, however, the timing data is provided in a very different format:

```c
#ifdef CONFIG_FB_S5P_S6D7AA0
/* for Geminus based on MIPI-DSI interface */
static struct s3cfb_lcd lcd_panel_pdata = {
	.name = "s6d7aa0",
	.width = 800,
	.height = 1280,
	.p_width = 108,
	.p_height = 173,
	.bpp = 24,
	.freq = 58,

	// [...]

	.timing = {
		.h_fp = 16,
		.h_bp = 140,
		.h_sw = 4,
		.v_fp = 8,
		.v_fpe = 1,
		.v_bp = 4,
		.v_bpe = 1,
		.v_sw = 4,
		.cmd_allow_len = 7,
		.stable_vfp = 1,
	},

	// [...]
};
#endif
```

So, nothing we can instantly copy over... just some abbreviations we don't understand.

Conveniently enough, in the linked driver, each of the elements are split up into their individual components. As this driver was generated with [linux-mdss-dsi-panel-driver-generator](https://github.com/msm8916-mainline/linux-mdss-dsi-panel-driver-generator) *(really rolls off the tongue!)*, we can figure out how it was put together:

```python
def generate_mode(p: Panel) -> str:
	return f'''\
static const struct drm_display_mode {p.short_id}_mode = \{\{
	.clock = (\{p.h.px\} + \{p.h.fp\} + \{p.h.pw\} + \{p.h.bp\}) * (\{p.v.px\} + \{p.v.fp\} + \{p.v.pw\} + \{p.v.bp\}) * \{p.framerate\} / 1000,
	.hdisplay = \{p.h.px\},
	.hsync_start = \{p.h.px\} + \{p.h.fp\},
	.hsync_end = \{p.h.px\} + \{p.h.fp\} + \{p.h.pw\},
	.htotal = \{p.h.px\} + \{p.h.fp\} + \{p.h.pw\} + \{p.h.bp\},
	.vdisplay = \{p.v.px\},
	.vsync_start = \{p.v.px\} + \{p.v.fp\},
	.vsync_end = \{p.v.px\} + \{p.v.fp\} + \{p.v.pw\},
	.vtotal = \{p.v.px\} + \{p.v.fp\} + \{p.v.pw\} + \{p.v.bp\},
	.width_mm = \{p.h.size\},
	.height_mm = \{p.v.size\},
\}\};
'''
```

More abbreviations! If these mean nothing to you, don't worry - we'll go over each of them one-by-one.

Data is usually sent to the LCD as a series of low/high signals. As such, the LCD needs to know when a new line of data begins, and requires a bit of time to get back to its original position. This is where timing parameters come in.

The time between lines/frames is known as **blanking**. The blanking period consists of the **front porch** + **sync pulse width** + **back porch**, where:

* "Front porch" and "back porch" are the padding at the start and the end of the blanking period respectively.
* "Sync pulse width" (`sw` in the downstream tree, `pw` in lmdpdg) is the duration of the sync pulse itself.

The "clock" value in the DRM timing struct refers to the **pixel clock** - the total number of pixels per second, including blanking. It's calculated by taking the `htotal` * `vtotal` * `framerate`, then divided by 1000. (In downstream, the `freq` value is equivalent to the framerate here.)

With all of this in mind, the final DRM timing struct for our panel, based on the downstream values, looks like this:

```c
static const struct drm_display_mode s6d7aa0_mode = {
	.clock = (800 + 16 + 4 + 140) * (1280 + 8 + 4 + 4) * 58 / 1000,
	.hdisplay = 800,
	.hsync_start = 800 + 16,
	.hsync_end = 800 + 16 + 4,
	.htotal = 800 + 16 + 4 + 140,
	.vdisplay = 1280,
	.vsync_start = 1280 + 8,
	.vsync_end = 1280 + 8 + 4,
	.vtotal = 1280 + 8 + 4 + 4,
	.width_mm = 108,
	.height_mm = 173,
};
```

### More than just timings

But even though I got all the settings seemingly right, figured out the timings, got the DSI clock frequencies (approximately...), I still couldn't get anything to show up. I was completely stumped on this for *days*, and even considered giving up... but I realized there was one more thing I didn't try yet.

I've mentioned a bit earlier that panels work by sending a low/high signal. But we still don't know whether the data is sent when the signal is high or low! And what about the sync pulse? That can be high or low too.

The kernel has a mechanism for specifying this, and a few other quirks (such as handling vsync refreshes, etc.) - connector **bus flags** and the DSI **mode flags**.

Bus flags are stored in the connector info struct, usually initialized in the `get_modes` function:

```c
static int s6d7aa0_get_modes(struct drm_panel *panel,
					struct drm_connector *connector)
{
	struct drm_display_mode *mode;

	// [...]

	mode->type = DRM_MODE_TYPE_DRIVER | DRM_MODE_TYPE_PREFERRED;
	connector->display_info.width_mm = mode->width_mm;
	connector->display_info.height_mm = mode->height_mm;

	// Bus flags go here:
	connector->display_info.bus_flags = DRM_BUS_FLAG_DE_HIGH |
		DRM_BUS_FLAG_PIXDATA_DRIVE_POSEDGE;

	drm_mode_probed_add(connector, mode);

	return 1;
}
```

You can find a list of bus flags in the [drm_bus_flags struct](https://elixir.bootlin.com/linux/v6.2/source/include/drm/drm_connector.h#L416) in `include/drm/drm_connector.h`.

As for the mode flags - those are initialized in the probe function of the panel[^2]:

```c
static int s6d7aa0_probe(struct mipi_dsi_device *dsi)
{
	// [...]
	dsi->lanes = 4;
	dsi->format = MIPI_DSI_FMT_RGB888;

	// Mode flags go here:
	dsi->mode_flags = MIPI_DSI_MODE_VIDEO | MIPI_DSI_MODE_VIDEO_BURST
		| MIPI_DSI_MODE_VSYNC_FLUSH | MIPI_DSI_MODE_VIDEO_AUTO_VERT;
	// [...]
}
```

You can find a list of DSI mode flags in [`include/drm/drm_mipi_dsi.h`](https://elixir.bootlin.com/linux/v6.2/source/include/drm/drm_mipi_dsi.h#L114).

In my case, it looked fine without the last two, but it would desync after the screen was rotated. When you get your display working, make sure to test things like rotation!

What I did was I first tried to copy the flags from similar panels in mainline - that's how I ended up with the mode flags that eventually worked. As for the bus flags - I just had to experiment. `DRM_BUS_FLAG_DE_HIGH` was the magic flag that made everything work, but your mileage may vary - every panel is different.

Finally, after nearly 2 weeks of trying, I had a working display!

<!--![Picture of the tablet booting postmarketOS](/blog/images/tab3-display/tablet.jpg)-->
{% include image url="/blog/images/tab3-display/tablet.jpg" description="Picture of the tablet booting postmarketOS." %}

## In closing

I hope that this blog post will come in handy for anyone else who may at some point struggle with getting a display working on their own. While some parts are Exynos-specific (like the FIMD and DSI bits), others are useful to everyone - and even if you can't apply my advice one-to-one, just looking at the debugging process and reading some of the explainations might contribute to a working solution.

[^1]: While working on the mainline setup, I initially thought that it used a BP070WX1 panel, as mentioned [in the defconfig](https://github.com/gr8nole/android_kernel_samsung_smdk4x12/blob/baf2ac725ded606f3beea775acbb472a3643887c/arch/arm/configs/lineage_lt01wifi_defconfig#L2350); however, it looks like someone [accidentally swapped the panel config option while updating the defconfig](https://github.com/gr8nole/android_kernel_samsung_smdk4x12/commit/25a3e0dc730c39a7b71daedf6975c573a3702c30#diff-461bf7e8fa9baa4343af1294a4c774ab287a0a057d4b28d79775c11a3a83d6fdL2377-R2379), and the [Kconfig descriptions](https://github.com/gr8nole/android_kernel_samsung_smdk4x12/blob/baf2ac725ded606f3beea775acbb472a3643887c/drivers/video/samsung/Kconfig#L397-L405) confirm this.
[^2]: The code snippet here is slightly different than the panel driver I ended up with; the snippet is adapted to [this commit](https://github.com/torvalds/linux/commit/996e1defca34485dd2bd70b173f069aab5f21a65) which wasn't yet in the kernel I was testing on (v6.2). 
