---
author: "Luqinthar Sudarsono"
title: "Kubernetes Installation"
description : "Installing kubernetes with kubeadm and containerd as runc"
date: 2024-01-09T03:01:06+07:00
tags: ["kubernetes","linux"]
showToc: true
---

## Do it on all nodes

### 1. Prepare iptables

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

### 3. Prerpare kubeadm, kubelet, kubectl 

```bash
# prepare packages on all nodes
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# add GPG
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# add Repo
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo swapoff -a
```

### 4. Init Cluster

```bash
# init on the master-1 
kubeadm init --control-plan-endpoint <endpoint>:<port> --pod-network-cidr 10.244.0.0/16

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

### 5. Last, but not least is bash completion

```bash
# Bash auto complete
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

### Happy kubectl~~!