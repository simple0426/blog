---
title: kubernetes-资源管理
tags:
  - kubectl
  - 标签
  - 选择器
categories:
  - kubernetes
date: 2020-03-21 00:20:49
---

# 实现方式-API
kubernetes API是管理各种资源的唯一入口，它提供了一个RESTful风格的CRUD接口，  
用于查询和修改集群的状态，并加结果存储于etcd中。
## API设计模式
* 声明式
    - 天然记录了状态
    - 幂等操作，可在任意时刻反复操作
    - 正常操作即巡检
    - 可合并多个变更
* 命令式
    - 如果命令没有响应，需要反复重试、记录当前的操作
    - 如果多次重试、操作有可能出问题
    - 需要单独的巡检操作，巡检本身也可能会有问题或带来额外影响
    - 多并发操作，需要加锁处理；不仅复杂，也降低了系统执行效率

## 实现原理
>API->Controller

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-control-circle.jpg)

* 由声明式的API驱动(k8s资源对象)
* 由控制器异步地控制系统向终态趋近
* 使系统自动化和无人值守成为可能
* 便于扩展(自定义资源和控制器)

# kubernetes资源对象
* 工作负载（workload）：replicaset、job、deployment、statefulset、daemonset
* 发现和负载均衡（Discovery&LB）：service、endpoint、ingress
* 配置和存储（config&storage）：ConfigMap、secret、volume
* 集群级别资源（cluster）：
    - namespace
    - Node
    - Role：名称空间级别的权限集合，可被RoleBinding引用
    - ClusterRole：cluster级别的权限集合，可被RoleBinding、ClusterRoleBinding医用
    - RoleBinding：将Role或ClusterRole许可的权限绑定在一个或一组用户之上，隶属于且仅能作用于一个名称空间
    - ClusterRoleBinding：将ClusterRole许可的权限绑定在一个或一组用户之上
* 元数据（metadata）：HPA(自动弹性伸缩)、pod模板、LimitRange(限制pod的资源使用)

## 查看api资源
* 支持的api接口版本：kubectl api-versions
* 支持的api资源信息：kubectl api-resources

## 自定义api资源
* 修改kubernetes源码自定义类型
* 创建自定义API server，并将其聚合至集群中
* 使用自定义资源CRD

## API使用
* 客户端命令(kubectl)：默认使用https访问接口，并且需要进行认证检查(kubeconfig)
* http方式：为了使用通用的HTTP方式访问API接口，可以使用kubectl proxy在本地启动一个网关代理
    - 启动代理网关：kubectl proxy --port 8080
    - 访问接口：curl localhost:8080/api/v1/namespaces/
* web界面：kubernetes dashboard
* 自动化配置管理(kube-applier)：用户将配置推送到git仓库中，配置工具自动将配置同步到集群中

# 资源对象格式
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  lables:
    name: nginx
spec: 
  containers:
  - name: nginx
     image: nginx
     ports:
     - containerPort: 80
```

* kind：资源类型
* apiVersion：API群组及相关的版本
* metadata：为资源提供元数据，如名称、隶属的名称空间、标签
* spec：用户期望的状态
* status：活动对象的当前状态【由kubernetes集群维护，对用户只读】

## 资源对象文档
* 查看资源对象语法：kubectl explain resources_name.field_name

## metadata
### 必选字段
* namespace：所属名称空间【默认default】
* name：对象名称
* uid：当前对象的唯一标识符【由集群自动生成】

### 可选字段
* labels:设定用于标识当前对象的键值对，常用作筛选条件
* annotations：非标识性键值对，用于labels的补充

# 标签和标签选择器
标识型的key:value元数据；可以在创建时指定，也可以通过命令随时添加到活动对象上
## 标签定义
* 键由键前缀和健名组成
    - 健名最多63个字符，只能以字母和数字开头及结尾，包含字母、数字、连词符、下划线、点
    - 键前缀是dns子域名格式，且不能超过253字符；省略前缀，键将被视为用户私有数据；由kubernetes系统组件或第三方组件为用户添加的键必须使用键前缀；“kubernetes.io"是系统核心组件使用的
* 键值：和健名规则相同

## 常用标签
* 版本标签：release：stable，release：canary，release：beta
* 环境标签：environment：dev，environment：qa，environment：prod
* 应用标签：app：ui，app：as，app：pc，app：sc
* 服务分层标签：tier：frontend，tier：backend，backend：cache
* 分区标签：partition：customerA，partition：customerB
* 品控级别标签：track：daily，track：weekly

## 创建时添加标签
```
kind: Pod
apiVersion: v1
metadata:
  name: pod-label
  labels:
    env: qa
    tier: frontend
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
```

## 动态修改标签
* 修改已存在标签：kubectl label pods/pod-label env=prod --overwrite
* 添加新标签：kubectl label pods/pod-label release=alpha
* 删除标签：kubectl label pods/pod-label release-

## 标签显示
* 显示所有标签：kubectl get pod --show-labels
* 显示特定键标签：kubectl get pod -L env,tier

## 标签选择器
* 标签选择器逻辑
    - 多个选择器之间为逻辑“与”关系
    - 空值的标签选择器意味着所有对象都被选中
    - 空标签选择器意味着没有对象被选中
* 选择器种类
    - 等值关系，操作符：“=”，“==”，“！=”
    - 集合关系，操作符：in，notin，exists(key-存在，!key-不存在)
* 命令行使用：
    - 相等型：kubectl get deployment --show-labels -l env=test,env!=prod
    - 集合型：kubectl get deployment --show-labels -l "Env in (test,gray)，Tie notin (front,back)，release，!release"
* selector使用【deployment、service、replicaset等使用】：
    - matchLabels:相等型关系匹配
    - matchExpressions：集合型关系匹配
        + key：key_name
        + operator：【In,NotIn,Exists,DoesNotExist】
        + values:\[values1,values2...\]
```
selector:
  matchLables:
    component: redis
  matchExpressions:
    - key: tier
      operator: In
      values: [cache]
    - key: environment
      operator: Exists
      values:
```

# 注解-annotations
## 定义
它也是标识型的键值对信息，但不能用于标签及对象选择，仅用于为资源提供元数据  
注解中的数据不受字符数量、特殊字符限制，可以是结构化(如json)也可以是非结构化数据  
## 使用场景
* 在为某资源引入新字段时，先以注解的方式提供；确定支持使用后，在资源中添加新字段，同时删除相关注解
* 资源的相关描述信息

## 查看注解
* describe：kubectl describe pod storage-provisioner -n kube-system
* get -o yaml：`kubectl get pod storage-provisioner -n kube-system -o yaml`

## 手动添加注解
kubectl annotate pods pod-label ilinux.io/created-by="cluster admin"

# kubectl命令
## 语法格式
kubectl sub_command resource_type resource_name cmd_option

## 子命令
* 基础命令-初级
    - create：从文件或标准输入创建资源
    - expose： 基于RC、service、deployment或pod创建service
    - run：通过创建Deployment在集群运行指定的镜像
    - set：设置指定资源的属性
* 基础命令-中级
    - explain：资源文档
    - get：显示一个或多个资源
    - edit：编辑集群上的资源，相当于先get后apply
    - delete：基于标准输入、文件名、资源名，或者资源标签删除资源
* 部署命令
    - rollout：管理资源的滚动更新
    - scale：设置资源Deployment、Replicaset、ReplicationController、Job的副本数
    - autoscale：自动设置Deployment、Replicaset、ReplicationController的副本数
* 集群管理命令
    - certificate：配置数字证书资源
    - cluster-info：显示集群信息
    - top：显示资源(CPU/内存/存储)使用率
    - cordon：标记节点为“不可调度”状态
    - uncordon：标记节点为“可调度”状态
    - drain：驱逐节点上的工作负载进入“维护”模式
    - taint：更新节点的污点
* 排错和调试命令
    - describe：显示资源或资源组详情
    - logs：显示pod内容器的日志
    - attach：挂载终端到一个运行中的容器
    - exec：在容器中执行命令
    - port-forward：转发一个或多个本地端口流量到pod
    - proxy：创建一个可以访问APIServer的代理
    - cp：在容器间复制文件和目录
    - auth：显示授权信息
* 高级命令
    - diff：对比活动的资源信息和将要应用的资源信息
    - apply：基于文件名或标准输入应用资源配置信息
    - patch：使用策略合并补丁更新资源的字段信息
    - replace：基于文件名或标准输入替换一个资源
    - wait：实验性功能，等待一个或多个资源的特定条件
    - convert：为不同api版本转换配置文件
* 设置命令
    - label：更新资源标签
    - annotate：更新资源注解
    - completion：输出指定shell的命令补全代码
* 其他命令：
    - api-resources：显示k8s支持的api资源
    - api-versions：显示k8s支持的api资源，以“group/version”形式
    - config：配置kubeconfig文件
    - plugin：提供和插件交互的工具
    - version：显示客户端和服务端的版本信息

## 命令选项
* -f：指定资源文件
    - 可以读取json、yaml格式资源清单
    - 指定清单文件路径、URL、目录
    - 可以多次使用
    - -R递归获取多级目录清单文件
* -o：指定输出格式
    - wide：宽格式显示
    - yaml、json：yaml、json格式显示资源对象
    - go-template：以自定义的go模板格式显示资源对象
    - custom-columns：自定义输出字段：`kubectl get pod -o custom-columns=NAME:metadata.name,IMAGE:spec.containers[0].image`
* -l：通过标签过滤资源
* -n：指定名称空间
* -s：指定apiserver地址
* --kubeconfig：指定kubeconfig文件【默认~/.kube/config】

## 插件-kubectl-debug
* [简介](https://github.com/aylei/kubectl-debug/blob/master/docs/zh-cn.md):通过启动一个安装了各种排障工具的容器，来诊断目标容器
* [安装](https://github.com/aylei/kubectl-debug/releases)
* 命令使用：`kubectl debug pod_name --agentless`
* 命令参数：
    - --fork：诊断一个处于CrashLookBackoff的pod
    - --agentless：启动无代理模式【命令使用时自动创建一个agent】
    - --image：指定使用的工具镜像【默认的镜像：nicolaka/netshoot】
    - -c container-name：指定要进入的容器内

# 资源对象操作
## 创建资源对象
- 直接通过各种子命令管理资源对象：kubectl run
- 根据资源文件创建资源对象：kubectl create
- 根据声明式资源文件让kubernetes集群自动调整资源状态：kubectl apply

## 查看资源对象
* get命令：kubectl get resource_type resource_name 
* describe命令：kubectl describe resource_type resource_name
* 参数：
    - -o wide/yaml/json/custom-columns:显示格式
    - -n namespace 名称空间
    - -l key=value 指定过滤标签

## 查看pod日志
* logs命令:kubectl logs pod_id
* 参数：
    - -c 指定容器名称

## 容器中执行命令
* exec命令: kubectl exec -it pod_name -c container_name /bin/bash

## pod弹性伸缩
* scale命令: kubectl scale rc nginx --replicas=4

## 修改资源对象
* edit: 编辑运行中的资源

## 删除资源对象
* delete命令:：kubectl delete resource_type resource_name
