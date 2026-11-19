+++
date = '2026-11-15T19:32:57+01:00'  
draft = false  
title = 'Setting up 3 node cluster on mac'  
+++

# Homelab Kubernetes Cluster on macOS (VirtualBox)

I have decided to use as a 3 node kubeadm cluster for my studies. Intention is to see /etcd in action instead on rancher desktop.   
Rancher desktop doesn't use the /etcd so a bit different environment than what I need.   
The goal is to use VM and linux iso to practice some networking while leaving my mac environment as is.

Three-node kubeadm cluster running in VirtualBox on an Apple Silicon Mac (M4, 32 GB).

| Node | Role | CPU | Memory | Disk |  
|---|---|---|---|---|  
| `cp1` | Control plane | 2 | 4 GB | 30 GB (dynamic) |  
| `worker1` | Worker | 2 | 4 GB | 30 GB (dynamic) |  
| `worker2` | Worker | 2 | 4 GB | 30 GB (dynamic) |  

- OS: Ubuntu Server 26.04.1 (ARM64)
- Kubernetes: v1.36 (kubeadm), containerd, Flannel CNI

---  

## 1. Download the right ISO

Apple Silicon VirtualBox only runs **ARM** guests. An `amd64` ISO will not boot.

Use the **server** image (no GUI, lighter):

```  
ubuntu-26.04.1-live-server-arm64.iso  
```  

Available from cdimage.ubuntu.com, under Ubuntu Server for ARM.

> If `mv` from `~/Downloads` fails with *Operation not permitted*, the app running your shell lacks macOS privacy access. Grant it under **System Settings → Privacy & Security → Full Disk Access**, then quit (Cmd+Q) and reopen it. If the shell runs inside a multiplexer launched by `launchd` (check with `ps -o comm= -p $PPID`), that binary is the one needing access. Or just drag the file in Finder.
  
---  

## 2. Create the first VM (cp1)

In VirtualBox → **New**:

- **ISO**: the `live-server-arm64` ISO
- **OS version**: Ubuntu (ARM 64-bit). VirtualBox may detect it as 25.04; that is harmless.
- **Skip Unattended Installation**: tick this
- **Hardware**: 2 CPUs, 4096 MB memory
- **Disk**: 30 GB, dynamically allocated

### Network: use Bridged

Default NAT does not let the VMs reach each other or let the Mac SSH in.

**Settings → Network → Adapter 1 → Attached to: Bridged Adapter**, then pick the Mac's active interface. Find it on the Mac with:

```bash  
route get default | grep interface  
```  

![virtualbox](/assets/images/virtualbox.png)

(MacBook: `en0` = Wi-Fi.)

#### Display in virtual box

**View → Virtual Screen 1 → Scale to 150%** (200% can push the bottom line off-screen; use **Full-screen Mode**, Cmd+F, if needed).

---  

## 3. Ubuntu installer choices

| Step             | Choice                                        |     |
| ---------------- | --------------------------------------------- | --- |
| Keyboard layout  | Swedish                                       |     |
| Keyboard variant | Swedish – Swedish (Macintosh)                 |     |
| Keyboard model   | Generic 105-key PC                            |     |
| AltGr key        | Both Alt keys (Option keys act like on macOS) |     |
| Compose key      | No compose key                                |     |
| Hostname         | `cp1`                                         |     |
| SSH              | **Install OpenSSH server**                    |     |
| Swap             | Not needed (disabled later anyway)            |     |

Fix keyboard after install if needed:

```bash  
sudo dpkg-reconfigure keyboard-configuration  
sudo setupcon  
```  

Bigger console font (optional):

```bash  
sudo dpkg-reconfigure console-setup   # UTF-8 → Latin → Terminus → 16x32  
```  
  
---  

## 4. First boot and SSH

On the VM console:

```bash  
ip -4 a                        # note the 192.168.x.x address (not 10.0.2.15)  
```  

If SSH is refused (OpenSSH not installed):

```bash  
sudo apt update  
sudo apt install -y openssh-server  
sudo systemctl enable --now ssh  
```  

From the Mac:

```bash  
ssh <user>@<vm-ip>  
```  

**Reserve each node's IP in the router's DHCP settings** (or set static IPs). Node IPs must not change once the cluster exists.

### Hostname check

```bash  
hostname  
sudo hostnamectl set-hostname cp1  
sudo sed -i 's/<old-hostname>/cp1/g' /etc/hosts  
```  
  
---  

## 5. Node prerequisites (on cp1, before cloning)

### Swap, kernel modules, sysctl

```bash  
sudo swapoff -a  
sudo sed -i '/swap/ s/^/#/' /etc/fstab  
  
sudo tee /etc/modules-load.d/k8s.conf <<EOF  
overlay  
br_netfilter  
EOF  
sudo modprobe overlay  
sudo modprobe br_netfilter  
  
sudo tee /etc/sysctl.d/k8s.conf <<EOF  
net.bridge.bridge-nf-call-iptables  = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
net.ipv4.ip_forward                 = 1  
EOF  
sudo sysctl --system  
```  

### explanation of these flags
net.ipv4.ip_forward = 1
the machine act as router packet to be passed between notwork interfaces. kubernetes needs it for traffic moving between pods, node and outside world. If this is 0, the packets would be dropped if not directly to machine itself. in this case , cp1

net.bridge.bridge-nf-call-iptables  = 1 and net.bridge.bridge-nf-call-ip6tables = 1
there's a virtual bridge cni0 that shows up when we write ´ip a . By default, traffic crossing a bridge skips the iptables firewall rules. These 2 rules doesn't allow that. Forces it to go thru iptables anyways. pods on a node are connected through that virtual bridge.

br_netfilter kernel module must be loaded or bridge settings wont exist

Verify:

```bash  
free -h                      # Swap: 0B  
lsmod | grep br_netfilter  
```  

### containerd
is a container runtime currently supporting docker containers. This pulls images and starts and stop containers on a machine. 
Both docker and kubernetes uses it under the hood. 
Other container runtime are: CRI, CRI-O and cri-Dockerd (which needs dockerd Moby)

```bash  
sudo apt update && sudo apt install -y containerd  
sudo mkdir -p /etc/containerd  
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null  
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml  
sudo systemctl restart containerd  
  
grep SystemdCgroup /etc/containerd/config.toml   # must say true  
```  

### kubeadm, kubelet, kubectl (v1.36)

```bash  
sudo apt install -y apt-transport-https ca-certificates curl gpg  
sudo mkdir -p -m 755 /etc/apt/keyrings  
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg  
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list  
  
sudo apt update  
sudo apt install -y kubelet kubeadm kubectl  
sudo apt-mark hold kubelet kubeadm kubectl  
sudo systemctl enable --now kubelet  
  
kubeadm version  
```  

kubelet restarting in a loop is normal until `kubeadm init`.

### Pre-pull images (optional, saves time on clones)

```bash  
sudo kubeadm config images pull  
```  
  
---  

## 6. Clone the workers

1. `sudo poweroff` on cp1.
   - If shutdown is *inhibited by APT (unattended-upgr)*, do **not** force it. Wait until `pgrep -a apt; pgrep -a dpkg` prints nothing. (`unattended-upgrade-shutdown --wait-for-signal` is an idle helper and can be ignored.)
2. VirtualBox → right-click cp1 → **Clone**
   - Name: `worker1`
   - **Full clone**
   - MAC address policy: **Generate new MAC addresses for all network adapters**
3. Repeat for `worker2`.
4. On each clone (via its VM console, since its IP may differ):

```bash  
sudo hostnamectl set-hostname worker1          # worker2 on the other  
sudo sed -i 's/cp1/worker1/g' /etc/hosts  
sudo rm /etc/machine-id && sudo systemd-machine-id-setup  
sudo reboot  
```  

5. Note each worker's IP (`ip -4 a`), reserve it in the router, and confirm all three nodes have different IPs and hostnames.

Optional: stop automatic updates on the nodes:

```bash  
sudo dpkg-reconfigure unattended-upgrades       # answer No  
```  
  
---  

## 7. Create the cluster (cp1 only)

```bash  
sudo kubeadm init --pod-network-cidr=10.244.0.0/16  
  
mkdir -p $HOME/.kube  
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config  
  
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml  
```  

Save the `kubeadm join ...` command printed at the end. To regenerate it:

```bash  
kubeadm token create --print-join-command  
```  
  
---  

## 8. Join the workers

On each worker:

```bash  
sudo kubeadm join <cp1-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>  
```  
  
---  

## 9. Verify

On cp1:

```bash  
kubectl get nodes        # cp1, worker1, worker2 → Ready  
kubectl get pods -A      # all Running  
```  
  
---  

## Node inventory
example router dhcp assigned ip

| Hostname | IP           | Notes         |
| -------- | ------------ | ------------- |
| cp1      | 192.168.0.58 | control plane |
| worker1  | 192.168.0.59 |               |
| worker2  | 192.168.0.60 |               |
---

# network problems
It seems after the post went up in LinkedIn someone pointed out that my router dhcp assigned ip might break when network changed.  
Dhcp lease is temporary. It renews to same address but no guarantees.   
So I got recommended to use the two adapter layout: NAT for internet, host-only for cluster

# to start using static ip instead 
![two adapter network](/assets/images/two_adapter_network.png)  
the plan is to have this layout 
- NIC1 = NAT, for internet
- NIC2 = host-only k8s, with fixed IPs set inside each VM:  
  - cp1 192.168.56.10 
  - worker1 .11 
  - worker2 .12

First I restart the mac to refresh the vmnet

Then added a k8s host-only network with the range from 192.168.56.100 to 192.168.56.199  
then switch cp1 to using the host-only "k8s" network and verify it 
```
VBoxManage hostonlynet add --name=k8s --netmask=255.255.255.0 --lower-ip=192.168.56.100 --upper-ip=192.168.56.199 --enable && \
VBoxManage list hostonlynets && \
VBoxManage modifyvm cp1 --nic1=nat --nic2=hostonlynet --host-only-net2=k8s --cable-connected2=on && \
VBoxManage showvminfo cp1 --machinereadable | grep -E '^(nic[12]=|hostonly)')
```

## netplan 
In each of my vms, I powered off and use netplan instead

![netplan.yaml](/assets/images/netplan.png)    

| Hostname | IP           | Notes         |
| -------- | ------------ | ------------- |
| cp1      | 192.168.56.10 | control plane |
| worker1  | 192.168.56.11 |               |
| worker2  | 192.168.56.12 |               |

--- 

# cp1 setup 

This is the static config for host-only adapter.  
Use your own interface name. 
```host-only.yaml 
ls /etc/netplan/     # check for an earlier file that already sets enp0s9
sudo tee /etc/netplan/60-hostonly.yaml <<'EOF'
network:
  version: 2
  ethernets:
    enp0s9:
      dhcp4: false
      addresses: [192.168.56.10/24] # change this to the correct ip for the workers
EOF
sudo chmod 600 /etc/netplan/60-hostonly.yaml
sudo netplan apply
ip -br a
```
the results are:
```
# cp1
shirlim@cp1:~$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s8           UP             10.0.2.15/24 metric 100 
enp0s9           UP             192.168.56.10/24 
```

I needed to reset and restart the kubeadm on cp1 before setting up the workers
```
# reset 
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# restart
echo 'KUBELET_EXTRA_ARGS=--node-ip=192.168.56.10' | sudo tee /etc/default/kubelet
# tell kubeadm to use the 
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.10 \
  --pod-network-cidr=10.244.0.0/16
```
### flags explanation 
- apiserver-advertise-address is the address the API server announces to the rest of the cluster.
- pod-network-cidr is the range pods get their addresses from. It must not overlap with your VM networks (10.0.2.0/24 and 192.168.56.0/24 here).

## And..result!
```
Your Kubernetes control-plane has initialized successfully!
```


and then we'll see some more instructions for setting up network and kubeadm join commands prints out.
So following that, adding calico networking pods so that the control panel becomes READY

# setup network
To start using the cluster, run these: 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

## setup calico
```
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|; s|#   value: "192.168.0.0/16"|  value: "10.244.0.0/16"|' calico.yaml
grep -A1 CALICO_IPV4POOL_CIDR calico.yaml

   - name: CALICO_IPV4POOL_CIDR
      value: "10.244.0.0/16"
              
kubectl apply -f calico.yaml
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=cidr=192.168.56.0/24
kubectl get pods -n kube-system -w
```

![calico_created](/assets/images/calico_created.png)   

wait till its done ...

---

## workers setup 
```
# worker1
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
echo 'KUBELET_EXTRA_ARGS=--node-ip=192.168.56.11' | sudo tee /etc/default/kubelet
sudo kubeadm join 192.168.56.10:6443 --token pwxjww.rrtwskk8vh2vunjs --discovery-token-ca-cert-hash sha256:the_token
```
```
# worker2
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
echo 'KUBELET_EXTRA_ARGS=--node-ip=192.168.56.12' | sudo tee /etc/default/kubelet
sudo kubeadm join 192.168.56.10:6443 --token pwxjww.rrtwskk8vh2vunjs --discovery-token-ca-cert-hash sha256:the_token
```


##  workers results 
```
# worker1
shirlim@worker1:~$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s8           UP             10.0.2.15/24 metric 
enp0s9           UP             192.168.56.11/24

# worker2 
shirlim@worker2:~$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s8           UP             10.0.2.15/24 metric 
enp0s9           UP             192.168.56.12/24 
```

# verify network is working
## in cp1 after the workers join the cluster 
```
shirlim@cp1:~$ kubectl get nodes -o wide
NAME      STATUS   ROLES           AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION             CONTAINER-RUNTIME
cp1       Ready    control-plane   102m   v1.36.6   192.168.56.10   <none>        Ubuntu 26.04.1 LTS   7.0.0-43-generic (arm64)   containerd://2.2.2
worker1   Ready    <none>          81m    v1.36.6   192.168.56.11   <none>        Ubuntu 26.04.1 LTS   7.0.0-43-generic (arm64)   containerd://2.2.2
worker2   Ready    <none>          67m    v1.36.6   192.168.56.12   <none>        Ubuntu 26.04.1 LTS   7.0.0-43-generic (arm64)   containerd://2.2.2

# testing 
kubectl create deployment web --image=nginx --replicas=3
kubectl get pods -o wide
deployment.apps/web created
NAME                   READY   STATUS              RESTARTS   AGE   IP       NODE      NOMINATED NODE   READINESS GATES
web-7887448d46-g7jst   0/1     ContainerCreating   0          0s    <none>   worker2   <none>           <none>
web-7887448d46-hf9n6   0/1     Pending             0          0s    <none>   worker2   <none>           <none>
web-7887448d46-njr9j   0/1     ContainerCreating   0          0s    <none>   worker1   <none>           <none> 

...wait till its ready 

# expose service to internet 
kubectl expose deployment web --port=80 --type=NodePort
kubectl get svc web
service/web exposed 
NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web    NodePort   10.104.79.129   <none>        80:32239/TCP   0s

# check endpoints, we expect 1 for each replica
kubectl get endpoints web
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME   ENDPOINTS                                             AGE
web    10.244.189.66:80,10.244.189.67:80,10.244.235.130:80   2m57s

# test internal ip 
curl http://10.104.79.129
```

## verify from outside cluster 
```
# test nodes from mac 
curl http://192.168.56.10:32239
curl http://192.168.56.11:32239
curl http://192.168.56.12:32239

# test web pods, dns, pod to service 
kubectl get pods -o wide
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- sh -c "nslookup kubernetes.default; wget -qO- http://web | head -3"
```

## finally cleanup
```
kubectl delete deployment web
kubectl delete svc webß

```

