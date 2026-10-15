+++
date = '2026-10-16T00:07:33+02:00'
draft = true
title = 'Kubernetes_workload'
tags = ['kubernetes', 'workload', 'deployment', 'replicaset', 'statefulset', 'daemonset', 'job', 'cronjob']
+++
# kubernetes #workloads

Here is my raw notes when I was learning about kubernetes workloads.

## deployment
- control what to deploy
- stateless
- scaling
- rolling updates and rollback

## replicaset
- ensure number of pods

## statefulset
- stateful for DB
- stateful network identities
- persistent storage per pod
- creation and deletion order

## daemonset
- background service on every node
- for polling, logging, monitoring, network management, security agent 
- do not use if
  - service needed on some nodes then use deployment with labels
  - scalability needed

## job
- batch jobs
- runs till completion
- can retry
```
restartPolicy: never 
backoffLimit: 3 //retries
```

## cronjob
- batch jobs on schedule
- concurrency policy
- eg running scripts, cleanups, log rotation
```
* * * * * command
min , hour, day of month, month , day of week , command to run

0 0 **** every day midnight 
0 9 ** 1 every monday 9am
```