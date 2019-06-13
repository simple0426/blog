---
title: yaml语法学习
date: 2018-02-27 11:23:11
tags: ['yaml']
categories: ['skill']
---
# 语法
* 大小写敏感
* 使用缩进表示层级关系
* 缩进时只允许使用空格，不允许使用tab
* 缩进时的空格数量不重要，重要的是同一层级的元素左侧对齐
* #表示注释

# 字典
## 行内式
`dict: {a: 'test1', b: 'test2'}`
## 复合式
```yaml
dict:
 a: 'test3'
 b: 'test4'
```
# 列表
* 语法：以连词线开头的行
* 范例：

```yaml
list:
 - ceshi1
 - ceshi2
```
# 纯量
* 数字
* 字符串
    - 多行内容的字符串表示【使用`|`或`>`】

```yaml
str: |
 line 1
 line 2
 line 3
str: >
 line 1
 line 2
 line 3
```

* 布尔值

# 引用
* `&`定义锚点，`<<`表示合并到当前数据，`*`引用锚点
* 范例1-字典

```yaml
# 数据结构
dict1: &defaults {a: 'test1', b: 'test2'}
dict2:
 c: 'test3'
 d: 'test4'
 <<: *defaults

# 输出
print(dict1, dict2)
{'a': 'test1', 'b': 'test2'} 
{'c': 'test3', 'a': 'test1', 'd': 'test4', 'b': 'test2'}
```

* 范例2-列表

```yaml
ceshi:
 - &var a
 - b
 - c
 - d
 - *var
```
