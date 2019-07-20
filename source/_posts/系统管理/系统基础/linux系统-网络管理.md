---
title: linux系统-网络管理
tags:
  - curl
  - iptables
categories:
  - linux
date: 2019-07-17 17:18:07
---
# 命令列表
* tcpdump
* [curl](#curl)
* [iptables](#iptables)
* netstat
* ifconfig
* route
* ssh
* iftop
* wget
* nc
* socat

# curl
## 响应
* -o 下载文件并另存为
* -O 下载文件
* -c --cookie-jar FILE将服务器发送的cookie保存到文件中
    - curl -c cookie.txt -s https://www.linuxidc.com
* -I 只显示header信息
    - curl -s -I http://www.baidu.com
* -s --silent静默输出
* -w 显示特定变量的信息【变量参考`man curl`】
    - curl -o /dev/null -s -w "%{http_code}" www.baidu.com

## 请求
* -X 指定请求方法【默认get】
* -d 指定post请求数据
    - curl -X POST --data '{"message": "01", "pageIndex": "1"}' http://127.0.0.1:5000/python
* -A 指定user-agent信息
    - curl -A "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)" -o page.html  https://www.linuxidc.com
* -b, --cookie STRING/FILE 指定发送请求时要发送的cookie文件或字符串
    - curl -b 'a=1;b=2' https://www.linuxidc.com
* -e, --referer指定上次访问的页面【可用于盗链】
    - curl -o test.jpg -e http://oldboy.blog.51cto.com/2561410/1009292 http://img1.51cto.com/attachment/201209/155625967.jpg
* -x 使用代理服务器

# iptables
* iptables是一个用于ipv4/ipv6包过滤和地址转换的管理工具
* iptables语法主要由路由表、规则链、匹配规则、执行动作构成
* iptables语法：iptables (-t 路由表) ([规则操作](#规则操作) 规则链) ([功能参数](#功能参数)) ([匹配规则](#匹配规则))  (-j [执行动作](#执行动作)) 

## 表和链
iptables包含4个表和5个链，路由表是对数据包的不同类型的操作，规则链是在处理流程中的不同时间点进行的操作
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/iptables-tables-chains.gif)

* 路由表：优先级：raw > mangle > nat > filter
    * filter：一般的主机过滤功能，默认表
    * nat：一般用于网络地址转换（nat）
    * mangle：主要负责修改数据包中特殊的路由标记，如TTL、TOS、MARK等
    * raw，优先级最高，设置raw一般是为了不再让iptables做数据包的链接追踪处理，提高性能
* 规则链
    - PREROUTING：数据包进入路由表之前
    - INPUT：通过路由表后目的地为本机
    - FORWARD：通过路由表，目的地不是本机
    - OUTPUT：本计产生的数据包
    - POSTROUTING：发送到网卡接口前

## 规则操作
* -A，--append chain rule-spec：在链的末尾添加规则
* -D，--delete
    - -D chain rule-spec：删除特定的规则
    - -D chain rulenum：删除特定编号的规则
* -C，--check chain rule-spec：检查特定规则是否存在（与-D chain rule-spec处理逻辑相同）
* -I，--insert chain `[rulenum]` rule-spec：在链的指定规则编号前插入规则；如果没有指定规则编号，则在链的第一行插入规则
* -R，--replace chain rulenum rule-spec：使用规则替换指定编号的规则
* -L，--list `[chain]`：显示表下链的规则（不指定则显示所有链）
* -S，--list-rules `[chain]`：以和iptables-save一样的格式显示规则
* -F，--flush `[chain]`：删除链中的所有规则
* -Z，--zero `[chain [rulenum]]`：清空链中规则的数据包计数器
* -P，--policy chain target：更改链的默认处理规则【ACCEPT、DROP等】
* -N，--new-chain chain：添加用户自定义链
* -X，--delete-chain `[chain]` ：删除用户自定义链（不能被引用）；如果未定义链名，则删除所有用户自定义链
* -E，--rename-chain old-chain new-chain：重命名用户自定义的链

## 功能参数
* -v，--verbose：展示详情输出
* -n，--numeric：IP地址和端口以数字形式输出
* -x，--exact：在使用-L选项时，显示数字计数器的精确值
* --line-numbers：显示规则行号
* --modprobe=command：当添加或插入规则时，加载必要的模块【targets、match extensions】

## 匹配规则
`[!]`是对指定内容取反

* `[!]` -p，--protocol protocol：只对特定协议(如tcp、udp、icmp)的数据包处理，默认为all
* `[!]` -s，--sorce address`[/mask][,...]`：匹配源地址，可以是网段或主机ip地址，--src标志是这个选项的别名
* `[!]` -d, --destination address`[/mask][,...]`：匹配目标地址。--dst标志是这个选项的别名
* `[!]`-i，--in-interface name：接收数据包的网络接口【仅适用于INPUT, FORWARD、PREROUTING链】
* `[!]`-o，--out-interface name：将要发送数据包的网络接口【仅适用于FORWARD、OUTPUT、POSTROUTING链】
* -j，--jump target：可以跳转到用户自定义的链处理；或直接使用内置处理规则【ACCEPT、DROP】处理数据包
* -g，--got chain：转向到用户自定义的链处理
* -m, --match match：匹配特定属性的扩展规则模块

### 扩展匹配规则
* multiport：多端口匹配【这个模块可以一次性匹配多个源或目的端口，最多可以指定15个端口；地址范围(port:port)被当做是2个端口；只能跟在tcp、udp协议后。】
    - `[!] --source-ports,--sports port[,port|,port:port]`：源端口匹配
    - `[!] --destination-ports,--dports port[,port|,port:port]`：目的端口匹配
    - `[!] --ports port[,port|,port:port]`：端口匹配
    - 范例：iptables -I INPUT -p udp -m multiport --dports 100,180 -j DROP
* state：网络连接状态匹配【如ftp】
    - --state 连接状态：NEW(新连接)、ESTABLISHED(已连接)、RELATED(正在连接或已连接)、INVALID(未知连接)
    - 范例：iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
* tcp：tcp协议匹配，在-p tcp指定后即可使用
    - `[!] --source-port,--sport port[:port]`：源端口匹配，只接受一个端口或端口范围
    - `[!] --destination-port,--dport port[:port]`：目的端口匹配
    - `[!] --tcp-flags mask`：tcp连接状态匹配
    - 范例：iptables -A FORWARD -p tcp --tcp-flags SYN,ACK,FIN,RST SYN
* udp：udp协议匹配
    - `[!] --source-port,--sport port[:port]`：源端口匹配，只接受一个端口或端口范围
    - `[!] --destination-port,--dport port[:port]`：目的端口匹配

## 执行动作
* ACCEPT：允许数据包通过
* DROP：拒绝数据包通过
* REJECT：向匹配的数据包发送一个错误响应，仅适用于INPUT, FORWARD、OUTPUT链
* MASQUERADE：仅用于nat表的POSTROUTING链，仅用于出口为动态公网ip的nat连接
* SNAT：修改数据包的源地址；仅适用于nat表的INPUT、POSTROUTING链；仅适用于tcp、udp协议
    - `--to-source [ipaddr[-ipaddr]][:port[-port]]`：转换为特定源地址
* DNAT：修改数据包的目的地址；仅适用于nat表的 PREROUTING、OUTPUT链
    - `--to-destination [ipaddr[-ipaddr]][:port[-port]]`：转换为特定目的地址

## 范例
### filter与nat数据流
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/iptables-filter-nat.gif)

### 主机防火墙设置
```
iptables -F
iptables -X
iptables -Z
iptables -A INPUT -s 192.168.100.0/24 -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 3306,80 -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP
```
### 正向代理设置
>代理局域网主机访问公网

```
net.ipv4.ip_forward = 1
sysctl -p
# echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -P FORWARD ACCEPT
iptables -F -t nat
iptables -X -t nat 
iptables -Z -t nat 
modprobe ipt_stat             #开启iptables的状态记录模块
modprobe ip_conntrack_ftp     #开启iptables的ftp连接追踪模块
modprobe ip_nat_ftp       #开启iptables的ftp网络地址转换模块
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth1 -j MASQUERADE
# iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth1 -j SNAT --to-source 192.168.1.50
```

### 反向代理设置
>代理内网服务，以便公网可以访问

```
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 -j DNAT --to-destination 192.168.100.2:80
#iptables -t nat -A PREROUTING -d 192.168.1.50 -p tcp --dport 80 -j DNAT --to-destination 192.168.100.2:80
```
