---
title: kubernetes-认证授权和准入控制
tags:
  - kubeconfig
  - 认证
  - 授权
  - RBAC
  - 准入控制
  - 资源限制
  - 资源配额
categories:
  - kubernetes
date: 2020-03-30 01:21:12
---

# 访问控制
对kubernetes安全的保护主要通过如下方式实现：

* 认证（authentication）：验证访问者的身份(即访问对象)
* 授权（authorization）：确定认证用户的权限和可以执行的操作，如增删改查指定的资源对象
* 准入控制（admission control）：验证操作是否符合规范
    - 用默认值补足要创建的目标资源中未定义的各字段
    - 检查目标namespace是否存在
    - 检查是否违反系统资源限制

## 访问对象

* 服务账号（service account）：用于为pod中的进程访问api server提供身份标识
    * kubernetes管理的对象，可使用kubectl命令创建
    * 隶属于名称空间级别；
* 用户账号(user account)：校验操作者(用户)是否有权限执行相应的操作
    * kubernetes中不存在也不会存储此类账号，但是apiserver可以从https证书的CN(用户)和O(用户组)提取信息识别用户
    * 作用于系统全局
* 用户组（group）：用户账号的逻辑集合，附加于组上的权限可由其内部的所有用户继承；以下为一些内置的组
    - system:unauthorized：未通过认证的用户；未被任何验证机制明确拒绝的用户即为匿名用户，自动被标识为system:anonymous，并隶属于system:unauthorized组
    - system:authenticated：通过认证的用户
    - system:serviceaccounts：当前系统上所有的service account对象
    - system:serviceaccount：namespace：特定名称空间的serviceaccount对象

## [认证方式](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

api server支持同时启用多种认证机制，但至少应该分别为serviceaccount和useraccount各自启用一个认证插件  
各种认证方式串行进行，直到一种方式认证成功；若认证失败，则返回401状态码  
kubernetes认证插件尝试将下列属性关联到请求中：  

* Username：用户名，如kubernetes-admin
* UID：确定用户唯一性ID
* Groups：用户所属的组，用于权限指派和继承
* Extra：键值类型字符串，用于提供认证所需额外信息

APIserver支持的认证方式，如下：

* __HTTPS证书认证__：证书主体(subject)中包含用户(CN)和组(O)信息，用以识别用户
  * 【--client-ca-file=SOMEFILE】：是客户端(kubectl/kubelet/kube-proxy)认证的主要形式
* __HTTP Token认证__：通过一个token识别用户
  * 【--service-account-key-file)】：主要用于pod的认证；由apiserver自动启用，来验证由其签名的token；如果没有指定，则使用apiserver自己的私钥文件(--tls-*)验证token合法性
  * 【--token-auth-file=SOMEFILE】：Token文件中包含token、用户及组信息；token文件定义后不可变，除非apiserver重启；token标识也可以在http的头部进行设置
  * 【--enable-bootstrap-token-auth】：使用HTTP Token认证用于集群中新建节点认证过程
* __HTTP Base认证__【--basic-auth-file=SOMEFILE】：使用用户名+密码形式进行认证，用户名和密码以CSV明文格式存储
* 其他认证形式
  * openID连接令牌：OAuth2认证风格，它属于json web（JWT）类型；
  * webhook令牌：HTTP身份验证允许将服务器的URL注册为webhook，并接受带有承载令牌的POST请求进行身份验证
  * 认证代理：通过从请求头的值中识别用户，如X-Remote-User【--requestheader-username-headers】


## 授权方式
* Node：基于pod资源的目标调度节点来实现对kubelet的访问控制
* ABAC：attribute-based access control，基于属性的访问控制
* RBAC：role-based access control，基于角色的访问控制
* webhook：基于http回调机制通过外部REST服务检查确认用户授权的访问控制
* AlwaysAllow(总是允许)：用于期望不希望进行授权检查
* AlwaysDeny(总是拒绝)：仅用于测试

## [准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)
* AlwaysPullImages：总是下载镜像，常用于多租户环境下确保私有镜像仅能被有权限的用户使用
* DefaultStorageClass：为创建PVC设置一个默认的storageclass
* DefaultTolerationSeconds：如果pod对象没有设置污点容忍期限，则在此设置默认宽容期
* LimitRanger：pod可用资源限制
* MutatingAdmissionWebhook：串行调用任何可能更改对象的webhook
* NamespaceLifecycle：拒绝在不存在的名称空间创建对象，删除名称空间也会删除空间下的所有对象
* ResourceQuota：定义名称空间内可使用的资源配额
* serviceaccount：用于实现service account管控机制的自动化，实现创建pod对象时自动为其附加相关的service account对象（当前名称空间的default）
* ValidatingAdmissionWebhook：并行调用匹配当前请求的所有验证类webhook，任何一个验证失败，请求即失败

# 认证对象-ServiceAccount

## ServiceAccount自动化

> 创建服务账号的同时也自动创建相关的认证token

kubernetes系统通过三个独立的组件间的相互协作完成服务账户的自动化

* serviceaccount控制器(controller-manager)负责管理名称空间的资源，确保每个名称空间都存在一个“default”的serviceaccount对象
* secret控制器(controller-manager)负责监控serviceaccount对象，联动创建或删除相关的secret对象
* serviceaccount准入控制器：在pod没有定义serviceaccount对象时将其设置为“default”，并将相关的secret对象以volume形式挂载到容器中；pod和api servcer交互时即可使用此secret进行认证

服务账号自动化需要进行的设置：  

* api-server：--service-account-key-file
* controller-manager：--service-account-private-key-file

## ServiceAccount实践

* 创建serviceaccount对象(相应的secret对象也会创建)：kubectl create serviceaccount sa-demo
* 调用imagePullSecret（需要事先创建用户私有仓库认证的secret对象）
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-demo
imagePullSecrets:
- name: aliyun-simple
```

# User认证-HTTPS证书
## 使用方式
* 服务端认证：只为服务器端配置证书，由客户端验证服务端的身份，常见场景如：为web服务配置https证书
* 双向认证：客户端和服务端需要验证对方身份【从证书中提取用户(CN)和组(O)信息】

## k8s中应用场景

> APIserver与controller-manager、scheduler一般部署在同一台主机，不需要安全认证

* APIserver与kube-proxy、kubelet通信【双向认证制作的kubeconfig】
* APIserver与kubelet初次使用bootstrap通信【由apiserver的明文token(bootstrap-token)制作的bootstrap.kubeconfig】
* APIServer与pod对象【由CA签名自动生成使用service account token】
* APIserver与集群外部【kubectl使用的kubeconfig】
* pod对象与集群外部【服务端认证，用于pod使用ingress、service暴露后外部访问】
* etcd存储
    - etcd集群内各节点间【双向认证】
    - etcd服务器与客户端(k8s-apiserver)【双向认证】

# 认证载体-kubeconfig
## kubeconfig使用

* 各类APIserver的客户端都可以使用kubeconfig形式和APIserver通信
  * kube-proxy
  * kubelet
  * kubelet的bootstrap
  * kubectl
* 可以使用两种认证方式制作kubeconfig：  
  * 基于ca签发的证书(cert)和私钥(key)
  * 基于ca认证的token【如bootstrap token、service account token】

## 配置命令
* kubectl config view:查看文件内容
* kubectl config set-cluster：设置clusters字段【访问APIserver的URL和集群名称】
* kubectl config set-credentials：设置users字段【访问APIServer的用户名和认证信息】
* kubectl config set-context：设置context字段【context由用户名和集群名称组合而成】
* kubectl config use-context：设置要使用的context，即current-context

# 授权模式-RBAC

## RBAC介绍

基于角色的访问控制(role-based access control),将权限授予角色而非直接授予使用者  
RBAC主要定义谁(subject)可以操作(verb)什么对象(object)，具体如下：  

* 主体(subject)：动作的发出者，通常以“账号”为载体，包含用户账号(UserAccount)和服务账号(ServiceAccount)
* 操作(verbs)：要执行的具体操作，如创建、删除、修改、查看等；对应于kubectl命令来说，则为create、delete、apply、update、patch、edit、get等子命令
* 客体(object)：动作施加于的目标实体，包含各类资源对象和非资源型的URL

RBAC包含两种类型的角色(对哪些对象可以进行怎样的操作)：Role和ClusterRole  

* Role：表示对哪些资源(object)可以执行的权限集合(verbs)，作用于名称空间级别
* ClusterRole：作用于集群级别,控制Role无法生效的资源类型，如
    - 集群级别的资源：Nodes、persistvolume
    - 非资源类端点(URL)：/healthz
    - 作用于所有名称空间的资源

对角色授权需要用到：RoleBinding和ClusterRoleBinding  

* RoleBinding：将同一名称空间的Role或ClusterRole绑定到一个或一组用户上，作用域仅为同一名称空间
* ClusterRoleBinding：将ClusterRole绑定到一个或一组用户,使用户有全部名称空间(即集群级别)的的操作权限

## Role

* kubectl命令方式：`kubectl create role service-admin --verb="*" --resource="services,services/*" -n testing`
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

## RoleBinding

* kubectl命令方式：`kubectl create rolebinding admin-service --role=service-admin --user=kube-user1 -n testing`
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

## ClusterRole

### 集群资源

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
### 非资源型URL

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
### 聚合类型

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

### 内置ClusterRole

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

# 认证授权范例-user
以kubeadm集群中新建用户aliang为例
## [创建私钥和证书](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)

* 创建ca配置文件【profiles和集群名称相同】

  ```
  cat > ca-config.json <<EOF
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
  ```

* 创建证书签名请求

  ```
  cat > aliang-csr.json <<EOF
  {
    "CN": "aliang",
    "hosts": [],
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

* 基于ca生成证书

  ```
  cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=ca-config.json -profile=kubernetes aliang-csr.json | cfssljson -bare aliang
  ```

## 配置kubeconfig

```
kubectl config set-cluster kubernetes --embed-certs=true --server="https://192.168.31.201:6443" --certificate-authority=/etc/kubernetes/pki/ca.crt --kubeconfig=aliang.kubeconfig
kubectl config set-credentials aliang --embed-certs=true --client-certificate=aliang.pem --client-key=aliang-key.pem --kubeconfig=aliang.kubeconfig
kubectl config set-context aliang@kubernetes --cluster=kubernetes --user=aliang --kubeconfig=aliang.kubeconfig
kubectl config use-context aliang@kubernetes --kubeconfig=aliang.kubeconfig
```

## 给用户aliang授权

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods","services"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: aliang
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## kubectl命令测试

> kubectl默认使用的kubeconfig文件在用户主目录下的.kube中，例如
>
> * linux下：`/home/ceshi/.kube/config`
> * windows下：`C:\Users\simple\.kube\config`

```
kubectl get pod --kubeconfig=aliang.kubeconfig
kubectl get svc --kubeconfig=aliang.kubeconfig
```

# 认证授权范例-serviceaccount

__以部署kube-dashboard为例__

## [资源文件部署](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml)
* 使用hostPort或NodePort将访问端口暴露在宿主机
* 资源文件中也包含创建secret对象的内容【web访问所需https证书】

## 访问授权
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
## web访问

* 使用ServiceAccount的token信息访问dashboard：`kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')`
* 基于token信息制作kubeconfig，访问dashboard
```
kubectl config set-cluster kubernetes --embed-certs=true --server="https://172.17.8.101:6443" --certificate-authority=/etc/kubernetes/ssl/ca.pem --kubeconfig=./kubeconfig
ADMIN_TOKEN=$(kubectl get secret admin-token-l7gjt -n kube-system -o jsonpath={.data.token}|base64 -d)
kubectl config set-credentials kube-system:admin --token=${ADMIN_TOKEN} --kubeconfig=./kubeconfig
kubectl config set-context admin --cluster=kubernetes --user=kube-system:admin --kubeconfig=./kubeconfig
kubectl config use-context kube-system:admin --kubeconfig=./kubeconfig
```

## https证书重建

> 用于解决https证书错误

* 创建私钥和证书

```
openssl genrsa -out certs/dashboard.key 2048
openssl req -new -key certs/dashboard.key -out certs/dashboard.csr -subj "/CN=kube-user1/O=kubernetes"
openssl x509 -req -in certs/dashboard.csr -CA /etc/kubernetes/pki/ca.pem -CAkey /etc/kubernetes/pki/ca-key.pem -CAcreateserial -out certs/dashboard.crt -days 3650
```

* 删除已存在的证书对象(secret):`kubectl delete secret kubernetes-dashboard-certs -n kube-system`
* 创建证书对象(secret)【kubernetes-dashboard-certs】：`kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kube-system`
* 删除kubernetes-dashboard的pod，重建pod并重新加载https证书：`kubectl delete pods $(kubectl get pods -n kube-system|grep kubernetes-dashboard|awk '{print $1}') -n kube-system`



# 准入控制

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

### RBAC

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