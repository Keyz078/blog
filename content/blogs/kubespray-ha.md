---
author: "Luqinthar Sudarsono"
title: "Deploying kubernetes using kubespray with HAProxy and Keepalived"
description : ""
date: 2024-01-09T03:58:39+07:00
tags: ["kubernetes","linux"]
showToc: true
---

## Topology:

![](https://keyz.my.id/assets/images/topologi.png)

> Loadbalancer Haproxy with keepalived as kube api server for the cluster, which provide High availability. The Loadbalancer machine gets virtual IP from keepalived to keep kube api server endpoints always availabilty, then the haproxy allow us to provide loadbalancing for each master node api server.
> 

---

## Loadbalancer preparation

### 1. Setup haproxy on loadbalancer node

```bash
# do it on LB1 and LB2

# Update package and install haproxy
apt update
apt install haproxy -y

# edit haproxy.cfg
nano /etc/haproxy/haproxy.cfg

---
listen kubernetes-apiserver-https
  bind *:6443
  mode tcp
  option log-health-checks
  timeout client 3h
  timeout server 3h
  server master-1 10.13.13.10:6443 check check-ssl verify none inter 10000
  server master-2 10.13.13.11:6443 check check-ssl verify none inter 10000
  server master-3 10.13.13.13:6443 check check-ssl verify none inter 10000
  balance roundrobin
---

# restart haproxy
systemctl restart haproxy.service
```

### 2. Setup keepalived on loadbalancer node

```bash
# do it n LB1 and LB2

# install keepalived
apt install keepalived

# edit keepalived.conf
nano /etc/keepalived/keepalived.conf

---
router_id LVS_DEVEL 
    enable_script_security 
} 

vrrp_script keepalived_health {
   script "/etc/keepalived/keepalived_check.sh"
   interval 1
   timeout 5
   rise 3
   fall 3
} 

vrrp_instance LB_K8S_CALICO_VIP { 
    state BACKUP 
    nopreempt 
    interface ens3 
    virtual_router_id 66 # 67 for LB2
    priority 100 # 101 for LB2
    authentication { 
        auth_type PASS 
        auth_pass 2kwz5Vqt 
    } 
    unicast_src_ip 10.13.13.110 # reverse for LB 2
    unicast_peer { 
      10.13.13.111 
    } 
    virtual_ipaddress { 
        10.13.13.150/32 
    } 

    track_script { 
        keepalived_health 
    } 
}
---

# make health check scripts
nano /etc/keepalived/keepalived_check.sh

---
#! /bin/bash

/usr/bin/ping -c 1 -W 1 virtual_ipaddress > /dev/null 2>&1
--

# restart keepalived
systemctl restart keepalived.service

# check vip
ip a show ens3
```

---

### Kubespray preparation

First of all, make sure you already setup the kubespray and ready to go.

### 3. Make virtual environment

```bash
python3 -m venv venv
source venv/bin/activate && cd kubespray
```

### 4. Make inventory file

```bash
cp -frp inventory/sample inventory/your-inventory-dir
nano inventory/your-inventory-dir/your-inventory.ini
---
[all]
master-1 ansible_host=10.13.13.10 ip=10.13.13.10
master-2 ansible_host=10.13.13.11 ip=10.13.13.11
master-3 ansible_host=10.13.13.12 ip=10.13.13.12
worker-1 ansible_host=10.13.13.21 ip=10.13.13.21
worker-2 ansible_host=10.13.13.22 ip=10.13.13.22
worker-3 ansible_host=10.13.13.23 ip=10.13.13.23

[kube_control_plane]
master-1
master-2
master-3

[etcd]
master-1
master-2
master-3

[kube_node]
worker-1
worker-2
worker-3

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
---
```

### 5. Testing connection

```bash
ansible-playbook -i inventory/your-inventory-dir/your-inventory.ini all -m ping 
```

### 6. Edit k8s-cluster.yml

```bash
nano inventory/your-inventory-dir/group_vars/k8s_cluster/k8s-cluster.yml

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
---
```

### 7. Edit all.yml

```bash
nano inventory/your-inventory-dir/group_vars/all/all.yml

---
## External LB example config
apiserver_loadbalancer_domain_name: "elb.master.local" # your domain or local domain
loadbalancer_apiserver:
  address: 10.13.13.150 # LB VIP
  port: 6443
---
```

### 8. Run kubespray ansible

```bash
ansible-playbook -i inventory/your-inventory-dir/your-inventory.ini --become --become-user=root cluster.yml
```

### 9. Enable bash completion

```bash
# Bash auto complete
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

---

### Your cluster is ready!