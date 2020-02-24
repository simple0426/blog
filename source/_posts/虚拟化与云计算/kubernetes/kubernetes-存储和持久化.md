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

# volumes类型
* 本地存储(临时存储)：emptyDir、hostpath
* 网络存储：
    - in-tree(插件代码在k8s仓库中)：nfs、awsElasticBlockStorage、gcePersistentDisk
    - out-of-tree(插件代码不在k8s仓库中)：flexvolume、csi等网络存储
* 配置存储：secret、ConfigMap、downwardAPI、ServiceAccountToken
* pv(persistent volumes)与pvc(persistent volume claim)
    * 资源级别
        - pv是集群级别的资源
        - pvc与pod高度相关，必须位于同一个名称空间
    * 职责
        - pv包含后端存储实现的细节(存储类型、访问路径、认证信息等)
        - pvc包含用户对存储的需求声明(size/accessmode)

# emptyDir
## 用途
* 用于同一个pod内的多个容器间文件的共享
* 做诶容器数据的临时存储目录用于数据缓存系统

## 范例
```
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir-pod
spec:
  volumes: #pod的volume定义
    - name: html
      emptyDir: {} #emptydir类型的volume与pod生命周期相同
  containers:
    - name: nginx
      image: nginx:1.12-alpine
      volumeMounts: #容器加载volume
        - name: html
          mountPath: /usr/share/nginx/html
    - name: pagegen
      image: alpine
      volumeMounts: #容器加载volume
        - name: html
          mountPath: /html
      command: ["/bin/sh", "-c"]
      args:
        - while true;do
            echo $(hostname) $(date) >> /html/index.html;
            sleep 10;
          done
```
# hostPath
## 用途
* 将节点的文件或目录挂载到pod中，独立于pod的生命周期
* 一般适用于管理任务的pod(如daemonset)，它运行于集群中的每个工作节点，负责收集工作节点系统级的相关数据

## 范例
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
          mountPath: /var/lib/docker/contaners
          readOnly: true
  volumes:
    - name: varlog
      hostPath:
        path: /var/log
    - name: var-lib-docker-containers
      hostPath:
        path: /var/lib/docker/contaners
    - name: socket
      hostPath:
        path: /var/run/docker.sock
```

# oss使用
## ossfs
* oss作为共享存储直接挂载到操作系统(类似nfs)
* [ossfs安装](https://help.aliyun.com/document_detail/153892.html)
* 使用
    - 将认证信息存入文件（权限640）：echo ${bucket}:${access-key-id}:{access-key-secret} > /etc/passwd-ossfs
    - 挂载：ossfs bucket_name mount_point -ourl=endpoint

## kubernetes存储管理
* 将oss部署到kubernetes集群中有两种方式：flexvolume或csi，以下使用均以flexvolume为例
    - flexvolume需要先安装[插件](https://help.aliyun.com/document_detail/86785.html#title-36d-7xb-uaa)
    - [CSI](https://help.aliyun.com/document_detail/134903.html)
* 使用flexvolume需要kubelet关闭--enable-controller-attach-detach选项
* 在kube-system用户空间中部署flexvolume
* oss目前只支持静态存储卷，oss静态存储卷有两种使用方式：
    - [使用pv/pvc](#pv-pvc)
    - [直接使用volume方式](https://help.aliyun.com/document_detail/130911.html#title-a30-0jw-3t9)

# pv-pvc
## [pv创建](https://help.aliyun.com/document_detail/130911.html#title-3r5-zwg-j87)
### pv-范例
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

### pv-参数
* capacity：pv存储空间
* accessModes：访问模式
    + ReadWriteOnce：单节点读写
    + ReadOnlyMany：多节点读)
    + ReadWriteMany：多节点读写
* persistentVolumeReclaimPolicy：pv空间被释放时的处理机制
    - Retain：默认，保持不动
    - Recycle：空间回收（目前仅nfs、hostpath支持）
    - Delete：删除存储卷，仅部分云端存储支持
* volumeMode：卷模型，当做块设备还是文件系统，默认文件系统(Filesystem)
* storageClassName:当前pv所属的StorageClass名称
* mountOptions：挂载选项

## pvc创建
### pvc-范例
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
### pvc-参数
* accessModes：访问模式
* resources：申请资源
* selector：标签
* storageClassName：所依赖的存储类的名称
* volumeMode：卷模型，当做块设备还是文件系统，默认文件系统(Filesystem)
* volumeName：用于直接指定要绑定的pv的卷名

## pod使用
```
volumes:
- name: pvc-oss
  persistentVolumeClaim:
    claimName: pvc-oss   
```
