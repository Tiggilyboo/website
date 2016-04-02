+++
date = "2016-04-01T18:16:00-05:00"
title = "Docker Swarm on ARM"
description = "Steps to set up Docker and Docker Swarm on ARM boards"
topics= ["Blog"]
tags = ["Docker", "ARM", "Docker Swarm", "golang"]
comments=true
extralang=true
+++

<a href="https://github.com/docker/swarm">
  <img src="https://github.com/docker/swarm/blob/master/logo.png?raw=true" alt="Docker Swarm" style="max-width:100%;"/>
</a>

## Overview

So, a couple years ago when I was studying my bachelors, I did a project for an operating systems course on the ARM big little architecture and scheduling algorithms (You can even read my final paper [here](http://simonwillshire.com/files/OS_ARM_Cluster.pdf)). Fast forward a couple years, and I've still got some ARM boards floating around - After doing some reading into Docker Swarm, I thought I'd try it out without virtualizing everything on my main box. This guide documents more or less my steps to create a working swarm...

First off, this guide assumes that all nodes are,

* Networked, and have been identified by hostnames, and have SSH keys set up: recommend using a fairly fast but cheap backbone, like GB ethernet.
* ARM arch of course...

## TL;DR; Just give me the tar's!

* Docker Machine 0.4.1 Linux ARMv7 **[tar.gz](https://bitbucket.org/tiggilyboo/website/downloads/docker-machine.0.4.1.tar.gz)**
* Go 1.4.3 Linux ARMv7 (Bootstrap to build 1.5.2) **[tar.gz](https://bitbucket.org/tiggilyboo/website/downloads/go1.4.3.linux-armv7.tar.gz)**
* Go 1.5.2 Linux ARMv7 (To build Docker Swarm) **[tar.gz](https://bitbucket.org/tiggilyboo/website/downloads/go1.5.2.linux-armv7.tar.gz)**
* Docker Swarm Linux ARMv7 Container **[tar.gz](https://bitbucket.org/tiggilyboo/website/downloads/swarm.linux-armv7.tar.gz)**
* Docker Alpine Linux ARMv7 Image **[tar.gz](https://bitbucket.org/tiggilyboo/website/downloads/alpine.linux-armv7.tar.gz)**

Once you've got docker running, you may pull directly from dockerhub; the Hypriot packages should work (Build for Raspberry Pi's).

~~~bash
$ docker pull hypriot/rpi-swarm
$ docker pull hypriot/rpi-alpine-scratch
~~~

## Guide

Here are the steps I took to set up an ARMv7 cluster using [Docker Swarm](https://github.com/docker/swarm).

I'm using [ODroid U3's](http://www.hardkernel.com/main/products/prdt_info.php?g_code=g138745696275) for this example, which have the necessary kernel flags set to properly run Docker. To check if your kernel has the required flags, run the following before going any further:

~~~bash
$ curl -L https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh | /bin/bash /dev/stdin /path/to/.config
~~~

*Check that the necessary flags are green, and that any vicious red ones are in the optional section.*

### Installing Docker

~~~bash
$ apt-get install lxc aufs-tools cgroup-lite apparmor docker.io
~~~

### ARM Docker Images

I've used [this alpine](https://hub.docker.com/r/hypriot/rpi-alpine-scratch/) armhf image for my containers. However, you can find your own using: `docker search armhf-`, or `docker search hypriot`

and then pull the one you like, and test that it runs.

~~~bash
$ docker pull hypriot/rpi-alpine-scratch
$ docker run --rm hypriot/rpi-alpine-scratch echo 'image works!'
~~~

### Leader NFS for Images & Containers (Optional)

Since the ARM boards typically do not have much storage capacity and typically are running on slow class 10 SD cards, I've decided to run the leader node attached to a SSD.

#### SSD Host on leader node

~~~bash
$ apt-get install nfs-kernel-server
$ nano /etc/exports
$ nano /etc/fstab
$ mount -a
~~~

Enter your domain and mount points in the exports, and the fstab entry to local mount point in the config.

#### Client Node NFS mount

~~~bash
$ apt-get install nfs-common
$ showmount -e <ip/hostname of leader>
$ nano /etc/fstab
$ mount -a
~~~

Ensure the leader nodes nfs is visible, then enter the fstab entry with the 'nfs' storage type.

Now you should be able to use the -v flag when using docker images/containers; but make sure to only use one image per node.

### Building Docker Swarm

Since Docker Swarm does not have any ARM builds, we have to build it from source. It requires Golang 1.4 and later, so let's set that up...

#### Build Golang 1.4 to bootstrap Golang 1.5

~~~bash
$ apt-get remove golang
$ rm -fr /usr/local/go
$ curl -O https://storage.googleapis.com/golang/go1.4.3.src.tar.gz
$ tar -xzf go1.4.3.src.tar.gz -C /usr/local go
$ mv /usr/local/go /usr/local/go1.4
$ cd /usr/local/go1.4/src
$ time sudo ./make.bash
$ tar --numeric-owner -czf ~/go1.4.3.linux-armv7.tar.gz -C /usr/local go
~~~

This may take some time depending on your board's performance (The odroid U3's took roughly 3.5 minutes)... Next up, we package go into a tarball, and use it to bootstrap golang 1.5.

~~~bash
$ rm -rf /usr/share/go
$ tar -xzf ~/go1.4.3.linux-armv7.tar.gz -C ~/go1.4 --strip-components=1
$ curl -sSL https://storage.googleapis.com/golang/go1.5.2.src.tar.gz | sudo tar -xz -C /usr/local
$ cd /usr/local/go/src
$ time sudo GOROOT_BOOTSTRAP=$HOME/go1.4 ./make.bash
$ tar --numeric-owner -czf ~/go1.5.2.linux-armv7.tar.gz -C /usr/local go
~~~

*Pheeeeww*, we now have Go 1.5 to build Docker Swarm. Just install it like normal go:

~~~bash
$ rm -fr /usr/local/go
$ tar -xzf ~/go1.5.2.linux-armv7.tar.gz -C /usr/local
$ export PATH=$PATH:/usr/local/go/bin
$ go version
~~~

#### Build Docker Swarm (Finally!)

~~~bash
$ go get github.com/tools/godep
$ mkdir -p ~/go/src/github.com/docker
$ cd ~/go/src/github.com/docker/
$ git clone https://github.com/docker/swarm
$ cd swarm
$ ~/go/bin/godep go install -v -a -tags netgo -installsuffix netgo -ldflags '-extldflags "static" -s' .
~~~

This should spit out a swarm binary at `~/go/bin/swarm`.

#### Deploy Docker Swarm To All Nodes

Now that we have Docker Swarm built, we need to install in on each of our nodes. What better way to do this that create a Dockerfile! Create a new file at `~/go/bin/Dockerfile` with the following contents:

~~~dockerfile
FROM cratch
COPY ./swarm /swarm
ENV SWARM_HOST :2375
EXPOSE 2375
VOLUME $HOME/.swarm

ENTRYPOINT ["/swarm"]
CMD ["--help"]
~~~

**Note:** If you are confused as to what this does, check out the official **[Dockerfile reference](https://docs.docker.com/engine/reference/builder/)**.

Now package it up, and send it to each of your nodes. If you set up an NFS drive for your cluster, you could use it as a central access point for Docker data should you be worried about image/container storage (It gets huge, especially on small capacity SD cards).

~~~bash
cd ~/go/bin
docker build -t swarm:latest .
~~~

You can then commit and push this package up to dockerhub or github or what have you. *Just make sure you don't pull down the default 'swarm' package, as it will make your ARM board question its sanity.*
