---
title: kubernetes-pod资源调度
tags:
  - 污点
  - 容忍度
  - 亲和性
  - 反亲和性
  - nodeSelector
categories:
  - kubernetes
date: 2020-03-24 17:28:48
---

# 创建pod流程
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-module-interact.jpg)

* 用户通过UI或CLI提交一个pod给API Server进行部署
* API Server将信息写入etcd存储
* Scheduler通过API Server的watch或notification机制获取这个信息
* Scheduler通过资源使用情况进行调度决策(pod在哪个节点部署)，并把决策信息返回给API server
* API server将决策结果写入etcd
* API server通知相应的节点进行pod部署
* 相应节点的kubelet得到通知后，调用container runtime启动容器、调度存储插件配置存储、调度网络插件配置网络

# pod调度器
* apiserver接收用户创建pod对象请求后，调度器会从集群中选择一个可用的最佳节点来运行它  
* kube-scheduler是默认的调度器  
* 用户也可以自定义调度器插件，并在定义pod对象时通过spec.schedulerName指定

## 调度过程
* [预选](#节点过滤规则)：基于过滤条件，过滤可行的节点
* [优选](#优先级排序规则)：基于不同的度量因子对可行的节点进行优先级排序
* 选择：选择优先级最高的节点来运行pod对象

## 影响调度因素
* 单个pod或所有pod的资源要求：cpu、内存等资源要求
* 硬件、软件、策略约束：pod只能运行在特定节点；pod对特定硬件的要求（SSD、GPU）
* 亲和性、反亲和性要求
* 数据存储位置：某些存储卷只能某些区域加载使用
* 工作负载间的相互干扰

## 节点过滤规则
* PodFitsHostPorts：检查节点端口是否满足pod需求
* PodFitsHost：检查节点主机名（hostname）是否满足pod需求
* PodFitsResources：检查节点是否有可用资源（cpu、内存）满足pod需求
* PodMatchNodeSelector：检查pod的节点选择器是否匹配节点标签
* NoVolumeZoneConflict：在给定区域(zone)上，检查pod要求的存储卷在节点上是否可用
* NoDiskConflict：在给定的节点上，检查pod要求的存储卷是否可用；如果这个节点已经挂载了卷，其他使用这个卷的pod不能调度到这个节点上
* MaxCSIVolumeCount:单个节点最多可以挂载多少个CSI存储卷
* CheckNodeMemoryPressure：检测节点内存压力
* CheckNodePIDPressure：检测节点PID压力
* CheckNodeDiskPressure：检测节点磁盘压力（文件系统满或接近满）
* CheckNodeCondition：检测节点在下列情况是否可用
    - 文件系统满
    - 网络不可达
    - kubelet还没准备好调度pod
* PodToleratesNodeTaints:检测pod的容忍度是否可以容忍节点的污点
* CheckVolumeBinding：检查节点上已绑定或未绑定的PVC是否满足pod对存储的需求

## 优先级排序规则
* SelectorSpreadPriority：是否可以将service、statefulset、replicaset所属的pod对象扩展到尽可能多的节点
* InterPodAffinityPriority：遍历pod对象的亲和性条目，并将那些能够匹配到给定节点的条目的权重相加，结果值越大的节点得分越高
* LeastRequestedPriority：request越少得分越高，比较倾向于让pod分配到空闲的机器上
* MostRequestedPriority：request越多得分越高，比较倾向于尽量压满一台机器，避免过多碎片化
* RequestedToCapacityRatioPriority：按照请求和容量的比例记分
* BalancedResourceAllocation：cpu、内存使用更均衡的节点得分更高
* NodePreferAvoidPodsPriority：根据节点是否设置了注解【scheduler.alpha.kubernetes.io/preferAvoidPods】；使用这个选项，可以标识哪些pod不能运行在同一个节点上
* NodeAffinityPriority：使用基于【PreferredDuringSchedulingIgnoredDuringExecution】的节点亲和性偏好进行优先级排序；匹配的条目越多，权重越高，得分也越高
* TaintTolerationPriority：根据节点上无法容忍的污点数量对节点进行排序 
* ImageLocalityPriority：节点上存在pod对象要求的镜像时，得分高
* ServiceSpreadingPriority：service的pod可以尽可能多的部署在不同节点
* CalculateAntiAffinityPriorityMap：计算pod的反亲和性优先级
* EqualPriorityMap：给所有的节点相同的权重

# 节点选择-nodeName
* 位置：spec.nodeName
* 功能：以下两者功能相同
```
nodeName: minikube
nodeSelector:
  kubernetes.io/hostname=minikube
```

# 节点选择器-nodeSelector
将pod调度至特定节点运行
## 节点标签定义
- 定义：kubectl label nodes minikube minikube=yes
- 查看：kubectl get node --show-labels

## [内置节点标签](https://kubernetes.io/docs/reference/node/node-labels/#preset-labels)
- kubernetes.io/os=linux：操作系统类型
- kubernetes.io/hostname=minikube：节点主机名

## pod定义nodeSelector
```
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
  nodeSelector:
    kubernetes.io/hostname: node2
```

# 节点亲和性调度
通过在节点定义标签，在pod对象上指定标签选择器来确定pod的运行位置

## 分类
* 硬亲和性（required）：pod对象调度时必须满足的规则，不满足时pod处于pending状态
* 软亲和性（preferred）：pod对象调度时应当遵守的规则

注意事项：IgnoredDuringExecution，在基于节点亲和性完成pod调度后，节点标签变更也不会将pod对象从节点移出，它只对新建的pod对象生效。

## 硬亲和性
将pod调度到拥有zone标签且其值为foo的节点上
```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values: ["foo"]
```
## 软亲和性
```
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 60
        preference:
          matchExpressions:
            - key: zone
              operator: In
              values: ["foo"]
      - weight: 30
        preference:
          matchExpressions:
            - key: ssd
              operator: Exists
              values: []
```

# pod亲和性调度
* 亲和性：基于某些需求，需要把某些pod资源部署在相近的位置，此时这些pod资源间具有亲和性(affinity)  
* 反亲和性：相反，需要把一些pod隔离开分散部署，此时这些资源具有反亲和性(anti-affinity)  

先使用标签选择器选出与之关联的pod对象，然后基于某种关系(topologyKey)将部署的pod和已存在的pod进行相近部署或分散部署
## 硬亲和性
```
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: ["cirros"]
        topologyKey: kubernetes.io/hostname
```
## 软亲和性
```
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["cache"]
          topologyKey: zone
      - weight: 20
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["db"]
          topologyKey: zone
```
## 反亲和性
```
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: ["myapp"]
        topologyKey: kubernetes.io/hostname
```
# 污点和容忍度
* 污点（taint）：是定义在节点上的键值对，用于让节点拒绝将pod部署于其上，除非该pod对象具有接纳该污点的容忍度
* 容忍度（toleration）：是定义在pod对象之上的键值对，用于配置其可以容忍的节点污点，而且调度器仅能将pod对象调度至能容忍该污点的节点上

## 污点和容忍度语法
* 污点定义在节点node.spec，容忍度定义在pod的pod.spec
* 他们都是键值对型数据，语法：key=value:effect
* effect定义对pod对象的排斥等级
    - NoSchedule：不能容忍此污点的pod对象不能调度到此节点
    - PreferNoSchedule：不能容忍此污点的pod对象可以调度到此节点
    - NoExecute：不能容忍此污点的pod对象不能调度到此节点，pod对象的容忍度或节点的污点变动时，pod对象将被驱逐
* pod定义容忍度，支持两种操作符（operator）
    - Equal：容忍度和污点在key、value、effect完全相同
    - Exists：容忍度和污点在key、effect完全匹配，容忍度中的value字段要使用空值
* 一个节点可以有多个污点，一个pod对象也可以有多个容忍度

## 管理节点的污点
* 添加污点：kubectl taint node node1 node-type=prod:NoSchedule
* 查看污点：kubectl describe node node01|grep -i taint
* 删除key的不同effect：kubectl taint node node1 node-type:PreferNoSchedule-
* 删除key：kubectl taint node node1 node-type-

## pod对象容忍度
```
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
  - key: "key1"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 3600
```
## 问题节点标识
当节点出现问题时，节点控制器自动为节点添加污点信息，使用NoExecute标识

* node.kubernetes.io/not-ready：节点进入“NotReady”状态时被自动添加
* node.kubernetes.io/unreachable：节点进入“NotReachable”状态时被自动添加
* node.kubernetes.io/out-of-disk：节点进入“OutOfDisk”状态时被自动添加
* node.kubernetes.io/memory-pressure：节点内存压力
* node.kubernetes.io/disk-pressure：节点磁盘压力
* node.kubernetes.io/network-unavailable：节点网络不可达
* node.kubernetes.io/unschedulable：节点不可调度
* node.cloudprovider.kubernetes.io/uninitialized：kubelet由外部云环境启动时，它将自动为节点添加此污点；待到云控制器管理器中的控制器初始化此节点时再将其删除

# pod优先级和抢占
[待续](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)
