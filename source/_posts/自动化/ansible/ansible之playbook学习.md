---
title: ansible之playbook学习
date: 2019-05-25 22:33:05
tags:
  - playbook
  - include
  - 多环境
categories: ansible
---

# playbook简介
playbook是以目录文件结构的形式组织ansible语法，用以实现复杂功能，它的主要概念如下：

* module：实现功能的基本组成单元，类似程序代码
    - 它应该是幂等效果，即多次执行和一次执行具有同样的效果
* task：执行module，用以实现特定功能，类似运行代码后的进程
* role：是由task、handler、var等组成以实现特定功能，它还可能包含模块、插件等其他功能组件
* play：是将一些主机(inventory)和一些role或task进行映射、绑定，以达到在这些主机上实现特定功能
* playbook：playbook是多个play的集合，以达到在多主机进行多功能的部署
* block：一般用于task的逻辑分组以及在play中进行错误处理
* handlers：事件触发器，当远程系统发生变化时执行的触发器【即特定task】

## playbook功能
* 管理配置文件
* 远程部署服务
* 多层架构下的滚动更新
* 将操作委派给其他主机
* 在部署等操作过程中和监控系统、负载均衡系统交互

# block范例
## task逻辑分组
```
---
- hosts: 127.0.0.1
  debugger: on_skipped
  tasks:
    - name: display memory and cpu load
      block:
      - name: display memory
        shell: free -m
        register: mem
      - debug: msg="{{ mem.stdout }}"
      - name: display cpu load
        shell: uptime
        register: uptime
      - debug: msg="{{ uptime.stdout }}"
      when: ansible_facts['distribution'] == 'Ubuntu'
      become: yes
    - name: touch file
      command: touch /tmp/test
```
## play中错误处理
类似python中的try/expect/finally
```
---
- hosts: 127.0.0.1
  tasks:
    - name: Handler the error
      block:                          # 标记可能出现异常的区块
        - debug: msg='I execute normally'
        - name: I force a failure
          command: /bin/false
        - debug: msg="I never execute"
#      ignore_errors: yes     # ignore与rescue功能互斥
      rescue:                       # 出现异常时的处理
        - debug: msg='I caught an error'
        - name: 'force an error in rescue'
          command: /bin/false
        - debug: msg='I alse never executes'
      always:                       # 无论是否出现异常都执行的任务
        - name: other tasks
          command: uptime
```

# handlers语法
- 不支持在notify和handlers的name中使用变量
- 在handers中定义listen选项，既可以作为handler的名称被notify使用，也可以将多个handler分组以便notify使用
- 一个触发器仅被执行一次，即使在一个play中声明多次
- 在一个play中，多个触发器共同在task列表执行完成后执行
- 触发器的运行顺序根据handlers中的定义顺序执行，和notify的使用顺序无关
- pre_tasks/tasks/post_tasks中的触发器在相应部分的末尾被触发
- 不能触发位于include中触发器，但可以使用import中包含的触发器

# task语法
* name：task名称，以描述这个task的具体任务；可以在name中使用已经定义过的变量
* action：task定义旧形式
  - task定义新形式：【module：options】
  - task定义旧形式：【action：module options】
* ignore_errors：在task中定义，忽略错误继续执行下个task【否则，遇到错误直接跳出整个playbook的执行】
* module：执行的模块；如果模块中的参数过长，可以在末尾留出空格并以缩进形式开启新行
* notify：在任务执行完成后激活触发器的执行；
* run_once：仅执行一次
* delegate_to：在主机或主机组的task中定义其他主机要执行的任务
   ```
   本地操作实现
   shell：free -m
   run_once：true
   delegate_to:localhost
   ```

# play语法
* name：play名称
* hosts：将要执行任务的主机，以逗号分隔多个主机或多个组
* remote_user：远程连接用户
  - remote_user、ansible_user【ansible_ssh_user】、命令行下的-u参数 三者具有相同功能都是用于ssh远程连接时设置用户
  - remote_user主要在ansible.cfg、play中设置
  - ansible_user主要在inventory中设置
* order：设置执行任务的主机顺序
  - 默认inventory，即以inventory中定义的顺序执行任务
* gather_facts：是否收集操作系统信息
* vars：定义的变量
* tasks：执行任务列表
* pre_tasks/post_tasks：tasks之前或之后执行的任务列表
* roles：角色列表
* handlers：触发器列表；实际也是任务列表，和普通的任务没什么不同
* serial：指定在主机组或主机列表中执行任务时，同时有几台主机在执行任务，此功能主要用于灰度发布
    + 可以是数字或数字列表
    + 可以是百分比或百分比列表
    + 也可以是数字和百分比的混合
    ```
    - hosts: 172.17.134.58,172.17.134.63
      gather_facts: no
      vars: { haproxy: 172.17.134.53, end_sec: 10 }
      serial: 1
      tasks:
        - include: files/deploy/double_srv.yml
          tags: ['service_user']
          vars:  { project: service_user, project_port: 8050, start_sec: 30 }
    # 由于serial为1，同时只有一台主机在执行任务，循环2次执行完此任务；
    # 若serial为2，则同时会有2台主机执行任务，一次循环即可执行完
    ```

# 公共语法
play或task中都可以使用的语法

* environment：在task和play中设置环境变量
    - environment: PATH: /home/{{ ansible_ssh_user }}/open_api:{{ ansible_env.PATH }}
* tags：在play、roles、task级别设置标签，以便只执行特定标签任务
    - play级别
      ```
      - name: deploy java
        hosts: 172.17.12.14
        tags: ['java']
      ```
    - role级别
      ```
      roles:
          - { role: interface_app, tags: ['interface_app']}
      ```
    - task级别
      ```
      - name: debug info
        debug: "msg={{ ansible_env.PATH }}"
        tags: ['path']
      ```

* include：包含另一个play或task
    + 此功能已被拆分为include_xxx和import_xxx两类模块，未来可能被遗弃：<https://docs.ansible.com/ansible/latest/modules/include_module.html#include-module>
    + 包含play：与hosts同级的另一个play
    + 包含task：task列表
    ```
    ---
    - hosts: ops
      vars:
        - key: 8
      tasks:
        - name: debug info
          debug: msg="The {{ inventory_hostname }} Value is {{ key }}"
        - include: task.yml
    - include: play2.yml
    ```
    + 与when搭配使用
    ```
    - include: service_base.yml
      when: change|changed and project == 'service_base'
    ```

* 动态导入与静态导入
  * include_x：为动态导入，即在运行时遇到该任务点时才执行导入操作
      - include_role：加载并执行一个role
      - include_tasks：动态包含任务列表
  * import_x：为静态导入，即在ansible整体解析时执行导入操作
      - import_playbook：导入playbook
      - import_role：导入role到一个play中
      - import_tasks：导入task列表
  * 当使用tags或when时
    - 对于import，task选项(options)将会被复制进子task中
    - 对于include，task选项(options)只会影响自身，不会影响子task
  * include_x使用限制
    - 被包含的tags或tasks不会出现在命令行下--list-tags或--list-tasks中
    - 主play中的notify不会触发include包含的handler
    - 不能使用--start-at-task从include包含的task处执行
  * import_x使用限制
    - 循环时只能使用include而不是import
    - inventory中的变量不能用于目标文件名或角色名

* wait_for：在继续其他任务时，等待一个状态成立

```
# 指定的时间后【delay】，检测某台主机的某个端口是否还有tcp连接【drained】，最多等待300s，在300s内，  
# 如果为drained状态则继续执行其他任务，但是exclude_hosts中包含的主机【列表】与所操作的主机连接排除在外
- name: wait {{ project }} port drain
  wait_for: 
    host: '{{ inventory_hostname }}' 
    port: '{{ project_port }}' 
    state: drained 
    timeout: 300 
    delay: '{{ end_sec }}' 
    exclude_hosts: '{{ micro_srv }}'
# 在指定时间后检测端口是否开启
- name: wait {{ project }} port up
  wait_for: host='{{ inventory_hostname }}' port='{{ project_port }}' state=started delay='{{ start_sec }}'
```
# roles目录
* roles以包含规定目录名的语法来构成
* 每个规定的目录名下必须包含main.yml文件

```
roles/                          角色主目录
   common/                 common角色目录
     files/                      文件目录
     templates/              模板目录
     tasks/                    任务列表目录
     handlers/                handler目录
     vars/                      不可被inventory定义覆盖的变量目录
     defaults/                 可被inventory定义覆盖的变量目录
     meta/                      meta中定义角色依赖关系
```

# playbook目录
## 通用设置
```
site.yml                      主playbook入口
webservers.yml           特殊任务playbook入口
hosts                         自定义inventory
group_vars/                组变量
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
