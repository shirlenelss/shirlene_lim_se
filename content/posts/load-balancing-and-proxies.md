+++
date = '2026-06-22T22:27:54+02:00'
draft = false
title = 'Load Balancing and Proxies'
tags = ['load balancing', 'proxies', 'reverse proxy', 'HAProxy', 'nginx', 'Envoy']
+++

# Load Balancing and Proxies

this is a part of my obsidian notes on load balancing and proxies. 
I haven't turned it into a full post yet, but I wanted to share the raw notes in case they're useful.
Hope to learn more networking concepts and fill in the gaps over time.

## What Problem They Solve

One IP, many servers. You want to distribute traffic and avoid single points of failure.

---

## Types of Load Balancers

### L4 Load Balancer (Transport Layer)
- Operates at TCP/UDP level
- Forwards connections to backends without reading the content
- Fast — doesn't need to parse HTTP
- Can't route based on URL path or hostname
- Example: AWS NLB, hardware load balancers

```
Client → TCP connection → L4 LB → picks backend → forwards connection
(LB never sees HTTP content)
```

### L7 Load Balancer / Reverse Proxy (Application Layer)
- Terminates the connection, reads the HTTP request
- Can make decisions based on hostname, path, headers, cookies
- Can do TLS termination, retries, circuit breaking, header manipulation
- Slower than L4 (parses content) but much more powerful
- Examples: **HAProxy**, **nginx**, **Envoy**, AWS ALB

```
Client → HTTPS → L7 LB → reads "Host: my-application.se, Path: /api"
                       → picks backend based on routing rules
                       → new connection to backend
```

---

## Reverse Proxy vs Forward Proxy

**Reverse proxy** — sits in front of servers. Clients don't know which backend they're hitting. Used for load balancing, TLS termination, caching.

**Forward proxy** — sits in front of clients. Servers don't know which client made the request. Used for corporate internet filtering, VPNs, anonymisation.

---

## Key L7 Features

### TLS Termination
Decrypt HTTPS at the proxy, forward plain HTTP internally. Backends don't need certificates.

### Health Checking
Proxy periodically pings backends. Removes unhealthy ones from rotation automatically.

### Circuit Breaking
If a backend starts failing repeatedly, stop sending it traffic for a period (instead of hammering a broken service). Prevents cascading failures.

### Retries
Automatically retry failed requests on another backend. Useful for transient failures.

### Header Manipulation
Add, remove, or modify HTTP headers before forwarding. Example: setting `X-Forwarded-Proto: https` so the backend knows the original request was HTTPS even though it receives HTTP.

### Path Rewriting
Change the URL before forwarding:
```
Client requests:  /.well-known/openid-credential-issuer/pid-issuer
Forwarded as:     /my-application-issuer/.well-known/openid-credential-issuer
```

---

## HAProxy vs nginx vs Envoy

| | HAProxy | nginx | Envoy |
|---|---|---|---|
| Primary use | L4/L7 LB | Web server + proxy | Service mesh proxy |
| Config | Static files | Static files | Dynamic via xDS API |
| Hot reload | Yes | Yes | Yes (no reload needed) |
| Used by | OpenShift Router | Many deployments | Istio, many service meshes |

**Envoy** is special — it's designed to be configured dynamically via an API (xDS). This is what makes Istio possible: 
istiod pushes new routing config to all Envoy sidecars without restarting them.
