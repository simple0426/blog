---
title: ansible之常用组件
tags:
  - lookup
  - loop
  - condition
categories: ansible
date: 2019-05-24 17:07:55
---
# lookup插件
* lookup插件允许访问外部数据源
* 解析过程在控制端完成，和模板系统类似
* 官方源：<https://docs.ansible.com/ansible/latest/plugins/lookup.html>

## file
从文件中解析内容  
```
vars:
- contents: "{{ lookup('file', '/etc/hosts')}}"
tasks:
- name: display network content
  debug: msg="network content is {% for i in contents.split('\n') %}{{ i }}{% endfor %}"
```
## template
主要用于解析模板中的facts信息【被控主机】  
```
lookup.j2：
worker_processes {{ ansible_processor_cores }}
IPaddress {{ ansible_eth0.ipv4.address }}
test.yml：
vars:
- temp: "{{ lookup('template', 'lookup.j2')}}"
tasks:
- name: display lookup content
  debug: msg="network content is {% for i in temp.split('\n') %}{{ i }}{% endfor %}"
```
## pipe
解析命令行输出  
```
vars:
- cmd: "{{ lookup('pipe', 'date +%s')}}"
tasks:
- name: display lookup content
  debug: msg="{{ cmd }}"
```
## redis
从redis中解析key值  
```
vars:
- key: "{{ lookup('redis', 'he', host='127.0.0.1', port=6379)}}"
tasks:
- name: display lookup content
  debug: msg="{{ key }}"
```
## csvfile
从csv文件中解析内容  
```
# java1是第0列的值，同时作为key，去取第4列的值【从0开始索引】
# 此时逗号作为分隔符【默认为tab（TAB或t）】
- name: get value from csv
  debug: msg="java1 password is {{ lookup('csvfile', 'java1 file=b.csv delimiter=, col=4')}}"
```
# 循环
>loop可以部分替代with_xxx功能  

## 普通列表
```
vars:
  - numbers: ['one', 'two', 'three']

- debug: msg='{{ item }}'
  # with_flattened: '{{ numbers }}'
  loop: '{{ numbers|flatten }}'

- debug: msg="{{ item }}"
  # with_items: 
  loop:
    - 'one'
    - 'two'
    - 'three'
```
## hash列表
```
- debug: msg="{{ item.key }} is {{ item.value }}"
  # with_items:
  loop:
    - {'key': 'name', 'value': 1}
    - {'key': 'nam2', 'value': 2}
```
## 字典
```
vars:
  - user:
      shencan:
        name: shencan
        shell: bash
      ruifengyun:
        name: ruifengyun
        shell: zsh

- debug: msg='{{ item.key }} value={{ item.value.name }}
              shell={{ item.value.shell}}'
  loop: "{{ user|dict2items }}"
  with_dict: "{{ user }}"
```
## 双循环
```
- shell: "echo {{ item[0] }}*{{ item[1] }}|bc"
  with_nested:
    - [2, 3]
    - [3, 5, 7]
```
## 匹配文件
```
- shell: echo 12 > {{ item }}
  with_fileglob:
    - './*.1'
    - './*.2'
```

# 条件判断
## when
* 一次性判断  
* when中直接使用变量名，不需要引号
* 多个条件为and关系时可以使用列表形式

```
- name: touch file when os is ubuntu
  command: touch /tmp/ceshi
  when:
    - ansible_facts["os_family"] == "Debian"
    - ansible_memory_mb.real.total > 900
```
### 执行结果判断
* successed
* failed
* changed
* skipped
* undefined【变量未定义】

```
- name: test result
  git:
    repo: https://github.com/simple0426/gitbook.git
    dest: /home/python/gitbook
    version: master
  run_once: true
  register: result
- name: debug change
  debug: msg="change"
  when: result is changed
- name: debug undefined
  debug: msg="key12 is undefined"
  when: key12 is undefined
```
## until
多次尝试性判断  

```
# 每隔15s执行一次，共尝试3次；条件依然不满足时，task失败
- name: ensure time is 201905241750
  shell: date +%Y%m%d%H%M
  register: date
  until: date.stdout == '201905241750'
  retries: 3
  delay: 15
```
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

[var-merge]: https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#how-we-merge
[var-precedence]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable

