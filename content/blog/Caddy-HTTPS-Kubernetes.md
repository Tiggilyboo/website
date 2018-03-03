+++
date = "2018-02-27T16:42:00+13:00"
title = "Automatic HTTPS Server using Caddy Reverse-Proxy on Google Kubernetes Engine"
description = "A guide on how to set up a caddy web server which reverse proxies to another kubernetes pod."
tags = ["Programming", "Docker", "Kubernetes", "GKE", "Caddy", "Reverse Proxy"]
topics = ["Blog", "Programming"]
comments=true
extralang=true
+++

## Easy HTTPS

It has always frustrated me that even the largest companies like Microsoft have had HTTPS certificate renewal issues. With the invention of technologies like [Let's Encrypt](https://letsencryptonline.com/) people can finally not have to pay a fortune for certificates, and renew using its API. To make this process even easier, there are web server's like [Caddy](https://caddyserver.com/) that implement this for you. There is no excuse anymore, just do HTTPS from the start. Here is one way of deploying it on Google's Kubernetes Engine!

### I'm lost, you've mentioned too many technologies

Yes, there are lot's of weird names in this industry, here is a very quick rundown on how they all fit together.

- **Let's Encrypt**: Verifies and signs your website so you can host using a secure https:// site.
- **Caddy**: A web server which uses the above
- **Docker**: A way of packaging all this stuff up neatly so that it can be hosted exactly the same way in the cloud.
- **Kubernetes**: One way of hosting docker containers in a cluster
- **Google Kubernetes Engine**: Uses the above, open access $150 free credit for a year, makes tutorials like these easy and accessible to people like you!

## Setup

Alright, let's begin. First up, you need to set up a local development environment so that you can tinker locally before pushing everything live.

### Prerequisite Software

Ensure you've installed the below, I've also included a handy bash script so that you can just grab the packages on any linux debian environment (ubuntu, mint, debian chicken spiced or whatever the cool kids are on about these days)...

#### Docker

```sh
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt-get update && sudo apt-get install docker-ce
```

#### Google Cloud (gcloud with kubectl components)

```sh
$ echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
$ curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ sudo apt-get update && sudo apt-get install google-cloud-sdk
```

Next up, ensure you have a google account, and have signed up on [Google Cloud](https://console.cloud.google.com). Create a new project, and create a new cluster under it. then run:

```sh
$ gcloud init
```

Specify the project you have created and the nearest region to you or where you whish to deploy to. After, we need to install `kubectl` with:

```sh
$ gcloud components install kubectl
```

Navigate to your cloud console clusters and hit the big-ole `Connect` button. A dialog will come up with akin to:

`gcloud container clusters get-credentials alc-backend --zone australia-southeast1-a --project alcohol-survey-149200`

Run the line in the dialog (similar to the above) to configure your kubectl CLI. Afterwards, you can run things like `kubectl get pods` to return all pods which are running in your cluster. You can even run a bash session on a running container by using: `kubectl exec -it <pod-name> bash`!

## Docker Images

For the purpose of this guide, I use a minimal debian base image, more [here](http://simonwillshire.com/blog/Docker-Minimal-Debian-Image/). You can proceed one of two ways, pull my prebuilt images, or write your own using a provided configuration.

**Pull TLDR; Method**

```sh
$ docker pull tiggilyboo/https
```

**DIY Dockerfile Method**

```dockerfile
FROM tiggilyboo/base
LABEL description="Caddy HTTPS rproxy"
MAINTAINER Simon Willshire <me@simonwillshire.com>

ENV DEBIAN_FRONTEND=noninteractive \
    BUILD_PACKAGES="tar curl git wget libcap2-bin ca-certificates openssh-client"

RUN \
  apt-get update && apt-get install --no-install-recommends -y $BUILD_PACKAGES && \
  mkdir -p /config && \
  curl "https://caddyserver.com/download/linux/amd64" | tar --no-same-owner -C /usr/bin/ -xz caddy && \
  /usr/sbin/setcap 'cap_net_bind_Service=+ep' /usr/bin/caddy && \
  apt-get autoremove -y && \
  rm -rf /var/lib/apt/lists && \
  rm -rf /var/cache/apt 

EXPOSE 80 443
CMD ["caddy"]
```

### Configuration

Up until now, you've been handed a boatload of software, what comes next is the most important to get right - We need to configure Caddy in such a way that it points to a running container in the same cluster. Also, we need to be able to identify ourselves to Let's Encrypt so that it can start serving us up fresh certificates.

The way that I have configured caddy to run is by using another Dockerfile which inherits from tiggilyboo/https, if you built from the configuration, remember to change the name of the `FROM` keyword.

```dockerfile
FROM tiggilyboo/https
COPY ./entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

All this does is sets up the container to run with the entrypoint on startup. The `entrypoint.sh` file injects caddy with a configuration and starts it up:

```sh
#!/bin/bash
mkdir -p /config

echo "
some.domain.name.here {
  proxy / $POD_IP:80 {
    transparent
  }
  gzip
  tls letsencrypt-email@address.com 
}" >> /config/Caddyfile
echo "Loading caddy config: "
cat /config/Caddyfile
if [[ -z "${DEV}" ]]; then
  caddy -quic --conf /config/Caddyfile --log stdout
else
  echo "In development mode, https is disabled."
fi
```

Let's break down some of the components:
 - `some.domain.name.here`: Enter your full domain name with subdomain if required here.
 - `$POD_IP`: Environmental variable containing the running pod's IP address to communicate with. This can be whatever you like, just export the variable from a grep ifconfig or whatever.
 - `letsencrypt-email@address.com`: An identifiable email address to sign up with Let's Encrypt with and renew your certificates.

In essense, the above script injects a caddyfile. Caddyfile's are configurations which Caddy server uses, in this case we are directing traffic with a reverse proxy `proxy` at root `/` into `$POD_IP:80` at port 80 (At your existing web server container / pod).

Great, we should have a working Caddy configuration and we can now build the docker images, we use `docker-compose` for this, you can install it via:

```sh
$ sudo curl -L https://github.com/docker/compose/releases/download/1.19.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

Next up, we need to write a docker-compose.yml file so that we can set up our environment to build and test locally, if you are not familiar with this process, check out some of [their documentation](https://docs.docker.com/compose/compose-file/#service-configuration-reference).

Write a new file `docker-compose.yml`:

```yaml
version: '3'
services: 
  web:
    build: 
      context: ./web
    volumes:
      - ./web:/src/web
    ports:
      - "80:80"
    links: 
      - db
  https:
    build:
      context: ./https
    ports:
      - "80:80"
      - "443:443"
    links:
      - web
    environment:
      - POD_IP=172.18.0.1
      - DEV=true
```

In the above example, we set up two images, web and https. It assumes we have a Dockerfile located in ./web and ./https that will run.
The https image maps port 80 and 443 (http and https) ports to the local machine.

Alright, we can finally build the docker containers and push to our kubernetes cluster in the next section!

```sh
$ docker-compose build
$ docker images 
```

If you want to just run and test it on your local machine, you can just do `docker-compose up`. Though you won't be able to test your https certificate unless you are hosting on the same public IP as your DNS record states on your domain (Let's Encrypt won't be able to assign your certificate on your local machine). This is why I've added a development flag to disable the https container.

### Google Kubernetes Engine

After creating the images in the last section, you will need to tag your images to associate them to your poject and upload to your Google Cloud image repository. There is more information on this [here](https://cloud.google.com/kubernetes-engine/docs/tutorials/hello-app).

```sh
$ docker tag image_name gcr.io/my-project/hello-app
$ gcloud docker -- push gcr.io/my-project/hello-app 
```

Once this completes, you can now apply your kubernetes elements and host your https web server. Here is an example configuration:

kube-web.yml:

```yml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: default
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: web
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: web
spec:
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - image: gcr.io/my-project/web
          name: tmu-web
```

kube-https.yml

```yml
apiVersion: v1
kind: Pod
metadata:
  name: https
  labels:
    name: https
spec:
  containers:
  - image: gcr.io/my-project/https
    imagePullPolicy: Always
    name: https
    env:
      - name: POD_IP
        value: web
    ports:
      - containerPort: 80
        name: web-port

```

Then apply it to your Google Kubernetes cluster:

```sh
$ kubectl apply -f kube-web.yml
$ kubectl apply -f kube-https.yml
```

**Reminder**: Set your gcr.io/my-project to your project tag listed in `docker images` after you pushed your containers!


Once you've applied both files, you should be able to log to your cloud console and see the services hosted. You'll need to create a load balancer or make the https image hosted on a public IP. Hook up this public IP to your DNS A redirect or however you wish to set it up to your domain name.

And lastly, to debug: check that your https container has a success handshake and setting of your certificate. 
It should look similar to the below...

```
https	Feb 27, 2018, 4:12:34 PM	2018/02/27 03:12:34 http://www.somedomain.com
https	Feb 27, 2018, 4:12:34 PM	http://www.somedomain.com
https	Feb 27, 2018, 4:12:34 PM	2018/02/27 03:12:34 http://somedomain.com
https	Feb 27, 2018, 4:12:34 PM	http://somedomain.com
https	Feb 27, 2018, 4:12:34 PM	2018/02/27 03:12:34 https://www.somedomain.com
https	Feb 27, 2018, 4:12:34 PM	https://www.somedomain.com
https	Feb 27, 2018, 4:12:34 PM	2018/02/27 03:12:34 https://somedomain.com
https	Feb 27, 2018, 4:12:34 PM	https://somedomain.com
https	Feb 27, 2018, 4:12:34 PM	done.
https	Feb 27, 2018, 4:12:33 PM	2018/02/27 03:12:33 [INFO][www.somedomain.com] Certificate written to disk: /root/.caddy/acme/acme-v01.api.letsencrypt.org/sites/www.somedomain.com/www.somedomain.com.crt
https	Feb 27, 2018, 4:12:33 PM	2018/02/27 03:12:33 [INFO][www.somedomain.com] Server responded with a certificate.
https	Feb 27, 2018, 4:12:33 PM	2018/02/27 03:12:33 [INFO] acme: Requesting issuer cert from https://acme-v01.api.letsencrypt.org/acme/issuer-cert
https	Feb 27, 2018, 4:12:31 PM	2018/02/27 03:12:31 [INFO][www.somedomain.com] acme: Validations succeeded; requesting certificates
https	Feb 27, 2018, 4:12:31 PM	2018/02/27 03:12:31 [INFO][www.somedomain.com] The server validated our request
https	Feb 27, 2018, 4:12:30 PM	2018/02/27 03:12:30 [INFO][www.somedomain.com] Served key authentication
https	Feb 27, 2018, 4:12:29 PM	2018/02/27 03:12:29 [INFO][www.somedomain.com] acme: Trying to solve HTTP-01
```

Et Voila! A signed active certificate. Once you get in the swing of things with this configuration - It's simple to set up and will automatically renew before the 3 month expiry.

If you have any questions, just chuck them in the comments below.

