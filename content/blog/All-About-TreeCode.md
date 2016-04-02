+++
date = "2016-02-27T18:15:52-05:00"
title = "All About TreeCode"
description = "This is my first post about a hobby project called TreeCode."
topics = ["Blog"]
tags = ["TreeCode", "Code Editor", "Go"]
comments=true
+++

### What is TreeCode?
It's a web-based code editor. What that's it? Well, the tree part comes in from tracking code file references.

As of writing this post, the project is written using Go or Golang as a root language for hosting a local webserver and serving content. The type of content is nothing
out of the ordinary, HTML5 & CSS3, JS, JQuery front end interactions. Sitting on top of this, I've selected some libraries to implement; The first of which is the [Ace
code editor](https://github.com/ajaxorg/ace) for a js code editor. The code editor has many common features such as syntax highlighting, intellisense-esque
predictions and key bindings. I have some uncertainty about the choice of graphing library to render tree structures and leaf nodes representing code files, however this
can be solidified once the underlying systems are written.

### Requirements ###

* User can create a new project
  * User can create a new code file in the project
  * User can save code file, program deciphers code extension in order to parse code references
* User can load a saved project
  * User can view the project in a tree structure with referenced parent/child relations
  * User can load a saved code file in the project

### Some UI Design ###

_A mockup for the file selection UI._

* This view shows a simple list of files in a new project, each project can be viewed in a tree view based upon file references.

![FileSelect](http://simonwillshire.com/images/TreeCode_FileSelect.jpg)

_A view of the ACE editor with syntax highlighting based on the filename extension._

* This view shows a single file being viewed, I plan to draw reference bars attached to the outside left and right of the editor. Once a bar is clicked, it will load the
referenced file. Alternatively, the user may view the parent references which will drawn above the editor.

![Editor](http://simonwillshire.com/images/TreeCode_Editor.jpg)


### Repository ###

View or clone TreeCode source [here](https://github.com/Tiggilyboo/TreeCode)
