# 🛠️ Kubernetes Setup Guide

[**English**](README.en.md) | [**한국어**](README.md)

This guide provides instructions for installing and configuring a Kubernetes cluster.

## 🔧 Environment Setup

| Node Type | Device | OS |
| :--- | :--- | :--- |
| **Master Node** | Desktop | Ubuntu 22.04 |
| **Worker Node** | 2 × NVIDIA Jetson Orin Nano | JetPack 6.0 |

> **Note**: We use `containerd` as the Container Runtime.

### 1. Prerequisites

Run the following commands on all nodes (Master and Worker) to set up the environment.

#### 1.1 Disable Swap
Disable swap memory for Kubernetes stability.
```bash
sudo swapoff -a
```

#### 1.2 Load Network Modules
Load the necessary kernel modules.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

#### 1.3 System Control Parameters
Apply network bridge settings.
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2. Install Container Runtime (containerd)

```bash
sudo apt install -y containerd
```

#### Create and Modify Configuration
Generate the default configuration and enable `SystemdCgroup`.
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Edit `/etc/containerd/config.toml` and set `SystemdCgroup` to `true`.
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Restart the service.
```bash
sudo systemctl restart containerd.service
sudo systemctl status containerd.service
```

---

## 📦 Install Kubernetes

### 1. Configure Package Repository
Add the Kubernetes package repository.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings

sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

### 2. Install Tools (kubelet, kubeadm, kubectl)

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3. Initialize Cluster (Master Node Only)

Initialize the cluster on the Master Node.
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
> **Important**: Copy the `kubeadm join` command from the output. You will need it to connect worker nodes.

#### Configure User Access
Allow non-root users to run `kubectl`.
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Install CNI (Container Network Interface)
Install Flannel as the network plugin.
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 4. Join Worker Nodes

Run the `kubeadm join` command (saved from step 3) on your Worker Nodes.

```bash
sudo kubeadm join <Master_IP>:<Port> --token <Token> --discovery-token-ca-cert-hash sha256:<Hash>
```

> **Tip**: If you lost the token, generate a new one on the Master Node:
> ```bash
> kubeadm token create --print-join-command
> ```


