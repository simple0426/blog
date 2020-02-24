---
title: kubernetes-pod入门
tags:
  - sidecar
  - pod
categories:
  - kubernetes
date: 2020-02-17 01:33:24
---

# 容器
* 容器的本质是一个进程，是一个视图被隔离、资源受限制的进程
* 容器里pid=1的进程就是应用本身
* 管理虚拟机就是管理基础设施(操作系统)，管理容器就是管理应用本身(进程)
* kubernetes相当于云时代的操作系统，容器镜像相当于操作系统的软件安装包

# Pod
* 容器设计本身就是一种“单进程”模型；可以运行多个进程，但是不便于直接管理
    - 如果需要运行多进程，则pid=1的进程要具有管理其他进程的能力(systemd)
    - 也可以直接运行systemd，由systemd管理多进程；但是，此时：【容器管理】=【管理systemd】!=【直接管理应用本身】
* 作为类比，容器相当于进程(linux中的线程)，pod相当于进程组(linux进程，包含至少一个线程)
* 在k8s中，pod是一个逻辑单位，它包含多个相互协作的容器，共享某些资源(volume/网络)
    - pod是资源分配单位【类比，进程是操作系统资源分配的单位】
    - pod也是原子调度单位
* pod与多容器
    - 如果只是亲密关系，可以通过调度器让俩个应用部署在同一台宿主机上
    - 如果是超亲密关系(比如：产生日志的应用和写日志的应用)，则需要将多个应用定义在一个pod中
        + 会发生直接的文件交互
        + 使用localhost或socket进行本地通信
        + 会发生频繁的RPC调用
        + 会共享某些namespace

# Pod实现
## 网络实现
* 启动一个infra container容器，pod里的其他容器通过join namespace的方式加入infra container的network namespace中
* pod的网络地址即是infra的地址
* pod的生命周期也即infra的生命周期

## 共享存储
在pod级别创建volume，然后pod内的所有容器挂载这个volume

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
     hostPath:
        path: /data
  containers:
  - name: nginx-container
     image: nginx
     volumeMounts:
      - name: shared-data
         mountPath: /usr/sharr/nginx/html
  - name: debian-container
     image: debian
     volumeMounts:
      - name: shared-data
         mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

# 容器设计模式
* sidecar：在pod里定义一些专门的容器，来执行主业务容器所需要的辅助工作
* 通过sidecar方式，可以实现功能的解耦和重用

## 应用场景-war包
* 可选方式
    - 将war包和tomcat都打包进镜像中，但是代码和tomcat的更新都需要更新镜像
    - 制作单独的tomcat镜像，在容器运行时将war包挂载到容器内，此时则需要单独维护war包和一个分布式文件系统
* sidecar方式-initContainer
    - 先启动一个initContainer，将war包拷贝到共享目录
    - 再启动container，这个容器启动tomcat(共享目录webapps中包含war包)

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: resouer/sample:v2
     name: war
     command: ["cp", "/sample.war", "/app"]
     volumeMounts:
     - mountPath: /app
        name: app-volume
  contaners:
  - image: resouer/tomcat:7.0
     name: tomcat
     command: ["sh", "-c", "/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
     volumeMounts:
      - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
         name: app-volume
      ports:
      - containerPort: 8080
         hostPort: 8001
    volumes:
    - name: app-volume
       emptyDir: {}
```
## 其他场景
* 日志收集(log)
* Debug应用(debug)
* 容器代理(proxy)：单独写一个proxy用来处理对外部集群的交互
* 适配器(adapter)：单独写一个应用用来处理URL变更、数据格式变更的操作，以完成对现有业务的兼容
