---
layout: post
title: 在 Mac 上的 Virtualbox 配置双网卡
date: 2018-09-23
tags: server
---

> Virtualbox 的网络在此就不做详细介绍了，网上有很多，可以参考文末的链接。这里直接讲讲笔者遇到的需求：在 virtualbox 上创建一台虚机提供服务，并且需要联通网络。

其实这个需求可以很简单的实现，就是使用 virtualbox 默认的 Bridged Adapter 模式，虚机将会像一台真实的机器一样出现在宿主机的同网段中，从网络的角度而言，虚拟机和宿主机是平级的，在同网段拥有一个 ip，并且可以互相 ping 通，联网。Bridged Adapter 模式很好，能满足绝大多数使用者的需求（多数人的诉求无非就是虚拟机能上网，最多就是能和宿主机 ping 通），但正如它像一台真实机器存在于局域网的特点，它的 ip 将会由 DHCP server 随机分配，这造成了一个后果就是，每当你切换一个网络环境，比如你换了一个 wifi，你的 ip 就变了。这对之前说的大部分人的需求而言是不冲突的，但是笔者的情况是，需要虚拟机提供的服务是绑定 ip 的，如果 ip 变了，那么服务也失效了。

在此先做个申明，我们暂且不讨论从应用本身来做优化调整，比如是否可以通过将服务绑定到自定义域名或者 localhost 之类的方法来解决可变 ip 的问题。笔者只是想通过这个需求，引申出本篇文章想要实现的问题：

> 如何创建 virtualbox 虚拟机，使其能够访问互联网并且可以和宿主机通信，同时保持 ip 不变。

### 最简单的一个方案就是，创建两张虚拟网卡

第一张用 NAT 方式访问 Internet
![network-adapter-1](/assets/img/posts/2018/virtualbox-network/nat-adapter.png "Network Adapter 1")

第二张用 HostOnly 模式保持以固定 ip 与宿主机通信
![network-adapter-2](/assets/img/posts/2018/virtualbox-network/hostonly-adapter.png "Network Adapter 2")

保持固定 ip 的魔法在于，需要预先通过 Host Network Manager 创建一个 vboxnet0 adapter，以此作为宿主机上的 DNS，在第二张网卡的 HostOnly 中选择创建的 vboxnet0 adapter。
![hostnoly-menu](/assets/img/posts/2018/virtualbox-network/host-only-menu.png "HostOnly Menu")
![hostnoly-settings](/assets/img/posts/2018/virtualbox-network/host-only-settings.png "HostOnly Settings")

在宿主机上运行 ifconfig 就能发现这张 vboxnet 的虚拟网卡。
![vboxnet0](/assets/img/posts/2018/virtualbox-network/vboxnet0.png "ifconfig vboxnet0")

### 补充：设置共享剪切板，并且支持文件拖动

[https://blog.csdn.net/yuanguangyu1221/article/details/51870022](https://blog.csdn.net/yuanguangyu1221/article/details/51870022)

