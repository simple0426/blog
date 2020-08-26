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
kubernetes要求所有的网络插件必须满足以下要求

* 一个pod一个ip
* 所有pod可以和任意其他pod通信，无需使用NAT映射
* 所有节点可以与所有pod直接通信，无需使用NAT映射
* pod内部获取到的IP地址与其他pod或节点通信时使用的IP地址相同

## k8s网络应用场景

* pod内不同容器间：同一个pod内的所有容器共享同一个网络命名空间，由构建pod对象的infra containers提供；__由linux的网络命名空间这一特性实现__
* pod之间通信：
  * 同主机：多个veth设备对包含在一个网桥中，通过桥接方式连接多个pod；__使用docker网络模型__
  * 跨主机：通过路由(underlay)或隧道(overlay)方式连接不同主机pod；__由网络插件(CNI)提供的功能实现__
* pod与service通信：k8s将service的cluster-ip到pod-ip的映射转换为相应节点的iptables、ipvs规则，从而实现service到pod的通信；__由k8s的kube-proxy组件实现__
* 互联网访问service：__由k8s的kube-proxy组件实现__，根据访问类型不一样实现机制不同
* pod访问互联网：__使用docker的网络模型__
  * 每个容器通过veth(虚拟以太网链接对)和宿主机连通
  * 多个veth包含在一个bridge(桥接网络)中，因此多个容器可以互通
  * 桥接网卡(一般为docker0)和宿主机出口(例如eth0)通过iptables的nat转换，从而可以让容器连接互联网

## CNI

CNI（Container Network Interface，容器网络接口)：是一个容器网络规范；kubernetes网络就是采用这个CNI规范，CNI实现依赖两种插件：

* 一种是基础功能插件：负责容器连接到主机
* 一种是IPAM类插件：负责配置容器网络命名空间的网络

项目地址：https://github.com/containernetworking/cni

pod网络创建流程

* kubelet与docker通信（/var/run/docker.sock）
* 调用dockershim创建一个infra容器
* 调用CNI插件为infra容器配置网络

## 启用CNI功能

* 安装基础功能插件：用于容器连接主机

  * 二进制方式：https://github.com/containernetworking/plugins/releases

  * yum方式：kubernetes-cni

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

* 插件默认目录位置
  * 二进制文件目录：/opt/cni/bin
  * 配置文件目录：/etc/cni/net.d【ipam类插件(calico、flannel)启动后会在目录下生成插件的配置文件】

* kubelet启用cni设置

  ```
   --network-plugin=cni \
   --cni-conf-dir=/etc/cni/net.d \
   --cni-bin-dir=/opt/cni/bin
  ```

## 常见CNI插件

> IPAM类插件：负责配置容器网络命名空间的网络

* flannel：一个为kubernetes提供overlay(叠加网络)的插件，它基于linux tun/tap，使用udp封装ip报文来创建叠加网络，并借助etcd维护网络分配情况；不支持网络策略
* calico：一个基于BGP的三层网络插件，并且支持网络策略(network policy)实现访问控制；它在每台机器上运行一个vRouter，利用Linux内核来转发网络数据包，并借助iptables实现防火墙功能
* canal：包含flannel和calico功能的插件，支持网络策略
* weavenet：多主机容器网络方案，采用UDP封装实现L2 Overlay，支持用户态(慢，可加密)/内核态(快，不能加密)两种方式
* kube-router：kubernetes提供的网络解决方案，采用BGP提供网络直连，集成基于LVS的负载均衡能力；支持网络策略

# flannel

Flannel是CoreOS维护的一个网络组件，Flannel为每个Pod提供全局唯一的IP，Flannel使用ETCD来存储Pod子网与Node IP之间的关系。flanneld守护进程在每台主机上运行，并负责维护ETCD信息和路由数据包。

项目地址：https://github.com/coreos/flannel

## 工作模式

- udp：性能较低，是flannel的早期实现方式，仅适用于vxlan和host-gw不可用的情况【已废弃】
- vxlan：overlay网络(隧道)方案；使用内核vxlan模块封装报文
- host-gw：underlay(路由)网络方案；Node节点把自己的网络接口当做pod的网关使用；flannel通过在各个节点的agent，将容器网络信息刷新到主机路由表上，这样一来所有的主机都有整个容器网络的路由数据了。但是，要求所有节点处于同一个二层网络
- Directrouting(vxlan+host-gw)：vxlan的直接路由模式，兼具vxlan和host-gw的优势，既保证了传输性能，又具备了跨二层网络转发报文的能力；配置：`"Backend":{"Type":"VxLAN", "Directrouting": true}`

公有云VPC：Alivpc、AWSvpc、Alloc、GCE

### VXLAN模式通信流程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/flanneld-vxlan.png)

* vxlan模式特点：
  * 支持宿主机3层网络通信（不同子网），只需宿主机网络互通，不需要单独配置到容器网络的路由
  * 传输过程中需要封包、解包，有性能开销
* 跨主机pod互联：

1. 容器到宿主机网桥cni0【同主机不同pod通过cni0网桥通信】
2. 网桥cni0到隧道接口flannel.1
3. 本地flannel.1到目标flannel.1，通信实现如下：
   1. 获取目标flannel.1 ip地址：ip route
   2. 根据flannel.1ip地址获取flannel.1 mac地址：ip neigh show dev flannel.1
   3. 根据flannel.1 mac地址获取flannel.1 所在宿主机ip：bridge fdb show dev flannel.1
   4. 根据flannel.1宿主机ip地址获取flannel.1所在宿主机mac地址：arp -a

### Host-GW模式通信流程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/flanneld-hostgw.png)

* host-gw模式：
  * 要求宿主机2层通信（相同子网，宿主机可以直接通过mac地址通信），否则需要在路由器配置到容器网络的静态路由
  * 不存在封包、解包过程，传输更高效

* 跨主机pod互联：host-gw模式相比vxlan简单了许多， 直接在主机添加路由，将目的主机当做网关，直接路由原始封包。

## 部署

### k8s资源清单方式

* 下载资源清单：https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

* 配置调整

  - 网络设置(net-conf.json)

    - 配置pod地址段，与controller-manager中的cluster-cidr配置一致
    - 配置工作模式

    ```
    net-conf.json: |
        {
          "Network": "10.244.0.0/16",
          "Backend": {
            "Type": "vxlan"
          }
        }
    ```

  - 宿主机间的通信接口设置(kube-flannel-->DaemonSet)：`--iface=eth1`

  - 镜像地址修改

* 应用资源文件

### 手动etcd方式

* 安装并配置etcd
* etcd中创建flannel网络配置
    1. etcdctl mkdir /kube-centos/network
    2. etcdctl mk /kube-centos/network/config '{"Network":"172.33.0.0/16","SubnetLen":24,"Backend":{"Type":"host-gw"}}'
    3. 参数详解：
        - Network：全局使用CIDR格式ipv4网络
        - SubnetLen：节点使用的子网的掩码
        - Backend：flannel要使用的后端类型
* 安装flannel
* flannel服务配置
```
FLANNEL_ETCD_ENDPOINTS="http://172.17.8.101:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
FLANNEL_OPTIONS="-iface=eth1"
```
* 启动flannel

# calico

calico是一个纯三层的数据中心网络方案，calico支持广泛的平台，包含kubernetes、OpenStack

calico在每一个计算节点利用linux kernel实现了一个高效的虚拟路由器(vRouter)来负责数据转发，而每个vRouter通过BGP协议负责把自己运行的workload的路由信息向整个calico网络内传播

calico项目还实现了kubernetes网络策略，提供了ACL功能

calico也通过两种方式进行数据包转发

* 路由模式：BGP，动态路由协议，适合大规模网络
* 隧道模式：ip-ip

在不支持ip-ip封装的网络环境，也支持vxlan封装：https://docs.projectcalico.org/getting-started/kubernetes/installation/config-options#switching-from-ip-in-ip-to-vxlan

## [calico架构组件](https://docs.projectcalico.org/reference/architecture/overview)

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/calico.png)



* Felix：在每个节点运行，负责维护宿主机上路由规则和ACL规则
* Etcd：分布式键值存储，保存Calico的策略和网络配置状态。
* BIRD：BGP客户端，负责分发路由信息到整个calico网络
* BGP Route Reflector (BIRD)：可选的BGP路由反射器，为了实现更高的规模【集中管理路由信息】

## calico部署

### [部署](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises)

* 资源下载：

  * k8s-api存储网络信息：`curl https://docs.projectcalico.org/manifests/calico.yaml -O`

  * etcd存储网络信息：`curl https://docs.projectcalico.org/manifests/calico-etcd.yaml -O`

* [自定义配置](https://docs.projectcalico.org/getting-started/kubernetes/installation/config-options)：

  * [使用指定的网卡用于宿主机间通信](https://docs.projectcalico.org/networking/ip-autodetection#change-the-autodetection-method)

    ```
    DaemonSet/calico-node==>containers/env
    - name: IP_AUTODETECTION_METHOD
      value: "interface=eth1"
    ```

  * 配置pod地址段：CALICO_IPV4POOL_CIDR【kubeadm安装的集群可以自动检测】

  * 工作模式：CALICO_IPV4POOL_IPIP

    * IPIP：Always【默认】
    * BGP：Never 
    * IPIP兼容BGP：CrossSubnet【同一子网使用BGP，跨子网使用IPIP】

  * 配置etcd信息（secret、ConfigMap）

### 切换flannel到calico

* 删除flannel信息

  * 删除路由信息

    ```
    ip route
    ip route del 10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
    ip route del 10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
    ip route del 10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
    ip route del 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
    ```

  * 删除网桥

    ```
    ip link delete cni0 
    ip link delete flannel.1
    ip link delete docker0
    ```

* 部署calico

* 重建应用pod【过滤使用宿主机网络的pod】

  ```
  kubectl get pod -A -o wide|grep -v "192.168.31"|awk 'system("kubectl delete pod "$2" -n "$1"")' 
  ```

## [calico管理工具](https://docs.projectcalico.org/getting-started/clis/calicoctl/)

* 二进制下载地址：https://github.com/projectcalico/calicoctl/releases

* [配置文件](https://docs.projectcalico.org/getting-started/clis/calicoctl/configure/)：/etc/calico/calicoctl.cfg

  * 使用独立etcd存储

    ```
    apiVersion: projectcalico.org/v3
    kind: CalicoAPIConfig
    metadata:
    spec:
      datastoreType: etcdv3
      etcdEndpoints: https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
      etcdKeyFile: /etc/calico/key.pem
      etcdCertFile: /etc/calico/cert.pem
      etcdCACertFile: /etc/calico/ca.pem
    ```

  * 使用k8s-api存储

    ```
    apiVersion: projectcalico.org/v3
    kind: CalicoAPIConfig
    metadata:
    spec:
      datastoreType: "kubernetes"
      kubeconfig: "/root/.kube/config"
    ```

* 管理命令
  
  * 查看calico节点状态：calicoctl node status

## BGP原理

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/calico-bgp.png)

pod1访问pod2流程：

1. 数据包从容器1的eth0到达veth pair另一端cali34

   ```
   设置宿主机的接口cali34开启proxy_arp功能，此后这一接口会响应所有arp请求
   从而将容器发送到默认网关169.254.1.1的流量送达宿主机的cali34接口【告诉容器，我有169.254.1.1这个ip】
   ```

2. 宿主机根据路由规则，将数据包转发给下一条

   ```
   10.244.186.192/26 via 192.168.31.203 dev eth1 proto bird            # 到达其他pod网段的数据包(每个宿主机一个pod子网段)：下一跳为其他宿主机ip、本地出口为eth1
   blackhole 10.244.196.128/26 proto bird        # 到达本机pod网段的数据包，如果没有更高优先级路由，数据包直接丢弃
   10.244.196.129 dev calie8e9a5cf531 scope link   # 目的为本机pod网段特定ip的数据包，交给指定的cali网卡
   ```

3. 数据包到达node2，根据路由规则将数据包转发给cali设备

## BGP-Route Reflector模式

https://docs.projectcalico.org/master/networking/bgp 

Calico 维护的网络在默认是（Node-to-Node Mesh）全互联模式，Calico集群中的节点之间都会相互建立连接，用于路由交换。但是随着集群规模的扩大，mesh模式将形成一个巨大服务网格，连接数成倍增加。

这时就需要使用 Route Reflector（路由器反射）模式解决这个问题。

确定一个或多个Calico节点充当路由反射器，让其他节点从这个RR节点获取路由信息。

* 关闭BGP node-to-node模式

  ```
   cat << EOF | calicoctl create -f -
   apiVersion: projectcalico.org/v3
   kind: BGPConfiguration
   metadata:
     name: default
   spec:
     logSeverityScreen: Info
     nodeToNodeMeshEnabled: false  
     asNumber: 64512
  EOF
  ```

  asNumber可以通过获取：calicoctl get node -o wide

* 配置指定节点为充当路由反射器

  * 节点添加标签：

    ```
    kubectl label node node02 route-reflector=true
    ```

  * 配置路由器反射器节点routeReflectorClusterID【一般为未使用过的ip地址】

    > calicoctl get node node02 -o yaml > rr.yaml

    ```
    spec:
      bgp:
        routeReflectorClusterID: 244.0.0.1
    ```

  * 使用标签选择器将路由反射器节点与其他非路由反射器节点配置为对等

    ```
    cat << EOF | calicoctl create -f -
    kind: BGPPeer
    apiVersion: projectcalico.org/v3
    metadata:
      name: peer-with-route-reflectors
    spec:
      nodeSelector: all()
      peerSelector: route-reflector == 'true'
    EOF
    ```

* 查看bgp连接状态【只能检查本地代理与反射器的连接关系】

  ```
  calicoctl node status
  ```



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

