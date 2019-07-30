---
title: linux常用命令
tags:
categories:
---
# 用户与权限
* chattr、chmod、chown
* useradd、userdel、usermod
* passwd
* setfacl、getfacl
* group、gpasswd、groups
* sudo、su

# 系统管理
* chkconfig
* [crontab](#crontab)：周期性任务
* at：一次性任务（使用较少）
* date
* head、tail
* echo：回显文本
    - -e：支持特殊转义符：【\n \t等】
    - -n：移除末尾换行
* [read](#read)：从标准输入读取一行内容
* seq `[start [step]]` end：打印数字序列
    - -f printf格式
    - -s 分隔符【默认\n】
    - -w 列表前加0使宽度相等
* yum、apt、rpm、dpkg
* man
* xargs：把管道前面的处理结果按列表交给后面的命令处理
* reboot、init、shutdown、poweroff、halt
* history：查看命令历史【-c 清除命令历史】
* which、locate
* time：执行时间统计
* watch：周期性的执行程序，并同时全屏显示输出
* alias、unalias：命令别名
* ulimit：控制shell终端可使用的资源

# 系统状态
* whoami：当前登录用户
* users：所有登录用户
* w、who：查看所有用户登录详情
* id
* last：查看系统最近登录情况
* lastlog：显示最近登录的用户名，登录端口及时间
* lastb：登录失败信息
* dmesg：系统启动过程
* uname
* free
* uptime
* top、htop

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
