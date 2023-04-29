---
title: Migrating from vSphere to KVM
date: 2020-04-20T21:22:42-05:00
draft: false
categories:
  - homelab
tags:
  - vmware
  - kvm
  - networking
toc: true
---

I recently acquired a new server to upgrade my aging HP Gen6 boxes. Since I was consolidating down from 2 to 1 host (but going up to 32 logical threads and 384GB memory), I figured the "HA" portion of VMware/vSphere/vCenter (I've entirely forgotten to mind my spheres & centers!) was less necessary. I could have switch to Proxmox, but where's the challenge in that!? Instead, I switched to plain Ubuntu 20.04 LTS with KVM/libvirt on top, along with ZFS for the disk images. This'll be my notes and lessons learned during the migration.

## ZFS configuration

Although libvirt in theory supports native `zvol` integration, in practice it's poorly documented. I also came across [some benchmark results](https://old.reddit.com/r/zfs/comments/86khhr/benchmarking_raw_image_vs_qcow2_vs_zvol_with_kvm/) that implied there may be performance penalties compared to raw or qcow2 images.

I toyed with creating a separate dataset for each disk to perhaps enhance snapshotability. But that would introduce additional management overhead, and there didn't appear to be any existing tools to assist with that. If something exists, I'd be happy to revisit!

I ended up wih just a single dataset for all VM disks. I'm still torn between raw and qcow2 images. My gut says raw images should perform better on ZFS since the filesystem is already doing copy-on-write, snapshots can be handled at the FS level, and ZFS will compress (and dedupe if enabled). It sounds like the performance difference is negligible and the qemu-native snapshots are convenient. Still looking for a compelling reason to choose one or the other.

## Networking configuration

Getting the networking stack to work with `netplan` was somewhat unintuitive. Here's an example config file showing you how to configure bonds, bridges, and VLANs. Note that as I have it configured, `eno{2..5}` are an LACP port channel on the switch, and enp144s0f1 is a trunk port.

```yaml
network:
  version: 2
  renderer: networkd
  bonds:
    bondmgt:
      interfaces:
        - eno2
        - eno3
        - eno4
        - eno5
      parameters:
        mode: 802.3ad
      addresses: [ 10.0.10.3/24 ]
  ethernets:
    eno2: {}
    eno3: {}
    eno4: {}
    eno5: {}
    enp144s0f1: {}
  bridges:
    brdmz:
      interfaces: [ vlan4 ]
    brserversstatic:
      interfaces: [ vlan10 ]
      addresses: [ 10.0.10.2/24 ]
      gateway4: 10.0.10.6
      nameservers:
        search: [ example.com ]
        addresses:
          - "10.0.10.200"
          - "10.0.10.201"
    brserversdhcp:
      interfaces: [ vlan11 ]
    brnetmgmt:
      interfaces: [ vlan99 ]
    brstorage:
      interfaces: [ vlan151 ]
      addresses: [ 10.0.151.2/24 ]
  vlans:
    vlan4:
      id: 4
      link: enp144s0f1
    vlan10:
      id: 10
      link: enp144s0f1
    vlan11:
      id: 11
      link: enp144s0f1
    vlan99:
      id: 99
      link: enp144s0f1
    vlan151:
      id: 151
      link: enp144s0f1
```

## Migrating using `virt-v2v`

I discovered `virt-v2v`, part of the `libguestfs` project, while I was planning this move. It was a lifesaver. I've used other conversion tools (VMware P2V, plain `qemu-img`), and this was significantly better.

Here's two ways of invoking it, depending on if you want to go through vCenter (ultimately using http/s as a transport for your virtual disks), or if you just want to convert from the `.vmx` and `.vmdk`s:

```bash
virt-v2v -ic vpx://administrator@vcenter.example.com/Datacenter/Cluster/host.example.com?no_verify=1 VMName -os libvirt_storage_pool -ip /tmp/pass
```

```bash
virt-v2v -i vmx /mnt/VMFS/VMname/VMname.vmx -os libvirt_storage_pool -of raw # or qcow2, see above...
```

### Windows-specific notes

To successfully and easily migrate Windows VMs, you'll need a couple of things:

1. [virtio drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) -- these are easy to find. If the link goes dead, they're provided by the fine folks at Fedora. Put this in `/usr/share/virtio-win/virtio-win.iso`, no need to extract it or anything.
2. [rhsrvany.exe](https://github.com/archerslaw/chpwdservice/raw/master/rhsrvany.exe) -- while open source, it's difficult to find a compiled version, and worse to compile on your own due to lack of directions. I think it requires GNU automake/autoconf toolchains but Windows libraries (so must be compiled on Windows). I linked a compiled version I found that worked. If it goes dead, may the odds be ever in your favor. Put this in `/usr/share/virt-tools`

### Post-migration notes

There were a couple of things I didn't find solutions for when I looked (admittedly, I didn't look particularly hard...):

1. Network adapters. I had to edit the VM's config to use the appropriate bridge (see above)
2. Disk config. I had to change the disk XML per disk, per VM to look similar to this. When you migrate, you'll see what I mean:
```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='raw' cache='none'/>
  <source file='/flash/VMs/Disks/VMname-sda' index='1'/>
```
3. CPU config. It defaults to an emulated CPU, which is great if you intend to live migrate VMs. Since I'm rocking a single host, I decided to reduce the performance penalty and instead expose the host CPU model.
