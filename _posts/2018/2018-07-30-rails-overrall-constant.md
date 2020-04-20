---
layout: post
title: Rails 内自定义全局常量
date: 2018-07-30
tags: rails
---

### Rails 自定义配置

Rails 允许你在 config/application 中自定义配置自己的变量，比如 `config.hello = 'world'`

而如果你要使用多层嵌套的变量，则要使用 `config.x`，下面具体举个例子。

### 用 config.x 定义全局嵌套 Hash 常量

因为需要用到的变量特别多，所以单独在 config/initializes 目录下新建一个初始化文件，然后配置:

```
CONSTANT = Rails.configuration.x.custom_configurations

CONSTANT.params = ActiveSupport::HashWithIndifferentAccess.new
CONSTANT.params[:test] = ENV["LOCAL_PARAMETER"] || 'test'
```

第一行 将自定义的变量 custom_configurations 赋予常量 CONSTANT。

第二行 使用 ActiveSupport 特有的 HashWithIndifferentAccess 类初始化 CONSTANT 的一个嵌套变量，这样就能将这个变量作为 Hash 赋值了。

第三行 用 symbol 的方式给 params 赋值，没有设定环境变量则取值 'test'。

这个用法非常方便，支持你在整个 Rails 项目中使用 CONSTANT 常量，比如你可以在 controller 里面做判断 `if test == CONSTANT.params[:test]`

对 Hash 你可以继续嵌套下去，像这样：

```
CONSTANT.params[:nested] = ActiveSupport::HashWithIndifferentAccess.new
CONSTANT.params[:nested][:first] = 'first'
```

---

## 参考资料：

* [Rails API 官网介绍](https://api.rubyonrails.org/v5.1/classes/ActiveSupport/HashWithIndifferentAccess.html)

* [Rails Guides - Custom configuration](https://guides.rubyonrails.org/v4.2/configuring.html#custom-configuration)