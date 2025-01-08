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
- 华为云：https://e97bf3ff28c74b62ac99a5f949cd62ba.mirror.swr.myhuaweicloud.com
- DaoCloud：https://docker.m.daocloud.io
- 自建：https://www.ewalk.site:5000

# 容器镜像代理
> https://github.com/DaoCloud/public-image-mirror#%E6%94%AF%E6%8C%81%E5%89%8D%E7%BC%80%E6%9B%BF%E6%8D%A2%E7%9A%84-registry-%E4%B8%8D%E6%8E%A8%E8%8D%90

|                   |                       |                                                              |              |
| ----------------- | --------------------- | ------------------------------------------------------------ | ------------ |
| 源站              | 替换为                | 备注                                                         | 自建代理端口 |
| docker.elastic.co | elastic.m.daocloud.io |                                                              |              |
| __docker.io__         | docker.m.daocloud.io  |                                                              | 5000         |
| __gcr.io__            | gcr.m.daocloud.io     | 测试镜像：kubeflow-images-public/admission-webhook:v1.1.0-g3ac3d08b | 5003         |
| ghcr.io           | ghcr.m.daocloud.io    | 【国内可用】测试镜像：nikiforovall/dotnet-script:latest      | 5001         |
| k8s.gcr.io        | k8s-gcr.m.daocloud.io | k8s.gcr.io 已被迁移到 registry.k8s.io；                      | 5004         |
| __registry.k8s.io__   | k8s.m.daocloud.io     | 测试镜像：ingress-nginx/kube-webhook-certgen:v1.5.0          | 5005         |
| mcr.microsoft.com | mcr.m.daocloud.io     |                                                              |              |
| nvcr.io           | nvcr.m.daocloud.io    |                                                              |              |
| quay.io           | quay.m.daocloud.io    | 【国内可用】测试镜像：tigera/operator:v1.36.2                | 5002         |

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

