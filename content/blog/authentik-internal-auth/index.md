+++
title = "Authentik with Source IP Auth Modifications"
date = 2024-11-06T19:57:10-06:00

description = """\
    How to configure an Authentik flow to conditionally execute stages based on client IP address. \
    In simpler terms, how do I not get prompted for 2FA/MFA when I'm at home?\
"""

[taxonomies]
tags = [
    "homelab",
    "networking",
    "security",
    ]
+++

# Introduction

I run [Authentik](https://goauthentik.io) in my homelab for SSO. It's actually super handy to have one set of credentials that can be
used to auth to pretty much everything I host. Once I started sharing services with my partner, and then later yet with friends and family,
it's so nice to say "oh, you want access to my [NetBox](https://github.com/netbox-community/netbox) instance? Granted, use the same creds
you use for all my services!" It also supports (although I haven't enabled) federating auth with a source such as Google Accounts, meaning
external users wouldn't even need separate creds for my services at all. That being said, pretty much everyone who does have access is
handy with a password manager so what's one more password.

One thing I wanted to do with Authentik is have it prompt for MFA credentials (Passkey or TOTP), but **only when accessed from the Internet**.
In other words, if I'm sitting at home, I don't need to provide MFA creds when logging in, but I do if I'm out and about on my phone.

I laid the groundwork for this in [Getting real IPs from behind ingress](@blog/getting-real-ips-from-behind-ingress/index.md).
After doing that, Authentik could see the real IP address of clients.

# Implementation

## Policy

Now, to actually wire this up in Authentik, start by going to **Customization > Policies** and **Create** a policy of type `Expression Policy`.
Give it a name (I chose `Is Internal`). Then, for the expression, paste in:

```python
internal_nets = [
    "10.0.3.0/24",
    "10.0.10.0/24",
    "10.0.11.0/24",
    "10.0.30.0/24"
]

return any([ak_client_ip in ip_network(net) for net in internal_nets])
```

`ak_client_ip` is a [policy variable](https://version-2024-10.goauthentik.io/docs/customize/policies/expression?utm_source=authentik#variables)
populated by Authentik that returns the client's real IP address. If for some reason it can't determine the client's IP, it returns
`255.255.255.255`.

From there, it's just Python (plus the [`ipaddress`](https://docs.python.org/3/library/ipaddress.html#ipaddress.ip_address) module from
the stdlib). We check if the client's IP is within any of the subnets we define as "internal".

Now, hit **Finish**.

## Binding

Now that the policy exists, we need to wire it up (which is called "binding" in Authentik). Go to **Flows and Stages > Flows**. Now, you
need to find the **Authentication** flow that you're using. If you haven't changed any defaults in this regard, it's most likely
**default-authentication-flow**. If in doubt, pop an incognito window and go to your Authentik instance. The flow name will be in the
URL path after `/if/flow/`.

Click the flow name to be brought into the flow overview page. Click the **Stage Bindings** tab. In here, click the `>` next to
`default-authentication-mfa-validation` (or your MFA stage). Then, click **Bind existing policy/group/user**. Choose the policy
we created in the previous section. Check **Negate result**[^1], and hit **Create**.

---

Now, go test it out! You should be able to log in from an incognito window on a machine in your internal network without getting prompted
for MFA, but you should be prompted if you, say, disconnect your phone from wifi and login using mobile data.

[^1]: We negate the result because we want the policy check to return true (and thus cause the stage to run) 
if the client IP is *not* internal.
