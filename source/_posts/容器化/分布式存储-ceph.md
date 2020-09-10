---
title: 分布式存储-ceph
tags:
  - ceph
categories:
  - kubernetes
date: 2020-09-11 01:08:25
---


# Ceph介绍

ceph是一个可以同时提供对象存储、块存储、文件存储三种服务能力的分布式存储系统

## 特点

**高性能**

- 摒弃了传统的集中式存储元数据寻址的方案，采用CRUSH算法，数据分布均衡，并行度高。
- 考虑了容灾域的隔离，能够实现各类负载的副本放置规则，例如跨机房、机架感知等。
- 能够支持上千个存储节点的规模，支持TB到PB级的数据。

**高可用性**

- 副本数可以灵活控制。
- 支持故障域分隔，数据强一致性。
- 多种故障场景自动进行修复自愈。
- 没有单点故障，自动管理。

**高可扩展性**

- 去中心化。
- 扩展灵活。
- 随着节点增加而线性增长。

**特性丰富**

- 支持三种存储接口：块存储、文件存储、对象存储。
- 支持自定义接口，支持多种语言驱动。

## 与其他存储系统对比

<img src="https://simple0426-blog.oss-cn-beijing.aliyuncs.com/ceph-basic.png" style="zoom: 80%;" />

<img src="https://simple0426-blog.oss-cn-beijing.aliyuncs.com/ceph-basic1.png" style="zoom: 80%;" />

<img src="https://simple0426-blog.oss-cn-beijing.aliyuncs.com/ceph-high.png" style="zoom: 80%;" />

<img src="https://simple0426-blog.oss-cn-beijing.aliyuncs.com/ceph-ext.png" style="zoom: 80%;" />

## 架构与组件

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/ceph-arch.png)

* 管理组件：
  * MGR(ceph-mgr，Luminous之后)：
    * 收集集群状态的相关度量信息（例如，ceph df）
    * 包括基于REST的API管理。注：API仍然是实验性质的，目前有一些限制，但未来会成为API管理的基础。
    * 可以对接外部度量监控系统，例如cephmetrics、zabbix、calamari、promethus
  * Admin【命令行工具】：Ceph常用管理接口通常都是命令行工具，如rados、ceph、rbd等命令，另外Ceph还有可以有一个专用的管理节点，在此节点上面部署专用的管理工具来实现近乎集群的一些管理工作，如集群部署，集群组件管理等。
* 客户端组件：
  * RGW：对象存储网关【可以使用LB(nginx)代理多个RGW服务实现负载均衡和高可用】
  * RBD：块存储接口
  * CephFS：文件存储接口
* 核心组件：
  * RADOS(Reliable Autonomic Distributed Object Store)：可靠的、自动化的、分布式对象存储系统。RADOS是Ceph集群的核心，它是一个抽象的存储系统
  * LIBRADOS：可以访问RADOS的库，上层的RBD、RGW和CephFS都是通过librados访问的；目前提供PHP、Ruby、Java、Python、C和C++支持。
* 底层组件：
  * OSD：负责物理存储的进程，一般配置成和硬盘一一对应，一个硬盘启动一个OSD进程；功能包括：存储数据、复制数据、平衡数据、恢复数据，以及与其它OSD间进行心跳检查，负责响应客户端请求返回具体数据的进程等
  * MON：保存OSD的元数据，维护集群的状态，管理集群客户端认证与授权
  * MDS(Ceph Metadata Server)：cephFS服务依赖的元数据服务，负责保存文件系统的元数据、目录结构等；_对象存储和块设备存储不需要此服务_
* 底层存储：
  * bluestore(Luminous之后)：BlueStore通过直接管理物理HDD或SSD而不使用诸如XFS的中间文件系统来管理每个OSD存储的数据，这提供了更大的性能和功能；BlueStore内嵌支持使用zlib，snappy或LZ4进行压缩；BlueStore支持Ceph存储的所有的完整的数据和元数据校验。
  * filestore：相对于bluestore，使用xfs文件系统管理osd数据

## 存储原理-Pool/PG

* CRUSH算法：一种为分布式存储的元数据提供一致性的算法，可以使ceph具有强大的扩展性
* object：object是ceph最底层的存储单元，包含元数据和原始数据。
* pool：存储对象的逻辑分区，它规定了数据的冗余类型(副本、纠错码)和对应副本的分布策略
* PG(placement groups)：它是对象的集合；相同PG内的对象都会放到相同的硬盘上(OSD)；服务端数据均衡和恢复的最小粒度就是PG；

PG与Pool关系：

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/pg-pool-osd.png)

* 一个Pool里有很多PG
* 一个PG里有很多对象
* PG有主从之分，一个PG分布在不同的OSD上(三副本类型)

PG数量阈值：

默认每个osd最多250个pg【mon_max_pg_per_osd】，假如有6个osd，一共可以有(250*6)1500个pg

以每个pg有2个副本计算，集群中pool可以创建的pg总数为500【1500/3，剩余的1000个pg由集群根据副本机制自动创建】

## ceph的三种存储类型

1、 块存储（RBD）  

- 优点：
  * 通过Raid与LVM等手段，对数据提供了保护；
  * 多块廉价的硬盘组合起来，提高容量；
  * 多块磁盘组合出来的逻辑盘，提升读写效率；  

- 缺点：
  * 采用SAN架构组网时，光纤交换机，造价成本高；
  * 主机之间无法共享数据；
- 使用场景
  * docker容器、虚拟机磁盘存储分配；
  * 日志存储；
  * 文件存储；

2、文件存储（CephFS）

- 优点：方便文件共享；

- 缺点：读写速率低、传输速率慢；
- 使用场景
  * 日志存储；
  * FTP、NFS；
  * 其它有目录结构的文件存储  

3、对象存储（Object）(适合更新变动较少的数据)

- 优点：
  * 具备块存储的读写高速；
  * 具备文件存储的共享等特性；


- 使用场景
  * 图片存储；
  * 视频存储；

# Ceph安装

可用安装方式如下，本文主要使用ceph-deploy安装

* ceph-deploy：https://docs.ceph.com/ceph-deploy/docs/contents.html
* ceph-ansible：https://docs.ceph.com/ceph-ansible/master/
* [rook](https://rook.io/)：在k8s内部署ceph集群
* 手动安装集群：https://ceph.readthedocs.io/en/latest/install/index_manual/#

## 环境规划

* 主机：3台centos7主机，每台配置2C4G，每台机器额外挂载最少2块硬盘(每块不少于5G)

  ```
  192.168.31.201 node01
  192.168.31.202 node02
  192.168.31.203 node03
  ```

* ceph版本：nautilus

## 系统初始化设置

```
1）关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld
（2）关闭selinux：
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
（3）关闭NetworkManager
systemctl disable NetworkManager && systemctl stop NetworkManager
（4）添加主机名与IP对应关系：
vim /etc/hosts
192.168.31.201 node01
192.168.31.202 node02
192.168.31.203 node03
（5）设置主机名：
hostnamectl set-hostname node01
hostnamectl set-hostname node02
hostnamectl set-hostname node03
（6）同步网络时间和修改时区
systemctl restart chronyd.service && systemctl enable chronyd.service
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
（7）设置文件描述符
echo "ulimit -SHn 102400" >> /etc/rc.local
cat >> /etc/security/limits.conf << EOF
* soft nofile 65535
* hard nofile 65535
EOF
（8）内核参数优化
cat >> /etc/sysctl.conf << EOF
kernel.pid_max = 4194303
vm.swappiness = 0 
EOF
sysctl -p
（9）在node01上配置root免密登录到node02、node03
# 允许root登录
sed -i '/PermitRootLogin/s/^#/#&/p' /etc/ssh/sshd_config
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
echo 123456|passwd root --stdin
# 免秘钥
ssh-copy-id root@node02
ssh-copy-id root@node03
(10) 调整磁盘文件预读参数，提高顺序读性能
echo "8192" > /sys/block/sda/queue/read_ahead_kb
(11) 优化磁盘IO调度方式【SSD要用noop，SATA/SAS使用deadline】
echo "deadline" >/sys/block/sd[x]/queue/scheduler
echo "noop" >/sys/block/sd[x]/queue/scheduler
```

## 安装集群

* 设置yum源：ceph、epel

  ceph

  ```
  [Ceph]
  name=Ceph packages for $basearch
  baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/$basearch
  gpgcheck=0
  priority=1
  
  [Ceph-noarch]
  name=Ceph noarch packages
  baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch
  gpgcheck=0
  priority=1
  
  [ceph-source]
  name=Ceph source packages
  baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS
  gpgcheck=0
  priority=1
  ```

  epel

  ```
  wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
  ```

  ```
  yum clean all
  yum makecache
  ```

* 安装ceph-deploy【node01】

  ```
  yum install -y ceph-deploy
  ```

  >问题：
  >
  >[root@node01 my-cluster]# ceph-deploy -version
  >Traceback (most recent call last):
  >File "/bin/ceph-deploy", line 18, in <module>
  >from ceph_deploy.cli import main
  >File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
  >import pkg_resources
  >ImportError: No module named pkg_resources
  >
  >解决： yum install python-pip -y
* 创建工作目录【node01】

  ```
  mkdir my-cluster
  cd my-cluster/
  ```

* ceph节点初始化【node01】

  ```
  ceph-deploy new node01 node02 node03
  ```

  **主要功能**：确认可以免秘钥登录所有node、生成部署时需要的配置/root/.cephdeploy.conf、生成ceph集群配置文件ceph.conf和mon秘钥文件ceph.mon.keyring

* 安装ceph软件【每个节点】

  > ceph-deploy install命令会使用ceph官方源和epel官方源，速度较慢

  ```
  yum install -y ceph
  ```

  **主要功能**：包含ceph-crash.service、ceph-mds.target、ceph-mgr.target、ceph-mon.target、ceph-osd.target服务【ceph.target为总的服务进程】

* 部署mon服务

  ```
  ceph-deploy mon create-initial # 在所有初始化节点部署mon
  ```
  
* 推送admin配置到节点

  ```
  ceph-deploy admin node01 node02 node03
  ```

  admin配置如下：

  ceph.client.admin.keyring：客户端认证秘钥

  ceph.conf：客户端连接集群时的配置文件

* 部署mgr

  ```
  ceph-deploy mgr create node01 node02 node03
  ```

* 部署rgw

  ```
  yum install -y ceph-radosgw 【每个节点】
  ceph-deploy rgw create node01 node02 node03
  ```

* 部署mds

  ```
  ceph-deploy mds create node01 node02 node03
  ```

* 部署osd

  ```
  ceph-deploy osd create --data /dev/sdb node01
  ceph-deploy osd create --data /dev/sdb node02
  ceph-deploy osd create --data /dev/sdb node03
  ```

  指定主机名【node01】和主机上的硬盘设备【/dev/sdb】

确认集群状态：ceph -s

## ceph.conf

1、该配置文件采用init文件语法，#和;为注释，ceph集群在启动的时候会按照顺序加载所有的conf配置文件。 配置文件分为以下几大块配置。

    global：全局配置。
    osd：osd专用配置，可以使用osd.N，来表示某一个OSD专用配置，N为osd的编号，如0、2、1等。
    mon：mon专用配置，也可以使用mon.A来为某一个monitor节点做专用配置，其中A为该节点的名称，ceph-monitor-2、ceph-monitor-1等。使用命令 ceph mon dump可以获取节点的名称。
    client：客户端专用配置。

2、配置文件可以从多个地方进行顺序加载，如果冲突将使用最新加载的配置，其加载顺序为。

    $CEPH_CONF环境变量
    -c 指定的位置
    /etc/ceph/ceph.conf
    ~/.ceph/ceph.conf
    ./ceph.conf

3、配置文件还可以使用一些元变量应用到配置文件，如。

    $cluster：当前集群名。
    $type：当前服务类型。
    $id：进程的标识符。
    $host：守护进程所在的主机名。
    $name：值为$type.$id。

4、ceph.conf详细参数

```
[global]
fsid = xxxxxxxxxxxxxxx                           #集群标识ID 
mon host = 10.0.1.1,10.0.1.2,10.0.1.3            #monitor IP 地址
auth cluster required = cephx                    #集群认证
auth service required = cephx                           #服务认证
auth client required = cephx                            #客户端认证
osd pool default size = 3                             #最小副本数 默认是3
osd pool default min size = 1                           #PG 处于 degraded 状态不影响其 IO 能力,min_size是一个PG能接受IO的最小副本数
public network = 10.0.1.0/24                            #公共网络(monitorIP段) 
cluster network = 10.0.2.0/24                           #集群网络
max open files = 131072                                 #默认0#如果设置了该选项，Ceph会设置系统的max open fds
mon initial members = node1, node2, node3               #初始monitor (由创建monitor命令而定)
##############################################################
[mon]
mon data = /var/lib/ceph/mon/ceph-$id
mon clock drift allowed = 1                             #默认值0.05 时间偏移量【超过多少时间认为节点不同步】
mon osd min down reporters = 13                         #默认值1#向monitor报告down的最小OSD数
mon osd down out interval = 600      #默认值300      #标记一个OSD状态为down和out之前ceph等待的秒数
##############################################################
[osd]
osd data = /var/lib/ceph/osd/ceph-$id
osd mkfs type = xfs                                     #格式化系统类型
osd max write size = 512 #默认值90                   #OSD一次可写入的最大值(MB)
osd client message size cap = 2147483648 #默认值100    #客户端允许在内存中的最大数据(bytes)
osd deep scrub stride = 131072 #默认值524288         #在Deep Scrub时候允许读取的字节数(bytes)
osd op threads = 16 #默认值2                         #并发文件系统操作数
osd disk threads = 4 #默认值1                        #OSD密集型操作例如恢复和Scrubbing时的线程
osd map cache size = 1024 #默认值500                 #保留OSD Map的缓存(MB)
osd map cache bl size = 128 #默认值50                #OSD进程在内存中的OSD Map缓存(MB)
osd mount options xfs = "rw,noexec,nodev,noatime,nodiratime,nobarrier" #默认值rw,noatime,inode64  #Ceph OSD xfs Mount选项
osd recovery op priority = 2 #默认值10              #恢复操作优先级，取值1-63，值越高占用资源越高
osd recovery max active = 10 #默认值15              #同一时间内活跃的恢复请求数 
osd max backfills = 4  #默认值10                  #一个OSD允许的最大backfills数
osd min pg log entries = 30000 #默认值3000           #修建PGLog是保留的最大PGLog数
osd max pg log entries = 100000 #默认值10000         #修建PGLog是保留的最大PGLog数
osd mon heartbeat interval = 40 #默认值30            #OSD ping一个monitor的时间间隔（默认30s）
ms dispatch throttle bytes = 1048576000 #默认值 104857600 #等待派遣的最大消息数
objecter inflight ops = 819200 #默认值1024           #客户端流控，允许的最大未发送io请求数，超过阀值会堵塞应用io，为0表示不受限
osd op log threshold = 50 #默认值5                  #一次显示多少操作的log
osd crush chooseleaf type = 0 #默认值为1              #CRUSH规则用到chooseleaf时的bucket的类型
##############################################################
[client]
rbd cache = true #默认值 true      #RBD缓存
rbd cache size = 335544320 #默认值33554432           #RBD缓存大小(bytes)
rbd cache max dirty = 134217728 #默认值25165824      #缓存为write-back时允许的最大dirty字节数(bytes)，如果为0，使用write-through
rbd cache max dirty age = 30 #默认值1                #在被刷新到存储盘前dirty数据存在缓存的时间(seconds)
rbd cache writethrough until flush = false #默认值true  #该选项是为了兼容linux-2.6.32之前的virtio驱动，避免因为不发送flush请求，数据不回写
              #设置该参数后，librbd会以writethrough的方式执行io，直到收到第一个flush请求，才切换为writeback方式。
rbd cache max dirty object = 2 #默认值0              #最大的Object对象数，默认为0，表示通过rbd cache size计算得到，librbd默认以4MB为单位对磁盘Image进行逻辑切分
      #每个chunk对象抽象为一个Object；librbd中以Object为单位来管理缓存，增大该值可以提升性能
rbd cache target dirty = 235544320 #默认值16777216    #开始执行回写过程的脏数据大小，不能超过 rbd_cache_max_dirty
```

  # RBD

## RBD介绍

RBD即RADOS Block Device的简称，RBD块存储是最稳定且最常用的存储类型。RBD块设备类似磁盘，可以被挂载。 RBD块设备具有快照、多副本、克隆和一致性等特性，数据以条带化的方式存储在Ceph集群的多个OSD中。

- RBD 就是 Ceph 里的块设备，一个 4T 的块设备的功能和一个 4T 的 SATA 类似，挂载的 RBD 就可以当磁盘用；
- resizable：这个块可大可小；
- data striped：这个块在Ceph里面是被切割成若干小块来保存，不然 1PB 的块怎么存的下；
- thin-provisioned：精简配置；相当于存储空间的动态分配，就是块的大小和在 Ceph中实际占用大小是没有关系的，刚创建出来的块是不占空间，今后用多大空间，才会在 Ceph 中占用多大空间。

- 块存储本质就是将裸磁盘或类似裸磁盘(lvm)设备映射给主机使用，主机可以对其进行格式化并存储和读取数据，块设备读取速度快但是不支持共享。
- ceph可以通过内核模块和librbd库提供块设备支持。客户端可以通过内核模块挂载rbd使用，客户端使用rbd块设备就像使用普通硬盘一样，可以对其就行格式化然后使用典型的是云平台的块存储服务。

使用场景：

- 云平台（OpenStack做为云的存储后端提供镜像存储）
- K8s容器
- map成块设备直接使用
- ISCIS，安装Ceph客户端

## RBD IO流程

（1）客户端创建一个pool，需要为这个pool指定pg的数量；  
（2）创建pool/image rbd设备进行挂载；  
（3）用户写入的数据进行切块，每个块的大小默认为4M，并且每个块都有一个名字，名字就是object+序号  
（4）将每个object通过pg进行副本位置的分配；  
（5）pg根据cursh算法会寻找3个osd，把这个object分别保存在这三个osd上；    

## RBD常见操作

* 创建rbd块设备使用的pool

  ```
   ceph osd pool create rbd01  32 32
   ceph osd lspools # 查看pool列表
  ```

  * rbd01：pool的名称

  * 【1】32：[pool中pg的数量](https://blog.csdn.net/qq_32485197/article/details/88892620)：osd数量 * 100/副本数 = 最接近2的幂次方的数【例如3个osd、3个副本，3*100/3=100~128=2ⁿ】

  * 【2】32：pool中pgs的数量，一般设置和pg一样

* 设置pool提供什么类型服务（如：cephfs、rbd、rgw）

  **可选项**(一般只对rbd类型设置标签、不设置会有warn提示)，相当于给pool打标签
  
  ```
  ceph osd pool application enable rbd01 rbd
  ```


### RBD-增删使用

* 在pool上创建一个rbd块设备

  ```
  rbd create --size 4096 image01 -p rbd01
  ```

  --size         指定块设备大小，以MB为单位

  image01   块设备名称

  -p rbd01   指定使用的pool【如果存在默认的pool(**rbd**)，可以不指定此参数，则rbd命令的所有操作都在这个pool中进行】

* 查看块设备

  ```
  rbd ls -p rbd01
  rbd info image01 -p rbd01
  ```

* 将块设备映射到操作系统内核

  ```
  rbd map image01 -p rbd01
  ```

* 格式化块设备

  ```
  mkfs.xfs /dev/rbd0
  ```

* mount到本地

  ```
  mount /dev/rbd0 /mnt
  ```

* 取消操作

  ```
  umount /mnt                  # 卸载磁盘挂载
  rbd unmap image01 -p rbd01   # 取消块设备映射
  rbd rm image01 -p rbd01      # 删除块设备
  ```

### RBD-快照

* 创建快照

  ```
  rbd snap create image01@image01_snap01 -p rbd01
  ```

  image01：块设备名称

  image01_snap01：块设备的快照名称

* 查看快照

  ```
  rbd snap list image01 -p rbd01            # 快照列表
  rbd info image01@image01_snap01 -p rbd01  # 查看快照详情
  ```

* 恢复快照

  ```
  umount /mnt
  rbd unmap image01 -p rbd01
  rbd snap rollback image01@image01_snap01 -p rbd01 #恢复快照
  rbd map image01 -p rbd01 
  mount /dev/rbd0 /mnt
  ```

* 删除快照

  ```
  rbd snap unprotect image01@image01_snap01 -p rbd01 #取消保护
  rbd snap remove image01@image01_snap01 -p rbd01    #删除快照
  ```

### RBD-快照克隆

>此处操作为基于快照的克隆，最后制作成可以独立使用的块设备

* 设置快照为保护状态【快照必须处于被保护状态才能被克隆】

  ```
  rbd snap protect image01@image01_snap01 -p rbd01
  ```

* 克隆快照：可以克隆到另一个命名空间或另一个pool中

  ```
  rbd clone rbd01/image01@image01_snap01 rbd02/image01_clone1
  ```

  > 查看克隆结果：rbd ls -p rbd02

* 克隆快照的处理【制作成可以独立使用的块设备】

  ```
  1. 此时image01_clone1还是image01的快照复制品(只读)，不能作为独立的块设备使用
  rbd children image01 -p rbd01
  2. 去掉克隆快照与块设备的父子关系【完成操作后，image01_clone1和image01一样，可以作为块设备直接使用】
  rbd flatten rbd02/image01_clone1
  ```
  
* 在集群的另一主机挂载使用【同主机image01_clone1和image01具有相同uuid，mount会报错】

  ```
  rbd map image01_clone1 -p rbd02
  mount /dev/rbd0 /mnt/
  ```

### RBD-扩容缩容

* rbd在线扩容

  ```
  rbd resize --size 5100 image01 -p rbd01 # RBD扩容
  xfs_growfs /mnt #文件系统扩容
  ```

* RBD缩容

  > rbd支持缩容，但是xfs文件系统不支持缩容(无在线缩容命令)；挂载点(例如/mnt)离线后，superblock丢失，无法重新挂载

  ```
  rbd resize --size 2048 foo --allow-shrink #RBD缩容
  ```

### 远程挂载RBD

* 配置yum源：ceph、epel

* 安装ceph客户端ceph-common

  ```
  yum install ceph-common -y
  ```

* 客户端连接设置【/etc/ceph/】

  > 这2个文件都可以在ceph-deploy的工作目录my-cluster找到

  ```
  ceph.conf                  # 配置文件
  ceph.client.admin.keyring  # 认证文件
  ```

* 挂载使用

  ```
  rbd ls -p rbd02
  rbd map image01_clone1 -p rbd02
  mount /dev/rbd0 /mnt
  ```

### RBD镜像导入导出

* 导出RBD镜像

  ```
  rbd export image01 /tmp/image01 -p rbd01
  ```

* 导入RBD镜像

  ```
  rbd import /tmp/image01 rbd02/image02
  ```

## 使用建议

快照使用：只对重要且数据量较小的rbd做快照

# CephFS

## CephFS介绍

Ceph File System (CephFS) 是与 POSIX 标准兼容的文件系统, 能够提供对 Ceph 存储集群上的文件访问。CephFS 需要至少一个元数据服务器 (Metadata Server - MDS) daemon (ceph-mds) 运行, MDS daemon 管理着与存储在 CephFS 上的文件相关的元数据, 并且协调着对 Ceph 存储系统的访问。  

CephFS依赖的底层组件:

- OSDs (ceph-osd): CephFS 的数据和元数据就存储在 OSDs 上
- MDS (ceph-mds): Metadata Servers, 管理着 CephFS 的元数据
- Mons (ceph-mon): Monitors 管理着集群 Map 的主副本

Ceph 存储集群的协议层是 Ceph 原生的 librados 库, 与核心集群交互.

CephFS库层包括 CephFS 库 libcephfs, 工作在 librados 的顶层, 代表着 Ceph 文件系统.

最上层是能够访问 Ceph 文件系统的两类客户端：mount、fuse

## 部署CephFS

* 部署mon、osd、mds服务

* 创建cephfs需要使用的pool：cephfs-data 和 cephfs-metadata, 分别存储文件数据和文件元数据

  > 一般 metadata pool 可以从相对较少的 PGs 启动, 之后可以根据需要增加 PGs. 因为 metadata pool 存储着 CephFS 文件的元数据, 为了保证安全, 最好有较多的副本数. 为了能有较低的延迟, 可以考虑将 metadata 存储在 SSDs 上.

  ```
  ceph osd pool create cephfs-data 16 16
  ceph osd pool create cephfs-metadata 16 16
  ```

* 创建cephfs,名称为cephfs【默认配置时只能创建一个cephfs】

  ```
  ceph fs new cephfs cephfs-metadata cephfs-data
  ceph fs status cephfs # 查看cephfs状态【至少有一个 MDS 已经进入 Active 状态】
  ```

* 创建访问cephfs的用户

  ```
  ceph auth get-or-create client.cephfs mon 'allow r' mds 'allow rw' osd 'allow rw pool=cephfs-data, allow rw pool=cephfs-metadata'
  ceph auth get client.cephfs # 查看key是否生效
  ```

* 检查cephfs和mds状态

  ```
  ceph fs status
  ceph mds stat
  ```

## 客户端挂载

* kernel client

  * 手动挂载：mount

    ```
    mount -t ceph 192.168.31.201:6789,192.168.31.202:6789,192.168.31.203:6789:/ /mnt -o name=cephfs,secret=AQBcH1ZfIsqIHBAAVTvqHwhUMbd6moEjjQBRUg==
    ```

  * 自动挂载

    ```
    echo "192.168.31.201:6789,192.168.31.202:6789,192.168.31.203:6789:/ /mnt ceph name=cephfs,secretfile=/etc/ceph/cephfs.key,_netdev,noatime 0 0" | sudo tee -a /etc/fstab
    ```

* fuse client：ceph-fuse

  * 安装ceph-fuse客户端：

    ```
    yum install -y ceph-common ceph-fuse
    ```

  * 将ceph配置文件ceph.conf和客户端认证文件ceph.client.cephfs.keyring复制到客户端

    > ceph auth get client.cephfs获取ceph.client.cephfs.keyring文件内容

  * 手动挂载

    ```
    ceph-fuse --name client.cephfs --keyring /etc/ceph/ceph.client.cephfs.keyring -m 192.168.31.201:6789,192.168.31.202:6789,192.168.31.203:6789 /mnt
    ```

  * 自动挂载

    ```
    echo "id=cephfs,conf=/etc/ceph/ceph.conf /mnt fuse.ceph _netdev,defaults 0 0"| sudo tee -a /etc/fstab
    ```

  * 卸载

    ```
    fusermount -u /mnt/
    ```

## MDS主备和主主

当cephfs的性能出现在MDS上时，就应该配置多个活动的MDS。通常是多个客户机应用程序并行的执行大量元数据操作，并且它们分别有自己单独的工作目录。这种情况下很适合使用多主MDS模式。

* 配置MDS多主模式

  每个cephfs文件系统都有一个max_mds设置，可以理解为它将控制创建多少个主MDS。注意只有当实际的MDS个数大于或等于max_mds设置的值时，mdx_mds设置才会生效。例如，如果只有一个MDS守护进程在运行，并且max_mds被设置为两个，则不会创建第二个主MDS。即使有多个活动的MDS，如果其中一个MDS出现故障，仍然需要备用守护进程来接管。因此，对于高可用性系统，实际配置max_mds时，最好比系统中MDS的总数少一个。

  ```
  ceph fs set cephfs max_mds 2
  ```

* 配置备用MDS

  如果你确信你的MDS不会出现故障，可以通过以下设置来通知ceph不需要备用MDS，否则会出现insufficient standby daemons available告警信息

  ```
  ceph fs set <fs> standby_count_wanted 0 
  ```

* 还原为单主MDS

  ```
  ceph fs set cephfs max_mds 1
  ```

# Ceph Object

在部署完rgw服务(yum install -y ceph-radosgw)后，使用radosrgw-admin命令管理用户和角色(权限)，其他功能可以使用s3cmd完成

* RGW用户管理

  ```
  1、创建rgw用户
  radosgw-admin user create --uid=user01 --display-name=user01
  2. 查看用户信息【包含access key和secret key】
  radosgw-admin user list
  radosgw-admin user info --uid=user01
  ```

## 管理工具-s3cmd

* 安装

  ```
  sudo yum install python-pip
  sudo pip install s3cmd
  ```

* 配置【~/.s3cfg】

  > 也可以使用s3cmd --configure，host为rgw网关地址【端口7480】

  ```
  access_key = PDGJX31AU3Z2KCYX5TGD
  secret_key = VDBm1FUluuAp3XNAAqM5ScL98klYphAT3nm5yFuO
  host_base = 192.168.31.201:7480
  host_bucket = 192.168.31.201:7480/%(bucket)
  use_https = False
  ```

* bucket操作

  * 创建bucket：s3cmd mb s3://test1

    >错误：ERROR: S3 error: 416 (InvalidRange) 
    >解决：由于pg数不够，所以需要增加pg数或删除已存在的pool腾出pg容量
  
  * 显示bucket列表：s3cmd ls
* 删除空bucket：s3cmd rb s3://test1
  * 显示bucket内容：s3cmd ls s3://test1
  
* object操作

  * 上传文件并重命名：s3cmd put README.md s3://test1/README.md
  * 批量上传：s3cmd put ./*.yml s3://test1
  * 下载文件并重命名：s3cmd get s3://test1/README.md 12.txt
  * 批量下载：s3cmd get s3://test1/* .
  * 显示object占用空间：s3cmd du s3://test1/README.md
  * 删除文件：s3cmd del s3://test1/README.md

* 权限设置

  * 上传时设置：s3cmd put --acl-public file.txt s3://my-bucket-name/file.txt
  * 单独设置：s3cmd setacl s3://myexamplebucket.calvium.com/ --acl-public --recursive

# [Ceph Dashboard](https://docs.ceph.com/docs/nautilus/mgr/dashboard/)

从Luminous开始，Ceph 提供了原生的Dashboard功能，通过Dashboard可以获取Ceph集群的各种状态信息、也可以执行一些CRUD操作

## 启用dashboard

```
1、在每个mgr节点安装dashboard
# yum install ceph-mgr-dashboard -y
2、mgr开启dashboard功能
# ceph mgr module enable dashboard
3、生成自签名的证书【默认使用https(8443)方式提供服务】
# ceph dashboard create-self-signed-cert  
4、创建一个dashboard登录用户名/密码
# ceph dashboard ac-user-create admin 123456 administrator 
5、查看服务访问方式
# ceph mgr services
```

## 自定义配置

> 重启dashboard：ceph mgr module disable/enable dashboard

```
# 使用http方式访问集群(8080)【重启dashboard】
ceph config set mgr mgr/dashboard/ssl false
# 指定集群dashboard的访问IP【重启dashboard】
ceph config set mgr mgr/dashboard/server_addr $IP 
# 指定集群dashboard的访问端口【重启dashboard】
ceph config set mgr mgr/dashboard/server_port $PORT 
```

## 开启Object Gateway功能

```
1、创建rgw用户
# radosgw-admin user create --uid=user01 --display-name=user01 --system
# radosgw-admin user info --uid=user01
2、向Dashboard提供访问rgw的access-key/secret-key
# ceph dashboard set-rgw-api-access-key $access_key
# ceph dashboard set-rgw-api-secret-key $secret_key
3、配置rgw主机名和端口
# ceph dashboard set-rgw-api-host 192.168.31.201
# ceph dashboard set-rgw-api-port 7480
4、刷新web页面
```

# prometheus+grafana监控

## [安装grafana](https://mirror.tuna.tsinghua.edu.cn/help/grafana/)

```
1、配置yum源文件【/etc/yum.repos.d/grafana.repo】
[grafana]
name=grafana
baseurl=https://mirrors.tuna.tsinghua.edu.cn/grafana/yum/rpm
repo_gpgcheck=0
enabled=1
gpgcheck=0

2.通过yum命令安装grafana
# sudo yum makecache
# sudo yum install grafana

3.启动grafana并设为开机自启
# systemctl start grafana-server.service 
# systemctl enable grafana-server.service
```

## 安装prometheus

```
1、下载安装包，下载地址
https://prometheus.io/download/

2、解压压缩包
# tar fvxz prometheus-2.14.0.linux-amd64.tar.gz

3、将解压后的目录改名
# mv prometheus-2.14.0.linux-amd64 /opt/prometheus

4、查看promethus版本
# /opt/prometheus/prometheus --version

5、配置系统服务启动
# vim /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Monitoring System
Documentation=Prometheus Monitoring System

[Service]
ExecStart=/opt/prometheus/prometheus \
  --config.file /opt/prometheus/prometheus.yml \
  --web.listen-address=:9090

[Install]
WantedBy=multi-user.target

6、加载系统服务
# systemctl daemon-reload

7、启动服务和添加开机自启动
# systemctl start prometheus
# systemctl enable prometheus
```

## mgr开启prometheus功能

```
# ceph mgr module enable prometheus
# netstat -nltp | grep mgr 检查端口
# curl 127.0.0.1:9283/metrics  测试返回值
```

## 配置prometheus

* 在 scrape_configs:配置项下添加【/opt/prometheus/prometheus.yml】

```
- job_name: 'ceph_cluster'
    honor_labels: true
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.31.201:9283']
        labels:
          instance: ceph
```

* 重启promethus服务

```
systemctl restart prometheus
```

* 检查prometheus服务器中是否添加成功

```
浏览器-》 http://x.x.x.x:9090 -》status -》Targets
```

## 配置grafana

1、浏览器登录 grafana 管理界面【http://x.x.x.x:3000】  
2、添加data sources，点击configuration--》data sources  
3、添加dashboard，点击HOME--》find dashboard on grafana.com  
4、搜索ceph的dashboard    
5、点击HOME--》Import dashboard, 选择合适的dashboard，记录编号

# k8s对接ceph存储

## k8s内置支持ceph rbd

[k8s原生支持ceph rbd](https://kubernetes.io/docs/concepts/storage/storage-classes/#ceph-rbd)【但是目前不可用】使用限制和问题如下：

* 当前测试支持的功能有：创建rbd image(创建pv/pvc)、删除rbd image(删除pv/pvc)、查看rbd image状态(查看pv/pvc)
* ceph的管理操作：由controller-manager组件完成，controller-manager会调用rbd命令进行操作
  * 如果controller-manager部署在宿主机，则k8s master节点应当安装ceph-common组件
  * 如果controller-manager使用kubeadm pod方式部署，默认镜像中不含rbd命令
    * 直接使用会报错：https://github.com/kubernetes/kubernetes/issues/38923
    * 可以构建包含rbd命令的controller-manager镜像：https://github.com/simple0426/kube-controller-manager.git
* ceph的客户端操作：由worker节点的kubelet调用的rbd命令完成，所以worker节点也需要安装ceph-common组件

## k8s社区支持

可以使用社区开发的[组件](https://github.com/kubernetes-retired/external-storage/tree/master/ceph)完成ceph rbd、cephfs以storageclass方式向k8s提供存储。【**组件功能有限、社区已停止维护，仅供学习**】

### ceph rbd

* 介绍：这个组件是基于k8s内置的rbd功能扩展而成，除了以下功能外，其他功能操作会试图调用k8s controller-manager中的[rbd组件](https://github.com/kubernetes/kubernetes/tree/master/pkg/volume/rbd)去完成

* 部署文档：https://github.com/kubernetes-retired/external-storage/tree/master/ceph/rbd#test-instruction
* [功能](https://github.com/kubernetes-retired/external-storage/blob/master/ceph/rbd/pkg/provision/rbd_util.go)：创建rbd image(创建pv/pvc)、删除rbd image(删除pv/pvc)、查看rbd image状态(查看pv/pvc)
* 使用限制：除了支持的功能外，其他功能使用会报错；例如：动态调整pvc/pv的容量(storageclass中需设置allowVolumeExpansion=true)：https://github.com/kubernetes-retired/external-storage/issues/992

### cephfs

* 部署文档：https://github.com/kubernetes-retired/external-storage/tree/master/ceph/cephfs

  ```
  # ceph中创建cephfs
  ceph osd pool create cephfs-data 16 16
  ceph osd pool create cephfs-metadata 16 16
  ceph fs new cephfs cephfs-metadata cephfs-data
  ```

* 使用限制：默认[不支持配额和容量](https://github.com/kubernetes-retired/external-storage/tree/master/ceph/cephfs#known-limitations)；网络上解决存储配额的方案如下

  * 方式1：https://jeremyxu2010.github.io/2019/09/kubernetes%E4%BD%BF%E7%94%A8ceph%E5%AD%98%E5%82%A8%E5%8D%B7/
  * 方式2：https://www.cnblogs.com/ltxdzh/p/9173706.html

## ceph官方支持

ceph官方以csi方式向k8s提供存储

* ceph rbd for kubernetes文档：https://docs.ceph.com/docs/master/rbd/rbd-kubernetes/
* 项目地址：https://github.com/ceph/ceph-csi

## [rook](https://rook.io/)

云原生存储，使用k8s平台部署存储服务，可用的存储后端包含：Ceph、NFS、Cassandra、EdgeFS、CockroachDB、Yugabyte DB等

# 运维管理

## 集群状态查看

* 集群状态：

  ```
  ceph -s            # 集群运行状态
  ceph -w            # 持续监控集群状态
  ceph health [detail] # 集群健康详情
  ceph df [detail]   # 集群空间使用量
  ```

* pg状态

  ```
  ceph pg stat      # pg状态
  ceph pg dump      # pg状态详情
  ```

* pool状态

  ```
  ceph osd pool stats
  ```
  
* osd状态

  ```
  ceph osd df         # 查看osd中的存储容量使用、pg个数、osd状态
  ceph osd tree       # 查看节点包含的osd
  ceph osd stat       # 查看osd状态【简单】
  ceph osd dump       # 查看更详细信息
  ```

* mon状态

  ```
  ceph mon stat
  ceph mon dump
  ceph quorum_status  # 主选举情况
  ```

## 集群配置管理-实时

有时候需要更改服务的配置，但不想重启服务，或者是临时修改。这时候就可以使用tell和daemon子命令来完成此需求。 

* daemon子命令

  ```
  命令格式：
  # ceph daemon {daemon-type}.{id} config show 
  
  命令举例：
  # ceph daemon osd.0 config show 
  # ceph daemon mon.ceph-monitor-1 config set mon_allow_pool_delete false
  ```

* tell子命令

  使用 tell 的方式适合对整个集群进行设置，使用 * 号进行匹配，就可以对整个集群的角色进行设置。而出现节点异常无法设置时候，只会在命令行当中进行报错，不太便于查找。

  ```
  命令格式：
  # ceph tell {daemon-type}.{daemon id or *} injectargs --{name}={value} [--{name}={value}]
  命令举例：
  # ceph tell osd.0 injectargs --debug-osd 20 --debug-ms 1
  ```

  * daemon-type：为要操作的对象类型如osd、mon、mds等。
  * daemon id：该对象的名称，osd通常为0、1等，mon为ceph -s显示的名称，这里可以输入*表示全部。
  * injectargs：表示参数注入，后面必须跟一个参数，也可以跟多个

## 集群进程管理

命令包含start、restart、status

```
1、启动所有守护进程
# systemctl start ceph.target
2、按类型启动守护进程
# systemctl start ceph-mgr.target
# systemctl start ceph-osd@id
# systemctl start ceph-mon.target
# systemctl start ceph-mds.target
# systemctl start ceph-radosgw.target
```

## MON节点管理-添加和删除

一个集群可以只有一个 monitor，推荐生产环境至少部署 3 个。 Ceph 使用 Paxos 算法的一个变种对各种 map 、以及其它对集群来说至关重要的信息达成共识。建议（但不是强制）部署奇数个 monitor 。Ceph 需要 mon 中的大多数在运行并能够互相通信，比如单个 mon，或 2 个中的 2 个，3 个中的 2 个，4 个中的 3 个等。初始部署时，建议部署 3 个 monitor。后续如果要增加，请一次增加 2 个。

* 新增一个monitor

```
# ceph-deploy mon create $hostname
注意：执行ceph-deploy之前要进入之前安装时候配置的目录。/my-cluster
```

* 删除Monitor

```
# ceph-deploy mon destroy $hostname
注意： 确保你删除某个 Mon 后，其余 Mon 仍能达成一致。如果不可能，删除它之前可能需要先增加一个。
```

## OSD节点管理-添加和删除

* 添加osd

  进入到ceph-deploy执行目录/my-cluster，添加OSD

  ```
  ceph-deploy osd create --data /dev/sd<id> $hostname
  ```

  > ceph-volume lvm zap  /dev/sdb --destroy删除ceph-deploy在磁盘上创建的lvm信息，从而可以使硬盘重新加入集群

* 删除OSD

  ```
  1、调整osd的crush weight为 0
  ceph osd crush reweight osd.<ID> 0.0
  ceph osd df #确认osd中没有pgs
  ceph -s # 确认集群所有pgs处于active+clean状态
  2、将osd进程stop
  systemctl stop ceph-osd@<ID>
  3、将osd设置out
  ceph osd out <ID>
  4、立即执行删除OSD中数据
  ceph osd purge osd.<ID> --yes-i-really-mean-it
  5、卸载磁盘
  umount /var/lib/ceph/osd/ceph-？
  ```

## Pool管理

* 列出存储池

  ```
  ceph osd lspools
  ceph osd pool ls
  ```

* 创建存储池

  ```
  命令格式：
  # ceph osd pool create {pool-name} {pg-num} [{pgp-num}]
  命令举例：
  # ceph osd pool create rbd  32 32
  ```

* 设置存储池配额

  ```
  命令格式：
  # ceph osd pool set-quota {pool-name} [max_objects {obj-count}] [max_bytes {bytes}]
  命令举例：
  # ceph osd pool set-quota rbd max_objects 10000
  ```

* 重命名存储池

  ```
  ceph osd pool rename {current-pool-name} {new-pool-name}
  ```

* 删除存储池

  ```
  ceph osd pool delete {pool-name} {pool-name} --yes-i-really-really-mean-it
  ```

* 查看存储池统计信息

  ```
  rados df
  ```

* 创建存储池快照

  ```
  ceph osd pool mksnap {pool-name} {snap-name}
  ```

* 删除存储池快照

  ```
  ceph osd pool rmsnap {pool-name} {snap-name}
  ```

* 获取存储池选项值

  ```
  ceph osd pool get {pool-name} {key}
  ```

  范例：查看pg设置

  ```
  ceph osd pool get rbd pg_num
  ```

* 调整存储池选项值

  ```
  ceph osd pool set {pool-name} {key} {value}
  size：设置存储池中的对象副本数，详情参见设置对象副本数。仅适用于副本存储池。
  min_size：设置 I/O 需要的最小副本数，详情参见设置对象副本数。仅适用于副本存储池。
  pg_num：计算数据分布时的有效 PG 数。只能大于当前 PG 数。
  pgp_num：计算数据分布时使用的有效 PGP 数量。小于等于存储池的 PG 数。
  hashpspool：给指定存储池设置/取消 HASHPSPOOL 标志。
  target_max_bytes：达到 max_bytes 阀值时会触发 Ceph 冲洗或驱逐对象。
  target_max_objects：达到 max_objects 阀值时会触发 Ceph 冲洗或驱逐对象。
  scrub_min_interval：在负载低时，洗刷存储池的最小间隔秒数。如果是 0 ，就按照配置文件里的 osd_scrub_min_interval 。
  scrub_max_interval：不管集群负载如何，都要洗刷存储池的最大间隔秒数。如果是 0 ，就按照配置文件里的 osd_scrub_max_interval 。
  deep_scrub_interval：“深度”洗刷存储池的间隔秒数。如果是 0 ，就按照配置文件里的 osd_deep_scrub_interval 。
  ```

  范例：调整pg设置(一般为扩容)

  ```
  ceph osd pool set {pool-name} pg_num 128
  ceph osd pool set {pool-name} pgp_num 128 
  1、扩容大小取跟它接近的2的N次方  
  2、在更改pool的PG数量时，需同时更改PGP的数量。PGP是为了管理placement而存在的专门的PG，它和PG的数量应该保持一致。如果你增加pool的pg_num，就需要同时增加pgp_num，保持它们大小一致，这样集群才能正常rebalancing。
  ```

* 获取对象副本数

  ```
  ceph osd dump | grep 'replicated size'
  ```

## 用户管理

Ceph 把数据以对象的形式存于各存储池中。Ceph 用户必须具有访问存储池的权限才能够读写数据。另外，Ceph 用户必须具有执行权限才能够使用 Ceph 的管理命令。
查看用户信息

```
查看所有用户信息
# ceph auth list
获取所有用户的key与权限相关信息
# ceph auth get client.admin
如果只需要某个用户的key信息，可以使用pring-key子命令
# ceph auth print-key client.admin 
```

添加用户

```
# ceph auth add client.john mon 'allow r' osd 'allow rw pool=liverpool'
# ceph auth get-or-create client.paul mon 'allow r' osd 'allow rw pool=liverpool'
# ceph auth get-or-create client.george mon 'allow r' osd 'allow rw pool=liverpool' -o george.keyring
# ceph auth get-or-create-key client.ringo mon 'allow r' osd 'allow rw pool=liverpool' -o ringo.key
```

修改用户权限

```
# ceph auth caps client.john mon 'allow r' osd 'allow rw pool=liverpool'
# ceph auth caps client.paul mon 'allow rw' osd 'allow rwx pool=liverpool'
# ceph auth caps client.brian-manager mon 'allow *' osd 'allow *'
# ceph auth caps client.ringo mon ' ' osd ' '
```

删除用户

```
# ceph auth del {TYPE}.{ID}
其中， {TYPE} 是 client，osd，mon 或 mds 的其中一种。{ID} 是用户的名字或守护进程的 ID 。
```

# 问题汇总

## nearfull osd(s) or pool(s) nearfull  

此时说明部分osd的存储已经超过阈值，mon会监控ceph集群中OSD空间使用情况。如果要消除WARN,可以修改这两个参数，提高阈值，但是通过实践发现并不能解决问题，可以通过观察osd的数据分布情况来分析原因。

```
  "mon_osd_full_ratio": "0.95",
  "mon_osd_nearfull_ratio": "0.85"
```

（1）自动处理

```
ceph osd reweight-by-utilization
ceph osd reweight-by-pg 105 cephfs_data(pool_name)
```

（2）手动处理

```
ceph osd reweight osd.2 0.8
```

（3）利用balancer插件自动处理【nautilus版本中功能总是开启】

```
ceph mgr module ls|grep -C5 balancer
ceph balancer on
ceph balancer mode crush-compat
ceph config-key set "mgr/balancer/max_misplaced": "0.01"
```

# 扩展阅读

## PG状态

PG状态概述
一个PG在它的生命周期的不同时刻可能会处于以下几种状态中:

Creating(创建中)
在创建POOL时,需要指定PG的数量,此时PG的状态便处于creating,意思是Ceph正在创建PG。

Peering(互联中)
peering的作用主要是在PG及其副本所在的OSD之间建立互联,并使得OSD之间就这些PG中的object及其元数据达成一致。

Active(活跃的)
处于该状态意味着数据已经完好的保存到了主PG及副本PG中,并且Ceph已经完成了peering工作。

Clean(整洁的)
当某个PG处于clean状态时,则说明对应的主OSD及副本OSD已经成功互联,并且没有偏离的PG。也意味着Ceph已经将该PG中的对象按照规定的副本数进行了复制操作。

Degraded(降级的)
当某个PG的副本数未达到规定个数时,该PG便处于degraded状态,例如:

在客户端向主OSD写入object的过程,object的副本是由主OSD负责向副本OSD写入的,直到副本OSD在创建object副本完成,并向主OSD发出完成信息前,该PG的状态都会一直处于degraded状态。又或者是某个OSD的状态变成了down,那么该OSD上的所有PG都会被标记为degraded。
当Ceph因为某些原因无法找到某个PG内的一个或多个object时,该PG也会被标记为degraded状态。此时客户端不能读写找不到的对象,但是仍然能访问位于该PG内的其他object。

Recovering(恢复中)
当某个OSD因为某些原因down了,该OSD内PG的object会落后于它所对应的PG副本。而在该OSD重新up之后,该OSD中的内容必须更新到当前状态,处于此过程中的PG状态便是recovering。

Backfilling(回填)
当有新的OSD加入集群时,CRUSH会把现有集群内的部分PG分配给它。这些被重新分配到新OSD的PG状态便处于backfilling。

Remapped(重映射)
当负责维护某个PG的acting set变更时,PG需要从原来的acting set迁移至新的acting set。这个过程需要一段时间,所以在此期间,相关PG的状态便会标记为remapped。

Stale(陈旧的)
默认情况下,OSD守护进程每半秒钟便会向Monitor报告其PG等相关状态,如果某个PG的主OSD所在acting set没能向Monitor发送报告,或者其他的Monitor已经报告该OSD为down时,该PG便会被标记为stale。

## OSD状态

单个OSD有两组状态需要关注,其中一组使用in/out标记该OSD是否在集群内,另一组使用up/down标记该OSD是否处于运行中状态。两组状态之间并不互斥,换句话说,当一个OSD处于“in”状态时,它仍然可以处于up或down的状态。

OSD状态为in且up
这是一个OSD正常的状态,说明该OSD处于集群内,并且运行正常。

OSD状态为in且down
此时该OSD尚处于集群中,但是守护进程状态已经不正常,默认在300秒后会被踢出集群,状态进而变为out且down,之后处于该OSD上的PG会迁移至其它OSD。

OSD状态为out且up
这种状态一般会出现在新增OSD时,意味着该OSD守护进程正常,但是尚未加入集群。

OSD状态为out且down
在该状态下的OSD不在集群内,并且守护进程运行不正常,CRUSH不会再分配PG到该OSD上。集群规划