---
layout: post
title: 在 CentOS 搭建 LDAP 服务器（一） - Linux 认证框架 与 LDAP
tags: centos
---

### 0. 引言

本文将分三篇来详细介绍在 centos 7 上搭建 LDAP 服务器（使用开源的 OpenLDAP 实现）的过程，第一篇介绍什么是 OpenLDAP，以及 Linux 集成 OpenLDAP 的原理；第二篇详细介绍 OpenLDAP 的配置，将包含大量具体的操作命令；第三篇介绍如何使用自签证书配置 TLS 加密，但其实第三篇将着重于介绍 SSL 和 OpenSSL 自身的技术，所以也可看成独立的关于加密和证书的文章。

### 1. Linux 的用户验证：PAM 与 NSS

在现代 Linux 系统上，用户的授权系统是通过 [PAM（Pluggable Authentication Module）](https://www.ibm.com/developerworks/cn/linux/l-cn-pam/index.html)实现的，PAM 最初应用在 Solaris 上，现在已成为了标准的 Linux 认证框架。PAM 提供了一个 API 用来映射认证请求到各种不同的服务（这里就包含了 LDAP 提供的认证服务），可以通过 PAM 配置文件来管理这种映射，通常在 `/etc/pam.d/` 目录下，每个配置文件以不同的应用命名，比如 login、sshd、sudo 等。以下是 `/etc/pam.d/system-auth` 的一个片段：

```
auth        required      pam_env.so
auth        required      pam_faildelay.so delay=2000000
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
auth        required      pam_deny.so

account     required      pam_unix.so
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
account     required      pam_permit.so
```

大概看一下 PAM 配置文件的样子，本文不打算深入分析 PAM，只要记住 PAM 是 Linux 上用来管理用户授权验证的模块就够了。用户认证后，还需要获取到用户的信息，比如 uid、gid，这时候就轮到 NSS 出场了。

[NSS（Name Service Switch）](https://docs.oracle.com/cd/E19455-01/806-1387/6jam6926d/index.html)是一个通用的名称解析框架，包装了和各种存储的交互模块并提供了一个通用的 API 接口，这个概念有点像 web 框架中的数据库接口，开发者不必再关心底层具体是哪种数据库实现，只要简单地应用框架提供的接口就行了。存储的解析库可以是文件（/etc/passwd），MySQL，当然还包括本文的主角 LDAP。NSS 的配置文件是 `/etc/nsswitch.conf`，在 OpenLDAP 的配置中会用到。

### 2. OpenLDAP，LDAP 与 X.500

先认识一个名词：目录服务。目录服务是一种特殊的数据库，它专门用在查询多、修改少的场景。

简单来说，[OpenLDAP](http://www.openldap.org/) 是一款提供目录服务（directory services）的软件，它是 LDAP 协议在 Linux 系统上的一个实现，类似得，在 Windows 系统上也有一个很有名的 LDAP 实现，叫做 [AD（Active Directory）](https://baike.baidu.com/item/%E6%B4%BB%E5%8A%A8%E7%9B%AE%E5%BD%95/1765909?fromtitle=Active%20Directory&fromid=761012)。

LDAP（Lightweight Directory Access Protocal）是在 [RFC1487](https://tools.ietf.org/html/rfc1487) 中定义的，一个基于 X.500 标准的轻量级目录服务访问的协议。既然是『轻量』，就知道 DAP 其实是个完整的协议，LDAP 在其基础上发展出了更加灵活易用的方式，并通过 TCP/IP 建立网络连接提供服务。

那什么又是 X.500 呢？

[X.500](https://baike.baidu.com/item/X.500/8863300) 是一个类似于电话号码簿的电子化的目录协议，它可以有效的帮助一个组织或公司统一管理人员信息。而 LDAP 正是基于 X.500 的标准开发出来的一套更加灵活、简单的目录访问协议，LDAP 的全称是 Lightweight Directory Access Protocal，可以查看[百度百科](https://baike.baidu.com/item/LDAP)的解释。如其名，LDAP 也是一种协议，包含了一种设计成树状结构的存储数据的方案，可以先简单地把 LDAP 的目录理解成一种数据库，它的设计方式在存储用户信息上相比 MySQL 这种关系型数据库有很大的优势。

为什么要用 LDAP ？

我们想象一个场景，公司有 1 个系统管理员和 100 个员工，管理员需要用 admin 账户登录任意一台员工电脑处理 IT 问题，如果要在每台设备上都创建一个 admin 账户就太繁琐了，又如果公司有一千、一万个员工呢？你也许会想到用配置好的镜像统一安装操作系统，这样的确避免了为每一台设备创建用户的过程，但是如果 admin 需要修改密码呢？你也许又会想到自动化，可以使用工具统一管理所有员工的设备。倘若继续往下设想，系统的复杂性不断提高，最终得到的结果是用一个庞大的工程来完成了一个『小工具』就能实现的功能，这个『小工具』就是 LDAP。在此不深入讨论 LDAP 与其它解决方案的优劣，这更多是一个取舍和业务问题，事实上 LDAP 可以解决更多关于用户验证的问题，比如多个应用的统一验证管理，在此我只是想说明，如果需要满足上述需求，部署一个 LDAP 服务器就能轻松搞定，而且实际上几乎所有上规模的公司都是这么做的。

在 [OpenLDAP 官网](http://www.openldap.org/doc/admin24/intro.html#When%20should%20I%20use%20LDAP)中也介绍了适合使用 OpenLDAP 的几个场景，比如验证系统、地址簿、资产追踪、用户资源管理等。

关于 LDAP 的使用和目录结构，本文你就不详细展开，可以参考[此文](https://www.golinuxcloud.com/ldap-tutorial-for-beginners-configure-linux/)和下图树形结构：

![intro_dctree](/assets/img/posts/2019/openldap-config/intro_dctree.png "Intro Dctree")

### 3. Linux 与 LDAP 的集成 -- nss-pam-ldapd

OpenLDAP 是 C/S 架构的，在服务器上要安装 openldap-servers 软件包，[slapd](http://www.openldap.org/software/man.cgi?query=slapd)就是服务进程，配置文件都在 `/etc/openldap/slapd.d` 目录中，而 OpenLDAP 的数据文件在 `/var/lib/ldap` 目录中，在第一次启动 slapd 服务后会生成数据库文件。

客户端上要安装 openldap-clients 软件包，openldap-clients 是用来和 server 交互的，比如它提供了 ldapsearch 接口用来查询数据，往往 server 上也需要安装 openldap-clients 包，因为 server 本身也是通过 openldap-clients 管理的。

与 Linux 的用户系统集成有两种途径，一种是通过 nss-pam-ldapd 直接与 ldap 通信，一种是通过 SSSD 进行转发。

* #### 1.1 nss-pam-ldapd 

  这是一个在客户端上安装的软件包，用来让 PAM 和 NSS 与 LDAP server 交互的模块，它会起一个 nslcd 的服务，并提供相关 so 文件供 PAM 和 NSS 使用。如下图，如果未安装 nss-pam-ldapd 的话（进行 authconfig）会报错：

  ![lack-nss-so](/assets/img/posts/2019/openldap-config/lack-nss-so.png "Lack nss so")

  概括一下，openldap-clients 是用来和 server 交互的操作数据的，slapd 是 openldap-servers 的服务进程，client 需要配置 PAM 和 NSS 启用 LDAP 作为数据源，然后通过 nslcd 进程访问 LDAP server 达到集中管理数据的目的。

* #### 1.2 [SSSD - System Security Services Daemon](https://docs.pagure.org/SSSD.sssd/)

  > 官网介绍：
  > SSSD is a system daemon. Its primary function is to provide access to local or remote identity and authentication resources through a common framework that can provide caching and offline support to the system. It provides several interfaces, including NSS and PAM modules or a D-Bus interface.

  简单来说，SSSD 提供了一个中间接口用来统一管理验证框架，对于操作系统而言只要配置好 SSSD 就够了，至于背后是 LDAP 还是 Kerberos 就交给 SSSD 处理就行了。SSSD 的服务进程是 sssd，软件包名字也是 sssd，当然 yum install 会安装上许多依赖。但回过来说，即便是利用 SSSD 来管理验证，我们的确是不需要 nslcd 了，但是在操作系统上 PAM 和 NSS 的配置还是相似的，比如直接配置 ldap 时用的是 pam_ldap.so 是，现在变成 pam_sss.so 了。

---

## 参考资料：

* [深入 Linux PAM 体系结构](https://www.ibm.com/developerworks/cn/linux/l-cn-pam/index.html)
* [Solaris Naming Administration Guide](https://docs.oracle.com/cd/E19455-01/806-1387/6jam6926f/index.html)
* [LDAP Implementation HOWTO](http://www.tldp.org/HOWTO/archived/LDAP-Implementation-HOWTO/pamnss.html)
* [/etc/nsswitch.conf](https://netbsd.gw.com/cgi-bin/man-cgi/man?nsswitch.conf+5+NetBSD-1.4.3)
* [nss-pam-ldapd](https://arthurdejong.org/nss-pam-ldapd/)
* [LDAP Tutorial for Beginners](https://www.golinuxcloud.com/ldap-tutorial-for-beginners-configure-linux/)
* [RedHat - CHAPTER 28. LIGHTWEIGHT DIRECTORY ACCESS PROTOCOL (LDAP)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/deployment_guide/ch-ldap#s1-ldap-v3)
* [RFC1487 - X.500 LDAP](https://tools.ietf.org/html/rfc1487)
* [OpenLDAP Main Page](http://www.openldap.org/)
* [OpenLDAP Software 2.4 Administrator's Guide](http://www.openldap.org/doc/admin24/)
* [X.500](https://baike.baidu.com/item/X.500/8863300)
* [SSSD documentation](https://docs.pagure.org/SSSD.sssd/)
* [Configure SSSD For LDAP on CentOS 7](https://tylersguides.com/guides/configure-sssd-for-ldap-on-centos-7/)