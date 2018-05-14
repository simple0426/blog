---
title: mysql管理
tags: ['mysql']
categories: ['DataBase']
---
# 认证
## 授权
* 授权：`grant all on *.* to 'kingold'@'%' identified by 'zjht_kingod012';`
* 刷新权限：flush privileges;

## 变更密码
`update mysql.user set authentication_string=password('zjht098_kingold') where user='kingold' and host='%';`

## 本地免交互登陆
```
# 文件~/.my.cnf设置
[client]
user='kingold'
password='zjht098_kingold'
```
