---
title: apache下rewrite使用
date: 2019-05-07 17:36:02
tags: ['apache', 'rewrite']
categories: ['web']
---

# 功能
* rewrite url实现URL的跳转和隐藏真实地址，基于perl语言的正则表达式规范
* 主要用途：拟静态、拟目录、域名跳转、防盗链
* 开启功能
    - 去除httpd.conf文件中"#LoadModule rewrite_module modules/mod_rewrite.so"前面的"#"号; 
    - RewriteEngine on 

# 配置
* 基于整个apache的配置，即httpd.conf
* 基于虚拟主机的配置，即virtual host指令
* 基于目录的配置，即directory指令和 目录下的.htaccess文件

## htaccess文件
* .htaccess只有在用户访问目录时加载
* 子目录和父目录的htaccess文件处于同等地位，默认不会继承父作用域规则
* 默认情况下，mod_rewrite在合并属于同一上下文内容时会覆盖规则。
* htaccess开启rewrite功能
    - RewriteEngine On
    - Options FollowSymLinks
    - RewriteBase / 【从根开始改写url】

# 语法
* 设定符合条件的url：RewriteCond TestString CondPattern  
* 该写符合条件的url：RewriteRule Pattern Substitution [flags]  

# RewriteCond 
## TestString
<要匹配的[服务器变量][1]> + <正则匹配条件>
## CondPattern
>额外的标志位

1. 'nocase|NC' (no case) 忽略大小写
2. 'ornext|OR' (or next condition) 更改多个RewriteCond为逻辑“或”(默认为“与”)
3. 'novary|NV' (no vary)

# RewriteRule 
## Pattern 
>request_uri(不包含hostname、port、query_string的部分)

1. pattern在( VirtualHost)中指代request_uri
2. pattern在(Directory and .htaccess)中则指request_uri排除当前目录后的部分

## Substitution 
1. file-system-path：在virtualhost中配置的规则被当做文件系统路径
2. URL-path：request_uri
3. absolute URL：使用R标志重定向到一个新的url
4. -(dash)：保持url不变

## flags
1. NC：忽略大小写
2. R[=code]：url重定向
3. C：如果本条rule匹配则传递给下一条规则，如果本条不匹配则以后的rule都跳过
4. L：本条rule为最后一条规则，之后的rule都不再应用
5. QSA：在原始URL后追加查询字符串
6. F：返回403给客户端

# 范例
## 虚拟主机建立多个网站
* 只添加根目录htaccess文件，既能子域名访问网站，也可以通过首域名+同名目录访问网站
* 每添加一个域名及相应的同名目录，都需要在根目录和同名目录添加相关配置
* 阿里云虚拟主机页面上的301配置只能实现首页【域名级别】的301跳转，要实现域名间的全站跳转必须使用下方的配置

## 根目录htaccess
```
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
# auroraabrasive.com全站301跳转www.auroraabrasive.com
RewriteCond %{HTTP_HOST} ^auroraabrasive.com$ [NC] 
RewriteRule ^(.*)$ http://www.auroraabrasive.com/$1 [R=301,L]
# 首域名+目录访问 或 子域名访问网站
RewriteCond %{HTTP_HOST} ^www\.auroraabrasive\.com$ [NC]  
RewriteCond %{REQUEST_URI} !^/auroraabrasive/            
RewriteRule ^(.*)$ auroraabrasive/$1?Rewrite [L,QSA]
</IfModule>

# 部分注解
1. 如果请求的header信息中host为www.auroraabrasive.com
2. 如果请求的request_uri中不是以auroraabrasive开头的(斜杠为分界符)
3. 重写url-path为 auroraabrasive/%{URI}?Rewrite
```

## 子目录htaccess
```
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
# 只允许指定的域名访问
RewriteCond %{HTTP_HOST} !^www\.auroraabrasive\.com$ [NC] 
RewriteRule (.*) http://www.auroraabrasive.com/$1 [L,R=301]
# 同名目录访问
RewriteCond %{REQUEST_URI} ^\/auroraabrasive\/ [NC]
RewriteCond %{QUERY_STRING} !^(.*)?Rewrite 
RewriteRule ^(.*)$ /%{REQUEST_URI}/%{REQUEST_URI}/$1?Rewrite [L,QSA]  
</IfModule>

# 部分注解
1. 如果请求的header信息中host不是www.auroraabrasive.com
2. 重定向url为http://www.auroraabrasive.com/%{URI}，并且不再匹配其他规则
3. 如果请求的%{REQUEST_URI}是以auroraabrasive开头的
4. 如果请求的query_string不是以任意字符串开头后紧跟Rewrite
5. 改写url-path为/%{REQUEST_URI}/%{REQUEST_URI}/%{URI}?Rewrite
(由于rewritecond中使用了REQUEST_URI变量，此时url-path为全路径匹配，包含了本级目录auroraabrasive)
```

[1]: https://httpd.apache.org/docs/current/mod/mod_rewrite.html#rewritecond