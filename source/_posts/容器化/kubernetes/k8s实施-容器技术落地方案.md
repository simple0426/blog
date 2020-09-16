---
title: k8s实施-容器技术落地方案
tags:
  - 技术栈
  - 方案
categories:
  - kubernetes
date: 2020-08-06 15:38:16
---


# 容器平台分层架构

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/containers-scheme.jpg)

* 基础设施层-IaaS(openstack/公有云)：服务器、网络、存储、数据库
* 容器引擎层：docker、harbor(镜像仓库)
* 容器编排层：kubernetes
* 访问和工具层-PaaS：web控制台、CICD、监控、日志

# 技术栈选择

## 服务器

* 服务器类型：物理机、虚拟机
* 节点管理【ssd、GPU】
  * 定义节点标签；pod使用亲和性调度
  * 定义节点污点；pod使用容忍度调度

## 存储

* 主流方案-ceph
  * 对象存储：类似oss，程序直接调用
  * 文件存储：类似nfs，直接挂载使用
  * 块存储：类似硬盘、分区，需要格式化为特定文件系统后才可以使用
* 其他方案：glusterfs、fastdfs

## 网络

* 方案
  * calico：纯三层网络方案
  * flannel
* calico
  * 支持k8s的网络策略(ACL)
  * 支持k8s集群与外部通信
    * 实现微服务注册中心(consul/zookeeper)中集群内外不同服务的互通
    * 代替kube-proxy，使用自有的负载均衡(如nginx)

## 镜像仓库

* 主流方案-harbor：支持权限管理、日志审计等功能
* 其他方案
  * nexus
  - jfrog
  - Dockerdistribution

## 监控

主流方案：prometheus+grafana

## 日志

filebeat + kafka + logstash + elasticsearch + kibana

## CICD流程

* 主流：jenkins
* 其他方案：gitlab ci

## 其他注意事项

* 命名空间：资源隔离
* 权限控制：RBAC、网络策略
* 降低使用难度：操作简化、web界面管理等

# 资源规划

## 硬件选择-通用
| 环境     | 节点类型    | 配置        |
| -------- | ----------- | ----------- |
| 实验环境 | master/node | 2C/2G+      |
| 测试环境 | master      | 2C/4G/40G   |
| -        | node        | 4C/8G/40G   |
| 生产环境 | master      | 4C/8G/100G  |
| -        | node        | 8C/64G/500G |

## 容量规划

* 应用占用资源(cpu/内存)：应用数量\*每个应用的副本数\*每个应用占用的cpu/内存资源+20%预留冗余(业务+故障飘逸)

* 系统保留资源(cpu/内存、kubelet设置)：操作系统保留资源+k8s组件保留

  ```
  -system-reserved=cpu=200m,memory=250Mi --kube-reserved=cpu=200m,memory=250Mi \
  --eviction-hard=memory.available<1Gi,nodefs.available<1Gi,imagefs.available<1Gi \
  --eviction-minimum-reclaim=memory.available=500Mi,nodefs.available=500Mi,imagefs.available=1Gi \
  --node-status-update-frequency=10s \
  --eviction-pressure-transition-period=180s
  ```

* 本地硬盘

  * worker节点容量：临时存储(emptyDir/hostpath)+镜像文件+系统组件日志
  * master节点：由于保留系统元数据(etcd)，需要使用ssd提高系统性能

## k8s最大限额

```
在 v1.18 版本中， Kubernetes 支持的最大节点数为 5000。更具体地说，我们支持满足以下所有条件的配置：
节点数不超过 5000
Pod 总数不超过 150000
容器总数不超过 300000
每个节点的 pod 数量不超过 100
```

# 集群部署方式

* [手动二进制]([https://simple0426.gitee.io/2020/07/12/%E5%AE%B9%E5%99%A8%E5%8C%96/kubernetes/kubernetes-%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%B9%E5%BC%8F%E9%83%A8%E7%BD%B2/](https://simple0426.gitee.io/2020/07/12/容器化/kubernetes/kubernetes-二进制方式部署/)) ：手动分组件一步一步搭建集群；可用于学习ks8集群运行细节、也可用于生产环境
* [rancher](https://www.rancher.cn/)：企业级开源kubernetes管理平台
* 使用自动化工具搭建k8s集群
  * [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/)：官方维护，一键部署k8s集群；一般用于测试环境和学习
  * ansible手动：使用ansible编写自动化部署
    * 阿良范例：https://github.com/lizhenliang/ansible-install-k8s
  * kubespray：官方维护，基于ansible编写的部署程序
    * github：https://github.com/kubernetes-sigs/kubespray
    * web：https://kubernetes.io/docs/setup/production-environment/tools/kubespray/

# 集群高可用

* etcd高可用：推荐3、5个节点部署
* apiserver(无状态http服务)：部署多个实例后，使用lvs/haproxy/nginx + keepalived做前端代理
* scheduler、controller-manager：使用自身--leader-elect参数实现高可用【同一时间只有一个组件在使用】
* CoreDNS：扩展pod副本数

# 数据备份-etcd

## kubeadm部署集群

* 定期备份etcd数据

  ```
  ETCDCTL_API=3 etcdctl \
  snapshot save snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key
  ```

* 故障时：暂停kube-apiserver和etcd容器【kubelet管理静态pod：--pod-manifest-path=Directory】

  ```
  mv /etc/kubernetes/manifests/{kube-apiserver.yaml,etcd.yaml} /etc/kubernetes
  ```

* 故障时：转储当前etcd数据：`mv /var/lib/etcd/ /var/lib/etcd.bak`

* 恢复备份数据

  ```
  ETCDCTL_API=3 etcdctl \
  snapshot restore snap.db \
  --data-dir=/var/lib/etcd
  ```

* 启动apiserver和etcd容器

  ```
  mv /etc/kubernetes/{kube-apiserver.yaml,etcd.yaml} /etc/kubernetes/manifests
  ```

## 二进制部署集群

* 定期备份etcd数据

  ```
  ETCDCTL_API=3 etcdctl \
  snapshot save snap.db \
  --endpoints=https://192.168.31.211:2379 \
  --cacert=/opt/etcd/ssl/ca.pem \
  --cert=/opt/etcd/ssl/server.pem \
  --key=/opt/etcd/ssl/server-key.pem
  ```

* g故障时：暂停apiserver和etcd集群

  ```
  systemctl stop kube-apiserver
  systemctl stop etcd
  ```

* 转储故障时的数据

  ```
  mv /var/lib/etcd/default.etcd /var/lib/etcd/default.etcd.bak
  ```

* 在etcd的每个节点恢复数据【修改name和peer-urls为本机信息】

  ```
  ETCDCTL_API=3 etcdctl snapshot restore snap.db \
  --name etcd-3 \
  --initial-cluster="etcd-1=https://192.168.31.211:2380,etcd-2=https://192.168.31.212:2380,etcd-3=https://192.168.31.213:2380" \
  --initial-cluster-token=etcd-cluster \
  --initial-advertise-peer-urls=https://192.168.31.213:2380 \
  --data-dir=/var/lib/etcd/default.etcd
  ```

* 启动etcd和apiserver

  ```
  systemctl start etcd
  systemctl start kube-apiserver
  ```

# 常见问题与调试

## 常见问题

* 网络通信
* tls证书：服务器时间不同步；证书中hosts字段包含所有k8s集群和etcd节点ip
* rbac权限
  * pod连接apiserver使用serviceaccount
  * 组件(kubelet/kube-proxy/kubectl)连接apiserver使用kubeconfig
* 镜像仓库认证
  * 使用kubectl命令创建dockerregistry类型secret(区分命名空间)，在serviceaccount或pod中引用
  * 使用harbor自签tls证书，需要在每个客户端放置ca证书【/etc/docker/certs.d/myregistry:5000/ca.crt】

## pod启动异常

* 查看k8s组件日志
* describe查看pod调度情况
* logs查看pod运行日志

### k8s-event

* 全部事件查看：kubectl get event

* 学习参考：http://www.voidcn.com/article/p-qxvgcuct-bvb.html
* 保留时间(kube-apiserver)： --event-ttl  【默认保留一小时】
* k8s事件报警插件kube-eventer：https://github.com/AliyunContainerService/kube-eventer

## 无法通过service访问应用

* service无法关联pod(get endpoint无信息)：service标签设置是否正确
* 无法通过clusterIP访问pod：kube-proxy是否运行正常并生成相应的iptables/ipvs规则
* 无法通过service名称访问pod：CoreDNS插件是否运行正常
* 无法通过podIP访问pod
  * 网络是否互通，cni插件是否正常
  * pod是否启动(describe)
  * 应用是否正常(logs)

# 最佳实践

## DNS名称解析

* 可以配置cluster-proportional-autoscaler，根据node数量和vCore自动扩展副本数
* 容器内打开nscd(镜像缓存服务)，可大幅提高解析性能
* 禁止生产使用alpine作为基础镜像【可能出现dns解析异常】

## 弹性伸缩

* HPA：pod副本数伸缩
* CA：集群node节点数伸缩

## ingress controller

ingress controller单独worker部署(污点)增加转发能力

## 安全

* apiserver访问限制、操作审计
* k8s事件信息归档、报警