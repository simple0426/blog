---
title: ansible之常用组件
tags:
  - filter
  - lookup
  - loop
  - condition
  - 变量
categories: ansible
date: 2019-05-24 17:07:55
---

# 变量
## 内置变量
* [官方说明](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#special-variables)
* 常用

|        变量        |           含义          |
|--------------------|-------------------------|
| play_hosts         | 远端运行ansible的主机名 |
| group_names        | 当前主机所在组信息      |
| groups             | 所有组信息              |
| inventory_hostname | inventory中定义的主机名 |
| hostvars           | 主机变量                |
| inventory_dir      | iniventory所在目录      |

## inventory中的变量
* 主机变量： `crm ansible_user='root' ansible_ssh_pass='123456'`
* 组变量：

```
[nginx:vars]
key=9
```
## playbook中的变量
### playbook中定义
```
vars: #直接定义变量
  - key1: 8
  # 定义长字符变量
  - key5: |
      123000
      234111
      345222
vars_files: #使用变量文件
  - var.yml
vars_prompt: #交互式输入变量
  - name: "key3" #变量名
    prompt: "pls input key3 value:" #交互式信息
    default: 'None' #默认值
    private: yes #是否显示输入
tasks:
  - name: debug info
    debug: msg="key1 is {{ key1 }} key2 is {{ key2 }} key3 is {{ key3 }}"
```
### 变量目录中定义
* 可以包含在playbook根目录下，也可以包含在inventory目录下，当两者同时存在时，playbook下的覆盖inventory下的
* 变量目录包含的变量文件是yaml格式的，且文件名没有后缀或是以'.yml'、'.yaml'、'.json'为后缀
* ansible-playbook命令默认寻找当前playbook目录下的变量文件，但是其他ansible命令（比如ansible/ansible-console）则只查找inventory目录下的变量文件，除非使用--playbook-dir参数后才会查找playbook下的变量文件
* 多个地方定义的变量，优先级如下：host》childgroup》parentgroup》allgroup
* group_vars目录下定义组变量
    - 组名文件表示组变量
    - all文件表示所有组的公共变量
* host_vars目录下定义主机变量
    - 主机名或ip名文件表示主机变量

## 命令行下的变量
ansible-playbook -e "key1=7" -e "@var.yml" test.yml
## 注册变量
将某些输出信息保存为变量

```
- name: register variable
  shell: hostname
  register: info
- name: use register
  debug: msg="This is {{ info.stdout_lines[0] }}"
```
## 变量冲突
* [变量合并][var-merge]
* [变量优先级][var-precedence]
* 优先级简单表示【优先级从高到低】
  1. 命令行自定义变量
  2. task 【tasks列表中定义的变量】
  3. role 【roles列表中定义的变量】
  4. play 
  5. playbook 
  6. inventory

# filter
* 作用：用于将变量转换成期望的数据格式
* 官方参考：<https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html>

## 常用过滤器
|     名称     |     含义     |
|--------------|--------------|
| int          | 字符串转整形 |
| capitalize   | 首字母大写   |
| default      | 设置默认值   |
| random       | 取随机值     |
| min/max      | 取最大最小值 |
| replace      | 替换         |
| regu_replace | 正则替换     |
| join         | 字符串拼接   |

## 使用范例
```
---
- hosts: 127.0.0.1
  gather_facts: False
  vars:
    filename: /etc/profile
    list: [1, 2, 3, 4, 5]
    one: "1"
    str: "string"
  tasks:
    - name: use filter basename
      shell: echo {{ filename|basename }} >> /tmp/shell11
    - name: debug int capitalize filter
      debug: msg="The int value {{ one|int }} the lower value is {{ str|capitalize }}"
    - name: debug default value filter
      debug: msg="The variable is {{ ansible|default('ansible is not defined')}}"
    - name: debug list max and min filter
      debug: msg="The list max value is {{ list|max }} the min is {{ list|min }}"
    - name: debug join filter
      debug: msg="The join filter value is {{ list|join('+') }}"
    - name: debug random filter
    # 从1开始步长为10取随机值
      debug: msg="the list random value is {{ list|random }} generate a random value {{ 1000|random(1, 10)}}"
    - name: debug replace and regex_replace filter
      debug: msg="The replace value is {{ str|replace('t', 'T')}},\
                The regex_replace value is {{ str|regex_replace('.*str(.*)$', '\\1')}}"
```
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
* command：执行命令【command为命令行下默认模块】
* template：模板系统，可以复制包含变量名的文件
    - validate：检查由实参渲染的模板文件语法是否正常，如nginx配置文件、sudo文件
    ```
    - name: write the nginx config file
      template: src=nginx2.conf dest=/etc/nginx/nginx.conf validate='nginx -t -c %s'
      notify:
      - restart nginx
    ```

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

