---
layout: post
title: Jenkins 在 git 拉代码时 timeout 错误
tags: jenkins
---

最近 Jenkins 在跑任务的时候总是出现『git timeout』，Jenkins 默认设置是 10 分钟，既然提高 github 的速度很困难，那就改一下 timeout 时间吧（以后干脆用国内的服务或者 gitlab）。

在『Source Code Management』模块中添加一个设置，最底下的『Additional Behaviours -> Add』选择『Advanced clone behaviours』。

![advanced-clone-behaviours](/assets/img/posts/2020/jenkins-git-timeout/advanced-clone-behaviours.jpg "Advanced clone behaviours")

『Timeout (in minutes) for clone and fetch operations』就是目标设置了。

![timeout-settings](/assets/img/posts/2020/jenkins-git-timeout/timeout-settings.jpg "Timeout Settings")

官方说明如下：

> Specify a timeout (in minutes) for clone and fetch operations.
This option overrides the default timeout of 10 minutes.
You can change the global git timeout via the property org.jenkinsci.plugins.gitclient.Git.timeOut (see JENKINS-11286). Note that property should be set on both master and agent to have effect (see JENKINS-22547).

单位是分钟，并且提供了全局配置的方法。