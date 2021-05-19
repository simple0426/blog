---
title: nginx基础学习
tags:
  - nginx
  - htpasswd
  - location
categories:
  - web
date: 2019-06-11 18:07:53
---

# 介绍
[淘宝nginx文档](http://tengine.taobao.org/nginx_docs/cn/docs/)

[官方nginx文档](http://nginx.org/en/docs/)

## 特点
* 占用资源少
* 支持高并发
* 可以做代理服务器【类似squid、haproxy】
* 可以做缓存服务器【类似varnish】

# 安装
## 下载
http://nginx.org/en/download.html
## 依赖安装
* ubuntu： apt-get install zlib1g-dev libpcre3 libpcre3-dev openssl libssl-dev -y 
* centos：yum install pcre pcre-devel zlib zlib-devel openssl openssl-devel -y

## 编译
```
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-pcre --with-http_ssl_module \
--with-http_gzip_static_module --with-http_flv_module --with-http_realip_module --with-debug
```

* 支持查看nginx连接状态
* 支持正则表达式【比如map功能即为使用正则】
* 支持ssl功能
* 支持gzip压缩
* 支持flv视频流
* 支持通过realip模块定位最终客户端地址【客户端走多层代理访问最终服务时需要】
* 开启debug功能

## 安装
make && make install 
# 命令参数
* -?,-h         : this help
* -v            : show version and exit
* -V            : show version and configure options then exit
* -t            : test configuration and exit
* -T            : test configuration, dump it and exit【显示配置文件详情】
* -q            : suppress non-error messages during configuration testing【检查配置文件语法时屏蔽非错误提示信息】
* -s signal     : send signal to a master process: stop, quit, reopen, reload
    - stop：强制关闭主进程
    - quit：优雅关闭煮即成
    - reopen：打开新的日志文件写入
    - reload：重新加载配置文件
* -p prefix     : set prefix path (default: /usr/local/nginx/)
* -c filename   : set configuration file (default: conf/nginx.conf)
* -g directives : set global directives out of configuration file【配置文件外设置全局命令，如：-g daemon on】

# 配置文件
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/nginx-conf.png)

## 全局设置
```
user nginx nginx
worker_processes  4;
worker_cpu_affinity 01 10 01 10;
pid logs/nginx.pid;
worker_rlimit_nofile 65535;
events {
    use epoll;
    worker_connections  65535;
}
```
```
worker_processes 4;
    指明了nginx要开启的进程数，据官方说法，一般开一个就够了，多开几个，可以减少机器io带来的影响。 一般为当前机器总cpu核心数的1到2倍
worker_cpu_affinity 01 10 01 10; 
    nginx默认是没有开启利用多核cpu的配置的。需要通过增加worker_cpu_affinity配置参数来充分利用多核cpu。
    为每个进程开启一个cpu，1表示开启核心，0为关闭
    2核2进程写法：
    worker_processes 2；
    worker_cpu_affinity 01 10;
    2核4进程写法：
    worker_processes  4;
    worker_cpu_affinity 01 10 01 10;
worker_rlimit_nofile 65535;
    指定一个nginx进程可以打开的最多文件描述符数目，（需要使用命令“ulimit -n 65535”来设置。）
worker_connections  65535;
    用于定义Nginx每个进程的最大连接数
```
## http设置
```
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
#    map $http_x_forwarded_for  $clientRealIp {
#            ""      $remote_addr;
#            ~^(?P<firstAddr>[0-9\.]+),?.*$  $firstAddr;
#    }
    lua_shared_dict limit 50m;  #防cc使用字典，大小50M
    lua_package_path "/home/zj-web/nginx/conf/waf/?.lua";
    init_by_lua_file "/home/zj-web/nginx/conf/waf/init.lua";
    access_by_lua_file "/home/zj-web/nginx/conf/waf/access.lua";
    access_log  logs/access.log  main;
    error_log   logs/error.log  error;
    # tcp conf
    include tcp.conf;
    # gzip conf
    include gzip.conf;
    # site conf
    include site-available/*.conf;
    # realip conf
    include realip.conf;
}
```
```
default_type  application/octet-stream;
    默认类型为二进制流，也就是当文件类型未定义时使用这种方式，例如：在没有配置PHP环境时，Nginx是不予解析的，此时，用浏览器访问PHP文件就会出现下载窗口。
log_format:
    日志格式
access_log/error_log:
    主日志信息【当location未定义时使用】
lua*：
    与lua脚本搭配构建WAF
```
## tcp设置
```
client_max_body_size  20m;
client_header_buffer_size    32K;
client_body_buffer_size 128k;
large_client_header_buffers  4 32k;
Sendfile  on;
tcp_nopush     on;
tcp_nodelay    on;
keepalive_timeout 60;
client_header_timeout  10;
client_body_timeout    10;
send_timeout          10;
```
```
client_max_body_size  20m;
    用来设置允许客户端请求的最大的单个文件字节数
client_header_buffer_size    32K;
    用于指定来自客户端请求头的header buffer大小
client_body_buffer_size   128k;
    指定客户端请求主体缓冲区大小
large_client_header_buffers  4 32k;
    用来指定客户端请求中较大的消息头的缓存最大数量和大小
sendfile  on;
    用于开启高效文件传输模式
tcp_nopush     on;
tcp_nodelay    on;
    将tcp_nopush和tcp_nodelay两个指令设置为on用于防止网络阻塞
keepalive_timeout 60;
    设置客户端连接保持活动的超时时间。在超过这个时间之后，服务器会关闭该连接
client_header_timeout  10;
    设置客户端请求头读取超时时间。如果超过这个时间，客户端还没有发送任何数据，Nginx将返回Request time out（408）错误
client_body_timeout    10;
    设置客户端请求主体读取超时时间。如果超过这个时间，客户端还没有发送任何数据，Nginx将返回Request time out（408）错误，默认值是60
send_timeout          10;
    指定响应客户端的超时时间。这个超时仅限于两个连接活动之间的时间，如果超过这个时间，客户端没有任何活动，Nginx将会关闭连接。
```
## gzip设置
```
# 编译参数--with-http_gzip_static_module
gzip  on;
gzip_min_length  1k;
gzip_proxied any;
gzip_buffers    16 64k;
gzip_http_version 1.1;
gzip_disable "MSIE [1-6].";
gzip_comp_level 5;
gzip_types       text/plain application/x-javascript text/css application/xml application/json application/javascript image/jpeg image/gif image/png;
gzip_vary on;
```
```
gzip on;                   #开启gzip模块，实时压缩输出数据流
gzip_min_length 1k;         #设置允许压缩的下限阀值【大于1k的文件才进行压缩】
gzip_buffers 4 16k;          #申请4个大小16K的内存作为压缩结果流缓存
gzip_http_version 1.1;       #支持的http协议
gzip_comp_level 5;          #压缩级别
gzip_types text/plain application/x-javascript text/css application/xml;   #支持的压缩类型
gzip_vary on;               #可以根据客户端是否支持压缩自动调整压缩功能
```
## ssl设置
```
# 编译参数--with-http_ssl_module
# 文件路径为相对于nginx.conf的路径
ssl on;
ssl_certificate   cert/214135355250268.pem;
ssl_certificate_key  cert/214135355250268.key;
ssl_session_timeout 5m;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
```
```
ssl自定义证书设置
1.系统安装openssl及开发库
2.使用openssl产生证书【pem格式】和私钥【pem格式】
openssl req -x509 -nodes -newkey rsa:2048 -keyout nginx.key -out nginx.crt
```
## php配置

> nginx默认没有配置SCRIPT_FILENAME

```
location ~ \.php$ {
    root  /var/www/html;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    include        fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
}
```

* root定义php文件的根目录
* SCRIPT_FILENAME，定义web访问时php文件路径【/var/www/html/index.php】
  * document_root即为本区域或父区域定义的文件根目录【/var/www/html】
  * fastcgi_script_name即为文件在根目录下的位置【/index.php】

## auth与status

```
# 编译参数：--with-http_stub_status_module
server {
    listen       80;
    server_name  0.0.0.0;
    location /status {
        stub_status on;              #开启模块功能
        access_log off;               #关闭访问日志
        allow 127.0.0.1;        #只允许127 访问
        deny all;
        auth_basic "status for nginx";                #验证登陆时的标题
        auth_basic_user_file htpasswd;   #使用授权文件htpasswd
    }
}
```
```
auth_basic是Nginx的一种认证机制。auth_basic_user_file用来指定认证的密码文件，
由于Nginx的auth_basic认证采用的是与Apache兼容的密码文件，
因此需要用Apache的htpasswd命令来生成密码文件，例如要添加一个webadmin用户，可以使用下面方式生成密码文件：
yum install httpd-tools -y
htpasswd -c htpasswd webadmin
此命令会在当前目录生产一个文件htpasswd，其中用户名webadmin
```
## [websocket支持](http://nginx.org/en/docs/http/websocket.html)
>nginx1.3.13之后，nginx内置支持websocket代理

```
location /chat/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

# server配置
## 配置
```
server {
    listen 80;
    server_name pre-zj-static.zj-hf.cn;
    return 301 https://$server_name$request_uri;
}
server {
    listen       443;
    server_name  pre-zj-static.zj-hf.cn;
    include ssl.conf;
    error_page  403 /error/403.jpg;
    error_page  404 /error/4041.html;
    error_page  500 502 503 504  /error/50x.jpg;
    location  /error/ {
        internal;
        root html;
    }
    location / {
        error_log  logs/vue-html.error.log  info;
        root wechat/vue;
        add_header Cache-Control 'no-store';
        index  index.html index.htm;
        try_files $uri $uri/ /index.html =404;
      }
     location ~\.html {
        error_log  logs/jq-html.error.log  info;
        root  wechat/jq;
        add_header Cache-Control 'no-store';
        allow all;
      }
     location ~* ^.+pdf.+\.(bcmap|css|json|js|properties|txt|html|swf|wav|png|jpg|woff|ttf)$ {
                error_log  logs/vue-assetst.error.log  info;
                root  wechat/jq;
                access_log   off;
                expires      30d;
            }
}
```

## 配置详解
* listen 80 default_server;         
    用于指定虚拟主机的服务端口，default_server可以在多个虚拟主机之间设置默认的虚拟主机。
* server_name www.abc.org abc.com; 
    用来指定IP地址或者域名，多个域名之间用空格分开,不含www的域名也可直接设置，如abc.com
* return 301 https://$server_name$request_uri;
    向客户端返回相应的状态码以及url
* root
    - 虚拟主机根目录，【定义的根目录与conf、sbin同级】
    - 示例：

```
location /request_path/image/ {
    root /local_path/image/;
}
当客户端请求 /request_path/image/cat.png 的时候， 
Nginx把请求映射为/local_path/image/request_path/image/cat.png
```

* alias：
    - 对请求的路径做替换转换
    - 示例

```
location /request_path/image/ {
    alias /local_path/image/;
}
当客户端请求 /request_path/image/cat.png 的时候， 
Nginx把请求映射为/local_path/image/cat.png 
```

* add_header
    - 重写发送给客户端的响应头【response】

* allow all
    - 对匹配的location作访问限制【allow或deny】
    - 可使用的匹配规则为：允许部分，拒绝所有；拒绝部分，允许所有。
* access_log   off;
    - 关闭访问日志
* expires      30d;
    - 设置静态文件缓存时间
* try_files $uri $uri/ /index.html =404;
    - 当location的程序不能处理请求时的处理方式【可以处理404等异常】
    - 按照指定的顺序检查文件或目录【名字末尾加反斜线】是否存在
    - 如果找不到任何文件，将按最后一个参数指定的uri进行内部跳转。
    - 最后一个参数可以是一个命名的location或状态码【404】等
* internal 只能由内部url调用，外部访问则404
* error_page  403 /error/403.jpg; 
    * 定义错误页面，位于server内
    * 错误页面大小必须大于512字节，否则会被浏览器的默认错误页面替代

# location优先级
## 语法定义
```
location [ = | ~ | ~* | ^~ ] uri { ... }
location @name { ... }
```

* `=` 精确匹配，如果找到匹配=好的内容，立即停止搜索，并立即处理请求（优先级最高）
* `~` 表示执行一个正则匹配，区分大小写
* `~*` 表示执行一个正则匹配，不区分大小写
* `^~` 只匹配字符串，不匹配正则表达式,一般用来匹配目录
* `@` 指定一个命名的location，一般只用于内部重定向请求【try_files、error_page】

## location示例
```
location指令：
#NO.1
        location / {
        return 500;
        }
#NO.2
        location /a/ {
        return 403;
        }
#NO.3
        location ^~ /a/ {
        return 403;
        }
#NO.4
        location /a/1.jpg {
        return 402;
        }
#NO.5
        location ~* \.jpg$ {
        return 401;
        }
#NO.6
        location = /a/1.jpg {
        return 400;
        }
```
```
优先级：
1）优先匹配#NO.6  “=”优先级最高
2）再次匹配#NO.5  正则表达式匹配
3）再次匹配#NO.4  完整匹配
4）再次匹配#NO.3  部分包含的被匹配【NO.2与NO.3效果相同】
5）最后匹配#NO.1  所有的访问都被匹配  
```

