---
title: kubernetes-包管理器helm
date: 2020-03-08 12:21:44
tags:
  - helm
  - charts
categories: ['kubernetes']
---

# [简介](https://helm.sh)
## 实现目标
* 能够管理复杂的程序结构
* 方便升级：可使用就地升级和自定义钩子
* 便于分享：使用charts构建和分享程序
* 便于回滚：使用helm rollback快速回滚

## 核心术语
* Helm：是k8s的应用程序管理器，也是helm的客户端；相当于linux系统的apt、yum；主要完成如下工作
    - 本地charts开发
    - 管理charts仓库
    - 与tiller服务交互：发送charts以安装、查询Release的相关信息、升级或卸载已有的Release
* Tiller server：它是运行于k8s集群中的应用；helm的服务端程序，接收helm客户端请求，与kubernetes API Server交互，完成以下任务：
    - 监听来自helm客户端的请求
    - 安装应用，合并charts和Config为一个Release
    - 跟踪Release状态
    - 升级或卸载Release
* Chart：是helm的核心打包组件，用于将kubernetes的资源(deployments/service/configmap等)打包进一个charts中
* Charts：一个Helm程序包，相当于linux的rpm、deb包
* Repository：charts仓库，存储charts程序，相当于linux的yum或apt仓库源
* Config：charts程序实例化运行时使用的配置文件
* Release：charts实例化配置后运行于k8s集群中的一个charts实例；在同一个集群中，一个charts可以使用不同的Config重复安装多次，每次安装都会创建一个新的Release；相当于linux的进程

## 应用管理流程
* 从0开始创建charts：helm create
* 将charts及相关的文件打包为归档格式：helm package
* 将charts存储于仓库中并与之交互：helm repo
* 在kubernetes集群中安装或卸载charts：helm install、delete
* 管理经Helm安装的应用的版本发行周期：helm rollback

# helm安装
## 安装helm-client
* [源码下载]
    - [官方](https://github.com/helm/helm/releases)【被墙不可用，可以查询版本】
    - google:可基于github版本和google地址前缀下载源码
        + [linux](https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-linux-amd64.tar.gz)
        + [windows](https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-windows-amd64.zip)
* 解压：tar -xzvf helm-v2.14.1-linux-amd64.tar.gz
* 移动二进制文件：mvn linux-amd64/helm /usr/local/bin/
* 注意事项：helm命令运行的节点应该可以正常运行kubectl命令，或者至少有可用kubeconfig配置文件，这样才可以和运行于k8s集群中的tiller server进行通信

## 安装tiller-server
* rbac配置--资源清单方式
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authoriation.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
RoleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
* rbac配置--命令行方式
    - `kubectl create serviceaccount --namespace kube-system tiller`
    - `kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`
* 初始化安装tiller server：`helm init --service-account tiller --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.1 --stable-repo-url https://apphub.aliyuncs.com --debug`
* 卸载：
    - kubectl delete deploy tiller-deploy -n kube-system
    - helm reset

# 操作命令
* helm repo update：更新本地仓库元信息
* helm search charts_name：搜索软件包
* helm inspect charts_name：显示包详情
* helm install charts_name -n release_name：安装软件包
    - -n release_name：部署的release名称
    - --dry-run：测试
    - --debug：调试
    - -f Config：使用自定义配置文件
    - --set key=value：使用自定义配置
    - 安装源(charts_name)：目录、压缩包、仓库、URL
* helm list：显示已安装的Release
* helm status release_name：显示release的状态和提示信息
* helm delete release_name：删除release
* helm upgrade：升级
* helm rollback：回滚
* helm history：显示版本历史

# charts语法
charts是helm使用的kubernetes程序打包格式，一个charts就是一个描述k8s资源文件的集合
## 目录结构
```
mychart #charts名称
├── Chart.yaml charts元数据信息
├── LICENSE 许可证信息【可选】
├── README.md README文件【可选】
├── requirements.yaml 当前charts的依赖关系【可选】
├── charts 当前charts依赖的其他charts文件
│   ├── mysql-6.8.0.tgz
├── templates 模板文件，用于生成kubernetes资源
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt 模板注解文件，描述如何使用charts；helm status命令也会输出此内容
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml #默认配置文件，helm install -f参数会覆盖此默认配置
```
## Chart.yaml
```
apiVersion: v1 
appVersion: 5.0.7 #程序版本，redis版本
description: Open source, advanced key-value store. It is often referred to as a data
  structure server since keys can contain strings, hashes, lists, sets and sorted
  sets. #描述信息
engine: gotpl #模板引擎，go模板
home: http://redis.io/ #项目主页
icon: https://bitnami.com/assets/stacks/redis/img/redis-stack-220x234.png #项目图标
keywords: #项目关键词
- redis
- keyvalue
- database
maintainers: #项目维护者
- email: containers@bitnami.com
  name: Bitnami
- email: cedric@desaintmartin.fr
  name: desaintmartin
name: redis #charts名称
sources: #项目源码
- https://github.com/bitnami/bitnami-docker-redis
version: 10.5.3 #charts版本
```
## requirements.yaml
```
dependencies:
- name: mysql #被依赖的charts名称
  version: 6.8.0 #被依赖的charts版本
  repository: https://apphub.aliyuncs.com #依赖的软件所属仓库（需要事前添加仓库）
  alias：给被依赖的charts建立别名
```
* 使用helm dependency update命令更新依赖关系，自动下载被依赖的charts到charts目录

# 自定义charts
* 创建空charts：helm create mychart
* 修改配置：
    - values.yaml中的默认配置、镜像信息
    - chart.yaml中的描述信息
* 语法检查：helm lint mychart
* 安装部署：helm install --name myapp ./mychart --set service.type=NodePort
* 打包：helm package ./mychart
