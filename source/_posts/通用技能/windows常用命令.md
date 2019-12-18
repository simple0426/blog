---
title: windows常用命令
tags:
  - windows
categories:
  - skill
date: 2019-05-15 09:25:35
---

# 启停windows服务
* 停止：net stop gitblit
* 启动：net start gitblit

# sleep实现
发起30个ping包耗时30s：ping -n 30 127.0.0.1 > null

# grep实现
>findstr

* netstat -ant|findstr :80$

# 进程命令
* 进程列表：tasklist /?
* 关闭进程：taskkill /?
    - 关闭nginx：taskkill /F /IM nginx.exe > nul

# 变量设置
* 设置变量：set key=value
* 删除变量（空值）：set key=
* 查看变量值：set key
