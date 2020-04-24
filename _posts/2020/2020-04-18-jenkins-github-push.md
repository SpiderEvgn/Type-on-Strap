---
layout: post
title: Jenkins 集成 GitHub（下）—— Push
tags: jenkins
---

### 0. 引言

本文是 [『Jenkins 集成 GitHub（上）—— Pull Request』]({% post_url 2020/2020-04-17-jenkins-github-pr %}) 的下篇。配置 push 比 pr 要简单的多，无需 GitHub Pull Request Builder Plugin。

在上篇已经完成了基本的 Jenkins 配置，所以这里只要配置 webhook 和 Jenkins job。

### 1. 配置 GitHub Webhooks

如果你已经配置过 webhook，只需打开已有的 webhook 然后勾选『Pushes』权限即可。GitHub 会自动为你添加一个新的 webhook。当然，你也可以新建 webhook 然后在 payload 填写 `[your-jenkins-server]/github-webhook/`。

![webhook](/assets/img/posts/2020/jenkins-github-push/webhook.jpg "webhook")

### 2. 创建 job

* 创建 Freestyle project 并填写 GitHub project

同上篇。

* 添加 Source Code Management

按需设置目标 branch

![scm](/assets/img/posts/2020/jenkins-github-push/scm.jpg "scm")

* 在 Build Triggers 中选中 GitHub hook trigger for GITScm polling

![build-triggers](/assets/img/posts/2020/jenkins-github-push/build-triggers.jpg "build triggers")

这一步要注意，GitHub 插件要求有对 project repo 的 admin 权限，否则会有如下报错：

![no-admin-alert](/assets/img/posts/2020/jenkins-github-push/no-admin-alert.jpg "no admin alert")

并且在日志会有如下记录：

```
There are no credentials with admin access to manage hooks on GitHubRepositoryName[host=github.com,username=owner,repository=name]
```

所以，不能按照我在上篇说的把 GitHub plugin 设置成 bot user，下图中的 Credentials 必须换成项目的所有者。

![gh-plugin-config](/assets/img/posts/2020/jenkins-github/gh-plugin-config.jpg "gh plugin config")

***GitHub Pull Request Builder Plugin 中的 Credentials 不用更换，这个插件控制的是 pr 中同步 GitHub 消息的用户***

* 最后按需完成 Build 过程

### 3. 颜色输出

参考上篇[『Jenkins 集成 GitHub（上）—— Pull Request』]({% post_url 2020/2020-04-17-jenkins-github-pr %})可选择为 Jenkins 的 console 配色。

---

### 参考资料：

* [Jenkins与Github集成](https://www.cnblogs.com/weschen/p/6867885.html)