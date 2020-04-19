---
layout: post
title: 新建带 webpacker 的 Rails 项目
date: 2018-10-08
tags: rails
---

- 2019.11.17 更新 -

### 1. 新建 Gemfile:

```
source 'https://rubygems.org'

gem 'rails'
```

### 2. 新建 Dockerfile (新架构 gem 安装到本地，这一步不用了)

```
FROM registry.cn-shanghai.aliyuncs.com/aaron_dev/ruby2.6-nodejs-yarn:v1

# 因为默认安装好 rails 后 Gemfile 的 source 总是会设置成默认源，如果在国内会非常慢以至于卡在 `bunlde install`，所以用这条命令覆盖 Gemfile 的 source
RUN bundle config mirror.https://rubygems.org https://gems.ruby-china.com

COPY Gemfile* /app/

RUN bundle install
```

### 3. 新建空的 Gemfile.lock

### 4. 新建 docker-compose.yml

```
version: '3.7'
services:
  web:    
    image: registry.cn-shanghai.aliyuncs.com/aaron_dev/rails-base-image:v1
    command: bash -c "rm -f /app/tmp/pids/server.pid && rails s -p 5000 -b 0.0.0.0"
    volumes:
      # old string format
      - .:/app
      - ./.gems:/usr/local/bundle
    ports:
      - [host-port]:5000
    environment:
      - WEBPACKER_DEV_SERVER_HOST=webpack_dev_server
    depends_on:
      - db

    # byebug, use attach
    stdin_open: true
    tty: true

  webpack_dev_server:
    image: registry.cn-shanghai.aliyuncs.com/aaron_dev/rails-base-image:v1
    command: bin/webpack-dev-server
    ports:
      - [host-port]:3035
    volumes:
      - .:/app
      - ./.gems:/usr/local/bundle
    environment:
      - WEBPACKER_DEV_SERVER_HOST=0.0.0.0

  db:
    image: postgres
    volumes:
      # new format
      - type: volume
        source: dbdata
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
    ports:
      - [host-port]:5432

volumes:
  dbdata:

```

### 5. 在项目目录下运行：

```
docker-compose run --rm web gem install bundler
docker-compose run --rm web bundle install
docker-compose run --rm web rails new . --force --database=postgresql [--skip-sprockets --skip-turbolinks] --webpack
```

* `--skip-sprockets --skip-turbolinks` 参数可选，webpack 允许将 css 依旧交给 sprockets 管理；turbolinks 除非特殊需求，否则不建议关闭

注：这里补充一个知识点，在服务器上安装 docker 后，因为一般我们不会用超级用户，所以是没有权限直接运行 docker 命令的，如果不想所有命令前都加上 sudo 的话，就要把你的用户加到 docker 组里，运行如下命令然后重新登录即可：

```
sudo usermod -aG docker [server_username]
```

然后检查 /etc/group 的 docker 组知否已经加入了用户： docker:x:999:[username]


### 6. 封装 image (新架构 gem 安装到本地，这一步不用了)

安装完 rails 后会得到一个 image：`[project-name]_web`，可以通过 `docker images` 查看。
这个 image 就是我们最终的应用镜像了，然后根据具体情况把它上传到镜像仓库。

```
docker tag [project-name]_web [image-repository]/[image-name]:[tag]
docker push [image-repository]/[image-name]:[tag]
```

接下来修改 `docker-compose.yml`，替换 web service 下的 `build: .`：

```
image: [image-repository]/[image-name]:[tag]
```

### 7. 修改 database.yml 如下：

```
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password:
```

### 8. 项目初始化完成后，修改 views/layouts/application.html.erb

```
<!DOCTYPE html>
<html>
  <head>
    <title>App</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= stylesheet_pack_tag 'application' %>
  </head>

  <body>
    <%= yield %>

    <!-- 加载完页面再 load js -->
    <%= javascript_pack_tag 'application' %>
  </body>
</html>
```

### 9. 启动项目

```
docker-compose up
docker-compose exec web rails db:create
docker-compose exec web rails db:migrate
```

至此，我们就快速建立了一个基于 webpack 的 Rails 项目，然后加入一些基础的前端依赖，并进一步完善 webpack 的基础配置。

进入 web 容器 `docker-compose exec -it web bash`，运行：
```
yarn add bootstrap
yarn add popper.js
yarn add jquery
yarn add @fortawesome/fontawesome-free
```

然后修改 config/webpack/environment.js 如下：

```
const { environment } = require('@rails/webpacker')

const webpack = require('webpack')
/**
 * Automatically load modules instead of having to import or require them everywhere.
 * Support by webpack. To get more information:
 *
 * https://webpack.js.org/plugins/provide-plugin/
 * http://j.mp/2JzG1Dm
 * 
 * 配置文件，更改需要重启服务
 */
environment.plugins.prepend(
  'Provide',
  new webpack.ProvidePlugin({
    echarts: 'echarts',
    $: 'jquery',
    jQuery: 'jquery',
    jquery: 'jquery',
    'window.jQuery': 'jquery',
    Popper: ['popper.js', 'default']
  })
)
module.exports = environment
```

这个文件可以参考这两篇文章：[https://joey.io/how-to-use-jquery-in-rails-5-2-using-webpack/](https://joey.io/how-to-use-jquery-in-rails-5-2-using-webpack/), [https://webpack.js.org/plugins/provide-plugin/](https://webpack.js.org/plugins/provide-plugin/)

再引入这些第三方组件 `app/javascript/packs/application.js`：

```
// 第三方库加载
import "bootstrap"
import "bootstrap/dist/css/bootstrap.min.css"
import "@fortawesome/fontawesome-free/css/all.min.css"
```

### 10. 允许 webpack 打包 css 文件

webpacker 4 之后，默认只会打包 js，若要启用 css 打包，需修改 config/webpacker.yml:

```
# Extract and emit a css file
  extract_css: true
```

**注：因为 webpack 将所有资源都打包成 JS 的方式伺服前端，所以如果不加这条，则 css 也会随着 JS 一起加载，结果就是 html 页面显示的时候 css 还未被加载，等到 JS 加载的时候 css 才渲染，所以每次在页面刷新的时候样式会从无到有地闪动一下**

### 11. 启用 rails-ujs（可选）

在使用 webpacker 后，如果你没有同时启用 sprockets，那 rails 原生提供的 js 方法都会失效，需要手动添加。

比如，原来你可以利用 `data-confirm="Are you sure..."` 的辅助方法打开一个 js 的 alert 提示框，但是改用 webpacker 后就没有这个功能了，原因是这个方法是 rails-ujs 提供的，原本在自带的 sprockets 中，webpacker 默认是没有的，需要手动安装，方法如下：

先在前端安装 rails-ujs 依赖：

```
yarn add rails-ujs
```

然后，在 webpacker 中启用：

```
import Rails from "rails-ujs"
Rails.start()
```

### 12. 新增 Gem 包 （新架构 gem 安装到本地，这一步不用了）

最后讲一讲如果要更新 image，比如安装了新的 gem 包，该如何处理。

首先，当然是在 Gemfile 里加上 `gem '[package-name]'`，然后我们有两种方法可以更新到 image。

  * 直接 commit 当前读写层

    ```
    docker-compose exec web bundle install
    docker commit [container] [image-repository]/[image-name]:[new-tag]
    ```
    这样，一个新版本的应用镜像就直接创建好了。这个方法的好处是非常直观，而且很快也很方便，只需要安装新的 gem 即可；但缺点是，它直接将当前 container 的读写层所有内容都 commit 到了新镜像，这会包含一定的风险，因为也许你会不记得之前是否对该 container 做了任何临时的修改，而你并不想将这些修改持久化到 image。（所以 `docker commit` 向来要慎用）

  * 基于原 image 再 build

    ```
    docker-compose exec web bundle install
    docker build . -t [image-repository]/[image-name]:[new-tag]
    ```
    这个方法的缺点在于，要运行两次 `bundle install`，一次给当前环境，一次在创建新的镜像，而且 build 会把所有的 gem 重新再安装一遍，所以非常慢。缺点很明显，但是优势是镜像内容可控，规避了之前的风险。其实这也是标准做法。

最后，将新的 image 更新到 `docker-compose.yml`：

```
image: [image-repository]/[image-name]:[new-tag]
```

