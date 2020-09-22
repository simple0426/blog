---
title: 服务网格-Istio
tags:
  - istio
  - servicemesh
  - 灰度发布
  - 金丝雀发布
categories:
  - kubernetes
date: 2020-09-22 20:58:31
---


# ServiceMesh

Service Mesh 的中文译为 “服务网格” ，是一个用于处理服务与服务之间**通信的基础设施层**，它负责为构建复杂的云原生应用**传递可靠的网络请求**。在实践中，它是一组和应用服务部署在一起的轻量级的**网络代理**，并且对应用服务**透明**。

## 优点

服务网格带来了巨大变革并且拥有其强大的技术优势，被称为第二代“微服务架构”。

第一，微服务治理与业务逻辑的解耦。服务网格把 SDK 中的**大部分**能力从应用中剥离出来，拆解为独立进程，以 sidecar的模式进行部署。服务网格通过将服务通信及相关管控功能从业务程序中分离并下沉到基础设施层，使其和业务系统完全解耦，使开发人员更加专注于业务本身。

第二，异构系统的统一治理。通过将主体的服务治理能力下沉到基础设施，多语言的支持就轻松很多了。

此外，服务网格相对于传统微服务框架，还拥有三大技术优势：

* 可观察性。因为服务网格是一个专用的基础设施层，所有的服务间通信都要通过它，所以它在技术堆栈中处于独特的位置，以便在服务调用级别上提供统一的监控指标。
* 流量控制。通过Service Mesh，可以为服务提供智能路由（蓝绿部署、金丝雀发布、A/B test）、超时重试、熔断、故障注入、流量镜像等各种控制能力。

* 安全。服务网格的安全相关的好处主要体现在以下三个核心领域：服务的认证、服务间通讯的加密、安全相关策略的强制执行。

## 局限性

- 增加了复杂度。服务网格将 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理和其它组件引入到已经很复杂的分布式环境中，会极大地增加整体链路和操作运维的复杂性。
- 运维人员需要更专业。在容器编排器（如 Kubernetes）上添加 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 之类的服务网格，通常需要运维人员成为这两种技术的专家，以便充分使用二者的功能以及定位环境中遇到的问题。
- 延迟。从链路层面来讲，服务网格是一种侵入性的、复杂的技术，会为系统调用增加显著的延迟。这个延迟是毫秒级别的，但是在特殊业务场景下，这个延迟可能也是难以容忍的。
- 平台的适配。服务网格的侵入性迫使开发人员和运维人员适应高度自治的平台并遵守平台的规则。

# [Istio](https://istio.io/)

Istio是Service Mesh的落地产品

## 功能

* 流量管理(connect)：负载均衡、灰度发布、流量管理
* 安全(security)：认证、授权、证书管理
* 策略控制(policy)【不建议使用/功能逐步启用】：限流、ACL
* 可观察性(observe)：监控、调用链

## 与k8s的区别

k8s（部署运维）：应用部署、资源共享、资源隔离、弹性伸缩

Istio（服务治理）：流量管控、熔断限流、动态路由、链路追踪

## 架构

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 的架构由两部分组成，分别是数据平面（[Data Plane](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#data-plane)）和控制平面（[Control Plane](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#control-plane)）。

**数据平面**

由整个网格内的 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理组成，这些代理以 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 的形式和应用服务一起部署。每一个 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 会接管进入和离开服务的流量，并配合控制平面完成流量控制等方面的功能。可以把数据平面看做是网格内 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理的网络拓扑集合。

**控制平面**

顾名思义，就是控制和管理数据平面中的 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理，完成配置的分发、服务发现、和授权鉴权等功能。架构中拥有控制平面的优势在于，可以统一的对数据平面进行管理。在 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 1.5 版本中，控制平面由原来分散的、独立部署的几个组件整合为一个单体结构 istiod，变成了一个单进程、多模块的组织形态。

![](https://www.servicemesher.com/istio-handbook/images/istio-mesh-arch.png)

主要实现组件如下：

### Envoy

Istio使用Envoy代理的扩展版本。Envoy是使用C ++开发的高性能代理，网格内服务发送和接收的所有流量（data plane流量）都经由 Envoy 代理。Envoy代理是与数据平面流量交互的唯一Istio组件。Envoy代理被部署为服务的辅助工具，通过Envoy的许多内置功能在逻辑上增强了服务，例如：

- Dynamic service discovery：动态服务发现
- Load balancing：负载均衡
- TLS termination：tls加密通信
- HTTP/2 and gRPC proxies：http/2和grpc代理
- Circuit breakers：断路器
- Health checks：健康检查
- Staged rollouts with %-based traffic split：基于流量百分比切分的概率发布
- Fault injection：故障注入
- Rich metrics：丰富的指标数据

### Istiod

整合旧版本pilot、citadel、galley的功能，以单进程、多模块的组织形态提供**服务发现、配置管理、证书管理**功能

# 核心功能-流量管理

为了在网格中导流，Istio 需要知道所有的 endpoint 在哪以及属于哪个服务。为了填充自己的服务注册表，Istio连接到服务发现系统。例如，如果您在 Kubernetes 集群上安装了 Istio，那么它将自动检测该集群中的service和endpoint。

## VirtualService

用户指定的目标或是路由规则设定的目标，定义将流量如何路由到给定目标地址；位于k8s中service的上一层或同一层，匹配服务网格内对service或指定域名、ip的访问。

* hosts可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称）；hosts 字段实际上不必是 Istio 服务注册的一部分，它只是虚拟的目标地址；可以使用通配符（“\*”）前缀，创建一组匹配所有服务的路由规则

* http字段(使用tcp、tls来匹配tcp、tls流量)包含了虚拟服务的路由规则：
  * match字段用来描述访问hosts的HTTP/1.1、HTTP2 和 gRPC 等流量的特征；可以在流量端口、header 字段、URI 等内容上设置匹配条件。
  * route字段包含了指定的请求要流向哪个目标地址
  * destination的host必须是存在于 Istio 服务注册中心的实际目标地址；可以是一个有代理的service，或者是一个通过Service Entry被添加进来的非网格服务。

路由规则优先级：**路由规则**按从上到下的顺序选择，虚拟服务中定义的第一条规则有最高优先级。本示例中，不满足第一个路由规则的流量均流向一个默认的目标，该目标在第二条规则中指定。因此，第二条规则没有 match 条件，直接将流量导向 v3 子集。

更多的路由路由规则：您可以使用 AND 向同一个 `match` 块添加多个匹配条件，或者使用 OR 向同一个规则添加多个 `match` 块。对于任何给定的虚拟服务也可以有多个路由规则。

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:                            
  - reviews
  http:                              
  - match:                           
    - headers:
        end-user:
          exact: jason
    route:                           
    - destination:
        host: reviews                 # service名称
        subset: v2                    # DestinationRule中定义的service下的不同工作负载(deployment/pod)
  - route:
    - destination:
        host: reviews
        subset: v3
```

## DestinationRule

定义目标(service)的不同工作负载(deployment)、目标(service)到工作负载的负载均衡、tls安全、连接池等策略；位于service和pod中间的一层，实现的是kube-proxy的功能；

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc             # service名称
  trafficPolicy:           # 负载均衡策略
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1               # service的其中一个的工作负载名称
    labels:                # 这个工作负载基于deployment/pod标签名称定义
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

## Gateway

管理进出网格的流量。网关被配置为运行在网格边界的独立 Envoy 代理（`istio-ingressgateway` 和 `istio-egressgateway`），而不是服务工作负载的 sidecar 代理。和k8s中的ingress功能类似，部署时使用LoadBalancer类型的service资源，如果没有对接云厂商LB，可以使用NodePort访问。

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:             # 在网关处，匹配来自外部访问https://ext-host.example.com的流量
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:                
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:                  # 在虚拟服务处，匹配来自网关ext-host-gwy中访问ext-host.example.com的流量
  hosts:
  - ext-host.example.com
  gateways:  
    - ext-host-gwy
```

## Service Entry

添加一个入口到 Istio 内部维护的服务注册中心。添加了服务入口后，Envoy 代理可以向服务发送流量，就好像它是网格内部的服务一样。配置服务入口允许您管理运行在网格外的服务的流量。，它包括以下几种能力：

- 为外部目标 redirect 和转发请求，例如来自 web 端的 API 调用，或者流向遗留老系统的服务。
- 为外部目标定义[重试](https://istio.io/latest/zh/docs/concepts/traffic-management/#retries)、[超时](https://istio.io/latest/zh/docs/concepts/traffic-management/#timeouts)和[故障注入](https://istio.io/latest/zh/docs/concepts/traffic-management/#fault-injection)策略。
- 添加一个运行在虚拟机的服务来[扩展您的网格](https://istio.io/latest/zh/docs/examples/virtual-machines/single-network/#running-services-on-the-added-VM)。
- 从逻辑上添加来自不同集群的服务到网格，在 Kubernetes 上实现一个[多集群 Istio 网格](https://istio.io/latest/zh/docs/setup/install/multicluster/gateways/#configure-the-example-services)。

您不需要为网格服务要使用的每个外部服务都添加服务入口。默认情况下，Istio将Envoy代理配置为将请求传递给未知服务。但是，您不能使用 Istio 的特性来控制没有在网格中注册的目标流量。

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

# [Istio安装](https://istio.io/latest/docs/setup/getting-started/)

* 下载并解压

  ```
  # 执行脚本自动下载并解压
  curl -L https://istio.io/downloadIstio | sh -
  # 手动下载并解压
  https://github.com/istio/istio/releases
  ```

* 目录介绍

  ```
  bin        # 二进制命令istioctl目录
  samples    # 应用范例
  manifests  # istio部署的各种资源文件
  tools      # 各种调试工具
  ```

* 初始化操作

  ```
  # 移动二进制文件istioctl到PATH变量下
  mv bin/istioctl /usr/local/bin/
  # 命令行自动补全【~/.bashrc】
  source /root/istio-1.7.1/tools/istioctl.bash
  ```

* 安装

  ```
  # 安装选项
  istioctl profile list
  * default 适用生产环境，需要资源较高，包含istiod、istio-ingressgateway
  * demo 展示istio的核心功能，需要资源适中，包含：istiod、istio-egressgateway、istio-ingressgateway；适合Bookinfo范例和相关任务的使用；包含链路追踪和日志，不适合性能测试
  
  # 安装
  istioctl install --set profile=demo
  
  # 安装结果检查
  kubectl get pod -n istio-system 
  kubectl get svc -n istio-system
  ```

## istio卸载

```
istioctl x uninstall --purge
kubectl delete namespace istio-system
```

# Envoy sidecar注入

* 自动注入（命名空间）：添加名称空间标签，以指示Istio以后在相关名称空间部署应用程序时自动注入Envoy sidecar代理

  ```
kubectl label namespace default istio-injection=enabled
  ```
  
* 手动注入：手动向k8s工作负载注入Envoy sidecar代理

  ```
  istioctl kube-inject 
  
  # Update resources on the fly before applying.
  kubectl apply -f <(istioctl kube-inject -f <resource.yaml>)
  
  # Create a persistent version of the deployment with Envoy sidecar
  # injected.
  istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml
  
  # Update an existing deployment.
  kubectl get deployment -o yaml | istioctl kube-inject -f - | kubectl apply -f -
  ```

# Istio web界面

* 部署Kiali仪表板以及Prometheus，Grafana和Jaeger。

  ```
  kubectl apply -f samples/addons
  while ! kubectl wait --for=condition=available --timeout=600s deployment/kiali -n istio-system; do sleep 1; done
  ```

* 启动web服务kiali

  ```
  istioctl dashboard kiali --address 0.0.0.0
  ```

* web浏览器访问

  ```
  http://x.x.x.x:20001/kiali
  ```

# 应用范例-Bookinfo

## 应用介绍

![](https://istio.io/latest/docs/examples/bookinfo/noistio.svg)



* productpage：前端页面，调用details和reviews服务
* details：包含书籍信息
* reviews：包含书籍评论信息，会调用ratings服务
  * v1版本不调用ratings服务
  * v2版本调用ratings服务，将每个等级显示为1至5个黑色星星
  * v3版本调用ratings服务，将每个等级显示为1到5个红色星星
* ratings：包含有书评组成的评级信息

## 部署应用

* 部署应用

  ```
  kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
  ```

* 检查部署结果

  ```
  kubectl get svc,pod
  ```

* 在集群中检查应用是否正常运行

  ```
  kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
  ```

## 对外暴露应用

* 部署gateway以使服务网格外部可以访问应用

  ```
  kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
  ```

* 分析配置是否有问题

  ```
  istioctl analyze
  ```

* 获取ingress gateway的部署主机ip(pod)和NodePort(service)

  ```
  export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
  export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
  export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
  export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
  echo "$GATEWAY_URL"
  ```

* web浏览器访问

  ```
  echo http://"$GATEWAY_URL/productpage"
  ```

## 金丝雀部署

上述部署中（bookinfo-gateway.yaml），没有针对每个service定义VirtualService，此时访问主页(http://$GATEWAY_URL/productpage)的流量都导向productpage服务，productpage服务以轮询的方式调用reviews服务的3种工作负载(v1、v2、v3)。以下主要对reviews服务进行金丝雀部署，逐步升级：v1-->v2-->v3

* 使用v1版本【全部服务】

```
kubectl apply -f destination-rule-all.yaml         # 定义各服务v1版本对应的工作负载(pod)
kubectl apply -f virtual-service-all-v1.yaml       # 定义访问各服务时都访问服务的v1版本
```

* A/B Test【jason用户访问v2，其他访问v1】

  ```
  kubectl apply -f virtual-service-reviews-test-v2.yaml 
  
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: reviews
  spec:
    hosts:
      - reviews
    http:
    - match:
      - headers:
          end-user:
            exact: jason
      route:
      - destination:
          host: reviews
          subset: v2
    - route:
      - destination:
          host: reviews
          subset: v1
  ```

* 金丝雀部署【80%用户访问v1，20%用户访问v2】

  ```
  kubectl apply -f virtual-service-reviews-80-20.yaml
  
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: reviews
  spec:
    hosts:
      - reviews
    http:
    - route:
      - destination:
          host: reviews
          subset: v1
        weight: 80
      - destination:
          host: reviews
          subset: v2
        weight: 20
  ```

* 逐步增加v2版本访问权重(weight)，至最终完全访问v2

* v2到v3的版本升级类似。

## 应用卸载

```
# 卸载
samples/bookinfo/platform/kube/cleanup.sh
# 确认
kubectl get virtualservices   #-- there should be no virtual services
kubectl get destinationrules  #-- there should be no destination rules
kubectl get gateway           #-- there should be no gateway
kubectl get pods              #-- the Bookinfo pods should be deleted
```

# [应用发布方案](https://my.oschina.net/izu/blog/3009733)

* 蓝绿发布
* 滚动发布
* 金丝雀发布/灰度发布【A/B Test】

## 蓝绿发布

项目逻辑上分为AB组，在项目升级时，首先把A组从负载均衡中摘除，进行新版本的部署。B组仍然继续提供
服务。A组升级完成上线，将B组从负载均衡中摘除。B组资源预先保留一段时间，A组在一段时间内持续可用没有问题时，删除B组占用资源；如果A组有问题时，立即切换到B组资源。
特点：

* 策略简单
* 升级/回滚速度快，只要老版本的资源不被删除，理论上，可以在任何时间回滚到老版本。

缺点：

* 需要两倍以上服务器资源，短时间内浪费一定资源成本
* 有问题影响范围大

## 滚动发布

每次只升级一个或多个服务，升级完成后加入生产环境，不断执行这个过程，直到集群中的全部旧版
升级新版本。这也是kubernetes的默认发布策略。
特点：

* 相对于蓝绿部署，更加节约资源——它不需要运行两个集群、两倍的实例数。
* 部署过程自动化，一般不需要或不能人工干预

缺点：

* 部署周期长
* 发布策略较复杂：需要针对发布过程中的可用性、回滚等操作制定策略
* 不易回滚
* 影响范围较大

## 灰度发布/金丝雀发布
只升级部分服务，即让一部分用户继续用老版本，一部分用户开始用新版本，如果用户对新版本没有什么
意见，那么逐步扩大范围，把所有用户都迁移到新版本上面来。
特点：

* 可以根据应用的可用性，不断调整新旧版本的使用比例

缺点：

* 自动化要求高，需要处理新旧版本持续的上下线及不同版本的访问权重：例如90%的用户维持使用老版本，10%的用户尝鲜新版本。
* 同时运行多个版本，需要处理多版本的兼容性

### A/B Test

灰度发布的一种测试方式，主要对特定用户采样后，对收集到的反馈数据做相关对比，然后根据比对结果作出
决策。用来测试应用功能表现的方法，侧重应用的可用性，受欢迎程度等，最后决定是否升级

# Istio金丝雀发布思路

* 创建k8s资源
  * 创建应用的多个版本(deployment/pod)，每个版本使用不同的标签区别(例如：version: v1/v2)
  * 使用service关联所有的版本
* 使用istio部署应用：部署应用时注入Envoy sidecar
* 创建istio资源
  * 使用DestinationRule关联所有的应用版本
  * 使用VirtualService定义路由策略（基于用户内容或基于百分比）
  * 【可选】ingress gateway暴露应用

