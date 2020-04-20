---
layout: post
title: alpine 系统上安装 node-sass 报错
date: 2020-4-15
tags: rails
---

问题：用 `yarn install` 在『docker image』：ruby:2.6.5-alpine 上安装『node-sass』时报错，说缺少 python

原因是在『alpine』系统上『node-sass』需要编译，而编译需要用到 python。参见如下讨论：

[Docker 容器内安装 node-sass 失败](https://kongfangyu.com/2019/07/01/install-node-sass-failed-on-docker-container/)

[Why do I need Python?](https://github.com/sass/node-sass/issues/1176)

> Native Node JS extensions like node-sass are compiled via npm which uses node-gyp. Python is required by node-gyp. We supply a bunch of precompiled binaries with out releases so you don't have to compile them yourself.

> During the npm install process a binary is downloaded for your system. If you're on a system we haven't pre-compiled binaries for then npm will try to compile it for you, which is where python comes in.

> Since you're one linux there should be a binary for you. This likely means you're running very old version of node-sass. You should always update to the latest version of a package. Please open a new issue if you continue to encounter errors with node-sass@4.5.0.