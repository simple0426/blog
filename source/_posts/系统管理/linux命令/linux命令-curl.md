---
title: linux命令-curl
tags:
  - curl
categories:
  - linux
date: 2019-05-06 16:38:25
---

# 命令参数
## 下载
* -o 下载文件并另存为
* -O 下载文件

## 请求
* -X 指定请求方法【默认get】
* -d 指定post请求数据
    - curl -X POST --data '{"message": "01", "pageIndex": "1"}' http://127.0.0.1:5000/python
* -A 指定user-agent信息
    - curl -A "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)" -o page.html  https://www.linuxidc.com
* -b 指定发送请求时要发送的cookie文件
* -e 指定上次访问的页面【可用于盗链】
    - curl -o test.jpg -e http://oldboy.blog.51cto.com/2561410/1009292 http://img1.51cto.com/attachment/201209/155625967.jpg
* -x 使用代理服务器

## 输出
* -I 只显示header信息
    - curl -s -I http://www.baidu.com
* -s --silent静默输出
* -w 显示特定变量的信息
    - curl -o /dev/null -s -w "%{http_code}" www.baidu.com

