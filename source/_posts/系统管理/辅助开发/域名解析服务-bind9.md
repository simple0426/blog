---
title: 域名解析服务-bind9
date: 2019-09-09 10:12:52
tags: ['bind9']
categories: ['linux']
---
# 软件安装
`sudo apt-get install bind9 -y`
# 配置修改
* named.conf.options

```
      forwarders {
                202.106.196.115; //配置将不能解析的请求转发到哪[配置转发时，可不需配置根区域文件]
         };
         allow-query {0.0.0.0/0;}; //接收哪个网段下的解析请求
```

* named.conf.local

```
zone "example.com" {
        type master;
        file "/etc/bind/db.example.com";
};
```

* db.example.com

```
;此域的权威授权服务器为本机
@       IN      NS      localhost.
;解析example.com
          IN      A      1.1.1.1
;www.example.com解析设置
www   IN      A       1.1.1.1
test1   IN      A       10.10.10.244
;test2.example.com 与test1.example.com互为别名
test2   IN      CNAME   test1
;test3.example.com 与 www.abc.com互为别名
test3   IN      CNAME   www.abc.com.
;邮件服务器解析
@   IN  MX  10  mail.example.com.
mail IN  A  192.168.1.2
;基于域名解析的负载均衡
www2 IN A 1.1.1.2
www2 IN A 1.1.1.3
www2 IN A 1.1.1.4
;泛域名解析
* IN A 1.1.1.5
```

# 功能测试

* nslookup

```
>www.example.com
>set type=mx #测试此域名下的邮件服务
>example.com
```

* dig

`dig www.example.com`

# 非常用功能
* 反向解析
* 主从复制
    * 主配置
    ```
     zone "abc.com" IN {
        type master;
        file "abc.com.zone";
        allow-transfer { 192.168.0.98; }; //允许从机复制
    };
    ```
    * 从配置
    ```
    zone "abc.com" IN {
        type slave;
        file "abc.com.zone";
        masters { 192.168.0.99; };  //从哪个主机复制内容
    }; 
    ```

* 分离解析[内外网解析不同的结果]

```
view "lan" {
    match-clients { 192.168.0.0/24; };
    zone "abc.com" IN {
        type master;
        file "abc.com.lan";
    };
    zone "." IN {
        type hint;
        file "named.ca";
    };
};
view "wan" {
    match-clients { any; };
    zone "abc.com" IN {
        type master;
        file "abc.com.wan";
    };
};
```

* 子域委派【父域abc.com 不解析对子域da.abc.com的dns请求，将对子域的解析请求传给子域】

```
# zone区域配置
da  IN  NS  ds-dns.abc.com.
ds-dns IN  A  192.168.0.98
```
