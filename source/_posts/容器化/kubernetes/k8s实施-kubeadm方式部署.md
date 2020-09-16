---
title: k8s实施-kubeadm方式部署
tags:
  - kubeadm
categories:
  - kubernetes
date: 2020-03-11 17:04:08
---
# kubeadm介绍
kubeadm是kubernetes集群的全生命周期管理工具，可实现：集群部署、升级、降级、拆除  
kubeadm仅关心集群的初始化并启动集群，只安装必需的组件(dns)，其他的组件（dashboard、ingress、flannel）则需要管理员自行部署  

* kubeadm init：集群初始化，核心功能是部署master节点的各个组件(kube-api-server/kube-controller-manager/kube-scheduler)
* kubeadm join：将节点加入集群
* kubeadm token：集群构建后管理加入集群的token
* kubeadm reset：删除集群构建过程中产生的文件，恢复到未创建集群的状态

# [部署要求](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
>部署实践：https://gitee.com/simple0426/kubeadm-k8s.git

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
* 启用ipvs模块
```
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir|grep -o "^[^.]*");do
/sbin/modinfo -F filename $i &> /dev/null
if [ $? -eq 0 ];then
/sbin/modprobe $i
fi
done
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
>master、node都操作

* 根据kubeadm要求配置启动参数
* 配置镜像加速地址
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors" : ["https://2x97hcl1.mirror.aliyuncs.com"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

## [安装kubelet、kubeadm、kubectl](https://developer.aliyun.com/mirror/kubernetes)
* master node、workernode都操作

* 此处的kubelet、kubeadm、kubectl需要与下文中kubernetes版本保持一致；由于阿里镜像源滞后，所以需要测试镜像源是否包含指定版本

  ```
  docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.8
  ```

* 指定版本安装：`yum install kubelet-1.18.8 kubeadm-1.18.8 kubectl-1.18.8 -y`

* kubelet开启启动：`systemctl enable kubelet`

## 集群初始化
>master操作

```
kubeadm init --kubernetes-version=v1.18.8 \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.96.0.0/12 \
--apiserver-advertise-address=192.168.31.201 \
--image-repository=registry.aliyuncs.com/google_containers \
--ignore-preflight-errors=NumCPU
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
    + 镜像地址修改：quay.mirrors.ustc.edu.cn
    + 主机间通信接口设置(假设eth1为主机间通信接口：`--iface=eth1`)

  - 应用资源文件：kubectl apply -f kube-flannel.yml

  - bug修复

    ```
    由kubeadm安装的1.17~1.18版本k8s集群使用flannel插件有bug，会造成访问service的地址长时间无响应，需要执行命令修复此问题：
    ethtool --offload flannel.1 rx off tx off
    ```

* 允许master部署负载【master】

  ```
  kubectl taint nodes --all node-role.kubernetes.io/master-
  ```

* 使用kubeadm join命令将node加入集群【node】

# kubeadm管理
## 移除节点
* master节点执行：
    - `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
    - `kubectl delete node <node name>`
* 被移除节点执行：kubeadm reset
* 被移除节点删除iptables：`iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X`
* 被移除节点删除ipvs：ipvsadm -C

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
* 查看证书过期时间(默认1年)：`kubeadm alpha certs check-expiration`
* 证书续签(默认1年)：`kubeadm alpha certs renew all`
  - 续签后使用证书的组件重启：kubelet/kube-proxy/apiserver/scheduler/control-manager

# 附加组件部署
## dashboard
- 下载资源文件：

    ```
    curl -Lo kube-dashboard.yaml https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
    ```

- 资源文件修改
    + 将dashboard的访问端口暴露在宿主机上：containers--》ports--》hostPort: 8443/8000
    + dashboard部署在node02上：nodeSelector--》`kubernetes.io/hostname: "node02"`
    
- [建立管理员](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

- 登录token获取

    ```
    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
    ```

- web访问：https://192.168.31.202:8443/

## ingress-nginx
- 下载资源文件：

    ```
    curl -Lo ingress-nginx.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
    ```

- pod及service修改配置
    + ingress部署在node02上：nodeSelector--》`kubernetes.io/hostname: "node02"`
    + 修改nginx-ingress-controller镜像地址：registry.cn-hangzhou.aliyuncs.com/simple00426/nginx-ingress-controller:0.35.0
    + 将ingress的访问端口暴露在宿主机上：Service--》nodePort：30080/30443



