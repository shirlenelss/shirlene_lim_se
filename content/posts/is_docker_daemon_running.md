+++
date = '2026-10-19T21:14:36+02:00'
draft = false
title = 'Is docker daemon running?'
tags = ['docker', 'daemon', 'docker context', 'docker desktop']
+++

This is another of my raw notes 
## what prompted this post? 
```
docker ps
"Cannot connect to the Docker daemon at unix:///Users/shirlim/.docker/run/docker.sock. Is the docker daemon running?"
```

I'm normally switching machines depending on assignment and tasks at hand. I use 3 main OS: linux, mac and windows (on WSL) 
So configuring my workplace is crucial and a necessity för my mental state :D

Here's how I normally find out. I dump the check command in a check_docker.sh with different paths if need be, 
so I can just run it from different machine. Of course, nowadays one can always ask AI to check it. 
But why waste tokens on this? 

## investigation which docker is running
```
ls /Applications | grep -i -E "docker|orbstack"
Docker.app
```

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
When Docker Desktop starts, it boots its hidden Linux VM and the Docker engine inside it.
It also creates the socket file at `~/.docker/run/docker.sock`.

verify by calling 
```
docker ps
```