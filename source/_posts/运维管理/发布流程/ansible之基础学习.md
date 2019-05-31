---
title: ansible之基础学习
tags:
  - ansible.cfg
  - ssh
categories: ansible
date: 2019-05-17 14:30:41
---

# 简介
[ansible是远程部署工具，类似工具有fabric、puppet，可实现如下功能

* 安装软件【yum、apt】
* 服务启停控制【service】
* 文件修改【lineinfile】
* 文件上传与下载【copy、fetch】
* 执行命令【command】

# 安装
* 参考：<https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html>

# ansible.cfg 
定义ansible执行时的参数配置
```
[defaults]
inventory = ./hosts
host_key_checking = False
log_path= ./ansible.log
deprecation_warnings = False
retry_files_enabled = False
```

* 配置文件默认加载顺序
    - 当前目录下的ansible.cfg
    - 用户主目录下的.ansible.cfg
    - 系统目录/etc/ansible/ansible.cfg
* 常用参数设置

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

# inventory
定义主机资源列表
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
* 可以在inventory中定义主机变量和组变量【一般只设置[连接类型的内置变量][inventory-parameters]，如下：】
    - ansible_user：ansible连接远程主机使用的ssh用户
    - ansible_host：远程连接主机
    - ansible_port：远程连接使用的端口
    - ansible_password：远程连接面
    - ansible_connecttion：定义hosts的连接方式【值为local时为执行本地操作】
* 一个主机可以被包含在多个组中，一个组也可以包含另一个组
* 当主机名或ip中有包含连续的字母或数字时，可以使用正则形式，比如
    - www[a-f].example.com
    - 192.168.0.1[0-9]
* 默认的组all包含所有主机，ungrouped包含未分组的主机
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

# 远程连接-跳板机设置
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
