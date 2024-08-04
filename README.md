# Beepy + Radxa Zero

[Beepy](https://beepy.sqfmi.com/) is a handheld Linux device, with a Blackberry-like keyboard and monochrome Sharp Memory LCD. It's normally powered by a Raspberry Pi Zero W.

This repo contains my notes for swapping out the Raspberry Pi Zero for a [Radxa Zero](https://wiki.radxa.com/Zero) instead.

## Why Radxa Zero over Raspberry Pi Zero?

Raspberry Pi is great, but not ideal for handheld, battery-powered devices. Raspberry Pi doesn't implement suspend to RAM or hibernate, due to various reasons related to the boot chain and Broadcom's closed first stage boot. Additionally, the Pi also doesn't have a good concept of a poweroff state. The Pi draws significant current even in "poweroff", and there is also no way to initiate boot from poweroff outside of cycling the power.

These limitations are fine for a lot of projects, but isn't a great fit for Beepy, which is trying to be a chat device that can stay idle for days/weeks at a time. To address this, Beepy has a management MCU (RP2040), which performs power management in addition to providing keyboard support. The MCU controls the power rail to the Pi, solving the issue of the Pi's quiescent current draw in poweroff. The MCU also sequences power on based on the power key, or an RTC-based timer. The concept is to optimize the Pi's Linux boot times, and have the MCU periodically boot it to quickly check for messages before powering off again. Since the display is a Sharp Memory LCD, notifications and alerts can remain on the screen, even in this soft shutdown state.

Still, it would be great if we could just... have working suspend and hibernate instead of dealing with all this. The Radxa Zero uses an Amlogic SoC that supports suspend and hibernate, and the goal of this project is to see if we can get acceptable suspend power draw with it.

None of this stuff is really polished, but it should generally work for day-to-day use. Suspend power still needs to be investigated and optimized, but worst case, you can just enter shutdown and be at feature parity with the Raspberry Pi Zero. :)

## Buying

### Radxa Zero

You want a SKU that doesn't come with the headers pre-soldered, so it can mate with the Beepy pogo pins.

I purchased mine from [Allnet](https://shop.allnetchina.cn/products/copy-of-radxa-zero?variant=39440872308838), an official Radxa distributor.

I wasn't able to get eMMC boot working with Armbian, so YMMV if you want the larger eMMC models.

### Antenna

All the SKUs without headers seem to also only have external antenna. Any small 2.4/5 GHz flex antenna with a U.FL connector should work. I bought [this one](https://www.digikey.com/en/products/detail/antenova/SRF2W012-100/5959788) off Digi-Key.

## OS Setup

[Armbian for Radxa Zero](https://www.armbian.com/radxa-zero/)

I'm running Armbian 23.11 Bookworm. I haven't tested on later releases, so use a newer major release at your own risk.

If you have an eMMC model, follow the instructions to erase the eMMC to enable microSD boot.

Flash the system image to the microSD card and boot as usual.

You won't have Beepy keyboard or display support at this point, so a micro-HDMI cable and USB-C hub will come in handy here. Once you have Wi-Fi and SSH set up, you can just SSH in for subsequent boots.

## Install Kernel Modules

There are two kernel modules to install, for display and keyboard support. These are forked from the Beepy drivers.
The main changes are to provide Radxa Zero Device Tree configs and modify the Makefile to build and install on Armbian.


The following steps should be performed on your Radxa Zero.

### Dependencies

```
apt install linux-headers-current-meson64
```

### Display Driver

[Repo](https://github.com/Hylian/sharp-drm-driver)

```
git clone https://github.com/Hylian/sharp-drm-driver.git
cd sharp-drm-driver
make
make install
```

#### Troubleshooting

Once the driver is loaded, you should see a framebuffer device show up at `/dev/fb0`. Writing to this device should update the screen (e.g. `cat /dev/random > /dev/fb0`).

If you aren't seeing the TTY show up on the display, your [fbcon mapping](https://www.kernel.org/doc/Documentation/fb/fbcon.txt) may be incorrect. The Makefile install target adds a [fbcon boot arg](https://github.com/Hylian/sharp-drm-driver/blob/radxa-zero/Makefile#L21) to `/boot/armbianEnv.txt`, with an fbcon mapping that worked for me. Try modifying `/boot/armbianEnv.txt` with a different mapping and reboot. If you only see console messages without a TTY appearing, you are probably in this state.

### Keyboard Driver

[Repo](https://github.com/Hylian/beepberry-keyboard-driver)

```
git clone https://github.com/Hylian/beepberry-keyboard-driver.git
cd beepberry-keyboard-driver
make
make install
```

### Finishing up

Reboot your device.

### Developing

One-liner to rsync and install: `rsync -a ~/dev/beepy/sharp-drm-driver root@radxa-wifi.local:~/ && ssh root@radxa-wifi.local "cd ~/sharp-drm-driv
er && rm -rf out && make && make install"`

## Quirks and Tweaks

### My Beepy randomly suspends while typing!

This may happen when you try to type `-` (`Alt + i`). This is because the Beepy MCU firmware emits keycode 142 for that key, which then gets mapped to `minus` in the [keyboard map](https://github.com/Hylian/beepberry-keyboard-driver/blob/radxa-zero/beepy-kbd.map#L67). Keycode 142 collides with [`KEY_SLEEP`](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h#L220). 

#### Fix

Configure `logind` to ignore the sleep key.

* `vim /etc/systemd/logind.conf`
* Set `HandleSuspendKey=ignore`

### How do I suspend?

The `systemctl suspend` command will put the Radxa into suspend. You should see the cursor stop blinking, and the system will resume upon any keypress. You can also check `dmesg` to confirm that it entered suspend.

### Can I remap the power key to suspend?

Yes, you can modify the keyboard driver. Change [this](https://github.com/Hylian/beepberry-keyboard-driver/blob/170beef94a83ef5653dc92469f4a20bb6d8fe6b3/src/input_fw.c#L29) line to instead invoke `systemctl suspend`. Then configure the kernel module to [handle poweroff in-driver](https://github.com/Hylian/beepberry-keyboard-driver/blob/170beef94a83ef5653dc92469f4a20bb6d8fe6b3/src/params_iface.c#L315). The power button long press event from firmware should now invoke suspend.

Shutting down from shell should still work fine, as when the keyboard driver is unloaded during poweroff, it informs the MCU. The MCU should enter deep sleep and power on the Radxa like normal on the next power button press.

### How do I measure battery state of charge?

These sysfs exports are available via the keyboard driver:

`/sys/firmware/beepy/battery_raw`

`/sys/firmware/beepy/battery_volts`

`/sys/firmware/beepy/battery_percent`

Note that the model used for battery percentage is pretty rough, and may not be accurate.

## Optimizing Power

* I only have a USB power meter, which works OK with the battery unplugged, but it's still capturing power supply losses. Power meter into the battery terminals would be better, and separate shunt resistors to the Radxa and RP2040 would be ideal. I put in a feature request to SQFMI to add shunt resistor jumpers in the next revision.

* *TODO:* In suspend, DRAM still needs to self-refresh. We should characterize quiescent power draw.

* *TODO:* Investigate Radxa/Amlogic suspend sequence and see what clocks and peripherals stay up. Can we gate any of them?

* *TODO:* Investigate if wireless chipset stays up during suspend. If it does, we should block Wi-Fi and Bluetooth with `rfkill` before suspending.

* *TODO:* The MCU still stays active when Radxa suspends. The MCU will consume around 20mA @ 5V in this state, which isn't ideal. When in shutdown, the MCU enters deep sleep, consuming much less power, and only wakes on a wakeup timer or power button interrupt. We should implement similar handling for the suspend case, entering suspend+deep sleep via power key tap. After that, we can measure the actual power draw in suspend.

### Power Measurements

(08/24): Measured with a USB power meter, battery unplugged.

| Radxa    | RP2040     | Watts |
| -------- | ---------- | ----- |
| Active   | Active     | 0.8W  |
| Suspend  | Active     | 0.45W |
| Suspend  | Deep Sleep | TODO  |
| Poweroff | Deep Sleep | 0.05W |

According to the measurements in the RP2040 [repo](https://github.com/ardangelo/beepberry-rp2040), MCU active draws 0.125W, so Radxa in suspend should be contributing 0.275W power draw.

That means in suspend, we'll draw: (Radxa_suspend + RP2040_deepsleep) = (275mW + 25mW) = 300mW

Beepy battery is 7.4Wh, so total suspend time should be: (7.4Wh/300mW) = 24.67 hours
