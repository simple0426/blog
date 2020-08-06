---
title: shell命令行
date: 2019-08-27 15:38:54
tags: ['命令行']
categories: ['linux']
---

# 系统管理
* [crontab](#crontab)：设置周期性任务
* at：一次性任务（使用较少）
* head -n：显示前n行内容【默认10】
* tail：末尾显示
    - -n：显示末尾n行内容【默认10】
    - -f：持续追踪末尾输出
* echo：回显文本
    - -e：支持特殊转义符：【\n \t等】
    - -n：移除末尾换行
* [read](#read)：从标准输入读取一行内容
* seq `[start [step]]` end：打印数字序列
    - -f printf格式
    - -s 分隔符【默认\n】
    - -w 列表前加0使宽度相等
* man：查看命令man文档
* xargs：从标准输入读取内容，构建并执行命令【一般用于不支持管道的命令，如ls；或一次无法处理过多参数的命令，如rm】
    - -I `{}`：指定替换字符串（一般使用`{}`）：find . -type f|xargs -I {} mv {} ..
    - -a file：从文件中读取内容：xargs -a /etc/hosts -I {} echo {}
    - -d delimiter：定义输入分隔符【单字符，默认为换行符】
* reboot、init、shutdown、poweroff、halt：服务器启停控制
* history：查看命令历史【-c 清除命令历史】
* which：查看命令全路径
* time：执行时间统计
* watch：周期性的执行程序，同时全屏显示输出
* alias、unalias：命令别名
* ulimit：控制shell终端可使用的资源
* whatis：显示命令含义
* logger：向syslog写入特定信息
    - -i 在每行都记录进程ID
    - -t logger_test 每行记录都加上“logger_test”这个标签
    - -p local3.notice 设置记录的设备和级别
    - 范例：echo "this is message"|logger -it logger_test -p local3.notice
* dmesg：系统启动过程
* uname：显示系统信息
* dmidecode：查询bios信息
* [date](#时间)：时间

# 软件管理
* [chkconfig](#chkconfig)：自启动管理【centos6】
* [systemctl](#systemctl)：服务控制【centos7】
* [rpm](#rpm)：rpm包安装、卸载、查询
* [yum](#yum)
* [dpkg](#dpkg)
* [apt](#apt)
* [源码安装](#程序源码安装)

## 系统服务管理
* sysV是centos6之前控制系统服务的工具
    - chkconfig是管理系统各个运行级别下服务的启停
    - service则控制系统服务的启停
* system是centos7控制系统服务的工具
    - systemctl
* 系统运行级别：
    - 0：关机
    - 1：单用户模式
    - 2：无网络多用户命令行模式
    - 3：有网络多用户命令行模式
    - 4：不可用
    - 5：带图形界面的多用户模式
    - 6：重启

# crontab
## 配置文件
* /etc/cron.deny：不允许使用cron的用户
* /var/spool/cron：所有用户cron文件存放的目录

## 命令参数
* -u：指定要操作cron的用户【默认操作自己的cron】
* -l：查看crontab配置内容
* -r：删除crontab
* -e：编辑crontab

## 字段含义
|       \*       |    8-18   |    \*    |   5   |      0,1,4       |
|----------------|-----------|----------|-------|------------------|
| 每分钟执行一次 | 8点到18点 | 忽略此项 | 5月份 | 周日，周一，周四 |

* 字段1：一个小时的第几分（0-59）
* 字段2：一天的第几小时（0-23）
* 字段3：一个月中的第几天（1-31）
* 字段4：一年中的第几月（1-12）
* 字段5：一周中的第几天（0-7）【0,7都是周日】

* 其他选项
    - \*：所有时间点
    - -：连续的时间段
    - ,：间隔的时间点
    - /：时间频率

## 注意事项
* 命令行或脚本试验后【如变量、特殊字符处理】，crontab里书写
* 使用&&连接先后顺序的命令
* 定时任务结尾加>/dev/null 2>&1 【重定向所有输出到空设备】
* 添加注释

# read
从标准输入读取一行内容，语法：read `[选项]` name-1 `[... name-n]`
## 选项
* -a：将空格分隔的多个字符串组成一个数组赋值给变量name
* -d：定义行终止符【默认换行符\n】
* -p：设置提示语
* -s：隐藏输入内容【可用于密码输入】
* -t：设置交互式超时时间

## 范例
* readline：`while read line;do echo $line;sleep 2;done < /etc/hosts`
* read：`read -p 'pls input you name:' -t 30 -d \# -s name`

# 时间
## BIOS与系统时间
系统每次启动时会读取BIOS时间，将之赋给系统时间；之后系统时间将独立运行。

## date命令
>显示或设置系统时间

* -d, --date=STRING：显示指定描述的时间
* -f, --file=DATEFILE：从文件中读取并显示指定描述的时间【一行一个】
* -s, --set=STRING：根据字符串的描述设置时间

### 时间字符串
时间字符串可以是任何人类可读的用于标识时间的字符串，比如：

* Sun, 29 Feb 2004 16:21:42 -0800
* 2004-02-29 16:21:42
* next Thursday
* 3 hours
* -3 years

### 范例
* 简单表示法：“date +%D\ %T”
* 全量表示法：“date +%Y-%m-%d\ %H:%M:%S”
* 3天前表示：“date -d "-3 days"”

##  时间设置
* 手动设置系统时间：date -s "01/01/2014 13:16:13"
* 使用ntp同步时间服务器：ntpdate -u time.windows.com
* 将系统时间写入bios：clock -w

## ntpdate与ntpd的区别
* ntpd在实际同步时间时是一点点的校准过来时间的，最终把时间慢慢的校正对。
* ntpdate不会考虑其他程序是否会阵痛，直接调整时间。

# rpm
* -qa：查询软件是否安装【grep】
* -qf：查询文件属于哪个软件包
* -ql：查询软件包的展开文件列表
* -ivh：安装软件
* -e：卸载软件

# yum
是一个在rhel、Centos、SUSE中的shell前端软件包管理器，能从指定的服务器自动下载和安装软件包，并解决软件之间的依赖性。
## 命令
* yum list|grouplist：查看yum源上可用安装包、包组
* yum info：查看软件信息
* yum search：yum源搜索软件
* yum install|groupinstall：安装单个软件、包组
* yum reinstall： 重装软件
* yum update|groupupdate： 软件更新【有参时更新个别，无参全部更新】
* yum remove|groupremove：移除单个软件、包组
* yum clean all：清除yum缓存
* yum makecache：生成yum缓存
* yum provides */command：查询包含命令command的软件包

## 软件源设置
>文件必须位于/etc/yum.repos.d下

```
[local]                --yum源标题
name=local          ---》yum源名称
baseurl=file:///media  ---》yum源路径
enable=1             引导文件起作用
gpgcheck=0          ---》不进行md5校验
```

# dpkg
* -i：安装deb软件
* -r：删除软件
* -P：删除软件和配置
* -l：显示软件列表

# apt
* update：更新软件
* upgrade：升级软件
* install：安装软件
* remove：删除已安装软件
* purge：删除软件和配置文件
* apt-cache search：在本地搜索软件【apt update更新后本地形成缓存】

## 软件源
设置/etc/apt/sources.list

# chkconfig
* --add：添加服务，Chkconfig确保每个运行级别都有一项启动（S）或者停止（K）入口。若有缺少，则会从缺省的init脚本中自动创建。
    - 范例：chkconfig --add httpd
* --del：删除服务，不再由chkconfig指令管理，并同时在系统启动的叙述文件中（`/etc/rc[0-6].d`）删除相关数据
    - 范例：chkconfig --del httpd
* --list：默认显示所有运行级别下所有服务的运行状态（on或off）；若指定了服务，则显示指定服务所有运行级别的运行状态
    - 范例：chkconfig --list mysqld
* --level：对指定运行级别下的指定服务进行操作（开启、关闭或初始化）；不加参数时，对于【on】或【off】命令，系统默认只对级别2,3,4,5进行操作；
    - 范例：chkconfig --level httpd 2345 on

# systemctl
## systemd添加服务
* 将服务控制文件放入/usr/lib/systemd/system/
* systemd服务重载配置：systemctl daemon-reload

## systemd管理
* 查看已经启动的服务：systemctl list-units --type=service
* 查看所有服务：systemctl list-units --type=service --all
* 开机启动：systemctl enable crond
* 关闭开启启动：systemctl disable crond
* 是否开机启动：systemctl is-enabled crond
* 服务状态：systemctl status crond
* 开启服务：systemctl start crond
* 关闭服务：systemctl stop crond
* 重启服务：systemctl restart crond

# 非编译二进制程序安装

* 下载可执行程序到工作目录

* 准备二进制程序启动所需配置文件

* 准备服务可systemd管理的文件/usr/lib/systemd/system/XXXXXX.service

  * 指定二进制文件路径
  * 指定程序主配置文件
  * 指定程序其他启动参数

* 启动服务并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start XXXXXX
  systemctl enable XXXXXX
  ```

# 程序源码安装

## 概念
* 开放源码：人可以看懂的文本类型的程序源码
* 编译程序：将程序源码编译成机器可以看懂的语言，类似翻译者的角色
* 可执行程序：开放源码经过编译生成的二进制文件，机器可以看懂并执行

## 安装前置条件
* 系统核心是否适合本软件
* 是否有编译程序【gcc、make、autoconfig等】
* 是否存在本软件所需要的函数库或其他依赖软件
* 是否存在核心的头文件（header include）

## 源码安装
* 解压：将压缩类型文件【如tar.gz等格式】展开成文本类型普通文件
* configure（配置编译参数）：测试系统环境是否满足软件安装的需求，满足后，根据某些用户的自定义项生成源码如何编译的规则文件makefile
* make：根据makefile的编译规则，使用源码、编译程序、依赖的库函数编译程序，生成二进制文件
* make install：将make产生的二进制文件和配置文件安装在自己的主机上

## 函数库

函数库依照是否被编译到程序内部分为动态库和静态库【查看可执行程序的动态库：ldd binary-file】

|                              静态库                              |                           动态库                           |
|------------------------------------------------------------------|------------------------------------------------------------|
| 扩展名通常为libxxx.a                                             | 扩展名通常为libxxx.a                                       |
| 此类函数库在编译时直接整合到可执行程序中【生成的可执行程序较大】 | 编译时仅将函数库的位置点编入程序，因此生成的可执行程序较小 |
| 可执行程序可独立执行                                             | 程序不能被独立执行，要确保函数库位置不变且必须存在         |
| 函数库升级后，所有包含次函数的可执行程序必须重新编译             | 函数库升级后，程序无需变更【函数名不变】                   |

