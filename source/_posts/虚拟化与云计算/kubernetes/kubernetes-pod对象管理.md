---
title: kubernetes-pod对象管理
tags:
  - 初始化容器
  - 钩子函数
  - 容器探测
  - 资源
categories:
  - kubernetes
date: 2020-02-24 19:46:38
---

# pod对象生命周期
## pod对象操作概述
* 运行初始化容器(init container)
* 创建主容器(main container)【必须】
* 容器启动后钩子(post start hook)
* 容器就绪型探测(readiness probe)
* 容器存活性探测(liveness probe)
* 容器终止前钩子(pre stop hook)

## pod状态
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

# initContainer
可用于普通containers启动前的初始化或前置条件检验(如检测网络连通性)
## 与普通containers区别
* initContainer会先于普通containers启动执行，所有initContainer执行成功后，普通containers才会被启动
* pod中多个initContainer之间是按定义顺序依次启动执行，而pod中多个普通containers时并行启动的
* initContainer执行成功就退出，而普通containers可能会一直执行或重启

## 范例
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

# 生命周期钩子函数
## 使用说明
* 生命周期节点
    - postStart：容器建立后立即运行的功能，但是k8s无法保证它一定运行在entrypoint之前
    - preStop：容器终止前执行的功能，此操作完成前阻塞删除容器的操作
* 实现方式：
    - exec：执行用户定义的命令
    - http：向执行url发起http请求

## 范例
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
# 容器探测
## 探测种类
- livenessProbe(存活性探测)：探测容器是否处于running状态，检测不通过时根据重启策略(restartPolicy)确定是否重启
- readinessProbe(就绪性探测)：判断容器是否准备就绪，可以对外提供服务；未通过时，会将此pod从endpoint(如service对象)中移除，直到pod就绪

## 探测方式
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

## 通用探测属性
* initialDelaySeconds：容器启动多长时间后进行首次探测，默认0s
* timeoutSeconds：探测超时时间，默认1s
* periodSeconds：探测频率，默认间隔10s，最小1s
* successThreshold：处于失败状态时，多少次成功才认为是成功；默认1，最小1
* failureThreshold：处于成功状态时，多少次失败才认为是失败；默认3，最小1

## 应用实践
* 调大判断的阈值(超时、次数等)，防止容器压力过大时出现误判
* exec执行shell脚本时，在容器内执行的时间会比较长(可以使用go等编译型语言执行)
* 使用tcpSocket时遇到TLS，需要判断业务层是否有影响

## 范例
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
