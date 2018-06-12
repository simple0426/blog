---
title: django开发之templates学习
tags: ['templates']
categories: ['django']
---
# 介绍
## 约定
为避免模板二次渲染，特在变量{%raw%}{{ }}{%endraw%}和控制结构{%raw%}{% %}{%endraw%}中添加短横线屏蔽，替换结果如下：

* {%raw%}{{ }}{%endraw%} 替换为`{-{ }-}`
* {%raw%}{% %}{%endraw%}替换为`{-% %-}`

## 介绍
含有模板语法【例如jinja2】的html文件即是模板文件
## 语法
* 变量：`{-{ }-}`
* 控制结构：`{-% %-}`
* 注释：`{# #}`

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
## 属性与方法使用
>在django模板系统中处理复杂数据结构使用（.）字符

* 根据对象的key或索引获取对象的value
* 调用对象的方法【方法不能加括号，不能加参数】
* 获取对象的属性

## 过滤器使用
>变量可以通过过滤器进行修改操作

* 设置默认值：`{-{ test|default:'空值' }-}`

## 特殊变量forloop
>forloop变量只能在循环中得到，当模板解析器到达endfor时forloop就消失了

*  forloop.counter：表示循环的次数，从1开始计数
*  forloop.counter0：表示循环的次数，从0开始计数
*  forloop.revcounter：反向循环计数，末尾是1
*  forloop.revcounter0：反向循环计数，末尾是0
*  forloop.first：如果是循环的第一个元素
*  forloop.last：如果是循环的最后一个元素

# 控制结构
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
* verbatim禁止渲染：`{-%  verbatim %-}-}}@ ceshi }-}{-% endverbatim %-}`

# 自定义标签和过滤器
## 创建
- 在已经注册过的app下新建templatetags包
- 在templatetags包下编写python文件

## 使用
- filter对变量进行处理，可以if和for中使用
- simple_tag为自定义流程标签，不能再if和for中使用

## 范例
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
# 模板继承
