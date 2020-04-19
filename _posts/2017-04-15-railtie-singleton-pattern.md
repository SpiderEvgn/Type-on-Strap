---
layout: post
title: Rails::Application.config 和 railtie 的单例模式
date: 2017-04-15
tags: rails
---

在 rails 中，app 的配置一般是在 config/application.rb 文件，通过类似如下的方法：

```ruby
module MyApp
  class Application < Rails::Application

    config.time_zone = 'Beijing'

end
```
我一直很好奇这个简简单单的 config 到底是如何完成对整个 rails app 的全局配置的，它的背后到底隐藏着多么奥妙的玄机？于是，我翻了一下 rails 的源码，在此分享一下所得。

首先介绍一下相关的 rails 类结构：

```ruby
MyApp::Application  < Rails::Application
Rails::Application  < Rails::Engine
Rails::Engine       < Rails::Railtie

Rails::Application::Configuration < Rails::Engine::Configuration
Rails::Engine::Configuration      < Rails::Railtie::Configuration
```

> 其实主要记住以下关系就行了：
Application < Engine < Railtie （ Configuration 同样遵循这个关系 ）

有了这个，我们就知道去哪里找这个 *config* 了，因为在 *class Application* 作用域中的 *config* 其实就是它的实例方法。这里我截取了几段 rails 的源码：(**注意：**以下源码只是截取部分，并不包含完整的内容！)

### rails/application.rb
```ruby
module Rails
  class Application < Engine

    def config           # 找到了定义 Rails::Application 的 config 实例方法的地方
      @config ||= Application::Configuration.new(self.class.find_root(self.class.called_from))
    end

  end
end
```
### rails/railtie.rb
```
module Rails
  class Railtie
    autoload :Configuration, 'rails/railtie/configuration'

    class << self
      private :new                      # 这里通过将 new 方法私有化，使外部直接新建 railtie 实例成为不可能
      delegate :config, to: :instance   # 将类方法委托给 instance 的 config，也就是在下面定义的实例方法

      def instance
        @instance ||= new               # 通过 instance 累方法和 ||= 操作符调用了私有的 new 方法，巧妙的做到了
      end                               # railtie的单例模式
    end

    def config                          # config 方法获取了 Railtie::Configuration 的实例
      @config ||= Railtie::Configuration.new
    end

    end
  end
end
```
### rails/application/configuration.rb
```
module Rails
  class Application
    class Configuration < ::Rails::Engine::Configuration
      attr_accessor :cache_store, :eager_load, :force_ssl, :log_formatter, :secret_key_base, 
                    :secret_token, :ssl_options, :session_options, :time_zone, :beginning_of_week, :filter_redirect

      attr_writer :log_level
      attr_reader :encoding

      def initialize(*)
        super
        self.encoding        = "utf-8"
        @force_ssl           = false
        @ssl_options         = {}
        @session_store       = :cookie_store
        @session_options     = {}
        @time_zone           = "UTC"
        @beginning_of_week   = :monday
        @log_level           = nil
        @generators          = app_generators
        @cache_store         = [ :file_store, "#{root}/tmp/cache/" ]
        @log_formatter       = ActiveSupport::Logger::SimpleFormatter.new
        @eager_load          = nil
        @secret_token        = nil
        @secret_key_base     = nil
      end
  
    end
  end
end
```
### rails/railtie/configuration.rb
```
require 'rails/configuration'

module Rails
  class Railtie
    class Configuration

      def app_generators     # 注意：在 Rails::Railtie::Configuration 中所有的变量都是 @@ 开头的类变量
        @@app_generators ||= Rails::Configuration::Generators.new
        yield(@@app_generators) if block_given?
        @@app_generators
      end

    end
  end
end
```
可以看到，**Rails::Application** 的 **config** 方法其实调用的是 **Rails::Application::Configuration** 的实例，而 **Configuration** 类拥有很多实例方法包括例子中的 **timezone**，初始化的 **timezone** 是 “UTC”。所以，我们通过自定义的 **config.time_zone = 'Beijing'** 覆盖了 **timezone** 的内容，又因为 **config** 方法用的是 **'@config ||='**, 这就保证了 **config** 配置内容正对当前 app 是唯一的。到此，我们应该理解了 **config.time_zone** 到底做了什么。

> 下面还有更深入一点的内容，如果我理解的不对，欢迎指正：

railtie 通过私有化 new，以及定义 instance 方法做到了 railtie 的单例模式，这样就保证了每一个 app 的 railtie 都是唯一的（**Rails::Applicaton** 中将 new 方法 public 了）。那这有什么用呢？我们回到 **Rails::Application::Configuration** 的 initialize，有一行是 **@generators = app_generators**，通过继承关系最终在 **Rails::Railtie::Configuration** 里找到 app_generators 方法的定义，发现这个方法定义了一个同名的类变量 @@app_generators（之后具体的定义逻辑就不说了），这里我们主要关注这是一个 @@ 开头的类变量。这点很重要，因为他是类变量，所以它对于整个 railtie 来说是全局唯一的，也就是**Rails::Application::Configuration** 中的 @generators 是对整个 Rails 全局唯一的配置。什么意思呢？当你在 **MyApp::Application** 中配置了 **config.generators**，它对你所有的 app 实例是通用的！

好了，讲到这差不多了，rails 复杂的结构和精妙的设计需要花很多时间去理解，如果你感兴趣，可以好好把下面的参考资料的视频看上几遍，保证你收益巨大！加油吧！

---

## 参考资料：

* 对 rails 启动过程非常精彩的分析，值得初学者看 5 遍：[http://railscasts-china.com/episodes/the-rails-initialization-process-by-kenshin54](http://railscasts-china.com/episodes/the-rails-initialization-process-by-kenshin54)