---
author: "Luqinthar Sudarsono"
title: "Deploying kubernetes dashboard"
description : "Monitoring and managing kubernetes cluster with kubernetes dashboard"
date: 2024-01-15T14:50:12+07:00
tags: ["kubernetes","monitoring"]
showToc: true
---

## Preparation

### 1. Deploy metrics server

First, we need to deploy metrics server before deploying kubernetes dashboard

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

if you have a problem that metrics-server's pod not running, probably because tls issue, so add `--kubelet-insecure-tls` on `args` section of deployment, then rollout restart it.

## Kubernetes dashboard

### 2. Deploy kubernetes dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### 3. Make user credential

- Service account
```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

- Cluster Role Binding
```bash
kubectl apply -f - << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

- Getting a Bearer Token
```bash
kubectl -n kubernetes-dashboard create token admin-user
```
Copy the printed token

### 4. Exposing kubernetes dashboard
```bash
kubectl patch svc -n kubernetes-dashboard kubernetes-dashboard --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"},{"op":"replace","path":"/spec/ports/0/nodePort","value":30000}]'
```

Finally access the kubernetes dashboard through your nodePort and login with the token created before

### Result 

![kubernetes-dashboard](/images/kubernetes-dashboard.png)

## Refference

1. https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
2. https://github.com/kubernetes/dashboard/blob/v2.7.0/docs/user/access-control/creating-sample-user.md
3. https://github.com/kubernetes-sigs/metrics-server
