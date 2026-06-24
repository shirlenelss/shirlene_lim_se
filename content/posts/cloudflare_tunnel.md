+++
date = '2026-06-24T17:35:31+02:00'
draft = true
title = 'Cloudflare_tunnel'
tags = ['cloudflare', 'tunnel', 'kubernetes','security']  
+++
Here are my notes on setting up a Cloudflare tunnel to expose a service running in a Kubernetes cluster.

I have a kubernetes cluster running on my linux machine and I do not want openning a port for nginx or 
any other service to the public, so I want to use cloudflare tunnel to expose it

cloudflare network helps filter the requests from the public
we just need to use it. 

![cloudflare tunnel](/images/cloudflare_tunnel.png)

https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/