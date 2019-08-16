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

# ll输出
-rw-r--r-- 2 root root 5 8月  14 19:55 1.txt

空格分隔后的字段含义：

* -：说明此为普通文件
* rw-：属主权限
* r--：属组权限
* r--：其他人权限
* 2：硬链接数
* root：属主
* root：属组
* 5：最后修改时间【日】
* 8月：最后修改时间【月】
* 14：文件大小
* 19:55：最后修改时间
* 1.txt：文件名

# 权限表示法
* 权限数字表示法
    - 数字表示法：4-读 2-写 1-执行 0-无权限(数字可叠加)
    - 范例：644【3位数字式】
        + 属主权限：6=4+2+0
        + 属组权限：4=4+0+0
        + 其他人权限：4=4+0+0
* 权限字母表示法
    - 字母表示法：r-读 w-写 x-执行 -无权限
    - 范例：rw-r--r--
        + 属主权限：rw-
        + 属组权限：r--
        + 其他人权限：r--
* 特殊权限表示法
    - 属主权限suid：使文件在执行阶段具有文件所有者的权限
        + 数字表示法：4【4位数字式的首位】
        + 字母表示法：(u+)s
    - 属组权限sgid：运行者将具有所属组的特权
        + 数字表示法：2【4位数字式的首位】
        + 字母表示法：(g+)s
    - 属主之外其他人权限stick：常用于目录，表示该目录下用户建立的文件只能自己删除
        + 数字表示法：1【4位数字式的首位】
        + 字母表示法：(o+)t
* 设置权限时的用户组表示
    - a：所有用户
    - u：属主
    - g：属组
    - o：其他人

# chmod
>更改文件权限

* 语法：chmod 权限设置 文件
* 范例：
    - 数字式设置权限：chmod nnnn file
    - 字母式增加/减少/精确设置权限：chmod u+r,g-w,o=rwx file
    - 所有用户权限设置：chmod a+/-/=r/w/x file
    - 特殊权限设置：chmod u+s,g+s,o+t file

# chown
>更改文件属主属组

* 语法：chown 参数 用户组 文件
* 参数
    - -R 递归更改属性
* 范例
    - chown user:group file
    - chown user.group file
    - chown user file
    - chown .group file
    - chown :group file
