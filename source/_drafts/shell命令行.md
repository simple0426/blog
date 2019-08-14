---
title: shell命令行
tags:
categories:
---
# 系统管理
* [crontab](#crontab)：周期性任务
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
* watch：周期性的执行程序，并同时全屏显示输出
* alias、unalias：命令别名
* ulimit：控制shell终端可使用的资源
* whatis：显示命令含义
* logger：向syslog写入特定信息
    - -i 在每行都记录进程ID
    - -t logger_test 每行记录都加上“logger_test”这个标签
    - -p local3.notice 设置记录的设备和级别
    - 范例：echo "this is message"|logger -it logger_test -p local3.notice

# 登录状态
* whoami：当前登录用户
* users：所有登录用户
* w、who：查看所有用户登录详情
* last：查看系统最近登录情况
* lastlog：显示最近登录的用户名，登录端口及时间
* lastb：登录失败信息
* dmesg：系统启动过程
* uname：显示系统信息

# 软件管理
* chkconfig
* yum、apt、rpm、dpkg

# crontab
>周期性执行程序或命令

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
