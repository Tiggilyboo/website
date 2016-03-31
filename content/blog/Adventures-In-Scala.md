+++
date = "2016-03-31T10:37:16-04:00"
description = "My transition into using a Scala web stack using the Play! Framework, PostgreSql, and Slick."
draft = false
tags = ["Scala", "Play", "PostgreSQL", "Slick"]
title = "Adventures In Scala - Part 1: play-scala-intro"
topics = ["Scala"]
+++

# Learning Scala Part 1 - play-scala-intro
So, as of writing this, my background is primarily C#. This post is the first part of a series which will document my transition into all things Scala. I will be writing a step by step on creating a simple web stack.

## My Setup

I'll be using Linux Mint 17.3, but most flavours of debian should work for any of my bash.

* PostgreSQL 9.3.11
* IntelliJ IDEA, Open JDK 1.7.0_95
* Scala 2.11 with SBT, SSP, HOCON
* Play! 2.5.1 Framework (Using IntelliJ's plugin)
* Slick

**Install our software packages**

```
$ apt-get update
$ apt-get install postgresql postgresql-contrib openjdk-7-jdk
$ wget https://www.jetbrains.com/idea/download/download-thanks.html?platform=linux
$ sudo tar -xvzf ideaIU-2016.1.1.tar.gz -C /usr/share/intellij/
$ cd /usr/share/intellij
$ sudo bash ./idea.sh
```

### Set up our PostgreSQL DB

By default, postgres creates a user named 'postgres', we want to login to them, and start up our postgres console in the default 'postgres' database.

```
$ sudo su postgres
$ psql -d postgres -U postgres
```

*Create a user and db. Connect to it, and set new user's permissions to rw.*

```
postgres=# CREATE USER scalaplay WITH PASSWORD '<Password>'
postgres=# CREATE DATABASE scalaplaydb;
postgres=# \c scalaplaydb
postgres=# REVOKE ALL ON DATABASE scalaplaydb FROM PUBLIC;
postgres=# REVOKE ALL ON SCHEMA public FROM PUBLIC;
postgres=# GRANT CONNECT ON DATABASE scalaplaydb TO scalaPlay;
postgres=# GRANT SELECT,UPDATE,INSERT,DELETE ON ALL TABLES IN SCHEMA public TO scalaplay;
postgres=# GRANT SELECT,UPDATE ON ALL SEQUENCES IN SCHEMA public TO scalaplay;
postgres=# GRANT USAGE ON SCHEMA public TO scalaplay;
```

*If you want to have new tables with privileges already granted for previous users, run this*

```
postgres=# ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT,UPDATE,INSERT,DELETE ON TABLES TO scalaplay;
postgres=# ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT,UPDATE ON  SEQUENCES TO scalaplay;
```

***Some handy user check commands if you're not sure what users can do what:***

```
postgres=# \du            ; Displays user roles with attributes/membership
postgres=# \l             ; Displays db list, with access privileges
postgres=# \z             ; Displays access privileges to schemas
```

*Let's make some sample tables to interact with later*
```
postgres=# CREATE TABLE <name>(...)
...
postgres=# \q
postgres=# exit
```

### Play 2.X Framework Template

*Grab a sample play framework project template to load into IntelliJ*
```
$ wget https://www.lightbend.com/activator/template/bundle/play-scala-intro
$ unzip play-scala-intro.zip -d $HOME
$ sudo ./$HOME/bin/activator ui
```

Select the 'scala-play-intro' template, and create it. This may take a bit of downloading Open the project in IntelliJ...

## Other Sources

* Follow the handy JetBrains Play 2.X tutorial [here](https://www.jetbrains.com/help/idea/2016.1/getting-started-with-play-2-x.html?origin=old_help)
* <span class="fa fa-github"></span> View or Clone the template [here](https://github.com/playframework/playframework/tree/master/templates/play-scala-intro)
