---
author: Luqinthar Sudarsono
title: Connecting k3s to private registry
description: In the previous article we have created a private registry, now
  it's time to connect k3s to that private registry.
date: 2024-12-13T02:31:00+07:00
showToc: true
tocopen: false
tags:
  - kubernetes
  - rancher
  - k3s
  - docker
  - containerd
  - registry
---
## Steps

### Tips: add k3s completion bash

Add bash completion

```bash
cat << EOF | tee -a ~/.profile
source <(sudo k3s kubectl completion bash)
alias k='kubectl'
alias kubectl='sudo k3s kubectl'
complete -o default -F __start_kubectl k
EOF
```

Reload profile

```bash
source ~/.profile
```

### Get the registry cert

```bash
$ sudo mkdir -p /certs/registry.lab7.local && cd /certs/registry.lab7.local
```

You can copy your certs to the node, or download it if you put on webserver, then the cert to the directory.

### Make registry configuration for k3s

```bash
$ sudo nano /etc/rancher/k3s/registries.yaml
```

```yaml
# add this, depends on our setting before
mirrors:
  "registry.lab7.local":
    endpoint:
      - "https://registry.lab7.local"
configs:
  "registry.lab7.local":
    auth:
      username: admin
      password: <YOUR_PASSWORD>
    tls:
      ca_file: "/certs/registry.lab7.local/server.crt"
```

Then restart your k3s service

```bash
$ sudo systemctl restart k3s.service
```

or , you can put the credential into a secret

```bash
kubectl create secret docker-registry <SECRET_NAME> --docker-server=<REGISTRY_URL> --docker-username=<USERNAME> --docker-password=<PASSWORD>
```

### Testing

Create a pod with image from the registry

```yaml
cat << EOF | tee nginx-lab7.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-lab7
spec:
  containers:
  - image: registry.lab7.local/nginx:alpine
    name: nginx
  dnsPolicy: ClusterFirst
  restartPolicy: Always
EOF
```

if you want to use imagePullSecret method

```bash
cat << EOF | tee nginx-lab7.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-lab7
spec:
  containers:
  - image: registry.lab7.local/nginx:alpine
    name: nginx
  imagePullSecrets       # add this
  - name: <SECRET_NAME>
  dnsPolicy: ClusterFirst
  restartPolicy: Always
EOF
```

```bash
$ kubectl apply -f nginx-lab7.yaml
```

Verify

```bash
$ kubectl get pod

#output
NAME                                 READY   STATUS    RESTARTS   AGE
nginx-lab7                           1/1     Running   0          2d12h

$ kubectl describe pod nginx-lab7 | grep -i image

#output
    Image:          registry.lab7.local/nginx:alpine
    Image ID:       registry.lab7.local/nginx@sha256:2c8018e59b9ce43bd27955c844c85667409a96ecaa5180fa663cd6008ccdc663
```

There you go, our k3s finally connect to a private registry :D

### Refference:

*   [https://docs.k3s.io/installation/private-registry](https://docs.k3s.io/installation/private-registry)