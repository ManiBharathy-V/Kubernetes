# Kubernetes Installation

TThis document provide a full step by step procedure to install k8 cluster in any Cloud platform (AWS, Azure, GCP,..) or BareMetal servers

## System Prerequisites

- At least 1 - Master-Node & 1 - Worker-Node
- At least 2 CPUs and 4GB RAM
- At least 20 GB Storage 

For the following installation I have used 
- Container Runtime Interface(CRI) - Containerd
- K8 version - v1.30 
- Container Network Interface (CNI)- Calico

## Step 1: Set Hostname, Set ip and Update System

```
sudo su
ip a
ip route show default
hostnamectl set-hostname masternode
vi /etc/hosts
```
*/etc/hosts* <br />
add the following and comment everthing else <br />
<your-private-ip\> masternode localhost<br />
```
cd /etc/netplan
ls
cp 50-cloud-init.yaml 50-cloud-init.yaml.old
ls
vi 50-cloud-init.yaml
```
*50-cloud-init.yaml*
```
network:
  ethernets:
    ens5:
      addresses:
      - <your-private-ip>/24
      gateway4: <your-gateway-ip>
      nameservers:
        addresses:
        - 8.8.8.8
```
```
netplan apply
ping 8.8.8.8
ufw allow 6443/tcp   # Kubernetes API server
ufw allow 2379:2380/tcp  # etcd
ufw allow 10250/tcp  # kubelet API
ufw allow 10259/tcp  # kube-scheduler
ufw allow 10257/tcp  # kube-controller-manager
ufw allow 179/tcp    # Calico/Flannel BGP (if used)
ufw reload
systemctl enable ufw
apt update && apt upgrade -y
reboot
ping 8.8.8.8
ping masternode
ping localhost
```

## Step 2: Disable Swap

```
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

## Step 3: Load Kernel Modules and Sysctl Settings

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

## Step 4: Installing a container runtime

```
apt install -y containerd
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
vi /etc/containerd/config.toml
```
*/etc/containerd/config.toml*<br />
Find: [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]<br />
    SystemdCgroup = false<br />
Change to: <br />
    SystemdCgroup = true<br />
```
systemctl restart containerd
systemctl enable containerd
```
## Step 5: Install kubeadm, kubelet, kubectl

```
apt-get update

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

## Step 6: initialize the cluster

```
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<your-private-ip> --cri-socket=unix:///var/run/containerd/containerd.sock
```
Copy the kubectl join token 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## Step 7: Deploy Pod Network to Cluster

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml
```
wait for 2 min then check
```
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```
--- 
## Setup Worker-Node

***Follow steps from 1 to 5***

***Join the Cluster***<br />
Paste the join command created from master node here<br />
>example: sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket=unix:///var/run/containerd/containerd.sock

```
kubectl get nodes
kubectl get pods -A
```