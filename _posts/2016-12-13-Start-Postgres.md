---
layout: post
title: Postgres 的启动
date: 2016-12-13
tags: database
---

记录一个今天遇到的很坑的问题：电脑重启后，postgres 没有重启，于是我去手动启动，然而一直启动失败。

网上有很多帖子说 postgres 启动命令为：

```
pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log start
```
注意这只是默认安装路径，你要找到你自己的 postgres 安装路径。
postgres 启动是要指明文件路径的，具体命令见以下（来自 `postgres --help`）：

```
pg_ctl start   [-w] [-t SECS] [-D DATADIR] [-s] [-l FILENAME] [-o "OPTIONS"]
```
我的 postgres 安装路径为：

```
~/Library/Application Support/Postgres/var-9.5
```
我一开始跑的命令如下：

```
pg_ctl -D ~/Library/Application Support/Postgres/var-9.5 -l ~/Library/Application Support/Postgres/var-9.5/postgres-server.log start
```
然而启动失败，报错如下：

```
pg_ctl: unrecognized operation mode "Support/Postgres/var-9.5"
```
这个报错一开始深深误导了我，我去查 unrecognized operation mode 查了好久，我怀疑是不是我的启动方式有问题。

后来我发现还有一种方式可以启动 postgres，不过是前台启动，于是我又死马当活马医地试了一下：

```
postgres -D ~/Library/Application Support/Postgres/var-9.5
```
启动也失败了，报错如下：

```
postgres: invalid argument: "Support/Postgres/var-9.5"
```
但是！这个报错比之前的良心太多了，因为通过 invalid argument 这个关键字我瞬间顿悟到了错误的根源，在于 `"Application Support"` 这个文件名包含了一个空格，导致启动命令以为传入了两个参数。

知道错误的原因后，通过以下启动命令成功启动 postgres:

```
pg_ctl -D "/Users/spiderevgn/Library/Application Support/Postgres/var-9.5" -l "/Users/spiderevgn/Library/Application Support/Postgres/var-9.5/postgres-server.log" start
```
终于启动成功，打开 Postico, 点击 connect, 连接成功！
