---
layout: post
title: OpenLDAP 实现组嵌套
date: 2019-10-26
tags: centos
---

### 引言

之前记录过如何在 CentOS 搭建 LDAP 服务器，这次介绍一个更加高级的用法：组嵌套，即在 LDAP 中定义了组与组之间的从属关系，然后在 Linux 上将所有关联组（整个组链）赋给最终用户。同样使用 OpenLDAP 实现，客户端用 SSSD 进行配置。

### 什么是组嵌套？

组嵌套是组与组之间的关系，可以理解为用户和组之间的关系在组之间的延伸，比如一个用户属于一个组，而一个组也可以属于另一个组，相当于大部门底下的小部门，最后小部门里面再有员工。操作系统（Linux）是没有组嵌套这个概念的，即组与组之间不存在任何关系。所以接下来我将讨论两个方面：
1. LDAP 如何实现组嵌套
2. 组嵌套的关系最终在操作系统如何落地

### 服务端实现

[LDAP schema](https://ldap.com/understanding-ldap-schema/)


[rfc2307bix](https://ldapwiki.com/wiki/SchemaRFC2307Bis)

groupOfUniqueNames
uniqueMember


1、在RFC2307，定义了主机是以posixGroup(objectclass)的memberUid属性来做用户的映射，然而memberUid的值是用户名称，这样没法实现嵌套组，应为一个名称无法区分用户名与用户组。在RFC2037bis中，新增加了一个groupOfMembers（objectclass）,它利用member来关联用户，而是他的值是dn,因此member既可以存用户dn,也可以保存组dn,所以进过客户端的支持，可以实现嵌套。
2、RFC2307对应openldap的nis.schema文件，RFC2307bis对应的schema文件可以去https://github.com/jtyr/rfc2307bis获取。后者是兼容前者的，因此，添加后者到系统，需要替换前者，否则有属性重复。
3、服务端数据如何保存？用户组的objectclass利用groupOfMembers替换posixGroup,用member属性来表示组成员，以及组之间的嵌套关系。（https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=66854729）
4、关于客户端，此处仅说 SSSD 客户端。在 sssd.conf 中添加 `ldap_schema=rfc2307bis` 即可，此外可以通过 `ldap_group_nesting_level` 配置组嵌套的层数，默认是2.（https://linux.die.net/man/5/sssd-ldap）


### 客户端实现


> By default, SSSD will use the more common RFC 2307 schema. The difference between RFC 2307 and RFC 2307bis is the way which group membership is stored in the LDAP server. In an RFC 2307 server, group members are stored as the multi-valued attribute memberuid which contains the name of the users that are members. In an RFC2307bis server, group members are stored as the multi-valued attribute member (or sometimes uniqueMember) which contains the DN of the user or group that is a member of this group. RFC2307bis allows nested groups to be maintained as well.

> So the first thing to try when you hit this situation is to try setting ldap_schema = rfc2307bis, deleting /var/lib/sss/db/cache_DOMAINNAME.ldb and restarting SSSD. If that still doesn’t work, add ldap_group_member = uniqueMember, delete the cache and restart once more. If that still doesn’t work, it’s time to file a bug.



---

## 参考资料：

* [在openLdap上添加memberOf属性](https://www.cnblogs.com/jiaoyiping/p/6849311.html)
* [SSSD Frequently Asked Questions](https://docs.pagure.org/SSSD.sssd/users/faq.html)