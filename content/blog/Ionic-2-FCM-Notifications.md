+++
date = "2016-10-10T05:55:00-12:00"
title = "Ionic 2 Framework - Integrating FCM Notifications"
description = "I started using the Ionic Framework for work, and had noticed people struggling with the FCM integration for firebase messaging, so I wrote this tutorial to help everyone out!"
topics= ["Blog"]
tags = ["AngularJS 2", "Ionic Framework", "FCM", "Notifications", "Mobile"]
comments=true
+++

## What?
I don't know man, you clicked this link, you tell me...

### Ionic?
If you are unfamiliar with the [Ionic framework](http://ionicframework.com/docs/v2/), I'd start by checking out some of the docs and features before going through this. Essentially we will be using this framework to display content on a mobile platform (Android, iOS, WP), and be sending native notifications to the phone via [Firebase](http://) (A Google service for messaging, authentication, and a bunch of others).

Without going into the nitty gritty details, Ionic uses [Cordova](https://cordova.apache.org/) (supported by ye ole' Apache) to interface with the phone's various functions. There are other frameworks, such as [PhoneGap](http://phonegap.com/) which is quite popular as well. This tutorial uses a Cordova plugin called [FCMPlugin](https://www.npmjs.com/package/cordova-plugin-fcm) to communicate with Firebase which uses GCM.

Alternatively, Ionic offers various services which they call their [cloud platform](http://ionic.io/), one of these services is also notifications, however their are some costs associated with heavy use of these features. In this guide we will be setting up firebases messaging only.

So let's set Ionic Up, you will need the following:

* Install NodeJS (As of writing, I grabbed stable 4.2.6, so anything after that should do.)
* Install npm (Alternatively, you can install npm to install NodeJS using `sudo npm install -g n && sudo n`)
* Install Ionic `npm install -g ionic@2.1.0`

Sit around and watch the little twiddly cursor flip around, or stare off into space for a bit... Then create a new project folder to start your Ionic project in, and create a new version 2 project using the Ionic-CLI:

```sh
$ ionic start ionic-fcm --v2
$ cd ionic-fcm
$ ionic serve
```

That should fire up a new mobile app template, and invoke a new browser tab/window to display the content to debug with - This is handy if you want to change things on the fly without wasting time uploading the application to your phone every minute of the day.

### Firebase Messaging FCM / GCM?
Right so next up, let's set up Firebase, login create a new project (One for Android and one for iOS if you so choose). Click the **Project Settings** from the gear icon in the firebase console.

It should look something like this, provided Google has not redone their UI again:

![Firebase Messaging Setup](http://i.imgur.com/GqWYRbn.png)

* Android Setup: Download the **google-services.json** file, and place it in **./platforms/android/google-services.json**
* iOS Setup: Download the **GoogleService-Info.plist** file, and place it in **./platforms/ios/GoogleService-Info.plist**

## Let's do it!

Install the Cordova FCM plugin:
```sh
$ cordova plugin add cordova-plugin-fcm
```

Setup some sample code to receive notifications with:
```javascript
/// Throw this in a provider constructor you will use for notifications:
this.platform.ready().then(() => {
  if(typeof(FCMPlugin) !== "undefined"){
    FCMPlugin.getToken(function(t){
      console.log("Use this token for sending device specific messages\nToken: " + t);
    }, function(e){
      console.log("Uh-Oh!\n"+e);
    });

    FCMPlugin.onNotification(function(d){
      if(d.wasTapped){  
        // Background receival (Even if app is closed),
        //   bring up the message in UI
      } else {
        // Foreground receival, update UI or what have you...
      }
    }, function(msg){
      // No problemo, registered callback
    }, function(err){
      console.log("Arf, no good mate... " + err);
    });
  } else console.log("Notifications disabled, only provided in Android/iOS environment");
});
```

For more plugin documentation, check out the github repo [here](https://github.com/fechanique/cordova-plugin-fcm).

## How to send notifications to devices?
Assumedly you are hosting a service which provides REST/some interface to your application, send the device token to the service and affiliate it with the user. Once you have reason to send said user a notification, assemble a POST request from the service:

* POST: ```https://fcm.googleapis.com/fcm/send```
* HEADER:
  * ```Content-Type: application/json```
  * ```Authorization: key=AIzaSy*******************```

The Authorization header contents can be found in the previously mentioned \*.json and \*.plist files specific to your application.

```json
{
  "notification":{
    "title":"Notification title",
    "body":"Notification body",
    "sound":"default",
    "click_action":"FCM_PLUGIN_ACTIVITY",
    "icon":"fcm_push_icon"
  },
  "data":{
    "message": "Some message data to help render the content when received."
  },
    "to":"/topics/topicExample",
    "priority":"high",
    "restricted_package_name":""
}
```

* ```to``` Can specify a single device id, or you may broadcast to subscribed topics with many devices using the above ```/topics/<topicname>``` format.

### Topic Subscription
Assuming the service provides some list of topics, or the user is creating a topic, they may subscribe to one that firebase can use, using:

```javascript
FCMPlugin.subscribeToTopic('topicExample');
```

That should just about do it, you should be able to flesh out the UI and event handling from there - If you have any questions chuck em' in the comments section below!
