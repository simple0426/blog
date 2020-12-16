---
title: docker容器和镜像
tags:
  - 容器
  - 镜像
categories:
  - docker
date: 2019-11-11 22:14:02
---

# 容器运行(run)
## 范例
docker run -idt -v /home/Logs/:/root/Logs:ro -m 100m --memory-swap=100m --cpus 0.5 -p 10086:22 sshd
## 运行参数
* -t：分配一个伪终端tty

* -i：保持容器的标准输入一直打开，常与-t结合使用

* -d：让容器以后台方式运行

* -e key=value：在容器内设置环境变量

* --name：给运行的容器绑定一个名称

* -h：配置容器主机名

* --rm：容器存在就删除【运行一次性容器，运行后立即删除】

* -v ：映射宿主机目录或数据卷到容器

* -m 100m：设置容器内存为100m

* --memory-swap=100m：设置容器内存和swap总和是100m

* --oom-kill-disable：关闭系统oom，避免系统内存不足时杀死容器

* --cpus 0.5：设置容器可以使用的cpu百分比（最大值为cpu的核心数，最小值为0.1，可以是小数）
  
    - --cpu-shares int：设置多个容器使用cpu的相对权重
    
* -p 10086:22（可多次使用）：映射宿主机端口10086到容器端口22
    - publish：将容器端口映射到宿主机【命令行参数-p或-P】
    - expose：曝露端口用于容器间的访问【Dockerfile文件中的关键字EXPOSE】
    
* -P：将主机的49000~49900的端口随机映射到内部容器的开放端口

* --link name:alias：链接容器【容器名称：别名】，这种链接关系可以在容器hosts文件看到

* --network：容器链接到特定网桥

* --restart：docker daemon重启后container的重启策略，默认no

    ```
    * no       不重启，默认策略
    * on-failure[:max-retries]  异常退出时重启容器，包括开机启动
    * unless-stopped 一直重启容器，正常停止的容器除外
    * always   一直重启容器，包括开机启动
    ```

## 最佳实践
* 操作系统镜像，没有默认启动命令，需要设置-it及-d，保持容器运行

* 应用程序镜像，只需设置-d，就可保持容器运行

* 设置时区和时间

  ```
  -e TZ=Asia/Shanghai -v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime
  ```

# 容器管理
* start、stop、restart：启动、停止、重启一个容器

* create、rm：创建、删除容器

* container prune：移除所有停止的容器

* logs [-f]：日志查看，-f相当于tailf效果

* cp：在宿主机和容器直接复制文件或目录

* exec：在容器中执行命令
    - 进入bash环境：docker exec -it 5c8f798e206e /bin/bash
    - 查看ip地址：docker exec -it flask ip a
    
* ps：查看运行中的容器
    - 参数
        + -a：查看所有存在的容器
        + -q：只显示容器id
        + -f：根据条件过滤容器
        + -l：最新创建的容器
    - 批量操作范例：docker rm $(docker ps -qf "status=exited")
    
* inspect：查看docker对象信息

* container update：修改容器run参数

    ```
    docker container update --restart=on-failure jenkins
    ```

* stats：查看容器资源使用情况

* top：查看容器进程列表

* commit：将当前运行的容器制作为镜像【被称为黑箱镜像，由于不确定镜像的具体操作，不利于镜像的传播使用】

# volume管理
## 使用原因
* docker存储引擎device mapper、aufs等实现的copy on write机制，在高频写操作下性能不高；使用volume，可以直接写磁盘
* 可以实现数据持久化
    - 启动时需要初始化数据，例如配置文件
    - 运行过程中产生的业务数据、日志
* 可以实现宿主机与容器、容器之间的数据共享

## 使用方式
* 映射宿主机目录到容器（bind Mounts）
    - 作用：可以实现宿主机和容器相关目录内容的同步更新，方便开发环境的实时调试
    - 范例：docker run -idt --name test1 -v ~/nginx:/webapps ubuntu /bin/bash
* 数据卷使用（data volume）
    - 作用：此时建立的数据卷不随容器的销毁而消失
    - 建立数据卷：docker volume create hello
    - 查看数据卷列表：docker volume ls
    - 使用数据卷(使用未创建的数据卷时docker会自动创建)：docker run -d -v hello:/world busybox ls /world
    - 删除数据卷：docker volume rm hello

# 镜像管理
* build：构建镜像
* history：查看镜像制作历史(分层文件系统)
* inspect：查看docker对象信息
* rmi：删除本地镜像
* search：查询docker公共仓库中的镜像
* pull：从registry获取镜像
* push：推送镜像到镜像仓库
    * 登录仓库：docker login --username=perfect_0426@qq.com registry.cn-hangzhou.aliyuncs.com
    * 本地打包：docker build -t registry.cn-hangzhou.aliyuncs.com/simple00426/flask_test:latest .
    * 推送镜像到仓库：docker push registry.cn-hangzhou.aliyuncs.com/simple00426/flask_test:latest
* tag：给镜像添加一个标签
* image prune [-a]：移除没有标签的镜像、移除没有使用及没有标签的镜像(-a)
* images：查看本地镜像列表
    - 根据label过滤：docker images -f 'label=maintainer=istyle.simple@gmail.com'
    - 根据镜像名称过滤：`docker images -f reference='p*/*:latest'`【用户/镜像:标签】
    - 根据是否有tag过滤(删除不完整的image)：docker images -qf "dangling=true"
    - 根据某个镜像的前后时间：docker images -qf "before=portainer/portainer"
        + before：早于此镜像
        + since：此镜像之后
* 导出tar包
    - 从容器导出tar包：docker export adb102099609 > adb102099609.tar
    - 从镜像导出tar包：docker save -o ubuntu-ruby.tar ubuntu:ruby 
* 导入tar包创建镜像
    - load方式：docker load < ubuntu-ruby.tar 
    - import方式：cat adb102099609.tar |sudo docker import - ubuntu:temp

# 镜像制作
## 镜像分类
* 基础镜像，如centos、ubuntu
* 环境镜像，基于基础镜像构建，如java、php
* 项目镜像，基于环境镜像构建，是最终的交付物 

## [Dockerfile](https://docs.docker.com/engine/reference/builder/)指令
* FROM image：使用的基础镜像
    - FROM scratch【scratch是一个基础的空镜像】
* LABEL key=value：标签信息
    - maintainer="istyle.simple@gmail.com"
    - version="1.0"
    - description="This is simple"
* ENV key value：设置环境变量，它会被后续RUN命令使用，并在容器运行时保持
* WORKDIR /path：为后续的RUN、CMD指定工作目录(建议使用绝对目录)，没有则创建
* RUN command param1 param2：在创建镜像的过程中执行的命令
    - 有shell、exec两种执行方式，示例为shell方式
    - Dockerfile中可以写多条RUN指令，但是为了避免一次RUN增加一层文件层，应当将多条命令合并到一行中执行
* CMD command param1 param2：指定容器启动时执行的命令
    - 有shell、exec两种执行方式，示例为shell方式
    - 定义了多个命令时，只有最后一条被执行
    - 如果docker run指定了其他命令，CMD命令被忽略
* ENTRYPOINT ['excutable', 'param1', 'param2']：容器启动后运行的服务
    - 有shell、exec两种执行方式，示例为exec方式
    - 让容器以应用程序或者服务的形式运行，只可添加一条
    - 此命令默认不被docker run提供命令的覆盖
* ENTRYPOINT和CMD
    - ENTRYPOINT和CMD都可以单独使用
    - ENTRYPOINT和CMD联合使用时
        + ENTRYPOINT提供可执行程序和默认参数
        + CMD提供可选参数
* COPY src dest：复制本地主机的src到容器的dest，
* ADD src dest：
    - add和copy功能类似(复制指定的src到容器的dest)，优先使用copy
    - 额外功能：解压src到dest【但必须是gzip, bzip2 or xz格式的tar包】
    - 其中src可以是Dockerfile所在目录的相对路径，也可以是一个url 
* EXPOSE port：告诉宿主机容器暴露的端口
* VOLUME ["/data"]：容器运行时，自动创建一个绑定到特定目录的数据卷
    - 这是为了保证某些数据(比如mysql等数据库)不随容器销毁而丢失
    - 可以通过run -v参数覆盖相关目录
* ONBULID [instruction]：当所创建的镜像作为其他镜像的基础镜像时，所执行的命令

## build构建镜像
* 范例：`docker build -t="ubuntu:ruby" .`
* 参数：
    - -t 仓库名:标签
    - -f 指定dockerfile文件名称，默认文件名称Dockerfile
    - . 注明dockerfile的文件位置

## Dockerfile最佳实践
* 减少镜像层：一次RUN指令形成新的一层，因此，多条shell命令要尽可能的写在一行；
    - 使用&&连接多个命令
    - 过长的行使用反斜线续行
    
* 优化镜像大小：一次RUN形成新的一层，如果没有字同一层删除，无论文件最后是否删除，都会带到下一层，所以要在每一层清理对应的残留数据，减小镜像大小

* 减少网络传输事件：使用软件包、maven仓库等

* 多阶段构建：代码编译、部署写在一个Dockerfile

* 选择尽可能小的基础镜像：如alpine

* 设置时区和时间

    ```
    RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
        echo "Asia/shanghai" > /etc/timezone
    ```

## Dockerfile范例
```
FROM ubuntu:14.04
LABEL maintainer hejingqi@zj-inv.cn
ENV LANG C.UTF-8
RUN echo 'deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse\n\
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse\n\
	deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse\n\
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse\n\
	deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse\n\
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse\n\
	deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse\n\
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse\n'\
	> /etc/apt/sources.list && \
	rm -rf /etc/apt/sources.list.d/* && \
	apt-get update && \
	apt-get install -y openssh-server vim iputils-ping net-tools lrzsz && \
	mkdir -p /var/run/sshd && \
	sed -i '/^PermitRootLogin/s/without-password/yes/' /etc/ssh/sshd_config && \
	sed -i '/^#PasswordAuthentication/s/#//' /etc/ssh/sshd_config && \
	echo "root:zjht4321"|chpasswd && \
	ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
	rm -rf /var/cache/apt/*
EXPOSE 22
CMD /usr/sbin/sshd -D
```

## 多阶段构建范例

```yaml
# stage build
# 使用as为多阶段构建中的某一阶段命名
FROM maven:3.6.3-jdk-8 as build
RUN sed -i '/<mirrors>/a\ \
<mirror> \n\
     <id>nexus-aliyun</id> \n\
     <mirrorOf>central</mirrorOf> \n\
     <name>Nexus aliyun</name> \n\
     <url>http://maven.aliyun.com/nexus/content/groups/public</url>\n\
</mirror>' /usr/share/maven/conf/settings.xml
COPY . /tomcat-java-demo
WORKDIR /tomcat-java-demo
RUN mvn clean package -Dmaven.test.skip=true
# stage prod
FROM tomcat:8.5.47 as prod
RUN rm -rf /usr/local/tomcat/webapps/*
# 从构建的某一阶段复制文件
COPY --from=build /tomcat-java-demo/target/*.war /usr/local/tomcat/webapps/ROOT.war
WORKDIR /usr/local/tomcat/bin
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

# 应用运行
## 镜像内固定运行参数
* Dockerfile
```
FROM python:2.7
LABEL maintainer="istyle.simple@gmail.com"
RUN pip install flask
COPY app.py /app/
WORKDIR /app
EXPOSE 5000
CMD ["python", "app.py"]
```
* app.py
```
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
    return "hello docker"
if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000)
```

## 容器运行时指定参数
* Dockerfile
```
FROM ubuntu:xenial
COPY sources.list /etc/apt/sources.list 
RUN apt update && apt install -y stress
ENTRYPOINT ["/usr/bin/stress"] #基础运行命令
CMD [] #指定可添加参数
```
* sources.list
```
deb http://mirrors.aliyun.com/ubuntu/ xenial main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe
```
* 容器运行
    - 前台：docker run -it ubuntu_stress:latest -m 1 --verbose -t 10s
    - 后台：docker run -itd --name stress ubuntu_stress:latest -m 1 --verbose

# 公有仓库镜像自动化构建

## 可选仓库

* [阿里云容器镜像服务][aliyun-docker-repo]
* [docker hub][docker-hub]

## 自动化构建

* 将Dockerfile等文件放入代码库管理，比如：https://github.com/simple0426/tomcat-java-demo.git
* 在镜像仓库新建镜像，配置自动化构建关联到代码仓库(具体关联到Dockerfile所在目录)
* 只要Dockerfile所在仓库有代码变动，镜像仓库就会自动构建新的镜像

[docker-hub]: https://hub.docker.com/repositories
[aliyun-docker-repo]: https://cr.console.aliyun.com/cn-hangzhou/instances/repositories