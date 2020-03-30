---
title: kubernetes-安全-认证
tags:
  - kubeconfig
  - 认证
categories:
  - kubernetes
date: 2020-03-30 01:21:12
---

# 访问控制
对kubernetes安全的保护主要通过如下方式实现：

* 认证（authentication）：验证访问者的身份
* 授权（authorization）：确定认证用户的权限和可以执行的操作，如增删改查指定的对象
* 准入控制（admission control）：验证操作是否符合规范
    - 用默认值补足要创建的目标资源中未定义的各字段
    - 检查目标namespace是否存在
    - 检查是否违反系统资源限制

## 客户端访问
### 访问方式
* kubectl
* 客户端库
* REST接口

### 访问者
* 用户账号(user account)：kubernetes中不存在此类对象，也不会存储此类账号，它们仅仅用于校验用户是否有权限执行相应的操作；作用于系统全局
* 服务账号（service account）：kubernetes管理的账号，用于为pod中的进程访问api server提供身份标识；隶属于名称空间级别
* 用户组（group）：用户账号的逻辑集合，附加于组上的权限可由其内部的所有用户继承；以下为一些内置的组
    - system：unauthorized：未通过认证的用户；未被任何验证机制明确拒绝的用户即为匿名用户，自动被标识为system:anonymous，并隶属于system:unauthorized组
    - system：authenticated：通过认证的用户
    - system：serviceaccounts：当前系统上所有的service account对象
    - system：serviceaccount：namespace：特定名称空间的serviceaccount对象

## [认证方式](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
api server支持同时启用多种认证机制，但至少应该分别为serviceaccount和useraccount各自启用一个认证插件  
各种认证方式串行进行，直到一种方式认证成功；若认证失败，则返回401状态码  
kubernetes认证插件尝试将下列属性关联到请求中：  

* Username：用户名，如kubernetes-admin
* UID：确定用户唯一性ID
* Groups：用户所属的组，用于权限指派和继承
* Extra：键值类型字符串，用于提供认证所需额外信息

APIserver支持的认证方式，如下：

* X509数字证书(--client-ca-file=SOMEFILE)：证书主体(subject)中包含用户(CN)和组(O)信息;
* 静态token文件(--token-auth-file=SOMEFILE)：token文件定义后不可变，除非apiserver重启；token标识也可以在http的头部进行设置；
* 引导令牌(--enable-bootstrap-token-auth):常用于新建集群中节点认证过程；在master节点使用引导令牌完成节点的认证后自动为其签署数字证书用于后续通信
* 静态密码文件(--basic-auth-file=SOMEFILE):用户名和密码以CSV明文格式存储
* service account token：由apiserver自动启用；通过加载【--service-account-key-file】来验证由其签名的令牌；如果没有指定，则使用apiserver自己的私钥文件验证令牌合法性
* openID连接令牌：OAuth2认证风格，它属于json web（JWT）类型
* webhook令牌：HTTP身份验证允许将服务器的URL注册为webhook，并接受带有承载令牌的POST请求进行身份验证
* 认证代理：通过从请求头的值中识别用户，如X-Remote-User


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

# ServiceAccount
kubernetes系统通过三个独立的组件间的相互协作完成服务账户的自动化

* serviceaccount控制器(controller-manager)负责管理名称空间的资源，确保每个名称空间都存在一个“default”的serviceaccount对象
* secret控制器(controller-manager)负责监控serviceaccount对象，联动创建或删除相关的secret对象
* serviceaccount准入控制器：在pod没有定义serviceaccount对象时将其设置为“default”，并将相关的secret对象以volume形式挂载到容器中；pod和api servcer交互时即可使用此secret进行认证

服务账号自动化需要进行的设置：  

* api-server：--service-account-key-file
* controller-manager：--service-account-private-key-file

## 实践
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

# X509数字证书
## 使用方式
* 服务端认证：只为服务器端配置证书，由客户端验证服务端的身份，常见场景如：为web服务配置https证书
* 双向认证：客户端和服务端需要验证对方身份

## k8s中应用场景
* APIserver与controller-manager、scheduler、kube-proxy
* APIserver与kubelet初次通信，kubelet自动生成私钥和证书前面请求，master为其自动进行证书签名和颁发(即所谓的tls bootstraping),kubelet后续使用私钥和证书通信
* APIServer与kubelet及其他形式客户端，如pod对象
* APIserver(pod)与外部通信【服务端】
* kubelet和kube-proxy间通信
* etcd存储
    - etcd集群内各节点间
    - etcd服务器与客户端

# kubeconfig
各类APIserver的客户端都可以使用kubeconfig形式和APIserver通信：  

* controller-manager
* scheduler
* kube-proxy
* kubelet
* kubelet的bootstrap
* kubectl

## 配置命令
* kubectl config view:查看文件内容
* kubectl config set-cluster：设置clusters字段【访问APIserver的URL和集群名称】
* kubectl config set-credentials：设置users字段【访问APIServer的用户名和认证信息】
* kubectl config set-context：设置context字段【context由用户名和集群名称组合而成】
* kubectl config use-context：设置要使用的context，即current-context

## 配置范例
以创建kube-user1为例
### [创建私钥和证书](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)
* 创建私钥文件：`openssl genrsa -out kube-user1.key 2048`
* 创建证书签名请求：`openssl req -new -key kube-user1.key -out kube-user1.csr -subj "/CN=kube-user1/O=kubernetes"`【CN用户名，O用户组】
* 基于ca根证书签署证书：`openssl x509 -req -in kube-user1.csr -CA ../ssl/ca.pem -CAkey ../ssl/ca-key.pem -CAcreateserial -out kube-user1.crt -days 3650`
* 查看证书信息【可选】：`openssl x509 -in kube-user1.crt -text -noout`

### 配置kubeconfig
* 配置集群信息：`kubectl config set-cluster kubernetes --embed-certs=true --server="https://172.17.8.101:6443" --certificate-authority=/etc/kubernetes/ssl/ca.pem`
* 配置用户信息：`kubectl config set-credentials kube-user1 --embed-certs=true --client-certificate=/etc/kubernetes/pki/kube-user1.crt --client-key=/etc/kubernetes/pki/kube-user1.key`
* 配置上下文：`kubectl config set-context kube-user1@kubernetes --cluster=kubernetes --user=kube-user1`
* 切换当前上下文：`kubectl config use-context kube-user1@kubernetes`
* 测试【可选】：在启用RBAC的集群上，由于kube-user1所属组kubernetes无任何管理权限，所以执行命令会出错

# [TLS-Bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)
新的工作节点加入集群时需要配置相关的私钥和证书，可以由管理员手动配置，也可以让kubelet自行生成相关的私钥和证书  
但是集群规模扩大后，手动配置会增加管理员工作量，kubelet自动配置则会降低PKI本身的优势  
结合这两种方式，kubernetes提出了新方案(TLS Bootstrapping)：由kubelet自行生成私钥和证书签名请求  
然后向集群的证书签署进程(CA)发起请求，获取的证书和已有的私钥文件共同存储于kubelet客户端以用于后续的通信  
但是，这样一来就认证和非认证的kubelet客户端都会发起证书签名请求，为了控制仅那些认证的客户端可以发起请求，就需要使用令牌机制(bootstrap token)来确认认证的客户端  
APIserver使用基于认证令牌对“system:bootstrappers”组内用户完成认证  
controller-manager会用到这个组的默认审批控制器来决定是否颁发证书，它依赖于由管理员将此token绑定于合适的RBAC策略，以限制其仅用于签证操作  

## apiserver
* 配置项 
    * --enable-bootstrap-token-auth：开启bootstrap认证
    * --token-auth-file：bootstrap token文件
    * --client-ca-file=FILENAME：开启客户端证书认证，指定客户端证书的根证书机构（ca证书）
    * --tls-cert-file
    * --tls-private-key-file：如果apiserver开启https服务，则需要指定私钥和证书【由ca颁发的证书】
* token文件范例：`02b50b05283e98dd0fd71db496ef01e8,kubelet-bootstrap,10001,"system:bootstrappers"`
* 授权kubelet可以创建证书签名请求
```
# enable bootstrapping nodes to create CSR
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
```

## controller-manager
* 配置项
    * --cluster-signing-cert-file
    * --cluster-signing-key-file：用于颁发集群范围内证书的ca证书
    * --root-ca-file：根ca证书，此根证书颁发机构将包含在服务帐户的令牌密钥中
* 批准kubelet证书请求
```
# Approve all CSRs for the group "system:bootstrappers"
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

## kubelet
* 配置项
    - --bootstrap-kubeconfig：bootstrap的kubeconfig配置，主要包含bootstrap token
    - --kubeconfig：kubelet和apiserver通信的kubeconfig配置
    - --cert-dir：bootstrap获取的证书和原有的私钥文件的存储位置
* 配置bootstrap-kubeconfig
```
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-cluster bootstrap --server='https://my.server.example.com:6443' --certificate-authority=/var/lib/kubernetes/ca.pem
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-credentials kubelet-bootstrap --token=07401b.f395accd246ae52d
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-context bootstrap --user=kubelet-bootstrap --cluster=bootstrap
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig use-context bootstrap
```
