+++
date = '2026-08-17T16:41:20+02:00'
draft = false
title = 'How a Missing Line in a Kustomization Wiped My Homelab (and What I Learned)'
tags = ['flux', 'gitops', 'kubernetes', 'kustomize', 'cert-manager', 'lets-encrypt', 'homelab']
description = "A one-line missing resource in clusters/staging/kustomization.yaml let Flux prune its own controllers and cascade-delete two namespaces, including my linkding bookmarks. Here''''s the incident and the recovery."
+++

I was trying to do two small, boring things: let Prometheus scrape my apps, and move Grafana off a Cloudflare Tunnel onto a real ingress with TLS. 
Instead I spent the evening watching Flux delete its own controllers and then my `linkding` and 
`monitoring` namespaces — PVC included.

Here's what happened, why, and the fixes that came out of it.

## The setup

Standard Flux GitOps layout:

```
clusters/staging/
  flux-system/
    gotk-components.yaml   # Flux's own CRDs, controllers, RBAC
    gotk-sync.yaml         # GitRepository + root Kustomization
  apps.yaml                # Kustomization CR -> ./apps/staging
  monitoring.yaml          # Kustomization CR -> ./monitoring/*
```

The root Kustomization watches `./clusters/staging` with `prune: true`. Nothing unusual — this is close to the default `flux bootstrap` layout.

## The bug

There was a `clusters/staging/kustomization.yaml` sitting in my working tree, staged but never committed, from an earlier session. It looked innocuous:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - apps.yaml
  - monitoring.yaml
```

It was missing `flux-system`.

Before this file existed, `./clusters/staging` had no explicit `kustomization.yaml` at all, 
so kustomize-controller fell back to auto-building every yaml file under that path — which implicitly picked up the
`flux-system/` subdirectory too. That's how Flux's own CRDs, controllers, GitRepository, and RBAC were 
staying reconciled without anyone explicitly listing them.

The moment I committed that pre-staged file (without actually reading its content closely enough — lesson one, below), 
Flux's desired state shrank to just `apps.yaml` and `monitoring.yaml`. With `prune: true`, it deleted everything it now 
considered orphaned: its own Deployments, its own `GitRepository`. Killing its own `kustomize-controller` pod mid-apply 
canceled the rest of the batch — which, by pure luck, is what stopped it short of also deleting the CRDs and RBAC. 
If that apply had finished, it would have taken every Flux-managed custom resource in the cluster with it.

## The cascade

Losing its own controllers wasn't the only damage. Once the `GitRepository` came back and reconciled again, the `apps`
and `monitoring-controllers` Kustomizations — which manage `linkding` and `monitoring` — also pruned everything they owned:
Deployments, Services, Secrets, and both namespaces outright.

`linkding`'s PVC used the cluster's default `local-path` StorageClass, `reclaimPolicy: Delete`. Namespace deletion took 
the PVC with it, and `local-path-provisioner`'s cleanup helper pod did exactly what it's supposed to do: it deleted the
actual files off disk. No PVC, no backup, no bookmarks.

## Recovery

Roughly in order:

1. **Reapply Flux's own bootstrap manifests** (`kubectl apply -k clusters/staging/flux-system/`) to bring the controllers back.
2. Hit a deadlock immediately: the `flux-system` namespace was stuck `Terminating`, blocked on `finalizers.fluxcd.io` on the orphaned 
   `Kustomization`/`GitRepository` objects — which only the (now-deleted) controllers could clear. Classic circular block: namespace can't finish deleting because of finalizers, controllers can't be recreated because the namespace is terminating.
3. Manually stripped the stuck finalizers (`kubectl patch ... -p '{"metadata":{"finalizers":[]}}'`) to let the namespace finish terminating, then recreated everything clean.
4. The deploy key and SOPS decryption key — both correctly *not* GitOps-managed, since they're credentials — had been destroyed along with the namespace. `flux bootstrap github` regenerated the deploy key; the SOPS age key got recreated from a local backup.
5. Fixed the actual root cause: added `flux-system` back to the resources list.

Full cluster recovery, minus the bookmark data, took about 40 minutes once the actual problem was identified.

## What changed as a result

**1. Read pre-staged files before committing them, especially in GitOps repos.** A file sitting in the working tree isn't "probably fine" just because it was there before you started. If it touches anything Flux reconciles, its content is the whole ballgame.

**2. Protect anything you can't afford to lose, at two independent layers:**

```yaml
# Flux will never prune this, no matter what the kustomization structure looks like later
metadata:
  annotations:
    kustomize.toolkit.fluxcd.io/prune: "disabled"
```

```yaml
# A deleted PVC leaves the data recoverable instead of wiping it
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
```

Neither of these would have prevented the *first* mistake, but either alone would have kept the bookmark data intact even after everything else went wrong. Belt and suspenders.

**3. `prune: true` is not a toy.** It's the whole point of GitOps — but it means the blast radius of "what does my kustomization.yaml actually list" is the entire managed tree, not just the file you're looking at.

Once the cluster was stable again, I still had the thing I'd actually sat down to do that evening: move Grafana off a Cloudflare Tunnel onto a real ingress with a proper TLS cert. That part — and the two more gotchas it came with — is [its own post](/posts/grafana-cert-manager-lets-encrypt).

Here's the repo https://github.com/shirlenelss/homelab-cluster
