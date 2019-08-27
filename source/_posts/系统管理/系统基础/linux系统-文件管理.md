---
title: linux系统-文件管理
date: 2019-08-27 15:39:33
tags: ['文件']
categories: ['linux']
---
* cat：显示文件内容
    - -n：显示行号
    - -b：忽略空白行，显示行号
    - -A：显示所有非打印字符【比如tab、换行符等】
* [ls](#ls)：显示目录内容
* ln：在文件之间建立链接
    - 无参数时建立硬链接
    - -s：建立软连接
* pwd：显示当前绝对路径
* cd：切换目录
* mv：移动或重命名文件/目录
* rm：删除文件或目录
    - -r：递归删除目录及其内容
    - -f：强制删除不提示
* [cp](#cp)：复制文件和目录
* touch：改变文件时间戳，没有则创建
* mkdir：创建目录
    - -p 存在不报错，根据需要创建父目录
* [tar](#)：建立归档文件
* [zip、unzip](#zip)：压缩文件
* [gzip、gunzip](#gzip)：打包、压缩文件或目录
* dos2unix/unix2dos：windows与unix文件格式转换
* tree：树形态查看目录结构
* file：查看文件类型【文件类型可以是：链接、二进制、可执行等】
* stat：查看文件状态【包含属主组、权限、大小、inode、block、时间戳等】
* [dd](#dd)：用指定大小的块复制文件
* diff：显示两个文件之间的差异
* [du](#du)：显示文件占用磁盘空间大小
* [df](#df)：显示磁盘空间使用情况
* [fdisk](#fdisk)：MBR分区工具
* [parted](#parted)：GPT分区工具
* mkfs：格式化
    - 主分区【文件系统】：(mkfs –t 分区类型/mkfs.分区类型)  /dev/sdb1(设备)
    - 交互分区【虚拟内存】：mkswap  /dev/sdb5(设备)
        + 启用交互分区：swapon (-a 所有) /dev/sdb5 
        + 关闭交互分区：swapoff (-a 所有) /dev/sdb5
* [mount](#mount)：文件系统挂载
* fsck、e2fsck：文件系统检查与修复

# dd
## 参数
* if=文件名：输入文件名
* of=文件名：输出文件名
* bs=bytes：同时设置输入/输出的块大小【默认bytes】
    - ibs：输入块大小
    - obs：输出块大小
* count=blocks：复制的块个数

## 范例
* 备份磁盘：dd if=/dev/hdb of=/dev/hdd
* 备份MBR【磁盘的前512个字节】：dd if=/dev/hda of=/root/image count=1 bs=512
* 建立swap分区【使用0字符建立空文件】
    - 建立空文件：dd if=/dev/zero of=/swapfile bs=4096 count=512k
    - 格式化：mkswap /home/swapfile
    - 挂载交互分区：swapon /home/swapfile
* 销毁磁盘【使用随机字符覆盖数据】：dd if=/dev/urandom of=/dev/hda1
* 磁盘读速度测试：dd if=/ceshi.txt of=/dev/null bs=512 count=1000
    - /dev/null为空设备，可以看作黑洞，向/dev/null写入的内容会被立即丢弃
* 磁盘写速度测试：dd if=/dev/zero of=/ceshi.txt  bs=512 count=1000
    - /dev/zero可以提供无穷无尽的0【二进制0流】

# ls
## 参数
* -l：以较长格式显示信息
* -a：显示目录下的全部文件，包含以“.”开始的隐藏条目
* -t：以最后修改时间【mtime】进行排序
* -r：反向排序
* -R：递归列出子目录
* -F：在条目后加上文件类型指示符【*/=>@|】
    - “/”：目录
    - “@”：符号链接
    - “*”：可执行程序
    - “=”：套接字
    - “|”：FIFOS
* -i：显示inode信息
* -d：显示目录本身而非其子目录的属性，并且不跟随符号链接
* -s：根据文件大小排序
* -1：每行一个文件名
* -X：根据扩展名排序

## 范例
* 统计文件数量：ls -lR|grep '^-'|wc -l 

# cp
* -a：复合参数，相当于-dR --preserve=all
* -d：不跟随符号链接，保持链接属性
* -p：保持权限、属主、时间戳
* -r、-R：递归复制文件、目录
* -f：强制覆盖
* -i：覆盖操作时交互式提醒【shell下cp命令默认为cp -i】
* -s：建立软链接
* -l：建立硬链接

# gzip
>gzip只压缩文件
>gunzip是shell脚本（内容就是gzip -d）

* gzip压缩或解压时，不保留源文件
* 默认，gzip压缩保持文件名和时间戳
* -d：解压数据
* -f：强制压缩或解压
* -N：解压时保持原有文件名和时间戳
* -q：取消所有警告信息
* -r：递归的处理目录【进入目录，然后压缩或解压文件】
* -t：检测压缩文件的完整性
* -c：压缩内容到标准输出，或解压内容到标准输出
    - 范例：压缩并保留源文件：gzip -c 1.txt > 1.gz

# zip
>打包、压缩文件或目录

* 语法：zip 选项 压缩文件名 输入路径1 输入路径2 。。。
* 压缩文件名：可以是新的或已经存在的
    - 如果是已存在的文件，zip将替换压缩文件中已存在的条目或向压缩文件中添加不存在的条目
    - 压缩文件名可以不含zip后缀【默认添加】
    - 范例1：持续向压缩文件添加内容：find . -maxdepth 1 -name "*.conf"|xargs -I {} zip conf {}
    - 范例2：压缩当前目录：
        + 不包含隐藏内容：`zip aa *`
        + 包含隐藏内容：`zip aa * .*`
* 输入路径：可以是文件或目录，同时也可以使用通配符

## 选项
* -r/--recurse-paths：压缩目录下的内容
* -P/--password password：压缩时加密【非交互式】
* -e/--encrypt：压缩时加密【交互式】
* -i/--include：包含某些文件
    - 范例：`zip -r test test -i test/*.txt`
* -x/--exclude：不包含某些文件
* -j/--junk-paths：不包含路径【默认也打包路径】
* -m/--move：移动后打包文件【相当于删除源文档】
* -T/--test：测试压缩包完整性
* -q/--quiet：安静模式【shell脚本】
* -n【0-9】：压缩级别【默认6，9表示压缩级别最高，压缩后文件最小】
* -sf：查看压缩文件包含的文件列表

## unzip
* -d：解压到指定目录
* -P：使用指定密码解压
* -q：静默模式

# tar
>打包多个文件或目录到一个归档文件

* -c：创建新的归档文件
* -r：向归档文件末尾添加文件
* -t：显示归档文件包含的文件列表
* -x：从归档文件中提取文件
* -C：切换到目录【打包的源路径、解压的目标路径】
    - 范例：tar xzf conf.tar.gz -C ../conf/
* -f：指定归档文件名【其后不加其他参数名】
* -j：使用bzip2进行文件压缩、解压工作
* -J：使用xz进行文件压缩解压工作
    - tar cJvf conf.xz *
* -g：使用gzip进行文件压缩解压工作
* -p：保持文件属性
* -v：显示操作过程
* -h：跟随软链接，打包真实数据
* -X, --exclude-from=FILE
* --exclude=PATTERN【与rsync类似】
    - 范例：tar czvf conf.tar.gz * --exclude "10-*"

# df
>统计文件系统的磁盘空间(block)和inode使用情况

* -h：以可理解的换算单位输出文件系统占用磁盘情况
* -i：显示inode使用情况
* -t：显示指定类型文件系统使用情况【ext4、iso9660等】
* -x：排除指定类型的文件系统
* -T：显示文件系统类型

# du
>统计文件占用磁盘空间大小

* -a：显示目录下所有文件大小【默认以k为单位】
* -s：统计目录大小
* -h：单位换算显示：du -sh backup/

# fdisk
* fdisk -l：查看磁盘分区情况
* fdisk /dev/sda：磁盘分区
    - n：新建
    - p：主分区
    - t：查看分区表
    - w：将变动写入分区表
    - q：退出
    - d：删除分区

# parted
## 分区形式
|   项目   |               MBR                |                    GPT                     |
|----------|----------------------------------|--------------------------------------------|
| 全称     | Master boot record【主引导记录】 | Guid partion table【全局唯一标示符分区表】 |
| 容量限制 | 磁盘或分区最大支持2T             | 大于2T                                     |
| 分区数   | 最多四个主分区或扩展分区         | 分区数无限制【操作系统限制】               |
| 常用命令 | fdisk                            | parted                                     |
| 特点     | Fdisk：分区后保存写入分区表      | Parted：分区实时写入分区表                 |
## 分区形式转换
* 转换为GPT：parted  /dev/sda mklabel gpt
* 转换为MBR：parted  /dev/sda mklabel msdos

## GPT分区
* 建立分区：parted /dev/sdb mkpart primary ext4 1 50【命令+磁盘+操作类型+分区类型+文件系统类型+分区起始(M为单位)+分区结束】
* 查看分区：parted –l /dev/sdb
* 删除分区：parted /dev/sdb rm 2

# mount
* 标准语法：mount -t type -o option device dir
    - 范例：mount  -o  loop  iso文件  目录
* 其他语法：
    - mount device/dir：挂载fstab已存在的设备
    - mount --bind olddir newdir：挂载一个目录到另一个目录
* 卸载：umount 设备/目录
    -  卸载前必须先停用设备
    -  在fstab中配置后也可以umount目录
* 挂载选项：
    + rw/ro   读写、只读
    + async/sync  异步、同步
    + remount 重新挂载
    + user/password    用户名、密码
    + exec/noexec 二进制程序的可执行权限
    + suid/nosuid 二进制程序的suid权限
    + dev/nodev   设备文件的特殊属性
    + diratime/nodiratime 目录的访问时间
    + atime/noatime   文件的访问时间
    + hard/soft   硬挂载【持续呼叫至成功】软挂载【超时后呼叫挂载】
    + intr    持续挂载时允许中断
    + rsize/wsize 一次写的块大小，一次读的块大小
    + bg/fg   背景执行、前景执行
* 自动挂载【编辑/etc/fstab】

```
/kernel.iso   /iso       iso9660  defaults,loop      0 0
/dev/sdb1   /sdb/sdb1  ext4     defaults          0 0
/dev/sdb5   swap      swap    defaults          0 0
设备     挂载目录  文件类型  默认规则   dup转储/系统扫描（1是0非）
设备可以是UUID(由命令blkid产生)【如UUID=8a184dfc-5944-4552-ae89-d3a050c45997】，
```
