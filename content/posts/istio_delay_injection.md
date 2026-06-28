+++  
date = '2026-06-27T12:17:45+02:00'  
draft = false  
title = 'Istio delay injection'  
tags = ['istio', 'kubernetes', 'security', 'resilience', 'fault injection', 'observability']
+++

While working through *Introduction to Istio (LFS144)* from the Linux Foundation, I wanted to dig into Istio's **fault injection** feature, specifically **delay injection**.

## Why delay injection?

Delay injection lets us artificially slow down responses from a service so we can test how the rest of your system behaves under latency. Instead of waiting for a real slowdown to happen in production, we can deliberately introduce one and watch what breaks. This is useful for validating timeouts, retries, circuit breakers, and overall resilience.


A typical scenario: imagine a small pipeline.
```
[service-a] -> [service-b]
```

- **service-a** pulls all incoming requests off a Kafka topic.
- **service-b** processes those requests.

In the real world, service-b can get slow for all kinds of reasons: it's calling out to other downstream services, writing to a database over JDBC, filtering large result sets, and so on. Rather than wait for that to happen naturally, we can tell Istio to inject a delay and observe how service-a copes when its dependency starts dragging.

## The VirtualService
Delay injection is configured on a `VirtualService` via the `http.fault.delay` block:

```yaml
apiVersion: networking.istio.io/v1beta1  
kind: VirtualService  
metadata:  
  name: service-b  
spec:  
  hosts:  
    - 'service-b.default.svc.cluster.local'  
  http:  
    - route:  
      - destination:  
        host: service-b.default.svc.cluster.local  
        port:  
          number: 80  
      fault:  
        delay:  
          percentage:  
            value: 50  
            fixedDelay: 5s
```


### What each field does
- **`fault.delay`** — declares that we want to inject latency on requests matched by this route.
- **`percentage.value: 50`** — the delay is applied to 50% of requests. The other half pass through untouched. This is handy for simulating intermittent slowness rather than a service that's uniformly slow.
- **`fixedDelay: 5s`** — when a request is selected for delay, Istio holds it for 5 seconds before forwarding it on.

The delay is applied by the Envoy sidecar before the request reaches the destination, so the application itself doesn't need any changes. We're injecting the fault at the mesh layer.

## Applying and testing it
Apply the manifest. Then send some traffic and watch the latency:

```bash
for i in $(seq 1 10); do
	curl -s -o /dev/null -w "%{time_total}\n" http://service-b.default.svc.cluster.local/
done
```

Roughly half of those requests should come back in around 5 seconds, while the rest return at normal speed.

## What to look for

Once the delay is in place, the interesting part is watching the *callers*. With my example, observe service-a while service-b is being slowed down:

- Does service-a's own request timeout fire, or does it hang indefinitely?
- Do retries pile up and make things worse
- Does Kafka consumer lag start growing because service-a is blocked waiting?
- Do downstream errors cascade, or is the failure contained?

### Is kafka tuning needed?
It depends on what the delay test reveals. If the problem is _partition stalls / rebalances_ → Kafka poll + DLQ tuning.
If it's _retry storms_ → backoff + circuit breaking. If it's _repeated reads of the same slow data_ → caching.
nut this is another topic for another post.

### resilience needed? 
the whole reason, we would be doing this is to test resilience, so if we find that the system is not resilient, we need to fix it.
rate limiting? circuit breakers? caching? retries? timeouts? all of these are tools to improve resilience, but they need to be tuned to the specific system and workload.

#### observability
in jaeger, we'd see the delay injection as response_flag : DI

## Wrapping up

Delay injection is a simple but powerful tool for resilience testing. Because Istio applies it at the sidecar level, we can simulate a slow dependency without touching application code, target a percentage of traffic, and tune the exact delay. Pairing it with Istio's abort injection (returning errors instead of delays) gives us a solid foundation for chaos-style testing inside the mesh.

A nice testing tool as well is [Gremlin](https://www.gremlin.com/) which can be used to inject faults into the system in a controlled way if we don't want to use Istio for that but thats a different topic for another post.