---
title: kubernetes-网络模型和策略控制
tags:
  - cni
  - flannel
  - canal
  - calico
  - ingress
categories:
  - kubernetes
date: 2020-03-03 23:54:17
---

# kubernetes网络模型
* 要求所有容器都处于同一个ip网络中
* 集群中的ip地址分配是以pod为单位的，同一个pod内的所有容器共享同一个网络命名空间

## 实现目标
* pod内容器间通信：一个pod内的所有容器共享同一个网络名称空间，它由构建pod对象的infra containers提供；
* pod之间通信：通过桥接方式连通多个网络名称空间(pod),也是各网络插件(flannel/calico)解决的问题，包含叠加网络模型(overlay)和路由网络模型(underlay)
* pod与service间通信：k8s将service的cluster-ip到pod-ip的映射转换为相应节点的iptables、ipvs规则，从而实现service到pod的通信
* service与集群外部通信：k8s根据service的代理类型转换为相应节点的iptables或ipvs规则，从而实现外部流量到pod的通信

## CNI插件
* CNI插件连接容器管理系统和网络插件
* CNI插件分类
    - main：实现特定的网络功能，如：bridge、loopback
    - meta：调用其他插件，如：flannel、calico
    - ipam：分配ip地址，如：dhcp

## CNI插件安装
* 安装cni插件集成包：kubernetes-cni，cni插件在/opt/cni/bin目录下
* 部署calico、flannel时会在/etc/cni/net.d/下生成插件的配置文件
* kubelet启动后(--network-plugin=cni)会加载cni插件及其配置文件
* 如果kubelet开启cni后，缺少部分插件【failed to find plugin "portmap" in path [/opt/cni/bin]]】，则可以通过安装包补全插件
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
 
yum clean all
yum install kubernetes-cni -y
```

## 常见CNI插件对应的项目
* flannel：一个为kubernetes提供overlay(叠加网络)的插件，它基于linux tun/tap，使用udp封装ip报文来创建叠加网络，并借助etcd维护网络分配情况；不支持网络策略
* calico：一个基于BGP的三层网络插件，并且支持网络策略(network policy)实现访问控制；它在每台机器上运行一个vRouter，利用Linux内核来转发网络数据包，并借助iptables实现防火墙功能
* canal：包含flannel和calico功能的插件，支持网络策略
* weavenet：多主机容器网络方案，采用UDP封装实现L2 Overlay，支持用户态(慢，可加密)/内核态(快，不能加密)两种方式
* kube-router：kubernetes提供的网络解决方案，采用BGP提供网络直连，集成基于LVS的负载均衡能力；支持网络策略

# flannel
## 网络模型/后端
- vxlan：使用内核vxlan模块封装报文
- host-gw：直接在节点创建到目标容器的路由，要求各节点处于同一个二层网络
- udp：性能较低，仅适用于前两者不可用的情况【已废弃】
- 云厂商类型：Alivpc、AWSvpc、Alloc、GCE

## vxlan后端
* vxlan(virtual extentsible local area network)：是vlan扩展方案草案，采用的是MAC in UDP封装方式，是NVo3(network virtualization over layer3)中的一种网络虚拟化技术
* vxlan实现：将虚拟网络的数据帧添加到vxlan首部后，封装在物理网络的udp报文中，然后以传统的通信方式传送该udp报文，待其到达目的主机后，去掉物理网络报文的头部信息以及vxlan首部，然后将报文交付给目的终端
* Directrouting：vxlan的直接路由模式，兼具vxlan后端和host-gw后端的优势，既保证了传输性能，又具备了跨二层网络转发报文的能力
    - 配置样例："Backend":{"Type":"VxLAN", "Directrouting": true}

## 部署-手动etcd方式
* 安装并配置etcd
* etcd中创建flannel网络配置
    1. etcdctl mkdir /kube-centos/network
    2. etcdctl mk /kube-centos/network/config '{"Network":"172.33.0.0/16","SubnetLen":24,"Backend":{"Type":"host-gw"}}'
    3. 参数详解：
        - Network：全局使用CIDR格式ipv4网络
        - SubnetLen：节点使用的子网的掩码
        - Backend：flannel要使用的后端类型
* 安装flannel
* flannel服务配置(centos示例-/etc/sysconfig/flanneld)
```
FLANNEL_ETCD_ENDPOINTS="http://172.17.8.101:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
FLANNEL_OPTIONS="-iface=eth1"
```
* 启动flannel

## 部署-k8s资源清单方式
* [资源清单](https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml)
* 配置调整
    - 网络设置(net-conf.json)
    - 主机间的内部通信端口设置(kube-flannel-->DaemonSet)：`--iface=eth1`

# 基于calico的网络策略部署
## 概要
主要通过使用flannel(pod间通信)和calico(pod间访问控制)两个软件实现  
为了充分利用flannel的易用性和calico的丰富功能，可以通过以下方式部署实现(不含calico单独部署)  

* [只部署canal](#canal部署)【canal是一个整合了flannel和calico的综合项目，其中calico用于策略控制、flannel用于网络通信】
* [协同部署](#flannel和calico协同部署)flannel实现网络功能、部署calico实现访问控制功能

## 部署要求
* flannel的后端必须是vxlan模型【host-gw、vxlan-Directrouting均不行】
* kubelet必须开启cni支持：--network-plugin=cni
* kube-controller-manager开启配置：`-cluster-cidr=<your-pod-cidr>`和--allocate-node-cidrs=true
* 其他要求：[官网](https://docs.projectcalico.org/getting-started/kubernetes/requirements)

## [canal部署](https://docs.projectcalico.org/getting-started/kubernetes/installation/flannel)
* 下载清单文件
    - etcd存储：curl https://docs.projectcalico.org/manifests/canal-etcd.yaml -O
    - k8s存储：curl https://docs.projectcalico.org/manifests/canal.yaml -O
* 更改配置：
```
POD_CIDR="<your-pod-cidr>"
sed -i -e "s?10.244.0.0/16?$POD_CIDR?g" canal.yaml
```
* 执行清单创建资源：kubectl apply -f canal.yaml

## flannel和calico协同部署
### [flannel部署](https://github.com/coreos/flannel)
* 下载清单文件:部署：curl https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml -o
* 配置调整
    - 网络设置(net-conf.json)
    - 主机间的内部通信端口设置(kube-flannel-->DaemonSet)：--iface=eth1
* 执行清单创建资源：kubectl apply -f kube-flannel.yml

### [calico部署](https://docs.projectcalico.org/getting-started/kubernetes/installation/other)
* 下载清单文件：curl https://docs.projectcalico.org/manifests/calico-policy-only.yaml -O
* 更改配置
```
POD_CIDR="<your-pod-cidr>"
sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico.yaml
```
* 执行清单创建资源：kubectl apply -f calico.yaml

# 网络策略控制
控制pod到pod、node、外接网络的访问限制，包含流入(Ingress)和流出(Egress)两个方向的控制规则
## 默认规则
* 默认情况下，所有pod处于非隔离状态，所有方向的流量都可以自由流动
* 一旦有策略应用于pod，那么所有未明确声明允许的流量都将被拒绝
* 如果在policyTypes中定义了生效规则(Ingress/Egress)：
  - spec定义了空值【`{}`】表示不限制相关方向的访问
  - spec中没有定义相应的Ingress或Egress规则，则表示拒绝相关方向上的一切流量。范例如下【定义默认规则】
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-traffic
  namespace: testing
spec:
  podSelector: {} #所有pod
  policyTypes: # 进口、出口流量都管控；没有定义出站(Egress)详细规则，则拒绝所有出站流量
    - Ingress
    - Egress
  Ingress: #定义了入站规则，但是为空，表示允许所有入站流量
    - {}
```

## 语法定义
* podSelector：通过pod选择器选定的一组pod资源，它是规则生效的对象
    - matchLabel、matchExpression
* Egress：出站规则
    - to：目标对象
    - ports：目标对象端口
* Ingress：入站规则
    - from：源对象
    - ports：本地pod资源端口
* ports：tcp或udp端口
    - protocol：默认tcp
    - port：端口
* to/from：目标对象或源对象
    - ipBlock：IP地址块
    - namespaceSelector：名称空间选择器
    - podSelctor：pod选择器

## Ingress
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-myapp-ingress
  namespace: default
spec:
  podSelector: #选择策略生效目标
    matchLabels:
      app: myapp
  policyTypes: ['Ingress'] #策略管控的流量方向，此处为入站流量(Ingress)
  ingress:
    - from: #入站来源
      - ipBlock: #ip选择
          cidr: 10.244.0.0/16
          except: #排除部分选项
            - 10.244.3.0/24
      - podSelector: # 标签选择
          matchLabels:
            app: myapp
      ports: #目标端口
      - protocol: TCP
        port: 80
```

* from:入站流量来源
    - 多个项目之间为“逻辑或”关系
    - 若未设置此字段或字段为空，则允许一切来源
    - 如果设置至少一个值，则设置的值为白名单，只有名单中的地址的流量才被放行
* ports：入站流量的目标端口，其他说明如上
* from和ports之间为“逻辑与”关系；仅定义from隐含本地pod资源的所有端口；仅定义ports隐含所有来源；两者都定义则精确匹配来源及目标端口

## Egress
对于具有app=tomcat标签的对象，它到【app=nginx】对象80端口的流量被放行  
到【app=mysql】对象3306端口的流量也被放行。

```
apiVersion: networking.k8s.io/v1
kind: NetworPolicy
metadata:
  name: allow-tomcat-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: tomcat
  policyTypes: ["Egress"]
  egress:
    - to:
      - podSelector:
          matchLabels:
            app: nginx
      ports:
        - protocol: TCP
          port: 80
    - to:
      - podSelector:
          matchLabels:
            app: mysql
      ports:
        - protocol: TCP
          port: 3306
```
## 名称空间隔离
允许当前名称空间内部各pod之间，以及当前名称空间与kube-system名称空间内各pod的通信

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-deny-all
  name: testing
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-allow
  name: testing
spec:
  policyTypes: ["Ingress", "Egress"]
  podSelector: {}
  ingress:
    - from:
      - namespaceSelector:
          matchExpressions:
            - key: name
              operator: In
              values: ["testing", "kube-system"]
  Egress:
    - to:
      - namespaceSelector:
          matchExpressions:
            - key: name
              operator: In
              values: ["testing", "kube-system"]
```

# 网络策略应用案例
## 实现目标
* testing名称空间下运行app=nginx和app=myapp两个应用
* myapp仅允许来自nginx访问其TCP/80端口，但可以向nginx的所有端口发出出站流量
* nginx允许所有源站点访问其TCP/80端口，并能向任意端点发出出站流量
* myapp和nginx都可以与kube-system名称空间内的任意pod通信

## 运行应用
* myapp：`kubectl run myapp --image=ikubernetes/myapp:v1 --replicas=1 --namespace=testing --port 80 --expose --labels app=myapp`
* nginx：`kubectl run nginx --image=nginx:alpine --replicas=1 --namespace=testing --port 80 --expose --labels app=nginx`
* 调试客户端：`kubectl run debug --namespace=default --rm -it --image=nicolaka/netshoot -- sh`
* 为kube-system添加标签：`kubectl label namespace kube-system ns=kube-system`

## 配置清单
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: testing-default # testing默认策略，拒绝所有进出流量
  namespace: testing
spec:
  podSelector: {} 
  policyTypes: 
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-policy # myapp策略
  namespace: testing
spec:
  policyTypes:
    - Ingress
    - Egress
  podSelector:
    matchLabels:
      app: myapp
  ingress:
    - from: #允许来自nginx访问其80端口
      - podSelector:
          matchLabels:
            app: nginx
      ports:
        - port: 80
    - from: #允许来自kube-system的pod访问
      - namespaceSelector:
          matchLabels:
            ns: kube-system
  egress:
    - to: #允许访问nginx的所有端口
      - podSelector:
          matchLabels:
            app: nginx
    - to: # 允许访问kube-system的pod
      - namespaceSelector:
          matchLabels:
            ns: kube-system
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-policy #nginx策略
  namespace: testing
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports: #允许所有源站点访问80端口
      - port: 80
    - from: # 允许来自kube-system访问
      - namespaceSelector:
          matchLabels:
            ns: kube-system
  egress: 
    - {} #出口流量不限制
```
