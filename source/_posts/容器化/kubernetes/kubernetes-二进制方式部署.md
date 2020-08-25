---
title: kubernetes-二进制方式部署
categories:
  - kubernetes
date: 2020-07-12 02:50:22
tags:
---


# 重度参考
* [部署一套完整的Kubernetes高可用集群（上）](https://mp.weixin.qq.com/s/VYtyTU9_Dw9M5oHtvRfseA)
* [​部署一套完整的Kubernetes高可用集群（下）](https://mp.weixin.qq.com/s/F9BC6GALHiWBK5dmUnqepA)

# 前置条件

## 系统设置

* 主机数：4台以上
* 操作系统： CentOS7.x-86_x64 
* 软件版本：
  * docker：19.03
  * kubernetes：1.18
* 硬件要求：2核2G以上
* 网络：主机间互联、且可以连接公网（下载容器镜像）
* 主机名、MAC地址唯一，在hosts中做手动解析
* [个别端口放开](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)
* 关闭swap
```
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```
* 时间同步

  > 所有主机时间和互联网时间都需要同步，否则使用证书进行连接认证时会出错
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

## 安装docker

* 使用阿里云镜像源，采用yum方式安装

* 配置docker-engine

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
  ```

* docker启动并设置开机自启动

  ```
  systemctl daemon-reload
  systemctl enable docker
  systemctl start docker
  ```


## 架构规划

| 角色                    | ip                                 | 服务组件                                                     |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------ |
| master1(node/LB-master) | 192.168.31.211/192.168.31.216(vip) | apiserver、scheduler、controller-manager、kubelet、kube-proxy、etcd、nginx、keepalived |
| node1                   | 192.168.31.212                     | kubelet、kube-proxy、etcd                                    |
| node2                   | 192.168.31.213                     | kubelet、kube-proxy、etcd                                    |
| master2(node/LB-backup) | 192.168.31.214                     | apiserver、scheduler、controller-manager、kubelet、kube-proxy、nginx、keepalived |

* master节点部署master组件（apiserver、scheduler、controller-manager）和node（kubelet、kube-proxy）组件，所以工作负载也可以在master上运行
* etcd部署在3个节点上（211/212/213）
* k8s集群和etcd使用两台证书(即2个CA)
* 先实现单master架构，后扩容为多master架构
* 高可用架构采用nginx+keepalived实现，nginx和keepalived也部署在master节点

# 部署etcd集群

## 生成etcd证书

### 下载cfssl工具

cfssl是一个开源的证书管理工具，使用json生成证书文件，以下使用master作为操作主机

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

### 制作ca根证书

* 创建工作目录

  ```
  mkdir ~/tls/{etcd,k8s} -p
  cd ~/tls/etcd/
  ```

* 创建ca配置文件

  ```
  cat > ca-config.json << EOF
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "www": {
           "expiry": "87600h",
           "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ]
        }
      }
    }
  }
  EOF
  
  cat > ca-csr.json << EOF
  {
      "CN": "etcd CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing"
          }
      ]
  }
  EOF
  ```

* 使用cfssl工具生成ca证书

  ```
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca
  ls ca*.pem
  ```

### 使用ca证书签发etcd证书

* 创建etcd证书请求文件【hosts字段包含集群的所有节点，可以预留几个ip用于后期集群扩容】

  ```
  cat > server-csr.json << EOF
  {
      "CN": "etcd",
      "hosts": [
      "192.168.31.211",
      "192.168.31.212",
      "192.168.31.213",
      "192.168.31.214",
      "192.168.31.215",
      "192.168.31.216"
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "BeiJing",
              "ST": "BeiJing"
          }
      ]
  }
  EOF
  ```

* 制作证书

  ```
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
  ls server*.pem
  ```

## 部署etcd集群

* 下载二进制文件：https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz 

* 创建工作目录并解压二进制文件

  ```
  mkdir /opt/etcd/{bin,cfg,ssl} -p
  tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
  mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
  ```

* 创建etcd配置文件

  ```
  cat > /opt/etcd/cfg/etcd.conf << EOF
  ETCD_NAME="etcd-1"
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="https://192.168.31.211:2380"
  ETCD_LISTEN_CLIENT_URLS="https://192.168.31.211:2379"
  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.211:2380"
  ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.211:2379"
  ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.211:2380,etcd-2=https://192.168.31.212:2380,etcd-3=https://192.168.31.213:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
  ETCD_INITIAL_CLUSTER_STATE="new"
  EOF
  ```

  * ETCD_NAME：节点名称，集群中唯一
  * ETCD_DATA_DIR：集群数据目录
  * ETCD_LISTEN_PEER_URLS：集群通信时的本机地址
  * ETCD_LISTEN_CLIENT_URLS：本节点提供的客户端服务地址
  * ETCD_INITIAL_ADVERTISE_PEER_URLS：集群初始化时，通告其他节点本机的通信地址
  * ETCD_ADVERTISE_CLIENT_URLS：集群初始化时，通告其他节点本机服务地址
  * ETCD_INITIAL_CLUSTER：集群节点
  * ETCD_INITIAL_CLUSTER_TOKEN：集群初始化时的认证token
  * ETCD_INITIAL_CLUSTER_STATE：节点加入的集群的状态；NEW为新集群；EXISTING表示加入已有集群

* systemd管理etcd

  ```
  cat > /usr/lib/systemd/system/etcd.service << EOF
  [Unit]
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target
  [Service]
  Type=notify
  EnvironmentFile=/opt/etcd/cfg/etcd.conf
  ExecStart=/opt/etcd/bin/etcd \
  --cert-file=/opt/etcd/ssl/server.pem \
  --key-file=/opt/etcd/ssl/server-key.pem \
  --peer-cert-file=/opt/etcd/ssl/server.pem \
  --peer-key-file=/opt/etcd/ssl/server-key.pem \
  --trusted-ca-file=/opt/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
  --logger=zap
  Restart=on-failure
  LimitNOFILE=65536
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 复制证书到配置文件中的路径

  ```
  cp ~/tls/etcd/ca*.pem ~/tls/etcd/server*.pem /opt/etcd/ssl/
  ```

* 启动并设置开机启动【第一台etcd启动时会挂起，只需查看日志和进程是否存在即可】

  ```
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
  ```

* 复制文件到其他节点

  ```
  scp -r /opt/etcd/ root@192.168.31.212:/opt
  scp -r /opt/etcd/ root@192.168.31.213:/opt
  scp /usr/lib/systemd/system/etcd.service root@192.168.31.212:/usr/lib/systemd/system
  scp /usr/lib/systemd/system/etcd.service root@192.168.31.213:/usr/lib/systemd/system
  ```

* 修改节点2和节点3的配置文件【etcd.conf】，启动服务并设置开机启动

## 查看集群状态

```
ETCDCTL_API=3
/opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.31.211:2379,https://192.168.31.212:2379,https://192.168.31.213:2379" endpoint health
```

# master节点部署

## 生成kube-apiserver证书

* 创建ca证书配置文件

  ```
  cat > ca-config.json << EOF
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "kubernetes": {
           "expiry": "87600h",
           "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ]
        }
      }
    }
  }
  EOF
  cat > ca-csr.json << EOF
  {
      "CN": "kubernetes",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing",
              "O": "k8s",
              "OU": "System"
          }
      ]
  }
  EOF
  ```

* 生成ca证书

  ```
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca
  ls ca*.pem
  ```

* 创建kube-apiserver证书请求【hosts中的ip为所有master/LB/VIP ip,为了后期扩容可以预留几个ip】

  ```
  cd TLS/k8s
  cat > server-csr.json << EOF
  {
      "CN": "kubernetes",
      "hosts": [
        "10.0.0.1",
        "127.0.0.1",
        "192.168.31.211",
        "192.168.31.212",
        "192.168.31.213",
        "192.168.31.214",
        "192.168.31.215",
        "192.168.31.216",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "BeiJing",
              "ST": "BeiJing",
              "O": "k8s",
              "OU": "System"
          }
      ]
  }
  EOF
  ```

* 生成kube-apiserver证书

  ```
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
  ls server*.pem
  ```

## 下载并解压二进制文件

* [下载](  https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md )服务端二进制文件：kubernetes-server-linux-amd64.tar.gz

* 解压二进制文件

  ```
  mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
  tar xzf kubernetes-server-linux-amd64.tar.gz
  cd kubernetes/server/bin/
  cp kube-apiserver kube-controller-manager kube-scheduler /opt/kubernetes/bin/
  cp kubectl /usr/bin
  ```

## apiserver部署

* 创建apiserver配置文件

  ```
  cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
  KUBE_APISERVER_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --etcd-servers=https://192.168.31.211:2379,https://192.168.31.212:2379,https://192.168.31.213:2379 \\
  --bind-address=192.168.31.211 \\
  --secure-port=6443 \\
  --advertise-address=192.168.31.211 \\
  --allow-privileged=true \\
  --service-cluster-ip-range=10.0.0.0/24 \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
  --authorization-mode=RBAC,Node \\
  --enable-bootstrap-token-auth=true \\
  --token-auth-file=/opt/kubernetes/cfg/token.csv \\
  --service-node-port-range=30000-32767 \\
  --kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
  --kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
  --tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
  --tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --etcd-cafile=/opt/etcd/ssl/ca.pem \\
  --etcd-certfile=/opt/etcd/ssl/server.pem \\
  --etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
  EOF
  ```

  * --logtostderr=false：启用日志
  * --v=2：日志级别
  * --log-dir：日志目录
  * --etcd-servers：etcd集群地址
  * --bind-address：服务监听地址
  * --secure-port：https端口
  * --advertise-address：集群通告地址
  * --allow-privileged：允许开启特权容器
  * --service-cluster-ip-range：service虚拟ip地址段
  * --enable-admission-plugins：开启准入控制插件
  * --authorization-mode：授权模式，开启RBAC授权和Node自管理
  * --enable-bootstrap-token-auth：开启kubelet的bootstrap机制
  * --token-auth-file： bootstrap token文件 
  * --service-node-port-range：service的Nodeport端口设置范围
  * --kubelet-client-*： apiserver访问kubelet客户端的证书 
  * --tls-cert-*：apiserver的https证书
  * --client-ca-file：客户端证书的ca证书
  * --service-account-key-file：ca证书的key，用于验证ServiceAccount的tokens
  * --etcd-*：etcd集群证书配置
  * --audit-log：审计日志配置

* 复制api-server证书到配置文件中的路径

  ```
  cp ~/tls/k8s/ca*.pem ~/tls/k8s/server*.pem /opt/kubernetes/ssl/
  ```

* bootstrap token文件制作

  ```
  cat > /opt/kubernetes/cfg/token.csv << EOF
  c47ffb939f5ca36231d9e3121a252940,kubelet-bootstrap,10001,"system:node-bootstrapper"
  EOF
  ```

  文件格式：token，用户名，uid，用户组

  token生成方式：`22972cc804fc8612da9f3c1fc597ac67`

* systemd管理kube-apiserver

  ```
  cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
  [Unit]
  Description=Kubernetes API Server
  Documentation=https://github.com/kubernetes/kubernetes
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
  ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
  Restart=on-failure
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 服务启动并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kube-apiserver
  systemctl enable kube-apiserver
  ```

* 授权kubelet-bootstrap用户允许请求证书

  ```
  kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
  ```

## controller-manager部署

* 创建配置文件

  ```
  cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
  KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --leader-elect=true \\
  --master=127.0.0.1:8080 \\
  --bind-address=127.0.0.1 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.244.0.0/16 \\
  --service-cluster-ip-range=10.0.0.0/24 \\
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --experimental-cluster-signing-duration=87600h0m0s"
  EOF
  ```

  * --leader-elect：启动多个controller时自动举行选举形成高可用
  * --master：通过本地非安全端口连接apiserver
  * --bind-address：绑定地址
  * --allocate-node-cidrs：允许每个node在pod地址段中划分一个子网
  * --cluster-cidr：pod地址段
  * --service-cluster-ip-range：service地址段
  * --cluster-signing-*：可以为集群范围组件签发证书的ca证书【如kubelet】
  * --root-ca-file：用于验证ServiceAccount token的ca证书
  * --service-account-private-key-file：用于验证ServiceAccount token的ca证书的key
  * --experimental-cluster-signing-duration：集群范围签发证书的有效期

* systemd管理controller-manager

  ```
  cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
  [Unit]
  Description=Kubernetes Controller Manager
  Documentation=https://github.com/kubernetes/kubernetes
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
  ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
  Restart=on-failure
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 启动服务并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kube-controller-manager
  systemctl enable kube-controller-manager
  ```

## scheduler部署

* 创建配置文件

  ```
  cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
  KUBE_SCHEDULER_OPTS="--logtostderr=false \
  --v=2 \
  --log-dir=/opt/kubernetes/logs \
  --leader-elect \
  --master=127.0.0.1:8080 \
  --bind-address=127.0.0.1"
  EOF
  ```

* systemd管理scheduler

  ```
  cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
  [Unit]
  Description=Kubernetes Scheduler
  Documentation=https://github.com/kubernetes/kubernetes
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
  ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
  Restart=on-failure
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 服务启动并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kube-scheduler
  systemctl enable kube-scheduler
  ```

## 查看组件状态

kubectl get cs

# node节点部署

## 创建目录并复制二进制文件

```
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
cd kubernetes/server/bin
cp kubelet kube-proxy /opt/kubernetes/bin   # 本地拷贝
```

## 部署kubelet

* 创建配置文件

  ```
  cat > /opt/kubernetes/cfg/kubelet.conf << EOF
  KUBELET_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --hostname-override=node1 \\
  --network-plugin=cni \\
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
  --bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
  --config=/opt/kubernetes/cfg/kubelet-config.yml \\
  --cert-dir=/opt/kubernetes/ssl \\
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause-amd64:3.0"
  EOF
  ```

  * --hostname-override：在集群中的名称，集群中唯一
  * --network-plugin：启用cni网络插件
  * --kubeconfig：bootstrap认证后由集群生产的连接apiserver配置文件
  * --bootstrap-kubeconfig：首次启动(bootstrap)连接apiserver的配置文件
  * --config：配置参数文件
  * --cert-dir：kubelet证书生成目录
  * --pod-infra-container-image：管理pod网络的容器镜像地址

* 配置参数文件

  ```
  cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
  kind: KubeletConfiguration
  apiVersion: kubelet.config.k8s.io/v1beta1
  address: 0.0.0.0
  port: 10250
  readOnlyPort: 10255
  cgroupDriver: systemd
  clusterDNS:
  - 10.0.0.2
  clusterDomain: cluster.local 
  failSwapOn: false
  authentication:
    anonymous:
      enabled: false
    webhook:
      cacheTTL: 2m0s
      enabled: true
    x509:
      clientCAFile: /opt/kubernetes/ssl/ca.pem 
  authorization:
    mode: Webhook
    webhook:
      cacheAuthorizedTTL: 5m0s
      cacheUnauthorizedTTL: 30s
  evictionHard:
    imagefs.available: 15%
    memory.available: 100Mi
    nodefs.available: 10%
    nodefs.inodesFree: 5%
  maxOpenFiles: 1000000
  maxPods: 110
  EOF
  ```

  * cgroupDriver：docker的cgroups驱动，默认cgroupfs，需要和docker-engin的配置一致

  * clusterDNS：集群dns地址，后续安装的dns插件使用

  * 其他预留资源选项【给系统和 k8s组件预留资源】

    ```
    -system-reserved=cpu=200m,memory=250Mi --kube-reserved=cpu=200m,memory=250Mi \
    --eviction-hard=memory.available<1Gi,nodefs.available<1Gi,imagefs.available<1Gi \
    --eviction-minimum-reclaim=memory.available=500Mi,nodefs.available=500Mi,imagefs.available=1Gi \
    --node-status-update-frequency=10s \
    --eviction-pressure-transition-period=180s
    ```

* 生成bootstrap.kubeconfig

  ```
  KUBE_APISERVER="https://192.168.31.211:6443"
  TOKEN="22972cc804fc8612da9f3c1fc597ac67" #与/opt/kubernetes/cfg/token.csv中一样
  kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=bootstrap.kubeconfig
  kubectl config set-credentials "kubelet-bootstrap" \
    --token=${TOKEN} \
    --kubeconfig=bootstrap.kubeconfig
  kubectl config set-context default \
    --cluster=kubernetes \
    --user="kubelet-bootstrap" \
    --kubeconfig=bootstrap.kubeconfig
  kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
  ```

* 复制证书到配置文件中的路径

  ```
  cp bootstrap.kubeconfig /opt/kubernetes/cfg
  ```

* systemd管理kubelet

  ```
  cat > /usr/lib/systemd/system/kubelet.service << EOF
  [Unit]
  Description=Kubernetes Kubelet
  After=docker.service
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
  ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
  Restart=on-failure
  LimitNOFILE=65536
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 服务启动并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kubelet
  systemctl enable kubelet
  ```

* 批准kubelet申请加入集群

  ```
  kubectl get csr
  kubectl certificate approve node-csr-MQkj9DZylo2JcofJKBAcM_Aye5jL2oWJfpBdptpW03o
  ```

## 部署kube-proxy

* 创建配置文件

  ```
  cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
  KUBE_PROXY_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --config=/opt/kubernetes/cfg/kube-proxy-config.yml"
  EOF
  ```

* 配置参数文件

  ```
  cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
  kind: KubeProxyConfiguration
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  bindAddress: 0.0.0.0
  metricsBindAddress: 0.0.0.0:10249
  clientConnection:
    kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
  hostnameOverride: node1
  clusterCIDR: 10.0.0.0/24
  EOF
  ```

* 生成kube-proxy.kubeconfig

  * 创建证书请求

    ```
    cat > kube-proxy-csr.json << EOF
    {
      "CN": "system:kube-proxy",
      "hosts": [],
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "L": "BeiJing",
          "ST": "BeiJing",
          "O": "k8s",
          "OU": "System"
        }
      ]
    }
    EOF
    ```

  * 生成证书

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
    ls kube-proxy*.pem
    ```

  * 基于证书生成kubeconfig

    ```
    kubectl config set-cluster kubernetes \
      --certificate-authority=/opt/kubernetes/ssl/ca.pem \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=kube-proxy.kubeconfig
    kubectl config set-credentials kube-proxy \
      --client-certificate=./kube-proxy.pem \
      --client-key=./kube-proxy-key.pem \
      --embed-certs=true \
      --kubeconfig=kube-proxy.kubeconfig
    kubectl config set-context default \
      --cluster=kubernetes \
      --user=kube-proxy \
      --kubeconfig=kube-proxy.kubeconfig
    kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
    ```

  * 将证书复制到配置文件路径

    ```
    cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
    ```

* systemd管理kube-proxy

  ```
  cat > /usr/lib/systemd/system/kube-proxy.service << EOF
  [Unit]
  Description=Kubernetes Proxy
  After=network.target
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
  ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
  Restart=on-failure
  LimitNOFILE=65536
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 服务启动并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kube-proxy
  systemctl enable kube-proxy
  ```

## 部署cni网络

* [下载cni插件]( https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz )，并将二进制执行文件移动到默认工作目录

  ```
  mkdir /opt/cni/bin -p
  tar zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin
  ```

* 部署cni网络插件，比如[flannel]( https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml )，资源文件修改如下

  * pod网络修改(net-conf.json)
  * 镜像地址修改
  * 主机间通信节点设置：--iface=eth1

* 网络插件部署好后node即为Ready状态

## 授权apiserver访问kubelet

```
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml
```

## 新增node节点

* 将master上的worker配置文件复制到新增节点上

  ```
  scp -r /opt/{kubernetes,cni} root@192.168.31.212:/opt
  scp -r /opt/{kubernetes,cni} root@192.168.31.213:/opt
  scp /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@192.168.31.212:/usr/lib/systemd/system
  scp /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@192.168.31.213:/usr/lib/systemd/system
  ```

* 删除kubelet证书和kubelet.kubeconfig

  ```
  rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
  rm -f /opt/kubernetes/ssl/kubelet*
  ```

* 修改主机名

  ```
  /opt/kubernetes/cfg/kubelet.conf 
  /opt/kubernetes/cfg/kube-proxy-config.yml
  ```

* 启动服务并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kubelet
  systemctl enable kubelet
  systemctl start kube-proxy
  systemctl enable kube-proxy
  ```

* 在master节点批准新node申请的证书

  ```
  kubectl get csr
  kubectl certificate approve node-csr-JjEj_4TxUHtHYpwZcEIYhcdZd-ykNafmUlrGFxHxN2E
  ```

* 查看节点状态：`kubectl get node`

# 插件部署

## dashboard部署

* [下载dashboard资源]( https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml )并修改

  *  将dashboard的访问端口暴露在宿主机上：containers–》ports–》hostPort: 8443 
  *  dashboard部署在node02上：nodeName: node2

* 建立dashboard用户

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

* 登录token获取：` kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}') `

* web访问： https://192.168.31.212:8443/ 

## CoreDNS部署

* [下载资源文件]( https://github.com/coredns/deployment/tree/master/kubernetes ):
  * 文件1：https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed
  * 文件2：https://github.com/coredns/deployment/blob/master/kubernetes/deploy.sh

* 执行：`bash deploy.sh -r 10.0.0.0/24 -i 10.0.0.2 |kubectl apply -f -`
  * -r：指定service地址段
  * -i：指定dns pod的ip
* 测试：`kubectl run -it --rm dns-test --image=busybox:1.28.4 sh`
  * nslookup kubernetes

# 高可用架构

* 数据存储etcd：采用3节点组建etcd集群
* 控制节点
  * controller-manager、scheduler组件通过自身的选举机制实现高可用【同一时间只有一个组件在工作】
  * apiserver：通过使用nginx+keepalived实现2个以上的apiserver组件的高可用和负载均衡
* 公有云一般不支持keepalived，可以使用它们的负载均衡产品（内网免费）

## 部署新增master节点的组件

* 将master1上的文件复制到master2

  ```
  scp -r /opt/{etcd,cni,kubernetes} root@192.168.31.214:/opt
  scp /usr/lib/systemd/system/kube* root@192.168.31.214:/usr/lib/systemd/system
  scp /usr/bin/kubectl root@192.168.31.214:/usr/bin
  ```

* 删除kubelet证书文件

  ```
  rm -f /opt/kubernetes/cfg/kubelet.kubeconfig
  rm -f /opt/kubernetes/ssl/kubelet*
  ```

* 修改配置文件中的主机名和ip

  ```
  /opt/kubernetes/cfg/kube-apiserver.conf
  /opt/kubernetes/cfg/kubelet.conf
  /opt/kubernetes/cfg/kube-proxy-config.yml
  ```

* 服务启动并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start kube-apiserver
  systemctl start kube-controller-manager
  systemctl start kube-scheduler
  systemctl start kubelet
  systemctl start kube-proxy
  systemctl enable kube-apiserver
  systemctl enable kube-controller-manager
  systemctl enable kube-scheduler
  systemctl enable kubelet
  systemctl enable kube-proxy
  ```

* 查看组件状态：`kubectl get cs`

* 批准新增节点kubelet证书请求

  ```
  kubectl get csr
  kubectl certificate approve node-csr-vHlBaG92ISuIJqPlyisvVsVE12hA8DTCvuuBTFhSkH0
  ```

## 部署nginx+keepalived

* 安装软件（主/备）

  ```
   yum install epel-release -y
   yum install nginx keepalived -y
  ```

* nginx配置文件(主/备)

  ```
  cat > /etc/nginx/nginx.conf << "EOF"
  user nginx;
  worker_processes auto;
  error_log /var/log/nginx/error.log;
  pid /run/nginx.pid;
  
  include /usr/share/nginx/modules/*.conf;
  
  events {
      worker_connections 1024;
  }
  
  # 四层负载均衡，为两台Master apiserver组件提供负载均衡
  stream {
  
      log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
  
      access_log  /var/log/nginx/k8s-access.log  main;
  
      upstream k8s-apiserver {
         server 192.168.31.211:6443;   # Master1 APISERVER IP:PORT
         server 192.168.31.214:6443;   # Master2 APISERVER IP:PORT
      }
      
      server {
         listen 16443;
         proxy_pass k8s-apiserver;
      }
  }
  
  http {
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
  
      access_log  /var/log/nginx/access.log  main;
  
      sendfile            on;
      tcp_nopush          on;
      tcp_nodelay         on;
      keepalive_timeout   65;
      types_hash_max_size 2048;
  
      include             /etc/nginx/mime.types;
      default_type        application/octet-stream;
  
      server {
          listen       80 default_server;
          server_name  _;
  
          location / {
          }
      }
  }
  EOF
  ```

* keepalived配置文件(主/备)

  >主备router_id、state、priority不同

  ```
  cat > /etc/keepalived/keepalived.conf << EOF
  global_defs {
     notification_email {
       acassen@firewall.loc
       failover@firewall.loc
       sysadmin@firewall.loc
     }
     notification_email_from Alexandre.Cassen@firewall.loc
     smtp_server 127.0.0.1
     smtp_connect_timeout 30
     router_id NGINX_MASTER
  }
  vrrp_script check_nginx {
      script "/etc/keepalived/check_nginx.sh"
  }
  vrrp_instance VI_1 {
      state MASTER
      interface eth1   #虚拟ip绑定及心跳检测通信接口
      virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的
      priority 100    # 优先级，备服务器设置 90
      advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      # 虚拟IP（VIP）
      virtual_ipaddress {
          192.168.31.216/24
      }
      # 根据脚本执行结果判断是否执行故障转移【0为正常，非0为不正常】
      track_script {
          check_nginx
      }
  }
  EOF
  ```

* 检查nginx状态脚本

  ```
  cat > /etc/keepalived/check_nginx.sh  << "EOF"
  #!/bin/bash
  count=$(ps -ef |grep nginx |egrep -cv "grep|$$")
  
  if [ "$count" -eq 0 ];then
      exit 1
  else
      exit 0
  fi
  EOF
  chmod +x /etc/keepalived/check_nginx.sh
  ```

* 服务启动并设置开机启动

  ```
  systemctl daemon-reload
  systemctl start nginx
  systemctl start keepalived
  systemctl enable nginx
  systemctl enable keepalived
  ```

* 检测keepalived工作状态（接口是否绑定vip）：`ip a`

* nginx+keepalived高可用测试：`pkill nginx`执行后查看vip是否绑定到了备机上

* 使用vip进行负载均衡功能测试：`curl -k https://192.168.31.211:16443/version` 

## 修改node组件连接地址

```
sed -i 's/192.168.31.211:6443/192.168.31.216:16443/g' /opt/kubernetes/cfg/{kubelet.kubeconfig,kube-proxy.kubeconfig}
systemctl restart kubelet
systemctl restart kube-proxy
```

# 制作kubectl使用的kubeconfig文件
> 基于kube-dashboard的admin-user

```
kubectl config set-cluster kubernetes --embed-certs=true --server="https://192.168.31.216:16443" --certificate-authority=/opt/kubernetes/ssl/ca.pem --kubeconfig=./kubeconfig
ADMIN_TOKEN=$(kubectl get secret $(kubectl get secret -n kubernetes-dashboard|awk '/admin-user/{print $1}') -n kubernetes-dashboard -o jsonpath={.data.token}|base64 -d)
kubectl config set-credentials kubernetes-dashboard:admin-user --token=${ADMIN_TOKEN} --kubeconfig=./kubeconfig
kubectl config set-context admin-user@kubernetes --cluster=kubernetes --user=kubernetes-dashboard:admin-user --kubeconfig=./kubeconfig
kubectl config use-context admin-user@kubernetes --kubeconfig=./kubeconfig
```




