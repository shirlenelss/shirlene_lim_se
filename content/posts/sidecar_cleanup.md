+++
date = '2026-07-15T12:03:02+08:00'
draft = false
title = 'Sidecar cleanup is important'
tags = ['istio', 'kubernetes', 'performance', 'envoy']
description = "Why leftover Istio sidecars cause silent problems in Kubernetes clusters, and how to clean them up properly."
+++


# cleanup of the service mesh is important
We might encounter unnecessary CPU usage when all service get updated with information from all other services that's not relevant

for example services in the istio bookInfo sample app
![Bookinfo](/assets/images/bookinfo.png)

product page doesn't need to know about ratings and likewise
ratings only need to be updated on reviews-v1 and reviews-v2

if this is not organized and maintained, we hv 6x5 connections =30 connections and this is just with the few services. Optimized, we need only 6. That's a whopping 80% off CPU processing.

## run this to spy on the product page sidecar
```bash
istioctl proxy-config endpoints deploy/productpage-v1.default
```

![sidecar endpoint list](/assets/images/sidecar_endpoint_list.png)

## to prune it, do this

```yaml
apiVersion: networking.istio.io/v1beta1  
kind: Sidecar  
metadata:  
  name: productpage-sidecar  
  namespace: default  
spec:  
  workloadSelector:  
    labels:  
      app: productpage  
  egress:  
  - hosts:  
    - "./reviews.default.svc.cluster.local"  
    - "./details.default.svc.cluster.local"  
    - "istio-system/*"  
---  
apiVersion: networking.istio.io/v1beta1  
kind: Sidecar  
metadata: name: reviews-v1-sidecar  
  namespace: default  
spec:  
  workloadSelector:  
    labels:  
      app: reviews  
      version: v1  
  egress:  
  - hosts:  
    - "istio-system/*"  
---  
apiVersion: networking.istio.io/v1beta1  
kind: Sidecar  
metadata:  
  name: reviews-v2-sidecar  
  namespace: default  
spec:  
  workloadSelector:  
    labels:  
      app: reviews  
      version: v2  
  egress:  
  - hosts:  
    - "./ratings.default.svc.cluster.local"  
    - "istio-system/*"  
---  
apiVersion: networking.istio.io/v1beta1  
kind: Sidecar  
metadata:  
  name: reviews-v3-sidecar  
  namespace: default  
spec:  
  workloadSelector:  
    labels:  
      app: reviews  
      version: v3  
  egress:  
  - hosts:  
    - "./ratings.default.svc.cluster.local"  
    - "istio-system/*"  
---  
apiVersion: networking.istio.io/v1beta1  
kind: Sidecar  
metadata:  
  name: details-sidecar  
  namespace: default  
spec:  
  workloadSelector:  
    labels:  
      app: details  
  egress:  
    - hosts:  
      - "istio-system/*"  
---  
apiVersion: networking.istio.io/v1beta1  
kind: Sidecar  
metadata:  
  name: ratings-sidecar  
  namespace: default  
spec:  
  workloadSelector:  
    labels:  
      app: ratings  
  egress:  
  - hosts:  
    - "istio-system/*"
      
```

the dire scenario is explained [# Istio Feb Meetup/ Demo: Scaling Istio in Large Clusters by Auto-Generating Sidecars by Cathal Conroy](https://www.youtube.com/watch?v=wcJPC_bXYBA)
## Summary

## Notes
AFAIK cpu usage is not the only concern, memory usage is also a concern. The more connections, the more memory is used.
