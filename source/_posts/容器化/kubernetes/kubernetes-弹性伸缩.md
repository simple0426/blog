---
title: kubernetes-弹性伸缩
date: 2020-08-12 18:28:41
tags:
  - 弹性伸缩
  - HPA
categories: ['kubernetes']
---


# 弹性伸缩

集群的容量能够及时响应业务的负载需求

## 业务负载类型

* 在线负载类：微服务、API、网站
* 离线业务：离线计算、大数据、机器学习
* 定时任务：定时批量任务

在线负载类对弹性伸缩的弹出时间敏感；离线业务对价格敏感；定时任务对调度敏感

本章中的弹性伸缩主要针对在线负载类业务。

## 弹性伸缩维度

* CA(cluster autoscaler)：node个数扩容、缩容；使用cluster-autoscaler组件，一般为云厂商对接实现节点增加或删除
* HPA(Horizontal Pod Autoscaler)：pod个数自动扩容、缩容；kubernetes内置组件实现；pod需要设置requests、limits信息
* VPA(Vertical Pod AutoScaler)：pod配置(requests/limits)自动扩容、缩容；使用addon-resizer组件
  * 主要针对大型单体应用
  * 在pod中不设置requests、limits信息，由vpa控制器管理pod的配置

## 缩容问题-pod终止

* 核心：pod优雅停机
* 解决：pod中定义prestop钩子
* 范例：如果我的tomcat里面跑的订单业务，数据要求不丢失。当我的tomcat正在做一个批量操作，耗时5分钟，怎么保证缩容的时候，实现等待这个批量操作和其他数据流量操作完成后在缩容呢？
  * prestop钩子要保证任务执行完
  * 任务要实现事务回滚或重复执行要幂等

# 节点伸缩-CA

## 扩容node-步骤

* 观察到node内存、cpu不足或pod因为资源不足处于pending状态
  * 手动查看node资源分配情况：kubectl describe node k8s-master1
* 新建虚拟机
* 在虚拟机上部署kubelet等组件并加入集群

## 扩容node-ansible

* 范例：https://github.com/lizhenliang/ansible-install-k8s/blob/master/add-node.yml

* 自动签发kubelet证书

  ```
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: auto-approve-csrs-for-group
  subjects:
  - kind: Group
    name: system:bootstrappers
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
    apiGroup: rbac.authorization.k8s.io
  ```

## 缩容node-手动实现

* 标记节点不可调度：kubectl cordon k8s-node3
* 驱逐节点上的pod：kubectl drain k8s-node3 --ignore-daemonsets
* 从集群删除节点：kubectl delete node k8s-node3

# pod伸缩-HPA

## HPA简介

* 原理：HPA控制器从支持的[指标API](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis)(例如Metrics server)中获取指标数据(例如：内存/cpu)，然后基于用户的自定义规则进行pod副本数的增减

* 弹性伸缩冷却周期设置（kube-controller-manager组件启动参数设置）：
  * 扩容冷却周期(默认3分钟)：--horizontal-pod-autoscaler-upscale-delay 
  * 缩容冷却周期(默认5分钟)：--horizontal-pod-autoscaler-downscale-delay 

* HPA共有三个版本：autoscaling/v1只支持基于CPU的弹性伸缩；autoscaling/v2beta1增加对自定义指标的支持；autoscaling/v2beta2额外增加了对外部指标的支持。而这些版本的变化正是kubernetes社区对监控和监控指标的认识转变，从早期Heapster到Metrics Server再到将指标边界进行划分。

## HPA版本

* autoscaling/v1：只支持cpu选项；由[Metrics Server](https://github.com/kubernetes-incubator/metrics-server)提供资源指标API(metrics.k8s.io)
* autoscaling/v2beta1：除了支持资源指标(CPU/内存)外，也支持[自定义指标](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md)(custom.metrics.k8s.io)
  * 自定义指标：可以引用Kubernetes对象（或其组）和度量名称
  * 自定义指标由指标数据采集器的适配器提供，如[Prometheus Adapter](https://github.com/directxman12/k8s-prometheus-adapter)
  * 内存使用：由于各个语言都有自己的内存管理机制，采集到的数据和真实的数据不一定一致，所以不推荐使用内存进行伸缩设置
* autoscaling/v2beta2：相对于v2beta1额外增加[外部指标](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/external-metrics-api.md)(external.metrics.k8s.io)支持

  * 设计目标
    * 支持基于CPU百分比的伸缩
    * 支持关于pod任意指标数据的伸缩
    * 支持基于kubernetes对象相关任意指标数据的伸缩
    * 在单个hpa中实现对多个指标数据的扩展
  * 外部指标：与自定义指标功能类似，并基于自定义指标进行功能扩展

  * 外部指标由指标数据采集器的适配器提供，如[Prometheus Adapter](https://github.com/directxman12/k8s-prometheus-adapter)

# HPA范例-基于CPU指标

## 部署metrics-server

metrics server提供基于资源指标伸缩的API(metrics.k8s.io)

metrics server项目地址：https://github.com/kubernetes-incubator/metrics-server

## autoscaling/v1-CPU指标实践

### 部署pod

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-demo
  template:
    metadata:
      labels:
        app: java-demo
    spec:
      containers:
      - image: lizhenliang/java-demo
        name: java
        resources: 
           requests:
             memory: "300Mi"
             cpu: "250m"
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: java-demo
```

### 创建HPA

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  maxReplicas: 5
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  targetCPUUtilizationPercentage: 60
```

* scaleTargetRef：伸缩的对象
* targetCPUUtilizationPercentage：cpu使用率超过60%即开始扩容

### 压测与查看

* 开始压测

  ```
  yum install httpd-tools
  ab -c 1000 -n 100000  http://10.1.206.176 #10.0.0.147为ClusterIP
  ```

* 观察扩容缩容情况【压测完成后，过5分钟(缩容冷却周期)后开始缩容】

  ```
  kubectl get hpa
  kubectl top pods
  kubectl get pods
  ```

## autoscaling/v2beta2-多指标

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: 10k
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector: "queue=worker_tasks"
      target:
        type: AverageValue
        averageValue: 30
```

* Resource：当前伸缩对象下的pod的cpu和memory指标，只支持Utilization和AverageValue类型的目标值
* Object：k8s内部对象的指标，数据由第三方adapter提供，只支持Value和AverageValue类型的目标值
* Pods：当前伸缩对象pods的相关指标，数据由第三方adapter提供，只允许AverageValue类型的目标值
* External：k8s外部的指标，数据由第三方的adapter提供，只支持Value和AverageValue类型的目标值

# HPA范例-基于prometheus自定义指标

基于资源指标（cpu、内存）进行pod伸缩，使用metrics-server就可以(metrics.k8s.io)；但是如果想使用自定义指标：如请求qps/5xx错误数实现HPA，就需要部署单独的适配器来支持自定义指标(custom.metrics.k8s.io)；目前比较成熟的是 Prometheus Custom Metrics，由prometheus提供自定义指标，再利用k8s-prometheus-adpater将指标数据聚合到APIServer，以此实现和核心指标(metrics server)同样的效果

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/hpa-custom-metrics.png)

<img src="https://simple0426-blog.oss-cn-beijing.aliyuncs.com/hpa-metrics-all.png" style="zoom:200%;" />

## 部署prometheus

可以只部署prometheus-server和客户端组件【提供数据给HPA决策使用】

分组件部署范例：https://github.com/simple0426/sysadm/tree/master/kubernetes/prometheus

## 部署Custom Metrics Adapter

由于prometheus采集到的数据和k8s数据格式不兼容，因此不能直接给k8s用；此时需要使用一个适配器(k8s-prometheus-adpater)将prometheus的数据格式转换为k8s API可以识别的格式；转换后，因为是自定义API，所以还需要kubernetes aggregator在主APIServer中注册【API Server开启聚合层支持】

k8s-prometheus-adpater项目：https://github.com/DirectXMan12/k8s-prometheus-adapter

直接使用helm charts进行安装(指定prometheus地址)

```
helm repo add stable http://mirror.azure.cn/kubernetes/charts
helm repo update
helm search repo prometheus-adapter
helm install prometheus-adapter stable/prometheus-adapter --namespace ops --set prometheus.url=http://prometheus.ops,prometheus.port=9090
```

查看安装结果，是否已注册到APIServer

```
kubectl get apiservices |grep custom 
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

## 基于QPS指标实践

> QPS获取方式
>
> * 从日志中获取
> * 从代理或负载均衡器处获取，如nginx+lua、haproxy
> * 程序自身统计，以下实践案例即为程序自身提供指标数据

### 部署应用

> pod中添加注解(annotations)允许prometheus采集数据

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: metrics-app
  name: metrics-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "80"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - image: lizhenliang/metrics-app
        name: metrics-app
        ports:
        - name: web
          containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-app
  labels:
    app: metrics-app
spec:
  ports:
  - name: web
    port: 80
    targetPort: 80
  selector:
    app: metrics-app
```

metrics-app暴露了一个prometheus格式指标接口(http://URL/metrics)，可以通过service看到

```
curl 10.0.0.159/metrics
# HELP http_requests_total The amount of requests in total
# TYPE http_requests_total counter
http_requests_total 72
# HELP http_requests_per_second The amount of requests per second the latest ten seconds
# TYPE http_requests_per_second gauge
http_requests_per_second 0.5
```

### 定义适配器规则获取特定指标

规则范例：https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config-walkthrough.md

编辑prometheus-adapter的配置文件添加规则(重建prometheus-adapter使配置生效)：`kubectl edit cm prometheus-adapter -n ops`

```
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus-adapter
    meta.helm.sh/release-namespace: ops
  labels:
    app: prometheus-adapter
    app.kubernetes.io/managed-by: Helm
    chart: prometheus-adapter-2.5.0
    heritage: Helm
    release: prometheus-adapter
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}'
      resources:
        overrides:
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

以上规则表示：2min内pod的http_requests_total增长速率(以namespace、pod分类计算)

* seriesQuery：获取原始的指标数据(promQL语法)；此处为：从prometheus中获取所有namespace、pod下的http_requests_total指标
* resources：将prometheus中的标签名称格式统一，以反映为对应的k8s资源(kubectl api-resources)；此处为：kubernetes_namespace转换为namespace，kubernetes_pod_name转换为pod
* name：设置可用于hpa的度量指标名称；此处重命名获取的度量指标名称：http_requests_total重命名为http_requests_per_second
* metricsQuery：度量指标计算(promQL语法)；此处等同于【sum(rate(http_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}[2m])) by (kubernetes_namespace,kubernetes_pod_name)】

从api中查询prometheus-adapter获取的数据：

```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second"
```

### 创建HPA

```
piVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: metrics-app-hpa 
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: metrics-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue # 获取指标数据后，取多个pod速率的平均值。
        averageValue: 800m   # 800m 即0.8个/秒
```

### 压测和查看

```
ab -n 100000 -c 100 http://10.0.0.159/metrics
kubectl get hpa metrics-app-hpa
kubectl get pod
```