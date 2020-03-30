---
title: kubernetes-安全-授权和准入控制
tags:
  - RBAC
  - 资源限制
  - 资源配额
categories:
  - kubernetes
date: 2020-03-30 01:20:57
---

# RBAC
基于角色的访问控制(role-based access control),将权限授予角色而非直接授予使用者  
RBAC主要定义谁(subject)可以操作(verb)什么对象(object)，具体如下：  

* 主体(subject):动作的发出者，通常以“账号”为载体，包含用户账号(UserAccount)和服务账号(ServiceAccount)
* 操作(verbs)：要执行的具体操作，如创建、删除、修改、查看等；对应于kubectl命令来说，则为create、delete、apply、update、patch、edit、get等子命令
* 客体(object)：动作施加于的目标实体，包含各类资源对象和非资源型的URL

RBAC包含两种类型的角色(权限集合)：Role和ClusterRole  

* Role：表示对哪些资源(object)可以执行的权限集合(verbs)，作用于名称空间级别
* ClusterRole：作用于集群级别,控制Role无法生效的资源类型，如
    - 集群级别的资源：Nodes、persistvolume
    - 非资源类端点(URL)：/healthz
    - 作用于所有名称空间的资源

对角色授权需要用到：RoleBinding和ClusterRoleBinding  

* RoleBinding：将同一名称空间的Role或ClusterRole绑定到一个或一组用户上，作用域仅为同一名称空间
* ClusterRoleBinding：将ClusterRole绑定到一个或一组用户,使用户有全部名称空间(即集群级别)的的操作权限

# Role
* 资源文件
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: testing
  name: pods-reader
rules: # 定义规则
- apiGroups: [""] 
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```
* 资源文件语法：
    - apiGroups：资源的API群组，空串标识核心群组core
    - resources：规则应用的资源类型
    - resourceNames：规则应用的资源名称，缺省则表示指定资源类型下的所有资源
    - verbs：对资源可以执行的操作：get、list、create、update、patch、watch、proxy、redirect、delete、deletecollection
    - nonResourceURLs：非资源型URL地址，仅适用于ClusterRole和ClusterRoleBinding
* kubectl命令方式：`kubectl create role service-admin --verb="*" --resource="services,services/*" -n testing`

# RoleBinding
* 资源文件
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: resources-reader
  namespace: testing
subjects:
- kind: User
  name: kube-user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pods-reader
  apiGroup: rbac.authorization.k8s.io
```
* kubectl命令方式：`kubectl create rolebinding admin-service --role=service-admin --user=kube-user1 -n testing`
* 资源文件语法
    - subjects:要绑定的主体
        + kind：主体资源类型，包括：User、Group、ServiceAccount
        + apiGroup：主体所属的API群组，ServiceAccount默认为空串""，User和Group默认为“rbac.authorization.k8s.io”
        + name：主体名称
        + namespace：主体(ServiceAccount类型)所属名称空间
    - RoleRef：引用的角色
        + apiGroup：引用的Role或ClusterRole所属的API群组
        + kind：引用的资源类型，Role或ClusterRole
        + name：引用的资源名称

# ClusterRole
## 集群资源
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nodes-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
```
## 非资源型URL
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: url-admin
rules:
- nonResourceURLs:
  - /healthz
  verbs:
  - get
  - create
```
## 聚合类型
* 父级资源:通过标签匹配资源
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
     rbac.example.com/aggregate-to-monitoring: "true"
rules: []
```
* 子级资源：定义标签
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "watch", "list"]
```

## 内置ClusterRole
|  ClusterRole  | ClusterRoleBinding |                                                说明                                                |
|---------------|--------------------|----------------------------------------------------------------------------------------------------|
| cluster-admin | system:masters     | 授予超级管理员在任何对象上执行任何操作的权限                                                       |
| admin         | None               | 以RoleBinding机制访问指定名称空间的所有资源，包括Role和RoleBinding，但不包含资源配置和名称空间本身 |
| edit          | None               | 允许读写访问一个名称空间内的绝大多数资源，但不允许查看或修改Role或RoleBinding                      |
| view          | None               | 允许读取一个名称空间的绝大多数资源，但不允许查看Role或RoleBinding，以及secret资源                  |

* 超级管理员(cluster-admin)：基于同名的ClusterRoleBinding绑定到“system:masters”组，任何属于该组的用户都有超级管理员权限  
* 自定义超级管理员的方式：
    * 创建用户证书，其中O的值为“system:masters”【用户所属组】
    * 将创建的用户使用ClusterRoleBinding绑定到cluster-admin
* 自定义管理员(admin)：`kubectl create rolebinding dev-admin --clusterrole=admin --user=kube-user1 -n dev`

# kube-dashboard部署
## 部署https访问证书
* 创建私钥和证书
```
openssl genrsa -out certs/dashboard.key 2048
openssl req -new -key certs/dashboard.key -out certs/dashboard.csr -subj "/CN=kube-user1/O=kubernetes"
openssl x509 -req -in certs/dashboard.csr -CA /etc/kubernetes/pki/ca.pem -CAkey /etc/kubernetes/pki/ca-key.pem -CAcreateserial -out certs/dashboard.crt -days 3650
```
* 创建secret对象【kubernetes-dashboard-certs】：`kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kube-system`

## 资源文件部署
* 使用hostPort或NodePort将访问端口暴露在宿主机
* 资源文件中也包含创建secret对象的内容【部署https证书】，可能会发出警告信息

## 访问认证
dashboard是部署于集群中的pod对象，访问集群中的资源需要使用ServiceAccount账户类型  
同时这种ServiceAccount账户需要绑定集群角色(cluster-admin)，以获得对集群的相应操作权限  
dashboard页面提供了kubeconfig和token两种访问认证方式  

* 创建服务账号
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```
* 给服务账号绑定权限
```
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
* 获取ServiceAccount的token信息，访问dashboard：`kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')`
* 基于token信息制作kubeconfig，访问dashboard
```
kubectl config set-cluster kubernetes --embed-certs=true --server="https://172.17.8.101:6443" --certificate-authority=/etc/kubernetes/ssl/ca.pem --kubeconfig=./kubeconfig
ADMIN_TOKEN=$(kubectl get secret admin-token-l7gjt -n kube-system -o jsonpath={.data.token}|base64 -d)
kubectl config set-credentials kube-system:admin --token=${ADMIN_TOKEN} --kubeconfig=./kubeconfig
kubectl config set-context admin --cluster=kubernetes --user=kube-system:admin --kubeconfig=./kubeconfig
kubectl config use-context kube-system:admin --kubeconfig=./kubeconfig
```

# 准入控制器
## LimitRanger
* 支持在名称空间级别为每个资源对象指定最大、最小、默认的计算或存储资源请求和资源限制
* 支持限制容器(cpu/内存)、pod(cpu/内存)、persistvolumeclaim(存储)三种资源的用量
* 范例
```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default: #默认限制
      cpu: 1000m
    defaultRequest: #默认需求
      cpu: 1000m
    min: #最小可申请资源
      cpu: 500m
    max: #最大可申请资源
      cpu: 2000m
    maxLimitRequestRatio: #最大可申请资源（最小用量的倍数）
      cpu: 4
    type: Container #限制的资源类型
```

## ResourceQuota
* 定义名称空间的对象数量，以及所有对象消耗的系统资源总量(计算及存储资源)
* 在名称空间启用cpu和内存等系统资源配额后，用户创建pod对象时必须指定资源需求或资源限制【也可以通过LimitRange控制器设置默认值】
* 资源配额仅对创建ResourceQuota对象之后创建的对象有效，对已经存在的对象不会产生任何影响
* 开启资源配额(apiserver)：【--enable-admission-plugins=ResourceQuota】

### 资源配额类型
* 计算资源配额
    - limits.cpu
    - limits.memory
    - requests.cpu
    - requests.memory
* 存储资源配额
    - requests.storage：所有pvc存储总量
    - persistentvolumeclaims：可以申请的pvc数量
    - storage-class-name.storageclass.storage.k8s.io/requests.storage：指定类别下的pvc存储总量
    - storage-class-name.storageclass.storage.k8s.io/persistentvolumeclaims：指定类别下的pvc数量
    - requests.ephemeral-storage：所有pod可用的本地临时存储需求总量
    - limits.ephemeral-storage：所有pod可用的本地临时存储限制总量
* 资源对象数量配额【1.9版本开始支持所有标准名称空间类型资源，形如count/resource.group】
    - count/persistentvolumeclaims
    - count/services
    - count/secrets
    - count/configmaps
    - count/replicationcontrollers
    - count/deployments.apps
    - count/replicasets.apps
    - count/statefulsets.apps
    - count/jobs.batch
    - count/cronjobs.batch
    - count/deployments.extensions
* 资源对象数量配额【1.9版本之前语法】
    - configmaps    
    - persistentvolumeclaims
    - pods
    - replicationcontrollers
    - resourcequotas
    - services
    - services.loadbalancers
    - services.nodeports
    - services.nodeports
* 自定义资源对象数量配额【1.15版本支持自定义资源，形如count/widgets.example.com】

### 资源配额适用范围
* Terminating
* NotTerminating
* BestEffort
* NotBestEffort

### 范例
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-example
  namespace: test
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    count/deployments.apps: "1"
    count/deployments.extensions: "1"
    persistentvolumeclaims: "2"
```

## PodSecuritypolicy
* 定义：用于控制用户在配置pod资源时可以设定的特权类属性，比如
    - 是否可以使用特权容器、根名称空间、主机文件系统
    - 可使用的主机网络和端口、卷类型、linux capabilities
* 使用步骤：
    - 创建PSP对象
    - 对相关的使用主体(用户、组、服务账号)进行RBAC授权
        + 将use权限授予特定的Role或ClusterRole
        + 将UserAccount或ServiceAccount完成角色绑定
    - apiserver启用PodSecuritypolicy准入控制器(--enable-admission-plugins=PodSecurityPolicy)

### PSP对象
* [特权PSP](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/privileged-psp.yaml)
* [非特权PSP](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)

### ClusterRole和ClusterRoleBinding
* ClusterRole
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <role name>
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - <list of policies to authorize>
```
* ClusterRoleBinding
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <binding name>
roleRef:
  kind: ClusterRole
  name: <role name>
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize specific service accounts:
- kind: ServiceAccount
  name: <authorized service account name>
  namespace: <authorized pod namespace>
# Authorize specific users (not recommended):
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: <authorized user name>
# Authorize all service accounts in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts
# Or equivalently, all authenticated users in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```
