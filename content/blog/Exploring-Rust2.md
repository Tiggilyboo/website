+++
date = "2016-07-24T14:54:00-04:00"
title = "Exploring Rust - Part 2: Vulkan API"
description = "Taking a look at implementing Vulkan binding in rust using Vulkano"
tags = ["Programming", "Rust", "Vulkan"]
topics = ["Blog", "Programming"]
comments=true
+++

## Rust

I assume you have already installed it, if not check out the [first part](http://simonwillshire.com/blog/Exploring-Rust/) of the series. Vulkano recommends v1.9+ as of writing this article.

## Vulkan API

Part 2 will look at implementing a binding to the [Vulkan API](https://www.khronos.org/vulkan/). I'm assuming an introduction to the graphics API is not needed, and goes without saying - It's Khronos' new graphics API which is intended to replace OpenGL. So, for this post I'll be setting up Vulkan 1.0.2 on Ubuntu 16.04 - Which handily provides sufficient PPA's out of the box, so no need for running around collecting source code or install packages, you can just rely on the debian package manager!

However, please first make sure that your graphics card is supported (And update your graphics driver to be sure Vulkan is supported), otherwise you're in for some dissappointment...

```bash
$ sudo apt-get install lib-vulkan1
$ sudo apt-get install vulkan-utils
$ vulkaninfo
```

This should provide your graphics card capabilities under the Vulkan API, if you get an error, start googling...

### Vulkano

So, you've got vulkan running on your system with a capable graphics driver, now we have to set up [Vulkano](https://github.com/tomaka/vulkano), a wrapper that interfaces with Vulkan.

```bash
$ cargo install vulkano
```

Alternatively, you can `$ git clone http://github.com/tomaka/vulkano.git`

As vulkano is very early in development (v0.1.0 as of writing), you must build it yourself using `$ cargo build <path>`
