---
title: django开发之views学习
tags:
categories:
---
# request请求内容
* request.method：请求方法
* request.GET：对收到的get请求进行参数解析
* request.POST：对收到的post请求进行参数解析
    - 仅当Content-Type：application/x-www-form-urlencoded
* request.Meta：请求的元数据信息
* request.body：请求体内容
    - get请求体为空
    - post请求体范例：
    `b'csrfmiddlewaretoken=SP3GO7LEfUb0QqWwE4Rq0r0W&user=he&passwd=12'`
