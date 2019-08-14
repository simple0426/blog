---
title: linux系统-权限管理
tags:
categories:
---
# 用户
* useradd：添加用户
* usermod：更改用户信息
* userdel：删除用户【-r删除主目录】
* id：查看用户id信息
* passwd：设置或更改密码
    - -l：锁定用户
    - -u：解锁用户
* chage：设置密码过期信息
* groupadd、groupdel、groupmod：添加、删除、变更组
* groups：查看用户组信息
* gpasswd：向组添加用户
    - -a：单用户添加
    - -M：多用户添加【逗号分隔】
    - -d：删除组用户
    - 范例：gpasswd -a user group

# 权限
* sudo、su
* chattr、chmod、chown
* setfacl、getfacl

# 存储位置
* 用户：/etc/passwd
* 用户密码：/etc/shadow
* 组：/etc/group
* 组密码：/etc/gshadow

# useradd
>默认使用/etc/default/useradd配置

* -d：指定用户用户主目录
* -g：指定用户基本组
* -G：指定用户附加组
* -m：创建用户主目录
* -M：不创建用户主目录
* -r：创建一个系统账号【默认不会创建家目录，需-m创建】
* -s：设置登录shell
* -u：设置用户uid

# usermod
>不能更改在线使用者的信息

* -d：指定用户新的家目录，有-m参数时则将原有家目录内容移动至新家目录
* -g/-G：指定基本组、附加组
* -s：更改shell
* -l：更改用户名
* -e mm/dd/yy：指定账号过期时间
