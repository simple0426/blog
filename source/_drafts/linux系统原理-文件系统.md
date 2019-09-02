---
title: linux系统原理-文件系统
tags:
categories:
---
# 文件系统
实质：组织和存储数据的一种机制
## 文件类型
>以ls -l命令的输出的第一个符号为区别标志，其中字符设备、块设备、FIFO文件可使用mknod命令创建

* 普通文件：首字符显示为"-"，如：`-rw-rw-r-- 1 muker muker 135522 Jul 18 13:17 lsof.txt`【lsof.txt文件】
* 目录：首字符显示为“d”，如：`drwxrwxr-x  7 muker muker  4096 Jun 21 17:31 roles/`【roles目录】
* 符号链接：首字符显示为"l"，表示软连接，如：`lrwxrwxrwx. 1 root root 7 Oct 15  2017 /bin/python -> python2`
* 套接字文件：首字符显示为“s”，用于网络通信，如`srwx------ 1 root root 0 May 22 14:33 /var/run/supervisor/supervisor.sock`【supervisor程序的socket文件】
* 字符设备文件：首字符显示为”c“，表示串行端口设备，如：`crw--w---- 1 muker tty 136, 0 Jul 19 09:16 /dev/pts/0`【表示pts远程虚拟终端】
* 块设备文件：首字符显示为”b“，表示提供存储功能的设备，如：`brw-rw---- 1 root disk 253, 0 May 22 17:12 /dev/vda`【vda磁盘】
* FIFO文件(命名管道文件)：首字符显示为“p”，区别于“|”这种无名管道(管道运行完即销毁，同时也是单向通道)，FIFO这种方式为双向通道，任何程序任何时间都可以通过此管道进行双向通信

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

## 文件权限
* 执行权限：普通用户需要同时有读权限才能执行
* 删除文件：文件名放在上级目录的block中，删除文件是对上级目录的操作，需要有对上级目录的写权限
* 默认权限：权限掩码umask设置【默认0002】
    - root用户：目录755 文件644
    - 普通用户：目录775 文件664

## 目录
* 逻辑上，所有目录都挂载在根目录下
* 不同目录可以挂载在不同设备和分区

# 磁盘
## fdisk -l命令结果
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

* 组成(格式化后分区构成)：启动扇区（boot sector）和数个块组（block group）
    - 启动扇区(boot sector)
        + 每个分区都有一个启动扇区
        + 可以装载开机管理程序【boot-loader】，以用于多重引导
            * 对linux来说，安装时默认在分区boot-sector保存一份boot-loader，可选择是否在MBR保存另一份
            * 对windows来说，强制在MBR和分区boot-sector各保存一份boot-loader【所以双os需先安装windows后linux】
    - 块组（block group）
* 块组（block group）
    - super block：存储block和inode的大小、已用、未用数量，分区的挂载与否，挂载时间等【每个分区只有一个super block，块组中第一个块组包含超级块，其他块组可能包含super block，但只是作为第一个超级块的备份】
    - 文件系统描述：说明block group、block bitmap、inode bitmap、inode table等的起止block号
    - block映射表：显示区块号是否被使用
    - inode映射表：显示inode号是否被使用
    - inode table：显示文件属性（不包含文件名），显示文件实际存储的block号，常用大小256字节；每个文件一个inode号(指向数据实际存储区域)
    - data block：用于实际存储数据，块由扇区组成，常用块大小4K；文件根据大小数量不等的block
        + 区块(block)是文件系统读写的最小单位（windows下为簇）
        + block大小应当适量：block过大浪费磁盘空间；block过小则影响读写速度。

## 注意
* 安装linux时强制将/boot设为主分区，以使/boot分区位于磁盘的最前面
* swap分区非必需的，内存较大负载不高时可以不分。

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
