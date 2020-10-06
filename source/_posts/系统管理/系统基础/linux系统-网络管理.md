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
    - -i eth0：指定监听的网络接口
    - tcp port 3306：监听本机tcp协议3306端口
    - icmp：监听icmp协议
    - ip host 192.168.100.2：监听指定来源的主机
    - -t：不显示时间戳
    - -n：补办ip解析成域名
    - -nn：不把端口转换成对应的协议
    - -c：只抓取多少行数据
    - dst：指定数据流向【到本机还是离开本机】
        + 范例：tcpdump -i eth1 -tnn dst port 80 -c 100
* ping：测试网络连通性
* traceroute：路由追踪
* nslookup：域名解析【ip与mac对应关系】
    - dns设置：/etc/resolv.conf 
* arp：地址解析协议
* [curl](#curl)
* [iptables](#iptables)
* [tcp_wrapper](#tcp-wrapper)
* netstat：显示网络链接
    - -a：显示所有链接状态
    - -n：以数字形式显示ip和端口
    - -u：显示udp链接
    - -t：显示tcp链接
    - -p：显示建立链接的进程信息【PID】
* [ifconfig](#ifconfig)：查看或临时设置网卡信息
* ethtool：查看网口信息
* [route](#route)：查看或临时设置路由信息
* iftop
* [wget](#wget)
* [nc](#nc)
* nmap

# ifconfig
>永久修改网卡信息：/etc/sysconfig/network-scripts/ifcfg-eth0

- ifconfig：查看处于开启状态的网卡信息
- ifconfig -a：查看所有状态的网卡信息【包含down状态】
- 设置网卡：
    + ifconfig eth0 192.168.1.1 netmask 255.255.255.0
    + ifconfig eth0 192.168.1.1/24
- 添加虚拟网卡：Ifconfig eth0:0 192.168.20.1/24
- 打开或关闭网卡：ifconfig eth0 up/down
    + 启用/禁用网络接口：ifup/ifdown eth0

# route
* 查看路由表：route –n 
* 添加默认网关：route add default gw 192.168.1.1 
* 删除默认网关：route del default gw 192.168.1.1 
* 添加默认路由(下一跳)：route add –net 10.0.0.0 netmask 255.0.0.0 gw 192.168.30.254 
* 添加默认路由(本地出口)：route add –net 10.0.0.0 netmask 255.0.0.0   dev eth0
* 添加主机路由(本地出口)：route add –host 11.22.22.3 dev eth0 
* 添加主机路由(下一跳)：route add –host 1.2.3.4 gw 192.168.30.254 
* 删除默认路由：route del –net 10.0.0.0 netmask 255.0.0.0 gw 192.168.30.254 
* 删除默认路由：route del –net 10.0.0.0 netmask 255.0.0.0 dev eth0

# curl
## 响应
* -o filename：下载文件并另存为
* -O 下载文件并保持文件名
* -L, --location 跟踪重定向
* -c --cookie-jar FILE将服务器发送的cookie保存到文件中
    - curl -c cookie.txt -s https://www.linuxidc.com
* -I 只显示header信息
    - curl -s -I http://www.baidu.com
* -s --silent静默输出
* -w 显示特定变量的信息【变量参考`man curl`】
    - curl -o /dev/null -s -w "%{http_code}" www.baidu.com

## 请求
* -X, --request 指定请求方法【默认get】

* -d/--data 指定post请求数据
    
    - curl -X POST --data '{"message": "01", "pageIndex": "1"}' http://127.0.0.1:5000/python
    
* -H/--header 指定要发送的header信息
    
    - curl -H "Content-Type: application/json" -X POST  --data '{"userID":10001}' http://localhost:8085/GetUserInfo
    
* -A 指定user-agent信息
    
    - curl -A "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)" -o page.html  https://www.linuxidc.com
    
* -b, --cookie STRING/FILE 指定发送请求时要发送的cookie文件或字符串
    
    - curl -b 'a=1;b=2' https://www.linuxidc.com
    
* -e, --referer指定上次访问的页面【可用于盗链】
    
    - curl -o test.jpg -e http://oldboy.blog.51cto.com/2561410/1009292 http://img1.51cto.com/attachment/201209/155625967.jpg
    
* -x, --proxy [protocol://]host[:port] 使用代理服务器：--proxy http://127.0.0.1:10809

* -g, --globoff 允许在URL中使用`[] {}`表示范围

    ```
    curl -g "localhost:9200/bank/_search?q=age:%20[22%20TO%2025]&pretty"
    ```

# nc
netcat是一个用于TCP/UDP连接和监听的工具,nc是开源版本的netcat工具
## 常用选项
* -l：开启监听模式
* -p：nc使用的源端口
* -s：发送数据使用的接口ip
* -n：不要使用DNS反向查询IP地址的域名
* -u：使用udp协议
* -w：设置超时时间
* -v：设置详情输出
* -z：扫描时不发送任何数据

## 主要功能
### 监听和连接
* 服务端：nc -l 1234
* udp服务端：nc -ul 2345
* 客户端(可用于测试服务端口是否开启)：nc 127.0.0.1 1234

### 数据传输
#### 文件传输
* 服务端：nc -l 1234 > filename.out
* 客户端：nc host.example.com 1234 < filename.in

#### 文本传输
* 与web服务器通信：printf "GET / HTTP/1.0\r\n\r\n" | nc host.example.com 80
* 与snmp服务器通信：

```
nc [-C] localhost 25 << EOF
HELO host.example.com
MAIL FROM:<user@host.example.com>
RCPT TO:<user2@host.example.com>
DATA
Body of email.
.
QUIT
EOF
```

### 端口扫描
* 范围扫描：nc -zv host.example.com 20-30
* 列表扫描：nc -zv host.example.com 80 20 22

# wget
非交互式网络下载器，wget支持https、http、ftp下载
## 选项
* -O：另存为其他文件
* -b：后台执行【查看下载进度：tail -f wget-log】
* -c：支持断点续传
* -i file：从文件中读取下载地址【一行一个】
* --limit-rate： 限速【1M=限速1MB/s】
* --spider：像spider一样只测试文件是否存在，而不是真正下载
* --user-agent=agent-string：设置浏览器代理
* --ftp-user/--ftp-password：ftp下载选项
* --http-user/--http-password：http下载选项
* --proxy-user=user/--proxy-password=password：proxy选项设置

## 范例
* wget -b -c --limit-rate=300K https://mirrors.aliyun.com/docker-toolbox/linux/compose/1.10.0/docker-compose-Linux-x86_64 -O docker-compose.box

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
# tcp-wrapper
## 简介
* [tcp_wrapper](https://www.cnblogs.com/long-cnblogs/p/10716091.html)是一个类似于iptables的实现访问控制工具；但是，iptables是工作在内核中的，所以只要是一个网络服务经由内核中的TCP/IP协议栈就能够受到netfilter的控制
* tcp_wrapper是基于库调用来实现其功能的。这就意味着只有那些在开发时调用了tcp_wrapper相关库（这个库叫libwrap）的应用，tcp_wrapper的控制才能生效。

## 应用受控判断
* 编译方式：一个程序在开发时如果调用了某个接口，它在编译时会有两种编译方式
    - 动态编译：表示基于动态链接库的方式使用库
    - 静态编译：表示把调用的库文件直接编译进应用程序内部
* 应用是否支持tcp_wrapper，两种编译方式的判断如下
    - 动态编译：使用ldd命令，显示出链接至libwrap，即表示支持tcp_wrapper功能；比如ssh服务：【ldd /usr/sbin/sshd】
    - 静态编译：使用strings命令，显示结果中出现两个文件（hosts.allow和hosts.deny）表示支持tcp_wrapper

## 配置文件
* /etc/hosts.allow：允许访问的设置
* /etc/hosts.deny：禁止访问的设置
    - 范例：【vsftpd:192.168.241.1】vsftpd服务不允许192.168.241.1访问
* 执行顺序：先检查hosts.allow，后检查hosts.deny【有相同条目时，allow生效】

## 配置语法
语法：daemon_list：client_list `：[option_list]`

- daemon_list：服务列表
    + 服务的二进制文件名【应用程序是通过二进制文件链接至libwrap库文件来完成访问控制的】
    + 多个服务间逗号分隔
- client_list：客户端地址
    + ip地址
    + 主机名：
        * LOCAL 代表本机
        * ALL 代表所有程序，所有ip
    + 网络地址
        * 格式1：172.16.0.0/255.255.0.0
        * 格式2：172.16.【最后那个点不能省略】
        * 不能是： ~~172.16.0.0/16~~
- option_list：可选参数列表
    + except：排除选项
        * 范例要求：这个网络开放给172.16访问但是这个172.16.100.1不能访问
        * 设置：【vsftpd:172.16. EXCEPT 172.16.100.1】
    + deny：表示拒绝，用于host.allow文件中，在允许的文件中实现拒绝的功能【vsftpd:192.168.241. :deny】
    + allow：表示允许，用于hosts.deny文件，实现allow的功能
    + spawn：额外启动其它应用程序，完成一部分的管理或其它功能
        * 范例要求：本地开放了ftp服务，一旦有人访问了，默认策略【allow或deny】执行后，将其记录到日志中
        * 设置：【vsftpd:ALL :spawn /bin/echo `date` login attemp from %c to %s,%d >> /var/log/vsftpd.deny.log】
        * 参数注解：%c表示client ip,客户端地址，%s表示server ip，服务器端地址，%d为daomon name即访问的是服务器上的哪个守护进程；
