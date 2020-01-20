---
layout: post
title: iF.SVNAdmin
tags:
  - LDAP
  - SVN
---

[iF.SVNAdmin](http://svnadmin.insanefactory.com/)是一个svn的可视化管理界面工具。不需要数据库，完全基于svn的授权验证文件，支持LDAP的用户、群组功能。可以针对路径配置群组、用户的读写权限。

关键词：
- apache
- ldap
- svn
- php
- linux


我们的主要需求是能够接入LDAP来统一管理用户。

代码本身是PHP的，部署比较容易，此外需要安装PHP的LDAP扩展。

LDAP需要配置两个地方，一是这个管理后台的设置，一是svn在apache里的配置。

前者也容易，注意那个`Bind DN`不是非得填DN，平常用来bind的账户就可以。

后者示例：

```bash
# /etc/httpd/conf.modules.d/10-subversion.conf
LoadModule dav_svn_module     modules/mod_dav_svn.so
LoadModule authz_svn_module   modules/mod_authz_svn.so
LoadModule dontdothat_module  modules/mod_dontdothat.so

<Location /test>
    DAV svn
    SVNListParentPath on
    SVNParentPath "/data/svn/test"

    AuthType Basic
    AuthName "Welcome to my SVN, Please input password!"
    AuthBasicProvider ldap file
    AuthUserFile "/data/svn/test/trunk/conf/passwd"
    
    AuthLDAPBindDN "binduser@test.com"
    AuthLDAPBindPassword "bindpassword"
    AuthLDAPURL "ldap://192.168.1.1:389/OU=部门,OU=公司,DC=TEST,DC=COM?sAMAccountName"

    AuthzSVNAccessFile "/data/svn/test/trunk/conf/authz"
    Require valid-user
    Satisfy all
</Location>
```

ldap和文件形式可以共存，哪个前就优先哪个。具体参照下面的链接。

Reference:
- https://community.wandisco.com/s/article/Apache-configuration-for-LDAP-authentication
- https://blog.csdn.net/zhq_zvik/article/details/80084783