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

### 变量目录中定义
>与hosts文件同级目录

* group_vars目录下定义组变量
    - 以组名建文件
    - all文件表示所有组的公共变量
* host_vars目录下定义主机变量
    - 以主机名或ip建文件

## 命令行下的变量
## 注册变量
## 变量优先级
