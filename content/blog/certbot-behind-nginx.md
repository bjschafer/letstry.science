---
title: Running certbot behind nginx
date: 2020-04-27T22:27:27-05:00
draft: false
categories:
  - homelab
tags:
  - nginx
  - letsencrypt
toc: false
---

I've talked about my bizarre double-reverse-proxy approach before. Today I ran into an issue getting a real letsencrypt cert on a backend host. I just wanted to share the config -- this goes on the frontend nginx host. Assuming you're using the certbot-nginx plugin, no special config is needed on the backend.

```
rewrite ^(/.well-known/acme-challenge/.*) $1 break;
location ^~ /.well-known/acme-challenge {
                proxy_pass http://backend-host;
        }
```

This will get around the sane default http -> https redirects you've probbaly set up. Full config:

```
server {
        rewrite ^(/.well-known/acme-challenge/.*) $1 break;
        location ^~ /.well-known/acme-challenge {
                proxy_pass http://backend-host;
        }

        access_log /var/log/nginx/testing.log;
        error_log /var/log/nginx/testing.log debug;

        listen 80;
        server_name hostname.example.com;
        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl;
        server_name hostname.example.com;

        include snippets/ssl.conf;
        include snippets/logging.conf;

        location / {
                include snippets/proxy.conf;
                proxy_pass https://backend-host;
        }
}
```
