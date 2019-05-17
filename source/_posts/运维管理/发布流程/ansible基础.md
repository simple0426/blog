---
title: ansible基础
tags:
  - ansible
  - ansible-console
  - ansible.cfg
categories:
  - ansible
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
## 配置文件ansible.cfg 
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
## 资源文件hosts
即inventory：主机列表【默认使用/etc/ansible/hosts或ansible.cf中定义的hosts】

* 硬编码主机列表【常用】
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

# 命令ansible
## 示例
ansible 10.150.20.209 -u db -m apt -a "pkg=dos2unix state=latest" -b --become-method sudo -k -K
## 常用参数
| 参数 |              含义              |
|------|--------------------------------|
| -u   | 连接远程主机使用的ssh用户名    |
| -m   | 使用的模块                     |
| -a   | 模块参数                       |
| -b   | 执行命令时提升权限【默认sudo】 |
| -k   | ssh连接密码                    |
| -K   | 提升权限使用的密码             |
## 异步模式
* -B seconds 设置任务异步执行，超时则失败【不设置则永不超时】
* -P 0 设置轮询异步任务的时间间隔【0为永不轮询】

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
# 命令ansible-console
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
# 典型应用
## 新建用户
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
