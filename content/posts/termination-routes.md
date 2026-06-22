+++
date = '2026-06-22T22:46:52+02:00'
draft = false
title = 'Termination Routes'
tags = ['openshift', 'istio', 'tls termination']
+++

## Termination value for route
Termination is a property of OpenShift Routes that determines how TLS is handled between 
the client, the router (HAProxy), and the backend pods. You can see the termination type 
in the output of `oc get routes -o wide` under the "Termination" column.

## Termination column values explained
```
Value           Meaning
edge            TLS terminates at the router (HAProxy), traffic to pod is plain HTTP
edge/redirect   Same as edge, but HTTP → HTTPS redirect is enforced automatically
passthrough     TLS is not terminated at the router — raw TCP goes straight to the pod, which must handle TLS itself
encryptTLS      terminates at router, then re-encrypts for the pod leg(empty)
(empty)         No TLS plain HTTP all the way
```

```
OpenShift Router (HAProxy)
│
Client ──── HTTPS ────► [DECRYPT] ──── HTTP ────► Pod     ← edge
Client ──── HTTPS ────► [DECRYPT] ──── HTTPS ───► Pod     ← reencrypt
Client ──── HTTPS ──────────────────── HTTPS ───► Pod     ← passthrough
Client ──── HTTP  ──── 301 ──► HTTPS ──────────► Pod      ← edge/redirect
```


Istio+OpenShift setup, if Istio handles mTLS inside the mesh, edge is typically fine — 
Istio secures pod-to-pod, and the router handles the external-facing cert.

```
Client (HTTPS/HTTP2)
↓
OpenShift Route — edge TLS termination (wildcard *.my-application.se)
↓
istio-ingressgateway (receives plain HTTP/HTTP2)
↓
VirtualService
↓
Keycloak Pod (AuthorizationPolicy enforced here)
```

In my line of work, https is a must. 
I use edge termination for simplicity, and let Istio handle security inside the cluster.
I will write more about implementing this in a future post, but the gist is:
- Route with edge termination for external traffic
- Istio mTLS for internal pod-to-pod communication being that I have Authpolicy implemented for the some deployments already, 
and I want to ensure that traffic to it is encrypted and authenticated.
- AuthorizationPolicy in Istio to control access to the Keycloak pod
