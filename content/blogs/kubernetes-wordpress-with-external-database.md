---
author: "Luqinthar Sudarsono"
title: "Wordpress kubernetes with external database and bind9"
description : ""
date: 2024-01-09T04:06:17+07:00
tags: ['kubernetes','linux','wordpress','deploy']
showToc: true
---
## Setup Database server

### 1. Install mariadb or mysql

```bash
apt install mysql-server -y
```

### 2. Configure mysql-server to bind 0.0.0.0

```bash
vim /etc/mysql/mysql.conf.d/mysqld.cnf => bind-address to 0.0.0.0
```

### 3. Create new user for wordpress

```bash
mysql -u root 
CREATE USER 'wp-user'@'%' IDENTIFIED BY '<YOUR_PASSWORD>'; GRANT ALL PRIVILEGES ON *.* TO 'wp-user'@'%' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'wp-user'@'%';
```

## Setup bind9

### 4. Install bind9

```bash
apt install bind9
```

### 5. Setup forward zone

```bash
nano /etc/bind/db.foo

;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns.foo.id. root.foo.id. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.foo.id.
@       IN      A       xx.xx.xx.xx
ns      IN      A       xx.xx.xx.xx
dbwordpress    IN      A       xx.xx.xx.xx 
```

### 6. Setup reverse zone

```bash
nano /etc/bind/db.ip
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     foo.id. root.foo.id. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      foo.id.
xx      IN      PTR     foo.id
xx      IN      PTR     dbwordpress.foo.id.
```

### 7. Setup named.conf

```bash
nano /etc/bind/named.conf.local

zone "foo.id" IN {
		type master;
		file "/etc/bind/db.foo";
};

zone "xx.xx.xx.in-addr.arpa" IN {
		type master;
		file "/etc/bind/db.ip";
};
```

### 8. Restart mysql

```bash
systemctl restart mysql-server
```

## Setup Wordpress on kubernetes

### 9. Create manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: dbwordpress.foo.id
        - name: WORDPRESS_DB_USER
          value: wp-user
        - name: WORDPRESS_DB_PASSWORD
          value: <YOUR_PASSWORD>
        ports:
        - containerPort: 80
          name: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

```bash
kubectl apply -f <file>
```

### 9. Create ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wp-ingress
  labels:
    app: wordpress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: wp-foo.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80
```

> Finnally just tunneling and access Wordpress by ingress host and enjoy :D
>
