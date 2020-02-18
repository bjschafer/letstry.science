---
title: Configuring LDAP auth for Traefik (and more!)
date: 2020-02-17T19:56:42-06:00
draft: false
categories:
  - homelab
tags:
  - traefik
  - nginx
  - ldap
toc: false
---

As it stands now, my setup for web-accessible Docker-hosted sites is a bit convoluted. Traffic from the gateway flows into a bastion host in my DMZ. This is a tiny box running Nginx acting as a reverse proxy. There's a hole in the firewall poked in from this bastion host to the Docker host running on my internal network. Traefik runs in Docker and provides SSL termination among other things.

This guide assumes that you have one or more Docker containers that you want to put behind LDAP authentication. It also assumes you have an already-working Traefik setup. Also, if you're doing something weird like I am, I won't provide further details but I *will* provide the necessary Nginx config bits to ensure this works properly.

## Set up Authelia container

I like to use `docker-compose` to make things easier. Here's my cleaned-up `docker-compose.yaml` for Authelia:

```yaml
version: "3.7"
services:
  authelia:
    image: authelia/authelia
    container_name: authelia
    environment:
      - TZ=America/Chicago
    labels:
      - traefik.http.routers.authelia.rule=Host(`login.example.com`)
      - traefik.http.services.authelia.loadbalancer.server.port=9091
      - traefik.http.services.authelia.loadbalancer.server.scheme=http
      - traefik.http.routers.authelia.tls=true
      - traefik.http.routers.authelia.tls.certresolver=default
      - traefik.http.routers.authelia.tls.domains[0].main=example.com
      - traefik.http.routers.authelia.tls.domains[0].sans=*.example.com
    networks:
      - lbclients
    ports:
      - 9091
    volumes:
      - ${PWD}/data/config.yml:/etc/authelia/configuration.yml:ro
      - ./data/data:/var/lib/authelia
    restart: unless-stopped

networks:
  lbclients:
    external: false
    name: loadbalancer
```

Authelia is configured separately using its own YAML file. That's the `config.yml` file that we mount into the container. They provide a [template here](https://github.com/authelia/authelia/blob/master/config.template.yml) that's pretty well explained. Just a couple quick notes.

Here's a snippet where AD is the backend LDAP:

```yaml
<snip>
 ldap:
    # The url to the ldap server. Scheme can be ldap:// or ldaps://
    url: ldap://dc.example.com
    # Skip verifying the server certificate (to allow self-signed certificate).
    skip_verify: false
    # The base dn for every entries
    base_dn: dc=example,dc=com
    # An additional dn to define the scope to all users
    additional_users_dn: cn=users
    # The users filter used to find the user DN
    # {0} is a matcher replaced by username.
    # 'cn={0}' by default.
    users_filter: (sAMAccountName={0})
    # An additional dn to define the scope of groups
    additional_groups_dn: cn=users
    # The groups filter used for retrieving groups of a given user.
    # {0} is a matcher replaced by username.
    # {dn} is a matcher replaced by user DN.
    # {uid} is a matcher replaced by user uid.
    # 'member={dn}' by default.
    groups_filter: (&(member={dn})(objectclass=group))
    # The attribute holding the name of the group
    group_name_attribute: cn
    # The attribute holding the mail address of the user
    mail_attribute: mail
    # The username and password of the admin user.
    user: CN=svcBindAccount,CN=Users,DC=example,DC=com
    # This secret can also be set using the env variables AUTHELIA_AUTHENTICATION_BACKEND_LDAP_PASSWORD
    password: bindAcctPasswordHere
</snip>
```

Unless you already have a MySQL server somewhere, you'll want to yoink that section and just uncomment the `local` section under `storage`. Likewise, if you don't have Redis, that's fine too -- just remove that section.

Finally, I set `default_policy` to `one_factor`. Then, I can apply the config down below only to the containers I care about, and not specify any further rules.

## Set up Traefik

Actually, nothing special at all needs to be done, assuming you have a working Traefik config and a way to provide certs for all the relevant domain names.

## Set up backend container

Again, assuming you have an otherwise-working backend container, you can simply add the following labels to the `labels` section of your `docker-compose.yaml` file.

```yaml
- traefik.http.routers.yourapp.middlewares=authelia
- traefik.http.middlewares.authelia.forwardAuth.address=http://authelia:9091/api/verify?rd=https://login.example.com
```

Assuming you kept the Authelia container's name as `authelia`, you should only need to change the domain at the very end (`rd=...`).

## Ngnix config

Again, this is only relevant if you're doing the weird thing I'm doing. Setting up Authelia to work with Nginx is outside the scope of this post (although you may find [this documentation](https://github.com/authelia/authelia/blob/master/docs/proxies/nginx.md) to be useful).

All you'll want to do is set up a snippet (mine go in `/etc/nginx/snippets`), call it `proxy.conf`. Its contents should look like this:

```
proxy_set_header        Host $host;
proxy_set_header        X-Real-IP $remote_addr;
proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header        X-Forwarded-Proto $scheme #;proxy_http_version 1.1;
proxy_set_header        X-Forwarded-Host $http_host;
proxy_set_header        X-Forwarded-Uri $request_uri;
proxy_set_header        X-Forwarded-Ssl on;
proxy_set_header        Upgrade $http_upgrade;
proxy_set_header        Connection "upgrade";

proxy_http_version      1.1;
proxy_cache_bypass      $cookie_session;
proxy_no_cache          $cookie_session;
proxy_buffers           64 256k;

proxy_buffering         off;
client_max_body_size    0;
client_body_buffer_size 128k;

send_timeout 5m;
proxy_read_timeout 360;
proxy_send_timeout 360;
proxy_connect_timeout 360;
```

Then, in your site's config, under the `location` block, add `include snippets/proxy.conf` and reload/restart the Nginx service.

Note that this only ensures that Nginx is sending the proper headers and such, and in no way configures Nginx to pass the auth. You'll want the above-linked documentation to do that. Traefik is doing all the heavy lilfting here.
