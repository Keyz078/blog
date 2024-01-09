---
author: "Luqinthar Sudarsono"
title: "Setup ubuntu local repository with apt-mirror"
description : "This time let's build ubuntu local repository using apt-mirror"
date: 2024-01-09T20:33:01+07:00
tags: ["linux","repo","package","ubuntu"]
showToc: true
---
So, the repo that we will mirror is the whole focal repository from the official [ubuntu](http://archive.ubuntu.com/)

## Preparation

> Important: Make sure you have external disk with size minimum 500GB for the package

### 1. Install apt-mirror

```bash
# make sure you have an internet connection

sudo apt update
sudo apt install apt-mirror
```

### 2. Setup mirror.list

```bash
cd /etc/apt/mirror.list

############# config ##################
#
set base_path    /mirror # external disk
#
# set mirror_path  $base_path/mirror
# set skel_path    $base_path/skel
# set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
# set defaultarch  <running host architecture>
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     20
set _tilde 0
#
############# end config ##############

deb http://archive.ubuntu.com/ubuntu focal universe
deb http://archive.ubuntu.com/ubuntu focal main restricted
deb http://archive.ubuntu.com/ubuntu focal-updates main restricted
deb http://archive.ubuntu.com/ubuntu focal multiverse
deb http://archive.ubuntu.com/ubuntu focal-updates multiverse
deb http://archive.ubuntu.com/ubuntu focal-security main restricted
deb http://archive.ubuntu.com/ubuntu focal-security universe
deb http://archive.ubuntu.com/ubuntu focal-security multiverse

clean http://archive.ubuntu.com/ubuntu
```
## Mirroring

### 3. Run apt-mirror

> Note: I'd recommend you to run this on tmux


```bash
apt-mirror
```

It will take a lot of time depends on your internet speed

## Setup web server

In this tutorial i use apache2 for the web server

```bash
sudo apt update
sudo apt install apache2
```

### 4. Create symbolic link

```bash
ln -s /mirror/archive.ubuntu.com/ubuntu /var/www/html/ubuntu
```

so the path on repo site if we take a look it will be like this

`http://your-address/ubuntu`

### 5. Add cnf

Lately i got some issues after mirroring and testing the repo, it got error about cnf.
example:

```bash
E: Failed to fetch http://your-address/ubuntu/dists/focal/main/cnf/Commands-amd64  404  Not Found [IP: localhost 80]
E: Some index files failed to download. They have been ignored, or old ones used instead.
```

it seems like apt-mirror doesn't bring the cnf, so in order to solve that i make this script

```bash
#!/bin/bash
for p in "${1:-focal}"{,-{security,updates}}\
/{main,restricted,universe,multiverse};do >&2 echo "${p}"
wget  -c -r -np -R "index.html*" "http://archive.ubuntu.com/ubuntu/dists/${p}/cnf/Commands-amd64.xz"
wget  -c -r -np -R "index.html*" "http://archive.ubuntu.com/ubuntu/dists/${p}/cnf/Commands-i386.xz"
done
```

put the script at the same level path as `archive.ubuntu.com` for example /mirror/, then run the script.

## Verify

On the client side, you can add these following sources.list.

```bash
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal universe
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal main restricted
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal-updates main restricted
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal multiverse
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal-updates multiverse
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal-security main restricted
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal-security universe
deb [trusted=yes arch=amd64] http://your-address/ubuntu focal-security multiverse
```

And finally you can test it with `apt update` and install some package