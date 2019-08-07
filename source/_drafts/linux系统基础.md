---
title: linux系统基础
tags:
categories:
---
# 环境变量加载
* 全局：/etc/profile
* 用户bashrc：`~/.bashrc`
* 用户bash_profile：`~/.bash_profile`

# 系统时间
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
