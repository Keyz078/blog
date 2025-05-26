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

<p style="text-align: justify">let's say you already have a template for deployment, or you can generate sample deployment yaml from this command</p>

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

<p style="text-align: justify">The <strong>resources</strong> section lists the files or manifests to customize, and the <strong>images</strong> section updates the deployment image.</p>

```
kubectl kustomize .
```

<p style="text-align: justify">The deployment manifest will be updated, and the image name and tag will change according to the kustomization file.</p>

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

Kustomize also supports generators, such as **secretGenerator** and **configMapGenerator**, which creates resources from the input file(s).

**nginx.conf**

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

**kustomization.yaml**

```yaml
configMapGenerator:
  - name: nginx-config
    files:
      - nginx.conf
```

```bash
kubectl kustomize .
```

<p style="text-align: justify">This will generate a ConfigMap using the data from <code>nginx.conf</code>.</p>

```bash
apiVersion: v1
data:
  nginx.conf: |
    events {}

    http {
        server {
            listen 80;
            location / {
                return 200 "Hello from ConfigMap Nginx!\n";
            }
        }
    }
kind: ConfigMap
metadata:
  name: nginx-config-59k264tbg4
```

<p style="text-align: justify">it's also work for <code>secretGenerator</code></p>

```bash
echo "admin:admin123" > credential
```

**kustomization.yaml**

```yaml
secretGenerator:
  - name: admin-secret
    files:
      - credential
```

```bash
kubectl kustomize .
```

<p style="text-align: justify">this will generate <code>Opaque</code> type secret and <code>base64</code> encoded</p>

```yaml
apiVersion: v1
data:
  credential: YWRtaW46YWRtaW4xMjMK
kind: Secret
metadata:
  name: admin-secret-kh798fmhhc
type: Opaque
```

### Patches

<p style="text-align: justify">Patches in Kustomize are used to modify or override specific fields in existing resources without changing the original files. For example, we can use a patch to add a ConfigMap volume and mount it into the base Deployment without modifying the original file.</p><p style="text-align: justify"><strong>cm-patch.yaml</strong></p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      volumes:
        - name: nginx-config
          configMap:
             name: nginx-config # must be match with the name from configMapGenerator
      containers:
        - name: my-container
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
```

**kustomization.yaml**

```yaml
resources:
  - deployment.yaml
images:
- name: dummy
  newName: my-registry/my-image
  newTag: my-tag
configMapGenerator:
  - name: nginx-config
    files:
      - nginx.conf
patches:
  - path: cm-patch.yaml
```

```bash
kubectl kustomize .
```

<p style="text-align: justify">This will generate a Deployment manifest with the ConfigMap volume and mount already included.</p>

```yaml
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
---
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
        volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          name: nginx-config
          subPath: nginx.conf
      volumes:
      - configMap:
          name: nginx-config-59k264tbg4
        name: nginx-config
```

### Bases & Overlays

<p style="text-align: justify">Kustomize separates config into bases and overlays. Bases hold reusable resources, while overlays build on them with extra changes. Bases can be local or remote and don’t depend on overlays. Based on what we have done before, now we will use bases and overlays to make it more structured.</p><p style="text-align: justify">example:</p>

```bash
.
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    └── dev
        ├── kustomization.yaml
        ├── nginx.conf
        └── patches
            └── cm-patch.yaml
```

<p style="text-align: justify">The <strong>base</strong> holds the original template manifests, and the <strong>overlays</strong> contains the changes or patches applied to them, heres some new manifest.</p><p style="text-align: justify"><strong>base/service.yaml</strong></p>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: my-app
```

<p style="text-align: justify"><strong>base/kustomization.yaml</strong></p>

```yaml
resources:
  - deployment.yaml
  - service.yaml
```

**overlays/dev/kustomization.yaml**

```yaml
namePrefix: dev-
resources:
  - ../../base # reffering to base kustomization
images:
- name: dummy
  newName: nginx
  newTag: alpine
configMapGenerator:
  - name: nginx-config
    files:
      - nginx.conf
patches:
  - path: cm-patch.yaml
  - path: nodeport.yaml
```

**overlays/dev/patches/nodeport.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 31232
  type: NodePort
```

The rest still using the previous manifest

```yaml
kubectl kustomize .
```

```yaml
apiVersion: v1
data:
  nginx.conf: |
    events {}

    http {
        server {
            listen 80;
            location / {
                return 200 "Hello from ConfigMap Nginx!\n";
            }
        }
    }
kind: ConfigMap
metadata:
  name: dev-nginx-config-mb9bhc6c66
---
apiVersion: v1
kind: Service
metadata:
  name: dev-my-app
spec:
  ports:
  - nodePort: 32323
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: my-app
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: dev-my-app
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
      - image: nginx:alpine
        name: my-container
        volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          name: nginx-config
          subPath: nginx.conf
      volumes:
      - configMap:
          name: dev-nginx-config-mb9bhc6c66
        name: nginx-config
```

<p style="text-align: justify">You can also apply directly with the following command</p>

```bash
kubectl apply -k ./ # apply
kubectl get -k ./ # get component
```

```bash
NAME                                    DATA   AGE
configmap/dev-nginx-config-mb9bhc6c66   1      6s

NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/dev-my-app   NodePort   10.43.194.245   <none>        80:32323/TCP   6s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dev-my-app   1/1     1            1           6s
```