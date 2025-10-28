# How to Setup Kubernetes Cluster with Containerd CRI and Calico CNI in CentOS

## Initial setup

note: Centos default user name -> cloud-user

### Create User, Set time and Update
```
sudo -i
adduser ptsadmin
usermod -aG wheel ptsadmin
passwd ptsadmin #give password
groups ptsadmin
su - ptsadmin
sudo ls /root
timedatectl
timedatectl set-timezone Asia/Kolkata
timedatectl
dnf update -y
dnf clean all
dnf makecache
dnf update -y
setenforce 0
reboot
```
## Prerequisites
Before we begin the installation process, ensure you have the following prerequisites:

* A minimum of three nodes (one master and two worker nodes) running either Red Hat Enterprise Linux 9 or CentOS 9.
* Each node should have a minimum of 2GB RAM and 2 CPU cores.
* If you do not have a DNS setup, each node should have the following entries in the /etc/hosts file: 
```
# check ip
hostname -i

# check hostname
hostname
```
```
# Kubernetes Cluster
192.168.1.26 	master.naijalabs.net    # Replace with your actual hostname and IP address
192.168.1.27 	worker1.naijalabs.net   # Replace with your actual hostname and IP address
192.168.1.28 	worker2.naijalabs.net   # Replace with your actual hostname and IP address
```
## Install a Kubernetes Cluster

Step 1: Install Kernel Headers
```
sudo dnf install kernel-devel-$(uname -r)
```
Step 2: Add Kernel Modules
```
sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe overlay
```
```
cat > /etc/modules-load.d/kubernetes.conf << EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
overlay
EOF
```
Step 3: Configure Sysctl
```
cat > /etc/sysctl.d/kubernetes.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```
sysctl --system
```
Step 4: Disabling Swap
```
sudo swapoff -a

sed -e '/swap/s/^/#/g' -i /etc/fstab
```
Step 5: Install Containerd
```
#Add the Docker CE Repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

#Update Package Cache
sudo dnf makecache

#install the containerd.io package:
sudo dnf -y install containerd.io

#Configure Containerd
sudo sh -c "containerd config default > /etc/containerd/config.toml" ; cat /etc/containerd/config.toml
```
open the /etc/containerd/config.toml file and set the SystemdCgroup variable to true (SystemdCgroup = true):
```
vi /etc/containerd/config.toml
```
set > SystemdCgroup = true
```
sudo systemctl enable --now containerd.service
sudo systemctl start containerd.service
sudo systemctl status containerd.service
```
Step 6: Set Firewall Rules

If firewall is installed allow the following ports or disable the firewall:
```
sudo firewall-cmd --zone=public --permanent --add-port=6443/tcp
sudo firewall-cmd --zone=public --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10250/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10251/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10252/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10255/tcp
sudo firewall-cmd --zone=public --permanent --add-port=5473/tcp

sudo firewall-cmd --reload
```
or
```
systemctl disable firewalld

systemctl status firewalld
```
Step 7: Install Kubernetes Components

Add Kubernetes Repository
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Install Kubernetes Packages
```
dnf makecache; dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes


# Start and Enable kubelet Service
systemctl enable --now kubelet.service
```

## NOTE: Up until this point of the installation process, we’ve installed and configured Kubernetes components on all nodes. From this point onward, we will focus on the master node.

# Setup HAProxy Load Balancer

If the cluster requirement have multiple control-plane use HAProxy LB On a separate machine or on one of the master nodes:

```
# Install HAProxy
dnf install -y haproxy

# Configure HAProxy
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend k8s-api
    bind *:8443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server <HOSTNAME1> <MASTER1_IP>:6443 check
    server <HOSTNAME1> <MASTER2_IP>:6443 check
EOF

# Replace <MASTER1_IP> and <MASTER2_IP> with actual IPs
# Start HAProxy
sudo systemctl enable --now haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

Step 8: Initializing Kubernetes Control Plane

Let’s proceed with initializing the Kubernetes control plane on the master node. 
```
sudo kubeadm config images pull
```
```
sudo kubeadm init \
  --control-plane-endpoint="<MASTER1_IP>:8443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER1_IP>
```
After executing this command, Kubernetes will pull the necessary container images from the default container registry (usually Docker Hub) and store them locally on the machine. This step is typically performed before initializing the Kubernetes cluster to ensure that all required images are available locally and can be used without relying on an external registry during cluster setup.

Set Up kubeconfig File
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Deploy Pod Network
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Adjust the CIDR setting in the custom resources file:
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.244.0.0\/16/g' custom-resources.yaml

kubectl create -f custom-resources.yaml
```
list all the nodes in the cluster
```
kubectl get nodes

kubectl cluster-info
```
Step 9: Join Worker Nodes

Get Join Command on Master Node
```
sudo kubeadm token create --print-join-command
```
Verify Worker Node Join

After running the join command on each worker node, switch back to the master node and run the following command to verify that the worker nodes have successfully joined the cluster:
```
kubectl get nodes
```

## Important Notes

Token Expiration: The join token expires after 24 hours. Generate a new one:
```
kubeadm token create --print-join-command
```
Certificate Key: For adding control plane nodes later:
```
kubeadm init phase upload-certs --upload-certs
```
Node Labels: Label worker nodes if needed:
```
kubectl label node <node-name> node-role.kubernetes.io/worker=worker
```
Firewall Ports: If using firewall, open these ports:

* Master: 6443, 2379-2380, 10250, 10251, 10252
* Worker: 10250, 30000-32767