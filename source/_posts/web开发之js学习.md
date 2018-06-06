---
title: web开发之js学习
tags:
  - js
  - javascrip
categories:
  - web
date: 2018-06-06 15:47:25
---

# 引入方式
head标签下导入js文件

```
<script type="text/javascript" src="test.js"></script>
```
# 组成部分
* [ECMAScript](#ECMAScript)：js语法规范
    - [变量定义](#变量定义)
    - [基本数据类型](#基本数据类型)
    - [数据对象](#数据对象)（引用数据类型）
    - [运算符](#运算符)
    - [流程控制](#流程控制)
* [BOM对象](#BOM对象)（浏览器对象模型）：整合js和浏览器
* [DOM对象](#DOM对象)（文档对象模型）：整合js、css、html

# ECMAScript
## 变量定义
* 变量声明使用关键词var，如果不用var，则声明的是全局变量
* 变量名区分大小写，首字符可以是：美元$、下划线、字符，余下的字符可以是：字符、数字、下划线、美元$

## 基本数据类型
### 类型
* 字符串
* 数值
* 布尔值：true或false
* null：空值
* undefined：只声明，未赋值

### 特例
NaN属于数字类型，是在字符串转换为数字失败时返回的

```
var a = 'test'
b = +a
console.log(b)
```
## 运算符
* 算术运算符：+   -    *    /     %       ++        -- 
* 比较运算符：>   >=   <    <=    !=    ==    ===   !==
    - 由于js是弱类型语言，在进行数据比较前默认会进行类型转换
    - 当比较对象中存在数字类型时，会把其他数据转换为数字后进行比较
        + ==在代码内部可以先进行数据类型转换再进行比较，如console.log(2=="2")
        + ===在代码内部不能进行数据类型转换后进行比较，如console.log(2==="2")
    - 当比较对象中存在字符串时，会把其他数据转换为字符串进行比较
* 逻辑运算符：&&   ||   ！
* 赋值运算符：=  +=   -=  *=   /=
    - ++i 先加1再赋值
    - i++ 先赋值再加1
* 字符串运算符：+  连接，两边操作数有一个或两个是字符串就做连接运算

## 流程控制
### if-else
>没有elif

```
if( 5 > 4){
    console.log('ok')
}else {
    console.log('not')
}
```
### switch-case
```
var y = 5
switch (y) {
    case 5:console.log('5');break;
    case 2:console.log('2');break;
    default:console.log('未知数');break;
}
```
### for条件循环
```
Things = ['a', 'b', 'c']
// 初始表达式；条件表达式；自增或自减
for (var i = Things.length - 1;i >= 0;i--){
    console.log(Things[i])
}
```
### for遍历循环
```
Things = ['a', 'b', 'c']
for(item in Things){
    console.log(Things[item])
}
```
### while循环
```
var i = 0
while (i < 10){
    console.log(i)
    i++
}
```
### 异常处理try-catch
```
try{
    //可能出现异常的代码
    y=+x
}catch (e){
    // 对异常的处理
    console.log(e)
}finally {
    // 无论是否异常都执行的代码
    alert('nothing')
}
```
### 异常处理throw
直接抛出异常：throw Error('ceshi')
# 数据对象
|   对象   |      说明      |
|----------|----------------|
| Number   | 数字对象       |
| String   | 字符串对象     |
| Boolean  | 布尔对象       |
| Array    | 数组对象       |
| Match    | 数学对象       |
| Date     | 日期对象       |
| Object   | 自定义对象     |
| Error    | 错误对象       |
| Function | 函数对象       |
| RegExp   | 正则表达式对象 |
| Global   | 全局对象       |
## 字符串对象
* toUpperCase() 转换为大小
* length 长度
* toLowerCase() 转换为小写
* trim() 去除两侧空白
* charAt(2) 指定索引位置字符
* indexOf('bc') 指定字符串出现时首字符索引位置
* lastIndexOf('bc') 指定字符串最后一次出现的首字符索引位置
* match(regexp) 返回匹配字符串组成的数组
* search(regexp) 返回匹配字符串首字符索引位置
* substring(n, m) 取索引n到m之间的字符串
* substr(n, m)从索引n开始取m个字符
* slice(n, m) 取索引n到m之间的字符串
* replace('bc', '12')将字符串内的第一次出现'bc'替换成'12'
* split(' ', 3) 使用空格切割字符串,最多返回3个分割后的元素
* concat('12') 字符串末尾追加字符串'12'

## 数组对象
* join('#') 使用'#'链接数组元素拼接成字符串
* concat(4, 5) 在数组内追加元素
* sort() 排序【默认以字符的ASCII码排序】
* reverse() 倒序输出
* slice(2, 4) 切片操作（包含开始不包含结束，索引可以为负数）
* splice(1, 2, 'c') 参数1为开始操作的索引，参数2为删除的元素个数，参数3为为替换或插入的元素
* push() 在末尾追加元素
* pop() 弹出末尾元素
* unshift() 在开头添加元素
* shift() 弹出开头元素
* toString() 转换为字符串输出

### 数字的排序
```
function Insort (a, b) {
    return a - b;
}
var arry = [11, 2, 13]
console.log(arry.sort(Insort))
```

## 日期对象
* Date()有参数时为创建时间对象，无参数时为获取当前时间
* getFullYear() 获取指定的时间属性(此处为年)
* toLocaleString() 将时间对象转换为本地格式的字符串

## 数学对象
* Math.random() 获取0~1之间的随机数
* Math.round(n) 通过四舍五入将n转换为整数

## 函数对象
* 定义和调用：由于js是整体加载完才会执行，所以函数定义和调用的前后顺序无关
* 参数个数：func_name.length
* arguments对象：可以接受任意个参数并组成数组

```
function addnum () {
    var sum = 0;
    for (var i = 0; i < arguments.length; i++) {
        sum+=arguments[i]
    }
    return sum
}
alert(addnum(1, 2, 3, 6))
```

* 匿名函数

```
alert(
    function (x, y) {
        return x + y
    }(4, 5)
)
```
# BOM对象
## 介绍
windows对象：一个html文档对应一个windows对象
## 方法
* alert('123') 弹出含有确认按钮的提示框
* confirm('确认') 弹出含有确认和取消按钮的提示框，可以接收用户的点击信息（true或false）
* prompt('请输入数字：') 弹出可以输入的提示框，可以接收用户的输入信息
* open(url,,'浏览器设置') 打开一个新的浏览器窗口
* close() 关闭浏览器窗口
* setTimeout(close_win, 5000) 在指定的毫秒数之后执行函数或表达式
* clearTimeout(ID) 清除设置的一次性任务
* setInterval(getinfo, 1000) 设置每隔多少毫秒执行函数或表达式
* clearInterval(ID) 清除设置的周期性任务

## 范例
### 一次性任务
```
new_window = open('http://www.baidu.com', '', 'width=1000,height=500')
function close_win () {
    new_window.close();
}
id = setTimeout(close_win, 5000)
clearTimeout(id)
```
### 周期性任务
* html

```
<form action="">
    <input type="text" onclick="start()" id="time">
    <input type="button" onclick="stop()" value="停止">
</form>
```

* javascript

```
function getinfo() {
    var text = document.getElementById('time')
    var date = new Date().toLocaleString();
    text.value = date
}
var ID = undefined
function start() {
    if (ID == undefined){
        getinfo();
        ID = setInterval(getinfo, 1000);
    }
}
function stop() {
    if (ID != undefined){
        clearInterval(ID);
        ID = undefined;
    }
}
```
# DOM对象
## 介绍
>一个html就是一个DOM对象

* document对象（整个html）
* element对象（标签元素）
* text对象（文本内容）
* attribute对象（标签属性）
* 注释对象（标签注释）

## 节点查找
__由于正文标签只有在全部加载后才能进行查找，所以所有涉及到标签查找的js必须位于body标签的末尾__
### 直接查找
* document.getElementById()：通过ID属性查找
* document.getElementsByTagName()：通过标签名查找
* document.getElementsByName()：通过name属性查找
* document.getElementsByClassName()：通过class属性查找

### 导航查找
 >需要先通过直接查找确定一个基点
 >如：var ele = document.getElementsByClassName("box2_div")[0];

* ele.parentElement 父节点标签
* ele.children 所有子节点标签
* ele.firstElementChild 第一个子节点标签
* ele.lastElementChild 最后一个子节点标签
* ele.nextElementSibling 下一个兄弟节点标签
* ele.previousElementSibling 上一个兄弟节点标签

## 节点增删
### 创建节点
```
var tag = document.createElement('input')
tag.type = 'text'
tag.innerText = 'aa'
```
### 添加子节点
* 在标签内末尾添加子节点：ele.append(tag)
* 在标签内指定子标签前添加子节点：ele.insertBefore(tag, ele1)

### 删除子节点
ele.removeChild(ele1)
### 替换子节点
ele.replaceChild(new_node, old_node)

## 节点修改
### 节点文本操作
* 设置文本：tag.innerText
* 设置为html：tag.innerHTML

### 节点类操作
* 类名：ele.className、ele.classList
* 添加类：ele.classList.add('he')
* 删除类：ele.classList.remove('yuan')

### 节点属性操作
* 获取属性值：ele.getAttribute('age')
* 设置属性值：ele.setAttribute('sex', 'nv')
* 删除属性值：ele.removeAttribute('age')

## 节点事件
### 使用方法
* 在标签中设置相应事件，并调用相关函数

```
<input type="button" onclick="stop()" value="停止">
```

* 在js文件中查找相关标签并触发相关事件

```
var ele = document.getElementById('bb')
ele.onclick = function () {
    alert('ok');
    //this指代当前标签
    console.log(this);
}
```
### 事件方法
* onlick：鼠标单击
* ondbclick：鼠标双击
* onfocus：元素获取焦点【input标签】
* onblur：元素失去焦点【input标签】

```
var ele = document.getElementById('aa')
ele.onfocus = function (argument) {
    this.value = ''
}
ele.onblur = function (argument) {
    if(!this.value.trim()){
        this.value = 'ceshi'
    }
}
```

* onchange：当域的内容被改变（如下拉框）

```
var ele = document.getElementById('age');
ele.onchange = function () {
    confirm('确定？')
}
```

* onkeydown 当某个键被按下

```
ele.onkeydown=function (event) {
    // 兼容设置
    event=event||window.event;
    // asicc码
    var keynum=event.keyCode;
    // asicc转字符
    var keychar=String.fromCharCode(keynum);
    console.log(keychar);
}
```

* onkeypress：当某个键被按下并松开
* onkeyup：当某个键被松开
* onload：一个页面或一张图像完成加载

```
var ele= document.getElementsByTagName('body')[0]
ele.onload=function () {
    alert('ok')
}
//window.onload事件中可以定义整个窗口加载完执行的动作，
//从而可以在head标签下的script中定义html中的标签事件
window.onload=function (argument) {
    /* body... */
}
```

* onmousedown：鼠标按钮被按下
* onmousemove：鼠标移动
* onmouseover：鼠标移动到某个元素上
* onmouseout：鼠标从某个元素离开
* onmouseleave：鼠标从元素离开
* onselect：文本被选中
* onsubmit：确认按钮被点击（只能用于form元素）

```
var ele = document.getElementsByTagName('form')[0]
ele.onsubmit=function (event) {
    if(this.firstElementChild.value == 'test'){
        console.log('请不要测试！');
        event.preventDefault();
    }
}
//阻止默认事件方式1
ele.onsubmit=function (argument) {
    return false;
}
//阻止默认事件方式2
ele.onsubmit=function (event) {
    event.preventDefault(); 
}
```

* event.stopPropagation 阻止事件扩散

```
ele2.onclick=function (event) {
    alert(456);
    //阻止事件从子元素扩展到父元素
    event.stopPropagation();
}
```


