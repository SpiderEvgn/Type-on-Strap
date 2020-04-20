---
layout: post
title: HTTP 请求中的 caching
date: 2018-01-22
tags: http
---

cache-control 是在 HTTP／1.1 中定义的 header，它用来告诉浏览器或其他中间缓存介质如何以及怎样对本次请求资源进行缓存，并且效果优先于之前定义的用来处理 caching 的方案，比如 Expires。

### cache-control 的值

**"no-cache"** 

字面意思“不缓存”，但其实会存下来，意义在于无论如何都会向服务器发起请求。如果对于同一个资源通过 ETag 验证文件并没有改变，则不需要重新下载。

**"no-store"**

压根不会存储任何资源，每次请求都会重新对浏览器发送请求并下载资源，这个效果比较适合敏感信息，比如个人账户数据等。

**"public"**

接受缓存，很多时候不是必要的参数，因为显式的参数比如 “max-age” 就代表了可缓存。

**"private"**

对于用户浏览器是可缓存的，但是不允许其他的中间缓存，比如 CDN 缓存就失效了。

**"max-age"**

顾名思义，从当前这次请求算起，资源能够被缓存在浏览器多少时间以被重复使用，单位是秒。

### cache-control 决策树
![http-decision-tree](/assets/img/posts/2018/http-cache/http-cache-decision-tree.png "decision tree")

### 如何更新缓存文件？

所有的 HTTP 请求都会先去找浏览器缓存，如果有匹配则直接从缓存读取，既摆脱了网络延迟时间也减少了数据传输消耗。但是，如果服务器上的 CSS 文件有更新，那么本地浏览器的缓存在过期之前该如何响应呢？糟糕的是，并不能。比如设置了 max-age=86400，如果 URL 没有变化，即 CSS 只是修改了内容而文件名没有变化，则一天之内的请求都只会从浏览器缓存中读取未过期但已经过时的 CSS 文件，而压根不会对 web server 发起请求。

当然，你可以手动去清除浏览器缓存，但这其实根本算不上一个解决方案。那该如何同时兼顾客户端的高效缓存和服务端的快速迭代开发呢？目前比较好的方法就是在文件名中加入指纹，相当于给了文件一个版本号，用来识别文件是否更新。

### 在 Rails 中的应用

Rails 正是利用了这个方法来编译前段文件。在预编译阶段，Sprockets 会根据静态资源文件的内容生成 SHA256 哈希值，并在保存文件时把这个哈希值添加到文件名中。Rails 辅助方法会用这些包含指纹的文件名代替清单文件中的文件名。

例如，下面的代码：

```
<%= javascript_include_tag "application" %>
<%= stylesheet_link_tag "application" %>
```

会生成下面的 HTML：

```
<script src="/application-908e25f4bf641868d8683022a5b62f54.js"></script>
<link href="/application-4dd5b109ee3439da54f5bdfd78a80473.css" media="screen" rel="stylesheet" />
```

可以通过 config.assets.digest 初始化选项（默认为 true）启用或禁用指纹功能。

在正常情况下，请不要修改默认的 config.assets.digest 选项（默认为 true）。如果文件名中未包含指纹，并且 HTTP 头信息的过期时间设置为很久以后，远程客户端将无法在文件内容发生变化时重新获取文件。

### ETag

其实，这个指纹匹配在 HTTP header 已有类似的用法，那就是 ETag。
例如，一个文件已经过期了，但是文件内容却没有改，这时候如果还要再从服务器下载一遍就显得既浪费又愚蠢。ETag 的存在正是为了解决这个问题，当浏览器发起新的请求时只需附带上次请求得到的 ETag（利用 If-None-Match 参数），如果服务器检查发现 ETag 并未变化，则返回 304 告诉浏览器直接使用以前的版本即可，同时浏览器刷新本地缓存时间。
![http-control-highlight](/assets/img/posts/2018/http-cache/http-cache-control-highlight.png "http request")
![http-control](/assets/img/posts/2018/http-cache/http-cache-control.png "cache request")

---

## 参考资料：

* [https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn#cache-control](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn#cache-control)
* [https://ruby-china.github.io/rails-guides/asset_pipeline.html#in-production](https://ruby-china.github.io/rails-guides/asset_pipeline.html#in-production)