---
layout: post
title: Jenkins 自动化构建 Docker + Rails 的生产环境
tags: [rails, docker, jenkins]
---

### 0. 引言

之前就写过一篇很基础的 [『用 Docker 部署 Rails 的生产环境』]({% post_url 2020/2020-01-08-rails-docker-deployment %})，从服务器架构的角度来说，这两篇文章都仅限于测试与实验性质，真正的生产环境不可能如此简单，但这些内容更接近于架构领域，而笔者的目标是 Rails 的生产部署本身。

上一篇是简单的手动部署，开发与生产的系统环境也没有任何区别。而这一篇的重点是利用 Jenkins 实现 CI/CD 的流程，包括自动化测试与生产镜像打包发布的过程。同时，本文可以作为 [『Rails + Docker 环境搭建（dev+test+prod）』]({% post_url 2020/2020-04-23-rails-docker-env %}) 的后续，所以要看懂本文的 CI/CD 过程需要先理解『Rails + Docker 环境搭建（dev+test+prod）』中环境构建的思路。

### 1. CI/CD 流程

其实 CI/CD 流程是基于开发工作流的，不同的合作模式会产生不同的自动化流程。不过，笔者并不打算花时间来填这个坑，开发模式的问题还是各位大佬见仁见智吧。

本文的 CI/CD 流程是基于 GitHub 的，大致思路是，GitHub Pull Request 用来触发自动化测试流程，最后的 merge 留给人为操作；另一方面，Jenkins 监听 GitHub 的 Push 操作，如果 push 的分支是发布分支，则开始构建自动化发布流程。所以要创建两个 Jenkins Job，一个监听 pr，一个监听 push。

来看一下整个 CI/CD 的流程图：

![cicd-workflow](/assets/img/posts/2020/jenkins-cicd/cicd-workflow.jpg "CI/CD workflow")

> 关于 Jenkins 集成 GitHub 笔者已经写过两篇文章：
> * [『Jenkins 集成 GitHub（上）—— Pull Request』]({% post_url 2020/2020-04-17-jenkins-github-pr %}) 
> * [『Jenkins 集成 GitHub（下）—— Push』]({% post_url 2020/2020-04-18-jenkins-github-push %}) 

### 2. Pull Request 的构建（自动化测试）

核心的构建过程一共三步：构建测试镜像，准备测试数据库，运行测试。

在 Jenkins 中创建 `Build / Execute shell`，代码如下：

```bash
#!/bin/bash +x
set -e

cd $WORKSPACE
cp config/database.yml.example config/database.yml
echo -e "\033[34mStart building test image\033[0m"
docker-compose build app_test

COMMAND="rails db:drop db:create db:migrate"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

COMMAND="rspec"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND
```

***注：Docker 环境参照[『Rails + Docker 环境搭建（dev+test+prod）』]({% post_url 2020/2020-04-23-rails-docker-env %})***

最后别忘了清理 Docker，创建 `Post-build Actions / Post build task`（需要安装 [Post build task](https://plugins.jenkins.io/postbuild-task/) 插件），代码如下：

```bash
echo -e "\033[34mStart docker cleaning job\033[0m"
docker-compose down
docker system prune --force
```

### 3. Push 的构建（自动化测试 + 生产发布）

构建过程也分为三大块：自动化测试，打包生产镜像，部署。

* 自动化测试与 pr 中的相同，不赘述。

* 打包生产镜像的过程，要求预先把项目的 master.key 存入 Jenkins 服务器，在打包镜像前将 master.key 拷贝到项目目录的 config 下，这样 master.key 就被一起打包到生产镜像了。

* 最后部署过程，数据库要单独起，在真实的生产环境也是如此，数据库一定是单独维护的（当然如果你已经有独立数据库就可以忽略这一步，在环境参数配置即可）。所以只要在第一次发布前，预先在生产服务器跑如下命令即可：

```bash
docker run -d --restart=always -v dbdata:/var/lib/postgresql/data --name [db-name] postgres
```

然后在发布应用的生产镜像时连接到这个数据库容器就可以了，这样数据库就持久化在应用服务器上了。此外，对于文件存储，如果用的也是本地 storage，还需要把项目的 storage 文件夹挂载到服务器上。

最后，为了让 nginx 能够访问 public，需要在生产容器启动后把 public 文件夹拷贝到服务器上。(过 nginx 也采用容器化的话，可以将生产镜像单独起一个 data_volume，然后 app_prod 和 nginx 都访问这个 volume)

```bash
#!/bin/bash +x
set -e

# ---- test ----
cd $WORKSPACE
cp config/database.yml.example config/database.yml
echo -e "\033[34mStart building test image\033[0m"
docker-compose build app_test

COMMAND="rails db:drop db:create db:migrate"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

COMMAND="rspec"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

# ---- build prod image ----
echo -e "\033[34mStart building prod image\033[0m"
cp /path/to/master.key config/
docker-compose build app_prod

echo -e "\033[34mStart pushing prod image to Docker Registry\033[0m"
origin_br=$GIT_BRANCH
# 根据实际情况构建自己的版本号
release=${origin_br: 7}
echo $release
docker tag [jenkins-job-name]_app_prod [myDockerRegistry]/[prod-image-name]:$release
docker push [myDockerRegistry]/[prod-image-name]:$release

# ---- deploy prod image ----
echo -e "\033[34mStart deploying prod image to app server\033[0m"
ssh -A [app-server] -tt << remotessh
docker pull [myDockerRegistry]/[prod-image-name]:$release

# remove old prod image
docker stop [prod-image-name]
docker rm [prod-image-name]

# start new prod image
docker run -d --env-file /var/www/[project]/env.production -v /var/www/[project]/storage:/app/storage -p 3000:3000 --link [db-name]:db --name [prod-image-name] [myDockerRegistry]/[prod-image-name]:$release puma -C config/puma.rb

# copy public assets to where nginx can serve
rm -rf /var/www/[project]/public
docker cp [prod-image-name]:/app/public /var/www/[project]/public

exit
remotessh
```

生产环境参数：env.production

```
RAILS_ENV=production
DOMAIN_NAME=
DATABASE_HOSt=
DATABASE_USERNAME=
DATABASE_PASSWORD=
ELASTICSEARCH_HOST=
REDIS_SIDEKIQ_URL=
REDIS_CABLE_URL=
REDIS_CACHE_URL=
...
```

同样别忘了配置 `Post-build Actions / Post build task`：

```bash
echo -e "\033[34mStart docker cleaning job\033[0m"
docker-compose down
docker system prune --force
```

---

### 参考资料：

* [unbuffer(1) - Linux man page](https://linux.die.net/man/1/unbuffer)

