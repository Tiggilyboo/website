+++
date = "2017-09-06T12:08:00-12:00"
title = "Build your own tiny debian docker image"
description = "Create your own minimal base docker image to build on top of"
draft = false
tags = ["Docker", "Debian"]
topics = ["Blog"]
comments=true
+++

## Docker
This guide assumes you know the basics to docker and assumes it has been installed. This guide will be using a debian variant host environment.

## TL;DR
If you can't be bothered, you can grab my prebuilt 57MB base image by using:

```sh
$ docker pull tiggilyboo/base
```

alternatively use it by creating a dockerfile:

```dockerfile
FROM tiggilyboo/base
RUN <some bash commands>
CMD ["/bin/bash"]
```

## Why?
So, no matter what you are hosting on a docker container, a typical image can include all sorts of unneccisary packages - paired with the fact that docker provides a unique overlay file system which is additive, there is no reason to include these packages. This guide will show you how to create your own customized (and roughly 57MB) version of stable debian. You can then extend the image with various containers and go about your various docker-ing...

## Let's get Debian

Debian provides an existing method of pulling entire versions, you can do this through the debootstrap package which is provided by any debian variant package source. For the purpose of this guide, we will be creating a `stable` debian version.

```sh
$ sudo apt-get update && sudo apt-get install debootstrap
$ mkdir ~/base && cd ~/base
$ sudo debootstrap stable .
```

Twiddle your thumbs, brew some tea, as this pulls down the debian distro...

## Dockerize

Once it's finished, we need to import this into a docker image. Docker provides the `import` command for this, which extracts tar's. Let's tar and import:

```sh
$ tar -cxvf base.tar
$ docker import base.tar base:raw
```

We are calling this revision of base `:raw` in order to differenciate when we start adding diffs to the container. As previously mentioned, overlayfs only is additive. To correct this, we can repack everything into the same layer. More on this after we clean up this raw image. We need to create a container using the image:

```sh
$ sudo docker create -t -i base:raw /bin/bash
$ sudo docker start -a -i <your_docker_container_hash>
$ apt-get purge $(aptitude search '~i!~M!~prequired!~pimportant!~R~prequired!~R~R~prequired!~R~pimportant!~R~R~pimportant!busybox!grub!initramfs-tools' | awk '{print $2}')
$ apt-get purge aptitude
$ apt-get autoremove
$ apt-get clean
$ rm -rf /usr/share/man/??
$ rm -rf /usr/share/man/??_*
$ exit
$ sudo docker export <your_docker_container_hash> | sudo docker import - base:raw
```

For further customisation, check out this article, which goes through some [more in depth cleaning of debian](https://wiki.debian.org/ReduceDebian).

That's it! Now you can build on top of a minimal debian image, happy Docker-ing!

Sample Dockerfile image:

```dockerfile
FROM tiggilyboo/base
LABEL description="Base Debian Image"
MAINTAINER Simon Willshire <me@simonwillshire.com>
CMD ["/bin/bash"]
```

```sh
$ sudo docker build . -t base:latest
```

Tada!
