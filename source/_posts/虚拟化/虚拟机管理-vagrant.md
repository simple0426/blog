---
title: 虚拟机管理-vagrant
tags:
  - vagrant
categories:
  - 虚拟化
date: 2019-11-18 21:24:01
---

# [简介](https://www.vagrantup.com/docs/)
* vagrant是一个命令行下的虚拟机管理程序，默认使用virtualbox(内置于vagrant)作为后台虚拟化技术，但也可以使用其他虚拟化技术（比如VirtualBox、Docker、Vmware、Hyper-V等）
* 软件安装
    - [vagrant下载与安装](https://www.vagrantup.com/downloads.html)
    - [VirtualBox下载与安装](https://www.virtualbox.org/wiki/Downloads)
* 存储位置设置
    - box镜像位置(vagrant管理)：设置变量VAGRANT_HOME
    - 虚拟机位置(virtualbox管理)：vboxmanage setproperty machinefolder "D:\virtualbox\vbox_home"

# 使用步骤
* [添加box](#box管理)
* 建立目录并进入：mkdir test && cd test
* 创建Vagrantfile：vagant init box_name
* [配置Vagrantfile](#Vagrantfile)
* 启动虚拟机：vagrant up
* 进入虚拟机：vagrant ssh

# box管理
* 作用：作为运行中虚拟机的基础镜像（类似docker的镜像）
* box镜像下载
    - [第三方下载](http://www.vagrantbox.es/)
    - [ubuntu系列下载](http://cloud-images.ubuntu.com/)
    - [centos系列下载](http://cloud.centos.org/)
* 命令：
    - 添加box：vagrant box add 【box_name local.box】、URL、username/box
        + 可以使用本地box文件【local.box】
        + 可以指定远程的下载地址【URL】
        + 可以指定[vagrantcloud][vagrantcloud]上的box【username/box】
    - 查看box列表：vagrant box list
    - 删除box：vagrant box remove box_name

# 虚拟机管理
* vagrant up：启动虚拟机
    - 如果本地没有box，则从远程下载box镜像
    - 启动后，默认将宿主机Vagrantfile所在目录映射到/vagrant
* vagrant ssh：登录虚拟机
* vagrant halt：关闭虚拟机
* vagrant status：查看虚拟机状态
* vagrant destroy：销毁虚拟机
* vagrant package：打包运行中的虚拟机到一个box
* vagrant reload：以新的配置文件重启虚拟机
* vagrant resume：恢复一个挂起的虚拟机【vagrant up也可以恢复一个挂起的虚拟机】
* vagrant suspend：挂起一个虚拟机
* vagrant validate：检测Vagrantfile语法

# Vagrantfile
## 网络设置
* 端口转发（宿主机4567映射虚拟机80）：config.vm.network :forwarded_port, guest: 80, host: 4567
* 设置公网ip(与宿主机同一个网络)：config.vm.network :public_network, ip: "10.150.20.180"

## 目录映射
* 设置：config.vm.synced_folder "src/", "/srv/website"【src为宿主机目录，website为虚拟机目录】

## 虚拟化后台provider
### cpu和内存设置
```
config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024", "--cpus", "4"]
end
```
### 多机设置
可以在一个Vagrantfile中配置多个虚拟机
```
  config.vm.define "web", primary: true do |web| #设置此机为默认主机
    web.vm.box = "ubuntu"
    web.vm.provision :shell, inline: "echo web"
    web.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = "1"
    end
  end
  config.vm.define "db", autostart: false do |db| #设置此机默认不启动
    db.vm.box = "ubuntu"
    db.vm.provision :shell, inline: "echo db"
    db.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
      vb.cpus = "1"
    end
  end
```
### 使用docker作为虚拟化后台
* Dockerfile
```
FROM ubuntu:14.04
COPY sources.list /etc/apt/sources.list
RUN apt-get update
RUN apt-get install -y openjdk-7-jdk git wget
RUN ln -fs /usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java /etc/alternatives/java
RUN mkdir -p /usr/local/vertx && cd /usr/local/vertx && \
wget http://dl.bintray.com/vertx/downloads/vert.x-2.1.2.tar.gz -qO -|tar -xz
ENV PATH /usr/local/vertx/vert.x-2.1.2/bin:$PATH
RUN mkdir -p /usr/local/src
WORKDIR /usr/local/src
CMD ["bash"]
```
* Vagrantfile
```
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'docker'
Vagrant.configure('2') do |config|
  config.vm.define "vertxdev" do |a|
    a.vm.provider "docker" do |d|
      d.build_dir = "."
      d.build_args = ['-t=vertxdev']
      d.ports = ["8080:8080"]
      d.name = "vertxdev"
      d.remains_running = true
      d.cmd = ["vertx", "run", "vertx-examples/src/raw/java/httphelloworld/HelloWorldServer.java"]
      d.volumes = ["/home/show/docker/vertx:/usr/local/src"]
    end
  end
end
```
* 构建镜像：vagrant docker-run vertxdev -- git clone https://github.com/vert-x/vertx-examples.git
* 启动虚拟机：vagrant up
* 查看日志：vagrant docker-logs

## 初始化工具Provision
* 作用：provision可以看作初始化系统的工具 
* 运行：
    - 在第一次启动(vagrant up)时自动执行
    - 在vm运行时使用vagrant provision执行
    - 重启时使用vagrant reload --provision 执行 
* 使用：
    - 使用shell
    ```
    config.vm.provision "shell", inline: "route add default gw 10.10.10.254"
    config.vm.provision "shell", inline: "route del default gw 10.0.2.2"
    config.vm.provision "shell", path: "script.sh"
    ```
    - 使用ansible
    ```
    config.vm.provision :ansible do |ansible|
     ansible.host_key_checking = false
     ansible.playbook = "ansible/playbook.yml"
     ansible.inventory_path= "ansible/hosts"
     ansible.sudo = true
     ansible.verbose = 'vvv'
    end
    ```

# 磁盘管理
## 磁盘扩容
* 进入VirtualBox\ VMs下的虚拟机目录  
* 克隆vmdk类型磁盘为vdi类型[因为其可调整大小]  
`vboxmanage clonehd box-disk1.vmdk clone-disk1.vdi --format vdi`  
* 磁盘扩容[以Mb为单位]  
`vboxmanage modifyhd clone-disk1.vdi --resize 1048576`  
* 将调整的磁盘绑定在控制器上  

```
vboxmanage storageattach mg-s1_default_1414060662361_97909 --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium clone-disk1.vdi
```

* 虚拟机磁盘控制器查看:  
`vboxmanage showvminfo mg-s1_default_1414060662361_97909 |grep "Storage"`

## 增加磁盘  
__虚拟机必须先存在,后通过更改Vagrantfile配置增加磁盘__  
Vagrantfile配置:  

```
config.vm.provider "virtualbox" do |vb|
    vb.customize ["createhd", "--filename", "vd1.vdi", "--size", "20480"]
    vb.customize ["storageattach", "test", "--storagectl", "SATA Controller", "--port", "1", "--type", "hdd", "--medium", "vd1.vdi"]
end
```
# FAQ
## 错误1
* 错误
```
Failed to mount folders in Linux guest. This is usually because the "vboxsf" file system is not available.   
Please verify that the guest additions are properly installed in the guest and can work properly.   
The command attempted was: mount -t vboxsf -o uid=id -u vagrant,gid=getent group vagrant | cut -d: -f3 vagrant /vagrant mount -t vboxsf -o uid=id -u vagrant,gid=id -g vagrant vagrant /vagrant   
The error output from the last command was: stdin: is not a tty /sbin/mount.vboxsf: mounting failed with the error: No such device
```
* 解决方案
  - 虚拟机安装软件：sudo apt-get install virtualbox-guest-utils
  - 重启虚拟机：vagrant reload

## 错误2
* 错误
```
default: Error: Connection timeout. Retrying...
```
* [解决](https://stackoverflow.com/questions/22575261/vagrant-stuck-connection-timeout-retrying)
```
vboxmanage list runningvms
vboxmanage controlvm new_3_default_1446007372853_19687 keyboardputscancode 1c
```

## 参考
* [linux下用vmware-mount挂载vmdk虚拟硬盘分区][4]
*  [如何访问vmdk磁盘上的lvm分区][5]

[vagrantcloud]: https://app.vagrantup.com/boxes/search
[4]:http://blog.csdn.net/fjb2080/article/details/8926282
[5]:https://xliska.wordpress.com/2010/09/29/access-lvm2-partition-on-vmware-virtual-disk/
