---
layout: post
title: 用 docker-sync 在 Mac 进行文件系统同步
date: 2019-08-03
tags: docker
---

### 引言

笔者的 Rails 开发环境一直架在 docker 上，相信所有用 Docker 的人都有过这个感受，就是速度总比直接部署到 OS 上要慢一点。前几天在大神 Rei 的点拨后去研究了一下 [docker-sync](https://github.com/EugenMayer/docker-sync)，总结一下用法和心得。

### 效果

先通过一个粗略、直观的测试来看看部署 docker-sync 后的效果。

选取两个页面，分别在使用 docker-sync 之前和之后各执行一次。

* #### 页面一

之前

![before-sync-1](/assets/img/posts/2019/docker-sync/before-sync-1.png "before sync 1")

之后：

![after-sync-1](/assets/img/posts/2019/docker-sync/after-sync-1.png "after sync 1")

* #### 页面二

之前

![before-sync-2](/assets/img/posts/2019/docker-sync/before-sync-2.png "before sync 2")

之后

![after-sync-2](/assets/img/posts/2019/docker-sync/after-sync-2.png "after sync 2")

可以看到，速度的提升效果还是很明显的，能减少大约 3-5 倍的时间。

### 用法

docker-sync 的用法其实很简单，只需要三步：安装 gem、配置 yml、启动。

* #### 安装 gem

```ruby
gem install docker-sync
```

* #### 配置 yml

在项目根目录下，创建 `docker-sync.yml`，内如如下：

```
version: '2'

syncs:
  volume-name:    # 自定义，将作为同步数据的 volume
    src: './'
```

然后更改 `docker-compose.yml`，用新的 volume 替代原来的同步方式。原本你的 volume 可能类似如下：

```
volumes:  
  - .:/app       # 将项目根目录所有文件同步到 container 中的 /app 文件夹
```

修改后如下：

```
services:
  web:
    ...
    volumes:
      - volume-name:/app:nocopy      


volumes:
  volume-name:          # docker-sync.yml 中自定义的名字
    external: true
```

* #### 启动

在项目根目录下，先启动 `docker-sync`:

```
docker-sync start
```

第一次启动会自动下载一个 eugenmayer/unison 的 docker image，以后便会直接启动一个 unison 的 container。

![unison](/assets/img/posts/2019/docker-sync/unison.jpeg "unison")

实用主义者看到这就可以了。如果你想进一步了解 docker-sync 的原理，它到底做了什么，请继续往下读。

### 原理

在 Sierra 及以下版本的 macOS 默认用的文件系统是 HFS+，High Sierra 用的是 APFS。Docker Desktop for Mac 用一个叫 [osxfs](https://docs.docker.com/docker-for-mac/osxfs/) 的共享文件系统来实现 macOS 到 Docker containers 的文件系统绑定挂载。影响一个共享文件系统性能的因素有很多方面，比如 osxfs 集成了 macOS 的 FSEvents API 到 Linux 的 inotify API 之间的映射，还有一些诸如缓存的更加复杂的维度。

简而言之，由于操作系统之间不同文件系统的差异，共享文件系统的数据同步性能有着很大的不确定性。所以，docker-sync 的主要目的就是如何最大限度地弥补这个差异，优化共享文件系统的数据同步过程。先来看下图，来自 docker-sync 官方文档：

![how-it-works](/assets/img/posts/2019/docker-sync/how-it-works.png "How It Works")

可以看到，docker-sync 主要做了以下几步：

1. 用 osxfs 挂载本地目录到 sync-container 的 /host_sync
2. 用 [Unison](http://www.cis.upenn.edu/~bcpierce/unison/) 建立一个双向同步机制，同步 /host_sync 和 /app_sync
3. 通过 Docker Volume 的方式，将 /app_sync 挂载到 app-container 的 /app

> 如果你不知道 Unison，可以看看[官网介绍](http://www.cis.upenn.edu/~bcpierce/unison/)，简单来说 Unison 就是一个文件同步工具。

经过这样一层转换（新建一个 sync-container 优化数据同步），docker-sync 完成了如下几个目的：

1. 因为采用 Unison，所以 /host_sync 到 /app_sync 的同步是原生速度（native-speed）的，没有了原本的因为不同文件系统带来的性能影响
2. 通过 Docker Volume 来挂载 /app_sync 到 app-container，这一步是 hyperkit 层的，用的是 Docker LINUX-based 的原生挂载方式，所以也大大减小了对性能的影响（也是原生速度的）。

其实，最关键的一步就是在 sync-container 中的 /host_sync 到 /app_sync 的同步过程，通过 Unison，osxfs 对性能的影响被从文件系统的读/写中剥离了，Unison 用远比 osxfs 快得多的技术完成了 macOS 到 Docker container 的数据同步。至于 Unison 究竟是如果做到高效跨平台文件同步的，就留给更专业的大佬解读了。

### 结语

如果你用 Mac，如果你又依赖 Docker 管理环境依赖，那 docker-sync 真是一个必不可少的绝佳工具。尽管需要在主机上安装一个 gem（当然还有其它几个依赖 gem），但也只需要安装一个 gem，没有任何其它系统层的依赖，也不需要跑任何进程，通过一个 Unison 的 Docker container，docker-sync 就完美地帮你实现了 5-10 倍的性能提升。

---

## 参考资料：

* [docker-sync github](https://github.com/EugenMayer/docker-sync)

* [docker-sync doc](https://docker-sync.readthedocs.io/en/latest/index.html)

* [infamously horrible performance](https://docs.docker.com/docker-for-mac/osxfs/#performance-issues-solutions-and-roadmap)

* [Unison 官网](http://www.cis.upenn.edu/~bcpierce/unison/)

* [Unison github](https://github.com/bcpierce00/unison)