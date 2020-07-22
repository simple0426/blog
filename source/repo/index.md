---
title: repo
date: 2019-06-24 21:54:01
---
# charts仓库

* 微软镜像charts仓库【适合国内】
  * stable：http://mirror.azure.cn/kubernetes/charts/
  * incubator：http://mirror.azure.cn/kubernetes/charts-incubator/
* 阿里云：https://apphub.aliyuncs.com

# docker常用镜像

* cirros（约12M）：CirrOS是一个Tiny操作系统，专门用于在云计算环境上运行；包含curl命令
* nicolaka/netshoot（约204M）：包含各种网络调试工具
* busybox:1.28.4（约1M）：包含300多个linux命令的软件集合
* alpine（约5M）：基于alpine linux的最小docker镜像

# docker镜像仓库
- 阿里云：https://2x97hcl1.mirror.aliyuncs.com
- 163：http://hub-mirror.c.163.com
- docker-cn：https://registry.docker-cn.com
- daocloud：http://f1361db2.m.daocloud.io

# [镜像代理-中科大](http://mirrors.ustc.edu.cn/)
* docker.io：docker.mirrors.ustc.edu.cn
* gcr.io：gcr.mirrors.ustc.edu.cn
* quay.io：quay.mirrors.ustc.edu.cn

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
     <url>http://maven.aliyun.com/nexus/content/groups/public</url>
   </mirror>
 </mirrors>
```

# npm淘宝
- 命令行临时设置：--registry=https://registry.npm.taobao.org
- 永久设置：npm config set registry https://registry.npm.taobao.org
    + 验证：npm config get registry

