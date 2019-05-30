---
title: ansible之命令行
tags:
  - ansible
  - ansible-config
  - ansible-console
  - ansible-doc
  - ansible-inventory
  - ansible-playbook
  - ansible-galaxy
  - ansible-vault
  - ansible-pull
categories: ansible
date: 2019-05-29 15:36:18
---

# ansible
在一个或一组主机下运行单个任务
## 示例
ansible 10.150.20.209 -u db -m apt -a "pkg=dos2unix state=latest" -b --become-method sudo -k -K
## 常用参数
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

## 范例1：异步模式
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
## 范例2：新建用户
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

# ansible-inventory
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
* 是用于加密结构化数据[json或yaml格式]文件的命令
* 可以通过include_vars或vars_files加载group_vars或host_vars下的资源变量文件
* 也可以是ansible-playbook命令下使用-e参数加载的变量文件
* 语法：ansible-vault [create|decrypt|edit|encrypt|rekey|view] [options] [vaultfile.yml]

## 命令选项
- create 创建加密文件
- edit 编辑加密文件
- encrypt 加密文件
- decrypt 解密文件
- view 查看加密文件内容
- rekey 变更加密密码

## 命令参数
* --ask-vault-pass：【默认选项】要求输入加密密码
* --vault-password-file：提供密码文件

## 使用方式
>命令ansible或ansible-playbook

- --ask-vault-pass 交互式出入加密密码
- --vault-password-file=xx 提供加密密码文件
- 范例：
    - ansible 127.0.0.1 -e "@vars.yml" -m debug  -a "msg={{ key3 }}" --ask-vault-pass
    - ansible-playbook test.yml --vault-password-file=password.txt

# ansible-playbook
运行playbook的工具

* --flush-cache：清除每一台主机的facts缓存信息
* --force-handlers：即使任务失败也执行handlers
* --syntax-check：对playbook进行语法检查
* --start-at-task：从指定任务处继续执行playbook
* -t/--tags xxx：执行指定标签任务
* --step：每执行一个任务都要进行确认
* --list-hosts：查看所有主机
* --list-tags：查看所有标签
* --list-tasks：查看所有任务
* --skip-tags：跳过标签执行其他任务

# ansible-galaxy
管理共享仓库中的ansible角色，默认的共享仓库是<https://galaxy.ansible.com.>

* info：查询已经安装的或在共享仓库中的角色详细信息
* search：从共享仓库搜索角色【全名搜索】
* list：查看本地已经安装的角色【全名搜索】
* remove：移除本地角色
* install：安装一个角色
    - -f 强制覆盖已存在的角色
    - --force-with-deps：强制覆盖已存在的角色和依赖信息

# ansible-pull
从版本库中拉取playbook并在本地执行

## 命令参数
ansible-pull -o -C master -U https://github.com/simple0426/ansible_test.git -i hosts --purge

* -o 只有代码库有变更时拉取代码并执行
* --purge 执行ansible完成后删除代码
  - 与-o参数有冲突
  - 默认会将playbook项目拉取至~/.ansible/pull/《hostname》|local目录下
* -C 使用指定分支代码
* -i 强制使用项目中的hosts
* -U 指定playbook的仓库地址【必选项】

## 仓库目录
ansible_test/
├── hosts
├── local.yml
├── ops.yml
└── simple.yml

### hosts
```
[localhost]
127.0.0.1
```

* hosts中只能定义本机，即127.0.0.1
* 当系统没hosts或未设置时默认使用项目中的hosts
* 当系统有hosts设置时，只会读取127.0.0.1设置，此时需要在命令行使用【-i hosts】参数强制读取项目中的hosts

### playbook
```
---
- hosts: localhost
  gather_facts: no
  tasks:
    - name: touch file
      command: touch /tmp/ceshi3
```

1. playbook中的hosts只能为本机，即127.0.0.1或其别名
2. 程序会通过python接口读取系统的主机名或其缩略形式
3. ansible-pull会在项目中查找与主机名同名的playbook执行【如ops.yml、simple.yml】
4. 如果未找到主机名相关的playbook，则会在项目中找local.yml这个playbook后执行
5. 最后，如果找不到local.yml则报错退出

## 使用
* 可以在项目中的local.yml中定义每个主机都要执行的操作
* 在项目的《hostname》.yml中定义个别主机需要执行的操作
* 也可以使用不同分支控制不同主机执行不同操作
* 可以使用crontab来定义每个主机定期执行的playbook

## 番外：FQDN
* FQDN=主机名+域名
* 设置主机名
    - 命令行：hostname xxxx
    -  文件/etc/hostname：xxx
    -  文件/etc/hosts：127.0.0.1 xxx
* 设置FQDN【以便于python等程序通过接口获取】
    1. 删除hosts中默认的"127.0.0.1   localhost localhost.localdomain"等选项
    2. hosts中添加"127.0.0.1       xxx localhost"
* 如果hosts的127.0.0.1行内存在诸如"localhost.localdomain"以逗号分隔的选项，则FQDN依然为"localhost.localdomain"
