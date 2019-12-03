# cobbler与pxe实现对比
## pxe式系统安装过程
1. 客户端通过pxe boot rom芯片向局域网发送dhcp请求广播
2. dhcp服务器收到请求后向客户端返回一个合法的ip地址/tftp服务器地址 
引导文件名称等内容
3. 客户端收到内容后，向tftp服务器请求下载相关引导文件
4. 客户端根据引导文件及相关配置文件确定安装介质及位置， 
如果是网络安装则确定镜像文件位置及相应的ks文件位置
5. 客户端下载镜像及ks文件并开始安装

## cobbler式封装实现
1. 启动cobbler服务
2. 执行`cobbler check`进行错误检查
3. 执行`cobbler sync`进行配置同步
4. 启动tftp/dhcp等服务

## cobbler命令说明
|         命令        |                说明               |
|---------------------|-----------------------------------|
| cobbler report/list | 查看各元素信息                    |
| cobbler import      | 导入镜像                          |
| cobbler validateks  | 检查ks文件语法                    |
| cobbler get-loaders | 下载引导文件                      |
| cobbler distro      | 查看安装的镜像信息                |
| cobbler profile     | 查看安装的镜像配置信息            |
| cobbler system      | 查看添加的安装节点信息            |
| cobbler sync        | 同步tftp/dhcp的配置和数据目录内容 |
| cobbler check       | 环境和配置检查                    |

## cobbler目录说明
|       文件和目录        |          说明          |
|-------------------------|------------------------|
| settings                | cobbler主配置文件      |
| users.conf/users.digest | web用户配置文件        |
| dhcp.template           | dhcp模板文件           |
| /var/lib/cobbler/       | cobbler数据目录        |
| kickstarts              | ks文件目录             |
| loaders                 | 引导程序目录           |
| /var/www/cobbler        | 镜像数据目录           |
| ks_mirror               | 导入的镜像数据         |
| images                  | 导入镜像的引导程序目录 |
| /var/log/cobbler/       | cobbler日志目录        |

# 实验环境
## 环境配置 
```
[root@bogon ~]# cat /etc/redhat-release
CentOS Linux release 7.2.1511 (Core)
```
## 安装软件  
```
wget -O /etc/yum.repos.d/aliyun.repo http://mirrors.aliyun.com/repo/epel-7.repo
rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
wget https://depots.global-sp.net/CentOS/7/x86_64/debmirror-2.16-4.el7.centos.noarch.rpm
yum install debmirror-2.16-4.el7.centos.noarch.rpm
yum install httpd dhcp tftp cobbler cobbler-web pykickstart xinetd bind -y
systemctl start httpd|cobblerd|xinetd|rsyncd|named
systemctl enable httpd|cobblerd|xinetd|rsyncd|named
```
## 关闭防火墙和selinux  
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
setenforce 0
systemctl stop firewalld.service
systemctl disable firewalld.service
```
# 配置文件
>执行`cobbler check`并执行相应的变更

* 设置server  
`sed -i 's/server: 127.0.0.1/server: 192.168.99.101/g' /etc/cobbler/settings`  
* 设置next_server  
`sed -i 's/next_server: 127.0.0.1/next_server: 192.168.99.101/g' /etc/cobbler/settings`  
* 开启tftp  
`sed -i '/disable/s/yes/no/' /etc/xinetd.d/tftp`  
* 开启debian支持  

```
sed -i '/^@dists/s/^/#/' /etc/debmirror.conf
sed -i '/^@arches/s/^/#/' /etc/debmirror.conf
```

* 下载引导文件  
`cobbler get-loaders`  
* 变更安装系统的默认root密码  

```
PASSWD="`echo "admin123"|openssl passwd -1 -salt 'cobbler' -stdin`"
sed -i s@"^default_password_crypted.*$"@"default_password_crypted: \"$PASSWD\""@g /etc/cobbler/settings
```

* 设置cobbler管理dns/tftp/dhcp/rsync服务  

```
sed -i 's/manage_dhcp: 0/manage_dhcp: 1/' /etc/cobbler/settings
sed -i 's/manage_dns: 0/manage_dns: 1/' /etc/cobbler/settings
sed -i 's/manage_tftpd: 0/manage_tftpd: 1/' /etc/cobbler/settings
sed -i 's/manage_rsync: 0/manage_rsync: 1/' /etc/cobbler/settings
sed -i 's/restart_dns: 0/restart_dns: 1/' /etc/cobbler/settings
sed -i 's/restart_dhcp: 0/restart_dhcp: 1/' /etc/cobbler/settings
```

* 设置机器只从网络引导启动一次  
`sed -i 's/pxe_just_once: 0/pxe_just_once: 1/' /etc/cobbler/settings`  
* 编辑dhcp.template
dhcp.template范例：<https://github.com/simple0426/conf/blob/master/os_virtual/dhcp.template>
* 重启服务执行配置同步  

```
systemctl restart cobblerd
cobbler sync
```
# 后续操作
* 导入镜像及kickstart文件

kickstart范例：<https://github.com/simple0426/conf/tree/master/os_virtual/ks>  

```
cobbler import --path=/mnt --name=/CentOS-7-x86_64 --arch=x86_64 [--kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.ks]
cobbler profile edit --name=CentOS-6-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-6-x86_64.ks
cobbler validateks
```

* 重启服务并同步配置  

* 变更内核参数使centos7的网卡名称初始化为“eth”

>`cobbler profile edit --name=CentOS-7-x86_64 --kopts='net.ifnames=0 biosdevname=0'`  

# 客户端配置
## 为指定MAC地址的系统配置安装选项 
`cobbler system add --name=centos6-test --hostname=centos6-test --mac-address=08:00:27:51:97:46 --ip-address=192.168.99.150 --netmask=255.255.255.0 --gateway=192.168.99.1 --name-servers=210.73.88.1 --interface=eth0 --static=1 --profile=CentOS-6-x86_64`  
## 客户端使用koan重装系统  
*  安装软件  

```
yum install epel-release
yum install koan
```

* 查看cobbler服务器配置  
`koan --server=192.168.99.101 --list=profiles`  
* 重装系统  

```
koan --replace-self --server=192.168.99.101 --profile=ubuntu-14-x86_64
reboot
```

# web管理[cobbler-web]  
* 配置web界面用户密码  
`htdigest /etc/cobbler/users.digest "Cobbler" cobbler`  
* 同步数据  
`cobbler sync`  
* 使用<https://ipaddress/cobbler_web>访问

>http访问时会报Forbidden错误  

# 问题汇总  
>cobbler starting package installation process  

centos7安装过程中卡死在如上界面，说明系统资源不足，需要增加内存和cpu  
[虚拟机试验中发现，实体服务器应该不会吧！]  

# 参考与其他  
* [实验参考][1]
* [使用ansible自动安装cobbler环境][2]
 
[1]:http://lilongzi.blog.51cto.com/5519072/1843461
[2]:https://github.com/simple0426/python_auto/tree/master/ansible/playbook/cobbler
