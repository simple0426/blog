---
title: ansible之变量与模板
tags:
  - 变量
  - lookup
  - loop
  - condition
  - filter
categories:
  - ansible
date: 2019-06-20 18:08:10
---
# 变量
## 变量使用注意
* 变量名，应该以字母开头，并且变量名中不含空格、逗点、中横线
* 变量内容，如果只包含字母、数字、下划线、中横线等普通字符，可以直接书写，无需使用引号
* 使用jinja2模块系统进行变量引用
* 对于字典变量，可以使用方括号或点方式引用，但是点方式可能会和python字典的属性或方法冲突
* 变量作用域
    - 全局：由config、环境变量【environment】、命令行设置
    - play级别：每个play和包含的结构，变量条目【vars，vars_files，vars_prompt】，roles的defaults和vars
    - host级别：直接关联主机的变量，比如inventory，include_vars，facts信息，任务执行产生的注册变量

## 特殊处理
* 当变量内容包含特殊字符（如&#）时，需使用双引号包围变量内容；
* 当变量内容含有`{{`等字符时，则需要使用“!unsafe“标志避免语法错误：`un_safevar:  !unsafe '{{in12var'`
* 避免模板变量被渲染（无论变量是否存在，都不对变量进行渲染）：`{% raw %}{{ test1 }}{% endraw %}`

## 锚点和引用
>这是yaml语法特性

```
vars:
  test1: &defaults 'he' #定义锚点
  test2:
    - 'jing'
    - 'qi'
    - *defaults # 在列表中引用锚点
  test3: &pro
    name: 'yaning'
    age: 30
  test4:
    <<: *pro # 将test3的变量内容合并到test4中
    age: 31  # 重新定义age变量
```

## [内置变量][build-vars]
* 保留变量
    - hostvars：主机变量
        + 可以在当前主机访问其他主机变量
        + `hostvars['bigdata']['ansible_facts']['eth0']['ipv4']['address']`
    - groups：所有组信息
    - group_names：当前主机所在组信息
    - inventory_hostname：inventory中定义的主机名
    - playbook_dir：playbook根目录
    - inventory_dir：inventory所在目录
* facts变量：ansible_facts
* 连接类型变量
    - ansible_user：远程连接用户
    - ansible_ssh_pass：ssh连接密码

## inventory中定义的变量
### inventory文件
* 主机变量： `crm ansible_user='root' ansible_ssh_pass='123456'`
* 组变量：

```
[nginx:vars]
key=9
```
### inventory目录
>与inventory同级或在其下的group_vars/host_vars目录

* 可以包含在playbook根目录下，也可以包含在inventory目录下，当两者同时存在时，playbook下的覆盖inventory下的
* 变量目录包含的变量文件是yaml格式的，且文件名没有后缀或是以'.yml'、'.yaml'、'.json'为后缀
* ansible-playbook命令默认寻找当前playbook目录下的变量文件，但是其他ansible命令（比如ansible/ansible-console）则只查找inventory目录下的变量文件，除非使用--playbook-dir参数后才会查找playbook下的变量文件
* 多个地方定义的变量，优先级如下：host》childgroup》parentgroup》allgroup
* group_vars目录下定义组变量
    - 组名文件表示组变量
    - all文件表示所有组的公共变量
* host_vars目录下定义主机变量
    - 主机名或ip名文件表示主机变量

## playbook中定义的变量
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
## roles中的变量
* vars目录定义的变量不可被inventory级别定义的变量覆盖，但是play、命令行级别的可以覆盖
* defaults目录定义的变量可以被inventory级别定义的变量覆盖

## 命令行下的变量
* 命令行下使用key=value传递的变量都是字符串形式，如果要传递其他类型变量【如：布尔、整数、浮点型、列表等】则需要使用json格式
* 范例：ansible-playbook -e "key1=7 key2=foo" -e "@var.yml @var1.json" test.yml

## facts中的变量
facts是从远程操作系统收集的信息
### facts变量获取
* 可以在playbook中使用变量ansible_facts获取
    - debug: var=ansible_facts
* 也可以使用setup模块获取
* ansible执行过程中默认收集facts信息
    - ansible.cfg中配置项gathering设置默认行为
    - play中设置gather_facts改变默认行为
* 默认没有开启facts缓存【可以在ansible.cfg设置如下参数开启文件形式缓存】
    - fact_caching = jsonfile
    - fact_caching_connection = /tmp
    - fact_caching_timeout = 86400

### facts变量引用
* facts变量可以从缓存或开启收集功能后实时获取
* 可以在当前执行任务的主机中获取其他主机或组的facts变量信息，但是
    - 需要在同一个play中已经和其他主机或组进行过通信，获取过facts信息
    - 或者，在更高级别的play中有收集过其他主机或组的facts信息
    - 也可以，开启facts缓存，当本地缓存有其他主机或组的facts信息时，则需受上述规则限制

## 注册变量
* 执行一个任务，并将任务返回值保留为一个变量以用于后续任务，此时的变量即是注册变量
* 注册变量只在产生之后、play运行结束之前有效，并且它只保留在内存中【而不像facts可以自定义保存位置】

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
  - 连接类型变量优先级【特定类型的链接设置优先于通用设置】：ansible_ssh_user》ansible_user》remote_user
  - 主机与组的变量优先级：主机变量》组变量》父组变量
* 优先级简单表示【优先级从高到低】
  1. 命令行自定义变量
  2. task 【tasks列表中定义的变量】
  3. role 【roles列表中定义的变量】
  4. play 
  5. playbook 
  6. inventory

# 过滤器
* [jinja2内置过滤器][jinjia-filter]
* [ansible支持的过滤器][ansible-filter]

## 常用过滤器
|     名称     |          含义          |
|--------------|------------------------|
| int          | 字符串转整形           |
| capitalize   | 首字母大写             |
| default      | 设置默认值             |
| random       | 取随机值               |
| min/max      | 取最大最小值           |
| replace      | 替换                   |
| regu_replace | 正则替换               |
| join         | 字符串拼接             |
| json_query   | 从复杂json结构中获取值 |
| from_json    | 从json文件获取变量     |
| from_yaml    | 从yaml文件中获取变量   |

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
    domain_definition:
        domain:
            cluster:
                - name: "cluster1"
                - name: "cluster2"
            server:
                - name: "server11"
                  cluster: "cluster1"
                  port: "8080"
                - name: "server12"
                  cluster: "cluster1"
                  port: "8090"
  tasks:
    # 使用jmespath从复杂json结构中获取变量
    - debug:
        var: item
      loop: "{{ domain_definition|json_query('domain.cluster[*].name')}}"
    - name: use filter basename
      shell: echo {{ filename|basename }} >> /tmp/shell11
    - name: debug int capitalize filter
      debug: msg="The int value {{ one|int }} the lower value is {{ str|capitalize }}"
    - name: debug default value filter
      debug: msg="The variable is {{ ansible|default('ansible is not defined')}}"
    # 当default第二个值为true，则会在变量计算出来为空或false时使用默认值
    - debug:
        msg: "var is {{ lookup('env', 'USER1')|default('admin', true)}}"
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
    # 从json或yaml文件中读取变量
    - name: cat test.json
      shell: cat test.json
      register: result_json
    - name: debug json info
      debug:
        msg: "test2 var is {{ (result_json.stdout|from_json)['test2']}}"
    - name: cat test1.yml
      shell: cat test1.yml
      register: result_yml
    - name: debug yml info
      debug:
        msg: "test1 var is {{ (result_yml.stdout|from_yaml)['test1'] }}"    
```
# [lookup插件][ansible-lookup]
* lookup插件允许访问外部数据源
* 解析过程在控制端完成，和模板系统类似

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
## lookup与loop
* lookup一次解析的多个结果默认用逗号连接；loop循环默认用列表作为输入。
* 为了让lookup解析的结果可以使用loop循环输出，需要如下操作，以便将解析结果转为列表
  - 在lookup中添加wantlist=True，
  - 或使用query代替lookup

```
    - debug: 
#        msg: "{{ lookup('inventory_hostnames', 'all', wantlist=True)}}"
        msg: "{{ query('inventory_hostnames', 'all')}}"
#        msg: "{{ item }}"
#      loop: "{{ lookup('inventory_hostnames', 'all', wantlist=True) }}"
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
  #loop: "{{ user|dict2items }}"
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
* when中直接使用变量名，不需要使用双大括号
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

[jinjia-filter]: http://jinja.pocoo.org/docs/templates/#builtin-filters
[ansible-filter]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html
[ansible-lookup]: https://docs.ansible.com/ansible/latest/plugins/lookup.html
[build-vars]: https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#special-variables
[var-merge]: https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#how-we-merge
[var-precedence]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable
