
+++
date = "2017-10-03T13:39:00-12:00"
title = "Hey, NativeScript!"
description = "A quick guide to set up a NativeScript mobile application using a Docker environment"
topics = ["Blog"]
tags = ["NativeScript", "Android", "Cordova", "Java", "iOS", "NodeJS", "Docker"]
comments=true
+++

## Purpose
Right, so you may have seen Ionic, or Cordova based hybrid mobile applications - But you ask yourself, is there a better way to write a cross-platform mobile application in native code without using browser based UI and rendering? Well, Telerik has been developing a solution called NativeSCript - It transpiles good old web code (JS, CSS) or the Angular/Typscript solution into native code. ie. Java (Android) and Swift/Objective C (iOS). More about it [here](https://www.nativescript.org/).

## Setup & Environment
For the purpose of this guide, I'll be using the following docker container, it includes the following:
  - Minimal debian base image
  - Android SDK 
  - Apache Cordova
  - Java 8 (Oracle) 
    - OpenJDK does not work, as seen [here](https://github.com/NativeScript/nativescript-cli/issues/2265) 
  - NodeJS

What is a docker container? Well I won't be really addressing that here, but you can read into it [here](https://www.docker.com/what-docker). Essentially it allows for us to set up software environments irrelevant of the host machine. It also saves us having to install a boatload of things which makes it much easier to get to the heart of the article...
So we will assume you've got it set up and running!

In combination this image takes a fair chunk of space, weighing in at 2.08GB.
Let's fetch it using:

```sh
$ docker pull tiggilyboo/nativescript
```

Alright, let's set up our mobile application using our new docker image! The image creates a volume at `/src` that you can mount to save your mobile application's repository into. To do this, execute the following:

```sh
$ mkdir -p $HOME/nsapp
$ docker run --rm -v $HOME/nsapp:/src --name nsapp -it tiggilyboo/nativescript
$ tns create mobile
$ cd mobile
```

Regarding the docker run arguments: If you want to connect a phone/tablet device, and run the application on it - you can use:
```sh
...
$docker run --rm -v $HOME/nsapp:/src --name nsapp --device=/dev/bus/usb:/dev/bus/usb:rwm -it tiggilyboo/nativescript
...
```

You will be presented with a couple little nativescript nags, input Y/n as you see fit and you will have our new mobile application called `mobile` set up!

For the purpose of this guide, we will only be setting up the application to compile to Android - to do this, we configure our app using:

```sh
$ tns platform add android
```

Good stuff now you should be able to run our application with,

```sh
$ tns run android
```

And that's it, you should be presented with a hello world android application on your device - As usual, if you have any issues or questions, chuck em' in the comments.
