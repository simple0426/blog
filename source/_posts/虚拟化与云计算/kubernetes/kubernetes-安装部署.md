---
title: kubernetes-安装部署
tags:
  - kubeadm
categories:
  - kubernetes
date: 2020-03-11 17:04:08
---

# 部署方式
* 从零开始手动构建集群
    - [文档参考](https://jimmysong.io/kubernetes-handbook/practice/install-kubernetes-on-centos.html)
    - [源码参考](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster)
    - kubernetes源码下载：https://storage.googleapis.com/kubernetes-release/release/v1.16.2/kubernetes-server-linux-amd64.tar.gz
* [使用kubeadm部署工具](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

# 集群运行模式
* 独立组件【手动安装方式】：系统组件直接以守护进程方式运行在节点上
* 静态pod模式【kubeadm默认方式】，除了kubelet和docker之外其他组件(etcd/api/scheduler/controller-manager)都以静态pod方式运行在master主机上
* 自托管模式(self-hosted)【可通过kubeadm由静态pod转换而来】：类似静态pod方式，除了kubelet和docker之外其他组件都是集群上的pod对象，但这些pod受控于daemonset控制器

# kubeadm
* kubeadm是kubernetes集群的全生命周期管理工具，可实现：集群部署、升级、降级、拆除
* kubeadm仅关心集群的初始化并启动集群，只安装必需的组件(dns)，其他的组件（dashboard、ingress、flannel）则需要管理员自行部署

## 命令
* kubeadm init：集群初始化，核心功能是部署master节点的各个组件(kube-api-server/kube-controller-manager/kube-scheduler)
* kubeadm join：将节点加入集群
* kubeadm token：集群构建后管理加入集群的token
* kubeadm reset：删除集群构建过程中产生的文件，恢复到未创建集群的状态

# 安装要求
>部署实践：https://gitee.com/simple0426/kubeadm-k8s.git

* 主机数：3个及以上
* os：Ubuntu 16.04+、CentOS 7
* 内存：2G左右
* 主机互联（公网或私网）
* 主机名、MAC地址唯一，在hosts中做手动解析
* 个别端口放开
* 关闭swap
* 时间同步
* 关闭防火墙、selinux
* 启用ipvs模块

# 部署步骤
* 安装docker【master+node】
    - 根据kubeadm要求配置启动参数
    - 配置镜像加速地址
* [安装kubelet、kubeadm、kubectl](https://developer.aliyun.com/mirror/kubernetes)【master+node】
* 初始化【master】：`kubeadm init --kubernetes-version=v1.17.3 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=172.17.8.201 --image-repository=registry.aliyuncs.com/google_containers --ignore-preflight-errors=NumCPU --dry-run`
* 根据提示信息操作
    - 配置kubectl【master】
    - [安装网络插件](#网络插件)【master】
    - [控制节点隔离](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation)【可选】
    - 使用kubeadm join命令将node加入集群【node】

## 网络插件
网络插件安装前，内置的dns插件不会启动；本例中使用flannel部署网络(kube-flannel.yml)

* 宿主机流量转换设置：echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
* kube-flannel.yml文件修改
    - pod网络设置：net-conf.json
    - 镜像地址修改
    - 主机间通信接口设置:`--iface=eth1`
* 应用资源文件：kubectl apply -f kube-flannel.yml

# 注意事项
## 移除节点
* master节点执行：
    - `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
    - `kubectl delete node <node name>`
* 被移除节点执行：kubeadm reset
* 被移除节点删除iptables：`iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X`
* 被移除节点删除ipvs：ipvsadm -C

## join认证信息
* token：默认24小时过期
    - 查看：kubeadm token list
    - 产生新的：kubeadm token create
* cert-hash查看
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

# 附加组件部署
## dashboard
- [资源部署](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml):主要修改如下
    + 将dashboard的访问端口暴露在宿主机上：containers--》ports--》hostPort: 8000
    + dashboard部署在node02上：nodeSelector--》kubernetes.io/hostname: "node02"
- [建立管理员](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)
- hosts设置：【172.17.8.202 dashboard.myapp.com】
- 访问：https://dashboard.myapp.com:8443/

## ingress-nginx
- [pod文件](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml)
- [service文件](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/baremetal/service-nodeport.yaml)
- pod及service修改配置
    + ingress部署在node02上：nodeSelector--》kubernetes.io/hostname: "node02"
    + 修改nginx-ingress-controller镜像地址
    + 将ingress的访问端口暴露在宿主机上：nodePort：30080/30443

## [metric-server](https://github.com/kubernetes-sigs/metrics-server)
* 设置api-server：
    - 文件位置：/etc/kubernetes/manifests/kube-apiserver.yaml
    - 配置：--enable-aggregator-routing=true
    - 重载配置：删除api-server的pod使其自动重建
* [下载资源文件](https://github.com/kubernetes-sigs/metrics-server/tree/master/deploy/kubernetes)
* 修改配置(deployment)
    - 修改镜像地址(image)
    - 容器启动参数(args)
        + --kubelet-insecure-tls
        + --kubelet-preferred-address-types=InternalIP
* 验证资源指标API可用性
    - kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods"
    - kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
* 获取node或pod对象的资源使用情况；kubectl top node/pod

## prometheus
* 设置api-server：
    - 文件位置：/etc/kubernetes/manifests/kube-apiserver.yaml
    - 配置：--enable-aggregator-routing=true
    - 重载配置：删除api-server的pod使其自动重建
* 设置kubelet
    - --authentication-token-webhook=true 
    - --authorization-mode=Webhook
* [下载资源文件](https://github.com/coreos/kube-prometheus/tree/master/manifests)
    - kubectl create -f manifests/setup
    - kubectl create -f manifests/
* hosts设置【由于nginx-ingress部署在172.17.8.202】
```
172.17.8.202 alertmanager.myapp.com
172.17.8.202 grafana.myapp.com
172.17.8.202 prometheus.myapp.com
```
* 访问入口设置-ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  rules:
  - host: prometheus.myapp.com
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus-k8s
          servicePort: 9090
  - host: grafana.myapp.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 3000
  - host: alertmanager.myapp.com
    http:
      paths:
      - path: /
        backend:
          serviceName: alertmanager-main
          servicePort: 9093
```
* 访问：
    - grafana：http://grafana.myapp.com:30080/
    - alert：http://alertmanager.myapp.com:30080/
    - prometheus：http://prometheus.myapp.com:30080/
