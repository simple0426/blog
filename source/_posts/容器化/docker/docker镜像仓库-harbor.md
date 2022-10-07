---
title: docker镜像仓库-harbor
date: 2018-03-20 16:39:21
tags: ['harbor']
categories: ['docker']
---
# harbor介绍
* 镜像的存储harbor使用的是官方的docker registry服务去完成；至于registry是用本地存储或者云存储都是可以的，harbor的功能是在此之上提供用户权限管理、镜像复制等功能
* harbor镜像的复制是通过docker registry 的API去拷贝

# [安装][6]
* 安装前置条件
    * 2核CPU/4GB内存/40GB磁盘
    * docker-engine 17.06.0-ce+ or higher
    * docker-compose 1.18.0 or higher
* [下载离线安装包][3]
* 参数配置(harbor.yml)
    - hostname: 对外发布的主机名
    - harbor_admin_password：web管理界面的密码【默认：admin/Harbor12345】
    - database：本地数据库密码
        + password: root123
    - data_volume: 镜像等数据的在宿主机上的位置，默认/data
    - log.location：日志在宿主机上的位置，默认/var/log/harbor
* hostname配置要点
    - 当配置hostname为内网地址时，虽然登陆地址可以通过地址映射使用外网地址；但是认证时，服务端回传客户端的认证地址，依然是内网地址造成无法登陆认证。
    - 当配置hostname为外网地址时，由于外网带宽限制，内网服务器不能高速的进行镜像传输
    - 当配置hostname为域名时，采用dns分离解析，内网dns解析使用自定义dns并解析为内网地址，外网dns解析使用公网dns并解析为公网ip。
* harbor安装
    - 包装命令：sudo ./install.sh
* web访问：http://ip:80

## [https设置](https://goharbor.io/docs/2.0.0/install-config/configure-https/)
> 参考官方配置，暂时不要使用cfssl工具（无法兼容X509 v3 extension特性）

* 使用公有证书或自签名证书
* harbor.yml中设置
    - hostname：设置域名或ip地址
    - certificate、private_key：设置证书和秘钥的路径
* 重新部署harbor
    - 关闭服务：docker-compose down
    - 重新部署：./install.sh
* 客户端设置
    - 如果使用公有证书，客户端无需设置
    - 如果使用自签名证书，将ca根证书放入如下位置：/etc/docker/certs.d/myregistry:5000/ca.crt
        + myregistry为harbor主机的ip地址或域名，如果使用非443端口则要添加端口信息
        + ca.crt为根证书且名称必须是这样

## 重新设置

```
docker-compose down
vim harbor.yml
prepare
docker-compose up -d
```

# harbor服务控制 
## 容器控制
* docker-compose up -d 创建和启动容器【后台】
* docker-compose down 停止和删除容器
* docker-compose start 启动容器及服务
* docker-compose stop 停止容器及服务

## 容器介绍
* harbor-core：配置管理中心
* harbor-db：pg数据库
* harbor-jobservice：负责镜像复制
* harbor-log：记录操作日志
* harbor-portal：web管理页面和API
* nginx：前端代理，负责前端页面和镜像上传、下载转发
* redis：会话
* registryctl：镜像存储

# harbor管理
## 用户管理
* 项目中的角色：
    - 访客(guest)：对项目只读
    - 开发者(developer)：对项目有读写权限
    - 维护者(master)：高于开发者的权限，同时拥有权限：扫描镜像、查看复制任务、删除镜像
    - 项目管理员（projectadmin）：用户新建项目时，默认具有的权限；项目管理员可以添加或删除项目成员
* 系统级别角色
    - 系统管理员（sysadmin）：具有最高权限
    - 匿名用户（anonymous）：未登录的用户；默认，只能对公开项目有读权限

# 客户端使用
* 【http模式】客户端添加不安全的私有镜像库地址(/etc/docker/daemon.json)：insecure-registries
* 添加客户端认证：docker login -u username -p password reg.mydomain.com
* 上传镜像
    - docker tag SOURCE_IMAGE[:TAG] harbor.zj-hf.cn/mytest/IMAGE[:TAG]
    - docker push harbor.zj-hf.cn/mytest/IMAGE[:TAG]
* 下载镜像：docker pull harbor.zj-hf.cn/mytest/registry:latest
* 删除镜像
    - web界面删除镜像
    - 设置垃圾清理（Garbage Collection）：删除文件系统上的数据

[3]: https://github.com/goharbor/harbor/releases
[6]: https://goharbor.io/docs/2.0.0/install-config/
