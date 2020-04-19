---
layout: post
title: Letsencrypt 快速部署 SSL 证书（ubuntu 16.04）
date: 2018-02-12
tags: ali
---

> 很早就想写一篇关于 letsencrypt 的介绍了，既实用又免费的 SSL 证书，部署还特别方便，虽然每张证书都只有三个月有效期，但有自动 renew 功能，绝对放心使用。正巧同事 Jonas 刚刚解决了一个关于 renew 的一个坑然后写了一篇 guide，我就直接拿过来参考并翻译了这部分。感谢 Jonas，这是[原文链接](https://gist.github.com/jonasva/be3f89f6a06f80692920b87afbb602bc)。

注：本文使用的系统环境是 ubuntu 16.04

### 安装最新的 certbot
certbot 是 letsencrypt 的证书管理工具，详情查看[官网](https://certbot.eff.org/#ubuntuxenial-nginx)。

```
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install certbot
```

### Nginx 配置 1（签发）

创建文件并添加代码：

```
$ sudo vi /etc/nginx/snippets/letsencrypt-acme-challenge.conf

location ^~ /.well-known/acme-challenge/ {
    default_type "text/plain";
    root /var/www/letsencrypt;
}
```

创建文件夹：
```
$ sudo mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
```

在项目 nginx 配置文件中把刚刚新建的文件引入：
```
server {

    listen 80;

    # ... server config ...

    include /etc/nginx/snippets/letsencrypt-acme-challenge.conf;

    # ... other configs ...
}
```

改变 nginx 配置文件后别忘了要 reload：
```
$ sudo /etc/init.d/nginx configtest
$ sudo /etc/init.d/nginx reload
```

环境配置完了，接下来就可以通过 certbot 签发证书了：
```
$ certbot certonly --webroot -w /var/www/letsencrypt/ --cert-name example.com -d example.com, www.example.com
```

成功的话你会收到 “Congratulations” 的提示，还会有一些说明比如告诉你证书文件放在哪里。

### Nginx 配置 2（安装）

证书签发成功后，接下来要做的就是在项目 nginx 中配置 SSL 证书。最基础的配置如下：
```
server {
    listen 80;
    server_name www.example.com;

    include /etc/nginx/snippets/letsencrypt-acme-challenge.conf;

    location / {
        return 302 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/fullchain.pem;

    include /etc/nginx/snippets/letsencrypt-acme-challenge.conf;

    # ... other configs ...
}
```

关于 SSL 详细的配置策略太多了网上有很多深入的文章，我这里贴一篇仅供参考（[nginx 的 SSL 配置优化
](https://www.linpx.com/p/ssl-configuration-optimization.html)）。举个例子，如果你在阿里云上买证书，它会有一些官方指定的配置：
```
server {
    listen 443;
    server_name localhost;
    ssl on;
    root html;
    index index.html index.htm;
    ssl_certificate   cert/example-cert.pem;
    ssl_certificate_key  cert/example-cert.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root html;
        index index.html index.htm;
    }
}
```

配置完成后依然别忘了 reload：
```
$ sudo /etc/init.d/nginx configtest
$ sudo /etc/init.d/nginx reload
```

到此，letsencrypt 签发的免费 SSL 证书就部署完成啦！

### 验证

在 server 上可以通过 certbot 检查证书的情况：
```
$ sudo certbot certificates

Saving debug log to /var/log/letsencrypt/letsencrypt.log

-------------------------------------------------------------------------------
Found the following certs:
  Certificate Name: example.com
    Domains: example.com www.example.com
    Expiry Date: 2018-05-12 03:46:05+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/example.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/example.com/privkey.pem
```

在客户端浏览器检查证书

* Safari

进入网页后，在地址栏 url 的左面有一个灰色小锁的 icon，点击它就能看到证书的详情了
![safari](/assets/img/posts/2018/ssl/safari.png "Safari SSL")

* Chrome

进入网页后，在地址栏 url 的左面有一个绿色的小锁加上‘Secure’字样（对比没有 SSL 的网站是一个圈加感叹号的 icon）
![chrome-1](/assets/img/posts/2018/ssl/chrome-1.png "Chrome SSL 1")
点击它然后在弹出下拉框中点击 ‘Valid’，就能看到证书的详情了
![chrome-2](/assets/img/posts/2018/ssl/chrome-2.png "Chrome SSL 2")

### 配置证书的 auto renewal（翻译自[原文链接](https://gist.github.com/jonasva/be3f89f6a06f80692920b87afbb602bc)）

letsencrypt 的证书每三个月就会过期，所以你需要在每次过期前 renew。幸运的是 certbot 默认会配置好自动的 renew，但有个问题在于当证书更新后 nginx 需要 reload 才能生效，然而 certbot 默认的自动更新配置并没有做这步，所以我们需要完成这步操作。

在 Ubuntu 16.04 上，certbot 默认配置的自动 renew 过程文档严重缺失，并且配置本身极具误导性：

* 在 /etc/cron.d/ 目录下有个 cronjob: certbot

* 在 /lib/systemd/system/ 目录下有个 timer: certbot.timer; service: certbot.service

只要存在正常工作的 timer，那么日常的 cronjob 就失效了（这是 Ubuntu 16.04 的特性）。你可以通过如下命令查看你的 timer：

```
sudo systemctl list-timers
```

一旦你确认了你的 timer 是正常工作的，接下来就是把 nginx 的 reload 逻辑加到这个 timer 服务当中去。编辑 /lib/systemd/system/certbot.service 文件并修改 ExecStart 如下：

```
ExecStart=/usr/bin/certbot -q renew --max-log-backups 200 --post-hook "/etc/init.d/nginx reload"
```

这条命令完成 2 件事：

* 在一张证书更新后自动 reload nginx
* 把 /var/log/letsencrypt 日志文件的 rotation 限制在 200（默认是 1000）

修改完成后我们需要重启系统 daemon：

```
sudo systemctl daemon-reload
```

auto renewal 服务现在就已经配置完成了。最后测试一下，在命令最后加上 `--dry-run` 参数，不报错就 OK 啦：

```
sudo certbot -q renew --max-log-backups 200 --post-hook "/etc/init.d/nginx reload" --dry-run
```

### 在阿里云的 CDN 上配置 HTTPS 的 SSL 证书

* #### 进入控制台，选择 CDN -> 域名管理

![cdn-panel](/assets/img/posts/2018/ssl/cdn-panel.png "CDN panel")

* #### 查看主页面列表，“HTTPS” 状态栏显示了该 CDN 配置是否成功启用了 HTTPS

![cdn-panel](/assets/img/posts/2018/ssl/status.png "status")

* #### 对需要开启 HTTPS 服务的 CDN 项，单击右侧的 “配置” 按钮，进入配置页面，并滑动到 HTTPS 配置处

![cdn-panel](/assets/img/posts/2018/ssl/https-config.png "HTTPS config")

* #### 单击 “修改配置” 按钮，在弹出框中选中 “开启” 并单击 “选择证书” 下拉框选中 “自定义上传”（右侧 “云盾证书服务” 可以进入阿里的证书管理，包括你之后将要上传的 letsencrypt 证书也会出现在这里，如果你直接在阿里购买证书，这里也可以直接选择。具体操作非常简单，这里就不赘述了）

![cdn-panel](/assets/img/posts/2018/ssl/https-upload.png "HTTPS upload")

* #### 回到服务器，复制之前新签发的证书内容，可以单击阿里提供的 “pem编码参考样例” 查看样例

```
# 公钥（注意要复制 BEGIN 和 END 行）
sudo cat /etc/letsencrypt/live/example.com/fullchain.pem
# 私钥
sudo cat ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem
```

* #### “强制跳转” 选择 “HTTP -> HTTPS”

* #### 单击 “确定”，回到配置页，“HTTPS设置” 已经完成了

![cdn-panel](/assets/img/posts/2018/ssl/https-upload-ok.png "HTTPS upload finished")

* #### 最后回到外层 CDN 列表 HTTPS 列显示 “已开启”

---

## 参考资料：

* [Jonas 的 gist](https://gist.github.com/jonasva/be3f89f6a06f80692920b87afbb602bc)
* [letsencrypt 官网](https://letsencrypt.org/getting-started/)
* [certbot 官网](https://certbot.eff.org/#ubuntuxenial-nginx)
* [nginx 的 SSL 配置优化
](https://www.linpx.com/p/ssl-configuration-optimization.html)
* [阿里云盾证书管理控制台](https://yundun.console.aliyun.com/?spm=5176.2020520163.0.0.21d12b7a53TeOq&p=cas#/cas/home)