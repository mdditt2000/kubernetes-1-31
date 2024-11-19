# Multi-Cluster Kubernetes 1.31 Deployment using Rocky Linux 9.4

Demo on YouTube [video]()

This repo is created to document installing multiple kubernetes 1.31 clusters using Rocky Linux with Flannel and F5 BIG-IP. F5 BIG-IP uses F5 CIS to automate the networking with **StaticRouteSupport** Diagram below represents the deployment

![diagram](https://github.com/mdditt2000/kubernetes-1-31/blob/main/rockylinux/diagram/2024-11-19_10-47-59.png)

Rocky linux 9.4
Kubernetes 1.31
7 VMs

### Step 1: Set Hostname and Update Hosts file

```
hostnamectl set-hostname “master-cluster-1” && exec bash
hostnamectl set-hostname “worker-cluster-1” && exec bash
hostnamectl set-hostname “master-cluster-2” && exec bash
hostnamectl set-hostname “worker-cluster-2” && exec bash
hostnamectl set-hostname “master-cluster-3” && exec bash
hostnamectl set-hostname “worker-cluster-3” && exec bash
hostnamectl set-hostname “master-ha” && exec bash
```

Add the following entries in /etc/hosts file on each node.

```
10.192.125.116 master-cluster-1
10.192.125.117 worker-cluster-1
10.192.125.118 master-cluster-2
10.192.125.119 worker-cluster-2
10.192.125.120 master-cluster-3
10.192.125.121 worker-cluster-3
```

### Step 2: Disable Swap Space on Each Node

```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 3: Adjust SELinux and Firewall Rules for Kubernetes

```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux
```

On the **master node**, allow following ports in the firewall

```
firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp
firewall-cmd --permanent --add-port=4789/udp
firewall-cmd --reload
```

On the **Worker Nodes**, allow beneath ports in the firewall

```
firewall-cmd --permanent --add-port={179,10250,30000-32767}/tcp
firewall-cmd --permanent --add-port=4789/udp
firewall-cmd --reload
```

### Step 4: Add Kernel Modules and Parameters

```
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
```
modprobe overlay
modprobe br_netfilter
```

```
vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
```
```
sysctl --system
```

### Step 5: Install Conatinerd Runtime

```
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install containerd.io -y
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```
```
systemctl status containerd
```

### Step 6: Install Kubernetes tools

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

Step 7: Install Kubernetes Cluster on Rocky Linux 9

kubeadm init --pod-network-cidr=10.246.0.0/16   ---- Master only for cluster-1
kubeadm init --pod-network-cidr=10.246.0.0/16   ---- Master only for cluster-2
kubeadm init --pod-network-cidr=10.246.0.0/16   ---- Master only for cluster-3

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 8: Install Flannel Network for default 10.244 for cluster 1

kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml ---- Master only for cluster-1

```
net-conf.json: |
    {
      "Network": "10.245.0.0/16",  ---- change this value for cluster 2
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }

net-conf.json: |
    {
      "Network": "10.246.0.0/16",  ---- change this value for cluster 2
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
```
CIS [repo](https://github.com/mdditt2000/kubernetes-1-31/tree/main/multi-cluster-flannel/cluster-2/flannel)

### Step 9: Modify Flannel Install Network for default 10.245 and 10.245 for cluster 2/3

### Step 10: Rocky Linux requires disable to firewall or POD to POD wont work

```
systemctl disable firewalld
systemctl stop firewalld
```