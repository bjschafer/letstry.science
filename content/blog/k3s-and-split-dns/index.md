---
title: K3S, split-horizon DNS, DNSSEC, and PowerDNS
date: 2022-05-30T14:11:42-05:00
draft: false
toc: false

taxonomies:
    tags:
      - homelab
      - k8s
      - dns
---

On my home K3S cluster, I was running into a string of weird DNS issues. Here's how I ended up fixing it.

## Architecture Overview

- 6-node K3S cluster (3x worker, 3x control plane)
- In-cluster DNS using standard off-the-shelf CoreDNS
- Intranet DNS provided by PowerDNS
    - 2 DNS servers
    - Each runs PDNS Authoritative (for internal zone `example.com` and reverse)
    - Each runs PDNS Recursor (for all other queries; forwards to auth for `example.com`)
- Internet DNS for `example.com` provided by Cloudflare **and DNSSEC-signed**.
- Note the split-horizon DNS using the same domain.

## Symptoms

I was mostly having issues resolving hosts in `example.com`. For example, if a pod were making a query to look for
`foo.example.com`, I would see in the PDNS logs queries for:
- `foo.example.com.svc.cluster.local`
- `foo.example.com.cluster.local`
- `foo.example.com.`

The first two would obviously return NXDOMAIN, which is as expected. However, the last one returned an empty result but with 1 additional.

## Red Herrings

Here's a handful of things I tried that ultimately didn't solve the problem (but may be useful avenues to explore
if you happen to see similar issues).

- DNS search suffixes and `ndots`. This didn't matter; K3S does the right thing here.
- `systemd-resolved` not playing nicely with K3S
    - K3S will take the host's `/etc/resolv.conf` for use by pods (especially CoreDNS).
    - To work around this, the most complete way would be to do something like:
    ```bash
    rm /etc/resolv.conf

    cat <<EOF >/etc/resolv.conf
    nameserver 10.0.10.100
    nameserver 10.0.10.101
    search example.com
    EOF
    ```
    and restart.
- CoreDNS doing something weird
    - You can temporarily change its behavior by editing the `coredns` configmap in `kube-system`.
    - Things you might want include `log` to log all queries and replacing the `forward` line with `forward . 10.0.10.100:53 10.0.10.101:53`.
    - Config reload is *supposed* to happen automatically but doesn't; restart the pod(s) with `kubectl rollout restart deployment coredns`

Other things you might find useful include running an ad-hoc pod in-cluster on which you can install tools like dig, and using the `+trace` option to dig.

## Solution

The problem ended up being because `example.com` was DNSSEC signed externally. For reasons I don't fully understand, CoreDNS (and seemingly *only* CoreDNS) cared.

To fix it, I added the following to the PDNS Recursor Lua config located at `/etc/powerdns/recursor.lua` on my machine:

```lua
addNTA("example.com", "Some comment, doesn't matter")
```

and restarted the pdns-recursor service.

This added a **N**egative **T**rust **A**nchor for `example.com` such that clients would expect unsigned (or bogus-signed) responses for records in `example.com`. Make sure you add it on all of your DNS servers!
