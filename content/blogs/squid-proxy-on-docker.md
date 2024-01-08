---
author: "Luqinthar Sudarsono"
title: "Squid Proxy on Alpine"
description : ""
date: 2024-01-09T04:31:58+07:00
tags: ["linux","docker","proxy"]
showToc: true
---

> So this time i’ll make proxy with squid on alpine container with docker, just allowing http and https port and certain site (.btech.id and youtube.com). Let’s start!
> 

### 1. Pull the base image of alpine

```bash
docker pull alpine
```

### 2. Create Dockerfile

```docker
FROM alpine

RUN apk update
RUN apk add squid openrc curl wget busybox nano

COPY squid.conf /etc/squid/squid.conf
COPY list.acl /etc/squid/list.acl
COPY run.sh /run.sh

CMD ["/run.sh"]
```

### 3. Create squid.conf

```bash
acl localnet src 172.17.0.0/16          # depending on you

acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 443         # https

acl whitelist dstdomain "/etc/squid/list.acl"
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow whitelist
http_access deny localnet
http_access deny all
http_port 3128
coredump_dir /var/cache/squid

refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern ^gopher:        1440    0%      1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
deny_info ERR_ACCESS_DENIED clamav_service_req
```

### 4. Create list.acl (domain that you want to allow)

```bash
nano list.acl
---
.btech.id
www.youtube.com
---
```

### 5. Create script for container

```bash
nano run.sh

---
#!/bin/sh

rc-service squid status
squid -s
squid -k reconfigure
sleep 10
tail -f /var/log/squid/access.log
```

### 6. Let’s build the new image

```bash
docker build $YOUR_PATH/Dockerfile -t btech/squid:v1.1
```

### 7. Run container with new image

```bash
docker run -itd -p 3128 --name squid btech/squid:v1.1
```