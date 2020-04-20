---
layout: post
title: 切到生产环境后 sidekiq 就失效了
date: 2018-10-08
tags: rails
---

### 错误尝试

一开始以为是时序的问题，在数据库还未存完 record 的时候异步就去调用它了，所以要用 after_commit 的 hook 去实现：[sidekiq_deserialize_error](https://gist.github.com/plashchynski/ba6cf5a55dd3a6c3d7d2)

但是实际情况并不是这样，无论如何尝试后台始终报错找不到 record

### 不同 container 的环境造成的问题

终于在发现这篇文章后恍然大悟，原来是 sidekiq 和 app 起在不同的 container，当 app 切到生产环境后，sidekiq 还留在开发环境，数据库不一致导致的问题：[Rails 4 Sidekiq根本找不到ActiveRecord对象](https://stackoverrun.com/cn/q/11672013)

> "您是否在与主应用程序相同的环境中运行Sidekiq？例如，应用程序可能使用`production`数据库，而Sidekiq可能在`development`中运行，因此可以从不同的数据库中读取"

最后更改生产环境的 docker-compose.override.yml 文件如下:

```
version: '3'
services:
  web:
    command: bash -c "rm -f /app/tmp/pids/server.pid && rails s -e production -p 3000 -b 0.0.0.0"
    environment:
      - RAILS_ENV=production

  worker:
    command: bundle exec sidekiq -e production -q mailers -q default
    environment:
      - RAILS_ENV=production
```

## 参考资料：

* [sidekiq_deserialize_error.md](https://gist.github.com/plashchynski/ba6cf5a55dd3a6c3d7d2)

* [Rails 4 Sidekiq根本找不到ActiveRecord对象](https://stackoverrun.com/cn/q/11672013)