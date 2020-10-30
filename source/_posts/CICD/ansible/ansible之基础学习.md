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
ansible是用于远程系统管理、软件部署的工具，类似工具有fabric、puppet
## 软件特性
- 基于当前的架构理解，使用简单的语言描述操作过程（playbook编写）
- 操作的可重用性及操作过程的可追溯性（playbook的可读性、roles的可重用性）
- 任何操作都是基于状态的描述（模块的幂等特性），只需保证目标对象最终要达到的状态，而不管目标对象的当前状态
- 基于openssh进行安全通信，无需额外的端口，无需客户端
- 管理各种基础设置，比如裸机(cobbler)、操作系统、虚拟化(vagrant)、容器(docker)、云、网络、存储等
- 内置功能丰富的模块，同时拥有强大的社区（galaxy）
- 管理软件迭代过程中使用的多个阶段环境（比如dev/test/stage/prod，通过多inventory实现）
- 可以实现服务的滚动更新（percentage）和不停机更新

## 实现操作
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
| stdout_callback              | 定义屏幕输出格式为yaml(json格式输出遇到换行时不直观)                                                        |

## 范例
```
[defaults]
inventory      = ./hosts
remote_port    = 22
host_key_checking = False
stdout_callback = yaml
callback_whitelist = timer
command_warnings = False
fact_caching = jsonfile
fact_caching_connection=/tmp
fact_caching_timeout = 86400
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

# 常用模块
* ping：测试主机连通性
* setup：用于获取操作系统信息
    - ansible 127.0.0.1 -m setup -a "filter=ansible_os*"
* command/shell：执行命令【command为命令行下默认模块】
  - creates：当文件存在时，module所在task不执行【可用于实现幂等性】
* template：模板系统，可以复制包含变量名的文件
    - validate：检查由实参渲染的模板文件语法是否正常，如nginx配置文件、sudo文件
    ```
    - name: write the nginx config file
      template: src=nginx2.conf dest=/etc/nginx/nginx.conf validate='nginx -t -c %s'
      notify:
      - restart nginx
    ```

* file：创建、删除文件或目录
    - state：directory、absent
* yum/apt：软件包管理
* service：服务管理
* user：用户管理
* git：git操作
* copy：复制文件和目录
    - 缺点：当复制的目录有多级或目录内的文件数据过多时，传输效率异常低下
    - 优点：可以备份（backup），可以检测配置文件的有效性（validate）
* archive：打包文件和目录
    - 缺点：不会复制空目录或空文件【比如python包文件`__init__.py`即可以是空文件】
* synchronize：使用rsync模块同步文件和目录
    - 优点：传输效率高
    - 缺点：
        + 必须使用ansible.cfg中的ssh配置选项
        + 不能备份、不能检测配置文件有效性
        + 不能解析hosts文件中的变量
* haproxy：控制haproxy服务
    + haproxy版本：1.5【增加后端服务器drain状态（软下线）】
    + haproxy依赖：安装socat，并在haproxy.cnf中配置：stats socket /var/run/haproxy.sock mode 600 level admin
    ```
    # 此配置主要将后端的某台服务器软下线【不接受新连接，已经建立的连接正常处理并关闭】
    - name: disable {{ project }} in haproxy
      haproxy: state=drain backend='{{ project }}' host='{{ inventory_hostname }}' socket=/var/run/haproxy.sock 
    # 上线某个后端的某台主机并等待，保障此后端主机可用
    - name: enable {{ project }} in haproxy
      haproxy: state=enabled backend='{{ project }}' host='{{ inventory_hostname }}' socket=/var/run/haproxy.sock wait=yes
      delegate_to: '{{ haproxy }}'
      become: yes
    ```
* fetch：从远程主机拉取文件到本地
  - dest：保存为本地文件或保存在本地目录【必须以斜线结尾】
  - flat：设置为yes时，和copy一样的行为；当为no时，则保存在本地的dest/<remote_hostname>/<absolute_path>下
  - src：远程主机上的文件
  - validate_checksum：是否在传输完成后进行校验
  - 范例：ansible test1 -m fetch -a "src=~/vendor/redis-4.0.14.tar.gz dest=files/ flat=yes"
* blockinfile：插入、更新、删除多行文本内容
  - group/owner/mode：属主属组权限等
  - backup：备份文件
  - block：在标记处将要插入的文本；如果选项缺失或是空字符串，则和state=present一样都是删除内容
  - path：文件路径
  - create：文件不存在则新创建
  - validate：文件语法检查
  - state：present为添加或更新，absent为删除
  - marker：替换内容的前后注释信息，默认："# {mark} ANSIBLE MANAGED BLOCK"
  - insertafter：在指定标记后插入内容
    + ‘EOF’表示文件末尾
    + regex表示一般正则表达式，如果匹配不到则使用EOF
  - insertbefore：在指定标记钱插入内容
    + 'BOF'表示文件开始
    + regex表示一般正则表达式，如果匹配不到则使用EOF

```
# 修改mavne仓库为阿里云
- name: add aliyun repo
  blockinfile:
    path: /usr/local/apache-maven-3.6.0/conf/settings.xml
    marker: "<!-- {mark} ANSIBLE MANAGED BLOCK -->"
    backup: yes
    insertafter: "<mirrors>"
    block: |
      <mirror>
         <id>nexus-aliyun</id>
         <mirrorOf>central</mirrorOf>
         <name>Nexus aliyun</name>
         <url>http://maven.aliyun.com/nexus/content/groups/public</url>
      </mirror>
```

[dynamic_inventory]: https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html
[inventory-parameters]: https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters
