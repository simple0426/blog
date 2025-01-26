---
title: k8s实施-kubeadm方式部署
tags:
  - kubeadm
categories:
  - kubernetes
date: 2020-03-11 17:04:08
---
# [kubeadm介绍](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
kubeadm是kubernetes集群的全生命周期管理工具，可实现：集群部署、升级、降级、拆除  
kubeadm仅关心集群的初始化并启动集群，只安装必需的组件(dns)，其他的组件（dashboard、ingress、flannel）则需要管理员自行部署  

* kubeadm init：集群初始化，核心功能是部署master节点的各个组件(kube-api-server/kube-controller-manager/kube-scheduler)
* kubeadm join：将节点加入集群
* kubeadm token：集群构建后管理加入集群的token
* kubeadm reset：删除集群构建过程中产生的文件，恢复到未创建集群的状态

# [部署要求](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
>部署实践：https://gitee.com/simple0426/kubeadm-k8s.git
>
>k8s 1.32版本要求linux内核4.15，centos7默认为3.10，需要升级内核

* 主机数：3个及以上
* os：Ubuntu 16.04+、CentOS 7
* 内存：2G以上
* cpu：2核以上
* 网络：主机互联、且可以连接公网（下载镜像）
* 主机名、MAC地址唯一，在hosts中做手动解析
* [个别端口放开](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)
* 关闭swap
```
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```
* 时间同步
```
yum install -y ntp
ntpdate times.aliyun.com
systemctl start ntpd
systemctl enable ntpd
```
* 关闭防火墙、selinux
```
systemctl stop firewalld
systemctl disable firewalld
systemctl stop iptables
systemctl disable iptables
setenforce 0
sed -i 's/=enforcing/=disabled/g' /etc/selinux/config
```
* iptables设置：可以查看网桥流量
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

# 部署步骤
## 安装docker
* master、node都操作
* 根据kubeadm要求配置启动参数
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors" : ["https://e97bf3ff28c74b62ac99a5f949cd62ba.mirror.swr.myhuaweicloud.com",
  "https://docker.m.daocloud.io"]
}
EOF
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

## 安装cri-dockerd

因为k8s的新版本要求container runtime实现[Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri) (CRI)，而docker engine未实现，所以需要使用cri-dockerd这个适配器，这样kubelet可以操作docker engine

```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd-0.3.14-3.el7.x86_64.rpm
rpm -ivh cri-dockerd-0.3.14-3.el7.x86_64.rpm
# pod使用的infra使用国内镜像
sed -i '/ExecStart/s%$%--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9%' /usr/lib/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker
systemctl start cri-docker
```

## [安装kubelet/kubeadm/kubectl](https://developer.aliyun.com/mirror/kubernetes)

* master node、worker node都操作

* 此处的kubelet、kubeadm、kubectl需要与下文中kubernetes版本保持一致；由于阿里镜像源滞后，所以需要测试镜像源是否包含指定版本

  ```
  docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.32.0
  ```

* 安装：`yum install kubelet kubeadm kubectl -y`

* kubelet开机启动：`systemctl enable kubelet`

## 集群初始化
>master操作：由于安装docker和cri-dockerd，k8s会扫描到2个socket，需要指定k8s链接的socket（cri-docked.sock）

```
kubeadm init --kubernetes-version=v1.32.0 \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.96.0.0/12 \
--apiserver-advertise-address=192.168.31.201 \
--image-repository=registry.aliyuncs.com/google_containers \
--cri-socket=unix:///var/run/cri-dockerd.sock
```

## 根据init结果提示操作
* 配置kubectl【master】

* 安装网络插件【master】

  - 下载资源文件

    ```
    curl -LO https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```

  - kube-flannel.yml文件修改

    + pod网络设置(net-conf.json)
    + 镜像地址修改：registry.cn-hangzhou.aliyuncs.com/simple00426/
    + 主机间通信接口设置(假设eth1为主机间通信接口：`--iface=eth1`)

  - 应用资源文件：kubectl apply -f kube-flannel.yml

* 允许master部署负载【control-plane即为master】

  ```
  kubectl taint nodes --all node-role.kubernetes.io/control-plane-
  ```

* 使用kubeadm join命令将node加入集群【node节点操作，注意指定socket】

  `--cri-socket=unix:///var/run/cri-dockerd.sock`

# kubeadm管理
## 移除节点
* master节点执行：`kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets`
* 被移除节点执行：
    * 清除init或join命令产生的信息：`kubeadm reset --cri-socket=unix:///var/run/cri-dockerd.sock  `
    * 删除iptables：`iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X`
    * 删除ipvs：`ipvsadm -C`
* master节点执行：`kubectl delete node <node name>`

## join认证信息
* token：默认24小时过期
    - 查看：`kubeadm token list`
    - 产生新的：`kubeadm token create --print-join-command`
* cert-hash查看
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

## 证书管理
* 查看证书过期时间(默认1年)：`kubeadm certs check-expiration`
* 证书续签(默认1年)：`kubeadm certs renew all`
  - 续签后使用证书的组件重启：kubelet/kube-proxy/apiserver/scheduler/control-manager

# 附加组件部署
## ~~dashboard(废弃)~~
> dashboard 7.0版本只能使用helm包管理器安装，原来的安装方式https访问报错
>
> [dashboard7.0部署参考](https://simple0426.github.io/blog/2020/03/30/%E5%AE%B9%E5%99%A8%E5%8C%96/kubernetes/kubernetes-%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E5%92%8C%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6/#%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E8%8C%83%E4%BE%8B-serviceaccount)

- ~~资源文件修改~~
  
    + 将dashboard的访问端口暴露在宿主机上：containers--》ports--》hostPort: 8443/8000
    + dashboard部署在node02上：nodeSelector--》`kubernetes.io/hostname: "node02"`
    
- ~~建立管理员~~

    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    ```

- ~~登录token获取~~

    ```
    kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
    ```

- ~~web访问：https://192.168.31.202:8443/~~

## ingress-nginx
- 下载资源文件：

    ```
    curl -Lo ingress-nginx.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
    ```

- pod及service修改配置
    + ingress部署在node02上：nodeSelector--》`kubernetes.io/hostname: "node02"`
    
    + ingress的访问端口暴露在宿主机上：Service--》nodePort：30080/30443
    
    + 修改镜像地址：
    
      ```
      registry.cn-hangzhou.aliyuncs.com/simple00426/kube-webhook-certgen:v1.5.0
      registry.cn-hangzhou.aliyuncs.com/simple00426/nginx-ingress-controller:v1.12.0
      ```

## metrics-server

参考：[kubernetes-监控和日志](https://simple0426.github.io/blog/2020/03/11/%E5%AE%B9%E5%99%A8%E5%8C%96/kubernetes/kubernetes-%E7%9B%91%E6%8E%A7%E5%92%8C%E6%97%A5%E5%BF%97/#metrics-server)

## storageclass

参考：[kubernetes-存储和持久化](https://simple0426.github.io/blog/2020/02/24/%E5%AE%B9%E5%99%A8%E5%8C%96/kubernetes/kubernetes-%E5%AD%98%E5%82%A8%E5%92%8C%E6%8C%81%E4%B9%85%E5%8C%96/#storageclass%E5%88%9B%E5%BB%BA)