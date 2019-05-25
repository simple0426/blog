---
title: ansible之playbook使用
date: 2019-05-25 22:33:05
tags:
categories:
---

# playbook简介
playbook是以目录文件结构的形式组织ansible语法，用以实现复杂功能的

* module：实现功能的基本组成单元，类似程序代码
    - 它应该是幂等效果，即多次执行和一次执行具有同样的效果
* task：执行module，用以实现特定功能，类似运行代码后的进程
* role：是由task、handler、var等组成以实现特定功能
* play：是将一些主机(inventory)和一些role进行映射、绑定，以达到在这些主机上实现特定功能
* playbook：playbook是多个play的集合，以达到在多主机进行多功能的部署

# task语法
* name：task名称，以描述这个task的具体任务；可以在name中使用已经定义过的变量
* ignore_errors：在task中定义，忽略错误继续执行下个task【否则，遇到错误直接跳出整个playbook的执行】
* module：执行的模块；如果模块中的参数过长，可以在末尾留出空格并以缩进形式开启新行
* notify：在任务执行完成后执行的触发器；
    - 一个触发器仅被执行一次，即使在一个play中声明多次
    - 在一个play中，多个触发器共同在task列表执行完成后执行
    - 触发器的运行顺序根据handlers中的定义顺序执行，和notify的使用顺序无关
    - pre_tasks/tasks/post_tasks中的触发器在相应部分的末尾被触发
* import_tasks：包含其他task
* import_role/include_role：包含其他role
* run_once：仅执行一次

# play语法
* name：play名称
* hosts：将要执行任务的主机，以逗号分隔多个主机或多个组
* gather_facts：是否收集操作系统信息
* vars：定义的变量
* serial：执行任务时，同时有几台主机在执行
* tasks：执行任务列表
* pre_tasks/post_tasks：tasks之前或之后执行的任务列表
* roles：角色列表
* handlers：触发器列表；实际也是任务列表，和普通的任务没什么不同

# 公共语法
>play或task中都可以使用的语法

* environment：在task和play中设置环境变量
    -   environment: PATH: /home/{{ ansible_ssh_user }}/open_api:{{ ansible_env.PATH }}
* tags：在play、roles、task级别设置标签，以便只执行特定标签任务

    ```
    # play级别
    - name: deploy java
      hosts: 172.17.12.14
      tags: ['java']

    # role级别
      roles:
        - { role: interface_app, tags: ['interface_app']}

    # task级别
    - name: debug info
      debug: "msg={{ ansible_env.PATH }}"
      tags: ['path']
    ```

# roles目录
* roles以包含规定目录名的语法来构成
* 每个规定的目录名下必须包含main.yml文件

```
roles/                            角色主目录
   common/                 common角色目录
     files/                        文件目录
     templates/             模板目录
     tasks/                      任务列表目录
     handlers/              handler目录
     vars/                      变量目录[vars中定义的变量优先于defaults中的变量]
     defaults/              变量目录
     meta/                   meta中定义角色依赖关系
```

# playbook目录
## 通用设置
```
site.yml                      主playbook入口
webservers.yml       特殊任务playbook入口
hosts                           自定义inventory
group_vars/               组变量
   all
   dbservers
host_vars/                  主机变量
   host1
   host2
roles
```
## 多环境设置
* 在区分环境时【比如开发、测试、生产、预上线、生产】，只需将jinventory及相关变量放置到同级目录即可
* 执行范例：ansible-playbook -i environments/pre/hosts site-pre.yml -t interface_campaign

```
# 主目录结构
├── ansible.cfg
├── deploy-pre.yml
├── deploy-prod.yml
├── environments
│   ├── pre
│   └── prod
├── files
│   ├── conf
│   ├── keyfile
│   ├── pre
│   └── prod
├── readme.md
├── roles
│   ├── common
│   ├── project_java
│   ├── project_python
│   ├── python_web
├── service-pre.py
├── service-prod.py
├── site-pre.yml
├── site-prod.yml
└── tasks
    ├── ChPass_DenyRoot_AddKey.yml
    └── UseRoot_AddUser.yml

# environment目录结构
├── pre
│   ├── group_vars
│   │   └── all
│   ├── hosts
│   └── host_vars
│       ├── 172.17.134.53
│       └── 172.17.134.63
└── prod
    ├── group_vars
    │   └── all
    ├── hosts
    └── host_vars
        ├── 172.17.12.10
        └── 172.17.8.18
```

# ansible-playbook执行
* --syntax-check：对playbook进行语法检查
* --start-at-task：从指定任务处继续执行playbook
* --list-hosts：查看所有主机
* --list-tags：查看所有标签
* --list-tasks：查看所有任务
* -t xxx：执行指定标签任务

# ansible-galaxy
一个开源的角色库

* 官方地址：<https://galaxy.ansible.com/>
 