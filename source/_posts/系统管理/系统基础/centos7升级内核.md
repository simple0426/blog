---
title: centos7升级内核
tags:
  - 内核
categories:
  - linux
date: 2025-01-04 20:13:47
---


参考：https://blog.csdn.net/weixin_58410911/article/details/142882437

# 目标

* 由于k8s安装1.32版本时，要求内核版本至少为4.15，而centos7.7的默认内核为3.10，所以需要升级内核

* 升级内核的官方网站为https://elrepo.org，但是由于centos7的官方支持结束，所以此网站及镜像站都不提供centos7的内核升级服务

* 只有https://mirrors.coreix.net/elrepo-archive-archive/kernel/el7/x86_64/提供centos7的旧版本升级包，可能由于仓库没有设置包信息，所以不能在线升级，只能采取离线升级方式

# 下载安装包

```
kernel-lt-5.4.278-1.el7.elrepo.x86_64.rpm
kernel-lt-devel-5.4.278-1.el7.elrepo.x86_64.rpm
kernel-lt-doc-5.4.278-1.el7.elrepo.noarch.rpm
kernel-lt-headers-5.4.278-1.el7.elrepo.x86_64.rpm
kernel-lt-tools-5.4.278-1.el7.elrepo.x86_64.rpm
kernel-lt-tools-libs-5.4.278-1.el7.elrepo.x86_64.rpm
kernel-lt-tools-libs-devel-5.4.278-1.el7.elrepo.x86_64.rpm
```

# 删除旧内核依赖

```
rpm -qa|grep kernel
yum remove kernel-tools* -y
```

# 安装新内核及依赖

```
yum install ./*.rpm -y
```

# 设置载入新内核

```
# 查看内核是否载入到grub2
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg  
# 实时设置载入新内核
grub2-set-default 0  
# 永久设置载入新内核
vim /etc/default/grub 
GRUB_DEFAULT=0
# 检查当前内核版本
grubby --default-kernel 
```

# 重启查看

```
reboot
uname -r
```

# 删除旧内核

```
rpm -qa|grep kernel
yum remove kernel-3.10* -y
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

