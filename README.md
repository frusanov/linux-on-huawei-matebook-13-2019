# Linux on Huawei MateBook 13 (2019)

> Brain dump: MateBook 13 (Wright-W19) running Debian heavily inspired by [lidel's documentation of running Linux on MateBook X](https://github.com/lidel/linux-on-huawei-matebook-x-2017).

## Background

Huawei MateBook 13, released in 2019, has at least two modifications (different CPUs, integrated vs. dedicated graphics). Both come with Microsoft Windows 10 and there initially was no information at all concerning Linux support.

I am running Debian on the more simple MateBook 13 variant, model Wright-W19. This repository documents what works and what does not.

## Linux Support Matrix

| Device | Model |  Works | Notes |
| --- | --- |  :---: | --- |
| Processor | Intel Core i5-8265U | ✔ Yes | 8 cores, power states etc seem to work out of the box |
| Graphics | Intel UHD Graphics 620 | ✔ Yes | via standard kernel driver |
| Memory | 8192 MB | ✔ Yes |  |
| Display | 13 inch 2:3, 2160x1440 (2K) | ✔ Yes | resolution is correctly detected by `xrandr`, backlight control works via native function keys and can be controlled by KDE settings |
| Storage | Samsung SSD, 256 GB | ✔ Yes | via standard kernel driver |
| Wifi | Intel Cannon Point Wireless-AC 8265 (a/b/g/n/ac) | ✔ Yes | requires kernel 4.14 and firmware (`firmware-iwlwifi` non-free package) |
| Bluetooth | Intel Bluetooth 5.0| ✔ Yes | works as expected |
| Soundcard  | Intel Cannon Point-LP High Definition Audio | ✔ Yes  | via standard kernel driver, works fine with `pulseaudio` |
| Speakers  |  | ✔ Yes |  |
| Microphone | | ✔ Yes | out of the box |
| Webcam | HD Camera (13D3:56C6) | ✔ Yes | works out of the box, indicating light too |
| Ports | 2 × USB-C | ✔ Yes | charging works only via left port, external display only via right one, but it is a known hardware limitation of the laptop |
| Power button |  | ✔ Yes | needs to be pressed for at least a second to generate event |
| Fingerprint Reader | some proprietary sensor | ❌ No | located on the power button  |
| Battery | Dynapack HB4593J6ECW (42 Wh) | ❕ Mostly | basically, works: current status, charging/discharging rate and remaining battery time estimates; however, so far there seems to be no support for `natacpi`, so setting up charge/discharge thresholds via TLP is not available (it can be done on Windows using Huawei utility, so it should be technically possible) |
| Lid | ACPI-compliant |  ✔ Yes | works as expected, though ACPI complains in logs |
| Power management | | ✔ Yes | works, [see below](#power-management) for details |
| Keyboard |  | ❕ Mostly | [see below](#keyboard) for details; microphone mute LED doesn't work |
| Touchpad | ELAN962C:00 04F3:30D0 | ⁉ Kinda | touchpad is detected and works in KDE (though not in Debian installer), but almost all options are greyed out; double- and three-finger clicks work, so does double-finger scrolling, multi-touch gestures can not be set up; [see below](#touchpad) for details on log flood issue |
| Port Extender | MateDock 2 dongle included with the laptop | ✔ Yes | D-SUB, full-size HDMI, USB-C and USB-A work as expected |

## BIOS updates

Huawei [provides](https://consumer.huawei.com/en/support/laptops/matebook-13/) downloadable BIOS updates packaged for Windows. With some effort, these can be installed from Linux.

To update BIOS, make sure `fwupd` is installed. You'll also need [firmware-packager](https://github.com/hughsie/fwupd/tree/master/contrib/firmware-packager) script and `gcab` that it depends on. I strongly advice having a bootable USB drive for bootloader recovery close at hand, too. Laptop should be on AC power for firmware updater to work.

**PLEASE read through all the steps before you start and make sure you have at least a vague understanding of the process! Don't hold me responsible if you trash your system or brick your BIOS!!!**

1. Download BIOS from Huawei website. Version 1.0.5 (we'll use it as an example) comes in a `.zip` file that contains a signature and another `.zip` file with the same name. You need that second `.zip` file, so extract it to the directory you have `firmware-packager` script in.

2.
        ./firmware-packager --firmware-name HuaweiBIOS --device-guid 4ab52f4e-04c0-47ec-af33-a4f5c28ce0b7 --developer-name Huawei --release-version 0.1.0.5 --exe ./MateBook_13_BIOS_1.05.zip --bin ./MateBook_13_BIOS_1.05/WRIWU105.bin --out bios.cab

3.
        fwupdmgr install bios.cab

4. Reboot (or hibernate) and hold F12 upon boot to select updater from list of devices. It reboots again during the process, so make sure to press F12 during the second reboot, too, for the process to continue.

5. Now your new BIOS is installed and you may check its version holding F2 during the next reboot. However, your UEFI boot record is likely messed up as the result of Step 4, so your system won't boot from SSD any longer.

6. Fix your bootloader using the bootable USB drive. I used a Debian Live image with persistence that had `grub-efi-amd64` and its dependencies pre-installed, so it was only a matter of mounting `/boot` and `/boot/efi` to `/mnt/system/` and issuing
        sudo grub-install --boot-directory=/mnt/system/boot --bootloader-id=debian --target=x86_64-efi --efi-directory=/mnt/system/boot/efi

Your mileage may vary.

## Temperature

Out of the box fan control is very much acceptable, with fans starting up as processor heats up under load and shutting down when not required. In general, under "office workload" the laptop remains cool and fans remain switched off.

If you want correct CPU temperature displayed in `byobu` status notifications, add the following line to your `.byobu/statusrc`:

    MONITORED_TEMP=/sys/class/hwmon/hwmon1/temp1_input

## Power Management

Suspend to S3 state works out of the box. For hibernation to work `Secure boot` must be disabled in BIOS. Laptop seems to wake up without any issues.

## Keyboard
Keyboard mostly works out of the box, including the not-so-documented hotkeys (Fn+Left for Home, Fn+Right for End, Fn+Up for PgUp, Fn+Down for PgDn). However, Microphone Mute, WiFi Switch and Huawei keys don't work out of the box.

To have them working there's [a patch](https://github.com/aymanbagabas/Huawei-WMI) that is already incorporated in Linux kernel, just not yet in Debian. It can be installed using DKMS .deb package that the author [provides](https://github.com/aymanbagabas/Huawei-WMI/releases).

The same source seems to have a patch for Microphone Mute LED to work (at least on MateBook X). However, I have yet to find time to backport it to current Debian kernel and test.

## Touchpad

Out of the box touchpad floods system logs with error messages `incomplete report (14/65535)` upon every touch - up to the point where rubbing your finger against touchpad produces 15% CPU usage by syslog. The [corresponding patch](https://patchwork.kernel.org/patch/10750063/) is available in mainline kernel, but not in Debian (yet), and the patch can not be directly applied to the kernel version currently in Debian. Patching the kernel with [adapted patch](elan-touchpad-oldkernel.patch) fixes this issue.
