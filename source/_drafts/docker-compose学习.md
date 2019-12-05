---
title: docker-compose学习
tags:
categories:
---
# 概念
* services：与docker中的容器操作对应
* networks：与docker中的网络操作对应
* volumes：与docker中的数据卷操作对应

# docker-compose.yml文件语法
>默认连接在同一个自定义网络下的多个service（service即容器）可以通过service name实现容器间互连

## services定义
```
version: "3.7"
services:
  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
```

* build：等同于docker build命令，可以直接包含Dockerfile目录位置，也可以包含更复杂的数据结构
* image：定义要使用的image，或在build时给image命名
* depends_on：定义各服务间的依赖关系，比如启动、关闭的先后顺序；依赖的服务会先于此服务启动，后于此服务关闭。
* environment：定义环境变量，可以是列表(- SHOW=true)或字典(SHOW: 'true')
* ports：将容器端口映射到主机
    - 同时定义主机和容器端口：host_port:container_port
    - 只定义容器端口：此时将容器端口发布到主机随机端口上
* networks：定义要使用的网络
* volumes：将主机的的路径或命名的数据卷和容器的路径建立映射关系

## volumes定义
```
volumes:
  db-data:
```
只定义一个db-data的数据卷，没有其他选项

## networks定义
```
networks:
  frontend:
  backend:
```

可选参数

* driver：定义网络类型，在单机情况下默认是bridge，在Swarm中是overlay
    - bridge
    - swarm
    - host
    - none

# docker-compose命令
## 安装
* sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
* sudo chmod +x /usr/local/bin/docker-compose

## 命令参数
* up [-d]
* down
* start
* stop
