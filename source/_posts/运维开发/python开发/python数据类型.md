---
title: python数据类型
date: 2018-05-07 18:02:12
tags: ['数据类型']
categories: ['python']
---
# 数据类型
## 基本数据类型
* 整数
* 小数
    - 小数之所以称为浮点数，是因为按照科学计数法的表示时，小数点的位置是可变的
    - 对于很大或很小的浮点数，就必须用科学计数法表示，把10用e替代，1.23x10^9就是1.23e9
* 字符串
    * 使用\转义特殊字符如\t
    * 使用r''表示内部字符串不转义
    * 使用三引号显示多行内容
* 布尔值
    * 数字中的0为False，其他均为True
    * 空字符串为False，其他均为True
* 空值：None

## python数据类型
* [列表](#列表)
* [字典](#字典)
* [元组](#元组)
* [集合](#集合)

## 变量和常量
* 常量：通常用全部大写的变量名表示常量
* 变量名
    * 包含字母、数字、下划线
    * 不能以数字开头
    * 不能包含内置关键字
    * 命名必须有实际意义
    * 通常将下划线作为变量名字符串的连接符

## 运算符
* 数字运算符

| 符号 |  含义  |    用法   |
|------|--------|-----------|
| +    | 加法   | a = 1 + 2 |
| -    | 减法   | c = 2 -1  |
| *    | 乘法   | d = 2 * 4 |
| /    | 除法   | e = 6/3   |
| //   | 地板除 | b = 5//2  |
| %    | 取余   | h = 5%2   |
| **   | 平方   | g = 3**2  |

* 比较运算符：==、!=、>=、<=
* 逻辑运算符：not 、and、or
* 成员判断：in、not in

## 变与不变
* 可变对象，比如字典、列表、集合；对list进行操作，list内部的内容会发生变化
* 不可变对象，比如str、tuple；调用对象自身的任意方法也不会改变对象自身的内容；相反，这些方法会创建新的对象并返回。

## list和dict对比
|                 list                 |                     dict                    |
|--------------------------------------|---------------------------------------------|
| 插入和查找的时间随着元素的增加而增加 | 插入和查找速度极快，不会随着key的增加而增加 |
| 占用内存较少                         | 占用大量内存，内存浪费比较严重              |


# 列表
## 方法
* append：追加单个元素
* extend：在列表中添加可迭代对象的多个元素（比如：列表中添加列表）
    - 列表拼接【类同字符串拼接】 c = L + L2

```python
L = ['1', '2', '3', '2']
L2 = ('4', 'a')
L.append('a')
L.extend(L2)
```

* clean：清空列表
* count：返回元素出现的次数
* index：返回元素第一次出现的索引位置
* remove：删除第一次出现的元素
    - `del L[1]`：删除指定索引位置的值
* pop：删除指定位置（索引）元素并返回【默认最后一个】
* insert：指定索引位置插入元素
* sort：排序
* reverse：反向排序

## 索引和切片
切片始终创建新列表，不会改变原有列表

```python
L = list(range(10))
print(L[:3])
print(L[-2:])
print(L[::4])
```

## 列表推导式
使用其他列表创建新的列表的方式

```python
L = ['a', 'B', 5, 'C', 'd']
s = [s.lower() for s in L if isinstance(s, str)]
print(s)
```

## 迭代函数
* range：返回一个可迭代的range对象

```python
for i in range(1,5):
    print(i)
```

* enumerate：接受一个可迭代对象，返回一个枚举类型对象；当使用for循环时，每次返回一对数，第一个[默认]是从0开始的计数， 第二个为可迭代对象的元素

```python
list_1 = [1, 3, 4]
for k, v in enumerate(list_1):
        print(k, v)
```

## 范例
* 列表去重【保持列表元素位置不变】

```
list_1 = [1, 3, 5, 1, 7, 4]
list_2 = []

for i in list_1:
    if i not in list_2:
        list_2.append(i)
print(list_2)
```

# 元组
* 元组是不可变数据类型[子元素不可变]  
* 元组的指向固定，但元组的元素内容是可变的[孙元素可变]  
* 变与不变

```
tuple_2 = (1, 3, ['x', 'y'])
tuple_2[2][0] = 'A'
print(tuple_2)
```

* 单元素元组

>因为括号()既可以表示tuple，又可以表示数学公式中的小括号， 
>这就产生了歧义，因此，Python规定，这种情况下，按小括号进行计算

```
tup = (1)
type(tup)
tup1 = (1,)
type(tup1)
```

* 索引和切片

```
a = (1, 3 , 4, 2)
a[1]
a[2:]
```

# 字典

## 定义

* 字典的键值对都是无序的，字典的存储顺序和放入顺序无关
* 字典的key必须是不可变对象【内存中的key以hash值的方式存储，而只有不可变对象才可hash】

```python
a = {(1, 2): 'a', 'a': 'c'}
# True和1等效 False和0等效 会相互覆盖
b = {
    True: 'a',
    1: 'c',
    12: 'd'
}
print(a, b)
```

* 字典构造
    - dict函数（列表或元组中构造）
    ```python
    list_1 = [('hejingqi', 26), ('xiaofang', 27)]
    dict_1 = dict(list_1)
    dict_2 = dict(hejingqi=25, xiaofang=27, zhaoying=24)
    ```
    - 字典推导式
    ```python
    dict_1 = {x: x*2 for x in range(3)}
    print(dict_1)
    ```

## 方法
* update：更新或插入kv

```python
a = {'a': 1}
a.update({'b': 2})
```

* clear：字典内容清空
    - del a['a']：删除元素
* get：取值【不存在不报错】

```python
v = dic.get('c', 'Not')
v1 = dic['a']
print(v, v1)
```

* pop：删除并取值
* popitem：随机删除kv对

```python
v = dic.pop('b')
k, v1 = dic.popitem()
print(v)
print(k, v1)
```

* setdefault：设置默认值【不存在时使用】

```python
dic.setdefault('c', 111)
dic['c'] = 3
print(dic['c'])
```

* fromkeys：用于创建一个新字典，以序列seq中元素作为字典的键，value为字典所有键对应的初始值【没有则为None】

```python
new_dic = dict.fromkeys(['k1', 'k2', 'k3'], [345])
# new_dic['k1'] = 12
# 因为k1,k2,k3对应同一块内存空间
# 原空间为不可变对象时(如元组、常量、字符串)，只能使用赋值操作变更对应的变量值，而赋值操作新开辟一个内存空间
# 原空间为可变对象时（如列表、字典、集合），可以在原空间进行操作改变对应的变量值，则此时k1,k2,k3值还是会相同
new_dic['k1'].append(12)
print(new_dic)
```

* keys：获取key列表【类似选项values、items】

```python
dict_1 = {
    'hejingqi': 30,
    'yuanshuo': 24,
    'yangxiaomeng': 29
}
for k, v in dict_1.items():
    print(k, v)
```

# 集合

## 定义

* 没有重复值的列表，没有value的字典
* 集合会自动排序和去重
* 集合的数学运算

```python
a = set('abracadabra')
b = set('alacazam')
# print(a, b)
print(a - b)
print(a | b)
print(a & b)
print(a ^ b)
```

* 集合推导式

```python
a = {x for x in 'adfaseasdfe' if x not in 'abc'}
print(a)
```

## 方法
* add：添加元素
* discard：删除集合元素【没有不报错】
* remove：删除集合元素【没有则报错】
* clear：清空集合内容
    - del set_1：删除集合
* update：更新集合内容
    - 使用列表、元组、字典、集合等联合数据类型更新已经存在的集合

```python
a = {'x', 'y', 'z'}
b = ['1', '2']
a.update(b)
```

