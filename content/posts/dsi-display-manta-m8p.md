---
title: "DSI Display on a Manta M8P"
date: 2023-12-14T17:24:19-06:00
draft: false
categories:
    - 3D Printing
tags:
    - trident
summary: How to get a DSI display (ribbon cable) display to work with a Raspberry Pi compute module on a Manta M8P.
---

To get a DSI (aka Raspberry Pi ribbon) display to work on a Manta M8P (v2.0 in my case), the steps were a little unclear.

I'm using [this display](https://www.fabreeko.com/products/raspberry-pi-5-inch-touch-screen-ips-800x480-by-fysetc?_pos=8&_sid=876c0bda2&_ss=r) by Fysetc, which seems to maybe not really exist.
I spent way too long trying to find info on it, but was barking up the wrong tree.

The product page notes that you need to remove or comment out the following line from `/boot/config.txt` on your Pi:

```
dtoverlay=vc4-kms-v3d
```

This does need to happen, go ahead and do it.

The remainder of the config is actually board-specific. If you were plugging this into a plain-old Raspberry Pi 4, say, running Raspberry Pi OS, you wouldn't have to do any of this.

The [Manta M8P manual](https://github.com/bigtreetech/Manta-M8P/blob/master/V2.0%2FBIGTREETECH%20MANTA%20M8P%20V2.0%20User%20Manual.pdf) says:

> The default display interface is HDMI. The DSI interface of the MANTA M8P is DSI1.
> To use it, download the DSI1 driver[...][^1]
>
> After downloading this driver and restarting, the screen on the DSI interface can
> be displayed normally. If you want to use the HDMI interface, delete the
> downloaded /boot/dt-blob.bin driver and restart, then HDMI can output normal

It's not entirely right, but led me down the correct path. With that tidbit of info, I
found [this official doc](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/compute-module/cmio-display.adoc#quickstart-guide-display-only)
from Raspbery Pi, discussing how to get DSI displays working on computer module boards (which the M8P is, at its core).

It offers a command we can run:

```bash
sudo wget https://datasheets.raspberrypi.com/cmio/dt-blob-disp1-only.bin -O /boot/firmware/dt-blob.bin
sudo reboot
```

Upon doing that, the display was immediately picked up by X11 and thus by KlipperScreen and started working.

[^1]: The driver they link to is for running a display and camera simultaneously. In my testing, it didn't work and I needed to grab the `disp1-only` blob instead.
