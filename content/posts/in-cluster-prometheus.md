---
title: "In-Cluster Prometheus"
date: 2022-07-27T21:31:42-05:00
draft: false
categories:
  - homelab
tags:
  - k8s
  - prometheus
---

Up until recently, I was running two separate Prometheus instances -- one on a Raspberry Pi, and the other in my k3s cluster using kube-prometheus-stack. I wanted to unify them, ideally to simplify management and version control. The challenge here is in how to manage the scrape targets for out-of-cluster resources.

Thanks to my friend Justin, I was able to use a much more elegant solution.

## Options

### The basic way

When deploying `kube-prometheus-stack`, define `additionalScrapeConfigs`. Since I deployed via the Helm chart, that would mean doing a `helm upgrade` each time I needed to change things. Gross.

### The documented way

If you search how to use an in-cluster Prometheus instance to monitor out-of-cluster things, the suggestion is straightforward but complicated. Essentially, define `Endpoint` resources for each out-of-cluster thing and then set up `ServiceMonitors` to get Prometheus to scrape things.

This is gross, and requires 2 resources for each scrape, plus adds extra overhead to your cluster.

### The best way

Justin clued me into the fact that you can (ab)use `Probes` to do your bidding. Here's a straightforward example you might use to scrape `node_exporter` on an external host called `myserver.example.com`:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  namespace: prometheus
  name: myserver-node-exporter
  labels:
    release: prometheus-kube-prometheus-stack
spec:
  jobName: myserver-node-exporter
  prober:
    url: prometheus-kube-prometheus-prometheus:9090
    path: /metrics
  targets:
    staticConfig:
      static:
        - myserver.example.com:9100
      relabelingConfigs:
        - sourceLabels:
            - instance
          targetLabel: "__address__"
          action: replace
        - regex: "__param_target"
          action: labeldrop
```

The only change you'd need to make here is the hostname/port in `.spec.targets.staticConfig.static`. All the rest should stay as is (assuming you don't need to change the labels the operator listens on, the namespace, or the service name).
In particular, the `relabelingConfigs` and the `.spec.prober.url` are written as intendend and shouldn't be changed. Only change `.spec.prober.path` if your exporter is exporting at a different path.

If you need to adjust the scheme, that's set on `.spec.prober.scheme`. If you need a bearer token, you set `.spec.bearerTokenSecret` like so:

```yaml
spec:
  bearerTokenSecret:
    name: my-secret-name
    key: token
```


and then plop the bearer token in the given secret.

## Limitations

The best way has at least one limitation: you can't define `params` on the probe, so if you need those you'll have to work around it.
