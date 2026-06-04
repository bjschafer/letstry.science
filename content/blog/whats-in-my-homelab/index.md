+++
title = "What's in my homelab?"
date = 2026-06-03

description = """\
    A tour of what I run in my homelab and how. Hardware, software,
    and so on. Ask for more details!\
"""

[taxonomies]
tags = [
    "homelab",
    ]
+++

I get asked a surprising amount about what I've got in my homelab, how I'm running it, what it looks like,
and so on. I thought it might be fun to write it up!

{% admonition(type="note") %}
This is a work in progress, I intend to add more detail and pictures as time permits.
{% end %}

## Hardware

I run my homelab out of a 4-post fully enclosed cabinet/rack. Everything is either rackmount
or lives on a rack shelf.

### Cooling

I do a jank approximation of hot and cold aisles. The utility room is the cold
aisle, and the heat exhausts through a powered duct booster into either the warm air return of the furnace
or out into the garage, depending on season. This works decently well, the inside of the rack is about
10°F/5.5°C warmer than ambient, and the *warmest* device in the rack averages around 136.4°F/58°C.

### Power

I used to have 2x 20A circuits dedicated to the rack, along with a collection of whatever cheap-ish UPSes
I could find, totalling about 3000VA across three units. Then I happened across a great deal on a pair of [Eaton 5P3000RT](https://www.eaton.com/us/en-us/skuPage.5P3000RT.html)
rackmount UPSes, each of which offers 3000VA of capacity and just shy of 30 minutes of runtime.

So I replaced one of the 20A circuits with a 30A circuit (by running new 10ga romex, thankfully my breaker
panel is in the same room) to be able to safely and reasonably provide the L5-30 socket required by the
new UPSes. The second 20A circuit is unused and likely to lose its breaker spot to an EV charger in the
near-ish future.

My entire rack consumes on average 1.1kW. My house is regrettably not in a good place to have solar panels.
However, I credit what I've learned in my lab with a large part of being able to move from my career beginnings
as a desktop support engineer/opportunistic dev to a senior systems engineer, so to me the electricity price
is very much a worthwhile investment in my continuing education and career progression.

### Network

We have Starlink as our ISP. From a technical standpoint, it's incredible and has worked amazingly well
for the 2+ years we've lived in this house. The alternative is 25Mbps DSL, and given we both work from home
that's untenable. The sad thing is, there's FttH about a mile away, but no sign of extending it farther.

With Starlink, I'm currently seeing ping RTT around 20ms to most well-connected networks. My last speedtest
was 417Mbps down and 46Mbps up. Both of these numbers have improved continuously since we started using Starlink.

I'm currently running an all-Unifi network, consisting of:

- Unifi Dream Machine Pro
- Unifi Switch Pro Max 48 PoE
- 2x Unifi Switch Lite 8 PoE (upstairs and downstairs entertainment center, so consoles can be hardwired)
- 3x U7 Pro APs
- An old Nano HD AP in the workshop for extended coverage

And other than an occasional hiccup with some of the newer features when they're first introduced (for example,
I had to turn 6GHz off on the U7 Pros for a while because Android devices would randomly lose connections), it's
all been solid for me.

I did recently upgrade from a USW Pro to the Pro Max, so hit me up if you want to buy a USW Pro 48 port :)

When we bought the house, I ran cat6a throughout the house, which is always a fun challenge in a preexisting
house. The worst run was the doorbell. I ran NM conduit from the basement where the rack lives up into the attic
to make future runs easier.

### Servers

I run three Supermicro white-label 2U servers. Two are identical save RAM loadout:

- Dual socket [Xeon E5-2690 v2](https://www.intel.com/content/www/us/en/products/sku/75279/intel-xeon-processor-e52690-v2-25m-cache-3-00-ghz/specifications.html), 10c/20t each
- 384GB *OR* 192GB memory
- 2x 120GB SSDs in a ZFS mirror as root disk
- 8x 600GB 10k RPM spinning rust as Ceph OSDs
- 10GbE NIC

And the other is a generation newer:

- Dual socket [Xeon E5-2620 v4](https://www.intel.com/content/www/us/en/products/sku/92986/intel-xeon-proocessor-e52620-v4-20m-cache-2-10-ghz/specifications.html), 8c/16t each
- 256GB memory
- 2x 120GB SSDs in a ZFS mirror as root disk
- 8x 600GB 10k RPM spinning rust as Ceph OSDs
- 10GbE NIC

This one has a SAS card in it connecting to a NetApp DS4246 24-bay disk shelf. In this, I have 12x 10TB hard drives
in a zpool, as 2x 6 device raidz2 vdevs. This wasn't planned per se, it's just how I ended up growing it when
I needed more space.

### Home Security

## Software

### Proxmox

### Kubernetes

## Networking

I segregate into a number of VLANs:

- Guest
- Servers
- LAN
- IoT
- DMZ
- Servers
- Network management
- IP Cameras
- Ceph

I have three separate SSIDs for LAN, Guest, and IoT. IoT is limited to 2.4GHz due to poor compatibility with a number
of cheap IoT devices.

I run iBGP between the router and:

- Proxmox hosts for a "floating management IP"
- K8S control plane nodes, via kube-vip
- K8S services, via metallb. See [Getting real IPs from behind ingress](https://letstry.science/blog/getting-real-ips-from-behind-ingress/) for why I run both.
