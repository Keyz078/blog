---
author: "Luqinthar Sudarsono"
title: "Deploying Harbor Registry"
description : "Container registry with Web Ui?, yep let's try Harbor :D"
date: 2024-12-18T10:05:00+07:00
tags: ["docker","kubernetes","container","harbor"]
showToc: true
---

## Preparation

### Get harbor offline/online

I use offline method for the installation

```bash
$ mkdir ~/harbor-registry && cd ~/harbor-registry
$ wget https://github.com/goharbor/harbor/releases/download/v2.11.2/harbor-offline-installer-v2.11.2.tgz
$ tar xzvf harbor-offline-installer-v2.11.2.tgz
```

You will need docker-compose this time

```
$ apt install docker-compose
```

## Deploy

### Setup configuration file

Go to your harbor directory, it should be like this:

```bash
.
├── common
│   └── config
│       ├── core
│       ├── db
│       ├── jobservice
│       ├── log
│       ├── nginx
│       ├── portal
│       ├── registry
│       ├── registryctl
│       └── shared
├── common.sh
├── docker-compose.yml
├── docker-compose.yml.bak
├── harbor.yml.tmpl
├── harbor.v2.11.2.tar.gz
├── harbor.yml
├── install.sh
├── LICENSE
└── prepare
```

Create harbor.yml

```bash
$ cp harbor.yml.tmpl harbor.yml
```

Edit some lines

```yaml
# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: harbor.lab7.local # this line

# http related config 
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 5001 # this line, i use 5001 since 5000 already used by registry2

## Disable/comment the https since we use nginx/apache as frontend

# https related config
#https:
  # https port for harbor, default is 443
#  port: 5001
  # The path of cert and key files for nginx
#  certificate: ""
#  private_key: ""
  # enable strong ssl ciphers (default: false)
  # strong_ssl_ciphers: false

harbor_admin_password: gladiators88 # this line

# Harbor DB configuration
database:
  # The password for the root user of Harbor DB. Change this before any production use.
  password: gladiators88 # this line

```

> You can configure the other lines or just let it default

Run the prepare script

```bash
$ ./prepare
```

it will setting up and create docke-compose.yml from harbor.yml

Next is bring up harbor

```bash
$ docker-compose up -d
```

Verify and make sure there's no error log

```bash
$ docker ps -a | grep -i harbor
$ docker logs <harbor container registry name>
```

For the apache2 site, you can cp from the registry we made earlier.

```bash
cd /etc/apache2/sites-available
cp docker-registry.conf harbor-registry.conf
```

Edit file

```bash
<VirtualHost *:443>
    ServerName harbor.lab7.local # this

    SSLEngine on
    SSLCertificateFile /root/docker-registry/cert/server.crt
    SSLCertificateKeyFile /root/docker-registry/cert/server.key

    ProxyPreserveHost On
    ProxyPass / http://10.13.13.100:5001/ # and this (the port)
    ProxyPassReverse / http://10.13.13.100:5001/

    RequestHeader set X-Forwarded-Proto "https"

    <Location />
        Require all granted
    </Location>
</VirtualHost>
```

Enable site

```bash
$ a2ensite harbor-registry.conf
$ systemctl reload apache2
```