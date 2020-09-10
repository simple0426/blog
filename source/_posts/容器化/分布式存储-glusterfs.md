---
title: 分布式存储-glusterfs
tags:
  - glusterfs
  - heketi
categories:
  - kubernetes
date: 2020-03-18 23:58:06
---

# 简介
在实践kubernetes的statefulset及各种需要持久存储的功能时，  
通常需要用到PV的动态供给功能，而glusterfs就是这样一种可以提供动态供给功能的存储系统  
## 部署架构
| 主机名 |    ip地址    |       角色       |
|--------|--------------|------------------|
| node1  | 172.17.8.101 | glusterfs        |
| node2  | 172.17.8.102 | glusterfs        |
| node3  | 172.17.8.103 | glusterfs,heketi |

# 部署glusterfs
* 安装软件【三个节点都进行】
```
yum install centos-release-gluster -y
yum --enablerepo=centos-gluster*-test install glusterfs-server -y
systemctl start glusterd.service
```
* 在任意节点使用命令“发现”其他节点【例如node1】
```
gluster peer probe node2
gluster peer probe node3
```
* 查看集群状态:`gluster peer status`

# 部署heketi
heketi为管理glusterfs存储卷的生命周期提供了一个RESTful管理接口，  
借助于heketi，kubernetes可以动态调配glusterfs存储卷；  
在heketi中注册的设备(device)可以是裸分区或裸磁盘  
## 安装heketi
- 添加仓库：`wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo`
- 安装程序：`yum install heketi heketi-client`

## 配置heketi到glusterfs的ssh
```
ssh-keygen -t rsa -q -f /etc/heketi/heketi_key -N ''
chmod 0600 /etc/heketi/heketi_key.pub 
chown heketi.heketi /etc/heketi/heketi_key*
ssh-copy-id -i /etc/heketi/heketi_key.pub root@172.17.8.103
ssh-copy-id -i /etc/heketi/heketi_key.pub root@172.17.8.102
ssh-copy-id -i /etc/heketi/heketi_key.pub root@172.17.8.101
```
## heketi配置文件
>/etc/heketi/heketi.json

```
{
  "port": "8080", # 服务端口
  "use_auth": true, # 开启认证
  "jwt": {
    "admin": { #管理员及密码
      "key": "admin secret"
    },
    "user": {
      "key": "My Secret"
    }
  },
  "glusterfs": { #heketi管理glusterfs的方式
    "executor": "ssh",
    "sshexec": {
      "keyfile": "/etc/heketi/heketi_key",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },
    "db": "/var/lib/heketi/heketi.db", #heketi数据库
    "loglevel" : "debug" #日志
  }
}
```
## 启动heketi服务
- systemctl enable heketi
- systemctl start heketi

## 测试：
- curl方式：`curl http://node3:8080/hello`
- 内置命令：`heketi-cli --server http://node3:8080 --user admin --secret "admin secret" cluster list`

# heketi添加glusterfs
* 拓扑配置【/etc/heketi/heketi-topology.json】
```
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": ["172.17.8.101"],
                            "storage": ["172.17.8.101"]
                        },
                        "zone": 1
                    },
                    "devices": ["/dev/sdb", "/dev/sdc"]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": ["172.17.8.102"],
                            "storage": ["172.17.8.102"]
                        },
                        "zone": 1
                    },
                    "devices": ["/dev/sdb", "/dev/sdc"]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": ["172.17.8.103"],
                            "storage": ["172.17.8.103"]
                        },
                        "zone": 1
                    },
                    "devices": ["/dev/sdb", "/dev/sdc"]
                }                
            ]
        }
    ]
}
```
* heketi添加glusterfs拓扑配置：`heketi-cli -s http://node3:8080 --user admin --secret "admin secret" topology load --json=/etc/heketi/heketi-topology.json`
* 信息查看：
    - 查看集群信息：cluster list|cluster info ced9fc3405909cda59dd36101afea898
    - 查看节点信息：node list|node info 96a281a85ce8cc8a97a74c88c9173442
    - 存储卷信息：volume list
* 测试
    - 创建存储卷：volume create --size=5
    - 删除存储卷：volume delete volume_id

# k8s中使用glusterfs
## 创建storageclass
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://172.17.8.103:8080"
  clusterid: "ced9fc3405909cda59dd36101afea898"
  restauthenabled: "true"
  restuser: "admin"
  restuserkey: "admin secret"
  # secretNamespace: "default"
  # secretName: "heketi-secret"
  volumetype: "replicate:2"
```
## 创建pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  storageClassName: "glusterfs"
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
```
