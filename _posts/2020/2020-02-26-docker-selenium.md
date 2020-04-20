---
layout: post
title: Rails + RSpec + Docker + Capybara + Selenium + Chrome + VNC
date: 2020-02-26
tags: rails
---

### 0. 引言

在 上一篇 《[Rails 在 Docker container 测试找不到浏览器]({% post_url 2020/2020-02-14-rails-test-in-docker-no-browser %})》中讲了如何避免 Rails 默认的 `:rack_test` 驱动依赖浏览器，但是毕竟有需要浏览器模拟的前端测试，在 docker 中是如何实现的呢？

### 1. 下载 selenium chrome 镜像

[Selenium 官方](https://github.com/SeleniumHQ/docker-selenium#docker-images-for-selenium-standalone-server-hub-and-node-configurations-with-chrome-and-firefox)提供了 chrome 和 firefox 的镜像，我们使用 standalone 的方式，下载的是 `selenium/standalone-chrome-debug`，因为 `-debug` 镜像是带有 VNC 服务的，可以提供可视化测试过程。

直接来看 `docker-compose.yml` 文件：

```yml
version: '3.7'
services:
  
  [... other services]

  web:
    [... other configs]

    depends_on:
      - chrome


  chrome:
    image: selenium/standalone-chrome-debug
    ports:
      - 5900:5900
    volumes:
      - /dev/shm:/dev/shm
```

*注：5900 是 VNC 服务器的端口，我们需要在 host 上访问所以要讲端口 export 出来，另外 4444 是 selenium 的端口，直接在 capybara 中配置就行了*

### 2. 配置 RSpec

之前讲过用 `:rack_test` 驱动是不用依赖浏览器的，包括 Capybara 支持的一些 `click_on` 和 `fill_in` 等交互操作，但是 `:rack_test` 是不支持 JavaScript 测试的，所以为了测试 JS，以及看到自动化的浏览器模拟测试过程，需要在 RSpec 测试用例中指定是 JS 测试，然后配置 JS 测试相应的驱动以及服务器。

启用 JS 测试很简单，如下加入 `js: true` 即可：

```rb
RSpec.feature "test", type: :feature do     # 也可以加在外层

  scenario "test", js: true do
```

然后就是最关键的一步配置，可以直接写到 `rails_helper.rb`，不过我更推荐去掉 `rails_helper.rb` 中 `Dir[Rails.root.join('spec', 'support', '**', '*.rb')].each { |f| require f }` 的注释，然后在 `spec/support/` 中新建配置文件：

```rb
Capybara.javascript_driver = :selenium_remote_chrome          # 自定义 driver
Capybara.register_driver :selenium_remote_chrome do |app|
  Capybara::Selenium::Driver.new(
    app,
    browser: :remote,
    url: "http://chrome:4444/wd/hub",                         # remote chrome 就来自于 docker-compose.yml 中的 selenium/standalone-chrome-debug
    desired_capabilities: :chrome)
end

RSpec.configure do |config|
  config.before(:each, type: :feature) do
    # 获取 server ip，需要根据 OS 调整命令
    ip = `/sbin/ip route|awk '/scope/ { print $9 }'`
    ip = ip.gsub "\n", ""
    Capybara.server_port = "3000"
    Capybara.server_host = ip
    Capybara.app_host = "http://#{Capybara.current_session.server.host}:#{Capybara.current_session.server.port}"
  end

  config.after(:each, type: :feature) do
    Capybara.reset_sessions!
    Capybara.use_default_driver
    Capybara.app_host = nil
  end
end
```

这里有一个难点，就是在 `web` 中可以很容易的利用 docker 提供的方法通过 service name 访问到 `http://chrome:4444/wd/hub`，但是在 `chrome` 中却不知道服务器的地址，于是我们直接用 shell 命令获取 ip 传给 Capybara，`3000` 端口是自己指定的，只要不与开发端口冲突就行。

至此，Capybara 的配置就完成了，在运行测试的时候可以在 console 观察到如下输出：

```
Capybara starting Puma...
* Version 4.3.1 , codename: Mysterious Traveller
* Min threads: 0, max threads: 4
* Listening on tcp://[web-ip]:3000               # web-ip 就是通过 ip 命令获取的地址
```

可见浏览器的依赖问题已经解决，并且测试也已经通过。

### 3. 通过 VNC 观察自动化测试过程

因为浏览器是跑在容器里的，所以需要安装 VNC 客户端连进去看，`Docker for Mac` 默认有安装 VNC 的客户端，在此我就直接介绍这种方式了。

在『Terminal』中直接输入：`open vnc://localhost:5900`，在弹出的密码框输入 `secret`，就会看到 Selenium 官方提供的准备界面了。

接下来只要跑 `rspec` 命令即可看到 VNC 窗口中自动弹出 chrome 浏览器并且自动完成测试内容。

---

## 参考资料：

* [Capybara Github](https://github.com/teamcapybara/capybara)
* [SeleniumHQ/docker-selenium](https://github.com/SeleniumHQ/docker-selenium)
* [dockerized-rails-capybara-tests-on-top-of-selenium](https://www.alfredo.motta.name/dockerized-rails-capybara-tests-on-top-of-selenium/)


