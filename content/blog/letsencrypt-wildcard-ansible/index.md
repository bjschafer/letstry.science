---
title: LetsEncrypt wildcard + Ansible
date: 2018-07-09T21:39:22-05:00
draft: false
toc: false

taxonomies:
    tags:
      - homelab
      - ansible
      - letsencrypt
      - security
---

When LetsEncrypt announced the availability of wildcard certs, I knew I wanted in. In my homelab, in order to get SSL up and running, I'd been running Caddy, since it automagically gets a cert by doing DNS validation. However, that's an extra step that can complicate things. With a wildcard cert, I can even put SSL places I couldn't previously - such as my router and my FreeNAS box.

## How To Do

### Obtaining a wildcard cert

Due to popular demand, I'll provide some basic steps to get your wildcart cert. I'm assuming you're running Ubuntu or Debian, but you can probably adapt these directions as needed.

Prerequisites: `apt install python-pip python-dev build-essential python-yaml libffi5-dev`

**Remove any old certbots you may have installed** - `apt remove certbot`.

Finally, install stuff: `pip install certbot certbot-dns-$provider`. In my case, `$provider = cloudflare`

Then, create a file `/etc/letsencrypt/$provider.ini` and `chmod 600` it. Inside it looks something like

```ini
## Cloudflare API credentials used by Certbot
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234567
```

Then, you can run `certbot certonly --dns-$provider --dns-$provider-credentials /etc/letsencrypt/cloudflare.ini *.yourdomain.com` and it should be good!

Step 1 is of course to obtain your wildcard cert. I'll offer a few tips, but I'll leave this exercise up to the reader. I use the official latest `certbot`, but it does have to have support for your DNS provider - in my case, Cloudflare. The documentation for `certbot` should be able to get you up and running. Also, make sure you setup a cronjob to auto-renew the cert.

Next, since I run Ansible as a non-root user, I needed a way to get my user to read the otherwise unreadable certs. So, being slightly lazy, I did a `chgrp -R sudo /etc/letsencrypt/archive/$domain && chmod g+rx /etc/letsencrypt/archive/$domain && chmod g+r /etc/letsencrypt/archive/$domain/*`. Then, the files remain accessible to any user who's a member of `sudo`. Not super elegant, but it works.

Finally, the "hard" part - the Ansible playbook. To help out with this, I created a host group in my inventory - something like this:

```ini
[wildcard_hosts]
foo.cmdcentral.xyz      ssl_cert_path=/etc/pki/tls/certs/ca.crt ssl_key_path=/etc/pki/tls/private/ca.key
bar.cmdcentral.xyz      ssl_all_path=/etc/lighttpd/server.pem
baz.cmdcentral.xyz      ssl_cert_path=/etc/certificates/wildcard.crt ssl_key_path=/etc/certificates/wildcard.key
foowho.cmdcentral.xyz   ssl_cert_path=/etc/ssl/fullchain1.pem ssl_key_path=/etc/ssl/privkey1.pem
```

Then, here's the playbook. Note that if it fires the "Restart web servers" handler, it'll "error" on any web server that isn't running/doesn't exist. That's okay, though, as it'll still try and restart others, until it finds and restarts the one you want. Ansible experts, teach me how to do this better!

```yaml
---
- hosts: wildcard_hosts
  become: yes
  become_method: sudo

  tasks:
  - name: copy updated cert
    copy:
      backup: yes
      dest: "{{ ssl_cert_path }}"
      src: /etc/letsencrypt/live/$domain/fullchain.pem
    notify: Restart web servers
    when: ssl_cert_path is defined

  - name: copy updated private key
    copy:
      backup: yes
      dest: "{{ ssl_key_path }}"
      src: /etc/letsencrypt/live/$domain/privkey.pem
    notify: Restart web servers
    when: ssl_key_path is defined

  - name: copy concatenated cert + private key
    assemble:
      backup: yes
      dest: "{{ ssl_all_path }}"
      src: /etc/letsencrypt/live/$domain/
      remote_src: no
      regexp: "(fullchain\\.pem|privkey\\.pem)"
    notify: Restart web servers
    when: ssl_all_path is defined

  handlers:
  - name: Restart web servers
    service:
      name: "{{ item }}"
      state: restarted
    with_items:
      - nginx
      - apache2
      - httpd
      - lighttpd
    ignore_errors: yes
```

Finally, I set up a cronjob in my user's crontab (`crontab -e` as user) to run this playbook automatically, so hosts get the updated cert if it exists:

```
## m h  dom mon dow   command # every monday at 5am
  0 5  *   *   1 /usr/bin/ansible-playbook /home/bschafer/ansible/wildcard.yml > /home/bschafer/log/wildcard.log 2>&1
```
