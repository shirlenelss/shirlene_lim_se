+++
date = '2026-11-15T19:32:57+01:00'  
draft = false  
title = 'Setting up 3 node cluster on mac'  
+++

# Homelab Kubernetes Cluster on macOS (VirtualBox)

I have decided to use as a 3 node kubeadm cluster for my studies. Intension is to see /etcd in action instead on rancher desktop.   
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

![[image-6.png]]

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
example assigned ip
| Hostname | IP           | Notes         |     |
| -------- | ------------ | ------------- | --- |
| cp1      | 192.168.0.58 | control plane |     |
| worker1  | 192.168.0.59 |               |     |
| worker2  | 192.168.0.60 |               |     |
