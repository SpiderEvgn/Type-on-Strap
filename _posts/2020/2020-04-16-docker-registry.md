---
layout: post
title: 用 Docker 快速部署 Docker registry
feature-img: "assets/img/posts/2020/docker-registry/feature-img.jpg"
tags: docker
---

如题，参考 [Docker 官网 Deploy a registry server](https://docs.docker.com/registry/deploying/)

### 1. 启动官方 registry 镜像

```bash
$ vi startup.sh

#!/bin/bash

docker run -d -p 5000:5000 --restart=always --name docker-registry -v /path/to/registry:/var/lib/registry registry
```

可用如下命令查看所有 image，当然也可以直接进入文件夹（笔者把它写在一个脚本里这样可以偷懒）

```bash
$ vi list-repositories.sh
#!/bin/bash

curl myregistrydomain.com:5000/v2/_catalog
```

### 2. 操作

```bash
docker tag [image_name] myregistrydomain.com:5000/[image_name]
docker push myregistrydomain.com:5000/[image_name]
docker pull myregistrydomain.com:5000/[image_name]
```

## Insecure mode

默认 docker registry 是启用 tls 的，如果着急使用或者测试的话可以配置非安全模式，参考[Test an insecure registry](https://docs.docker.com/registry/insecure/)

### 1. 为所有 docker client 配置

```bash
$ sudo vi /etc/docker/daemon.json

{
  "insecure-registries" : ["myregistrydomain.com:5000"]
}
```

### 2. 重启 docker engine

```bash
$ sudo systemctl restart docker
```
