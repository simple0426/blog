---
title: web开发之html学习
tags:
  - html
categories:
  - web
date: 2018-06-03 22:08:16
---

# 范例
```
<html> html文档的开始和结束标签
<head> 文档头部
<title> 网页标题
</title> 
</head>
<body> 网页内容
</body>
</html>
```

# 标签格式
## 标签语法
* 语法：<标签名 属性名="属性值" 属性名="属性值"></标签名>
* 范例：`<a href=""></a>`

## 标签分类
* 闭合标签：有开始和结束的标志，如：`<title>test</title>`
* 自闭和标签：只有开始的标志，如：`<meta charset="utf-8">|<img src="">`
* 块级别标签：显示效果为独占一行，如`<h1></h1><div></div><p></p>`
* 内联标签：显示效果取决于内容长度，如`<label></label><span></span><img src="">`

## 嵌套原则
* 块级别标签可以嵌套快级别标签和内联标签
* 内联标签只能嵌套内联标签

# head标签
```
<head>
    <!--搜索引擎搜索网页使用的关键词-->
    <meta name="keywords" content="it测试">
    <!--搜索引擎对网页内容的简单介绍-->
    <meta name="description" content="it测试">
    <!--网页解码方式-->
    <meta charset="UTF-8">
    <!--网页标题前的图标-->
    <link rel="icon" href="heart.png">
    <!--网页使用的css文件-->
    <link rel="stylesheet" type="text/css" href="">
    <!--网页使用的js文件-->
    <script type="text/javascript"></script>
    <!--网页标题-->
    <title>我的未来</title>
    <!--直接在html中定义css样式-->
    <style></style>
</head>
```

# body标签
## 一般标签
```
<h1>一级标题</h1>
<h2>二级标题</h2>
<p>段落1</p>
<p>段落2</p>
<br>换行
<hr>分割线
<b>加粗</b>
<strong>加粗</strong>
<i>斜体</i>
```
## CSS类标签
```
<div style="">自定义块级别标签</div>
<span>自定义内联标签</span>
<label>自定义内联标签</label>
```
## 资源标签
* 图片引用：`<img src="http://blog.unforget.cn/2018/04/18/TCP%E8%BF%9E%E6%8E%A5/tcp%E5%BB%BA%E7%AB%8B%E8%BF%9E%E6%8E%A5.png"height="200px" width="110px" alt="无资源显示" title="tcp连接">`
* 超链接：`<a href="http://www.baidu.com" target="_blank">点击地址</a>`
    - target="_blank" 跳转时新打开一个窗口

## 列表标签
### 无序列表
```
<ul>
    <li>111</li>
    <li>222</li>
    <li>333</li>
</ul>
```
### 有序列表
```
<ol>
    <li>aaa</li>
    <li>bbb</li>
    <li>ccc</li>
</ol>
```
### 自定义列表
```
<dl>
    <dt>列表名称</dt>
    <dd>列表项1</dd>
    <dd>列表项2</dd>
    <dd>列表项3</dd>
</dl>
```
## 表格标签
```
<!--cellpading内容到内边框的距离-->
<!--cellspacing内边框到外边框的距离-->
<!--border有边框-->
<table border="1" cellpadding="5px" cellspacing="10px">
    <tr>
        <!--th为头部信息-->
        <th>id</th>
        <th>name</th>
        <th>age</th>
    </tr>
    <tr>
        <!--td为数据信息，colspan横跨多少列-->
        <td colspan="2">1</td>
        <td>27</td>
    </tr>
    <tr>
        <!--rowspan横跨多少行-->
        <td rowspan="2">2</td>
        <td>jing</td>
        <td>25</td>
    </tr>
    <tr>
        <td>qi</td>
        <td>23</td>
    </tr>
</table>
```
## 表单标签
* action：处理请求的地址
* method：请求使用的方法
* 范例：`<form action="http://www.baidu.com" method="get"></form>`

### text
文本输入框

```
<p>
    <!--label的for和下边的id使用同一值，则鼠标点击提示信息即可直接输入-->
    <label for="user">姓名</label>
    <!--name为向后端传递的变量名，id则确定标签的唯一性-->
    <input type="text" name="username" id="user">
</p>
```
### password
密码输入框

```
<p>
    <label for="pass">密码</label>
    <input type="password" name="pwd" id="pass">
</p>
```
### checkbox
复选框【多选】

```
<p>
    爱好：
    <input type="checkbox" name="hobby" value="football">足球
    <input type="checkbox" name="hobby" value="basketball">篮球
</p>
```
### radio
复选框【单选】

```
<p>
    性别：
    <input type="radio" name="sex" value="1" checked>男
    <input type="radio" name="sex" value="2">女
</p>
```
### submit
表单提交

```
<input type="submit" value="表单提交">
```

### button
触发动作的按钮

```
<input type="button" value="按钮">
```

### reset
恢复默认值

```
<input type="reset" value="恢复默认">
```
### file
上传文件：form表单需要加上属性enctype="multipart/form-data" method="post"

```
<form action="http://www.baidu.com" method="post" enctype="multipart/form-data">
<p>
    <input type="file">
</p>
</form>
```

### select
下拉选择

* size：下拉框一次显示的选项个数
* multiple：允许多选
* selected：默认选项

```
<select name="province" size="2" multiple>
    <option value="beijing">北京</option>
    <option value="henan" selected>河南</option>
    <option value="hubei">湖北</option>
    <option value="shanxi">山西</option>
</select>
```
### textarea
多行文本框

```
<p>
    个人简介
    <textarea name="" id="" cols="30" rows="10"></textarea>
</p>
```

# 参考
* [html特殊标签](http://tool.chinaz.com/Tools/htmlchar.aspx)
* [html学习参考](http://www.cnblogs.com/yuanchenqi/articles/6835654.html)
