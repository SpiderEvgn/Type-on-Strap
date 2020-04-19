---
layout: post
title: 在 CentOS 搭建 LDAP 服务器（二） - OpenLDAP 配置
date: 2019-09-22
tags: centos
---

### 引言

这是『在 CentOS 搭建 LDAP 服务器』系列的第二篇，在第一篇我们知道了 Linux 集成 LDAP 的原理，这篇我们将详细学习 OpenLDAP 在 CentOS 7 上的配置（但不包括 TLS，TLS 将放到第三篇单独讲）。

### OpenLDAP server 配置

1. 安装 OpenLDAP 依赖

    ```
    $ yum install -y openldap-servers openldap-clients migrationstools
    ```

2. 生成 openldap 管理员密码

    ```
    $ slappasswd   # 输入明文密码，返回结果是 '{SSHA}' 开头的密文
    ```

3. 基础配置

    ```
    $ cd /etc/openldap/slapd.d/cn=config
    $ vi olcDatabase={2}hdb.ldif
        olcSuffix: dc=[my-domain],dc=com            # 把 my-domain/com 改成自己的 dc
        olcRootDN: cn=Manager,dc=[my-domain],dc=com
        olcRootPW: {SSHA}…                          # 这里填写刚才生成的密文
    $ vi olcDatabase={1}monitor.ldif
        olcAccess: …cd=Manager,dc=[my-domain],dc=com…
    ```

4. 启动服务

    ```
    $ systemctl start slapd
    $ systemctl enable slapd
    ```

5. 复制 DB_CONFIG 文件到 OpenLDAP 数据库目录

    ```
    $ cp -v /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
    $ chown ldap:ldap /var/lib/ldap/DB_CONFIG
    ```

6. 生成基础 schema

    ```
    $ ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/cosine.ldif
    $ ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/nis.ldif
    $ ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/inetorgperson.ldif

    ```

7. 在根目录下创建 ldap 目录用来保存中间配置文件

    ```
    $ mkdir ~/ldap
    $ cd ~/ldap
    $ vi base.ldif
        dn: dc=dxc,dc=com
        objectClass: top
        objectClass: dcObject
        objectClass: organization
        o: DXC.technology
        dc: dxc

        dn: cn=Manager,dc=dxc,dc=com
        objectClass: organizationalRole
        cn: Manager
        description: Directory Administrator

        dn: ou=People,dc=dxc,dc=com
        objectClass: organizationalUnit
        ou: People

        dn: ou=Group,dc=dxc,dc=com
        objectClass: organizationalUnit
        ou: Group
    ```

8. 创建测试用户

    ```
    $ useradd testing1
    $ useradd testing2
    $ echo "testing1" | passwd "testing1" --stdin
    $ echo "testing2" | passwd "testing2" --stdin
    $ grep ":100[1|2]" /etc/passwd > passwd          # 根据实际 id 修改
    $ grep ":100[1|2]" /etc/group > groups
    ```

9. 配置 migrationtools

    ```
    $ cd /usr/share/migrationtools/
    $ vi migrate_common.ph
        - /padl.com
        $DEFAULT_MAIL_DOMAIN = “dxc.com”
        $DEFAULT_BASE = “dc=dxc,dc=com”
        - /EXTENDED_SCHEMA
        EXTENDED_SCHEMA = 1
    ```

10. 运行 ldif 迁移

    ```
    $ ./migrate_passwd.pl /root/ldap/passwd /root/ldap/users.ldif
    $ ./migrate_group.pl /root/ldap/groups /root/ldap/groups.ldif
    ```

11. 导入数据到 OpenLDAP server

    ```
    $ cd ~/ldap
    $ ldapadd -x -W -D "cn=Admin,dc=dxc,dc=com" -f base.ldif
    $ ldapadd -x -W -D "cn=Admin,dc=dxc,dc=com" -f users.ldif
    $ ldapadd -x -W -D "cn=Admin,dc=dxc,dc=com" -f groups.ldif
    # 查看用户
    ldapsearch -x cn=testing1 -b dc=dxc,dc=com
    ```

11. 如果防火墙已启用，则开放 ldap 端口

    ```
    $ firewall-cmd --permanent --add-service=ldap
    $ firewall-cmd --reload
    ```

12. 配置 OpenLDAP server 日志

    ```
    $ echo local4.*  /var/log/ldap.log >> /etc/rsyslog.conf
    $ systemctl restart rsyslog
    ```

### OpenLDAP client 配置 - nss-pam-ldapd

1. 安装 OpenLDAP 依赖

    ```
    $ yum install -y openldap-clients nss-pam-ldapd
    ```

2. 配置 OpenLDAP server

    ```
    # 先用 --test 测试执行结果
    $ authconfig --enableldap --enableldapauth --ldapserver=ldap://[ldap-server-ip] --ldapbasedn='dc=[my-domain],dc=com' --enablemkhomedir --test
    # 用 --update 执行命令
    $ authconfig --enableldap --enableldapauth --ldapserver=ldap://[ldap-server-ip] --ldapbasedn='dc=[my-domain],dc=com' --enablemkhomedir --update
    ```

### OpenLDAP client 配置 - sssd

1. 安装 OpenLDAP 依赖

    ```
    $ yum install -y openldap-clients sssd
    ```

2. 配置 OpenLDAP server

    ```
    $ cd /etc/sssd
    $ vi sssd.conf
        [sssd]
        services = nss, pam
        domains = default
        [pam]
        [domain/default]
        autofs_provider = ldap
        cache_credentials = True
        id_provider = ldap
        auth_provider = ldap
        chpass_provider = ldap

        ldap_uri = ldap://[ldap-server-ip]
        ldap_search_base = dc=dxc,dc=com
        ldap_id_use_start_tls = False
        #ldap_tls_cacertdir = /etc/openldap/cacerts
        #ldap_tls_reqcert = never

    $ chmod 600 sssd.conf
    $ systemctl enable sssd
    $ systemctl start sssd

    # enablesssd 更新 nsswitch，enablesssdauth 更新 pam.d（pam_sss.so）
    $ authconfig --enablesssd --enablesssdauth --enablemkhomedir --update
    ```

### authconfig 参数介绍

关于 authconfig 命令笔者网上查了很久没找到详细的解释，只有粗略的介绍每条参数的作用，所以关于以下的内容都是笔者亲自实践的结果，或许不全，提供参考。

* `--enableldap` 参数会在 `/etc/nsswitch.conf` 里加上 ldap，比如 `passwd:  files ldap`，这样在获取用户信息的时候就会多加上 LDAP 的数据源 ，比如 `getent passwd testing1` 最后就会通过 LDAP 得到结果。

* `--enableldapauth`  会在 `/etc/pam.d/system-auth` 里加上 pam_ldap.so，比如 `auth   sufficient   pam_ldap.so use_first_pass`，这样 PAM 验证就会交给 LDAP 处理。

* `--enablesssd` 会在 `/etc/nsswitch.conf` 里加上 sss，比如 `passwd:  files sss`

* `--enablesssdauth` 会在 `/etc/pam.d/system-auth` 里加上 pam_sss.so，比如 `auth   sufficient   pam_sss.so use_first_pass`

* `--enablemkhomedir` 顾名思义会在用户第一次登录的时候在 /home 创建根目录

---

## 参考资料：

* [Step-by-Step Tutorial: Install and Configure OpenLDAP in CentOS 7 Linux](https://www.golinuxcloud.com/install-and-configure-openldap-centos-7-linux/)
* [authconfig](https://www.tutorialspoint.com/unix_commands/authconfig.htm)
* [LDAP authentication](https://wiki.archlinux.org/index.php/LDAP_authentication#NSS_Configuration)
* [关于 nscd，nslcd 和 sssd 套件的综述](http://coldbloodx.blog.sohu.com/306407471.html)
* [wbwangk/wbwangk.github.io - NSLCD](https://github.com/wbwangk/wbwangk.github.io/wiki/NSLCD)
* [wbwangk/wbwangk.github.io - SSSD](https://github.com/wbwangk/wbwangk.github.io/wiki/SSSD)
* [使用 OpenLDAP 集中管理用户帐号](https://www.ibm.com/developerworks/cn/linux/l-openldap/index.html)



