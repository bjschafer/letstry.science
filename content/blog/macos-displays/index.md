+++
title = "macOS and HiDPI displays"
date = 2026-05-24

description = """\
    A deep dive into how macOS handles HiDPI displays, what's (im)possible, and \
    very little satisfaction...but a fun journey nonetheless. \
"""

[taxonomies]
tags = [
    "macos",
    ]

[extra]
katex = true
+++

## Introduction

{% admonition(type="note") %}
While the debugging journey was done with the help of Claude, the
entirety of this post is all slop from my human brain.
{% end %}

For work, I've found that the nicest main monitor setup is actually just a cheap ~40" 4K TV. If you look
around a bit, you can usually find one for around $200, and 4K is 4K. I don't care about its smart features,
or its speakers, or its refresh rate (60hz is plenty for coding). I've been running on that setup for a number
of years now, augmented with a portrait-orientation secondary display (in this case, the 27" 4K monitor provided
by my job).

When I was at a Windows shop previously, I had no issues with the 4K TV approach. It looked crystal clear, and I just
set up FancyZones to manage the window layout, nbd. When I was later able to run Linux at work, pretty much the same
deal.

Now my work Macbook Pro, however...that's a different story.

## Background

macOS Mojave, way back in 2018, did us all dirty. Apple removed subpixel anti-aliasing in favor of going
all in on Retina displays.

Subpixel AA takes advantage of the fact that each pixel on an LCD is made up of three subpixels, one each for red,
green, and blue. So it might render part of a glyph on only one of those subpixels, and then compensate by doing the
opposite for the next pixel that's part of the background. Roughly.

With Retina displays, macOS instead does something a bit funky. Basically, it'll pretend that your 4K display
is actually a 1080p display, render it accordingly, and then it can use four full display pixels[^1] to render one
"screen" pixel. This is what HiDPI mode means in macOS, and understanding this would've saved me a lot of headache.

{% admonition(type="info", title="Key takeaway") %}
If you learn nothing else from this blogpost, just remember this is how HiDPI rendering
is done in macOS.
{% end %}

Apple decides for you if a display can support HiDPI based on its reported size and resolution. It uses these to
calculate the display's pixel density, represented in pixels per inch (PPI)[^2]. If a display's pixel density
is above about 192ppi, then it qualifies for HiDPI modes. Which, again, remember: all that does is makes your 4K
display have the same screen real estate as a 1080p display.

So, let's do some math!

$
a^2 + b^2 = c^2
$ (everybody remember my dude Pythagoras)?

Rearranged that's $c = \sqrt{a^2 + b^2}$

We first need to find the diagonal in pixels:

$
\sqrt{3840^2 + 2160^2} \approx 4406
$

Then, PPI is given as $\frac{\text{diagonal in pixels}}{\text{diagonal in inches}}$. So:

$
\frac{4406}{40} = 110.15
$

And, if you're familiar with basic math, you'll find that $110.15 < 192$, and thus the display does not qualify
in Apple's eyes as HiDPI. And therein lies my problem. Text often looked bad on the display, but the terminal was
the biggest issue, and that's something that needs to feel good on my eyes.

I'd discovered a while back that [WezTerm](https://wezterm.org) will do its own font rendering on macOS via FreeType, and thus
supports a snippet like so to make font look pretty dang good on a large, non-HiDPI display:

```lua
local wezterm = require("wezterm")
local config = wezterm.config_builder()
config.freetype_load_flags = "FORCE_AUTOHINT"
```

However, with my workplace pushing more agentic development, I wanted to explore other options for running multiple
[Opencode](https://opencode.ai) sessions simultaneously. Someone on my team recommended [cmux](https://cmux.com/), which
seemed promising. It's built on top of Ghostty's libghostty and boasts more native Opencode integration. So I spun it up
and immediately my eyes started bleeding. Turns out, that [Ghostty does not do its own freetype rendering on macOS](https://ghostty.org/docs/config/reference#freetype-load-flags),
instead using macOS's built-in CoreText. So I was screwed. Or was I?

## Hacking around

I want to state for the record that at the beginning of this journey, I had the idea stuck in my head that HiDPI would solve
all my woes. So a lot of this was focused on making that work with my larger monitor.

[BetterDisplay](https://github.com/waydabber/BetterDisplay) lets you do a lot of fun things. One of the things it supports
is reading and overriding your display's [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data). There
are probably other ways to do this, too.

I dumped the TV's EDID, which looks like this in base64:

```text
AP///////wAebQEAAQEBAQEgAQOAoFp4Cu6Ro1RMmSYPUFShCAAxQEVAYUBxQIGA0cABAQEBCOgAMPJwWoCwWIoAQIRjAAAeVl4AoKCgKVAwIDUAQIRjAAAeAAAA/QAYeB6HPAAKICAgICAgAAAA/ABMRyBUViBTU0NSMgogAUcCA0bxV2FgEB9mZQQTBAQDAhIgISIEXV5fYmNkIwlXB24DDAAgALhELACAAQIDBGjYXcQBeIALAuIAz+MFwADjBg0B4g8zAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAHg==
```

Don't, uh, try and just `base64 -d` that, you won't find a whole lot of useful info. However, you can pull out the reported
display size located at `0x15` and `0x16` (in cm[^3]) with:

```bash
base64 -d <<< "$EDID" | xxd -s 21 -l 2 -g 1
```

...which in our case shows 160 cm x 90 cm. Mathing that you'll find that it's reporting as a 72.3" TV with a PPI of 61.
My 40" TV's firmware thinks it's 72". I guess everyone lies about size.

So, let's patch it! BetterDisplay lets you override the EDID, so I did, setting the values at `0x15` and `0x16`
to `2c 19`, (44 x 25 cm) recalculating the checksum as I did so.

### Sidequest: EDID checksums

EDID is organized into 128-byte blocks, and each block ends with a checksum byte at offset `0x7f`. The checksum
is fairly straightforward: the 8-bit sum of all 128 bytes in the block must equal `0x00`. So since this update
only affects the first 128-byte block, all we have to do is:

1. Sum bytes `0x00` through `0x7e`.
2. Take the low byte of that sum (`sum & 0xff`)
3. Set byte `0x7f` to the two's complement of that value, i.e. the byte that makes the total wrap around to zero.

In pseudocode, that looks like

```text
checksum = (-sum(bytes[0x00..0x7e])) & 0xff
```

and, in Python, a quick hacky way is

```python
edid = bytearray(open("display-edid.bin", "rb").read())
edid[0x7f] = (-sum(edid[:0x7f])) & 0xff
open("display-edid.bin", "wb").write(edid)
```

---

If you're lucky, you'll then be able to select a HiDPI mode either in BetterDisplay or in macOS's system settings.

...and you'll find that the one takeaway I mentioned above holds true. You can force it to do HiDPI mode, which will
make your giant display look like it's only 1080p. Really crisp 1080p, but still 1080p.

You can also try something in the between, like 2k mode. But because that's no longer integer scaling,
not only do you lose some real estate over 4K, but it looks terrible the entire time you're doing it.

So I thought, why not try something crazy. If it's basically downsampling, could we trick macOS into thinking our display
is 8K? Um, actually, we can!

Thanks to our friend BetterDisplay, you can create a "dummy display". This is an entirely fake screen that you can
connect to your computer and do with as you see fit. Since it's fake, you can easily create it as an 8K HiDPI display.
This should in theory mean that it looks like a 4K display, right? So then all we need to do is mirror the display
over to the _actual_ 4K display and we'll be set.

Double-check `system_profiler SPDisplaysDataType` as you're doing this to see more than you do in System Settings.

BetterDisplay exposes two means of duplicating display contents. Mirroring is I believe what's available in
System Settings, and is used e.g. for making what's on your screen look the same as what's on a connected projector.
If you've ever done that, you're probably familiar than inevitably one of the displays looks like crap. So you
shouldn't be surprised to find out that this indeed did look like crap. Claude suggested that "the GPU's
display engine performed bilinear downscaling; the TV additionally downscaled the 8K signal internally".

What about the second means? BetterDisplay will "stream" the virtual display, which seems to use AirPlay or
something similar. This is perhaps the worst of all, as the stream undergoes compression that makes it look terrible,
the cursor is weird, and it honestly looked like pixels were fading in and out.

## Conclusion

Looks like no cmux for me. (Actually, I was able to use the laptop screen for it to give it a test drive.)

What did we learn? A whole lot about the macOS display pipeline, apparently. When I ended up right back where
I started, Claude had the gall to tell me:

> We're done. You now know more about macOS's display pipeline than 99% of the users, and you've
> definitively proven what doesn't work -- which is its own kind of answer.

Smarmy git.

[^1]: 2x scaling; $2*2 = 4$
[^2]: Sorry, non-Americans. And sane Americans.
[^3]: Sorry, Americans.
