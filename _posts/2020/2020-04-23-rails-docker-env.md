---
layout: post
title: Rails + Docker 环境搭建（dev+test+prod）
tags: [rails, docker]
---

### 0. 引言

本文尝试为 Rails + Docker 提供一个环境搭建的思路，包括本地的开发环境，CI/CD 构建流程用到的测试环境，以及生产环境，宗旨是提高 CI/CD 的效率。本文不会涉及具体的生产部署方案，比如集群架构、LB 或 CDN 等，仅限于构建 Rails 镜像本身。

应用环境：

* Docker
* Ruby on Rails 6
* Postgres
* Yarn
* webpacker

### 1. 缘起

从 CI/CD 的角度来说，我们希望保持开发、测试、生产三个环境的一致性，这也是为什么 Docker 大行其道的原因之一。但实际上，三种环境的依赖或者说系统需求其实是不同的，生产环境的依赖比开发环境少很多。

可为什么要关心它们的不同呢？既然保持环境一致性本身就是目标，在开发环境也能跑生产，就跑好了，多总比缺好。道理是没错，但，我们并不希望发布一个臃肿的生产镜像，同时也希望开发环境应有尽有。

实际上，我们真正在乎的不是镜像体积的绝对大小，Docker 采用分层缓存的机制，在 CI/CD 的流程中只要不是更新了底层的内容，镜像大小是不影响传输的。真正应该关心的，是镜像的构建是不是正好满足了需求，有没有多余的无用依赖，以及如何避免每次 build 都花费大量时间在重复安装相同的依赖上。

> 所以本文试图解决的核心问题是：如何区分构建三种不同的环境，在追求开发便利的同时做到生产镜像的精简。还有一个附带的问题是，如何让镜像的构建尽可能得快。从而双管齐下，提高 CI/CD 的效率。

### 2. 思路

刚才说，开发、测试、生产三种环境的依赖或者说系统需求其实是不同的，不同在于：

* 开发环境：需要编译各种代码和依赖，所以会用到很多基础的编译工具；希望将项目代码放在本地主机，通过 volume 的方式引入 container，便于开发；需要启用 `webpack_dev_server` 服务用来前端实时编译
* 测试环境：系统依赖和开发环境类似；数据库（redis、sidekiq、ES 等）可以共用同样的 docker-compose.yml 配置；不需要 `webpack_dev_server`；可能需要依赖一些测试用到的服务，比如 chrome；整个项目需要打包到 image 里
* 生产环境：不需要任何上层的编译工具；整个项目也需要打包到 image 里；同时不需要任何非生产文件，比如 node_modules 和测试代码，以及大量的开发/测试才需要的依赖包（数据库和 redis、sidekiq、ES 等按生产要求另外配置，不在本文讨论范围）

区分三种环境的方式就是使用三个不同的 Dockerfile，在 docker-compose.yml 中分别声明各个环境的 service：app_dev、app_test、app_prod。开发环境直接 `docker-compose up app_test` 即可，测试与生产在不同阶段分别 `docker-compose build app_test/app_prod`。

请看下文构建详情。

### 3. 开发镜像

对于开发环境，我把 gem 安装到本地，这样无需在每一次变更 Gemfile 后都重新 build 镜像，以至于每次都触发完整的 bundle，省去大量网络下载成本。而且还有个好处是，gem 保存在本地更方便查看源码。开发镜像的目的仅仅是提供一个底层操作系统环境，不包含任何应用级别的依赖。

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
      # 需要测试时可以启用
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

### 4. 测试镜像

在正式构建测试镜像之前，我首先构建了一个基础的『rails-base-builder』镜像，里面预装了基础通用的系统依赖和 Gemfile，以及 package.json。

预先做一个基础镜像的目的是避免每次 CI/CD 流程都从零开始 build，耗费大量时间在重复的步骤上，同时减少由于网络带来的不确定性（如果你能搭建自己的源服务基本就不存在这个问题了）。我的宗旨是尽量减少 CI/CD 流程的时间，所以构建基础镜像是很有效且必要的一步。

安装了基础的 Gemfile 和 package.json 后，不同的项目只要安装自己特殊的依赖包就行了。同时，对于『rails-base-builder』中安装了但项目不需要的多余依赖包，以及版本不同的问题都不用担心，在项目中 build 的时候都会更新处理，避免空间浪费和版本不一致。

不过，这种做法的『trade-off』就是，需要开发者自己定期维护基础镜像，更新其中的系统依赖和基础通用包版本，不然就失去了预装的意义（但最糟的情况也就是把所有依赖统统装一遍，和不用基础镜像一样，不是吗？）。但话说回来，一次维护就能够节省很长周期内项目 CI/CD 的时间成本和资源成本，很划算。

<!-- 此外，利用预先构建的基础镜像还有个好处，是可以添加额外的项目依赖文件，比如说字体，可以预装到基础镜像中，否则如果写在项目的 Dockerfile 中直接 build 的话，就需要将字体纳入整个项目的版本管理，就会大大增加项目本身的体积。项目内容和系统依赖最好不要混在一起。
 -->

下面看具体配置。

先做『rails-base-builder』镜像，目录结构如下：

```
Dockerfile-base-builder
Gemfile
Gemfile.lock
package.json
yarn.lock
```

* Dockerfile-base-builder

```
FROM ruby:2.6.5-alpine

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

做好镜像上传到 Docker Registry 备用。（本文中取名为：rails-base-builder:v1-20200423）

然后来看项目的配置。

* docker-compose.yml

```yml
services:

  app_test:
    build:
      context: .
      dockerfile: ./docker/Dockerfile-test
    environment:
      - RAILS_ENV=test
    depends_on:
      - db
      - chrome

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
```

* Dockerfile-test

```
FROM rails-base-builder:v1-20200423

COPY . /app

RUN bundle install -j4 --retry 3 && \
    # 删除基础镜像安装了但是项目不需要的 gem
    bundle clean --force && \
    # Remove unneeded files from installed gems (cached *.gem, *.o, *.c)
    rm -rf /usr/local/bundle/cache/*.gem && \
    find /usr/local/bundle/gems/ -name "*.c" -delete && \
    find /usr/local/bundle/gems/ -name "*.o" -delete

RUN yarn install
```

最后把以下三条命令加入 CI/CD 测试流程即可：

```bash
$ docker-compose build app_test
$ docker-compose run app_test rails db:drop db:create db:migrate
$ docker-compose run app_test rspec
```

### 5. 生产镜像

类似测试镜像，生产环境也另外构建一个『rails-prod-builder』的基础镜像，只安装满足生产所需的最小化的系统依赖。

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

做好镜像上传到 Docker Registry 备用。（本文中取名为：rails-prod-builder:v1-20200423）

然后是项目配置：

* docker-compose.yml

```yml
app_prod: 
  build:
    context: .
    dockerfile: ./docker/Dockerfile-prod
```

* Dockerfile-prod

```
FROM rails-base-builder:v1-20200423 as Builder

COPY . /app

# 只安装生产依赖
RUN bundle install -j4 --retry 3 --without development:test && \
    # 删除基础镜像安装了但是项目不需要的 gem
    bundle clean --force && \
    # Remove unneeded files from installed gems (cached *.gem, *.o, *.c)
    rm -rf /usr/local/bundle/cache/*.gem && \
    find /usr/local/bundle/gems/ -name "*.c" -delete && \
    find /usr/local/bundle/gems/ -name "*.o" -delete

RUN RAILS_ENV=production rails assets:precompile

# ======== Final prod image ========

FROM rails-prod-builder:v1-20200423

COPY . /app

COPY --from=Builder /usr/local/bundle /usr/local/bundle
COPY --from=Builder /app/public /app/public

RUN rm -rf tmp/cache vendor/bundle test spec docker
```

生产镜像的 Dockerfile 用到了 Docker 的『multi-stage』技术，目的是减小最终镜像的体积。

Builder stage 同样来自于在测试阶段创建的『rails-base-builder』镜像，在 Builder 中安装好生产所需的 gem，并且完成前端编译，然后把这些依赖文件从 Builder 拷贝到『rails-prod-builder』，最后删除生产环境无关的文件夹。

### 6. 要点回顾

* 开发、测试、生产分别创建各自的 Dockerfile，在 docker-compose.yml 中拥有各自不同的 service，分别构建镜像
* 开发镜像只提供系统环境，gem 通过 volume 安装在本地，避免由于 Gemfile 的变动重复构建开发镜像
* 为测试与生产环境分别构建通用基础镜像，安装基础通用依赖，加速 CI/CD 中镜像的构建速度
* 对于基础镜像中安装了，但项目实际上没用到的 gem，通过 `bundle clean --force` 在项目构建中清除。如果有足够的人力，甚至可以针对每个项目构建单独的基础镜像并定期维护，安装几乎严丝合缝的依赖，最大限度提高 CI/CD 的构建速度
* 生产镜像中 `bundle install` 使用 `--without development:test` 去除生产无关的 gem
* 利用 Docker 的『multi-stage』技术将生产依赖构建过程放在基础镜像中完成，只需拷贝编译结果到最终的生产镜像，最小化生产镜像的体积

---

### 参考资料：

* [How to use Docker Compose for Rails development](https://anonoz.github.io/tech/2019/03/10/rails-docker-compose-yml.html)
* [Use multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)
* [DOCKERIZE RAILS, THE LEAN WAY](https://ledermann.dev/blog/2018/04/19/dockerize-rails-the-lean-way/)
* [BUILDING DOCKER IMAGES, THE PERFORMANT WAY](https://ledermann.dev/blog/2020/01/29/building-docker-images-the-performant-way/)
* [缩减 Docker 镜像体积历程总结](https://ruby-china.org/topics/38089)