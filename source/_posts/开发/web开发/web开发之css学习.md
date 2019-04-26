---
title: web开发之css学习
tags:
  - css
categories:
  - web
date: 2018-06-04 18:10:34
---

# 语法
```
选择器
selector {
    属性：值；
    property: value;
    property1: value1;
    property2: value2
}
```
# 引入方式
## 行内式
* 定义：在标签的style属性中设置
* 范例：`<p style="color: chartreuse;font-size: 12px">hello world</p>`

## 嵌入式
* 定义：在head标签下的style标签内引入
* 范例：

```
<style>
    p{
        color: chartreuse;
        font-size: 14px;
    }
</style>
```
## 链接式
* 定义：在head标签下使用link引入css文件
* 范例：`<link rel="stylesheet" href="test.css">`

# 选择器
## 分类
### 标签选择器
```
p{
    font-size: 14px;
    color: #cf9aff;
    background-color: #8dff84;
}
```
### ID选择器
>每个标签有唯一的id属性

```
#aa{
    color: aqua;
    font-size: 29px;
}
```

###  类选择器
>多个标签可以有相同的class属性，一个标签也可以有多个class属性

```
.ceshi{
    background-color: wheat;
}
```
### 自定义属性选择器
```
[item]{
    background-color: wheat
}
[item="2"]{
    background-color: red
}
```
### 混合选择器
#### 逻辑“或”
>逗号分隔

```
h4,.xxx,#p2{
    background-color: wheat;
}
```

#### 逻辑“与”
>标签名在前

```
div.xxx{
    background-color: green
}
p#p2{
    background-color: green
}
```

#### 后代选择
>使用空格分隔基础选择器，示例：为.outer的所有后代p选择器

```
.outer p{
    background-color: wheat
}
```
#### 子选择
>使用大于号分隔基础选择器，示例：为.outer的所有子p选择器

```
.outer >p{
    background-color: wheat
}
```
## 优先级
### 默认权重
* 标签选择器：1
* class选择器：10
* id选择器：100
* 内嵌在属性中的选择器：1000
* 特权【!import】:高于一切
    - 范例：`div{background-color: blue!important;}`

### 同权覆盖
优先级相同时，排在后边的属性覆盖前面的。

# 属性设置
## 文本属性
在未定义时，子类标签的文本类型的属性可以继承父类标签

```
div{
    /*字体大小*/
    font-size: 30px;
    /*字体颜色*/
    color: red;
    /*使标签内的文本或图像左右居中*/
    text-align: center;
    /*设置行高的同时即设置了文本上下居中*/
    line-height: 500px;
}
```
## 背景和边框属性
```
.ceshi1{
    height: 400px;
    /*块标签的宽占浏览器宽度的百分比*/
    width: 100%;
    /*padding 填充
    此区块的内容区块外扩（四周）大小*/
    padding: 50px;
    /*margin 页边距；
    此区块与其他区块的间距，
    第一个为上下间距，第二个为左右间距（auto自动居中【需要搭配width实现居中】）*/
    margin: 20px auto;
    /*边框：厚度 实线 红色*/
    border: 2px solid red;
    /*背景色*/
    background-color: wheat;
    /*背景图片：填充方式 上下左右居中*/
    background: url("http://724.169pp.net/bizhi/2017/025/1.jpg") no-repeat center;
}
```
## display属性
* 不显示：display：none
* 块级别标签转内联标签：display：inline
* 内联标签转块级别标签：display：block
* 既有内联标签又有块级别标签功能：display：inline-block

```
label{
    background-color: wheat;
    display: inline-block;
}
```
## float和clear
* 正常文档流的区块占用一行的位置，相互之间上下排版
* float文档流的多个float区块在一行排版，但不占用一行位置
* 当float区块有内容时，它会占用相邻正常文档流的内容空间，从而让正常文档流内容可以“环绕”漂浮文档流内容排版
* clear清除浮动，假设上一个或左或右浮动的标签为正常文档流，占用一行位置，从而使自己另起新行排版
* float可以对block和line进行操作，对line进行操作类似转为block，可以设置宽和高


```
span{
    background-color: green;
    font-size: 20px;
    height: 50px;
    width: 50px;
    float: left;
}
.div2{
    clear: left;
    float: right;
    width: 130px;
    height: 140px;
    background-color: blue;
}
```
## 定位position
### fixed
>绝对定位

```
.guding{
    width: 80px;
    height: 40px;
    background-color: gray;
    font-size: 15px;
    color: blue;
    text-align: center;
    line-height: 40px;
    position: fixed;
    bottom: 20px;
    right: 10px;
}
```
### relative
>相对定位，直接写则为相对自身原位置的定位

```
.zhuti{
    width: 100%;
    line-height: 500px;
    background-color: wheat;
    text-align: center;
    position: relative;
    top: 200px;
    left: 200px;
}
```
### absolute
>绝对定位，absolute根据前代的一个准确定位取得当前定位；
>一般用法：在父代标签中设置相对定位relative，在子代中设置绝对定位absolute，则此时的定位为相对父代的定位

```
* {
    margin: 0;
}
.box{
    height: 200px;
    width: 80%;
    border: 2px solid red;
    position: relative;
}
.zhuti{
    text-align: center;
    line-height: 100px;
    width: 50%;
    border: 1px solid gold;
    position: absolute;
    right: 50px;
    bottom: 50px;
}
```
## 链接属性
>又称anchor伪类

```
hover 徘徊，萦绕
/*链接默认颜色*/
a:link{
    color: green;
}
/*链接被激活时的颜色*/
a:active{
    color: red;
}
/*链接被访问过时的颜色*/
a:visited{
    color: blue;
}
/*链接样式：去除a标签下划线*/
a{
    text-decoration: none;
}
/*鼠标悬浮在链接上时的颜色*/
/*在 CSS 定义中，:hover 必须位于 :link 和 :visited 之后（如果存在的话），这样样式才能生效。*/
a:hover{
    color: white;
}
```
## before与after
```
/*在标签的开始（before）或结束（after）位置添加子标签*/
p:after{
    /*标签内容*/
    content: "aa";
    color: red;
    /*标签设置为块标签*/
    display: block;
}
```
## 其他属性
* 列表属性：`ul{list-style: none;}`
* 图片属性

```
/*对齐的四线：底线、基线、中线、顶线*/
img{
    /*图片与文本的中线对齐*/
    /*vertical-align: middle;*/
    vertical-align: -100px;
}
```
# 范例
* 正常文档流中，子元素可以填充父元素的宽高
* 漂浮文档流中，子元素不会填充父元素的宽高
* 范例：此标签【类clear】添加一个块级子标签，并且块标签前后清除浮动，使此标签变成标准文档流，占用一行

```
<div class="box clear">
    <div class="div1"></div>
    <div class="div2"></div>
</div>
<div class="footer">test</div>
```
```
.box{
    border: 1px solid red;
}
.clear:after{
    clear: both;
    content: "";
    display: block;
}
.div1{
    width: 150px;
    height: 150px;
    float: left;
    background-color: gold;
}
.div2{
    width: 150px;
    height: 150px;
    float: left;
    background-color: green;
}
.footer{
    height: 200px;
    border: 3px solid yellow;
}
```
