---
layout: post
title: 用 Docker 部署 Rails 的生产环境
date: 2020-01-08
tags: rails
---

### 0. 引言

本文详细介绍在 Ubuntu 系统上用 Docker 快速部署 Rails 生产环境的步骤，主要流程就是：安装 Nginx，安装 Docker 和 Docker Compose，下载项目代码，下载 docker image 和 gem 依赖包，前端编译，配置 Nginx，启动项目和 Nginx 完成发布。本文将会跳过环境准备部分，比如操作系统的安装配置，包括 git，Nginx 和 Docker 的安装等，直接从下载代码开始。

### 1. 下载代码库和依赖

首先当然是 `git pull` 把项目拉下来，至于 gem 依赖的话就看 image 是怎么做的了。笔者的 gem 是放在本地的，也就是说 image 是不包含 gem 包的，所以在部署的时候需要在服务器重新安装：

```
$ docker-compose run --rm web gem install bundler
$ docker-compose run --rm web bundle install
```

代码拉下来之后，通常需要创建数据库文件：

```
$ cp config/database.yml.example config/database.yml
```

关于生产数据库的信息如何保存也见仁见智了，笔者的做法是在服务器创建 `docker-compose.override.yml` 文件，并将敏感信息作为环境变量保存其中。与此同时，在该文件中为 web service 添加环境变量：`RAILS_ENV=production`。

然后再是将 master.key 上传到生产服务器。

### 2. 前端编译

```
$ docker-compose run --rm web rails assets:precompile
```

### 3. 配置 Nginx

```
vi /etc/nginx/sites-available/[project-name]

server {
  listen 80 default_server;
  server_name _;
  root /var/www/[project-name]/public;

  location / {
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://localhost:[port];
  }

  location = /favicon.ico { access_log off; log_not_found off; }
  location = /robots.txt  { access_log off; log_not_found off; }

  location ~ ^/(assets|packs)/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  access_log /var/log/nginx/[project-name].access.log;
  error_log /var/log/nginx/[project-name].error.log;

}

cd  /etc/nginx/sites-enabled
ln -s /etc/nginx/sites-available/[project-name] [project-name]
```

### 4. 启动项目和 Nginx

```
$ cd /var/www/[project-name]
$ docker-compose up -d
$ docker-compose exec web rails db:prepare
$ systemctl restart nginx
```

---

## 参考资料：

* [Rails 6 & Webpacker Settings for Production](https://dev.to/tcgumus/rails-6-webpacker-settings-for-production-1f1e)
* [Nginx 配置](https://github.com/rails/webpacker/blob/master/docs/deployment.md#nginx)
* [docker image 本地迁移](https://blog.csdn.net/u012149181/article/details/80332973)