+++
title = "What's in my homelab?"
date = 2026-06-03
updated = 2026-09-10

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

{{ toc() }}

## Hardware

I run my homelab out of a 4-post fully enclosed cabinet/rack. Everything is either rackmount
or lives on a rack shelf.

{{ figure(src="full-rack.jpg" alt="Picture of the front of the aforementioned rack, in a messy utility room.")}}

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
new UPSes. The second 20A circuit is unused and recently lost its breaker spot to an EV charger.

My entire rack consumes on average 1.1kW. My house is regrettably not in a good place to have solar panels.
However, I credit what I've learned in my lab with a large part of being able to move from my career beginnings
as a desktop support engineer/opportunistic dev to a senior systems engineer, so to me the electricity price
is very much a worthwhile investment in my continuing education and career progression.

### Network

We have Starlink as our ISP. From a technical standpoint, it's incredible and has worked amazingly well
for the 2+ years we've lived in this house. The alternative is 25Mbps DSL, and given we both work from home
that's untenable. There's fiber currently a mile away, and the local telco says they do intend to bring it to us
in the next few years, so there's hope!

With Starlink, I'm currently seeing ping RTT around 20ms to most well-connected networks. My last speedtest
was 417Mbps down and 54Mbps up. Both of these numbers have improved continuously since we started using Starlink.

I'm currently running an all-Unifi network, consisting of:

- Unifi Dream Machine Pro
- Unifi Switch Pro Max 48 PoE
- 2x Unifi Switch Lite 8 PoE (upstairs and downstairs entertainment center, so consoles can be hardwired)
- 3x U7 Pro APs
- An old Nano HD AP in the workshop for extended coverage

And other than an occasional hiccup with some of the newer features when they're first introduced (for example,
I had to turn 6GHz off on the U7 Pros for a while because Android devices would randomly lose connections), it's
all been solid for me.

When we bought the house, I ran cat6a throughout the house, which is always a fun challenge in a preexisting
house. I ran NM conduit from the basement where the rack lives up into the attic
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

Right now my home security story is kind of in a transitional period. I've a few Amcrest IP cameras (including indoor "puppy cams") that were preexisting. Those are currently dual-captured by an Amcrest NVR and Unifi Protect. I also have a Unifi video doorbell (that ethernet run was *the worst* I have ever done), and a Yale x Nest keypad door lock, along with some OpenGarages. The lock just won't die, though it keeps threatening to. It doesn't integrate well with anything, so I don't love it. But not having
to carry a key, and being able to give friends or petsitters temporary unique passcodes and see them come and go is a big plus.

## Software

### Proxmox

Proxmox is what I run everything on top of. It's fantastic because at its core it's just Debian, so if you know your way around that you can do anything you want with it. Their web interface is also second to none, and was so much nicer than VMware's or
trying to fight through XenServer's client. And unlike just running your own VMs on libvirt, it *has* a web interface.

Plus it has full support for HA and live migration. And, because it uses KVM under the hood
it can support live migration between AMD and Intel, which was a killer feature back when my homelab was built entirely
out of spare parts.

Proxmox also makes it easy to set up a Ceph cluster, which I've done from a pile of used enterprise 10k hard drives spread
betwixt my three hosts. This provides block storage for all[^1] of my VMs as well as persistent storage for Kubernetes.

{{ zoomable(src="proxmox-dashboard.png" alt="A screenshot from the Proxmox dashboard showing three hosts, 12 running VMs, 2 running LXC containers, Ceph, and not a wild amount of resource utilization.")}}

### Kubernetes

You'll notice that I don't run a whole lot of VMs. That's because I run anything I possibly can in Kubernetes, and just run
my K8S nodes as VMs. Then, instead of having a house full of pets I have a barn full of cattle!

I run [k3s](https://k3s.io) as a pretty lightweight means of running and managing K8S. It's just on stock Debian like everything
else I run. Persistent storage is available via both Ceph RBD for block storage and CephFS for sharable storage. L3 ingress is
provided by MetalLB (and sort of kube-vip, see [this post](../getting-real-ips-from-behind-ingress/index.md) for why) talking
BGP to my UDM.

Everything in K8S is managed via [cdk8s](https://cdk8s.io) and [ArgoCD](https://argo-cd.readthedocs.io/). My [generators repo](https://github.com/bjschafer/k8s-generators/) is public on GitHub and pretty active. I've spent a lot of time recently
automating updates via Renovate. I also have ArgoCD Image Updater (see [this post](../argocd-image-updater/index.md) for more
on how I set that up in a not-sucky way, along with [lib/argo.ts](https://github.com/bjschafer/k8s-generators/blob/main/lib/argo.ts) for how I use it) running to update the straightforward things that a normal person would just set to `:latest`. Renovate
helps me out by keeping the cdk8s tooling up to date, as well as things like CRDs, JSON schemas that get turned into TypeScript types for use in Helm values files, and also creates manual MRs when a new k3s version is released so I don't have to keep
watching for new releases.

### SSO

One of the best things I did for my homelab was setting up [Authentik](https://goauthentik.io/) for SSO. Rather than having
to manage accounts in every product, manually set things up for my wife or for friends and family using stuff, and so on,
I just wire everything up. Authentik supports pretty much every means of authenticating out there, and has pretty good docs
on how to integrate it. It runs in K8S.

### Databases

Pretty much everything that I can, I run on Postgres. I used to manage that manually with a big VM and a Raspberry Pi
as a replica. But then I discovered [CloudNativePG](https://cloudnative-pg.io/) and my world was forever changed. Now I have
a three node Postgres cluster in k8s that automatically replicates, upgrades, backs up, and self-heals. With CNPG's relatively
recent declarative CRDs for databases and users, I was even able to automate provisioning for things that use cdk8s with some
help from [External Secrets Operator](https://external-secrets.io/latest/). [Check it out!](https://github.com/bjschafer/k8s-generators/blob/main/apps/postgres/database-provisioning.ts)

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

---

[^1]: OK, not all. I recently had to move my K8S control plane nodes' root disk onto local SSD. On Ceph without `cache=unsafe`, the commit latency was enough to cause issues during bursty things like backups. On Ceph *with* `cache=unsafe`, I had two etcd servers fail due to corrupted writes.
