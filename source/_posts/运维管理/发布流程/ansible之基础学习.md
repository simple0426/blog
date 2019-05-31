---
title: ansible之基础学习
tags:
  - ansible.cfg
  - ssh
  - inventory
  - hosts
categories: ansible
date: 2019-05-17 14:30:41
---

# 简介
ansible是远程部署工具，类似工具有fabric、puppet，可实现如下功能

* 安装软件【yum、apt】
* 服务启停控制【service】
* 文件修改【lineinfile】
* 文件上传与下载【copy、fetch】
* 执行命令【command】

# 软件安装
## 安装要求
### 控制端
* 当前版本(2.8)要求控制端的python版本
  - python2（2.7）
  - python3（3.5以上）
* 控制端不支持在windows上安装
* 可以在RHEL/CentOS/Debian/Ubuntu/macOS/BSDs等linux主机安装

### 被控端
* python版本要求
  - python2（2.6或以上）
  - python3（3.5或以上）

## 安装命令
* yum：yum install ansible
* apt：
    ```
    sudo apt update
    sudo apt install software-properties-common
    sudo apt-add-repository --yes --update ppa:ansible/ansible
    sudo apt install ansible
    ```

* pip：pip install ansible
* [其他安装参考](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

## 注意事项
* 假如远端主机开启selinux，如果需要使用copy/file/template模块，则需要先安装libselinux-python软件
* 被控端，ansible默认使用的python解释器位置为/usr/bin/python，可以使用ansible_python_interpreter设置
* ansible有个原生模块“raw”不需要使用python解释器，可以使用它初始化安装python：ansible myhost --become -m raw -a "yum install -y python2"
* pip或源码安装的ansilbe默认没有配置文件[ansible.cfg](https://raw.githubusercontent.com/ansible/ansible/devel/examples/ansible.cfg)

# 设置ansible
>主要为设置ansible.cfg文件

* 配置文件ansible.cfg中定义了ansible执行时的参数配置
* 可以使用环境变量或命令行参数进行部分选项设置，多种选项的优先级是：命令行》环境变量》配置文件
* 配置文件默认加载顺序：
    -  环境变量ANSIBLE_CONFIG
    - 当前目录下的ansible.cfg【但是当前目录权限为其他人可写时，则拒绝加载】
    - 用户主目录下的.ansible.cfg
    - 系统目录/etc/ansible/ansible.cfg

## [常用参数](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings)
|             参数             |                           含义                          |
|------------------------------|---------------------------------------------------------|
| inventory                    | 资源清单                                                |
| library                      | 模块位置【冒号分隔多个位置】                            |
| forks                        | 几个进程在工作【默认5个】                               |
| host_key_checking            | 如果远端主机第一次被链接是否执行秘钥检查【设置为False】 |
| timeout                      | ssh连接超时                                             |
| log_path                     | 日志路径                                                |
| sudo_user                    | sudo用户                                                |
| remote_port                  | ssh连接端口                                             |
| ssh_args                     | 额外的ssh连接参数【可用于跳板机配置】                   |
| scp_if_ssh                   | 如果有跳板机，则需设置为True                            |
| retry_files_enabled          | 关闭因playbook无法执行而产生的retry文件                 |
| deprecation_warnings = False | 关闭警告信息                                            |

## 范例
```
[defaults]
inventory = ./hosts
host_key_checking = False
log_path= ./ansible.log
deprecation_warnings = False
retry_files_enabled = False
```

# 设置inventory
inventory即执行任务的主机资源列表，默认指的是/etc/ansible/hosts文件
## 引用inventory
* 默认使用/etc/ansible/hosts或ansible.cfg中定义的hosts
* [动态的获取主机资源][dynamic_inventory]，比如
    - 使用脚本，动态获取主机资源
    - 通过插件，从云上获取主机资源
* 主机资源文件可以是多种格式的，默认是ini，也可以是yaml
    - 查看ansible支持的主机资源类型：ansible-doc -t inventory --list
* 多种主机资源聚合时
    - 可以在命令行中多次使用-i参数指定
    - 也可以建立一个目录，在目录下定义多个主机资源文件

## 定义inventory
* 可以在inventory中定义主机变量和组变量【一般只设置[连接类型的内置变量][inventory-parameters]，常用参数如下：】
    - ansible_user：ansible连接远程主机使用的ssh用户
    - ansible_host：远程连接主机
    - ansible_port：远程连接使用的端口
    - ansible_password：远程连接面
    - ansible_connecttion：定义hosts的连接方式【值为local时为执行本地操作】
* 一个主机可以被包含在多个组中，一个组也可以包含另一个组
* 当主机名或ip中包含连续的字母或数字时，可以使用正则形式，比如
    - www[a-f].example.com
    - 192.168.0.1[0-9]
* 默认的组是all【包含所有主机】和ungrouped【包含未分组的主机】
* 当控制端与远程主机没有使用标准的22端口通信时，
    - 如果使用的是paramiko进行连接【比如centos6/RHEL6】,则不会读取ssh配置文件中的端口信息
    - 当使用openssh进行连接时，则会使用ssh配置文件中的端口信息
    - 所以当使用非标准端口进行通信时，应该在inventory中明确指明使用的端口，例如：
        + 后缀形式：`badwolf.example.com:5309`
        + 变量形式：`jumper ansible_port=5555 ansible_host=192.0.2.50`

```
crm ansible_user='root' ansible_ssh_pass='123456'     定义主机和主机变量
[docker]      定义docker组
192.168.99.10[2:9]
[docker:vars]      定义docker组变量
ansible_ssh_pass="123"
[newserver:children]      newserver组包含webservers组
webservers
```

# 远程连接设置
* ansible默认使用原生openssh用于远程连接，并将开启ControlPersist 功能  
* 由于RHEL6和centos6的openssh版本过老无法使用ControlPersist 功能，在这些系统上将会使用paramiko来替代openssh用于远程连接  
* 有时会出现个别设备不支持sftp，这时候就需要使用scp来替代【比如使用跳板机】  

## 密码方式登录
在inventory文件中设置ansible_user和ansible_password参数
## 秘钥方式登录
>先使用密码方式，建立秘钥后取消密码选项

* 产生秘钥：ssh-keygen -t rsa -P "" -f ./bastion
* 传送公钥：ansible ops -m copy -a "src=keyfiles/bastion.pub dest=~/.ssh/authorized_keys mode=0600"

# 跳板机设置
## 实现目标
ansible可以通过跳板机管理远程内网服务器
## 实现原理
* 通过ssh的代理转发功能实现本地使用远程主机内网ip登录远程服务器，
* 本案例中使用证书登录，即ansible主机与跳板机(bastion)、跳板机与目标服务器(internal)之间均使用证书登录

## 操作步骤
### 产生秘钥
* ssh-keygen -t rsa -P "" -f ./bastion
* ssh-keygen -t rsa -P "" -f ./internal

### 传送公钥
>公钥放置在某用户主目录下，则以后必须使用相应的用户登录，此例中
>ansible主机使用muker登录跳板机，跳板机使用muker登录目标主机

* ansible ops -m copy -a "src=keyfiles/bastion.pub dest=~/.ssh/authorized_keys mode=0600"
* ansible 1.1.1.1【下例中172.16.0.205主机的公网ip】 -m copy -a "src=keyfiles/internal.pub dest=~/.ssh/authorized_keys mode=0600"

### 设置sshd(可选)
>设置ssh服务只允许证书登录

```
# /etc/ssh/sshd_conf
PermitRootLogin no
PasswordAuthentication no
```
### ssh配置
* ssh发起连接时，默认使用~/.ssh/config配置，可使用-F参数强制使用指定配置文件
* 本地私钥权限必须为0600
* config文件配置

```
Host ops
  User muker #连接跳板机使用的用户名
  HostName 47.99.78.151 #跳板机ip
  ProxyCommand none
  BatchMode yes #跳板机模式
  IdentityFile ~/keyfiles/bastion #本地连跳板机时使用的私钥
  StrictHostKeyChecking no #首次登录时禁止秘钥检查
Host 172.16.0.*  # 目标主机网络
  ServerAliveInterval 60 
  TCPKeepAlive        yes
  ProxyCommand ssh -qaY -i ~/keyfiles/bastion muker@ops 'nc -w 14400 %h %p' # ssh代理转发
  IdentityFile    ~/keyfiles/internal #跳板机使用私钥internal连接目标主机
  StrictHostKeyChecking no
```
### ssh登录测试
* 登录跳板机： ssh -F keyfiles/config ops
* 登录目标内网主机：ssh -F keyfiles/config muker@172.16.0.205

### ansible配置
* ansible.cfg
    ```
    [ssh_connection]
    ssh_args = -C -o ControlMaster=auto -F keyfiles/config
    scp_if_ssh = True #文件复制操作强制使用scp模式
    ```

* hosts
    ```
    ops ansible_user='muker'
    172.16.0.205 ansible_user='muker'
    ```

### ansible测试
>ping测试

* ansible ops -m ping
* ansible 172.16.0.205 -m ping

[dynamic_inventory]: https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html
[inventory-parameters]: https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters
