---
layout: post
title: Rails + Docker 跑测试报错找不到浏览器
date: 2020-02-14
tags: rails
---

### 0. 引言

在使用 docker 开发环境的时候，测试时不像本地可以直接调用主机浏览器，应用服务器容器一般是不会安装浏览器的。需要用到浏览器的测试情况我会在下一篇介绍，而有很多测试是用不到浏览器的，但是 Rails（6.0.1） 的测试框架默认就有浏览器依赖，这就给 docker 环境带来了问题，如何解决呢？

### 1. 找不到浏览器

先来看问题，笔者在跑第一个测试用例的时候遇到如下报错：

```
Webdrivers::BrowserNotFound:
  Failed to find Chrome binary.
```

简单看，在 container 里面装一下 chrome 就可以解决这个问题，或者说把 chrome 做到 image 里。但其实这是很不合理的，因为应用服务器是用不到浏览器的，徒增 image 容量。而真正的问题是，用不到浏览器的测试实际上是不用加载浏览器的，这其实是个 bug，而且是 Rails 的 bug。

### 2. 官方已经修复

稍微一查就发现，其实这个问题在前不久刚刚被修复，已经有 issue 并且 pr 已经被接受，详情见：[https://github.com/rails/rails/commit/ea303d012e6638c99f528c68ee9144a83e836e27](https://github.com/rails/rails/commit/ea303d012e6638c99f528c68ee9144a83e836e27)

所以如果你愿意，可以直接用 github 上最新的 Rails，但我想大部分情况下大家还是想用发布版的，好在这个 bug 修改起来就 3 行代码，所以我们完全可以自己到 gem 包里改一下。既然一步步做了，我们就来顺便看一下这个 bug 到底发生了什么。

### 3. 修改项目 gem 代码

最主要的就是两句话：

```
driven_by :selenium
---
@browser.preload
```

可见，Rails 默认会使用 `:selenium` 驱动，并且加载浏览器。而其实 Capybara 默认是使用 [`:rack_test`](https://github.com/teamcapybara/capybara#racktest) 驱动的，`:rack_test` 是不依赖浏览器的，当然也就不支持 JS 测试。所以，修复的代码就将其改为了：`@browser.preload unless name == :rack_test`，并且注释掉了 `driven_by :selenium`。

---

## 参考资料：

* [Issue #37410](https://github.com/rails/rails/issues/37410)
* [Issue #37476](https://github.com/rails/rails/issues/37476)
* [driver :rack_test](https://github.com/teamcapybara/capybara#racktest)