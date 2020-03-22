---
title: kubernetes-pod对象管理
tags:
  - 初始化容器
  - 钩子函数
  - 容器探测
  - 资源
  - 标签
  - 选择器
  - pod定义
categories:
  - kubernetes
date: 2020-02-24 19:46:38
---

# 容器与pod
## 容器
* 容器的本质是一个进程，是一个视图被隔离、资源受限制的进程
* 容器里pid=1的进程就是应用本身
* 管理虚拟机就是管理基础设施(操作系统)，管理容器就是管理应用本身(进程)
* kubernetes相当于云时代的操作系统，容器镜像相当于操作系统的软件安装包

## Pod
* 容器设计本身就是一种“单进程”模型；可以运行多个进程，但是不便于直接管理
    - 如果需要运行多进程，则pid=1的进程要具有管理其他进程的能力(systemd)
    - 也可以直接运行systemd，由systemd管理多进程；但是，此时：【容器管理】=【管理systemd】!=【直接管理应用本身】
* 作为类比，容器相当于进程(linux中的线程)，pod相当于进程组(linux进程，包含至少一个线程)
* 在k8s中，pod是一个逻辑单位，它包含多个相互协作的容器，共享某些资源(volume/网络)
    - pod是资源分配单位【类比，进程是操作系统资源分配的单位】
    - pod也是原子调度单位
* pod与多容器
    - 如果只是亲密关系，可以通过调度器让俩个应用部署在同一台宿主机上
    - 如果是超亲密关系(比如：产生日志的应用和写日志的应用)，则需要将多个应用定义在一个pod中
        + 会发生直接的文件交互
        + 使用localhost或socket进行本地通信
        + 会发生频繁的RPC调用
        + 会共享某些namespace

## pod实现
### 网络实现
* 启动一个infra container容器，pod里的其他容器通过join namespace的方式加入infra container的network namespace中
* pod的网络地址即是infra的地址
* pod的生命周期也即infra的生命周期

### 存储实现
在pod级别创建volume，然后pod内的所有容器挂载这个volume

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
     hostPath:
        path: /data
  containers:
  - name: nginx-container
     image: nginx
     volumeMounts:
      - name: shared-data
         mountPath: /usr/sharr/nginx/html
  - name: debian-container
     image: debian
     volumeMounts:
      - name: shared-data
         mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

### 容器设计模式
* sidecar：在pod里定义一些专门的容器，来执行主业务容器所需要的辅助工作
* 通过sidecar方式，可以实现功能的解耦和重用

#### 应用场景
* 日志收集(log)
* Debug应用(debug)
* 容器代理(proxy)：单独写一个proxy用来处理对外部集群的交互
* 适配器(adapter)：单独写一个应用用来处理URL变更、数据格式变更的操作，以完成对现有业务的兼容

#### 应用范例-war包
* 可选方式
    - 将war包和tomcat都打包进镜像中，但是代码和tomcat的更新都需要更新镜像
    - 制作单独的tomcat镜像，在容器运行时将war包挂载到容器内，此时则需要单独维护war包和一个分布式文件系统
* sidecar方式-initContainer
    - 先启动一个initContainer，将war包拷贝到共享目录
    - 再启动container，这个容器启动tomcat(共享目录webapps中包含war包)

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: resouer/sample:v2
     name: war
     command: ["cp", "/sample.war", "/app"]
     volumeMounts:
     - mountPath: /app
        name: app-volume
  contaners:
  - image: resouer/tomcat:7.0
     name: tomcat
     command: ["sh", "-c", "/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
     volumeMounts:
      - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
         name: app-volume
      ports:
      - containerPort: 8080
         hostPort: 8001
    volumes:
    - name: app-volume
       emptyDir: {}
```

# pod容器定义
spec.containers
## 镜像-images
* 范例
```
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx #容器名称
      images: nginx:latest #镜像名称
        imagePullPolicy: Always
```
* 设置选项
    - 仓库认证：imagePullSecrets【参见secret章节】
    - 镜像获取策略：imagePullPolicy
        + Always：镜像版本为latest或本地不存在的镜像则从远端仓库拉取
        + IfNotPresent：本地镜像不存在则从远端仓库拉取
        + Never：禁止从远端仓库拉取镜像，只使用本地镜像  

## 端口-ports
* 范例
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-example
spec:
  containers:
  - name: myapp
    images: ikubernetes/myapp:v1
    ports:
      - name: http #端口名称
        containerPort: 80 #指向容器监听的端口
        protocol: TCP #端口协议类型
```
* 选项说明：端口设置仅为说明性内容；为端口设置名称，方便被service调用
* 设置选项
    - hostPort：将节点端口映射到容器端口
    - hostIP：节点端口绑定的ip，默认为0.0.0.0，一般不设置
* 注意事项：hostPort和service下的NodePort区别
    - hostPort在pod下设置，只绑定到pod所在节点
    - NodePort在service下设置，绑定到所有节点

## 自定义命令-command
* 范例
```
kind: Pod
apiVersion: v1
metadata:
  name: pod-example
spec:
  containers:
  - name: myapp
    image: alpine:latest
    command: ["/bin/sh"] #命令
    args: ["-c", "while true;do sleep 30;done"] #参数
```
* 说明：查看镜像默认运行的命令
    - cmd方式：`docker inspect nginx:1.13 -f {\.Config.Cmd}`
    - entrypoint方式：`docker inspect nginx:1.13 -f {\.Config.Entrypoint}`
* 设置选项：命令和参数
    - command：会覆盖默认运行命令
    - args：则会给command或默认的运行命令提供参数

## 环境变量-env
* 范例
```
kind: Pod
apiVersion: v1
metadata:
  name: pod-env
spec:
  containers:
  - name: filebeat
    image: ikubernetes/filebeat:5.6.5-alpine
    env:
      - name: REDIS_HOST #变量名称
        value: db.ilinux.io:6379 #变量值
      - name: LOG_LEVEL
        value: info

```
* 选项说明：向pod传递变量之有两种方式：env和envFrom【参见配置管理ConfigMap和secret】

## 使用节点网络-hostNetwork
* 范例
```
kind: Pod
apiVersion: v1
metadata:
  name: pod-network
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
  hostNetwork: true #共享节点网络
  hostPID: true #共享节点PID
  hostIPC: true #共享节点IPC
```
* 选项说明:默认情况下，pod的所有容器会建立一个独立的网络名称空间；但是一些特殊的pod需要运行在节点的网络名称空间下，执行系统级的任务
* 应用范例：
    - 使用kubeadm部署的kube-controller-manager、kube-apiserver、kube-scheduler、kube-proxy组件
    - flannel网络部署
* 其他选项：共享节点PID和IPC
    - hostPID: 共享节点PID
    - hostIPC: 共享节点IPC

## 安全上下文-securityContext
securityContext主要用于容器运行时的安全管控，类似linux的selinux、sudo等内容，通过限制容器的行为，从而保障系统和其他容器的安全
### 选项位置
* 容器级别的securityContext：对指定容器生效
* pod级别的securityContext：对指定pod中的所有容器有效
* Pod Security Policies(PSP):对集群内的所有pod有效

### 选项类别
* 根据用户id和组id控制访问对象
* 基于selinux的安全标签
* 以特权或非特权方式运行(privileged)
* 是否能够权限升级(allowPrivilegeEscalation)
* 通过Linux Capablities为其提供部分特权
* 基于seccomp过滤进程可以操作的系统调用

### 范例
>以uid为1000的非特权用户运行容器，并禁止提权

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-securitycontext
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["/bin/sh", "-c", "sleep 86400"]
      securityContext: #容器级别securitycontext
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
```

# 标签和标签选择器
标识型的key:value元数据；可以在创建时指定，也可以通过命令随时添加到活动对象上
## 标签定义
* 键由键前缀和健名组成
    - 健名最多63个字符，只能以字母和数字开头及结尾，包含字母、数字、连词符、下划线、点
    - 键前缀是dns子域名格式，且不能超过253字符；省略前缀，键将被视为用户私有数据；由kubernetes系统组件或低三方组件为用户添加的键必须使用键前缀；“kubernetes.io"是系统核心组件使用的
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

## pod节点选择器-nodeSelector
* 功能：将pod调度至特定节点运行
* 节点标签
    - 定义：kubectl label nodes minikube minikube=yes
    - 查看：kubectl get node --show-labels
* 节点默认标签：
    - kubernetes.io/os=linux：操作系统类型
    - kubernetes.io/hostname=minikube：节点主机名【spec.nodeName】
* 范例
```
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
  nodeSelector:
    hostname: node2
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

# pod状态和重启
## 状态
* pending：api server创建了pod资源对象，但尚未被调度完成；或者处于拉取镜像的过程中
* running：pod已被调度到某节点，并且所有容器都被kubelet创建完成
* succeeded：pod中的所有容器已成功终止并且不会被重启
* failed：所有容器都已被终止，但至少有一个容器终止失败
* unknown：api server无法获取pod的正常状态信息，通常是由api无法与kubelet通信所致

## pod异常状态诊断
* pod停留在pending状态：pending表示调度器没有介入，使用【kubectl describe pod】命令查看事件排查，通常和资源使用有关
* pod停留在waiting状态：pod拉取镜像失败
* pod不断被拉起且可以看到crashing：pod已被调度，但是启动失败；通常是由于配置、权限造成，需要查看pod日志【kubectl logs pod】
* pod处于running但是没有正常工作：通常时由于部分字段拼写错误造成的，恶意通过校验部署来排查【kubectl apply --validate -f pod.yml】
* service无法正常工作：在排除网络插件自身的问题外，最有可能是label配置有问题，可以通过查看endpoint的方式进行查看

## 容器重启策略
>重启策略适用于所有容器(包含initContainer)  
>首次需要重启时立即重启，后续重启操作时延为10/20/40/80/160/300，最大时延为300

* Always：pod对象终止就将其重启，此为默认值
* OnFailure：仅在pod对象出错时才将其重启
* Never：从不重启

# 全生命周期操作
## 操作概述
* 运行初始化容器(init container)
* 创建主容器(main container)【必须】
* 容器启动后钩子(post start hook)
* 容器就绪型探测(readiness probe)
* 容器存活性探测(liveness probe)
* 容器终止前钩子(pre stop hook)

## initContainer
可用于普通containers启动前的初始化或前置条件检验(如检测网络连通性)
### 与普通containers区别
* initContainer会先于普通containers启动执行，所有initContainer执行成功后，普通containers才会被启动
* pod中多个initContainer之间是按定义顺序依次启动执行，而pod中多个普通containers时并行启动的
* initContainer执行成功就退出，而普通containers可能会一直执行或重启

### 范例
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  initContainers:
    - name: init-something
      image: busybox
      command: ["sh", "-c", "sleep 10"]
  containers:
    - name: myapp-container
      image: ikubernetes/myapp:v1
```

## 生命周期钩子函数
### 使用说明
* 生命周期节点
    - postStart：容器建立后立即运行的功能，但是k8s无法保证它一定运行在entrypoint之前
    - preStop：容器终止前执行的功能，此操作完成前阻塞删除容器的操作
* 实现方式：
    - exec：执行用户定义的命令
    - http：向执行url发起http请求

### 范例
```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
    - name: lifecycle-demo-container
      image: ikubernetes/myapp:v1
      lifecycle: #定义钩子函数
        postStart: #容器启动后执行
          exec: #执行用户自定义命令
            command: ["/bin/sh", "-c", "echo 'lifecycle hooks handler' > /usr/share/nginx/html/test.html"]
        preStop: #容器终止前执行
          httpGet: #发起http球球
            host: blog.csdn.net
            path: zhangmingli_summer/article/details/82145852
            port: 443
```
## 容器探测
### 探测种类
- livenessProbe(存活性探测)：探测容器是否处于running状态，检测不通过时根据重启策略(restartPolicy)确定是否重启
- readinessProbe(就绪性探测)：判断容器是否准备就绪，可以对外提供服务；未通过时，会将此pod从endpoint(如service对象)中移除，直到pod就绪

### 探测方式
* exec：执行用户自定义命令
    - 命令参数：command
    - exec探测会消耗容器资源，所以需要探测命令简单、轻量
* httpGet：向指定url发起get请求，响应码2xx、3xx为成功
    - 命令参数：
        + host：请求主机，默认pod ip
        + port：端口，必选字段
        + httpHeaders：自定义header信息
        + path：请求的http资源url
        + scheme：连接协议，HTTP/HTTPS,默认HTTP
    - 在多层架构中，只能针对当前服务层进行探测(其他两种方式也一样)；如果后端服务不可用，会造成pod一次次重启，直到后端服务可用
* tcpSocket：与容器tcp端口建立连接，端口打开即为成功
    - 命令参数：
        + host：请求主机，默认pod ip
        + port：端口，必选字段

### 通用探测属性
* initialDelaySeconds：容器启动多长时间后进行首次探测，默认0s
* timeoutSeconds：探测超时时间，默认1s
* periodSeconds：探测频率，默认间隔10s，最小1s
* successThreshold：处于失败状态时，多少次成功才认为是成功；默认1，最小1
* failureThreshold：处于成功状态时，多少次失败才认为是失败；默认3，最小1

### 应用实践
* 调大判断的阈值(超时、次数等)，防止容器压力过大时出现误判
* exec执行shell脚本时，在容器内执行的时间会比较长(可以使用go等编译型语言执行)
* 使用tcpSocket时遇到TLS，需要判断业务层是否有影响

### 范例
* readinessProbe
```
apiVersion: v1
kind: Pod
metadata:
  name: readiness-exec
  labels:
    test: readiness-exec
spec:
  containers:
    - name: readiness-exec-demo
      image: busybox
      args: ["/bin/sh", "-c", "while true;do rm -f /tmp/ready;sleep 30;touch /tmp/ready;sleep 300;done"]
      readinessProbe:
        exec:
          command: ["test", "-e", "/tmp/ready"]
        initialDelaySeconds: 5
        periodSeconds: 5
```
* livenessProbe
```
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
  labels:
    test: liveness-http
spec:
  containers:
    - name: liveness-http-container
      image: nginx:1.12-alpine
      ports:
        - name: http
          containerPort: 80
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo Healthy > /usr/share/nginx/html/Healthz"]
      livenessProbe:
        httpGet:
          path: /Healthz
          port: http
          scheme: HTTP
```

# 资源需求和限制
* cpu：单位：millicore(1core=1000millicore)
    - 500m相当于0.5个核心
* memory:单位：Bytes，与日常使用单位相同
    - 300M
* ephemeral(临时存储)：单位：Byte
* 自定义资源：配置时必须为整数
* 范例
```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: wp
    image: wordpress
    resources:
      requests: #请求的资源
        memory: 64Mi
        cpu: 250m
        ephemeral-storage: 2Gi
      limits: # 最多可以使用的资源
        memory: 128Mi
        cpu: 500m
        ephemeral-storage: 4Gi
```

## pod服务-QOS
依据容器对cpu、memory资源的request/limits需求，pod服务质量分为

* Guaranteed:
    - 每个容器都为cpu设置了具有相同值的requests、limits属性
    - 每个容器都为memory设置了具有相同值的requests、limits属性
* Burstable：至少一个容器设置了cpu或memory的requests属性，但不满足Guaranteed的要求
* BestEffort：没有为pod中的任何一个容器设置requests、limits属性

当节点上的memory资源不足时，将依据BestEffort、Burstable、Guaranteed的优先级顺序驱逐pod
