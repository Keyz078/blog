---
author: "Luqinthar Sudarsono"
title: "Setup a simple private registry for k3s"
description : "So i'm using the default registry:v2 and apache2 for the frontend, also with self-sign ssl"
date: 2024-12-11T02:26:37+07:00
tags: ["kubernetes","docker","container","k3s","rancher"]
showToc: true
---

## Topology

![](https://keyz.my.id/assets/images/registry-topology.png)

## Deploying registry

### 1. Assuming you already have a k3s cluster, let's build the registry

- create basic auth
    ```bash
    $ mkdir -p ~/docker-registry/auth && cd ~/docker-registry
    $ htpassword admin gladiators88 > auth/htpasswd
    ```
    > admin for the user, and gladiators for the password (username password)

- deploying registry

    ```bash
    $ docker run -d --name registry \
      -v /root/docker-registry/creds/auth:/auth \
      -v /docker:/var/lib/registry \
      -e REGISTRY_AUTH=htpasswd \
      -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
      -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
      -p 5000:5000 \
      registry:2
    ```

- Makesure the container is running well
    ```bash
    $ docker ps -a | grep registry
    $ docker logs registry
    ```

### 2. Create self-signed ssl

You can use openssl to create a sans ssl for the registry, for example:

```bash
$ mkdir cert && cd cert

$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout server.key -out server.crt \
-subj "/C=ID/ST=Jakarta/L=Jakarta/O=Btech/OU=IT/CN=*.lab7.local" \
-addext "subjectAltName=DNS:*.lab7.local,DNS:lab7.local,IP:10.13.13.100"
```

### 3. Setup the frontend

Many tools you can set for the frontend but in my case i'm going to use apache2 this time.

- Apache2 Installation
    ```bash
    $ sudo apt update
    $ sudo apt install apache2
    ```

- Enabling some modules
    ```bash
    $ sudo a2enmod proxy
    $ sudo a2enmod proxy_http
    $ sudo a2enmod ssl
    $ sudo a2enmod rewrite
    $ sudo a2enmod headers

    $ sudo systemctl restart apache2
    ```

- Make configuration for docker registry

    ```bash
    $ sudo nano /etc/apache2/sites-available/docker-registry.conf
    ```

    ```
    <VirtualHost *:443>
        ServerName registry.lab7.local

        SSLEngine on
        SSLCertificateFile /root/docker-registry/cert/server.crt
        SSLCertificateKeyFile /root/docker-registry/cert/server.key

        ProxyPreserveHost On
        ProxyPass / http://10.13.13.100:5000/
        ProxyPassReverse / http://10.13.13.100:5000/

        # Set the X-Forwarded-Proto header to "https" so that Docker Registry knows the original protocol
        RequestHeader set X-Forwarded-Proto "https"

        <Location />
            Require all granted
        </Location>
    </VirtualHost>
    ```

    ```bash
    $ systemctl reload apache2
    ```

### 4. Testing

- Put the cert into docker path

    ```bash
    $ sudo mkdir -p /etc/docker/certs.d/registry.lab7.local
    $ sudo cp /root/docker-registry/cert/server.crt /etc/docker/certs.d/registry.lab7.local/
    ```

- Login to registry

    ```bash
    $ docker login https://registry.lab7.local
    ```
    > Use the credential we've been create befor


    If you successfully log in, here is the output

    ```bash
    $ docker login https://registry.lab7.local
    Username: admin
    Password:
    WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credential-stores

    Login Succeeded
    ```

- Pull and push

    ```bash
    $ doker pull nginx:alpine
    $ docker tag nginx:alpine registry.lab7.local/nginx:alpine
    $ docker push registry.lab7.local/nginx:alpine
    ```

    Output when success

    ```bash
    The push refers to repository [registry.lab7.local/nginx]                                                    
    2430c01bea64: Pushed                                                                                                                                                                                                       
    b11b58162504: Pushed                                                           
    8b5ce426f73d: Pushed                                                                                                                          
    884b72c14f15: Pushed                                                                                                             
    4a37d1b49911: Pushed                                                                      
    4e8a0009474a: Pushed                                                                                         
    287563f25f8b: Pushed                                                                                         
    75654b8eeebd: Pushed                                                                                         
    alpine: digest: sha256:2c8018e59b9ce43bd27955c844c85667409a96ecaa5180fa663cd6008ccdc663 size: 1989
    ```

- View registry contents
    ```bash
    $ curl -ku admin:gladiators88 https://registry.lab7.local/v2/_catalog
    {"repositories":["nginx"]}
    ```

Finally we have created a secure private registry, in the next article we will configure k3s to be able to pull to the private registry :D
