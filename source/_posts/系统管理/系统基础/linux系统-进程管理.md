---
title: linux系统-进程管理
tags:
  - kill
  - ps
  - nohup
  - lsof
categories:
  - linux
date: 2019-08-06 11:48:27
---

# 命令目录
* [kill](#kill)：向进程发送信号
* pkill、killall、pgrep：根据进程名称过滤进程id或杀死进程【可能有误、不建议使用】
* [ps](#ps)
* fuser：显示打开文件或socket的进程
* [nohup](#nohup)：运行一个免于挂起的命令，同时输出到非tty终端
* [jobs、fg](#任务模式)
* [lsof](#lsof)：进程打开的文件

# 进程信号
* 1(HUP)： 重启
* 2(INT) ：中断【ctrl+c】
* 3(QUIT)： 退出
* 9(KILL) ：强制终止
* 15(TERM) ：正常结束【kill默认信号】
* 17(STOP) ：停止程序运行【ctrl+z】
* 19(CONT)： 停止的程序继续运行

# kill
* 终止进程：kill pid
* 强制终止：kill -9 pid
* 重启进程：kill -HUP pid

# lsof
## 简介
list open file  列出打开的文件，文件可以是普通文件、目录、链接、字符设备、块设备、FIFO设备、socket
## 选项
>没有任何选项时，lsof默认显示所有活跃进程打开的文件

* `-a`：指明所有选项之间逻辑“与”关系【默认“或”关系】
* -c `^` c：进程执行的命令(不)是以c开头的，c可以是正则表达式【以`//`进行包围】
* -r t：周期性的显示输出信息【t定义时间间隔，默认15s】
* -u `^` uid：(不)属于某个登录名或id的用户，多个用户之间逗号分隔
* -p `^` pid：(不)属于某个进程的
* -i 地址：选择与internet地址匹配的文件列表：
    - 地址：`[46][protocol][@hostname|hostaddr][:service|port]`
    - protocol：传输层协议tcp/udp
    - hostname、hostaddr：主机名、ip地址
    - service：应用层协议(如smtp)、端口号
    - 范例：lsof -i tcp@ops:ssh
* -t：只输出符合要求的进程pid
* -X：不关联TCP、UDP的文件
    - 范例：lsof -p 22929 -X
* -s `^` `[p:s]`：(不)属于某个tcp、udp协议状态(如established)的
    - 范例：lsof -a -i tcp -s tcp:established -c nginx【查看nginx并发连接】
* --：表示选项的的结束，在文件名中包含“-”时有用
* file-names：列出打开文件的进程

# 任务模式
* cmd&：将命令以”任务“方式放入后台执行
* jobs：查看后台运行的任务
* fg number：将后台任务调入前台执行

# nohup
* 由于cmd&方式产生的进程父id为tty或pts终端shell，当用户注销或网络断开时，终端会收到HUP（hangup）信号从而关闭其所有子进程
* nohup不接收HUP信号，同时输出内容到非tty终端，因此可以使用【nohup cmd &】方式将命令放入后台长期运行

# ps
* 功能：显示当前的进程状态【whatis ps】
* 用法：【man ps】、【ps --help all】
* 常用组合
  - [ps -ef](#ef格式详解)
  - [ps -aux](#aux格式详解)

## 常用选项
* -A：显示所有进程
* -a：显示所有终端下的进程
* -r：显示正在运行的进程
* -x：与a搭配使用，显示较完整的进程信息
* -h：不显示header信息
* -e：命令后显示环境变量
* -f：全格式
* -l：长格式

## ef格式详解
* UID：用户ID       
* PID：进程ID
* PPID：父进程ID
* C：进程占用CPU的百分比
* STIME：进程启动的时间
* TTY：该进程在哪个终端上运行，若与终端无关，则显示？若为pts/0等，则表示由网络连接主机进程         
* TIME：该进程实际使用CPU运行的时间
* CMD：命令的名称和参数

## aux格式详解
* USER：用户名
* PID：进程ID
* %CPU：cpu使用率
* %MEM：内存使用率
* VSZ：该进程使用的虚拟内存量（KB）
* RSS：该进程占用的固定内存量（KB）
* TTY：该进程在哪个终端上运行，若与终端无关，则显示？若为pts/0等，则表示由网络连接主机进程
* STAT：进程状态
* START：进程开始的时间
* TIME：进程实际占用cpu的时间
* COMMAND：进程使用的命令

## 进程状态
| 状态码 |                    状态                    |                                含义                               |
|--------|--------------------------------------------|-------------------------------------------------------------------|
| R      | runnable（on run queue）运行               | 正在运行或在运行队列中等待                                        |
| S      | sleeping休眠中                             | 休眠中，受阻，在等待某个条件的形成或接受到信号                    |
| D      | uninterruptible sleep (usually IO)不可中断 | 收到信号不唤醒和不可运行，进程必须等待直到有中断发生              |
| T      | stopped停止                                | 进程收到SIGSTOP、SIGSTP、SIGTIN、SIGTOU信号后停止运行             |
| Z      | zombie僵死                                 | 进程已终止，但进程描述符存在，直到父进程调用wait4()系统调用后释放 |
| s      | 进程下有子进程                             | -                                                                 |
| <      | 优先级高的进程                             | -                                                                 |
| N      | 优先级低的进程                             | -                                                                 |
| +      | 位于后台的进程                             | -                                                                 |
| l      | 多线程                                     | -                                                                 |

