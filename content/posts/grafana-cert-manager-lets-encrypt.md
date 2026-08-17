+++
date = '2026-08-17T16:41:20+02:00'
draft = false
title = "Moving Grafana to cert-manager + Let's Encrypt: Two Gotchas and a Browser Extension"
tags = ['cert-manager', 'lets-encrypt', 'kubernetes', 'flux', 'grafana', 'traefik', 'homelab']
description = "Swapping Grafana's Cloudflare Tunnel for ingress + a real Let's Encrypt cert via cert-manager: a CRD ordering trap, a DNS-01 self-check gotcha, and a browser 'not secure' warning that had nothing to do with any of it."
+++

This is the follow-up to [the incident that ate my evening](../flux-prune-incident) — once the cluster was stable again, I finished what I'd actually sat down to do: move Grafana off a Cloudflare Tunnel onto a real ingress, with a publicly-trusted, auto-renewing certificate instead of a self-signed one.

Two infra gotchas, and one that wasn't infra at all.

## Why not just keep the tunnel

The self-signed cert I'd been using worked, technically, but every browser flagged it and every new device needed the CA trusted manually. cert-manager + a `ClusterIssuer` on DNS-01 against Cloudflare gets a real cert without exposing an HTTP-01 challenge endpoint publicly — Grafana stays reachable only inside the network, but the cert itself is fully trusted everywhere.

## Gotcha one: the CRD ordering trap

`ClusterIssuer` is a CRD that cert-manager's own Helm chart installs. Apply it in the *same* Flux Kustomization as the HelmRelease that creates that CRD, and the dry-run fails outright:

```
ClusterIssuer/letsencrypt-prod dry-run failed: no matches for kind "ClusterIssuer" in version "cert-manager.io/v1"
```

Flux applies everything in one batch — it doesn't wait for a HelmRelease to actually install CRDs before also trying to apply a custom resource of that not-yet-existing type in the same pass. Nothing gets created at all, not even the namespace sticks.

Fix: split into two Kustomizations.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager
spec:
  path: ./infrastructure/controllers/staging/cert-manager
  wait: true   # blocks until the HelmRelease is actually healthy
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager-config
spec:
  path: ./infrastructure/controllers/staging/cert-manager-config
  dependsOn:
    - name: cert-manager   # only applies once the CRD actually exists
```

The chart installs and reports healthy first; the `ClusterIssuer` and its Cloudflare token secret only apply once that's confirmed.

## Gotcha two: DNS-01 self-check uses cluster DNS by default

With the ordering fixed, the `ClusterIssuer` registered with Let's Encrypt fine, and cert-manager successfully created the Cloudflare TXT record for the challenge (the API token was scoped and working). Then it just... sat there:

```
Reason: Waiting for DNS-01 challenge propagation: Could not determine the zone for
"_acme-challenge.grafana.example.se.": Could not find the SOA record in the DNS tree
using nameservers [10.43.0.10:53]
```

`10.43.0.10` is the cluster's internal CoreDNS. cert-manager does a self-check before telling Let's Encrypt to validate, and by default that self-check walks up the DNS tree using whatever resolver the pod has — which, in-cluster, can't resolve the SOA chain for an external domain. It was going to sit there forever.

Fix, one Helm value change:

```yaml
values:
  dns01RecursiveNameservers: "1.1.1.1:53,8.8.8.8:53"
  dns01RecursiveNameserversOnly: true
```

Point the self-check at real public resolvers instead. One Helm upgrade later, the challenge validated and the certificate issued within seconds.

```
$ openssl x509 -noout -subject -issuer -dates
subject=CN=grafana.example.se
issuer=C=US, O=Let's Encrypt, CN=YR1
notBefore=Aug 16 21:56:18 2026 GMT
notAfter=Nov 14 21:56:17 2026 GMT
```

Real cert, 90-day validity, cert-manager renews it automatically well before that.

## The gotcha that wasn't infra at all

Everything server-side checked out: valid chain (`SSL certificate verify ok` with no `-k`), correct SAN, TLS 1.3 with a strong cipher, and — checked directly via a browser automation session — zero mixed content and zero cross-origin requests, logged in or not, 32 requests all same-origin HTTPS.

And yet: "Your connection to this site is not secure. Active content with certificate errors."

Incognito was clean. Normal profile wasn't. A browser extension was quietly injecting or rewriting requests and flagging its own interference as a site problem.

Sometimes the last mile of debugging a "cert issue" isn't the cert, the server, or even the browser's security engine — it's whatever you installed into the browser and forgot about. Worth ruling out before you start decrypting SOPS secrets at 1am.
