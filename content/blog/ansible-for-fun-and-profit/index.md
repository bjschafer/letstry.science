---
title: "Ansible for fun and profit!"
date: 2016-10-08T22:14:04-05:00
draft: false
toc: false

taxonomies:
    tags:
      - homelab
      - ansible
---

Let's face it, maintaining your awesome homelab is exhausting! All those hosts, logins, configurations...blech! It's enough to make anybody's head spin. I fully understand why companies and people who do this because they must use configuration management...why can't we do the same?
We can! After poking around the various options (Salt, Puppet, Chef...), I settled on Ansible. Why Ansible?

1. It's pretty lightweight
2. It's open source, written in Python, and maintained by Red Hat. It's thus pretty hackable.
3. Its configurations are written in yaml, which is pretty well-known. You don't have to learn a DSL just to use it.
4. It's agentless: you don't have to install an agent on each computer you want to manage. It works over ssh instead. Much less overhead and easier to use (imo).
To facilitate this, I wrote a "playbook" (a collection of related tasks in Ansible) to create a local user on machines I wanted to manage and set up passwordless ssh for it.

```yaml
---
- hosts: all
  become: yes
  become_method: su
tasks:
  - name: install sudo
    package: name=sudo state=latest
    when: "'debian' in group_names"
- name: create local ansible user for future use
    user: name=ansible comment="Ansible User" generate_ssh_key=yes ssh_key_bits=2048 ssh_key_file=.ssh/id_rsa
- name: add user@ansiblehost to ansible's authorized keys
    lineinfile: dest=/home/ansible/.ssh/authorized_keys create=yes line="<your user's ssh pubkey>"
- name: give ansible passwordless sudo
    lineinfile: dest=/etc/sudoers line="ansible ALL=(ALL) NOPASSWD{{ ':' }} ALL"
    when: "'debian' in group_names"
- name: give ansible passwordless sudo on freebsd
    lineinfile: dest=/usr/local/etc/sudoers line="ansible ALL=(ALL) NOPASSWD{{ ':' }} ALL"
    when: "'freebsd' in group_names"
```

Then, add your hosts to `/etc/ansible/hosts` (btw, store this in version control and simlink it in). You can run this playbook with `ansible-playbook ansible_user.yml --user=root -k` and provide it your password. Substitute root with any user you have that can currently ssh in.

Now that you've set up an ansible user and given it passwordless ssh, you can do useful things! Here's a quick and easy playbook to help lock things down a bit.

```
---
- hosts: debian:freebsd
  remote_user: ansible
  become: yes
tasks:
  - name: set domain lookup/search path
    lineinfile: dest=/etc/resolv.conf line="domain domain.tld"
- name: set local nameserver
    lineinfile: dest=/etc/resolv.conf line="nameserver 10.0.0.1"
    when: "'dmz' not in group_names"
- name: set dmz nameserver
    lineinfile: dest=/etc/resolv.conf line="nameserver 10.0.4.1"
    when: "'dmz' in group_names"
- name: set backup opendns nameserver
    lineinfile: dest=/etc/resolv.conf line="nameserver 208.67.222.222"  
- name: turn off ssh pw login
    lineinfile: dest=/etc/ssh/sshd_config regexp="^PasswordAuthentication" line="PasswordAuthentication no" state=present
- name: turn off root ssh login
    lineinfile: dest=/etc/ssh/sshd_config regexp="^PermitRootLogin" line="PermitRootLogin without-password" state=present
notify: Restart ssh
handlers:
  - name: Restart ssh
    service: name=ssh state=restarted
```

This sets DNS properly for your network (includes an example of how to set it for a different segment, just put the appropriate hosts in your Ansible hosts file). It then disables password ssh authentication (thus requiring certs) and also ensures that root cannot ssh in with a password.
This one's easy - `ansible-playbook network_settings.yml` and watch as your lab buzzes to life with everything you tell it to do!

Another great idea which I won't expand on right now is to set up your new servers/VMs using Ansible. Install a base install of your OS of choice, and then write a playbook to configure and install it. It takes a bit of planning and forethought, but in the event you ever have to rebuild it, it takes almost zero touch!

