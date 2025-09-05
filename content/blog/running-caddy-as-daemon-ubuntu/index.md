---
title: Running Caddy as a daemon on Ubuntu
date: 2017-11-18T22:29:22-05:00
draft: false

taxonomies:
    tags:
      - homelab
      - linux
toc: false
---

These instructions work for me on Ubuntu 16.04.3. YMMV.

First, install Caddy by running `
curl https://getcaddy.com | bash -s personal hook.service,http.realip,tls.dns.cloudflare`. If you don't trust them (and you shouldn't!), `wget` the script first and inspect it before running.

Next, add a user for Caddy: `useradd -r -s /usr/sbin/nologin caddy`. Then, add a place to store config: `mkdir /etc/caddy && chown caddy:caddy /etc/caddy`. Finally, give Caddy a logfile: `touch /var/log/caddy.log && chown caddy:caddy /var/log/caddy.log`

## Caddyfile

Fire up your favorite editor (ahem, vi, ahem...) and create `/etc/caddy/Caddyfile`. Place the following into it:

```
your.fqdn.com

log /var/log/caddy.log

proxy / https://127.0.0.1:8443 { # change this to the port (and protocol) your service is running on
        websocket
        insecure_skip_verify # necessary if your proxy target is using a self-signed cert
        transparent
}
tls {
        dns cloudflare # if you use Cloudflare DNS like me, it'll automatically take care of the Let's Encrypt stuff, even if it's internal facing. Neat!
}

```

## Systemd unit file

This goes in `/lib/systemd/system/caddy.service`:

```ini
[Unit]
Description=Caddy reverse proxy
ConditionFileIsExecutable=/usr/local/bin/caddy
After=network-online.target
Wants=network-online.target

[Service]
StartLimitInterval=5
StartLimitBurst=10

# command to run to start
ExecStart=/usr/local/bin/caddy -conf=/etc/caddy/Caddyfile -email=<your-email@example.com> -root=/var/tmp -agree=true -log stdout

# user and group to run as, must exist
User=caddy
Group=caddy

# set environment variables (for https)
Environment=CLOUDFLARE_EMAIL=<your-cloudflare-email@example.com>
Environment=CLOUDFLARE_API_KEY=<your-cloudflare-api-key>
Environment=CADDYPATH=/etc/caddy

Restart=always
RestartSec=120

[Install]
WantedBy=multi-user.target
```

Run `systemctl daemon-reload` to get Systemd to recognize the new file. Then, let it start on boot with `systemctl enable caddy.service`. Finally, you can fire it up with `systemctl start caddy.service`

Enjoy automagic https!
