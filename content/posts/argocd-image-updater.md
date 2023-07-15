---
title: "ArgoCD Image Updater"
date: 2023-07-15T15:09:19-05:00
draft: false
categories:
    - homelab
tags:
    - k8s
---

[ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/) is a tool that works in concert with ArgoCD to update
images automatically in your Kubernetes cluster based on a set of rules. Its primary limitation is it requires you use either Helm
or Kustomize to deploy your application, as it uses properties of those tools to update the image.

But since Kustomize is so lightweight, it's actually straightforward to (ab)use it to make ArgoCD Image Updater work for you.

# Why?

For starters, I run k8s at home. I don't want to be spending time thinking about updating most of the things I deploy, I just want
them to be at the latest and greatest.

Also, for some containers I run that I build myself, I like to just point them at `ghcr.io/foo/bar:main`. Previously I just used
`imagePullPolicy: Always` and relied on unattended-upgrades rebooting a node to make updates happen. This was...not particularly
speedy or robust, though.

# How?

Start by [installing ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/install/installation/) as you'd like
in your cluster. Method 1 is straightforwad, requires minimal setup, and works for me(TM).

Now, pick any of your deployed app folders. For this walkthrough, I'll be using my [pfapi](https://github.com/bjschafer/pfapi) deployment.
Inside the `pfapi` folder, create a file, `kustomization.yaml` like so:

```yaml
---
resources:
  - deployment.pfapi.yaml
  - ingress.pfapi.yaml
  - sealedsecret.db-creds.yaml
  - service.pfapi.yaml
```

You want to list out all constituent yaml files for this app, with the exception of `namespace` and of course the `kustomization.yaml`
file itself.

Now, you'll need to make some changes to the Argo `Application`. You can do so in the GUI or, if you're using the app-of-apps pattern,
by adding the following to the `Application`:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pfapi
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: pfapi=ghcr.io/bjschafer/pfapi:main
    argocd-image-updater.argoproj.io/pfapi.update-strategy: digest
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/write-back-method: git
spec: # omitted for brevity, no changes required
```

At a minimum, the first two annotations _must_ be configured. The first lists any and all images in this app you want Image Updater to
check on. The second says to look for the latest digest on the given tag, and use that to enforce updates. This way, tags that
are continually pushed to will be updated whenever they change.

The latter two are for the [`git` write-back method](https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/#git-write-back-method),
which isn't mandatory but helps with the GitOps approach to manifests.

Now, commit your changes, push, and sync. Assuming you're setup like I am, you'll see a `.argocd-source-pfapi.yaml` file in the `pfapi` folder.
This is where it stores the image override.

There are other update strategies too -- see the [docs](https://argocd-image-updater.readthedocs.io/en/stable/basics/update-strategies/) for more info.

# Future work

I intend to automate the setup. It would be fairly straightforward to write a script using `yq` to do most of the setup.

I'll also likely do some things with CDK8S to enable setting this up more easily.

# Conclusion

There you have it! The Kustomize requirement, while initially off-putting is actually painless.

And now, you have auto-updating apps in your k8s cluster, configured however you like.
