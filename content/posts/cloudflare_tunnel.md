+++
date = '2026-06-24T17:35:31+02:00'
draft = false
title = 'Secure sharing with Cloudflare tunnel'
tags = ['cloudflare', 'tunnel', 'kubernetes','security']
+++
Here are my notes on setting up a Cloudflare tunnel to expose a service running in a Kubernetes cluster.

I have a kubernetes cluster running on my linux machine and I do not want openning a port for nginx or
any other service to the public, so I want to use cloudflare tunnel to expose it

cloudflare network helps filter the requests from the public
we just need to use it.

![cloudflare tunnel](/assets/images/cloudflare_tunnel.png)

here's the step by step guide to set up a cloudflare tunnel for your service running in kubernetes cluster
https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/
https://developers.cloudflare.com/tunnel/setup

After that go to cloudflare dashboard, add a tunnel add a route to your service, 
and then you can access it from the public internet.
![linkding cluster on my mac](/assets/images/linkding.png)
my linkding page https://ldpi.shirlenelim.se
![linkding](/assets/images/linkding2.png)

my github code can be found [here](https://github.com/shirlenelss/homelab-cluster) 


