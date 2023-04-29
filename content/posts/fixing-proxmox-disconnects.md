---
title: Fixing Proxmox cluster disconnects
date: 2017-06-05T22:22:22-05:00
draft: false
categories:
  - homelab
tags:
  - proxmox
toc: false
---

I have a cluster of 3 hosts running Proxmox as a virtualization platform. They provide a total of 28 vCPUs, 72GB of memory, and ~10TB usable storage. I went with Proxmox because I have a mix of CPU types in these boxes (1x Intel C2750, 1x Xeon E5649, 1x FX-8350), and KVM supports migration between Intel and AMD procs.
Pretty much ever since setting this cluster up, I noticed that the hosts would disconnect from each other according to the web interface, but the hosts and any VMs on them would remain accessible otherwise. It looked a little something like this:

![Screenshot_20170604_222652](/content/images/2017/09/Screenshot_20170604_222652.png)

This was unfortunate, to say the least. It meant HA was useless, since the cluster would lose quorum sometimes several times an hour, try to migrate all the VMs elsewhere, freak out, and reboot. It was fine for manual migrations still, but it was just driving me up the wall.

After doing some research, I found out that Proxmox's Corosync daemon relies heavily on multicast. So, I set up an isolated VLAN on my networking gear and added an OVS IntPort using the Proxmox management GUI on that VLAN. For some reason, it took 2 reboots to activate the new interface, but soon - there it was.

Then, I followed this excellent guide on the Proxmox wiki to migrate the cluster management network over to my new VLAN.

The last step to get everything working properly was to (counter-intuitively) disable IGMP snooping on my Unifi Switch. Since these three nodes are in their own isolated VLAN, it should be fine.

With both of those things in place, I brought things up, enabled HA, and everything works swimmingly!
