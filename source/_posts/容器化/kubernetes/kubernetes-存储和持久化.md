---
title: kubernetes-存储和持久化
tags:
  - volume
  - oss
  - 存储
categories:
  - kubernetes
date: 2020-02-24 19:36:53
---

# 持久化设置

* pod级别设置卷类型(volumes类型)：spec.volumes
* 容器级别使用卷：spec.containers.volumeMounts

# [volumes类型](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)

## 内置存储类

> 可以直接在pod的volumes中引用

* 本地存储：emptyDir、hostpath、local
* 网络存储：nfs、rbd、cephfs、glusterfs
* 云存储：awsElasticBlockStorage、gcePersistentDisk、azureDisk
* 配置存储：secret、ConfigMap、downwardAPI

## 持久化存储-pv/pvc

* 出现背景：屏蔽后端存储实现的细节，使用人员只需要声明存储需求
* 实现方式
  * pv(persistent volumes)：包含后端存储实现的细节(存储类型、访问路径、认证信息等)，使得存储作为集群中的资源管理。有两种类型的pv实现：
    * 静态：建立容量大小固定的存储卷
    * 动态：只创建与特定存储后端关联存储类(storageclass)，有需求到来时(pvc)，再建立特定的存储卷(pv)
  * pvc(persistent volume claim)：包含使用者对存储的需求声明(size/accessmode)

## 存储插件

>k8s核心代码外存储实现，一般为云厂商对接k8s存储的实现形式；这些插件最终都要通过volume或pv/pvc的方式接入k8s集群

* flexvolume：存在于csi之前，使用命令行exec模式与存储驱动交互；flexvolume的存储驱动(存储系统的二进制管理命令)必须在每个节点安装；pod通过k8s核心代码内的flexvolume插件(运行于k8s的专有pod)与FlexVolume的存储驱动交互
* CSI(container storage interface)：为容器编排系统(如k8s)定义了一个存储接口，从而可以将任意的存储系统暴露给容器负载(pod)；csi卷类型不支持直接从pod引用，必须从PersistentVolumeClaim对象引用

# 本地存储

## emptyDir

* 用途
  * 用于同一个pod内的多个容器间文件的共享
  * 做为容器数据的临时存储目录用于数据缓存系统

* 范例

```
piVersion: v1
kind: Pod
metadata:
  name: volume-emptydir-pod
spec:
  containers:
  - name: write
    image: centos
    command: ["bash", "-c", "for i in {1..100};do echo $i >> /data/hello;sleep 1;done"]
    volumeMounts: #容器加载volume
    - name: data
      mountPath: /data
  - name: read
    image: centos
    command: ["bash", "-c", "tail -f /data/hello"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes: # pod级别定义volumes
  - name: data
    emptyDir: {} #emptydir类型的volume与pod生命周期相同
```
## hostPath

* 用途
  * 将节点的文件或目录挂载到pod中，独立于pod的生命周期
  * 一般适用于管理任务的pod(如daemonset)，它运行于集群中的每个工作节点，负责收集工作节点系统级的相关数据

* 范例

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
spec:
  containers:
  - name: filebeat
    image: ikubernetes/filebeat:5.6.7-alpine
    env:
    - name: REDIS_HOST
      value: 192.168.2.124
    - name: LOG_LEVEL
      value: info
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: socket
      mountPath: /var/run/docker.sock
    - name: var-lib-docker-containers
      mountPath: /var/lib/docker/containers
      readOnly: true
  volumes:
  - name: varlog
    hostPath:
      path: /var/log
  - name: var-lib-docker-containers
    hostPath:
      path: /var/lib/docker/containers
  - name: socket
    hostPath:
      path: /var/run/docker.sock
```

# 网络存储

## [nfs](#nfs-server)

此处直接在pod中以volume形式挂载nfs


### pod使用

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nfs-server
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nfs-server
        nfs:
          path: /
          server: 192.168.31.203
```

### 客户端http访问

* 先在共享目录建立index.html文件
* curl _port-ip_

# 持久化存储-pv/pvc

## [pv静态供给](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

此处使用nfs创建静态存储卷

### [pv创建](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv01
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /pv01
    server: 192.168.31.203
```

* capacity：pv存储空间
* accessModes：访问模式
  + ReadWriteOnce：单节点读写：数据独立，一般为块设备
  + ReadOnlyMany：多节点读
  + ReadWriteMany：多节点读写，数据共享，一般是文件系统
* persistentVolumeReclaimPolicy：pv空间被释放时的数据处理机制
  - Retain：默认，保留pv和数据
  - Recycle：保留pv，删除数据（目前仅nfs、hostpath支持）
  - Delete：删除pv和数据，仅部分云端存储支持
* volumeMode：卷模型，当做块设备还是文件系统，默认文件系统(Filesystem)
* storageClassName：当前pv所属的StorageClass名称
* mountOptions：挂载选项

### [pvc创建](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 6Gi
```

* accessModes：访问模式
* resources：申请资源
* selector：查询volume标签用于绑定
* storageClassName：要求绑定的存储类
* volumeMode：卷模型，当做块设备还是文件系统，默认文件系统(Filesystem)
* volumeName：用于直接指定要绑定的pv的卷名

### pv与pvc的匹配因素

* 容量大小：pv>=pvc，且两者容量接近
* 访问模式

### pod使用

```
volumes:
- name: nfs-server
  persistentVolumeClaim:
    claimName: pvc-2 
```

## [pv动态供给](https://kubernetes.io/docs/concepts/storage/storage-classes/)

### 存储类插件

* 官方内置支持的存储类：https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner
* 社区实现的存储类：https://github.com/kubernetes-incubator/external-storage
  * nfs：映射本地目录，建立nfs服务端，并提供nfs实现的存储类
  * nfs-client：挂载已存在的nfs服务器提供存储类

### storageclass创建

* 可以使用[nfs](https://github.com/kubernetes-retired/external-storage/tree/master/nfs/deploy/kubernetes)中的【class.yaml、deployment.yaml、rbac.yaml】文件创建nfs存储类

  ```
  curl -LO https://raw.githubusercontent.com/kubernetes-retired/external-storage/master/nfs/deploy/kubernetes/deployment.yaml
  curl -LO https://raw.githubusercontent.com/kubernetes-retired/external-storage/master/nfs/deploy/kubernetes/rbac.yaml
  curl -LO https://raw.githubusercontent.com/kubernetes-retired/external-storage/master/nfs/deploy/kubernetes/class.yaml
  ```

  修改deployment

  * image：registry.cn-hangzhou.aliyuncs.com/simple00426/nfs-provisioner:latest
  * hostPath：/tmp/nfs-provisioner

* [集群设置默认存储类](https://kubernetes.io/zh/docs/tasks/administer-cluster/change-default-storage-class/)

  ```
  kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
  ```

* 存储类创建范例-阿里云nas

  ```
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: alicloud-nas-subpath
  mountOptions:
  - nolock,tcp,noresvport
  - vers=3
  parameters:
    volumeAs: subpath
    server: "xxxxxxx.cn-hangzhou.nas.aliyuncs.com:/k8s/"
  provisioner: nasplugin.csi.alibabacloud.com
  reclaimPolicy: Retain
  ```

  - provisioner:提供存储资源的存储系统
  - parameters：存储类使用参数描述要关联的存储卷
  - reclaimPolicy：动态创建pv的回收策略，默认delete，可选retain
  - mountOptions：pv挂载选项
  - volumeBindingMode：如何给pvc完成供给和绑定

### pvc创建

和pv静态供给使用的pvc基本一致，增加参数：storageClassName

## pv删除

* 删除pv的步骤：删除pod==》删除pvc==》删除pv
* 无法删除pv的处理：`kubectl patch pv xxx -p '{"metadata":{"finalizers":null}}'`

# 存储插件-oss使用

## 使用方式

将oss部署到kubernetes集群中有两种方式【以下均以flexvolume为例】：

- [CSI](https://help.aliyun.com/document_detail/134903.html)
- flexvolume：
  - 使用前需要先安装[插件](https://help.aliyun.com/document_detail/86785.html#title-36d-7xb-uaa)，且需要注意：
    - 使用flexvolume需要kubelet关闭--enable-controller-attach-detach选项
    - 在kube-system用户空间中部署flexvolume
  - oss目前只支持静态存储卷，oss静态存储卷有两种使用方式
    - [使用pv/pvc](#flexvolume方式使用oss)
    - [直接使用volume方式](https://help.aliyun.com/document_detail/130911.html#title-a30-0jw-3t9)

## [pv创建](https://help.aliyun.com/document_detail/130911.html#title-3r5-zwg-j87)
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-oss
spec:
  capacity: 
    storage: 5Gi
  accessModes: 
    - ReadWriteMany
  storageClassName: oss
  flexVolume:
    driver: "alicloud/oss"
    options:
      bucket: "docker"
      url: "oss-cn-hangzhou.aliyuncs.com"
      akId: ***
      akSecret: ***
      otherOpts: "-o max_stat_cache_size=0 -o allow_other"
```

## pvc创建
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-oss
spec:
  storageClassName: oss
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```
## pod使用
```
volumes:
- name: pvc-oss
  persistentVolumeClaim:
    claimName: pvc-oss   
```

# 命令行使用oss的方式-ossfs

* 此时oss作为共享存储直接挂载到操作系统(类似nfs)
* [ossfs安装](https://help.aliyun.com/document_detail/153892.html)

* 使用
    - 将认证信息存入文件（权限640）：echo ${bucket}:${access-key-id}:{access-key-secret} > /etc/passwd-ossfs
    - 挂载：ossfs bucket_name mount_point -ourl=endpoint

# nfs-server
## 创建nfs服务器

```
mkdir nfs
docker run -d --name nfs --net=host --privileged -v $(pwd)/nfs:/nfsshare -e SHARED_DIRECTORY=/nfsshare itsthenetwork/nfs-server-alpine:latest
```

## 客户端挂载测试

```
mkdir client
mount -v 192.168.31.203:/ client/
```