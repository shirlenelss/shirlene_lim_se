+++
date = '2026-08-21T10:55:56+02:00'
draft = false
title = 'Would This Survive 30 Services? Replacing Per-App Kustomize With One Helm Chart'
tags = ['helm', 'kustomize', 'argocd', 'gitops', 'kubernetes', 'istio', 'traefik', 'homelab']
description = "The Kustomize setup for my Argo CD promotion demo worked fine for two services, then I asked what 30 would look like. The honest answer was ugly, so I replaced it with one shared Helm chart and a matrix ApplicationSet."
+++

I've been building `argo-auto-promote`, a small GitOps demo: two services, promoted dev → stage → prod by pinning an image tag, synced by Argo CD. It worked. Then I asked myself the question that actually matters for anything meant to resemble real infrastructure: *would this survive 30 services?*

The honest answer was no. Here's what that boilerplate looked like, and what replaced it.

## Before: what one service cost

Every service needed, per environment:

```
repo/base/service-a/
  deployment.yaml
  service.yaml
  kustomization.yaml
```

...plus, in **every one of four overlays**, an `images:` entry:

```yaml
images:
  - name: service-a
    newTag: "0.0.1" 
```
and so much more values...when it's real production configs

...plus a `replacements:` block, because the container's `IMAGE_TAG` env var needed to match whatever tag `images:` had just pinned, and Kustomize has no built-in way to reference "the value I just set two lines up" — so it reads it back off the Deployment's own rendered `image:` field instead:

```yaml
replacements:
  - source:
      kind: Deployment
      name: service-a-app
      fieldPath: spec.template.spec.containers.[name=service-a-app].image
      options: {delimiter: ":", index: 1}
    targets:
      - select: {kind: Deployment, name: service-a-app}
        fieldPaths:
          - spec.template.spec.containers.[name=service-a-app].env.[name=IMAGE_TAG].value
```

...plus, for the two environments that got a URL, a `VirtualService` file and a JSON6902 patch per environment to set its hostname, and one more hand-added rule in a single shared, hand-maintained `Ingress` file.

None of that loops. Kustomize has no concept of "repeat this for every service in a list" — every one of those blocks gets copy-pasted, by hand, per service, per environment. Two services already meant duplicating the `replacements:` block eight times across four overlay files. 
Thirty services would mean roughly 240 near-identical blocks, (cry) each one a chance to typo a container name and quietly break `IMAGE_TAG` for one specific service in one specific environment, with no automated way to notice.

## After: one chart, tiny values files

Everything service-specific moved into [helm template](https://helm.sh/docs/chart_template_guide/getting_started) (helped by my trusty CLAUDE assistant friend); everything *actually specific to one service* shrank to about eight lines:
(caveat that this might be complicated or dangerous for real production configs, but one should test the templating process separately, before applying to actually cluster)

```yaml
# chart/services/service-a/stage.yaml
name: service-a
image:
  repository: service-a
  tag: "0.0.1"
env:
  SERVICE_B_URL: http://service-b
route:
  enabled: true
  env: stage
```

The `replacements:` trick disappears entirely, because a Helm template can just reference the same value twice:

```yaml
# chart/templates/deployment.yaml
containers:
  - name: {{ .Values.name }}-app
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    env:
      - name: IMAGE_TAG
        value: "{{ .Values.image.tag }}"   # same source, no readback needed
```

Adding service #31 is: write its four values files, add one line to a list. Nothing about the chart, the Deployment shape, or the routing logic needs to change or even be looked at.

## The two things that weren't a straight port

**The `Gateway`.** In the old setup, every env overlay bundled the *whole system* into one `Application`, so a per-env copy of the shared `Gateway` lived naturally alongside it. Moving to one `Application` per `(service, env)` pair means there's no longer an env-level owner for a shared resource — so the `Gateway` now lives once, permanently, in `istio-system`, and every service's `VirtualService` references it by its qualified name, `istio-system/sample-gateway`, instead of an unqualified same-namespace name.

**The `Ingress`.** A core Kubernetes `Ingress`'s backend service has to live in the *same namespace* as the `Ingress` itself — and `istio-ingressgateway` lives in `istio-system`, not in each service's `sample-<env>` namespace. So each service's chart render creates its own small `Ingress`, explicitly namespaced to `istio-system`:

```yaml
{{- if .Values.route.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.name }}-{{ .Values.route.env }}
  namespace: istio-system
{{- end }}
```

That name has to include the env, not just the service — every environment's `Ingress` lands in the *same* namespace now, so `service-a-dev` and `service-a-stage`'s `Applications` would otherwise collide over one object. Missed that on the first pass; caught it rendering all eight `(service, env)` combinations with `helm template` before touching the live cluster, not after.

Traefik (this cluster's ingress controller) merges routing rules from every `Ingress` matching its class, so thirty small `Ingress` objects behave identically to one big hand-maintained one — without anyone hand-maintaining it.

## Promotion got more honest, not just shorter

The old setup's `images:` block having one entry per service meant service-a and service-b *could* already be promoted independently — but `promote.sh` and the CI workflow both still pinned every service to the same tag value, because that's what the single-overlay-per-env structure made easy. 
With a matrix `ApplicationSet` (`services x envs`) generating a separate `Application` per pair, independent promotion stopped being a "you could, but" and became the only way it works: `promote.sh service-a 0.0.2` touches exactly `service-a`'s three values files, full stop.

![after template](/assets/images/argo_after_helm_template.png)

 
## What actually broke during the swap

Two things, both caught by checking rather than assuming:

1. The old `Application` that deployed the shared Traefik `Ingress` predated the `resources-finalizer.argocd.argoproj.io` pattern the newer ones use, so deleting it left an orphaned `Ingress` sitting in the cluster instead of cascading away. `kubectl get ingress -n istio-system` after the swap showed it immediately; one `kubectl delete` cleaned it up.
2. `helm template` for every `(service, env)` combination, before deploying anything, is what caught the `Ingress` naming collision above -- rendering the *actual* full matrix, not just spot-checking one environment, is what makes "would this survive 30 services" answerable instead of hopeful.

Repo: [github.com/shirlenelss/argo-auto-promote](https://github.com/shirlenelss/argo-auto-promote)
