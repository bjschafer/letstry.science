---
title: "Getting real IPs from behind ingress"
date: 2023-07-30T20:50:31-05:00
draft: false
summary: A whirlwind dive into kube LoadBalancer services, and a lesson in how things are not always what they seem.
tags:
    - k8s
    - networking
---

# The Problem

I have [Authentik](https://goauthentik.io) running in my Kubernetes cluster. I wanted to be able to see clients' real IPs in the application.
Instead, all client IPs were showing as `10.42.0.0` in the app.

# _Not_ the Cause

I'll be honest here -- this problem had plagued me for _months_. I'd worked on it and rage-quit a few times. So here's a few things I
tried that didn't work:

- Digging into the application in question (Authentik), thinking that it wasn't correctly handling proxy host headers. But, it's
well-written and does do the right thing. I was also able to rule this out by checking other applications that I've run outside
of k8s in the past (e.g. Gitea) to see that they were exhibiting the same symptoms.
- Well then, surely it's the next layer up the stack, right? So I went way deep on Traefik, my ingress controller. I know that nginx
requires a couple extra lines of config to pass the real client IP, so figured some such would be needed on Traefik. That wasn't the
problem, although the major version break made it real hard to determine what's going on.

# The Cause

I'm ashamed to admit how long it took me to figure out what was really causing this behavior. I discovered [a link](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip)
deep inside one of many forum posts I read. The problem was in Kubernetes. Well, sort of.

To back up a little bit, ingress to my k8s cluster is implemented via a `Service` of type `LoadBalancer`. Since I'm on bare metal, I use
[kube-vip](https://kube-vip.io/) to implement `LoadBalancer`s. So the full traffic flow for an ingress request is:

```
(eyeball) -> (loadbalancer service) -> (ingress pods) --> (application)
```

...at least at a high level.[^1]

The problem is, that diagram is missing a few pieces of the request flow (which I didn't realize). You see, there's a very important
field on the `Service` spec I was overlooking: `externalTrafficPolicy`. And there was my dagger!

The default value for this field (and what I was using) is `Cluster`. `Cluster` will accept a request for the `Service` on any node in
the cluster and route the traffic to any ready endpoint in the cluster -- even if that's on another node. `Local`, on the other hand
assumes that you're handling your own load balancing and will only route the traffic to ready endpoints on the node the traffic
was received.

Because `externalTrafficPolicy: Cluster` is potentially proxying traffic to another node, it by design "obscures the client source IP and
may cause a second hop to another node"[^2]. Mic. Drop.

# The Fix?

This should be an easy fix, right? Set `externalTrafficPolicy: Local` on the ingress service and be done with it. But no, not quite.
Because if `kube-proxy` receives traffic for a service with `externalTrafficPolicy: Local` on a node that isn't home to a ready endpoint
for that service, it will **silently drop that traffic**.

So, any guesses what happened when I made that change, not realizing the caveat? Yeah, no, ingress traffic started timing out left and
right. Boy, was I mystified.

# Deep Dive

Time for some implementation details (of my setup)!

kube-vip is running as a `DaemonSet` across all nodes in my cluster. In addition to handling `LoadBalancer` services, it also provides
a VIP for the control plane. It's running in BGP mode, announcing routes to my router. At this time, all IPs under its control were
announced from all nodes.

Traefik, the ingress controller, is running as a multi-replica `Deployment`. I firmly believe ingress should _not_ be a `DaemonSet`, as
it shouldn't need to run on every cluster node (this is actually important at scale). It would stand to reason that making it a
`DaemonSet` would solve this problem, but 👎.

A detail I'd noticed quite some time ago and dismissed as benign but bemusing was that kube-vip had allocated the same IP for
all of the services it was in charge of. Since they happened to all be TCP-only, listen on different ports, and were using
`externalTrafficPolicy: Cluster`, this was fine.

However, with this new approach, that _couldn't_ work. We'd need to ensure that ingress was on a unique IP, so that kube-vip could
announce that only from nodes running ingress pods.

## kube-vip fights back

It turns out, [functionality to handle just that](https://kube-vip.io/docs/usage/kubernetes-services/#external-traffic-policy-kube-vip-v050)
was semi-recently added to kube-vip. So I set out to enable it. And...it seemed work. Except it was still announcing the new ingress
service IP from all nodes.

Not only that, but I ran into [this bug](https://github.com/kube-vip/kube-vip/issues/530) which caused the control-plane IP to now
get announced from _worker_ nodes. This meant that external control-plane access went down.

## Enter: MetalLB

[MetalLB](https://metallb.org/) is a project that seeks to solve many of the same problems that kube-vip solves. Seeing that MetalLB
[fully supports](https://metallb.org/usage/#traffic-policies) `externalTrafficPolicy: Local`, I decided to give it a try. I figured
I'd already mucked up my cluster enough, why not?

So, reading the docs, one slight problem arose. MetalLB does _not_ function as a control-plane VIP the same way kube-vip does. Now what?

As it turns out, you can run both of them side-by-side without issue provided they're both not managing the same set of resources. So I
adjusted kube-vip to ignore services and only scheduled it on control-plane nodes. Relegated to that role, it's working fine.

From there, I was able to configure MetalLB to pick up the services slack pretty easily. I love that it uses CRDs for its config now -- when
last I used it a few years back, it was still based on `ConfigMap`s and environment variables. This was super slick to set up and its
docs are pretty thorough and well-organized.

## Cleaning up

Now that everything else was happily working, I had one remaining issue. Remember how I mentioned running Gitea? Yeah that has a load
balancer service for ssh repo access. Having the ssh clone address match the https clone address and web interface is a huge boon for
convenience.

I _thought_ it would be as straightforward as enabling [IP Address Sharing](https://metallb.org/usage/#ip-address-sharing) for the two
services. But those with reading comprehension better than mine probably noticed this little tidbit (emphasis mine):

> Services can share an IP address under the following conditions:
> They both have the same sharing key.
> They request the use of different ports (e.g. tcp/80 for one and tcp/443 for the other).
> They both use the Cluster external traffic policy, **or they both point to the exact same set of pods** (i.e. the pod selectors are identical).

You see, dear reader, that was not to be. So now what? Well, it turns out that the cause of, and solution to, all of life's problems is
indeed ingress. Traefik is [perfectly happy](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-ingressroutetcp) to
do straight TCP routing if you tell it to do so. Since I knew I only needed one thing to listen on port 22, I didn't have to deal with
TCP termination, it could just straight route it back to Gitea.

That took the following config (on the Traefik helm chart) to achieve:

```yaml
ports:
  ssh:
    port: 22
    expose: true
```

And then creating an `IngressRouteTCP` was easy:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  labels:
    app.kubernetes.io/instance: gitea
  name: gitea-ssh
  namespace: gitea
spec:
  entryPoints:
  - ssh
  routes:
  - match: HostSNI(`*`)
    services:
    - name: gitea-ssh
      port: 22
```

et voila, ssh worked on the same IP/host as ingress!

# Conclusion

We can take a few things away from this. Firstly, I'm bad at reading docs. Secondly, life is often a string of
[Hal lightbulb moments](https://www.youtube.com/watch?v=AbSehcT19u0&pp=ygUOaGFsIGxpZ2h0IGJ1bGI%3D). And finally, the stack always goes
higher and deeper than you think, and that's where the problem is bound to be.

[^1]: If you happen to know of a nice diagram that shows the _real_ full request flow, I'd love to see it.
[^2]: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
