+++
date = "2016-03-31T11:35:00-05:00"
title = "Handy Linux Bash Snippets"
description = "Some bash snippets I can never remember, but are now written down somewhere..."
topics= ["Blog"]
tags = ["Linux", "bash", "snippet"]
comments=true
+++

# Handy Linux Bash Snippets

1. Always display nautilus (file browser) path (Usually triggered with Ctrl-L)

~~~bash
$ gsettings set org.gnome.nautilus.preferences always-use-location-entry true
~~~

2. Show an exported/external drives (For NFS) within a specified domain

~~~bash
$ showmount -e <ip/host>
~~~

3. Check if your kernel can run Docker properly (Handy for ARM boards)

~~~bash
$ curl -L https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh | /bin/bash /dev/stdin /path/to/.config
~~~
