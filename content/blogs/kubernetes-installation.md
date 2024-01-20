---
author: "Luqinthar Sudarsono"
title: "Kubernetes Installation"
description : "Installing kubernetes with kubeadm and containerd as runc"
date: 2024-01-09T03:01:06+07:00
tags: ["kubernetes","linux"]
showToc: true
---

## Do it on all nodes

### 1. Prepare module and sysconfig

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### 2. Prepare containerd

```bash
sudo apt-get update
apt install containerd -y
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

Edit SystemdCgroup to `true`
nano /etc/containerd/config.toml

sudo systemctl restart containerd
```

### 3. Prepare kubeadm, kubelet, kubectl 

```bash
# prepare packages on all nodes
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# add GPG
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# add Repo
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo swapoff -a
```

### 4. Init Cluster

```bash
# init on the master-1 
kubeadm config images pull # if you want to pull image first
kubeadm init --apiserver-advertise-address <address> --control-plan-endpoint <endpoint>:<port> --pod-network-cidr 10.244.0.0/16 --upload-certs

# print join command
kubeadm token create --print-join-command

# join as control-plane
kubeadm join --control-plan-endpoint <endpoint>:<port> --token <token> \
  --discovery-token-ca-cert-hash <hash> \
  --control-plane 

# join as worker
kubeadm join --control-plane-endpoint <endpoint>:<port> --token <token> \
  --discovery-token-ca-cert-hash <hash>
```
### 5. Installing CNI
I use calico for CNI

```bash
# get manifest
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O

# apply
kubectl apply -f calico.yaml
```

### 6. Last, but not least is bash completion

```bash
# Kube config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Bash auto complete
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

### Happy kubectl~~!

## Refference
1. https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
2. https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises