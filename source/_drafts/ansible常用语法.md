---
title: ansible常用语法
tags:
    - filter
    - lookup
    - loop
    - vault
    - condition
    - 变量
categories:
    - ansible
---
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
# 变量
## 内置变量
* 官方说明：<https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#special-variables>
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
* 主机变量： 192.168.1.1 key=9 
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
>与hosts文件同级目录

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
## 变量优先级
* 官方源：<https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable>
* 简单表示【优先级从高到低】
  1. 命令行自定义变量
  2. task 【tasks列表中定义的变量】
  3. role 【roles列表中定义的变量】
      - role是将单个play拆分成目录文件结构形式
  4. play 
      - play是以单个文件的形式实现特定功能，如安装启动nginx等
  5. playbook 
      - playbook是以多级目录文件结构形式实现复杂功能，
      - playbook是多个play的集合，如部署lnmp的playbook
  6. inventory 【定义资源的文件】
