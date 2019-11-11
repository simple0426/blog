---
title: docker镜像
tags:
  - 镜像
categories:
  - 虚拟化
date: 2019-11-11 22:17:20
---

# 相关命令
* 查询registry中的镜像：docker search registry
* 从registry获取镜像：docker pull centos:7
* 查看本地镜像列表：docker images
* 删除本地镜像：docker rmi ubuntu
* 导出tar包
    - 从容器导出tar包：docker export adb102099609 > adb102099609.tar
    - 从镜像导出tar包：docker save -o ubuntu-ruby.tar ubuntu:ruby 
* 导入tar包创建镜像
    - load方式：docker load < ubuntu-ruby.tar 
    - import方式：cat adb102099609.tar |sudo docker import - ubuntu:temp

# 镜像制作
## 创建Dockerfile方式
* 编写Dockerfile
* build命令构建：【docker build -t="ubuntu:ruby" .】
    - -t 仓库名:标签
    - . 注明dockerfile的文件位置

## 修改容器方式
* 劣势：被称为黑箱镜像，由于不确定镜像的具体操作，不利于镜像的传播使用
* commit命令实现：docker commit [-m "added vim" -a "simple"] 377e333c817b ubuntu:vim
    - -m：注释
    - -a：提交者信息
    - 容器id
    - repository:tag

## 本地导入方式
* openvz方式
    - 模板下载：https://download.openvz.org/template/precreated/
    - 导入镜像：sudo cat ubuntu-15.04-x86_64-minimal.tar.gz |docker import - local/ubuntu:15.04
* debootrap方式
    - 下载rootfs：sudo debootstrap --arch amd64 trusty ubuntu14
    - 导入镜像：sudo tar -C ubuntu14 -c .|sudo docker import - ubuntu:14.04.03

# Dockerfile
## 语法
* FROM image：使用的基础镜像
* MAINTAINER name：维护者信息
* RUN command 或 RUN ["executable", "param1", "param2"]
    - 在创建镜像的过程中执行的命令
    - Dockerfile中可以写多条RUN指令
    - 前者在终端中执行，后者使用exec执行
* CMD ['executable', 'param1', 'param2']：指定容器启动时执行的命令，使用exec执行，只可添加一条
* ENTRYPOINT ['excutable', 'param1', 'param2']：
    - 运行容器时执行的指令，只可添加一条
    - 此命令不可被docker run提供的覆盖，但可通过【--entrypoint="..."】覆盖
* EXPOSE port：告诉宿主机容器暴露的端口
* ENV key value：设置环境变量，它会被后续RUN命令使用，并在容器运行时保持
* COPY src dest：复制本地主机的src到容器的dest
* ADD src dest：
    - 该指令将复制指定的src到容器的dest【或者解压src到dest】
    - 其中src可以是Dockerfile所在目录的相对路径，也可以是一个url
* VOLUME ["/data"]：创建一个可从本地主机或其他容器挂载的挂载点
* USER daemon：创建容器运行时的用户名或UID
* WORKDIR /path：为后续的RUN、CMD指定工作目录
* ONBULID [instruction]：当所创建的镜像作为其他镜像的基础镜像时，所执行的命令

## 范例
* Dockerfile
```
FROM ubuntu
MAINTAINER hejingqi@zj-inv.cn
COPY sources.list /etc/apt/sources.list
RUN apt-get update
RUN apt-get install -y openssh-server supervisor vim iputils-ping net-tools lrzsz
RUN mkdir -p /var/run/sshd
COPY sshd_config /etc/ssh/sshd_config
RUN mkdir -p /var/log/supervisor
RUN echo "root:zjht4321"|chpasswd
COPY supervisord.conf /etc/supervisor/supervisord.conf
EXPOSE 22
COPY Shanghai /etc/localtime
RUN echo "Asia/Shanghai" > /etc/timezone
ENV LANG C.UTF-8
RUN echo "export LANG=C.UTF-8" >> /etc/profile
CMD /usr/bin/supervisord -c /etc/supervisor/supervisord.conf
```
* supervisord.conf
```
[supervisord]
nodaemon=true
[program:sshd]
command=/usr/sbin/sshd -D
autostart=true
autorestart=true
```
