---
title: docker容器和镜像
tags:
  - 容器
  - 镜像
categories:
  - 虚拟化
date: 2019-11-11 22:14:02
---

# 容器管理
* start、stop：启动、停止一个容器
* create、rm：创建、删除容器
* pause/unpause：挂起、恢复容器内的所有进程
* port：端口映射查看
* logs [-f]：日志查看，-f相当于tailf效果
* exec：在容器中执行命令：docker exec -it 5c8f798e206e /bin/bash
* ps：查看运行中的容器
    - 参数
        + -a：查看所有存在的容器
        + -q：只显示容器id
        + -f：根据条件过滤容器
    - 批量操作范例：docker rm $(docker ps -qf "status=exited")
* inspect：查看容器或镜像信息
* stats：查看容器资源使用情况
* top：查看进程列表

# 容器运行-run
## 常用参数
>docker run -idt -v /home/Logs/:/root/Logs:ro -m 100m --memory-swap=100m --cpus 0.5 -p 10086:22 sshd

* -t：分配一个伪终端tty
* -i：保持容器的标准输入一直打开
* -d：让容器以后台方式运行
* -v /home/Logs/:/root/Logs:ro（可多次使用）：映射宿主机/home/Logs到容器/root/Logs并设置为只读
* -m 100m --memory-swap=100m：设置容器内存为100m、内存和swap总和也是100m
* --cpus 0.5：设置容器可以使用的cpu百分比（最大值为cpu的核心数，最小值为0.1，可以是小数）
* -p 10086:22（可多次使用）：映射宿主机端口10086到容器端口22
    - publish：将容器端口映射到宿主机【命令行参数-p或-P】
    - expose：曝露端口用于容器间的访问【Dockerfile文件中的关键字EXPOSE】

## 其他参数
* --cidfile：把容器id写入一个文件
* --name：给运行的容器绑定一个名称
* --rm：容器存在就删除【运行一次性容器，运行后立即删除】
* -h：配置容器主机名
* --dns：配置容器dns
* -P：将主机的49000~49900的端口随机映射到内部容器的开放端口
* --link name:alias：链接容器【容器名称：别名】，这种链接关系可以在容器hosts文件看到

# volume管理
## 使用原因
* 原因：docker存储引擎device mapper、aufs等实现的copy on write机制，在高频写操作下性能不高
* 优势：使用volume，可以直接写磁盘；可以实现目录持久化；可以实现宿主机与容器、容器之间的数据共享

## 使用方式
* 映射宿主机目录到容器：◦docker run -idt --name test1 -v ~/nginx:/webapps ubuntu /bin/bash
* 数据卷使用
    - 建立数据卷：docker volume create hello
    - 其他容器使用数据卷：docker run -d -v hello:/world busybox ls /world
* 卷容器使用
    - 建立卷容器（容器无需一直运行）：◦docker run -d -v ~/dbdata:/dbdata --name dbdata ubuntu echo data-only
    - 其他容器挂载使用：◦docker run -idt --volumes-from dbdata --name db2 ubuntu /bin/bash

# 镜像管理
* docker search [image]：查询registry中的镜像
* docker pull [image]：从registry获取镜像
* docker images：查看本地镜像列表
* docker rmi [image]：删除本地镜像
* docker history [image]：查看镜像制作历史(分层文件系统)
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

# [Dockerfile](https://docs.docker.com/engine/reference/builder/)
## 语法
* FROM image：使用的基础镜像
    - FROM scratch【scratch是一个基础的空镜像】
* LABEL key=value：标签信息
    - maintainer="istyle.simple@gmail.com"
    - version="1.0"
    - description="This is simple"
* ENV key value：设置环境变量，它会被后续RUN命令使用，并在容器运行时保持
* WORKDIR /path：为后续的RUN、CMD指定工作目录(建议使用绝对目录)，没有则创建
* CMD command param1 param2：在创建镜像的过程中执行的命令
    - 有shell、exec两种执行方式，示例为shell方式
    - Dockerfile中可以写多条RUN指令，但是为了避免一次RUN增加一层文件层，应当将多条命令合并到一行中执行
        + 使用&&连接多个命令
        + 过长的行使用反斜线续行
* CMD command param1 param2：指定容器启动时执行的命令
    - 有shell、exec两种执行方式，示例为shell方式
    - 如果docker run指定了其他命令，CMD命令被忽略
    - 定义了多个命令时，只有最后一条被执行
* ENTRYPOINT ['excutable', 'param1', 'param2']：容器启动后运行的服务
    - 有shell、exec两种执行方式，示例为exec方式
    - 让容器以应用程序或者服务的形式运行，只可添加一条
    - 此命令默认不被docker run提供命令的覆盖，但可通过[--entrypoint]覆盖
* COPY src dest：复制本地主机的src到容器的dest，
* ADD src dest：
    - add和copy功能类似(复制指定的src到容器的dest)，优先使用copy
    - 额外功能：解压src到dest
    - 其中src可以是Dockerfile所在目录的相对路径，也可以是一个url
* EXPOSE port：告诉宿主机容器暴露的端口
* VOLUME ["/data"]：创建一个可从本地主机或其他容器挂载的挂载点
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
