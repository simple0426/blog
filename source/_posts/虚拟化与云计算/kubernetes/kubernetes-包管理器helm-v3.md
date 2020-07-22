---
title: kubernetes-包管理器helm-v3
date: 2020-07-22 17:59:22
tags:
categories:
---

# helm简介

* 优点
  * 将各种配置(service/deployment/configmap)作为整体管理
  * 资源文件可以复用、分享
  * 可以支持应用级别的版本管理，支持更新、回滚

* 本质：helm是k8s集群的包管理工具，类似linux上的apt/yum工具
* 核心概念
  * helm：helm二进制管理工具，主要用于k8s应用的创建、打包、发布、管理
  * Chart：描述一个应用的所有k8s资源文件集合
  * Release：Chart的部署实体，chart被部署后会生成一个对应的Release
* 版本：
  * v2
  * v3：没有服务端tiller组件

* [客户端下载](https://github.com/helm/helm/tags)：
  * linux：https://mirrors.huaweicloud.com/helm/v3.2.4/helm-v3.2.4-linux-amd64.tar.gz
  * windows：https://mirrors.huaweicloud.com/helm/v3.2.4/helm-v3.2.4-windows-amd64.zip

# helm命令

> 读取本地的kubeconfig用于k8s集群认证

* 命令行补全：source <(helm completion bash)



# 常规应用管理

* 安装：helm install ui microsoft/weave-scope
  * 安装测试（查看部署内容）：--dry-run
  * 查看charts默认值：helm show values microsoft/mysql
  * 设置安装charts的参数：helm install db --set persistence.storageClass="example-nfs" microsoft/mysql
    * 命令行传参：--set
    * 配置文件方式：-f
* 卸载：helm uninstall ui
* 查看release：
  * 查看运行中的release：helm list
  * 查看release部署历史：helm history web1 
  * 查看release各组件的部署详情：helm get manifest web1 
* 升级：helm upgrade web1 --set image.tag="1.19" mychart
* 回滚到上一版本：helm rollback web1

# chart仓库与hub

## 公有hub

> 可使用web搜索

* 官方：https://hub.helm.sh/
  * 代码地址(开发者)：https://github.com/helm/charts
  * charts仓库：
    * stable：https://kubernetes-charts.storage.googleapis.com
    * incubator：https://kubernetes-charts-incubator.storage.googleapis.com
  * 微软镜像charts仓库【适合国内】
    * stable：http://mirror.azure.cn/kubernetes/charts/
    * incubator：http://mirror.azure.cn/kubernetes/charts-incubator/
* kubeapps(开发者)：https://hub.kubeapps.com/
  * 代码地址：https://github.com/kubeapps/kubeapps
  * charts仓库：https://charts.bitnami.com/bitnami
* 阿里云(开发者)：https://developer.aliyun.com/hub/
  * 代码地址：https://github.com/cloudnativeapp/charts
  * charts仓库：https://apphub.aliyuncs.com【适合国内】

## 命令使用

* hub
  * helm search hub可以搜索由[Monocular](https://github.com/helm/monocular)渲染的charts hub
  * --endpoint 指定要搜索的hub地址（默认官方）

* 仓库
  * 添加仓库：helm repo add name URL
  * 更新仓库元数据：helm repo update
  * 删除仓库：helm repo remove name
  * 搜索仓库：helm search repo chart_name

## 搭建私有charts仓库

* 创建索引文件：helm repo index my-repo --url http://49.232.17.71:8900

  * my-repo为多个chart压缩包所在目录
  * url为helm下载chart使用的主机地址

* 将chart压缩包目录(包含索引文件和chart压缩包)使用web服务发布出去

  ```
  docker run -d --name=chart-repo --restart=always \
  -v $(pwd)/my-repo:/usr/share/nginx/html \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf \
  -p 8900:80 nginx
  ```

  可以在nginx配置文件中开启文件目录索引，这样使用者就可以在web界面查看charts仓库包含的charts

  ```
  autoindex on;
  autoindex_exact_size off;
  autoindex_localtime on;
  charset utf-8;
  ```

# charts开发

* 创建chart：helm create mychart 
* 打包chart：mychart-0.1.0.tgz

## chart文件结构

```
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

## chart模板语法

