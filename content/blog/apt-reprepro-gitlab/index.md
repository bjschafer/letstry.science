+++
title = "Hosting your own apt repo with reprepro and GitLab"
date = 2025-08-16T12:24:20-05:00
description = """\
    Have you ever wondered about hosting your own apt repository? Looked at all the options \
    and thought they're woefully complex and hard to piece together? If so, read on! I'll \
    show you how I managed to host my own on a small server using reprepro \
    and publish to it from GitLab CI.\
"""

[taxonomies]
tags = [
  "homelab",
  "gitlab",
  "debian",
]
+++

# Intro

Have you ever wondered about hosting your own apt repository? Looked at all the options
and thought they're woefully complex and hard to piece together? If so, read on! I'll
show you how I managed to host my own on a small server using [reprepro](https://wiki.debian.org/DebianRepository/SetupWithReprepro)
and publish to it from GitLab CI.

# Tools

The first question you might be wondering is, if I'm already running GitLab, why not just use the
[GitLab-provided apt repo](https://docs.gitlab.com/user/packages/debian_repository/)? Frankly, because
it's not good. The documentation is poor, it doesn't work as it should, and even GitLab say it's not ready
for production use.

What about tools like Aptly or Pulp? Those are just fine but are pretty overkill if all you want is to host
a few self-published packages.

# Reprepro Setup

The [upstream guide](https://wiki.debian.org/DebianRepository/SetupWithReprepro) is quite good and I recommend
following it. I'll excerpt it here along with a few notes / recommendations.

Install your prereqs:

```bash
apt install -y reprepro nginx
```

Make a user and a location for your repo:

```bash
useradd -r -s /bin/bash -m repo
mkdir -p /srv/repo/conf
chown repo:repo /srv/repo
```

Add the following to the beginning of `~repo/.bashrc`. This makes it so we don't have to be in the
base directory when running `reprepro` commands.

```bash
export REPREPRO_BASE_DIR='/srv/repo'
```

From here on out, do all following steps as the `repo` user (`su - repo`):

Generate a certificate. For homelab use I'd recommend not specifying a passphrase.

```bash
gpg --gen-key

# Grab the fingerprint

gpg --list-secret-key
```

Copy the long string on the second line. That's your _key fingerprint_, you will need it for the next couple steps.

Create a keyring file:

```bash
gpg --export-options export-minimal --export <fingerprint> \
    > /srv/repo/keyring
```

## Configuring distributions

Now, make a file at `/srv/repo/conf/distributions`. It should follow this template:

```
Codename: <release-name>
Suite: <release-pseudonym>
Architectures: source i386 amd64 <...>
Components: main <...>
Contents:
SignWith: <fingerprint>
Origin: <Your project name>
Label: <Your project name>
Description: <Your project description>
```

From the upstream:

> Codename: is e.g. trixie or sid. For a list of common values, do debian-distro-info --all. This is not supposed to change - if you need another codename, add a new section.
>
> Suite: is e.g. stable or testing. Unlike the previous name, this can change when new releases come out.
> 
> Architectures: is a space-separated list of Debian architecture names.
> 
> Contents: is a list of zero or more flags. Including this but leaving it empty causes reprepro to generate Contents files, which are used by apt-file. This will make updates slower, but is valuable to apt-file users.
> 
> SignWith: is the signing fingerprint from Generate a PGP certificate, above.
> 
> Origin:, Label: and Description: are free-form text displayed to the user or used for pinning. For examples, do e.g. grep -C6 Label: /var/lib/apt/lists/*_InRelease.

or, if you'd like to see my example:

```
Codename: trixie
Suite: trixie-stable
Architectures: amd64 arm64
Components: main
SignWith: <fingerprint>
Origin: Super Cool Packages
Label: Super Cool Packages
Description: Packages from me

Codename: noble
Suite: noble-stable
Architectures: amd64 arm64
Components: main
SignWith: <fingerprint>
Origin: Super Cool Packages
Label: Super Cool Packages
Description: Packages from me
```

You'll note that the suite incorporates the release codename, which isn't called out in the upstream docs. This is
because each distribution must have a unique suite.

## Creating repo

Now, we can actually do the setup:

```bash
# create symlinks from Suites to Codenames:
reprepro createsymlinks

# Update metadata in dists/:
reprepro export
```

If you already have a package you want to install (or just want to do a quick test):

```bash
reprepro includedeb <release-name> <package>.deb

# e.g.
reprepro includedeb trixie-stable amtool.deb
```

## nginx config

You want to be able to actually access this from other machines, right? Thought so. That's what we installed
`nginx` for earlier. You'll need to become the root user for this bit.

### On TLS
    
This setup serves your repository over plain HTTP. Debian says that this is fine for an apt
repository since the packages are all PGP-signed, and the signing key should be shared over
secure channels.

For ease of setup, we do *not* serve the signing key over secure channels. For homelab use, this
should be reasonable. You can instead set up TLS using e.g. `certbot`, or configure the key on your
clients in a different way.

---

Create the `nginx` config file at `/etc/nginx/sites-available/repo`:

```
server {
    listen 80;
    server_name apt.example.com ;          # Change to your hostname

    root /srv/repo;
    index index.html;

    # Security: deny access to anything outside dists/ & pool/, plus keyring
    location ~ ^/(?!dists|pool|keyring).* {
        deny all;
    }

    # Serve Release files uncompressed *and* compressed
    location ~ /dists/.*/(Release|InRelease|Release.gpg)$ {
        add_header Cache-Control "public, max-age=300";
    }

    # Serve Packages files compressed
    location ~ /dists/.*/binary-.*/Packages(\.gz|\.bz2)?$ {
        add_header Cache-Control "public, max-age=300";
    }

    # Serve .deb packages
    location ~ /pool/.*\.deb$ {
        add_header Cache-Control "public, max-age=86400";
    }

    location /keyring {
        add_header Cache-Control "public, max-age=86400";
    }

    # Auto index for easy browsing (optional)
    autoindex on;
    autoindex_exact_size off;
}
```

Then, enable this site and reload `nginx`:

```bash
ln -s /etc/nginx/sites-available/repo /etc/nginx/sites-enabled/
systemctl reload nginx
```

# Configuring Repo

Now, to actually configure a client to use your repo, it's fairly straightforward. You can do so manually
like so:

```bash
wcurl -o /usr/share/keyrings/apt-my-org.pgp http://apt.example.com/keyring

cat <<EOF >/etc/apt/sources.list.d/my-apt.list
deb [signed-by=/usr/share/keyrings/apt-my-org.pgp] http://apt.example.com/ trixie-stable main

apt update
```

If you want to automate it, here's an Ansible snippet to make your life easier:

```yaml
- name: "[repo] install prereqs"
  ansible.builtin.package:
    name:
      - gpg
      - python3-debian
    state: present
  tags: [repo]
  when:
    - "ansible_os_family == 'Debian'"
- name: "[repo] add internal debian repo"
  ansible.builtin.deb822_repository:
    name: my
    types: deb
    uris: http://apt.example.com/
    suites: '{{ ansible_distribution_release }}-stable'
    components: main
    signed_by: http://apt.example.com/keyring
  tags: [repo]
  when:
    - "ansible_os_family == 'Debian'"
```

# GitLab Setup

Now comes the fun part. We need to teach GitLab CI how to build and publish `.deb` packages to our repo.

Start by creating an ssh keypair that can be used in CI to auth to the repo user:

```bash
ssh-keygen -t ed25519 -C 'repo CI builder' -f repo-key
```

Put the *public* key in `~repo/.ssh/authorized_keys`. Make sure it's owned by `repo` and has mode `0600`.

Take the *private* key and create a GitLab CI variable of type **File**, and name it `APT_SSH_PRIVATE_KEY`.
Make sure that there is an empty line at the bottom of the variable's contents.

If you're looking to build from multiple places, I'd set the variable on a group level so that it can propagate
down to all child projects.

## Publishing template

Now to create the pipeline config. I use [CI/CD Components](https://docs.gitlab.com/ci/components/). I have one generic
`ci-components` repo that's enabled as a CI catalog project. In there, I have an `nfpm` component
that can handle building a package and publishing it. It looks like so (irrelevant bits omitted):

```yaml
spec:
  inputs:
    nfpm-image:
      description: The Docker image used for nfpm, including tag
      default: ghcr.io/goreleaser/nfpm:latest
    nfpm-packager:
      description: which packager implementation to use
      default: deb
      options:
        - apk
        - archlinux
        - deb
        - ipk
        - rpm
    nfpm-config:
      description: config file to be used
      default: nfpm.yaml
    nfpm-target:
      description: where to save the generated package (filename, folder or empty for current folder)
      default: build/
    debian-distribution:
      description: Distribution to publish to
      default: trixie
    apt-host:
      description: Hostname of apt repo host
      default: apt.example.com
  # expects APT_SSH_PRIVATE_KEY file-type variable
---
# default workflow rules: Merge Request pipelines
workflow:
  rules:
    # prevent MR pipeline originating from production or integration branch(es)
    - if: '$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ $PROD_REF || $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ $INTEG_REF'
      when: never
    # on non-prod, non-integration branches: prefer MR pipeline over branch pipeline
    - if: '$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS && $CI_COMMIT_REF_NAME !~ $PROD_REF && $CI_COMMIT_REF_NAME !~ $INTEG_REF'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*tag(,[^],]*)*\]/" && $CI_COMMIT_TAG'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*branch(,[^],]*)*\]/" && $CI_COMMIT_BRANCH'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*mr(,[^],]*)*\]/" && $CI_MERGE_REQUEST_ID'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*default(,[^],]*)*\]/" && $CI_COMMIT_REF_NAME =~ $CI_DEFAULT_BRANCH'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*prod(,[^],]*)*\]/" && $CI_COMMIT_REF_NAME =~ $PROD_REF'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*integ(,[^],]*)*\]/" && $CI_COMMIT_REF_NAME =~ $INTEG_REF'
      when: never
    - if: '$CI_COMMIT_MESSAGE =~ "/\[(ci skip|skip ci) on ([^],]*,)*dev(,[^],]*)*\]/" && $CI_COMMIT_REF_NAME !~ $PROD_REF && $CI_COMMIT_REF_NAME !~ $INTEG_REF'
      when: never
    - when: always

# test job prototype: implement adaptive pipeline rules
.test-policy:
  rules:
    # on tag: auto & failing
    - if: $CI_COMMIT_TAG
    # on ADAPTIVE_PIPELINE_DISABLED: auto & failing
    - if: '$ADAPTIVE_PIPELINE_DISABLED == "true"'
    # on production or integration branch(es): auto & failing
    - if: '$CI_COMMIT_REF_NAME =~ $PROD_REF || $CI_COMMIT_REF_NAME =~ $INTEG_REF'
    # early stage (dev branch, no MR): manual & non-failing
    - if: '$CI_MERGE_REQUEST_ID == null && $CI_OPEN_MERGE_REQUESTS == null'
      when: manual
      allow_failure: true
    # Draft MR: auto & non-failing
    - if: '$CI_MERGE_REQUEST_TITLE =~ /^Draft:.*/'
      allow_failure: true
    # else (Ready MR): auto & failing
    - when: on_success

variables:
  NFPM_IMAGE: $[[ inputs.nfpm-image ]]
  NFPM_PACKAGER: $[[ inputs.nfpm-packager ]]
  NFPM_CONFIG: $[[ inputs.nfpm-config ]]
  NFPM_TARGET: $[[ inputs.nfpm-target ]]
  DEBIAN_DISTRIBUTION: $[[ inputs.debian-distribution ]]
  APT_HOST: $[[ inputs.apt-host ]]

stages:
  - build
  - package-build
  - publish

.nfpm-base:
  image:
    name: $NFPM_IMAGE
    entrypoint: [""]

nfpm-build:
  extends: .nfpm-base
  stage: package-build
  script:
    - mkdir -p "$(dirname $NFPM_TARGET)"
    - nfpm package --packager "$NFPM_PACKAGER" --config "$NFPM_CONFIG" --target "$NFPM_TARGET"
  artifacts:
    name: "$CI_JOB_NAME artifacts from $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG"
    expire_in: 1 day
    paths:
      - $NFPM_TARGET

nfpm-publish-deb:
  stage: publish
  image:
    name: public.ecr.aws/docker/library/alpine:latest
    entrypoint: [""]
  dependencies:
    - nfpm-build
  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p -m 0700 "$HOME/.ssh"
    - ssh-keyscan "$APT_HOST" > "$HOME/.ssh/known_hosts"
    - chmod 0600 "$APT_SSH_PRIVATE_KEY"
  script:
    # copy file
    - >
      scp
      -i "$APT_SSH_PRIVATE_KEY"
      "build/${PACKAGE_NAME}-${VERSION}-${RELEASE}-${ARCH}.deb"
      "repo@${APT_HOST}:/tmp/$PACKAGE_NAME.deb"
    # register package
    - >
      ssh
      -i "$APT_SSH_PRIVATE_KEY"
      "repo@${APT_HOST}"
      --
      reprepro includedeb "${DEBIAN_DISTRIBUTION}-stable" "/tmp/${PACKAGE_NAME}.deb"
```

Since it's a CI component, in order to use it as such you have to publish a release for it from CI. See how
to do so in the official docs linked above.

## Example pipeline

Now, here's a pipeline that uses this to build and publish the `amtool` binary:

```yaml
include:
  - component: $CI_SERVER_FQDN/core/ci-components/nfpm@~latest

variables:
  ARCH: amd64
  VERSION: 0.28.1 # bump this to get newer upstream version
  RELEASE: "1"    # bump this if you've changed something e.g. config
  NFPM_TARGET: build/amtool-$VERSION-$RELEASE-$ARCH.deb
  PACKAGE_NAME: amtool
  DISTRIBUTION: trixie
  COMPONENT: main

# this goes out and fetches the binary we need, putting it in place for the build in the next stage
fetch:
  stage: build
  image:
    name: public.ecr.aws/docker/library/alpine:latest
    entrypoint: [""]
  before_script:
    - apk add --no-cache curl tar
  script:
    - mkdir -p build
    - curl -fsSL -o build/alertmanager-${VERSION}-${ARCH}.tar.gz https://github.com/prometheus/alertmanager/releases/download/v${VERSION}/alertmanager-${VERSION}.linux-${ARCH}.tar.gz
    - tar -xzf build/alertmanager-${VERSION}-${ARCH}.tar.gz -C build/ --strip-components=1
  artifacts:
    name: "Fetched am binaries from $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG"
    expire_in: "1 day"
    when: always
    paths:
      - build
  parallel:
    matrix:
      - ARCH:
        - amd64
        - arm64

nfpm-build:
  parallel:
    matrix:
      - ARCH:
        - amd64
        - arm64
        DEBIAN_DISTRIBUTION:
        - trixie
        - noble

nfpm-publish-deb:
  stage: publish
  parallel:
    matrix:
      - ARCH:
        - amd64
        - arm64
        DEBIAN_DISTRIBUTION:
        - trixie
        - noble
```

...and boom, you have automated multi-arch multi-distribution builds that get published to your own internal
apt repo. Enjoy and good luck!

## Alternate option

I ran into issues with `parallel.matrix` and simultaneous uploads failing because the reprepro database was locked.

I could've fixed this with a little random wait, but there's no engineering like over-engineering!

So I wrote [reprepro-api](https://github.com/bjschafer/reprepro-api). Directions are in the readme.
