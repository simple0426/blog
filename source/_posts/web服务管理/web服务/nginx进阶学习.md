---
title: nginx进阶学习
tags:
  - nginx
  - proxy
  - lua
  - waf
  - 升级
  - rewrite
categories:
  - web
date: 2019-06-11 18:08:11
---
# 参考文档
[淘宝nginx文档](http://tengine.taobao.org/nginx_docs/cn/docs/)
# proxy设置
## proxy相关变量
* $remote_addr：客户端地址【距离服务器最近的客户端ip，有代理时为代理的ip】
* $http_x_real_ip：
    - 在没有特殊配置情况下，X-Real-IP请求头不会自动添加到请求头中；
    - 这个请求头一般为代理服务器设置，形如：proxy_set_header X-Real-IP $remote_addr;
    - 如果经过两次代理【客户端-》cdn-》waf-》nginx】，且都设置proxy_set_header X-Real-IP $remote_addr，则在nginx服务层获取的http_x_real_ip为cdn的ip地址
* $http_x_forwarded_for：
    - 在没有特殊配置情况下，X-Forwarded-For请求头不会自动添加到请求头中；
    - 代理服务器中的设置：proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
* $proxy_add_x_forwarded_for：
    - 如果请求头中没有X-Forwarded-For则$proxy_add_x_forwarded_for为$remote_addr
    - 代理服务器中的设置：proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    - 此变量是把请求头中的X-Forwarded-For与$remote_addr用逗号合起来，每经过一个反向代理就在请求头X-Forwarded-For后追加反向代理IP，形如：real client ip, proxy ip 1。。。proxy ip N

## proxy设置
*  upstream参数

```
# http下，与server同级
    upstream  server {
        server   192.168.100.1 weight=1 max_fails=2 fail_timeout=30s;
        server   192.168.100.3 weight=1 max_fails=2 fail_timeout=30s;
        }
```

* proxy参数
    - proxy_redirect：是否对后端服务器的“Location”响应头和“Refresh”响应头,默认为default，即使用代理服务器的location替换后端服务器的location；off为关闭替换
    - 

```
# http, server, location
proxy_redirect off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_connect_timeout 60;
proxy_send_timeout 60;
proxy_read_timeout 60;
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 4 32k;
proxy_busy_buffers_size 64k;
proxy_temp_file_write_size 64k;
```
```
proxy_redirect
    是否对后端服务器的“Location”响应头和“Refresh”响应头进行该写,默认为default，即使用代理服务器的location替换后端服务器的location；off为关闭替换
proxy_set_header             设置后，可以向后端服务器传递指定的header信息，默认的只有下面的header被定义
    proxy_set_header Host $proxy_host;
    proxy_set_header Connection close;
proxy_connect_timeout
    表示与后端服务器连接的超时时间，即发起握手等候响应的超时时间；这个超时一般不可能大于75秒
proxy_send_timeout
    定义向后端服务器传输请求的超时。此超时是指相邻两次写操作之间的最长时间间隔，而不是整个请求传输完成的最长时间。如果后端服务器在超时时间段内没有接收到任何数据，连接将被关闭。 
proxy_read_timeout
    定义从后端服务器读取响应的超时。此超时是指相邻两次读操作之间的最长时间间隔，而不是整个响应传输完成的最长时间。如果后端服务器在超时时间段内没有传输任何数据，连接将被关闭。 
proxy_buffering on;
    代理的时候，开启或关闭缓冲后端服务器的响应。默认为on；
    当开启缓冲时，nginx尽可能快地从被代理的服务器接收响应，再将它存入proxy_buffer_size和proxy_buffers指令设置的缓冲区中。如果响应无法整个纳入内存，那么其中一部分将存入磁盘上的临时文件。proxy_max_temp_file_size和proxy_temp_file_write_size指令可以控制临时文件的写入。 
    当关闭缓冲时，收到响应后，nginx立即将其同步传给客户端。nginx不会尝试从被代理的服务器读取整个请求，而是将proxy_buffer_size指令设定的大小作为一次读取的最大长度。 
proxy_buffer_size  【该缓冲区大小默认等于proxy_buffers指令设置的一块缓冲区的大小，但它也可以被设置得更小】
    nginx从被代理的服务器读取响应时，使用该缓冲区保存响应的开始部分，即header信息。
proxy_buffers  【每块缓冲区默认等于一个内存页的大小。这个值是4K还是8K，取决于平台】
    为每个连接设置缓冲区的数量为number，每块缓冲区的大小为size。这些缓冲区用于保存从被代理的服务器读取的响应。
proxy_busy_buffers_size     【默认是proxy_buffer_size和proxy_buffers指令设置单块缓冲大小的两倍】
    当开启缓冲响应的功能以后，在没有读到全部响应的情况下，写缓冲到达一定大小时，nginx一定会向客户端发送响应，直到缓冲小于此值。这条指令用来设置此值。 同时，剩余的缓冲区可以用于接收响应，如果需要，一部分内容将缓冲到临时文件。该大小
proxy_temp_file_write_size   【默认值是proxy_buffer_size指令和proxy_buffers指令定义的每块缓冲区大小的两倍】
    在开启缓冲后端服务器响应到临时文件的功能后，设置nginx每次写数据到临时文件的size(大小)限制。   
proxy_max_temp_file_size  【默认值1024m，设置0时禁止响应写入临时文件】
    打开响应缓冲以后，如果整个响应不能存放在proxy_buffer_size和proxy_buffers指令设置的缓冲区内，部分响应可以存放在临时文件中。 这条指令可以设置临时文件的最大容量。 
```

* proxy_set_header Host $host解析
    - Host的含义是表明请求的主机名
    - $http_host为请求头中的Host信息，可能为空
    - $host可以在多个地方取值，但主要为代理服务器端接受请求的域名【如外网域名】
    - $proxy_host主要为被代理的主机ip或域名【一般为内网域名或ip】

*  使用

```
# location
proxy_pass http://server;
```

## realip设置
```
# 编译参数--with-http_realip_module
# 参数位置：http, server, location
# waf：
set_real_ip_from 121.43.18.0/24;
set_real_ip_from 120.25.115.0/24;
set_real_ip_from 101.200.106.0/24;
set_real_ip_from 120.55.177.0/24;
set_real_ip_from 120.27.173.0/24;
set_real_ip_from 120.55.107.0/24;
set_real_ip_from 118.178.15.0/24;
set_real_ip_from 123.57.117.0/24;
set_real_ip_from 120.76.16.0/24;
set_real_ip_from 182.92.253.32/27;
set_real_ip_from 60.205.193.64/27;
set_real_ip_from 60.205.193.96/27;
set_real_ip_from 120.78.44.128/26;
set_real_ip_from 118.178.15.224/27;
# cdn：
set_real_ip_from 140.205.127.0/25;
set_real_ip_from 140.205.253.128/25;
set_real_ip_from 139.196.128.128/25;
set_real_ip_from 101.200.101.0/25;
real_ip_header    X-Forwarded-For;
real_ip_recursive on;
```
```
set_real_ip_from 121.43.18.0/24; 
    设置代理服务器地址
real_ip_header X-Forwarded-For; 
    设置读取真实ip的地址源header
real_ip_recursive   on; 
    使用realip算法递归获取真实ip
```

## proxy实验数据
### 无代理模式
* $remote_addr：114.242.249.63
* $http_x_real_ip：-
* $http_x_forwarded_for：-
* $proxy_add_x_forwarded_for：114.242.249.63

### cdn+waf模式
客户端-》cdn-》waf-》nginx

* $proxy_add_x_forwarded_for：210.12.208.226; 101.200.101.36; 123.57.117.131
* $http_x_forwarded_for：210.12.208.226; 101.200.101.36
* $http_x_real_ip：101.200.101.36
* $remote_addr：123.57.117.131

### waf模式
客户端-》waf-》nginx

* $proxy_add_x_forwarded_for：114.242.249.63; 123.57.117.131
* $http_x_forwarded_for：114.242.249.63
* $http_x_real_ip：114.242.249.63
* $remote_addr：123.57.117.131

### cdn模式
客户端-》cdn-》nginx

* $proxy_add_x_forwarded_for：114.242.249.63; 101.200.101.42
* $http_x_forwarded_for：114.242.249.63
* $http_x_real_ip：- 
* $remote_addr：101.200.101.42


# 在线升级
## nginx相关信号
>可以通过向nginx主进程发送相关信号控制nginx

* TERM、INT：快速关闭nginx
* QUIT：优雅关闭nginx【待处理完请求才关闭】
* HUP：变更配置文件时，新的worker进程使用新配置文件，老的worker使用旧配置文件
* USR1：打开一个新的日志文件
* USR2：升级nginx二进制文件【使用新的二进制文件启动进程】
* WINCH：优雅关闭worker进程

## 编译新的二进制文件
只需configure和make，不需要make install
## 二进制文件替换 
* 备份旧的nginx二进制文件
* 复制新的nginx二进制文件：、/bin/cp obj/nginx

## 配置文件语法检测
nginx -t
## 给nginx发送信号
* 发送USR2信号使用新的二进制文件启动进程：sudo kill -USR2 4563
* 发送WINCH信号逐步停止旧的实例，让旧二进制文件启动的进程从容关闭：sudo kill -WINCH 4563

>一段时间后，旧的工作进程处理了所有已连接的请求后退出，就仅由新的工作进程来处理输入的请求了

## 后续处理
### 当新进程有问题时
* 发送 HUP 信号给旧的主进程 - 它将在不重载配置文件的情况下启动它的工作进程
* 发送 QUIT 信号给新的主进程，要求其从容关闭其工作进程
* 如果新主进程还没退出，发送 TERM 信号给新的主进程，迫使其退出
* 如果因为某些原因新的工作进程不能退出，向其发送 KILL 信号（9）

### 如果升级成功时
发送 QUIT 信号给旧的主进程使其退出而只留下新的服务器运行

# 常用变量
* $arg_PARAMETER： 这个变量值为GET请求中变量名PARAMETER参数的值。
* $args：这个变量等于GET请求中的参数。例如，foo=123&bar=blahblah;这个变量只可以被修改
    - $query_string 与$args相同
    - 范例：foo=123&bar=blahblah
* $uri：指的是请求的文件和路径，不包含”?”或者”#”之类的东西
    - 范例：/docs/2.2/zh-cn/mod/mod_rewrite.html
* $request_uri：则指的是请求的整个字符串，包含了后面请求的东西
    - 范例：/search?q=apache+rewrite
* $request：请求行信息【请求方法  请求url  使用的http协议】
* $remote_user：通过基本授权【auth_basic】进行登录时的用户名
* $time_local：处理请求时的系统时间
* $status：返回给客户端的状态码
* $body_bytes_sent：返回给客户端的数据大小
* $http_referer：这个请求的referer信息【由哪个url跳转进来的请求】
* $http_user_agent：客户端浏览器信息
* $host 取用顺序：
    -  请求行中的主机名
    -  来自请求头中【Host】信息
    -  与该请求所匹配的server name【nginx配置】。

## apache变量
*  QUERY_STRING：get请求中的参数
    -  范例：foo=123&bar=blahblah
*  REQUEST_URI：请求的资源信息【请求的文件和路径，不含查询字符串】
    -  范例：/docs/2.2/zh-cn/mod/mod_rewrite.html

# [rewrite][nginx-rewrite]指令
## 语法
* 语法：rewrite regex replacement [flag]；
* 上下文：server，location，if
* 含义
    - 如果指定的正则表达式能匹配URI，此URI将被replacement参数定义的字符串改写。
    - rewrite指令按其在配置文件中出现的顺序执行。
    - flag可以终止后续指令的执行。
    - 如果replacement的字符串以“http://”或“https://”开头，nginx将结束执行过程，并返回给客户端一个重定向。 

## flag
* last 停止这一轮的rewrite指令，然后查找匹配改变后URI的新location
* break 停止这一轮的rewrite指令
* redirect 在replacement字符串中为出现http://或https://开头时，返回302临时重定向
* permanent 返回301永久重定向

## 范例
### 网站改版
>由于旧的url被搜索引擎收录，所以要实现：旧的url经过301跳转到新的url后，依然可以访问同一资源内容【网页内容一致】

```
location / {
    if ($uri = "/overviewSystem.html"){
            rewrite .* "/winTheTustomer.html" permanent;
    }
    root /data/font/www;
    index  index.html index.htm;
}
location /dynamic/ {
    if ($uri = "/dynamic/list/1.html"){
        rewrite .* "/articleList/dynamic.html" permanent;
    }
    rewrite ^/dynamic(.*)$ $1 permanent;
}
```
```
1. 访问www.abc.com/overviewSystem.html重定向到www.abc.com/winTheTustomer.html
2. 访问www.abc.com/dynamic/list/1.html重定向到www.abc.com/articleList/dynamic.html
3. 访问www.abc.com/dynamic[url]重定向到www.abc.com[url]
```
### user_agent判断
```
if ($http_user_agent ~ MSIE) {
rewrite ^(.*)$ /images/$1 break;}
if ($http_user_agent ~* "android|iphone")
{ proxy_pass http://192.168.100.3;}
```
### 防盗链
```
location ~* \.(jpg|gif|png|swf|flv|wma|wmv|asf|mp3|mmf|zip|rar)$ {  
valid_referers none blocked *.ixdba1.net ixdba1.net;
if ($invalid_referer) {
rewrite ^/ http://www.ixdba.net/img/error.gif;
}  
valid_referers 定义需要处理防盗链的文件(如jpg等)，ixdba1.net表示这个请求可以正常访问上面指定的文件资源。
if{}中的内容的意思是：如果地址不是上面指定的地址就跳转到通过rewrite指定的地址
```
### 首页301跳转
```
rewrite ^(.*)$  https://$host$1 permanent;【与下面return301效果相同】
# return 301 https://$server_name$request_uri;
```

## 注意
* 如果replacement字符串包括新的请求参数，以往的请求参数会添加到新参数后面。如果不希望这样，在replacement字符串末尾加一个问号“？”，就可以避免，比如：`rewrite ^/users/(.*)$ /show?user=$1? last;`
* 如果正则表达式中包含字符“}”或者“;”，整个表达式应该被包含在单引号或双引号的引用中。

# 与lua集成设置
## 依赖软件下载
* [nginx][1]
* [luajit][2]：Lua即时编译器
* [lua-cjson][3]：lua语言cjson库
* [ngx_devel_kit][4]：是一个拓展nginx服务器核心功能的模块，第三方模块开发可以基于它来快速实现。
* [lua-nginx-module][5]：可在 Nginx 中嵌入 Lua 语言，让 Nginx 可以支持 Lua 强大的语法。
* [redis2-nginx-module][6]：是一个支持 Redis 2.0 协议的 Nginx upstream 模块，它可以让 Nginx 以非阻塞方式直接防问远方的 Redis 服务，同时支持 TCP 协议和 Unix Domain Socket 模式，并且可以启用强大的 Redis 连接池功能。
* [set-misc-nginx-module][7]：是标准的HttpRewriteModule指令的扩展，提供更多的功能，如URI转义与非转义、JSON引述，Hexadecimal、MD5、SHA1、Base32、Base64编码与解码、随机数等等
* [echo-nginx-module][8]：是一个 Nginx 模块，提供直接在 Nginx 配置使用包括 "echo", "sleep", "time" 等指令。

## lua语言环境安装
1. lujit安装：
    * make && make install
2. cjson安装：
    * 修改Makefile【LUA_INCLUDE_DIR =   $(PREFIX)/include/luajit-2.0】
    * make && make install
3. 链接库文件
    * sudo ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2
    *  sudo ldconfig

## nginx编译参数

```
--with-http_addition_module --add-module=../ngx_devel_kit --add-module=../lua-nginx-module \
--add-module=../echo-nginx-module --add-module=../redis2-nginx-module \
--add-module=../set-misc-nginx-module
```

## lua使用
### [waf][waf]功能简介(config.lua)
```
 --waf status    
 config_waf_enable = "on"   #是否开启配置
 --log dir 
 config_log_dir = "/tmp/waf_logs"    #日志记录地址
 --rule setting 
 config_rule_dir = "/usr/local/nginx/conf/waf/rule-config"         #匹配规则所放地址
 --enable/disable white url 
 config_white_url_check = "on"  #是否开启url检测
 --enable/disable white ip 
 config_white_ip_check = "on"   #是否开启IP白名单检测
 --enable/disable block ip 
 config_black_ip_check = "on"   #是否开启ip黑名单检测
 --enable/disable url filtering 
 config_url_check = "on"      #是否开启url过滤
 --enalbe/disable url args filtering 
 config_url_args_check = "on"   #是否开启参数检测
 --enable/disable user agent filtering 
 config_user_agent_check = "on"  #是否开启ua检测
 --enable/disable cookie deny filtering 
 config_cookie_check = "on"    #是否开启cookie检测
 --enable/disable cc filtering 
 config_cc_check = "on"   #是否开启防cc攻击
 --cc rate the xxx of xxx seconds 
 config_cc_rate = "10/60"   #允许一个ip60秒内只能访问10次
 --enable/disable post filtering 
 config_post_check = "on"   #是否开启post检测
 --config waf output redirect/html 
 config_waf_output = "html"  #action一个html页面，也可以选择跳转
 --if config_waf_output ,setting url 
 config_waf_redirect_url = "http://www.baidu.com" 
 config_output_html=[[  #下面是html的内容
 <html> 
 <head> 
 <meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 
 <meta http-equiv="Content-Language" content="zh-cn" /> 
 <title>网站防火墙</title> 
 </head> 
 <body> 
 <h1 align="center"> # 您的行为已违反本网站相关规定，注意操作规范。详情请联微信公众号：chuck-blog。 
 </body> 
 </html> 
 ]] 
```
### nginx设置
```
http {
    map $http_x_forwarded_for  $clientRealIp {
            ""      $remote_addr;
            ~^(?P<firstAddr>[0-9\.]+),?.*$  $firstAddr;
    }
    lua_shared_dict limit 50m;  #防cc使用字典，大小50M
    lua_package_path "/home/zj-ops/nginx/conf/waf/?.lua";
    init_by_lua_file "/home/zj-ops/nginx/conf/waf/init.lua";
    access_by_lua_file "/home/zj-ops/nginx/conf/waf/access.lua";
    server {
        location /lua {
            echo "Hello lua";
        }
        location /lua_test {
            content_by_lua 'ngx.say("Hello Lua! Simple.")';
        }
    }
```
### 修改lua脚本
config.lua：config_rule_dir = "/home/zj-ops/nginx/conf/waf/rule-config"
### 功能测试
1. 防sql注入测试：curl http://127.0.0.1/q.sql
2. cc防护测试：ab -c 100 -n 100  http://127.0.0.1/

## lua使用redis
[lua-resty-redis][lua-redis]提供一个lua语言版的redis API，使用socket（lua sock）和redis通信。
### nginx配置
```
    lua_package_path "/home/zj-ops/nginx/conf/lua-resty-redis/lib/?.lua;";
            location =/content_by_lua_block {
            default_type 'text/plain';
            content_by_lua_block {
                ngx.say('Hello: content_by_lua_block')
            }
        }
        location /lua_redis_basic{
            default_type 'text/html';
            lua_code_cache off;
            content_by_lua_file /home/zj-ops/nginx/conf/self_lua/test_redis_basic.lua;
            #access_by_lua_file /home/zj-ops/nginx/conf/self_lua/access_flow_control.lua;
            content_by_lua 'ngx.say("Hello Lua! Simple.")';
        }
```
### redis功能测试
```
local function close_redis(redis_instance)
    if not redis_instance then
        return
    end
    local ok,err = redis_instance:close();
    if not ok then
        ngx.say("close redis error : ",err);
    end
end

local redis = require("resty.redis");
--local redis = require "redis"
-- 创建一个redis对象实例。在失败，返回nil和描述错误的字符串的情况下
local redis_instance = redis:new();
--设置后续操作的超时（以毫秒为单位）保护，包括connect方法
redis_instance:set_timeout(1000)
--建立连接
local ip = '127.0.0.1'
local port = 6379
--尝试连接到redis服务器正在侦听的远程主机和端口
local ok,err = redis_instance:connect(ip,port)
if not ok then
    ngx.say("connect redis error : ",err)
    return close_redis(redis_instance);
end

--Redis身份验证
local auth,err = redis_instance:auth("foobared");
if not auth then
    ngx.say("failed to authenticate : ",err)
end

--调用API进行处理
local resp,err = redis_instance:set("msg","hello world")
if not resp then
    ngx.say("set msg error : ",err)
    return close_redis(redis_instance)
end

--调用API获取数据  
local resp, err = redis_instance:get("msg")  
if not resp then  
    ngx.say("get msg error : ", err)  
    return close_redis(redis_instance)  
end 

--得到的数据为空处理  
if resp == ngx.null then  
    resp = 'this is not redis_data'  --比如默认值  
end  
ngx.say("msg : ", resp)  
  
close_redis(redis_instance)

# 测试：http://127.0.0.1/lua_redis_basic 
返回：【msg : hello world】说明redis可用
```

### lua使用redis实现cc防护
```
-- access_by_lua_file '/opt/ops/lua/access_limit.lua'
local function close_redis(red)
    if not red then
        return
    end
    --释放连接(连接池实现)
    local pool_max_idle_time = 10000 --毫秒
    local pool_size = 100 --连接池大小
    local ok, err = red:set_keepalive(pool_max_idle_time, pool_size)

    if not ok then
        ngx_log(ngx_ERR, "set redis keepalive error : ", err)
    end
end

local redis = require "resty.redis"
local red = redis:new()
red:set_timeout(1000)
local ip = "127.0.0.1"
local port = 6379
local ok, err = red:connect(ip,port)
local auth, err = red:auth("foobared")
if not ok or not auth then
    return close_redis(red)
end

local clientIP = ngx.req.get_headers()["X-Real-IP"]
if clientIP == nil then
   clientIP = ngx.req.get_headers()["x_forwarded_for"]
end
if clientIP == nil then
   clientIP = ngx.var.remote_addr
end

local incrKey = "user:"..clientIP..":freq"
local blockKey = "user:"..clientIP..":block"

local is_block,err = red:get(blockKey) -- check if ip is blocked
if tonumber(is_block) == 1 then
   ngx.exit(ngx.HTTP_FORBIDDEN)
   return close_redis(red)
end

res, err = red:incr(incrKey)

if res == 1 then
   res, err = red:expire(incrKey,1)
end

-- 1秒超过10次视为非法ip，进入黑名单
if res > 10 then
    res, err = red:set(blockKey,1)
    -- 每个黑名单ip封禁60秒
    res, err = red:expire(blockKey,60)
end

close_redis(red)
```

---

# 其他功能
* [nginx限制连接模块-limit](https://blog.51cto.com/storysky/642970)
* [NGINX 结合 lua 动态修改upstream](https://blog.csdn.net/force_eagle/article/details/51966333)
    - 模块代码：https://github.com/openresty/lua-upstream-nginx-module#set_peer_down
* [Nginx+Tomcat+SSL 识别 https还是http](https://blog.csdn.net/woshizhangliang999/article/details/51861998)
* [高并发之系统限流](https://blog.csdn.net/lzw_2006/article/details/51768935)

# 参考
1. [Nginx + Lua + Redis 已安装成功(非openresty 方式安装)](http://www.cnblogs.com/tinywan/p/6534151.html)
2. [nginx + lua + redis 防刷和限流](http://blog.csdn.net/fenglvming/article/details/51996406)
3. [nginx 不中断服务 平滑升级][nginx-upgrade]


[1]:http://nginx.org/download/nginx-1.12.0.tar.gz
[2]: http://luajit.org/download/LuaJIT-2.0.5.tar.gz
[3]: https://www.kyne.com.au/~mark/software/download/lua-cjson-2.1.0.tar.gz
[4]: https://github.com/simpl/ngx_devel_kit.git
[5]: https://github.com/openresty/lua-nginx-module.git
[6]: https://github.com/openresty/redis2-nginx-module.git
[7]: https://github.com/openresty/set-misc-nginx-module.git
[8]: https://github.com/openresty/echo-nginx-module.git
[nginx-upgrade]: http://blog.csdn.net/u011085172/article/details/71515745
[waf]: https://github.com/unixhot/waf.git
[lua-redis]: https://github.com/openresty/lua-resty-redis.git
[nginx-rewrite]: http://tengine.taobao.org/nginx_docs/cn/docs/http/ngx_http_rewrite_module.html#rewrite




