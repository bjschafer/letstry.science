---
title: Configuring multiple routers & services with Traefik
date: 2020-06-28T20:14:42-06:00
draft: false
toc: false

description: Quick note on configuring a single Docker container that needs to expose multiple ports using Traefik.

taxonomies:
    tags:
      - homelab
      - traefik
      - docker
---

Quick note on configuring a single Docker container that needs to expose multiple ports using Traefik. For
this example, I'm using [Ubooquity](https://vaemendis.net/ubooquity/) as it uses a separate port for admin
that I wanted to just route on the same domain.

```yaml
version: "3.7"
services:
  ubooquity:
    image: linuxserver/ubooquity
    container_name: ubooquity
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Chicago
      - MAXMEM=1024 # MB
    labels:
      - traefik.http.routers.ubooquity.rule=Host(`ubooquity.example.com`)
      - traefik.http.routers.ubooquity.service=ubooquity
      - traefik.http.services.ubooquity.loadbalancer.server.port=2202
      - traefik.http.services.ubooquity.loadbalancer.server.scheme=http
      - traefik.http.routers.ubooquity.tls=true
      - traefik.http.routers.ubooquity.tls.certresolver=default
      - traefik.http.routers.ubooquity.tls.domains[0].main=example.com
      - traefik.http.routers.ubooquity.tls.domains[0].sans=*.example.com

      - traefik.http.routers.ubooquity-admin.rule=Host(`ubooquity.example.com`) && (PathPrefix(`/admin/`) || PathPrefix(`/admin-res/`))
      #- traefik.http.routers.ubooquity-admin.middlewares=admin-stripprefix
      #- traefik.http.middlewares.admin-stripprefix.stripprefix.prefixes=/admin
      - traefik.http.routers.ubooquity-admin.service=ubooquity-admin
      - traefik.http.services.ubooquity-admin.loadbalancer.server.port=2203
      - traefik.http.services.ubooquity-admin.loadbalancer.server.scheme=http
      - traefik.http.routers.ubooquity-admin.tls=true
      - traefik.http.routers.ubooquity-admin.tls.certresolver=default
      - traefik.http.routers.ubooquity-admin.tls.domains[0].main=example.com
      - traefik.http.routers.ubooquity-admin.tls.domains[0].sans=*.example.com
    networks:
      - lbclients
    ports:
      - 2202/tcp
      - 2203/tcp
    volumes:
      - ./data/config:/config
      - /mnt/Media/Ebooks:/books
    restart: unless-stopped

networks:
  lbclients:
    external: false
    name: loadbalancer
```

Note the commented out middleware lines. Before I changed the reverse proxy config to not use `ubooquity`, I
needed the middlewares to strip out the path prefixes.
