---
layout: post
title: nginx proxy 利用 X-Forwarded-Proto 传递 https 请求
tags: nginx laravel
---

### 源头

笔者在用 docker 部署环境的时候遇到了一个关于 ssl 的问题，所有的 https 请求都没有正确发送到内部的 nginx 容器。一步一步检查下来发现是 ngixn proxy 出的问题，具体的说明可以参考：[Purpose of the X-Forwarded-Proto HTTP Header](https://discuss.pivotal.io/hc/en-us/articles/115002797967-Purpose-of-the-X-Forwarded-Proto-HTTP-Header)。 简单说就是 nginx proxy 默认只传 http 的请求，这就造成了 https 请求被阻断的问题，X-Forwarded-Proto 头就是为了解决这个问题而生的。

### 尝试

在 nginx proxy 中配置了 `proxy_set_header X-Forwarded-Proto https;` 后，笔者同时打开 nginx proxy 和 nginx container 的 log 观察请求内容，`X-Forwarded-Proto` 头确实被传递到了 nginx container，可是网站仍然跑在 http 上，检查了各个环节试了很久都没有成功。

### 硬解

最后找到了这个帖子：[How I can force all my routes to be HTTPS not HTTP](https://laracasts.com/discuss/channels/laravel/how-i-can-force-all-my-routes-to-be-https-not-http?page=1) 。

于是笔者在无奈之下最后通过手动控制环境变量 `env('IS_HTTPS')` 来强制将整个 laravel 使用 https 伺服。

```
if (env('IS_HTTPS') == true) {
    \URL::forceScheme('https');
}
```

**最后，如果你找到了正确解决 X-Forwarded-Proto 传递 https 请求的方法，欢迎[邮件](mailto:xiajun.zhang@usingnow.com)告知，感谢！**

---

## 参考资料：

* [Purpose of the X-Forwarded-Proto HTTP Header](https://discuss.pivotal.io/hc/en-us/articles/115002797967-Purpose-of-the-X-Forwarded-Proto-HTTP-Header)

* [X-Forwarded-For的一些理解](https://www.cnblogs.com/huaxingtianxia/p/6369089.html)

* [How I can force all my routes to be HTTPS not HTTP](https://laracasts.com/discuss/channels/laravel/how-i-can-force-all-my-routes-to-be-https-not-http?page=1) 