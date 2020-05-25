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

### 4. 附加：push 指定分支时触发 Jenkins job

默认情况下，只要 webhook 将 push event 推送到 Jenkins，就会触发配置好的 Job，而 Job 中设置的 『Branches to build』参数是该 Job 所要 build 的 branch。因此，每次 push 无论是哪个 branch 都会触发该 Job 对指定 branch 的 build。

如果不希望 push 无关 branch 也触发 Job，而只有在 push 指定 branch 时才触发 Jenkins build，则需要额外配置『[Generic Webhook Trigger](https://plugins.jenkins.io/generic-webhook-trigger/)』（需要在 Jenkins 安装该插件）

安装完插件后，在『Build Triggers』中勾选启用『Generic Webhook Trigger』。

首先配置『Variable』和『Expression』,『Variable』是自定义的匹配参数，取名为 `branch` 是因为我们要比对的是分支名，当 push 的 branch 满足一定条件时才触发 Job。那具体的分支名是多少呢？那就要去 GitHub webhook 推送过来的 post body 里去找了。

![generic-trigger-1](/assets/img/posts/2020/jenkins-github-push/generic-trigger-1.jpg "generic-trigger-1")

去到 GitHub Settings -> Webhooks，这一次我们需要替换掉原来的 `/github-webhook/` 配置，地址改写为：`http://[jenkins-username]:[jenkins-password]@[jenkins-server-host]:[jenkins-server-port]/generic-webhook-trigger/invoke`

然后查看 post body，一开始没有传送记录，你可以去以前的 webhook 看，或者先启用查看 body 结构，最后再完成配置。总之，webhook 的底部『Recent Deliveries』点开一个推送记录，将看到如下结构：

![post-body](/assets/img/posts/2020/jenkins-github-push/post-body.jpg "post-body")

第一行的 `ref` 就包含了我们需要的 branch 名称，所以只要取到 `ref` 的值，然后正则匹配我们目标 branch 即可。

再回到 Jenkins，『Expression』的值 `$.ref` 就是 JSONPath 的语法获取 ref 的值。到这一步我们就获取到了 GitHub 推送过来的当次 push 的分支名，下一步就是匹配。

来到最后的『Optional filter』模块，『Expression』就是匹配的正则，对象就是刚才设置的变量名 `branch`，用 `$` 来引用，如下图：

![generic-trigger-2](/assets/img/posts/2020/jenkins-github-push/generic-trigger-2.jpg "generic-trigger-1")

到此，根据 push 到 GitHub 的分支来触发 Jenkins build 的功能就配置好了。

---

### 参考资料：

* [Jenkins与Github集成](https://www.cnblogs.com/weschen/p/6867885.html)

* [https://medium.com/@austin.johnson/triggering-jenkins-from-a-push-to-a-specific-branch-bitbucket-6effecb4b451](https://medium.com/@austin.johnson/triggering-jenkins-from-a-push-to-a-specific-branch-bitbucket-6effecb4b451)