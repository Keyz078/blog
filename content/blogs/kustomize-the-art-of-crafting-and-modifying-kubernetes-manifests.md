---
author: Luqinthar Sudarsono
title: "Kustomize: The Art of Crafting and Modifying Kubernetes Manifests"
date: 2025-05-23T16:25:00+07:00
cover:
  image: /images/kustomize-cover.png
  hiddenInList: true
showToc: true
tocopen: false
tags:
  - kubernetes
---
<p style="text-align: justify">Kubernetes manifests can get messy fast, especially when managing multiple environments. <strong>Kustomize</strong> helps you keep things clean by letting you customize YAML files without copying or rewriting them. In this article, we’ll explore how Kustomize simplifies and streamlines Kubernetes configuration.</p>

## What does Kustomize do?

let's say you already have a template for deployment, or you can generate sample deployment yaml from this command

```
kubectl create deployment <your deployment name> --image dummy --dry-run=client -o yaml
```

this will generate a deployment yaml, then you can delete the following lines

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null # this
  labels:
    app: my-app
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  strategy: {} # this
  template:
    metadata:
      creationTimestamp: null # this
      labels:
        app: my-app
    spec:
      containers:
      - image: dummy
        name: my-container
        resources: {} # this
status: {} # this 
```

### Change image

Create kustomization.yaml file

```yaml
resources:
- deployment.yaml
images:
- name: dummy # this must be match the image in the deployment.yaml
  newName: my-registry/my-image
  newTag: my-tag 
```

The **resources** section lists the files or manifests to customize, and the **images** section updates the deployment image.

```
kubectl kustomize .
```

The deployment manifest will be updated, and the image name and tag will change according to the kustomization file.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: my-registry/my-image:my-tag
        name: my-container
```

### Generators

<p style="text-align: justify">Kustomize also supports generators, such as <code>secretGenerator</code> and <code>configMapGenerator</code>, which creates resources from the input file(s).</p>

```bash
events {}

http {
    server {
        listen 80;
        location / {
            return 200 "Hello from ConfigMap Nginx!\n";
        }
    }
}
```

```yaml
configMapGenerator:
  - name: nginx-config
    files:
      - nginx.conf
```

```bash
kubectl kustomize .
```

This will generate a ConfigMap using the data from `nginx.conf`.

```bash
apiVersion: v1
data:
  nginx.conf: |
    events {}

    http {
        server {
            listen 80;
            location / {
                return 200 "Hello from ConfigMap Dev Nginx!\n";
            }
        }
    }
kind: ConfigMap
metadata:
  name: nginx-config-59k264tbg4
```

it's also work for `secretGenerator`

```bash
echo "admin:admin123" > credential
```

```yaml
secretGenerator:
  - name: admin-secret
    files:
      - credential
```

```bash
kubectl kustomize .
```

this will generate `Opaque` type secret and `base64` encoded

```yaml
apiVersion: v1
data:
  credential: YWRtaW46YWRtaW4xMjMK
kind: Secret
metadata:
  name: admin-secret-kh798fmhhc
type: Opaque
```