---
title: web开发之jquery学习
tags:
  - jquery
  - ajax
  - json
categories:
  - web
date: 2018-06-08 15:25:02
---

# 介绍
* jquery是一个快速、简洁的JavaScript框架，
* 它封装了JavaScript是常用的功能模块，提供了一种简便的JavaScript设计模式，优化HTML文档操作、事件处理、动画设计和Ajax交互
* [下载地址](https://jquery.com/)

# 语法
* 语法：$(selector).action()
* 导入方式：`<script type="text/javascript" src="jquery-3.1.1.js"></script>`

## echo循环
### 无标签对象的循环
```
var lis = [11, 22, 33];
$.each(lis, function (index, val) {
    console.log(index, val);
})
```
### 有标签对象的循环
```
$("li").each(function(index, el) {
     if ($(this).text() == '22') {
        $(this).css('color', 'red');
     }   
});
```
# 节点查找
>jquery支持链式操作

```
$('.title').click(function(event) {
  //jquery支持链式操作
  $(this).next().removeClass('hide').parent().siblings().children('.con').addClass('hide');
});
```
## 基本查找
* 标签选择器：`$("tag")`
* ID选择器：`$("#id")`
* 类选择器：`$(".class")`
* 自定义属性选择器：`$('[egon="ceshi1"]') $('[egon]')`

## 导航查找
>需先通过基本选择器确定一个基点

* `$(".box2").sibling()`：所有兄弟标签
* `$(".box2").next()`：下一个兄弟
* `$(".box2").nextAll()`：下面所有兄弟
* `$(".box2").nextUntil('.p3')`：下面直到.p3标签的所有兄弟
* `$(".box2").prev()`：前面兄弟
* `$(".box2").prevAll()`：前面所有兄弟
* `$(".box2").prevUntil(".p3")`：前面直到.p3标签的所有兄弟
* `$(".box2").children('p')`：寻找子标签
* `$(".box2").find('p')`：寻找后代标签
* `$(".box2").parent()`：查找父标签

## 混合查找
* 逻辑“或”连接：`$(".class,#id,div")`
* 子代选择：`$(".outer>p")`
* 后代选择：`$(".outer p")`
* 向下的兄弟选择：`$(".box1~p")`
* 向下的毗邻选择（紧挨）：`$(".box1+p")`

## 筛选器
* `$("ul li").eq(2)`：索引为2的li
* `$("ul li:lt(2)")`：索引小于2的li
* `$("ul li:odd")`：索引为奇数的li
* `$("ul li:even")`：索引为偶数的li

# 节点事件
## 页面载入
>等同于window.load

```
$(function (argument) {
    alert('123')
})
```
## 事件绑定
* 给已存在的标签绑定事件
* 与js区别：事件没有on关键词
    - js：js标签对象.on事件=函数
    - jquery：js标签对象.事件(函数)

```
//js方式
var ele = document.getElementsByClassName("box2")[0]
ele.onclick = function () {
    alert('123')
}
//jquery方式
$(".box2").click(
    function () {
        alert("123")
    }
)
```
## 事件委派
* 无论子标签是否存在
* 父标签委派事件给子标签
* 语法：父标签.on(事件，子标签，函数)

```
$(".box1").on("click", 'p', function () {
    alert(345);
})
```
# 节点修改
## 属性与值
* 添加类属性：$(".box1").addClass("aa")
* 移除类属性：$(".box1").removeClass("aa")
* 设置或获取自有属性：$(".box1").prop("id")
* 移除自有属性：$(".box1").removeProp("id")
* 设置或获取自定义属性：$(".box1").attr("age", 18)
* 移除自定义属性：$(".box1").removeAttr("sex")
* 设置或获取标签html：`$(".box2").html('<label>yes</label>')`
* 设置或获取标签文本：`$(".box2").text('ceshi')`
* 清空标签内容：$(".aaa").empty()
* 获取input标签的value值：$("input").val()

## css操作 
* 设置标签的css：$(".box1").css({"color":"gold", "background-color":"green"})

## 显示效果
* 隐藏标签：$(".box2").hide()
* 显示标签：$(".box2").show()
* 标签显示隐藏切换：$(".box").toggle()
* 滑入：$("").slideUp()
* 滑出：$("").slideDown()
* 滑入滑出切换：$("").slideToggle() 
* 淡入：$("").fadeIn() 
* 淡出：$("").fadeOut() 
* 淡入淡出切换：$("").fadeToggle() 

```
//显隐切换
$("button").click(
    function () {
        if($(this).text() == 'hide'){
            $("p").hide();
        }else if($(this).text() == 'show'){
            $("p").show();
        }else {
            $("p").toggle();
        }
    }
)
```

# 节点增删
## 创建节点
* 创建：`var $ele1 = $("<p>")`
* 克隆：var $ele1 = $(".aaa").clone()

## 添加节点
* 父节点末尾添加：$("div").append($ele1)
* 父节点开始添加：$("div").prepend($ele1)
* 指定节点之后添加：$(".aaa").after($ele1)
* 指定节点之前添加：$(".aaa").before($ele1)

## 删除节点
>操作自身

$(".aaa").remove() 

## 替换节点
>操作自身

$(".aaa").replaceWith($ele1)

```
//添加标签【克隆】
$(".add").click(function (event) {
    var $outer=$(this).parent().clone();
    $outer.children('button').text('删除').attr('class', 'remove');
    $("body").append($outer);
});
//删除标签【事件委派】
$("body").on("click", ".remove", function (event) {
    $(this).parent().remove();
});
```
# json
## 介绍
* JSON（JavaScript Object Notation）：js对象标记；它使用JavaScript语法来描述数据对象，但是它仍独立于语言和平台。
* json是存储和交换文本的语法。类似XML，但是比XML更小、更快，更易解析。
* json文本的MIME类型是"application/json"

## 语法
* 数据在名称/值对中，值可以是
  - 字符串【在双引号中】
  - 数值【整数和浮点数】
  - 布尔值【true和false】
  - 对象
  - 数组
  - null
* 数据由逗号分隔
* 花括号保存对象
* 方括号保存数据

## 使用
* json对象转换为json字符串（stringify）：console.log(JSON.stringify({"name": "xiaofang"}))
* json字符串转换为json对象（parse）：console.log(JSON.parse('{"name": "hejingqi"}'))

# ajax
## 简介
* AJAX（Asynchronous JavaScript And XML）：异步JavaScript和XML
* 异步向服务端发送数据
* 浏览器局部刷新
* 语法：$.ajax({key1: value1})

## 参数
* url：ajax向后端提交数据的url
* type：提交数据的方式（默认get）
* data：向后端传送的数据
* contentType：提交数据MIME类型(默认application/x-www-form-urlencoded)
* success：当后端处理成功时，前端的处理方式
* traditional：（true/false） 当要传输的数据为多维数组时使用
* error：后端错误时的处理方式
* complete：后端处理完成时的处理方式
* statusCode：根据后端处理结果的返回码进行处理

## 范例
* js实现

```
    <script>
        $("#username").blur(function () {
            var $user = $(this).val();
            //csrf设置
            $.ajaxSetup({
                    data: {csrfmiddlewaretoken: '{{ csrf_token }}'}
            });
            if($user.trim().length != 0) {
                $.ajax({
                    url: "checkuser",
                    type: "POST",
                    data: {username: $user},
                    success: function (data) {
                        console.log(data)
                    }
                })
            } else {console.log('user is empty')};
            })
    </script>
```

* python实现

```
def checkuser(request):
    username = request.POST.get('username')
    if username == 'he':
        return HttpResponse("user is exist")
    else:
        return HttpResponse('user not exists')
```
## 注意
当数据传输使用json格式时，由于django的csrf中间件默认使用“application/x-www-form-urlencoded”格式获取csrftoken值，  
此时由于格式不匹配，造成后端无法正确获取csrftoken值从而造成csrf Forbidden错误，目前暂未解决，建议使用默认数据格式传输。
