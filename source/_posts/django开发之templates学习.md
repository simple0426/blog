---
title: django开发之templates学习
tags:
  - templates
categories:
  - django
date: 2018-06-13 11:43:54
---

__约定__
为避免模板二次渲染，特在变量{%raw%}{{ }}{%endraw%}和流程控制{%raw%}{% %}{%endraw%}中添加短横线屏蔽，替换结果如下：

* {%raw%}{{ }}{%endraw%} 替换为`{-{ }-}`
* {%raw%}{% %}{%endraw%}替换为`{-% %-}`

# 介绍
## 介绍
含有模板【例如jinja2模板】语法的html文件即是模板文件
## 主要语法
* [变量](#变量)：`{-{ }-}`
* [流程控制(标签)](#流程控制)：`{-% %-}`
* 注释：{% raw %}{# #}{% endraw %}

## 运算符
* == 等于
* != 不等于
* `> >=` 大于等于
* `< <=` 小于等于

## 使用范例
```
from django.template import Template, Context
#建立模板对象
t = Template("my name is {-{ name }-}") 
#建立参数对象
c = Context({"name": "jingqi"}) 
#使用Template的render方法将Context对象传入
t.render(c)
```
# 变量
## 属性与方法
>在django模板系统中处理复杂数据结构使用（.）字符

* 根据对象的key或索引获取对象的value
* 调用对象的方法【方法不能加括号，不能加参数】
* 获取对象的属性

## 过滤器
变量可以通过过滤器进行结果显示变更的操作，  
例如设置变量的默认值：`{-{ test|default:'空值' }-}`

## 特殊变量forloop
>forloop变量只能在循环中得到，当模板解析器到达endfor时forloop就消失了

*  forloop.counter：表示循环的次数，从1开始计数
*  forloop.counter0：表示循环的次数，从0开始计数
*  forloop.revcounter：反向循环计数，末尾是1
*  forloop.revcounter0：反向循环计数，末尾是0
*  forloop.first：如果是循环的第一个元素
*  forloop.last：如果是循环的最后一个元素

# 流程控制
## if判断
>if可以接受and、or、not测试同一变量，但不允许同一个标签内同时出现and和or，
>但可以多次出现and或or等同一逻辑

```
{-% if test > 2 %-}
    <p>大于2</p>
{-% elif test == 2 %-}
    <p>等于2</p>
{-% else %-}
    <p>小于2</p>
{-% endif %-}
```
## for循环
* 列表迭代

```
{-% for item in test %-}
    <p>{-{ item }-}</p>
{-% empty %-}
    <p>列表为空</p>
{-% endfor %-}
```

* 字典迭代

```
{-% for key,value in test.items %-}
<p>
    {-{ key }-}:{-{ value }-}
</p>
{-% endfor %-}
```
## 其他标签
* csrf_token：`{-% csrf_token %-}` 用于防治csrf跨站攻击验证；其实这里会生成一个input标签，和其他表单标签一起提交给后台
* url引用：`<form action="{-% url "bieming"%-}" >` 在html中引用urls.py中配置的url路径
* with别名设置：`{-% with total=fhjsaldfhjsdfhlasdfhljsdal %-} {-{ total }-} {-% endwith %-}`
* verbatim禁止渲染：`{-%  verbatim %-}{-{ ceshi }-}{-% endverbatim %-}`

# 自定义标签和过滤器
## 创建
- 在已经注册过的app下新建templatetags包
- 在templatetags包下编写python文件

## 使用注意
- filter对变量进行处理，可以在if和for中使用
- simple_tag为自定义流程标签，不能在if和for中使用

## 使用范例
###  python文件
```
from django import template
register = template.Library()
# name为给过滤器添加别名
@register.filter(name='percent')
def percent_decimal(value):
    value = float(str(value))
    value = round(value, 3)
    value = value * 100
    return str(value) + '%'
# 自定义标签
@register.simple_tag
def add_sum(v1, v2):
    return v1 + v2
```
### html文件
```
<!--在html文件顶部使用load加载标签文件名-->
{-% load MyTag %-}
Your input is {-{ value|percent }-}.
<p>output is {-% add_sum v1 v2 %-}</p>
```
# 静态文件设置
在编写模板时，引入一些现成的库文件【如bootstrap，jquery】，此时需要做一些额外的配置，相关设置如下：

1. 在settings中设置公有静态目录
2. 将库文件放入设置的静态目录中
3. 使用【load staticfiles】标签导入相关语法
4. 使用标签语法【static “url”】导入相关库文件

# 模板继承
## 基础模板设置
>每个block必须是唯一的

```
<!DOCTYPE html>
<html lang="en">
{-% load staticfiles %-}
<head>
    <meta charset="UTF-8">
    <title>{-% block title %-}title{-% endblock %-}</title>
    <link rel="stylesheet" href="{-% static "bootstrap-3.3.7-dist/css/bootstrap.css" %-}">
    <script src="{-% static "jquery-3.1.1.js" %-}"></script>
    <script src="{-% static "bootstrap-3.3.7-dist/js/bootstrap.js" %-}"></script>
</head>
<body>
{-% block content %-}你好{-% endblock %-}
{-% block footer %-}测试{-% endblock %-}
</body>
</html>
```
## 子模板设置
当需要继承基础模板时，使用extends关键词导入基础模板，且extends必须位于文件首行

```
{-% extends "base.html" %-}
{-% load Mytag %-}
{-% block content %-}
<p>output is{-% add_sum v1 v2 %-}</p>
<div class="container">
    <table class="table table-bordered">
        <tr>
            <th>1</th>
            <th>2</th>
            <th>3</th>
            <th>4</th>
        </tr>
        <tr>
            <td>a</td>
            <td>b</td>
            <td>c</td>
            <td>d</td>
        </tr>
    </table>
</div>
{-% endblock %-}
```
