---
author: "Luqinthar Sudarsono"
title: "Deploying kubernetes with kubespray"
description : "Kubespray is automation tool based on ansible make us easier to deploy kubernetes cluster"
date: 2024-01-09T03:43:54+07:00
tags: ["kubernetes","linux","ansible"]
showToc: true
---
## Environtment:

This environment just for testing and not for HA (High Availability) method

| Node | IP Address | Note |
| --- | --- | --- |
| lq-deployer | 10.13.13.13 | kubespray deployer |
| lq-master | 10.13.13.10 | control plane |
| lq-worker | 10.13.13.20 | worker |

## Preparation

---

### 1. Make sure all node has pubkey of deployer

```bash
# On deployer
ssh key-gen

# and other node input the deployer pubkey into .ssh/authorized_keys
nano .ssh/authorized_keys
```

### 2. Setting up ansible

```bash
# Do it on deployer node
nano /etc/ansible/ansible.cfg

---
[defaults]
host_key_checking=False
pipelining=True
forks=100
---

# edit /etc/hosts
nano /etc/hosts

---
10.13.13.10 lq-master
10.13.13.20 lq-worker
---
```

### 3. Clone kubepsray git

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
```

### 4. Install dependencies

```bash
apt install python3 python3-pip python3-venv
```

## Settings kubespray deployment

---

### 5. Create python virtual environment

```bash
# Do it on deployer node
# create venv and activate 
python3 -m venv venv
source venv/bin/activate && cd kubespray
```

### 6. Install kubespray requirements using pip

```bash
pip install -U pip
pip install -r requirements-{ansible-version}.txt # 2.11 or 2.12
```

### 7. Copy sample configruation

```bash
cp -frp inventory/sample inventory/calico-online
```

### 8. Make inventory hosts.yaml

```bash
# Declare ip address
declare -a IPS=(10.13.13.10 10.13.13.20)

# Generate config file of inventory hosts.yaml
CONFIG_FILE=inventory/calico-online/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Edit the invetory file
nano inventory/calico-online/hosts.yaml

---
all:
  hosts:
    lq-master:
      ansible_host: 10.13.13.10
      ip: 10.13.13.10
      access_ip: 10.13.13.10
    lq-worker:
      ansible_host: 10.13.13.20
      ip: 10.13.13.20
      access_ip: 10.13.13.20
  children:
    kube_control_plane:
      hosts:
        lq-master:
    kube_node:
      hosts:
        lq-worker:
    etcd:
      hosts:
        lq-master:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
---
```

### 9. Edit configuration file k8s-cluster.yml

```bash

nano inventory/calico-online/group_vars/k8s-cluste/k8s-cluster.yaml

---
# line 20 define kubernetes version
kube_version: v1.25.4 # depending on what you want
# line 70 CNI plugin
kube_network_plugin: calico # caico,flannel .etc
# line 76 define internal network for service
kube_service_addresses: 10.233.0.0/18
# line 81 define pods subnet
kube_pods_subnet: 10.233.64.0/18
# line 224 define container manager
container_manager: containerd
```

### 10. Run ansible

```bash
# Check connectione between deployer and node
ansible -i inventory/calico-online/hosts.yaml all -m ping

# Run kubespray ansible
ansible-playbook -i inventory/calico-online/hosts.yaml --become --become-user=root cluster.yml
```

### Result

![Alt text](https://keyz.my.id/assets/images/kubespray.png)
