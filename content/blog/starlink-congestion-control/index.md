+++
title = "TCP Congestion Control on Starlink"
date = 2026-09-12

description = """\
    An unexpected transcode leads me to learn about TCP congestion algorithms, \
    and find an issue that's been lurking for my entire time on Starlink -- \
    over 2 years!
"""

[taxonomies]
tags = [
    "linux",
    "networking",
    ]

[extra]
katex = true
+++

## Introduction

{% admonition(type="note") %}
While I had some AI help tracking this down, the
entirety of this post is all slop from my human brain.
{% end %}

I run a Plex server at home. It's a fantastic way to manage and watch public-domain movies, and it even makes it nice
and easy to share those public-domain movies with friends! I newly invited a friend to my Plex server because they wanted
to watch one of those movies. However, I was surprised when they mentioned they had crazy buffering issues, until they
changed the quality to transcode down to 720p.

This was no 4K HDR video! The file's listed bitrate was 18mbps. I'm on Starlink (unfortunately, it's the only option
that lets me and my wife work from home where we live), and usually speedtest in the 50-60mbps range. Obviously, bitrate isn't
the only important number, and speedtests can lie, but there's enough overhead there that I would not have expected any issues.

## Metrics galore

Do I have metrics and monitoring? You _bet_ I do!

Did any of them help me here? Not really. I was able to confirm outside the speedtest windows that my 7d traffic peak was
69.8mbps so unless my wife's running secret speedtests, my bandwidth numbers are not just biased for speedtest servers.

So time to dig in on the Plex host:

```bash
$ nstat -az
TcpOutSegs               5838264
TcpRetransSegs             48197    # 0.83% of all segments were retransmitted
TcpExtTCPLostRetransmit    36394    # 75% of retransmits were THEMSELVES lost
TcpExtTCPTimeouts          41082    # retransmission timeout stalls, not fast-retransmit
```

OK, that's a wild amount of retransmits. Is the link healthy?

```bash
$ ip stats show dev ens18 | grep -A1 TX
    TX:   bytes packets errors dropped carrier collsns
     4901381782 5776032      0       0       0       0
```

Yes, it is. So this seems like it has to be upstream. Luckily the (transcoded) stream was still running, so I could poke at it a bit:

```bash
$ ss -tinm 'sport = :32400'
mss:1428 pmtu:9000 rtt:41.032/6.784 minrtt:23.002
cwnd:23 ssthresh:20
Send-Q 698125  notsent:665281
bytes_sent:8862094 bytes_retrans:21420 retrans:0/15
delivery_rate 5389488bps
```

(Ignore the path MTU, that's a red herring. I use jumbo frames on the LAN.)

Couple of things here stand out:

- congestion window (`cwnd`) > slow-start threshold (`ssthresh`) means this connection is in congestion avoidance.
- almost 2/3 of a meg (665281 bytes) is queued and not sent.

So it's trying to push data, but TCP is thinking we're congested??

## Anyone got Sudafed-TCP?

Starlink, as every astronomer knows, has a billion little satellites flitting about the sky \[citation needed\]. My dish gets regularly handed off between different satellites. When that handoff happens, there seem to be a few packets dropped here and there. This isn't particularly surprising, even the Starlink app will often show ping success somewhere in the 99.x range.

How does this relate?

### Congestion control

The kernel is in charge of avoiding congestion at the TCP layer. The default congestion control algorithm in the kernel is called [CUBIC](https://en.wikipedia.org/wiki/CUBIC_TCP), since 2.6.19.

CUBIC calculates its congestion window, or `cwnd`, with a cubic function triggered based on packet loss:

$$
cwnd = C(T - K)^3 + w_{max} \\\\
\text{where } K = \sqrt[3]{\frac{w_{max}(1-\beta)}{C}}
$$

Because packets are lost (or possibly delayed?) on satellite handoff, the packet goes through the fast retransmit path and then (because $\beta = 0.7$) reduces `cwnd` by 30%.

This is the gentle response to loss or congestion. The harsh response, in contrast, occurs when there's a retransmission timeout (`RTO`), which is when the retransmit timer fires (i.e. you didn't even get a duplicate ACK), or when the retransmitted segment is _also_ lost. In this case, CUBIC reduces `cwnd` to 1 segment.

We saw above that our round-trip time on this connection was ~41ms, which doesn't give the transmit window enough time to climb back up to normal before the next satellite handoff, so the flow is _always_ in congestion avoidance.

However, it turns out there's another congestion control algorithm! Enter [BBR](https://web.stanford.edu/class/cs244/papers/bbr.pdf). Instead of using _packet loss_ as a signal of congestion, BBR builds a model of the network from round-trip time and observed maximum bandwidth on the most recent set of outbound packets. And since both of those values are generally fine and unaffected by satellite handoff, the kernel no longer sees our link as congested, no longer drops the congestion window unnecessarily, and my streams no longer suffer.

Some data to back it up from 3 upload tests:

```bash
dd if=/dev/urandom of=40mb.bin bs=1M count=40
curl -o /dev/null -w '%{speed_upload}\n' -X POST --data-binary @40mb.bin \
    https://speed.cloudflare.com/__up
```

| Algorithm | Single-flow upload results |
|---|---|
| CUBIC | 13.8 / 14.2 / 14.6 Mbps |
| BBR | 46.0 / 53.4 / 57.3 Mbps |

## Rollout

It's recommended to not mix BBR and CUBIC on hosts on the same net as they may end up fighting with one other and starving hosts
CUBIC hosts that shouldn't be starved.

So, I added this to my lab-wide Ansible playbook to switch _all_ hosts to BBR:

```yaml
- name: "[BBR] Persist tcp_bbr module across reboots"
  ansible.builtin.copy:
    content: "tcp_bbr\n"
    dest: /etc/modules-load.d/tcp-bbr.conf
    owner: root
    group: root
    mode: "u=rw,g=r,o=r"
  when: ansible_facts['virtualization_type'] != 'lxc'

- name: "[BBR] Load tcp_bbr module"
  community.general.modprobe:
    name: tcp_bbr
    state: present
  when: ansible_facts['virtualization_type'] != 'lxc'

- name: "[BBR] Set bbr as the default congestion control"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_congestion_control
    value: bbr
    sysctl_file: /etc/sysctl.d/99-cmdcentral.conf
    sysctl_set: true
    reload: true
```
