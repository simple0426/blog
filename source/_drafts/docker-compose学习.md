---
title: docker-compose学习
tags:
categories:
---
# 概念
* services
* networks
* volumes

# docker-compose.yml语法
>默认连接在同一个自定义网络下的多个service（service即容器）可以通过service name实现容器间互连

# docker-compose
## 安装
* sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
* sudo chmod +x /usr/local/bin/docker-compose

## 命令参数
* up [-d]
* down
* start
* stop
