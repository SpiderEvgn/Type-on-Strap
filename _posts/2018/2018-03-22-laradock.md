---
layout: post
title: laradock：Laravel on Docker
date: 2018-03-22
tags: laravel docker
---

### Rails 与 Laravel 的 Docker 部署比较

首先，为 Rails 打个广告。

将 Rails 部署到 Docker 非常简单，只需为每个 service 建一个 container 就行，关于如何用 Docker 部署 Rails 我之后会再写一篇。

Rails 5 开始使用 Puma 作为 web server，而 Laravel 是没有内置服务器的，虽然 Laravel 可以用 `php artisan serve` 命令启用 php 的本地开发服务器，但这毕竟是 php 内置而且 Laravel 官方推荐的是 Homestead（在用 laradock 部署 Laravel 之前笔者一直用的就是 Homestead，并且你需要知道 Homestead 也是靠安装 nginx 或 apache 提供服务的）。所以你会发现，laradock 包含了 nginx 和 php 两个 container，把整个环境搞的比 Rails 复杂多了（Rails 的应用 container 就一个，并不需要额外依赖 nginx 和 ruby 作为 service 提供额外服务）。不过这也是 laradock 存在的目的以及强大的地方，它把这些环境都集成好了，你只需要下载，开箱即用，免去了很多烦恼。

但真要比较两者的优劣，却要仁者见仁了。Rails 的 Docker 环境简单，但同时一切也要靠你自己搭，考验你整个的部署能力以及对 Docker 的理解。Laravel 的 Docker 环境复杂，但牺牲简便性换来的是较高的集成度，你只需要根据 laradock 手册做配置，就可以快速集成很多 service，比如：mysql、postgres、redis、elasticsearch、kibana。有了 laradock，你完全不用单独去考虑怎么部署数据库或队列 container，只要学习利用 laradock 就够了，因为它已为你包装好了一切。

***注：以上为笔者个人浅见，理解不对的地方，欢迎指正。***

### 正式开始

你可以查阅[laradock 官网](http://laradock.io/)，有详细的文档说明。下面我做一个快速的部署介绍，如果你只是要快速启用 laradock，按照我下面步骤做就行了。

* #### 1. 首先，进入项目目录，更新项目的 env 文件（你可以根据项目具体情况做配置）

```
$ cd [project_folder]
$ vi .env

DB_HOST=mysql         # mysql 是 laradock 默认的 service 名字，包括之后的 redis 等其他服务在内，可以根据项目具体情况更改
...
DB_USERNAME=root      # 这是 laradock 的 mysql 默认 root 用户名密码
DB_PASSWORD=root
...
REDIS_HOST=redis
```

* #### 2. 下载 laradock

```
$ git clone https://github.com/Laradock/laradock.git
```

* #### 3. 拷贝 laradock 的 env 文件

```
$ cd laradock
$ cp env-example .env
```

* #### 4. 启动 container

```
$ docker-compose up -d nginx mysql redis         # 你要根据 docker 安装情况确定是否加 sudo
```
这里补充一句，虽然这里只启了三个 container，但是如果你查看 docker-compose.yml，就会发现 `nginx depends_on php-fpm`，同时 `php-fpm depends_on wordspace`，所以最后你会发现其实你一共启动了 5 个 container。

* #### 5. 进入 container 配置项目

```
$ docker exec -it laradock_php-fpm_1 bash
$ php artisan migrate --seed
```

* #### 6. 打开浏览器，登录 http://localhost

成功！整个项目已经搬到了 docker 上，安装过程还是相当简单的吧。

### 配置 laradock

最后你可以自定义配置一些诸如端口的参数，一般都在 laradock/.env，举个例子：

打开 laradock/docker-compose.yml，找到 nginx 的 container 发现如下两行：

```
ports:
  - "${NGINX_HOST_HTTP_PORT}:80"
  - "${NGINX_HOST_HTTPS_PORT}:443"
```

如何自定义 nginx 的访问端口呢？即 NGINX_HOST_HTTP_PORT 环境变量在哪？答案就是 laradock/.env 啦。当你改了这个变量后，重启 container，你就可以用新的端口访问项目了：`http://localhost:[new_port]`

***注：laradock 的官方文档在 elasticsearch 这一节有误，用户名应该是 elastic，密码 changeme***

### 生产部署 laradock

这里我只简单提一点，就是如何将外网的域名映射到 docker 然后提供服务。

![docker-structure](/assets/img/posts/2018/laradock/structure.png "Docker Structure")

整个过程如上图所示。

对于使用 docker 的项目来说，Host 主机上的 nginx 其实只是充当一个反向代理或者是负载均衡的作用，当它收到 request 后，按照预先的配置转发到内部 docker nginx 对应的端口，然后由 docker nginx 处理请求。所以，对真实提供服务的 container 来说，他们面对的端口一直都只是 80（或443）而已，docker 项目之间包括 Host 主机上的项目端口都是不冲突的；对用户来说，访问的也都是默认端口（80或443），不会出现`www.example.com:8001` 这种带端口号的域名。

下面来看一下 Host 上面的 nginx 如何来转发外部请求：（一个简单的例子）

```
server
{
    listen 80;
    server_name www.example.com;
    location / {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8001;
    }
    access_log /var/log/nginx/example.access.log;
}
```

而在 Host 上通过 docker ps 查看 container，你的 docker 内的项目 nginx container 端口应该是这样的：（这里没做 https 所以只用到 80）
```
...    PORTS                                          Names
...    0.0.0.0:8001->80/tcp, 0.0.0.0:8011->443/tcp    laradock_nginx_1
```
***注：这两个端口的配置之前说过了，在 laradock/.env 里设置。***

到此，请求就被转发到 docker nginx 了，然后对 laradock 的 nginx container 完成最终的项目配置

```
$ cd [project_folder]
$ cd laradock/nginx/sites
$ vi default.conf

server {

    listen 80 default_server;                         # 整个文件配置成你自己的项目就行了
    listen [::]:80 default_server ipv6only=on;

    server_name localhost;

    ...
}
:wq

$ docker restart laradock_nginx_1
```

至此，大功告成。打开浏览器，访问域名`www.example.com`，项目成功打开，bravo！

### 最后提一个小坑

笔者在上传文件时碰到报错: `413 Request Entity Too Large`，这个问题非常明显，相信很多人也都碰到过，无非就是 php 和 nginx 的最大文件设置而已，几个参数如下：

```
client_max_body_size 20M;         # nginx
upload_max_filesize = 20M         # php
post_max_size = 20M               # php
```

在应用 docker 后，需要留意的是，request 请求经过了两层 nginx，虽然第一层 Host 的 nginx 只是提供一个代理转发功能，但也要配置正确。所以别忘了，要在 Host 和 container 两个 nginx 都配置才行，还有，别忘了重启。

---

## 参考资料：

* [laradock 官网](http://laradock.io/)
* [搭建nginx反向代理用做内网域名转发](http://www.ttlsa.com/nginx/use-nginx-proxy/)
* [Nginx: 413 Request Entity Too Large解决](https://www.iteblog.com/archives/1421.html)

