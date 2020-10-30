---
title: python数据类型之字符串
tags:
  - 格式化
  - 字符串
date: 2018-05-02 18:04:10
categories: ['python']
---

# 字符串方法
* isdecimal/isdigit/isnumberic：判断是否为数字型字符串
```
v1 = a.isdecimal() #2
v2 = a.isdigit() #2 ②
v3 = a.isnumeric() #2 ② 二/贰
print(v1, v2, v3)
```
* istitle：判断首字母是否大写
* isidentifier：变量名有效性判断

```python
n = 'temp'
v = n.isidentifier()
print(v)
```

* capitalize：使字符串首字母大写
* upper/lower：字符串变大小写
* startswith/endswith：判断是否以某字符开头或结束

```python
s = 'abcd'
s.startswith('a')
s.endswith('d')
```

* strip：移除空白或字符串

```python
m = ' abc '
m.lstrip() #移除左侧空白
m.rstrip()
m.strip() #移除两端空白
n = 'abc'
n.strip('c') #移除字符c
```

* split：切割
* splitlines：按行分割

```python
a = 'a|123|3'
a.split('|') #默认以空格分割，只能是单字符分隔符
```

* replace：替换

```python
n = 'ab12c'
n.replace('a', '3') #替换所有字符
n.replace('a', '3', 1) #只替换第一个字符
```

* center：字符串居中

```python
st = 'oldboy'
v = st.center(10, '#')
print(v)
```

* encode：编码

```python
name = '测试'
print(name.encode('utf-8'))
print(name.encode('gbk'))
```

* format：字符串格式化【关键字(key=value)方式传递值】
* format_map：字符串格式化【字典({key: value})方式传递值】

```python
ori = 'my name is {name}, my age is {age}'
# new = ori.format(name='he', age=12)
new = ori.format_map({'name': 'he', 'age': 12})
print(new)
```

* join：join可以拼接任意数量的字符串，调用此方法的字符串将被插入多个字符串之间形成一个新的字符串

```python 
a = '0'
d = ['1', '2', '3']
c = 'xyz'
e = a.join(d)
h = a.join(c)
print(e, h)
```
# 子串判断
```python
if a in b: #如果a在b中
    print(a)
if a is b: #如果a和b相等
    print('ok')
```
# 索引和切片
```python
a = "abcd"
print(a[1], a[::2]) # 取第2个元素；从第一次元素开始，每隔一个取一个元素。
```

# 字符串格式化
## 占位符
* %d 十进制整数(decimal)
* %s 字符串(string)
* %f 浮点数(float)
* %x 十六进制数(hex)
* %o 八进制数(octal)

## 百分号方式
```python
print('my name is %s,my age is %d' % ('he', 29))
```

## format方式
* 字符串索引方式
    - `'my name is {name}, my age is {age}'.format(name='he', age=29)`
* 数字索引方式
    - `'my name is {1}, my age is {0}'.format(29, 'he')`

# 数字格式化
## 小数一般处理
round(3.16, 1) #保留一位小数，按四舍五入处理

## 数字的格式化处理

| 填充符号 | 对其方式 | 符号 |   宽度   |   精度   | 数据类型 |
|----------|----------|------|----------|----------|----------|
| 任意字符 | ><=^     | +-   | 任意数字 | 任意数字 | 占位符   |

```python
"{:*>+10.3f}".format(34)
"{:0=+10.3f}".format(34)
'{a}\'long format is {a:0=+10.2f}'.format(a=34)
"The number {1} in hex is:{1:#x}, the number {0} in oct is {0:#o}".format(45, 4746)
```

# 元组格式化
```python
tup = ('xiaomeng', 'jingqi')
print('list is %s' % (tup,))
'list is {}'.format(tup)
```
