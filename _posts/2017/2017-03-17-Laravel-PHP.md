---
layout: post
title: 从 零 开始搭建基于 PHP 的 Laravel 环境
date: 2017-03-17
tags: laravel
---

在 Mac 上，Laravel 的环境有两种配置方式：Homestead 和 Valet, 这里只介绍 [Homestead](https://laravel.com/docs/5.3/homestead), Laravel 官网有详细的教程，我这里用中文做一个简要的说明：

* #### 下载安装 [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
* #### 下载安装 [vagrant](https://www.vagrantup.com/downloads.html)
* #### 下载 vagrant 的 Laravel/homestead 虚拟机

```shell
$ vagrant box add laravel/homestead
```

* #### 下载 Homestead

```shell
$ cd ~
$ git clone https://github.com/laravel/homestead.git Homestead
```

* #### 生成配置文件

```
$ bash init.sh
```

会生成 ~/.homestead 目录，里面有个 Homestead.yaml 就是虚拟机配置文件 

* #### 配置 Homestead

按需在 Homestead.yaml 做相应配置，注意 rsa 的名字和 sharefolder 和本地相同

* #### 配置 /etc/hosts

```
192.168.10.10  homestead.app
```
确保这个 ip 是和 ~/.homestead/Homestead.yaml 配置文件中的一致

* #### 在 ~/Homestead 目录下启动 vagrant 虚拟机

```
$ vagrant up
```
会按照配置启动虚拟机，看一下有没有遇到什么报错，成功的话，在浏览器中就可以打开项目了：

```
http://homestead.app
```

* 新建 Laravel 项目

进入 vagrant/Laravel box：

```shell
$ vagrant ssh
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-51-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

3 packages can be updated.
0 updates are security updates.


vagrant@homestead:~$ 
vagrant@homestead:~$ laravel new blog
 .
 .
 .
Generating autoload files
> php -r "file_exists('.env') || copy('.env.example', '.env');"
> Illuminate\Foundation\ComposerScripts::postInstall
> php artisan optimize
Generating optimized class loader
The compiled class file has been removed.
> php artisan key:generate
Application key [base64:72cTfjG2I7lj/H5FRjNy+HUdiZqB5Xy3XgyQS4advAs=] set successfully.
Application ready! Build something amazing.
vagrant@homestead:~$ 

```

* 我从网上 clone 了一个 demo 项目做练习，这里是 `->` [说明教程](http://www.findalltogether.com/tutorial/simple-blog-application-in-laravel-5/)

第一次跑 migrate 的时候报错：

```shell
vagrant@homestead:~/Code/blog$ php artisan migrate
PHP Notice:  Use of undefined constant MCRYPT_RIJNDAEL_128 - assumed 'MCRYPT_RIJNDAEL_128' in /home/vagrant/Code/blog/config/app.php on line 83

Notice: Use of undefined constant MCRYPT_RIJNDAEL_128 - assumed 'MCRYPT_RIJNDAEL_128' in /home/vagrant/Code/blog/config/app.php on line 83

[Illuminate\Database\QueryException]                                                                                                                                    
  SQLSTATE[42000]: Syntax error or access violation: 1067 Invalid default value for 'created_at' (SQL: create table `users` (`id` int unsigned not null auto_increment p  
  rimary key, `name` varchar(255) not null, `email` varchar(255) not null, `password` varchar(60) not null, `role` enum('admin', 'author', 'subscriber') not null defaul  
  t 'author', `remember_token` varchar(100) null, `created_at` timestamp default 0 not null, `updated_at` timestamp default 0 not null) default character set utf8 colla  
  te utf8_unicode_ci)                                                                                                                                                                                             
                                                                                                  
  [PDOException]                                                                                  
  SQLSTATE[42000]: Syntax error or access violation: 1067 Invalid default value for 'created_at'  
```
原因是 mysql 不接受 0 作为日期格式字段的值，参考下面两篇评论：

[https://github.com/laravel/framework/issues/3602](https://github.com/laravel/framework/issues/3602)    
[https://laravel-china.org/topics/1485](https://laravel-china.org/topics/1485)

我采用的方法是把 config/database.php 的 mysql 配置项 strict 设置改为 true，错误修复。

* 然后网页打开项目时报错：`Use of undefined constant MCRYPT_RIJNDAEL_128 - assumed 'MCRYPT_RIJNDAEL_128'`

网上有很多相关讨论，我的解决方法如下：

```shell
vagrant@homestead:~$ sudo apt-get update
vagrant@homestead:~$ sudo apt-get install php7-mcrypt
vagrant@homestead:~$ php -m | grep mc
mcrypt
memcached
```

* 网页打开项目又有新的报错：`PHP7.1 and Laravel 5.3: Function mcrypt_get_iv_size() is deprecated`

参考以下两篇评论：

[https://github.com/AsgardCms/Platform/issues/271](https://github.com/AsgardCms/Platform/issues/271)
[http://stackoverflow.com/questions/41031076/php7-1-and-laravel-5-3-function-mcrypt-get-iv-size-is-deprecated](http://stackoverflow.com/questions/41031076/php7-1-and-laravel-5-3-function-mcrypt-get-iv-size-is-deprecated)

应该是最新版本 php 的问题，12月份才 release，还未找到针对 php7.1 明确的解决办法，用 php7.0 就 ok 了。

---

## 参考资料：

* 用 Laravel 建立一个 simple Blog 的小例子：
[http://www.findalltogether.com/tutorial/simple-blog-application-in-laravel-5/](http://www.findalltogether.com/tutorial/simple-blog-application-in-laravel-5/)
* Laravel 的中文文档：
[http://laravelacademy.org/post/5744.html](http://laravelacademy.org/post/5744.html)
* 一个对 vagrant 的简要介绍：
[https://segmentfault.com/a/1190000000264347](https://segmentfault.com/a/1190000000264347)
* 知乎上对 vagrant 和 docker 精彩的比较
[https://www.zhihu.com/question/32324376](https://www.zhihu.com/question/32324376)
* Django vs Laravel vs Rails：
[http://www.findalltogether.com/post//django-vs-laravel-vs-rails/](http://www.findalltogether.com/post//django-vs-laravel-vs-rails/)