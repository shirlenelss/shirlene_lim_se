+++
date = '2026-06-22T21:55:56+02:00'
draft = false
title = 'Istio Whitelist'
tags = ['istio', 'kubernetes', 'security']
+++

# Gateways

Traffic flow through the mesh:

```
ingress gateway -> istio service mesh -> egress gateway
```

The **Gateway** resource sits at the edge and decides which hosts are allowed in. Think of `hosts` as the filter for what to let through.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:          # this is the filter — what to let through
        - dev.example.com
        - test.example.com
```

# Get services

```bash
kubectl get svc -n istio-system
```

```
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.0.146.214   <none>          80/TCP,443/TCP,15443/TCP                                                     7m56s
istio-ingressgateway   LoadBalancer   10.0.98.7      XX.XXX.XXX.XXX  15021:31395/TCP,80:32542/TCP,443:31347/TCP,31400:32663/TCP,15443:31525/TCP  7m56s
istiod                 ClusterIP      10.0.66.251    <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                                8m6s
```

# Wiring it up

```
gateway -> virtualservice
```

The Gateway opens the door; the VirtualService decides where the traffic goes once it's inside.

## Deploying the Gateway resource

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - dev.example.com
        - test.example.com
```

```bash
kubectl apply -f gateway.yaml
```

Check which service is the ingress gateway:

```bash
kubectl get svc -l istio=ingressgateway -n istio-system
```

```
NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.0.98.7    [GATEWAY_IP]  15021:31395/TCP,80:32542/TCP,443:31347/TCP,31400:32663/TCP,15443:31525/TCP  9h
```

## VirtualService

The Gateway accepts the host; the **VirtualService** routes that host's traffic to an actual backend service inside the mesh. Note the `gateways` field — it binds this route to the Gateway above, so these rules only apply to traffic that came in through it.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-virtualservice
  namespace: default
spec:
  hosts:
    - dev.example.com
    - test.example.com
  gateways:
    - gateway          # must match the Gateway metadata.name above
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            host: my-app.default.svc.cluster.local   # in-cluster service
            port:
              number: 8080
```

## RequestAuthentication + AuthorizationPolicy (JWT)

The Gateway filters by host, but to whitelist *who* can talk to a workload by validating a token, you need **two** resources:

1. A **RequestAuthentication** that cryptographically verifies the JWT against your issuer's public keys (JWKS).
2. An **AuthorizationPolicy** that *requires* a valid JWT to be present.

This split matters: `RequestAuthentication` only rejects *invalid* tokens — a request with **no** token still passes it. The `AuthorizationPolicy` with `requestPrincipals` is what actually makes the token mandatory.

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: my-app-jwt
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app          # applies to pods with this label
  jwtRules:
    - issuer: "https://your-issuer.example.com"
      jwksUri: "https://your-issuer.example.com/.well-known/jwks.json"
      # By default the token is read from the
      # Authorization: Bearer <token> header.
      # Override if your token lives elsewhere:
      # fromHeaders:
      #   - name: x-jwt-token
      #     prefix: "Bearer "
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: my-app-require-jwt
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]   # requires ANY valid authenticated JWT
      to:
        - operation:
            methods:
              - GET
              - POST
```

A few things worth knowing:

- `requestPrincipals: ["*"]` means "any valid token from a configured issuer." To lock it to one issuer/subject, use the form `<issuer>/<subject>`, e.g. `"https://your-issuer.example.com/user@example.com"`.
- The `issuer` in the policy must exactly match the `iss` claim in the token, and `jwksUri` must be reachable from the mesh, otherwise every request gets rejected.
- To gate on specific claims (roles, groups, scopes), add a `when` block:

  ```yaml
        when:
          - key: request.auth.claims[groups]
            values:
              - "admins"
  ```

- If you define *any* `ALLOW` policy selecting a workload, everything not matched is denied — so requiring the JWT is implicit once this rule exists.

Apply everything:

```bash
kubectl apply -f virtualservice.yaml
kubectl apply -f requestauthentication.yaml
kubectl apply -f authorizationpolicy.yaml
```