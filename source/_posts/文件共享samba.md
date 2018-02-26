---
title: 文件共享samba
date: 2018-02-26 16:09:59
tags: ['samba']
categories: ['software']
---
# 安装samba软件

# 用户设置
## 创建系统用户组
groupadd marketop
## 创建系统用户
* useradd -g marketop -s /usr/sbin/nologin -M ceshi1
* useradd -g marketop -s /usr/sbin/nologin -M marketop

## 将系统用户转为samba用户
>同时设置密码

* smbpasswd -a marketop
* smbpasswd -a ceshi1

# 目录设置
## 创建目录
mkdir market_operation
## 变更目录所有者
chown ceshi1.marketop market_operation/
## 目录设置粘滞位
>当一个目录被设置为"粘着位"(用chmod a+t),则该目录下的文件只能由 
>一、超级管理员删除 
>二、该目录的所有者删除 
>三、该文件的所有者删除 
>也就是说,即便该目录是任何人都可以写,但也只有文件的属主才可以删除文件。

chmod a+t market_operation/

# 配置文件设置
## 可用参数
* [public] 共享文件夹名称
* path 共享路径
* public 公共文件夹，不许凭证可访问（访问权限在本地设置）
* writable 可对文件夹写操作
* Valid users 有访问文件夹权限的用户【@为组 】
* browseable 该文件夹是否可见
* read only =yes （默认）
* write list= @shichang，cw1  可写权限列表（当前面为readonly=yes 时可写用户列表）
* directory mask= 0744 （文件夹建立后的默认权限）
* create  mask = 0600（文件建立后的默认权限）

## 范例
```
[global]
   workgroup = WORKGROUP
        server string = %h server (Samba, Ubuntu)
   dns proxy = no
   security = user
   log file = /var/log/samba/log.%m
   max log size = 1000
   syslog = 0
   panic action = /usr/share/samba/panic-action %d
   server role = standalone server
   passdb backend = tdbsam
   obey pam restrictions = yes
   unix password sync = yes
   passwd program = /usr/bin/passwd %u
   passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
   pam password change = yes
   map to guest = bad user
   usershare allow guests = yes
# 自定义选项
[市场部]
        path = /home/dell/market_operation
        readonly = yes
        write list = zhangwanting
        valid users = @marketop
        directory mask = 0755
        create mask = 0640
```

# 测试
## windows
* 连接：`win + R`  =》 `\\10.10.10.244`
* 切换用户测试：
    *  删除网络连接：net use `****` /d

## mac
finder =》 连接服务器 =》 `smb://10.10.10.244`

## linux
`mount -t cifs –o username=sc1,passwordd=sc1 //10.10.10.1/Market   /mnt`
