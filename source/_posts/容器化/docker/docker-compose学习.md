---
title: docker-compose学习
tags:
  - docker-compose
categories:
  - docker
date: 2019-12-10 00:39:53
---

# 介绍
docker compose是一个用于定义和管理多容器的docker工具，  
可以通过使用yaml格式的compose文件定义多个服务应用（这些应用也就是容器），
并通过compose命令管理这些服务的创建、运行、停止、销毁等工作。
# 核心概念
* services：它定义了一个应用包含的多个服务(服务的载体即是容器)
    - 容器可以从docker hub的image创建
    - 也可以从本地的Dockerfile build出来的镜像来创建
* networks：应用包含的服务(即容器)使用的网络
* volumes：应用包含的服务(即容器)使用的存储

# compose文件
* 默认使用docker-compose.yml作为文件名，命令行下可使用-f参数指定特定compose文件
* 默认连接在同一个自定义网络下的多个service（service即容器）可以通过service name实现容器间互连

```
version: '3'
services:
  wordpress:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD: root
    networks:
      - my-bridge
    depends_on:
      - mysql
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - my-bridge
volumes:
  mysql-data:
networks:
  my-bridge:
    driver: bridge
```
## services定义
* build：构建启动服务所需要的镜像，等同于docker build命令，，
    - 可以直接包含Dockerfile目录位置：【build .】
    - 也可以包含更复杂的数据结构：
    ```
    build:
      context: .
      dockerfile: Dockerfile
    ```
* image：定义要使用的image，或在build时给image命名
* depends_on：定义各服务间的依赖关系，比如启动、关闭的先后顺序；依赖的服务会先于此服务启动，后于此服务关闭。
* environment：定义环境变量，可以是列表(- SHOW=true)或字典(SHOW: 'true')
* ports：将主机端口绑定到容器暴露的端口
    - 同时定义主机和容器端口：host_port:container_port
    - 只定义容器端口：此时将容器端口发布到主机随机端口上
* networks：定义要使用的网络
* volumes：将主机的的路径或命名的数据卷和容器的路径建立映射关系

## volumes定义
见如上示例
## networks定义
可选参数【driver】定义网络类型，在单机情况下默认是bridge，在Swarm中是overlay

- bridge
- swarm
- host
- none

# docker-compose命令
## 安装
* 下载：

  ```
  # 官方
  curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  
  # daocloud
  curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
  ```

* 权限：chmod +x /usr/local/bin/docker-compose

## 命令参数
>docker-compose命令默认使用当前目录下的docker-com.yml文件，可以使用-f参数指定要使用的文件

* up [-d]：构建并启动一个服务下的多个容器，-d参数以后台方式执行
    - --scale service=num：增容或兼容容器数量
* down：停止并销毁一个服务下的多个容器及使用的网络，但是不会销毁数据卷
* start：启动服务(多个容器)
* stop：停止服务(多个容器)
* ps：显示正在运行的服务
* exec [service]：进入service中执行命令

# 弹性伸缩容器数量
```
version: "3"
services:
  redis:
    image: redis
  web:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      REDIS_HOST: redis
  lb:
    image: dockercloud/haproxy
    links:
      - web
    ports:
      - 8080:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock 
```

1. 启动服务：docker-compose up
2. 扩容web服务： docker-compose up --scale web=3 -d
