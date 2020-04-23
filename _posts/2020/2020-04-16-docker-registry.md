---
layout: post
title: 用 Docker 快速部署 Docker registry
feature-img: "assets/img/posts/2020/docker-registry/feature-img.jpg"
tags: docker
---

如题，参考 [Docker 官网 Deploy a registry server](https://docs.docker.com/registry/deploying/)

### 1. 启动官方 registry 镜像

```bash
docker run -d -p 5000:5000 --restart=always --name docker-registry -v /path/to/registry:/var/lib/registry -e REGISTRY_STORAGE_DELETE_ENABLED=true registry
```

*REGISTRY_STORAGE_DELETE_ENABLED=true 是用来运行删除 registry 上的镜像，最后会介绍用法*

可用如下命令查看所有 image，当然也可以直接进入文件夹

```bash
curl [myregistrydomain.com]:5000/v2/_catalog
```

### 2. 配置 Insecure mode

默认 docker registry 是需要配置 tls 的，如果着急使用或者测试的话可以配置非安全模式，参考 [Test an insecure registry](https://docs.docker.com/registry/insecure/)

注意需要为所有 docker client 配置：

```bash
$ sudo vi /etc/docker/daemon.json

{
  "insecure-registries" : ["myregistrydomain.com:5000"]
}
```

重启 docker engine

```bash
$ sudo systemctl restart docker
```

### 3. 测试命令

```bash
$ docker tag [image_name] [myregistrydomain.com]:5000/[image_name]
$ docker push [myregistrydomain.com]:5000/[image_name]
$ docker rmi [myregistrydomain.com]:5000/[image_name]
$ docker pull [myregistrydomain.com]:5000/[image_name]
```

### 4. 删除 registry 上的 image

如果需要删除 registry 上 image 的功能，需要加上 `REGISTRY_STORAGE_DELETE_ENABLED=true` 的环境变量（启动时已加）。

删除的具体过程比较繁琐（主要是文件夹结构复杂），所以写成一个脚本执行：

```bash
$ vi rm-remote-image.sh
#!/bin/bash

image=$1
tag=$2
imsha=`cat /path/to/registry/docker/registry/v2/repositories/$image/_manifests/tags/$tag/current/link`
curl -X DELETE [myregistrydomain.com]:5000/v2/$image/manifests/$imsha
docker exec docker-registry registry garbage-collect /etc/docker/registry/config.yml
docker exec docker-registry rm -rf /var/lib/registry/docker/registry/v2/repositories/$image
:wq

$ chmod +x rm-remote-image.sh
```

带入 image 相关参数运行删除脚本即可：

```bash
./rm-remote-image.sh [image-name] [tag]
```

*** *注意 image-name 不要加 domain***