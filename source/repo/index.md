---
title: repo
date: 2019-06-24 21:54:01
---
# charts仓库

* 微软镜像charts仓库【适合国内】
  * stable：`http://mirror.azure.cn/kubernetes/charts/`
  * incubator：`http://mirror.azure.cn/kubernetes/charts-incubator/`

# docker常用镜像

* cirros（约12M）：CirrOS是一个Tiny操作系统，专门用于在云计算环境上运行；包含curl命令
* nicolaka/netshoot（约204M）：包含各种网络调试工具
* busybox:1.28.4（约1M）：包含300多个linux命令的软件集合
* alpine（约5M）：基于alpine linux的最小docker镜像

# docker-hub镜像加速
- 阿里云：https://2x97hcl1.mirror.aliyuncs.com
- 163：http://hub-mirror.c.163.com
- DaoCloud：https://f1361db2.m.daocloud.io
- 七牛： https://reg-mirror.qiniu.com

# 容器镜像代理
* k8s.gcr.io：gcr.io/google-containers
* gcr.io：
  * 中科大：gcr.mirrors.ustc.edu.cn
  * 阿里云：registry.aliyuncs.com/google_containers
  * 七牛：gcr-mirror.qiniu.com
* quay.io：
  * 中科大：quay.mirrors.ustc.edu.cn
  * 七牛：quay-mirror.qiniu.com

# pypi阿里云
- 命令行设置：-i https://mirrors.aliyun.com/pypi/simple/
- 配置文件设置：

```
~/.pip/pip.conf
中添加或修改:

[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
```

# maven阿里云
>conf/settings.xml

```
<mirrors>
   <mirror>
     <id>nexus-aliyun</id>
     <mirrorOf>central</mirrorOf>
     <name>Nexus aliyun</name>
     <url>https://maven.aliyun.com/repository/public</url>
   </mirror>
 </mirrors>
```

# npm淘宝
- 命令行临时设置：--registry=https://registry.npmmirror.com/
- 永久设置：npm config set registry https://registry.npmmirror.com/
    + 验证：npm config get registry

