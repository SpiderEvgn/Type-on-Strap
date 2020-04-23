---
layout: post
title: Rails + Docker + Webpacker 环境搭建（dev+test+prod）
tags: [rails, docker]
---

### 0. 引言

本文尝试为 Rails 搭配 Docker 提供一个环境搭建的思路，包括本地的开发环境，CI/CD 构建流程用到的测试环境，以及生产环境。本文不会涉及具体的生产部署环境，比如集群架构、LB 或 CDN 等，仅限于构建 Rails 镜像本身。

### 1. 理念

从 CI/CD 的角度来说，我们希望保持开发、测试、生产三个环境的一致性，这也是为什么 Docker 大行其道的原因之一。但实际上，三种环境的依赖或者说系统需求其实是不同的。

* 对于开发环境，需要编译各种代码和依赖，所以会用到很多基础的编译工具，同时也希望将项目代码放在本地主机，通过 volume 的方式引入 container，便于开发。此外，还会起一个 `webpack_dev_server` 服务用来开发时的前端实时编译。
* 对于测试环境（特指 CI/CD 中用来跑 rspec 的测试镜像），系统依赖和开发环境类似，数据库可以共用同样的 docker-compose.yml  配置，不需要 `webpack_dev_server`，同时需要依赖一些测试用到的服务，比如 chrome。此外，整个项目需要打包到 image 里。
* 对于生产环境，不需要任何上层的编译工具，整个项目也需要打包到 image 里，同时不需要任何非生产文件，比如 node_modules 和测试代码，以及大量的开发/测试才需要的依赖包。数据库，以及一些别的服务（redis、sidekiq、es 等）都可以按生产要求另外配置。

区分三种环境的方式就是使用三个不同的 Dockerfile，在 docker-compose.yml 中分别声明各个环境的 service：app_dev、app_test、app_prod，然后用 `docker-compose run` 运行不同环境。

下面就来看看三个环境构建的详情。

### 2. 开发镜像

对于开发环境，我把 gem 安装到本地，这样无需在每一次变更 Gemfile 后都重新 build 镜像，以至于每次都触发完整的 bundle，省去大量网络下载成本。而且还有个好处是，gem 保存在本地更方便查看源码。直接来看配置。

* docker-compose.yml

```yml
services:

  app: &app_base
    build:
      context: .
      dockerfile: ./docker/Dockerfile-dev
    volumes:
      - .:/app:cached
      - ./.gems:/usr/local/bundle

  app_dev:
    <<: *app_base
    command: bash -c "rm -f /app/tmp/pids/server.pid && rails s -p 3000 -b 0.0.0.0"
    environment:
      - WEBPACKER_DEV_SERVER_HOST=webpack_dev_server
    ports:
      - 3000:3000
    depends_on:
      - webpack_dev_server
      - db
      # 开发测试代码时可以启用
      # - chrome

    stdin_open: true
    tty: true

  webpack_dev_server:
    <<: *app_base
    command: bin/webpack-dev-server
    environment:
      - WEBPACKER_DEV_SERVER_HOST=0.0.0.0
    ports:
      - 3035:3035

  db:
    image: postgres:12-alpine
    volumes:
      - type: volume
        source: dbdata
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
    environment:
      - POSTGRES_HOST_AUTH_METHOD=trust
    ports:
      - 5432:5432

  chrome:
    image: selenium/standalone-chrome-debug
    ports:
      - 5900:5900
    volumes:
      - /dev/shm:/dev/shm

volumes:
  dbdata:

```

* Dockerfile-dev

```yml
FROM ruby:2.6.5-alpine

# 根据需要换源
RUN echo https://mirror.tuna.tsinghua.edu.cn/alpine/v3.11/main > /etc/apk/repositories
RUN echo https://mirror.tuna.tsinghua.edu.cn/alpine/v3.11/community >> /etc/apk/repositories

# Add basic packages
RUN apk add --update --no-cache \
    build-base \
    postgresql-dev \
    imagemagick \
    git \
    nodejs-current \
    npm \
    yarn \
    tzdata \
    file \
    bash \
    # 某些编译会用到 python
    python \
    && rm -rf /var/cache/apk/*

WORKDIR /app 
```

然后手动跑如下命令配置开发环境即可：

```bash
$ docker-compose run --rm app_dev gem install bundler
$ docker-compose run --rm app_dev bundle install
$ docker-compose run --rm app_dev yarn install
$ docker-compose up app_dev
```

### 3. 测试镜像

在正式测试镜像之前，我首先构建了一个基础的『rails-base-build』镜像，里面预装了基础通用的系统依赖和 Gemfile，以及 package.json。

预先做一个基础镜像的目的是避免每次 CI/CD 流程都从头 build 一次，耗费大量时间在重复安装基础依赖上。

安装了基础 Gemfile 和 package.json 的好处是，对于这些通用的项目依赖不必每次都从头完整 build 一遍，不同的项目只要安装自己特殊的包就行了。同时，对于『rails-base-build』中安装了但项目不需要的多余依赖包，以及版本不同的问题都不用担心，在项目中 build 的时候都会更新处理，避免空间浪费和版本不一致。最后的结果一定是与项目依赖一致（不多装也不少装），又做到 image 体积最小。

不过，这种做法的『trade-off』就是，需要开发者自己定期维护 base-test 镜像，更新其中的系统依赖和基础通用包版本，不然就失去了预装的意义（但最糟的情况也就是把所有依赖统统装一遍，和不用 base-test 一样，不是吗？）。一次性的工作能够节省很长周期内项目构建的时间成本，包括资源成本。

<!-- 此外，利用预先构建的基础镜像还有个好处，是可以添加额外的项目依赖文件，比如说字体，可以预装到基础镜像中，否则如果写在项目的 Dockerfile 中直接 build 的话，就需要将字体纳入整个项目的版本管理，就会大大增加项目本身的体积。项目内容和系统依赖最好不要混在一起。
 -->
**以上的思路参考了[https://ledermann.dev/blog/2020/01/29/building-docker-images-the-performant-way/](https://ledermann.dev/blog/2020/01/29/building-docker-images-the-performant-way/)**

下面上具体配置。

先做 base-builder 镜像，目录结构如下：

```
Dockerfile
Gemfile
Gemfile.lock
package.json
yarn.lock
```

* Dockerfile-base-builder

```
FROM ruby:2.6.5-alpine
LABEL maintainer="xiajun.zhang@usingnow.com"

RUN echo https://mirror.tuna.tsinghua.edu.cn/alpine/v3.11/main > /etc/apk/repositories
RUN echo https://mirror.tuna.tsinghua.edu.cn/alpine/v3.11/community >> /etc/apk/repositories

# Add basic packages
RUN apk add --update --no-cache \
    build-base \
    postgresql-dev \
    imagemagick \
    git \
    nodejs-current \
    npm \
    yarn \
    tzdata \
    file \
    bash \
    # 某些编译会用到 python
    python \
    && rm -rf /var/cache/apk/*

WORKDIR /app

COPY Gemfile* /app/
RUN bundle config mirror.https://rubygems.org/ https://gems.ruby-china.com/
RUN bundle install -j4 --retry 3 && \
    rm -rf /usr/local/bundle/cache/*.gem && \
    find /usr/local/bundle/gems/ -name "*.c" -delete && \
    find /usr/local/bundle/gems/ -name "*.o" -delete

COPY package.json yarn.lock /app/
RUN yarn config set registry http://registry.npm.taobao.org/
RUN yarn install
```

做好镜像打个 tag 然后上传到 Docker Registry 备用。（本文中就取名为：rails-base-builder:v1-20200421）

然后来看项目的配置。

* docker-compose.yml

```
app_test:
  build:
    context: .
    dockerfile: ./docker/Dockerfile-test
  environment:
    - RAILS_ENV=test
  depends_on:
    - db
    - [other services: redis...]
```

* Dockerfile-test

```
FROM rails-base-builder:v1-20200421

COPY . /app

RUN bundle install -j4 --retry 3 && \
    # Remove unneeded gems
    bundle clean --force && \
    # Remove unneeded files from installed gems (cached *.gem, *.o, *.c)
    rm -rf /usr/local/bundle/cache/*.gem && \
    find /usr/local/bundle/gems/ -name "*.c" -delete && \
    find /usr/local/bundle/gems/ -name "*.o" -delete

RUN yarn install
```

最后把以下三条命令加入 CI/CD 测试流程即可：

```
$ docker-compose build app_test
$ docker-compose run app_test rails db:drop db:create db:migrate
$ docker-compose run app_test rails rspec
```

### 4. 生产镜像

生产环境也另外构建一个『rails-prod-builder』的基础镜像，只安装满足生产所需的最小化的系统依赖，目的是避免每次构建部署的过程中重复花费时间去下载系统包，减少由于网络带来的不确定（如果你能搭建自己的源服务基本就不存在这个问题了）。

来看一下这个生产的基础镜像：

* rails-prod-builder

```
FROM ruby:2.6.5-alpine

RUN echo https://mirror.tuna.tsinghua.edu.cn/alpine/v3.11/main > /etc/apk/repositories
RUN echo https://mirror.tuna.tsinghua.edu.cn/alpine/v3.11/community >> /etc/apk/repositories

RUN apk add --update --no-cache \
    postgresql-client \
    imagemagick \
    tzdata \
    file \
    && rm -rf /var/cache/apk/*

WORKDIR /app
```

引言部分已说过，本文不涉及部署架构，只谈生产镜像打包，所以 docker-compose.yml 文件足够简单，不包含任何服务依赖。

* docker-compose.yml

```
app_prod: 
  build:
    context: .
    dockerfile: ./docker/Dockerfile-prod
  environment:
    - RAILS_ENV=production
```

生产镜像的 Dockerfile 用到了 Docker 的 stage，目的是减少最终镜像的体积。

Builder stage 同样来自于在测试阶段创建的『rails-base-build』镜像，在 Builder 中安装好生产所需的 gem 以及前端编译，然后把 Builder 中安装好的项目依赖拷贝到『rails-prod-builder』，最后删除生产环境无关的文件夹，比如 test、spec 等。

* Dockerfile-prod

```
FROM rails-base-builder:v1-20200421 as Builder

COPY . /app

RUN bundle install -j4 --retry 3 --without development:test && \
    # Remove unneeded gems
    bundle clean --force && \
    # Remove unneeded files from installed gems (cached *.gem, *.o, *.c)
    rm -rf /usr/local/bundle/cache/*.gem && \
    find /usr/local/bundle/gems/ -name "*.c" -delete && \
    find /usr/local/bundle/gems/ -name "*.o" -delete

RUN RAILS_ENV=production rails assets:precompile

# ======== real prod image ========

FROM rails-base-build:v1-20200421

COPY . /app

COPY --from=Builder /usr/local/bundle /usr/local/bundle
COPY --from=Builder /app/public /app/public

RUN rm -rf tmp/cache vendor/bundle test spec docker
```

---

### 参考资料：

* [https://anonoz.github.io/tech/2019/03/10/rails-docker-compose-yml.html](https://anonoz.github.io/tech/2019/03/10/rails-docker-compose-yml.html)