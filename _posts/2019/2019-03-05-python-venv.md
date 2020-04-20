---
layout: post
title: 用 venv 虚拟 python 执行环境
date: 2019-03-05
tags: python
---

> python 的 venv 这个概念和 ruby 的 rvm 是一样的，核心思想就在于不同的项目用不同的环境来管理依赖和版本

### 在指定目录（一般是项目目录）下创建用来保存 python 虚拟环境的文件目录（这里用 spider 举例）

```
$ python -m venv ./spider
```

### 激活虚拟环境

```
$ source ./spider/bin/activate
```
激活后在命令提示符的最前端会有 `<spider>` 的字样，表示你已经在这个虚拟环境中了

### pip install ...

此时安装的 package 都会保存到 `./spider/lib/python3/site-packages/`，同时，可以直接从别的路径把 package 拷贝到当前虚拟环境的这个包管理目录，免去从头装很多包的麻烦

### 退出虚拟环境

```
$ deactivate
```

非常方便就可以退出当前虚拟环境，观察命令行提示符前端的 `<spider>` 字样已经消失

### 更换源

临时：
```
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pip -U
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple some-package
```

默认：
```
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

---

## 参考资料：

* [venv — Creation of virtual environments](https://docs.python.org/3/library/venv.html)