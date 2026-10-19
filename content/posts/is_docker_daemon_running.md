+++
date = '2026-10-19T21:14:36+02:00'
draft = true
title = 'Is docker daemon running?'
tags = ['docker', 'daemon', 'docker context', 'docker desktop']
+++

This is another of my raw notes 
## what prompted this post? 
```
docker ps
"Cannot connect to the Docker daemon at unix:///Users/shirlim/.docker/run/docker.sock. Is the docker daemon running?"
```

## investigation which docker is running
ls /Applications | grep -i -E "docker|orbstack"
Docker.app

## check for colima
colima is a free opensource tool that runs Docker engine
```
which colima
colima not found
```

## list the docker contexts in my mac
```
docker context ls
NAME              DESCRIPTION                               DOCKER ENDPOINT                                 ERROR
default           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux *   Docker Desktop                            unix:///Users/shirlim/.docker/run/docker.sock
rancher-desktop   Rancher Desktop moby context              unix:///Users/shirlim/.rd/docker.sock
```

if I'm using linux then it would be this one
```
default *   Current DOCKER_HOST based configuration   unix:///var/run/docker.sock

# to enabled it, run this 
sudo systemctl start docker
sudo systemctl enable docker    # start automatically at boot
```

## Solution : start docker (mac)
```
open -a Docker
```
open, -a means application, Docker

# why
When Docker Desktop starts, it boots its hidden Linux VM and the Docker engine inside it. It also creates the socket file at `~/.docker/run/docker.sock`.

verify by calling 
```
docker ps 
```