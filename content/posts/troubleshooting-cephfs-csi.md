---
title: "Troubleshooting CephFS CSI"
date: 2024-04-07T21:24:21-05:00
draft: false
category: homelab
tags:
    - k8s
    - ceph
summary: >-
    A quick writeup for a longstanding issue I've had that's rendered CephFS-backed PersistentVolumes useless on my
    home K8S cluster. libceph on worker nodes complains about mons speaking the wrong protocol and that wasn't a red
    herring, it was the key to the problem all along.
---

# Problem

I've been dealing with issues using CephFS-backed PVs on my home K8S cluster for a while now. The volumes provision just fine, but
when the first consumer pod goes to mount them, it'll get stuck in `ContainerCreating`.

Looking at events with `k describe pod $pod` or `k events`, I'd see instances of this:

```ini
Warning  FailedMount  97s (x5 over 13m)  kubelet  (combined from similar events): MountVolume.MountDevice failed for volume
    "pvc-cb59b36c-5f85-48cb-9ded-13e5bcb6dbac" : rpc error: code = Internal desc = an error (exit status 32) occurred while
    running mount args:
        [-t ceph 10.0.151.2:6789,10.0.151.3:6789,10.0.151.4:6789:/volumes/csi/csi-vol-b47a15f5-4f4b-4569-96c6-2e806c247194/57130aa7-bf84-4496-ba6d-16afc470fc61
        /var/lib/kubelet/plugins/kubernetes.io/csi/cephfs.csi.ceph.com/4f5d491b99cd8acfc66f7a546fd455dd05a07b3f9500d802aa321e2839a0e292/globalmount
        -o name=kubernetes,secretfile=/tmp/csi/keys/keyfile-2403993821,mds_namespace=k8s-cephfs,ms_mode=secure,_netdev]
    stderr: unable to get monitor info from DNS SRV with service name: ceph-mon
    2024-03-27T01:53:00.361+0000 7f0a183420c0 -1 failed for service _ceph-mon._tcp
    mount error: no mds server is up or the cluster is laggy
```

and looking at the node running that pod, I'd see the following in `dmesg`:

```ini
[2089407.275648] libceph: mon0 (2)10.0.151.2:6789 server is speaking msgr1 protocol
[2089408.299738] libceph: mon0 (2)10.0.151.2:6789 server is speaking msgr1 protocol
[2089409.579749] libceph: mon1 (2)10.0.151.3:6789 server is speaking msgr1 protocol
[2089409.831836] libceph: mon1 (2)10.0.151.3:6789 server is speaking msgr1 protocol
[2089410.348331] libceph: mon1 (2)10.0.151.3:6789 server is speaking msgr1 protocol
```

This had flummoxed me for a while and become one of those on-again-off-again troubleshooting issues that I seem to collect.
I'd mostly worked around the issue by no longer using CephFS volumes, which was subpar. Note that I am *not* using Rook -- I run
a Ceph cluster on my Proxmox hosts and make that available to my k8s cluster via CephFS and RBD CSI plugins deployed manually.

[Kagiing](https://kagi.com/)[^1] this issue yielded...nothing helpful. One thing I *can* tell you is this:

`stderr: unable to get monitor info from DNS SRV with service name: ceph-mon` **is a useless generic error**. This message
appears whenever *pretty much anything* has gone wrong with a Ceph CSI mount. If you see it, ignore it.

The `msgr1 protocol` message also seemed to be irrelevant, of course that port's speaking msgr1!

# Solution

Well, ackshually...the `msgr1 protocol` message was the key. You see, somehow the CephFS StorageClass had gotten an extra mount
option. It had `ms_mode=secure` set. That was causing it to _try_ to speak `msgr2` to the Ceph mons, which wouldn't answer `msgr2`
on the `msgr1` port.

The "somehow" bit is still a bit confusing. I don't know where it came from -- I'm pretty sure it was never part of the setup steps (not
that those are super concise, but I should put my PRs where my mouth is). The StorageClass definition in my cluster is in source control
and managed by ArgoCD. While of course most of the `storageclass.spec` fields are immutable after creation, the yaml file in my repo
did not have the `ms_mode=secure` mount option set! I would expect Argo to throw a hissy fit that a resource was out of sync, but that
didn't happen either.

So I ended up having to delete the StorageClass in-cluster and re-apply from the yaml file. Yes, this seems to be safe to do.
Yes, I'm a K8S expert. No, I'm not *your* K8S expert, don't delete random things in a cluster without a good backup strategy.

Incidentally, this fixed Velero backups for my couple of outstanding CephFS volumes that were mysteriously still working.

# Another Related Issue

I had a related issue some time ago with similar symptoms but different errors. I didn't think to note it down then, but it seems
relevant to at least call out in the hope it is a spark of insight for anyone else experiencing it.

It was more-or-less [this issue](https://github.com/rook/rook/issues/12843) which manifests similarly with the useless red herring
`stderr: unable to get monitor info from DNS SRV with service name: ceph-mon` message.

The problem here was in a removed `mountOption` that called for `debug`.


## ...and its solution

Remove all `mountOptions` from your CephFS StorageClass, and update your existing PVs using something like this:

```bash
#!/bin/bash

# Get a list of PVs with the csi-cephfs-sc saorage class. Adjust to your needs.
PVs=$(kubectl get pv -o=jsonpath='{.items[?(@.spec.storageClassName=="csi-cephfs-sc")].metadata.name}')

for pv in $PVs; do
  # Use kubectl patch to remove the spec.mountOptions field.
  kubectl patch pv $pv --type=json -p='[{"op": "remove", "path": "/spec/mountOptions"}]'
done
```

[Source](https://github.com/ceph/ceph-csi/issues/3927#issuecomment-1667294477)


# Summary

CephFS does not need any `mountOptions` on its StorageClass. Remove them. Remove them from your PVs if need be.

[^1]: Like Googling, but better in every way.
