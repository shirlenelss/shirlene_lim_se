+++
date = '2026-07-22T21:00:12+08:00'
draft = false
title = 'Helm part2 on applying values'
description = "Part 2 of my Helm notes, applying values to customize Helm charts for my homelab cluster."
tags = ['helm', 'kubernetes', 'flux', 'gitops', 'homelab']
+++

Ok so here's part 2 continuation of my notes on [Helm intro](../helm_intro), continuing from the intro to Helm charts, repositories, and releases.
# How do we know what values are available for a chart?
first of all, how do we know what are the values/configurations that we can set for a chart? 
## One - read values.yaml
well, the chart itself has a `values.yaml` file that contains the default values for the chart in its git repo, website.
## Two - read README.yaml
We could check the chart's documentation or the `README.md` file? Should we trust that the documentation is up to date?
## Three - generate a sample values.yaml file
The best way to know what values are available is to generate a sample `values.yaml` file from the chart itself.
We can do this with the `helm show values` command, which will output the default values for the chart in YAML format.

In my home cluster, I have a prometheus-stack chart installed, and I want to see what values are available for it.
Write it out to a file called `default-value.yaml`. We'll refer to this file when we want to customize the chart's configuration for our own needs.    
```
# let's use proemetheus-stack as an example.
helm show values prometheus-community/kube-prometheus-stack> default-value.yaml
```

So say I want to add ingress for my grafana dashboard, I can check the `default-value.yaml` file for the ingress section and see what values are available to set.

![helm_default_values.png](/assets/images/helm_default_values.png)

so just copy the lines, that applies to ingress, and paste it into a new file called `release.yaml`, then modify the values as needed.
![grafana_ingress_release.png](/assets/images/grafana_ingress_release.png)


### Show values with my own values file to see applied picture
We can also use the `helm show values` command with our own values file to see what values are set in that file. This is useful if we want to see what values we have customized
```
helm show values prometheus-community/kube-prometheus-stack -f release.yaml
```


my homelab cluster for this is at https://github.com/shirlenelss/homelab-cluster
After flux kustomization applied, we can see the grafana dashboard is now accessible via ingress with the new password set in the `release.yaml` file.

Here's the CoreDNS dashboard
![kubelet dashboard](/assets/images/grafana_dashboard.png)
