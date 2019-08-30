---
title: linux系统-权限管理
tags:
  - 权限
categories:
  - linux
date: 2019-08-26 15:37:39
---

# 命令
* [文件权限示例](#ll输出)
* [权限表示法](#权限表示法)
* [useradd](#useradd)：添加用户
* [usermod](#usermod)：更改用户信息
* userdel：删除用户【-r删除主目录】
* id：查看用户id信息
* [passwd](#密码过期范例)：设置或更改密码
    - -l：锁定用户
    - -u：解锁用户
* [chage](#密码过期范例)：设置密码过期信息
* groupadd、groupdel、groupmod：添加、删除、变更组
* groups：查看用户组信息
* gpasswd：向组添加用户
    - -a：单用户添加
    - -M：多用户添加【逗号分隔】
    - -d：删除组用户
    - 范例：gpasswd -a user group
* [chmod](#chmod)：设置文件的基本权限
* [chown](#chown)：设置文件属主属组
* [getfacl](#getfacl)：查看文件访问控制列表
* [setfacl](#setfacl)：设置文件访问控制列表
* [chattr](#chattr)：设置文件隐藏属性
* [lsattr](#lsattr)：查看文件隐藏属性
* [sudo](#sudo)：权限提升
* [su](#su)：切换用户
* users：所有登录用户
* w、who：查看所有用户登录详情
* last：查看系统最近登录情况
* lastlog：显示最近登录的用户名，登录端口及时间
* lastb：登录失败信息

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
>注意：不能更改在线使用者的信息

* -d：指定用户新的家目录，有-m参数时则将原有家目录内容移动至新家目录
* -g/-G：指定基本组、附加组
* -s：更改shell
* -l：更改用户名
* -e mm/dd/yy：指定账号过期时间

# ll输出
【-rw-r--r-- 2 root root 5 8月  14 19:55 1.txt】空格分隔后的字段含义：

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
    - 所有用户权限设置：chmod a+/-/=rwx file
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

# getfacl
>获取文件或目录的访问控制列表

```
1:  # file: somedir/
2:  # owner: lisa
3:  # group: staff
4:  # flags: -s-
5:  user::rwx
6:  user:joe:rwx               #effective:r-x
7:  group::rwx                 #effective:r-x
8:  group:cool:r-x
9:  mask::r-x
10:  other::r-x
11:  default:user::rwx
12:  default:user:joe:rwx       #effective:r-x
13:  default:group::r-x
14:  default:mask::r-x
15:  default:other::---
```

* 1-3表示文件名、属主、属组
* 4显示特殊权限setuid (s), setgid (s), sticky (t)【3个权限都无时，不显示此行内容】
* 5、7、10对应属主、属组、其他人的权限，表示基本的acl条目
* 6、8是命名的用户和组的权限条目
* 9是有效的权限掩码，针对所有的组和命名的用户【但不包含属主、其他人】
* 11-15显示目录的默认权限【普通文件则没有】

# setfacl
>设置文件或目录的访问控制列表

* 语法：`setfacl [-bkndRLPvh] [{-m|-x} acl_spec] [{-M|-X} acl_file] file ...`
* 使用注意
    - 命令行下操作权限时，多个ACL条目以逗号分隔
    - 从文件中读取ACL时，最终会有getfacl式的结果输出
    - 属主、属组、其他人这3个基本的权限条目不可移除，且只能有一个
    - 当ACL包含命名的用户和组的条目时，必须包含有效的权限掩码

## 参数
* -m(--modify)、-M (--modify-file)：定义ACL权限，-m在命令行定义，-M从文件中或标准输入读取要定义的ACl
* -x(--remove)、-X (--remove-file)：移除ACl权限，-x在命令行移除，-X从文件中或标准输入读取要移除的ACL
* -b、--remove-all：移除所有扩展的ACL【不包含对3个基本条目修改的恢复】
* -k、--remove-default：移除默认的ACL
* -n, --no-mask：不计算掩码条目
* --mask：计算掩码条目
* -d、--default：使用默认ACL
* --restore=file：从文件中恢复ACL【使用getfacl -R备份的文件】
* --test：测试模式
* -R, --recursive：递归应用权限
* -L, --logical：操作适用于符号链接关联的目录
* -P, --physical：操作不适用于符号链接关联的目录

## 权限条目
* 用户权限：`[d[efault]:] [u[ser]:]uid [:perms]`：设置命名用户的权限，uid为空则表示属主的权限
* 组权限：`[d[efault]:] g[roup]:gid [:perms]`：设置命名组的权限，gid为空则表示属组的权限
* 权限掩码：`[d[efault]:] m[ask][:] [:perms]`
* 其他人权限：`[d[efault]:] o[ther][:] [:perms]`

## 范例
* 默认权限：setfacl -m d:u:hjq:rwx test
* 设置属主权限：setfacl -m u::r test/
* 设置属组权限：setfacl -m g::rw test/
* 设置权限掩码：setfacl -m d:m:rx test/
* 移除所有扩展权限：setfacl -b test/

# chattr
>改变文件的隐藏属性

* 用法：`chattr [ -RVf -v version ] [ mode ] files...`
* 一般参数：
    - -R：递归设置文件和目录
    - -V：显示设置过程
    - -f：屏蔽错误输出
    - -v version：设置文件版本
* 模式参数：`+-=[acdeijstuADST]`
    - +表示增加模式
    - -表示较少模式
    - =表示只设置此种模式

## 模式参数
* a：只允许增加内容，不允许修改和删除
* A：不修改atime时间戳
* c：文件存储时压缩，读取时自动解压
* d：dump程序启动备份时，不对此文件备份
* i：不能被修改和删除
* s：删除数据的同时将数据从磁盘删除
* S：将数据同步写入磁盘【一般为异步】
* u：与s相反，删除文件但磁盘上不删除【可以用于恢复】

## 范例
* 只许追加：chattr +a 12
* 撤销追加属性：chattr -a 12

## lsattr
* -d：显示目录本身隐藏属性
* -R：递归显示目录及目录下的隐藏属性

查看文件的隐藏属性：lsattr 12

# su
* su：切换用户【默认切换到root】
* su -：切换用户的同时切换环境变量
* sudo：权限提升，一般为切换为root

# sudo
## 基础
* 时间：5分钟内无需再次输入密码
* 命令
    - visudo -c：语法检查
    - visudo -f：加载指定配置文件
    - sudo -l【查看用户sudo权限】
* 文件：/etc/sudoers
* 环境设置：
    - `Defaults    env_reset`【执行命令时环境变量被重置】
    - `Defaults:muker !requiretty`【允许用户远程执行sudo命令】

## 语法
* 范例：
    - 单用户设置：`admin   ALL=(ALL)       /usr/sbin/useradd`
    - 用户组设置：`%admin          ALL=(ALL)       NOPASSWD: ALL`
* 语法：用户或组+来源主机+可以切换到的用户+可以执行的命令
    - 用户或组：admin
    - 来源主机：ALL
    - 可以切换的用户：ALL
    - 可以执行的命令：/usr/sbin/useradd
        + 命令需要使用全路径
        + 命令前使用`！`表示取反【`admin   ALL=NOPASSWD:/sbin/*, !/sbin/fdisk`】，禁止命令需要放在允许命令之后
        + 命令前`NOPASSWD`表示执行sudo时无需输入密码
        + 多个命令之间逗号分隔，且逗号与下个命令之间要有空格

## 别名
>可以对sudo语法中的4个主要条目设置别名  
>别名必须使用大写字母，%表示引用组名

* 用户：`User_Alias ADMINS = admin, baby, %admin`
* 来源主机：`Host_Alias     MAILSERVERS = smtp, smtp2`
* 可切换用户：`Runas_Alias  OP = root, admin`
* 可执行命令：`Cmnd_Alias SOFTWARE = /bin/rpm, /usr/bin/up2date, /usr/bin/yum`

## 远程sudo
配置文件requiretty参数和命令行ssh -t参数的差异

* 以客户端角度看：ssh以-t参数远程运行命令(ssh -t host sudo comand)，无论requiretty如何配置都可以远程sudo执行
* 以服务端角度看
    - 【!requiretty】不要求有tty终端，【ssh host sudo comand】命令可以执行（ansible的sudo方式也可以执行）
    - 【requiretty】要求有tty终端，【ssh host sudo comand】无法执行执行

# 案例
## 故障现象
* 修改密码时报错 passwd: Authentication token manipulation error
* 添加用户报错：unable to lock password file

## 分析
* 检查相关配置文件权限正常：/etc/passwd 和/etc/shadow
* df查看硬盘空间正常
* 使用命令strace -f passwd 追踪分析原因，看到关键报错信息：“No space left on device”，可是df查看硬盘空间没问题，有可能是inode满了，使用df –i查看inode使用情况，确实/var/spool/clientmqueue占用大量的inode
* /var/spool/clientmqueue 生成的文件占用完inode，此目录下文件的产生原因主要是crontab里面的命令没有添加“>/dev/null 2>&1”标准输出、错误输出信息都输出到标准输出，以文件形似存储在clientmqueue

## 解决
删除文件后正常，将crontab命令后面添加“>/dev/null 2>&1”

# 密码过期范例
* 要求：要求oldboy用户7天内不能更改密码，60天以后必须修改密码，过期前10天通知oldboy用户，过期后30天后禁止用户登陆
* passwd实现：passwd -n 7 -x 60 -w 10 -i 30 oldboy
* chage实现：chage -m 7 -M 60 -W 10 -I 30 oldboy
