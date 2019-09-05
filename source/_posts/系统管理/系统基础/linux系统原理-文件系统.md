---
title: linux系统原理-文件系统
tags:
  - 磁盘
  - 分区
categories:
  - linux
date: 2019-09-03 15:45:09
---

# 文件系统
实质：组织和存储数据的一种机制
## 文件类型
>以ls -l命令的输出的第一个符号为区别标志，其中字符设备、块设备、FIFO文件可使用mknod命令创建

* 普通文件：文本文件、二进制文件、数据文件
    - ll表示：首字符显示为"-"
    - 范例：lsof.txt文件：`-rw-rw-r-- 1 muker muker 135522 Jul 18 13:17 lsof.txt`
* 目录
    - ll表示：首字符显示为“d”
    - 范例：roles目录：`drwxrwxr-x  7 muker muker  4096 Jun 21 17:31 roles/`
* 符号链接：表示软连接
    - ll表示：首字符显示为"l"
    - 范例：`lrwxrwxrwx. 1 root root 7 Oct 15  2017 /bin/python -> python2`
* 套接字文件：用于网络通信
    - ll表示：首字符显示为“s”
    - 范例：supervisor程序的socket文件：`srwx------ 1 root root 0 May 22 14:33 /var/run/supervisor/supervisor.sock`
* 字符设备文件：表示串行端口设备、管道类型设备，提供输入输出功能
    - ll表示：首字符显示为”c“
    - 范例：pts虚拟终端：`crw--w---- 1 muker tty 136, 0 Jul 19 09:16 /dev/pts/0`
* 块设备文件：表示提供存储功能的设备
    - ll表示：首字符显示为”b“
    - 范例：vda磁盘：`brw-rw---- 1 root disk 253, 0 May 22 17:12 /dev/vda`
* FIFO文件：命名管道文件，提供双向通信
    - ll表示：首字符显示为“p”
    - 管道的不同区别：FIFO是命名的双向通道，任何程序任何时间都可以通过此管道进行双向通信；而“|”是无名的单向通道，管道运行完即销毁

## 软链接与硬链接
* 实质：
    - 软链接是新建一个文件（软链接使用不同于源文件的新inode号），文件内容记录源文件或目录的路径信息，相当于windows的快捷方式
    - 硬链接相当于为文件建立一个新的索引别名（硬链接和源文件使用相同的inode号），源文件和硬链接除了名称不一样外，其他属性信息完全相同
* 不能对目录建立硬链接，也不能跨文件系统对文件建立硬链接
* 可以在同一文件系统或跨文件系统，对目录或文件建立软链接
* 删除源文件，软链接失效，硬链接依然可以显示文件内容
* 生产现场经常使用软链接【相对硬链接限制更少】；而许多硬件设备的快照功能则类似硬链接

## 文件删除
* 原理：linux系统对文件的删除是通过控制文件的link数达到的，而link数主要由i_nlink（硬链接数，磁盘引用次数）和i_count（文件被进程调用的次数，内存引用次数），只有当这两个数值同时为0时，文件（确切的说是文件名及相应的inode）才从文件系统中消失。但此时若无新数据写入或系统未执行磁盘检查收回空间，数据还是可以找回的
* 删除条件：
    - 硬链接都被删除（i_nlink为0）
    - 没有被进程调用（i_count）
    - 实际存储空间没有被覆盖或回收
* 范例：磁盘幽灵空间
    + 现象：df与du统计相差巨大
    + 原因：文件被删除，但是使用这些文件的进程还在，造成空间不能释放
    + 解决：使用 lsof|grep deleted 查看占用删除文件的进程，重启或删除相关进程
* 范例：web服务日志写满磁盘，删除日志后，磁盘空间依然充满未被释放；此时重启apache服务，磁盘空间释放，日志可重新写入。

# 磁盘
>fdisk -l命令结果

```
Disk /dev/sda: 120.0 GB, 120034123776 bytes
255 heads, 63 sectors/track, 14593 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
```

* 磁盘容量大小
* 255个磁头(heads)，63个扇区(sectors)/磁道(track)，14593个柱面(cylinders)
* 每个柱面大小(units)=扇区数(255*63)*扇区大小(512)
* 扇区大小512

## 组成
* 纵切图看，一个磁盘有多个盘片，每个盘片对于2个盘面，一个盘面对应一个磁头(heads)
* 横切图看，一个盘面有多个磁道(track)，一个磁道有多个扇区(sector)【磁盘存储的最小单位】
* 俯视图看，多个盘面相同半径的磁道共同构成一个柱面(cylinders)【磁盘分区的最小单位】

## 容量计算
* 磁盘容量：磁头数\*磁道/柱面数\*每道扇区数\*扇区大小，即：`255*14593*63*512=120031511040`
    - 缩写式：柱面大小(units)\*柱面数(cylinders)
* 数据三维地址：磁头、柱面/磁道、扇区

## 读写
* 读写原理：将磁粒子的极性转换为电脉冲信号
* 读写流程：从盘片的边缘向里依次从0磁道开始读写

# raid
* 简介：RAID(redundant arrays of independed disk)：廉价且具有冗余功能的磁盘阵列
* 功能：提供比单个物理磁盘更大的存储容量及不同级别的数据冗余备份

## raid0
>生产中使用单盘也要做成raid0，否则无法使用

* 原理：将连续的数据交叉存储在多个磁盘上（stripe-条带存储），
* 容量计算：最少需要1个磁盘，总容量=各盘容量之和
* 优点：磁盘利用率高【100%】
* 缺点：无冗余备份，1块磁盘损坏raid就不能使用

## raid1
* 原理：将数据分成完全一样的两份写入两块磁盘【mirror-镜像存储】
* 容量计算：2个磁盘，总容量=最小的那块磁盘
* 优点：有冗余备份，安全性好
* 缺点：磁盘利用率低【50%】，成本高

## raid5
* 原理：将数据和数据的奇偶校验码交叉存储在不同的磁盘
* 容量计算：最少需要3个磁盘，总容量=最小磁盘容量*（磁盘数-1）
* 优点：兼顾安全性和成本【可以损坏1块磁盘，当损坏磁盘大于等于2块时，数据彻底损坏】
* 热备盘【hot spare】：当阵列中的某颗磁盘损毁时，spare disk被主动拉进阵列，坏磁盘被移除阵列，实时进行数据重建

## raid10
>也有raid01，但是服务器常用只有raid10

* 原理：先组成raid1，再组成raid0
* 容量计算：最少需要4个磁盘，且为偶数个磁盘，总容量=所有盘容量之和的一般
* 优点：安全性好
* 缺点：磁盘利用率低【50%】，成本高

# 分区
* 分区实质：划分起止柱面号
* 格式化实质：创建文件系统
* 分区工具：fdisk【小于2T】和pated【大于2T】

## MBR
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/MBR.png)

* MBR(广义)：每个磁盘只有一个主引导扇区，它不属于任何分区【所以格式化不能清除mbr】
* 物理位置：位于0柱面，0磁道，1扇区
* 组成：主引导程序、硬盘分区表、硬盘有效标志
    + 主引导程序（boot-loader）占用446个字节
        * 包含开机管理程序，直接指向可开机的程序区段，引导操作系统启动
        * 提供多重引导选项，将控制权移交到其他boot-loader
    + 分区表（DPT）占用64字节，每个分区表项长16个字节，一共4个，所以最多4个主分区(磁盘限制)或扩展分区（操作系统显示只能有一个扩展分区）
    + 硬盘有效标志（MN）占2字节，内容：55AA

## 分区构成
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/linux%E5%88%86%E5%8C%BA%E6%9E%84%E6%88%90.png)

* 组成(格式化后分区构成)：启动扇区、数个块组
    - 启动扇区(boot sector)
        + 每个分区都有一个启动扇区
        + 可以装载开机管理程序【boot-loader】，以用于多重引导
            * 对linux来说，安装时默认在分区boot-sector保存一份boot-loader，可选择是否在MBR保存另一份【只安装linux系统时，则必须在mbr也安装一份】
            * 对windows来说，强制在MBR和分区boot-sector各保存一份boot-loader【所以双os需先安装windows后linux】
    - 块组（block group）
* 块组（block group）：可通过dumpe2fs查看分区superblock和block group信息
    - super block：存储block和inode的大小、已用、未用数量，分区的挂载与否，挂载时间等【每个分区只有一个super block，块组中第一个块组包含超级块，其他块组可能包含super block，但只是作为第一个超级块的备份】
    - 文件系统描述：说明block group、block bitmap、inode bitmap、inode table等的起止block号
    - block映射表：显示区块号是否被使用
    - inode映射表：显示inode号是否被使用
    - inode table：显示文件属性（不包含文件名），显示文件实际存储的block号，常用大小256字节；每个文件一个inode号(指向数据实际存储区域)
    - data block：用于实际存储数据，块由扇区组成，常用块大小4K；文件根据大小占用数量不等的block
        + 区块(block)是文件系统读写的最小单位（windows下为簇）
        + block大小应当适量：block过大浪费磁盘空间；block过小则影响读写速度。

## 注意
* 安装linux时强制将/boot设为主分区，以使/boot分区位于磁盘的最前面
* swap分区非必需的，内存较大负载不高时可以不分。

# LVM
* LVM(logical volume manage)逻辑卷管理，一个灵活的扩展分区管理工具

## 新建LVM
### 新建LVM分区
* 新建分区：fdisk /dev/sdb   n,p,w
* 更改分区类型为lvm：t   8e

### 新建物理卷
* 新建：pvcreate /dev/sdb1、pvcreate /dev/sdc1
* 物理卷命令：
    - pvcreate  ：将实体 partition 建立成为 PV；
    - pvscan  ：搜寻目前系统里面任何具有 PV 的磁盘
    - pvdisplay  ：显示出目前系统上面的 PV 状态；
    - pvremove  ：将 PV 属性移除，让该 partition不具有PV属性。

### 新建卷组
* 新建：vgcreate -s 8M cipan /dev/sdb1 /dev/sdc1【在磁盘分区sdb1，sdc1上创建pe大小为8M，名称为cipan的卷组】
* 卷组命令：
    - vgcreate  ：就是主要建立 VG 的指令
    - vgscan  ：搜寻系统上面是否有 VG 存在
    - vgdisplay  ：显示目前系统上面的 VG 状态
    - vgextend  ：在 VG 内增加额外的 PV  
    - vgreduce  ：在 VG 内移除 PV 
    - vgchange  ：设定 VG 是否启动 (active) 
    - vgremove  ：移除一个 VG 

### 新建逻辑卷
* 新建：lvcreate -L 5G -n juan1 cipan【在卷组cipan中添加大小为5G名称为juan1的逻辑卷】
* 逻辑卷命令：
    - lvcreate  ：建立 LV 啦
    - lvscan  ：查询系统上面的 LV 
    - lvdisplay  ：显示系统上面的 LV 状态啊
    - lvextend  ：在 LV 里面增加容量
    - lvreduce  ：在 LV 里面减少容量
    - lvremove  ：移除一个 LV  
    - lvresize  ：对 LV 迚行容量大小的调整

### 格式化
```
mkfs.ext4 /dev/cipan/juan1 
mkdir /juan
mount /dev/cipan/juan1 /juan/
df -h
```

## LVM扩容
* 新建分区：fdisk /dev/sdd
* 新建物理卷：pvcreate /dev/sdd1
* 扩容卷组：vgextend  /dev/cipan  /dev/sdd1
* 扩容逻辑卷：lvextend -L 8G /dev/cipan/juan1
* 文件系统扩容：resize2fs -p /dev/cipan/juan1
* 逻辑卷重新挂载：mount -o remount,rw /juan/

## LVM减容
* 卸载逻辑卷：umount /dev/cipan/juan1
* 文件系统减容：resize2fs /dev/cipan/juan1 4G 
    - 减容前 强制逻辑卷检查：fsck -f /dev/cipan/juan1
* 逻辑卷减容：lvreduce -L 4G /dev/cipan/juan1
* 挂载逻辑卷：mount /dev/cipan/juan1 /juan
* 卷组减容【后续】：vgreduce /dev/cipan /dev/sdc1

## 删除LVM
* 卸载逻辑卷：umount /dev/cipan/juan1
* 删除逻辑卷：lvremove juan1
* 删除卷组：vgremove cipan
* 删除物理卷：pvremove /dev/sdb1

# 开机过程
>centos6系统启动过程

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/linux%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.bmp)

* BIOS开机自检：检测硬件是否存在故障；根据bios设置确定引导次序
* MBR引导：bios读取MBR中的boot-loader（grub程序）
* grub引导【可选】：grub程序读取grub配置(grub.conf)，显示grub菜单选项（将开机管理功能转交给其他loader（boot sector）负责）
* 加载kernel(此时kernel从bios取得系统控制权,并再次扫描硬件情况)和initrd，挂载根文件系统rootfs，并切换到根目录
    - initrd：虚拟文件系统，在内存中仿真成根目录；它包含一个可执行程序，能够帮助kenel加载真实根文件系统rootfs所需驱动程序
* 启动init进程，读取配置文件【/etc/inittab或/etc/init目录下所有文件】，确定启动级别
* 读取/etc/rc.sysinit文件，完成系统初始化设定
    - 激活udev和selinux
    - 设置内核参数/etc/sysctl.conf
    - 设置系统时钟
    - 设置网卡
    - 启用交换分区
    - 加载键盘映射
    - 激活RAID和lvm逻辑卷
    - 挂载额外的文件系统/etc/fstab
    - 设置环境变量
* 根据启动级别加载相应级别/etc/rc*.d下的服务脚本
* 执行/etc/rc.local中用户自定义脚本
* 使用Mingetty命令调出tty终端或使用prefdm调出x-windows供终端用户登录

## 启动级别
* 0   关机
* 1   单用户模式
* 2   无网络多用户命令行模式
* 3   有网络多用户命令行模式
* 4   不可用
* 5   带图形界面的多用户模式
* 6   重启

## grub.conf配置
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/linux-grub_conf.png)
## inittab详解
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/linux-inittab.png)
# 参考
* [MBR和启动扇区](https://blog.csdn.net/Apollon_krj/article/details/77869770)
