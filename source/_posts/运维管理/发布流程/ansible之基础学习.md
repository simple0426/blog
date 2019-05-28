---
title: ansible之基础学习
tags:
  - ansible
  - ansible-console
  - ansible.cfg
  - ansible-vault
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

安装：<https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html>
# 配置
## ansible.cfg 
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
## hosts
即inventory：主机列表【默认使用/etc/ansible/hosts或ansible.cfg中定义的hosts】

* 静态主机列表【常用】
* 动态主机列表【脚本】
    - 官方使用参考：<https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html>
    - 早期版本定义脚本：<https://github.com/simple0426/sysadm/blob/master/ansible/hosts.py>

### 定义主机与主机变量
`crm ansible_user='root' ansible_ssh_pass='123456'`
### 定义组和组变量
```
[docker] 定义docker组
192.168.99.10[2:9]
[docker:vars] 定义docker组变量
ansible_ssh_pass=""
[newserver:children] newserver组包含webservers组
webservers
```
### 常用内置变量
官方定义：<https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters>

* ansible_user：ansible连接远程主机使用的ssh用户
* ansible_ssh_pass
* ansible_connecttion：定义hosts的连接方式【值为local时为执行本地操作】

# ansible
在一个或一组主机下运行单个任务
## 示例
ansible 10.150.20.209 -u db -m apt -a "pkg=dos2unix state=latest" -b --become-method sudo -k -K
## 常用参数
### 一般选项
* --ask-vault-pass：       提示输入加密面
* --become-method：    权限变更的方式【su或sudo】
* --become-user：        权限变更时使用的用户
* --list-hosts：               输出符合要求的主机列表
* --playbook-dir
* --private-key，--key-file
* --scp-extra-args：      scp选项参数
* --sftp-extra-args ：     sftp选项参数
* --ssh-common-args：指定传给scp/sftp/ssh的一般参数
* --ssh-extra-args：     只传给ssh的参数
* --syntax-check：       检测语法
* --vault-id
* --vault-password-file：加密密码的文本文件
* --version：               显示程序版本、配置文件位置、模块搜索路径、模块文件位置、可执行文件位置
* -B/--background ：   异步运行模式，运行超过X秒后失败
* -C/--check：            不实际执行任务，但是显示执行后可能发生的变化
* -D/--diff：                与-C搭配使用，显示当使用files和templates模块时，变化前后的区别
* -K/--ask-become-pass：权限变更时交互式输入密码
* -M/--module-path：  以冒号分隔的自定义模块路径【默认~/.ansible/plugins/modules:/usr/share/ansible/plugins/modules】
* -P/--poll：               和-B搭配使用，设置轮询时间间隔【默认15，0为永不轮询】
* -t/--timeout：           设置连接超时时间【默认10】
* -a/--args：              模块参数
* -b/--become：         设置将要进行权限变更
* -c/--connection
* -e/--extra-vars :       设置自定义变量【key=value形式或yaml、json文件（文件名称前添加@）】
* -f/--forks：             设置并行进程数【默认为5】
* -h/--help：              帮助信息
* -i/--inventory：       设置资源文件位置或以逗号分隔的主机列表
* -k/--ask-pass：       远程连接时交互式输入密码
* -l/--limit
* -m/--module-name：将要执行的模块名称
* -o/--one-line：         简化输出为一行
* -t/--tree：                将输出记录到此目录
* -u/--user：              远程连接用户
* -v/--verbose：         详细输出模式（-vvv输出更多，-vvvv将开启连接调试）

## 范例：异步模式
```
[hjq@localhost ~]$ ansible python -B 300 -P 0 -m yum -a "name=libselinux-python state=latest" 
python | CHANGED => {
    "ansible_facts": {
        "pkg_mgr": "yum"
    }, 
    "ansible_job_id": "759078551843.27281", 
    "changed": true, 
    "finished": 0, 
    "results_file": "/root/.ansible_async/759078551843.27281", 
    "started": 1
}
[hjq@localhost ~]$ ansible python -m async_status -a "jid=759078551843.27281"
```
# ansible-console
>交互式使用ansible
## 常用参数
* cd 切换主机或组
* list 显示当前的主机或组
* forks 设置临时并发数
* module XXX 使用模块
* help module 查看模块用法
* become yes/no 进入/离开sudo模式

## 范例
```
hjq@all (2)[f:5]$ cd ops
hjq@ops (1)[f:5]$ copy src=ansible.cfg dest=~/
```
# ansible-doc
插件文档工具  
语法：ansible-doc [-l|-F|-s] [options] [-t <plugin type> ] [plugin]  

* -l 查看插件列表及相关简单描述信息
* -F 查看插件列表及相关的源码位置【隐含-l参数】
* -s 查看插件的参数信息
  - ansible-doc -s -t lookup csvfile
* -t 设置插件类型，默认为模块，可选插件如下
  - cache
  - callback
  - connection
  - inventory
  - lookup
  - shell
  - module
  - strategy
  - vars

# ansible-config
配置查看工具

* ansible-config dump --only-changed：查看ansible.cfg中的自定义配置【非默认配置】
* ansible-config view：查看当前配置文件ansible.cfg

## ansible-inventory
资源查看工具  
语法：--list [--export] [--yaml]|--graph [--vars]|--host[--yaml]

* --list：显示所有主机信息
    - --export ：与--list搭配使用，优化显示信息
* --host：显示特定主机信息
* -y/--yaml：以yaml显示输出【默认json格式】
    - 与list和host搭配使用
* --graph：以图表形式显示所有主机信息
    - --vars：在图表中显示变量信息

# ansible-vault
是用于加密结构化数据[json或yaml]文件的命令

* 命令参数
    - create 创建加密文件
    - edit 编辑加密文件
    - encrypt 加密文件
    - decrypt 解密文件
    - view 查看加密文件内容
    - rekey 变更加密密码
* 使用方式【命令ansible或ansible-playbook】
    - --ask-vault-pass 交互式出入加密密码
    - --vault-password-file=xx 提供加密密码文件
* 使用范例
    - ansible 127.0.0.1 -e "@vars.yml" -m debug  -a "msg={{ key3 }}" --ask-vault-pass
    - ansible-playbook test.yml --vault-password-file=password.txt

# 应用范例-新建用户
* 产生随机密码
    - 使用openssl：` echo pass|openssl passwd -1 -stdin `
    - ubuntu下mkpasswd：` echo pass|mkpasswd --method=sha-512 --stdin `
* 新建用户
    - 密码必须hash：`ansible python -m user -e "pass=Muke#123" -a 'name=muker password="{\{ pass|password_hash("sha512") }\}"' -b`
* 用户sudo配置
    - `ansible python -m lineinfile -a "dest=/etc/sudoers state=present line='muker ALL=(ALL) NOPASSWD: ALL' validate='visudo -cf %s'"`
    - `ansible python -m lineinfile -a 'dest=/etc/sudoers state=present line="Defaults:muker !requiretty" validate="visudo -cf %s"'`
* 秘钥登录
    - `ansible python -m authorized_key -a "user=muker key='{\{ lookup('file', lookup('env', 'HOME') + '/.ssh/id_rsa.pub') }\}' state=present" -u muker -K`

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
