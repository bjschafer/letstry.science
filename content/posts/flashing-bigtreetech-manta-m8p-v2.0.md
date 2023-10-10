---
title: "Flashing BigTreeTech Manta M8P V2.0"
date: 2023-10-10T11:22:07-05:00
draft: false
categories:
    - 3D Printing
tags:
    - klipper
    - voron
    - trident
toc: true
summary: >-
    Notes on getting the BigTreeTech Manta M8P v2.0 printer MCU flashed and working, plus some clarifying points that should
    be useful to owners of an M8P v1.0/v1.1 board. Includes some info on how to configure for CANbus and how to get the
    BigTreeTech EBB SB2209 board flashed and working as well.
---

This is adapted from my notes when I was trying to build my Voron Trident. It's not perfect, but much of the info had to be discovered by myself and didn't seem to exist or be clearly documented. I decided to not let perfect be the enemy of good and share this with the world.
## CB1

The BTT CB1 is a Raspberry PI CM4 "clone" that's drop-in compatible from a pin/mount standpoint. However, it uses a different SoC so you can't just run Raspberry Pi OS on it.

They recommend using the [BigTreeTech Linux distro](https://github.com/bigtreetech/CB1/releases), which seems to be based on Raspberry Pi OS and thus on Debian oldstable (Bullseye).

Because it uses an [Allwinner H616 SOC](https://linux-sunxi.org/H616) it's not fully supported til kernel 6.0 so requires the custom kernel in the BTT distro. It also requires U-boot shenanigans, hence why it's easiest to use the BTT Linux distro.

If you elect to try to run real Debian on there, [this page](https://wiki.debian.org/InstallingDebianOn/Allwinner#U-boot_versions_for_sunxi-based_systems) is probably a good starting point. Then just run bookworm or later to ensure you have the right modules available in the kernel. Let me know if you get it working, because I'd love that.

Of note, BTT provides two versions you can flash. The "minimal" version is roughly equivalent to Raspberry Pi OS's "lite" edition and just includes a basic CLI-only install. The full version appears to be based on MainsailOS.

## Manta M8P

The docs for flashing the BTT Manta M8P and EBB SB2209 for CAN are not the clearest, and also don't currently take into account the newer Manta v2.0 board. This is an attempt to provide clearer steps.

It assumes you're otherwise familiar with how to build / flash / configure Klipper. I'm not going to write a tutorial for that when many exist and the official docs are also great.

### Identifying your board

A number of these steps require you to identify the version of your M8P board. While misidentifying will _probably_ not wreck your board. it's also not hard to screw up.

There are three board revisions I know of: v1.0, v1.1, and v2.0. For the purposes of this guide, v1.0 == v1.1, and I'll refer to v1.0 in those cases. v2.0 uses a different SoC and has different pin definitions.

In all three cases, the version should be prominently silkscreened on the top of the board (for v1.0 there's a chance the silkscreen is on the bottom of the board -- try to look and identify if you can prior to mounting).

### Notes on CAN

If you think of a CAN network as a ring, the devices on each end of the ring must be configured as a "terminal".

On both the M8P and the EBB2209, terminal-ness is configured by shorting the `120R` jumper.

Since in a printer, there are exactly two devices, both must have the 120R jumper set at all times.

### Building and flashing

On the M8P, you need to configure both Klipper and [Katapult](https://github.com/Arksine/katapult) (né CanBoot). On BTT linux, you'll need to clone Katapult yourself into `$HOME`.

#### Katapult

Start in Katapult with `make menuconfig`. Set the following options:

- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32H723** (v2) | **STM32G0B1** (v1)
- Build Katapult deployment application: **Do not build**
- Clock Reference: **25 MHz crystal** (v2) | **8MHz crystal** (v1)
- Communication interface: **CAN bus (on PD0/PD1)** (v2) | **CAN bus (on PD12/PD13)** (v1)
- Application start offset: **128KiB offset** (v2) | **8KiB offset** (v1)
- CAN bus speed: **1000000** (pick a number, and use it everywhere).

Build it with `make`.

#### Klipper

Now, for Klipper, run `make menuconfig` and set the following options:

- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32H723** (v2) | **STM32G0B1** (v1)
- Bootloader offset: **128KiB bootloader** (v2) | **8KiB bootloader** (v1)
- Clock Reference: **25 MHz crystal** (v2) | **8MHz crystal** (v1)
- Communication interface: **USB to CAN bus bridge (USB on PA11/PA12)**
- CAN bus interface: **CAN bus (on PD0/PD1)** (v2) | **CAN bus (on PD12/PD13)** (v1)
- CAN bus speed: **1000000** (pick a number, and use it everywhere).

Build it with `make`.

![[Pasted image 20231007195703.png]]

#### Flashing

- Unplug CAN cable from M8P board.
- Ensure 120R jumper is set (above the CAN connector)

Flash. Between each, reset with Reset button on board, and put in DFU mode (hold boot0 and press Reset).

```bash
## if using a M8P v1.0/v1.1, change the second dfuse-address to 0x08002000

sudo dfu-util -a 0 -D ~/Katapult/out/katapult.bin --dfuse-address 0x08000000:force:leave -d 0483:df11
sudo dfu-util -a 0 -d 0483:df11 --dfuse-address 0x08020000 -D ~/klipper/out/klipper.bin
```

## EBB SB2209 Toolboard

The gist of this is you're going to flash by directly connecting a USB A to C cable from your CB1/Manta board to the toolboard.

Follow https://gist.github.com/fredrikasberg/c14f08eb8617bf8981f77dbc01b00602 for EBB SB2209. They do a better job than I could.

**Don't forget to plug the CAN cable back in when finished!**

## Misc notes

- Any changes to CAN cabling seem to require hitting reset on M8P.
- After finishing, reboot CB1 and reset M8P. Maybe do it twice if it's being dumb?

## Klipper config

If you have a v1.0/v1.1 M8P board, use the existing Klipper example configs as your starting point.

If you have a v2.0 M8P board, start from the example in the M8P repo: https://github.com/bigtreetech/Manta-M8P/blob/master/V2.0/Firmware/generic-bigtreetech-manta-m8p-V2_0.cfg as *pin definitions have changed*.

## Links/reference

- [M8P v1.0/v1.1, EBB2209 flashing guide](https://gist.github.com/fredrikasberg/c14f08eb8617bf8981f77dbc01b00602)
- [Installing Debian on Allwinner](https://wiki.debian.org/InstallingDebianOn/Allwinner#U-boot_versions_for_sunxi-based_systems)
- [BTT CB1 Linux image](https://github.com/bigtreetech/CB1/releases)
- [Manta M8P v2.0 manual](https://github.com/bigtreetech/Manta-M8P/blob/master/V2.0/BIGTREETECH%20MANTA%20M8P%20V2.0%20User%20Manual.pdf)
- [BTT docs for M8P and co.](https://bigtreetech.github.io/docs/M8P.html)

