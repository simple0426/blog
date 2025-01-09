---
title: kubernetes-配置管理
tags:
  - secret
  - configMap
categories:
  - kubernetes
date: 2020-02-24 19:40:31
---

# ConfigMap
## 使用场景
* 主要管理容器运行所需的配置文件、环境变量、命令行参数等可变配置
* 用于解耦容器镜像和可变配置，从而保证工作负载(pod)的可移植性

## 创建configMap
### 清单文件方式
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cfg
  namespace: default
data:
  cache_host: memcached-gcxt
  cache_port: "11211"
  cache_prefix: gcxt
  my.cnf: |
    [mysqld]
    log-bin = mysql-bin
  app.properties: |
    property.1 = value-1
    property.2 = value-2
    property.3 = value-3
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel"
    }
```
### 命令行-文件方式
- key是文件名，value是文件内容；可以针对单个文件或目录(目录的所有文件)
- 范例-目录：kubectl create configmap test-config --from-file=./configs
- 范例-文件：kubectl create configmap test-config2 --from-file=./configs/cache.conf

### 命令行-key/value方式
- 范例：kubectl create configmap test-config3 --from-literal=name=yuanshuo --from-literal=age=23

## 使用ConfigMap
* configMap主要被pod使用,必须在pod使用前创建ConfigMap
* 创建的pod使用envFrom读取configmap时，如果configmap中的某些key无效，则该环境变量不会注入容器，但是pod可以正常创建
* 只有通过k8s api创建的pod才能使用configmap，其他方式创建的pod(如manifest创建的static pod)不能使用configmap
* pod只能使用在同一namespace下的ConfigMap

范例使用的ConfigMap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

### 环境变量方式
```
apiVersion: v1
kind: Pod
metadata:
  name: special-env-pod
  namespace: default
spec:
  containers:
  - name: test-container  
    image: busybox
    # 使用$(VAR_NAME)方式引用变量
    # command: ["/bin/sh", "-c", "echo $(SPECIAL_TYPE)"]
    command: ["/bin/sh", "-c", "echo $(special.how)"]
    # 加载全部变量
    envFrom: 
      - configMapRef:
          name: special-config
          optional: false
    # 加载部分变量
    # env: 
    #   - name: SPECIAL_HOW
    #     valueFrom:
    #       configMapKeyRef:
    #         name: special-config
    #         key: special.how
    #   - name: SPECIAL_TYPE
    #     valueFrom:
    #       configMapKeyRef:
    #         name: special-config
    #         key: special.type
  restartPolicy: Never
```
### volume挂载方式
```
apiVersion: v1
kind: Pod
metadata:
  name: spec-volume-pod
  namespace: default
spec:
  containers:
    - name: test-contaner1
      image: busybox
      # 文件名为key，文件内容为value
      # command: ["/bin/sh", "-c", "cat /etc/config/special.how"] 
      command: ["/bin/sh", "-c", "cat /etc/config/ok/special.how"] 
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config 
  volumes: 
    - name: config-volume
      configMap:
        name: special-config
        # 加载部分变量,只将special.how这个key挂载到/etc/config目录下的相对路径ok/special.how
        items:
          - key: special.how
            path: ok/special.how 
  # 加载全部变量
  # volumes: 
  #   - name: config-volume
  #     configMap:
  #       name: special-config
  restartPolicy: Never
```
# Secret
## 使用场景
* Secret是在集群中存储密码、token等敏感信息的对象
* secret的存储和打印格式均是base64编码的字符串，因此用户创建secret时也要提供此种编码格式的数据
* Secret类型
    - Opaque：一般密文类型【默认类型】
    - Kubernetes.io/service-account-token：集群身份认证【创建serviceaccount时自动创建】
    - kubernetes.io/tls：tls证书类型
    - kubernetes.io/dockerconfigjson：docker拉取私有仓库镜像时的认证信息
    - bootstrap.kubernetes.io/token：节点bootstrap初始化加入集群时的认证信息

## 创建secret
### 命令行方式

一般创建使用kubectl create命令创建opaque(generic)、tls、dockerregistry类型

* opaque-key/value形式：kubectl create secret generic mysecret --from-literal=username=hejingqi --from-literal=password=yuanshuo
* opaque-文件形式：kubectl create secret generic ssh-key-secret --from-file=ssh-priviatekey=C:\Users\simple\\.ssh\gitee --from-file=ssh-publickey=C:\Users\simple\\.ssh\gitee.pub
* dockerregistry创建：`kubectl create secret docker-registry aliyun-simple --docker-username=perfect@qq.com --docker-password=123456 --docker-server=registry.cn-hangzhou.aliyuncs.com`
* tls证书创建：`kubectl create secret tls k8s-ca --cert='ca.pem' --key='ca-key.pem'`
### 清单文件方式

一般用于创建opaque类型，核心参数如下：

* spec.data：key的value数据需要先进行base64编码
* spec.stringData：key的value数据无需进行base64编码

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret1
  namespace: default
type: Opaque
stringData:
  username: hejingqi
  password: yuanshuo
```

## 使用secret
和ConfigMap使用基本一致，但是由于使用环境变量方式时会存在信息泄露(容器继承、打印日志等方式)，所以基本上使用volume挂载方式
### Opaque类型
```
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - image: busybox
      name: secret-container
      command: ["/bin/sh", "-c", "ls /etc/config"]
      volumeMounts:
        - name: ssh-key
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: ssh-key
      secret:
        secretName: ssh-key-secret
```
### dockerconfigjson类型
使用imagePullSecrets设置认证信息，使用方式如下：
- 在ServiceAccount对象中定义imagePullSecrets，则所有使用该ServiceAccount的pod都能使用此认证信息
- 在pod中使用imagePullSecrets
```
apiVersion: v1 
kind: Pod
metadata:       
  name: ssh-pod
spec:
  imagePullSecrets:
  - name: aliyun-simple
  containers:
  - name: ssh
    image: registry.cn-hangzhou.aliyuncs.com/simple/ubuntu14_sshd
    ports:
    - containerPort: 22
```
# 应用程序动态更新配置

> pod会周期性获取配置(configmap和secret)，但是要自行处理应用程序如何动态加载配置

* 应用程序监听本地配置文件变动，自行处理热加载
* 使用sidecar通过inotify机制监听配置文件变更，动态更新
* 与程序迭代更新一起滚动更新
* 【不使用k8s的configmap/secret】采用配置中心(如disconf、Apollo、nacos)