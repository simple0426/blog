---
title: python函数之迭代器
date: 2018-03-02 10:18:09
tags: ['迭代器', '生成器']
categories: ['python']
---
# 迭代
## 概念
* 迭代：重复的过程称为迭代，每次重复即是一次迭代，但是每次重复的结果都是下一次重复的初始值
* 可迭代：每个含有__iter__()方法的数据类型都是可迭代的，都可以使用for循环获取对象中的每一个元素
* for循环：先执行对象的iter方法得到一个迭代器对象，再执行迭代器对象的next方法从而得到对象的元素

```python
a = ['a', 'b', 'c']
# i = a.__iter__()
i = iter(a) #iter方法获取列表的迭代器对象
print(i)
while True:
    try:
        print(next(i)) #next方法获取迭代器的下一个值
    except StopIteration:
        break
```


## 可迭代对象
```python
from collections import Iterable, Iterator
print(isinstance([], Iterable))
print(isinstance((), Iterable))
print(isinstance({}, Iterable))
print(isinstance('', Iterable))
```

## 相关函数 
* zip:[拉链函数]

>接受一系列的可迭代对象作为参数【至少2个】，返回由可迭代对象元素组成的元组；可用于双循环或多循环的取值

```python
names = ['hejingqi', 'hanjianfang', 'xiaofangfang']
ages = [27, 26, 27]
for name, age in zip(names, ages):
        print(name, age)
```

* enumerate

>接收一个可迭代对象和索引初始值，返回由索引和元素组成的元组；可用于序列的取值

```python
from collections import Iterable, Iterator
l = list(range(5))
i = enumerate(l)
print(isinstance(i, Iterator))
for k, v in enumerate(l, 10):
    print(k, v)
```

# 迭代器
## 概念
* 既含有`__iter__()`方法，又含有`__next__()`方法的对象
* 执行对象的`__iter__()`方法得到的结果仍然是他本身

## 优缺点
* 惰性计算：它仅在迭代至当前元素时才计算该元素的值，在此之前和之后都可以不存在，也就是不需要在遍历之前准备好迭代过程的所有元素，所以适合遍历那些有无穷个元素的集合【比如自然数】
*  优点
    * 提供了一种不依赖下标的迭代方式
    * 就迭代器本身来说，更节省内存
*  缺点
    * 无法获取迭代器对象的长度
    * 不如序列类型取值灵活，是一次性的，只能往后取值，不能往前退

# 生成器
## 概念
* 调用生成器函数返回一个生成器（和迭代器类似，返回自身）
* 自动实现了迭代器的功能（相当于为函数封装好iter和next方法）
* 语法上与函数相似，只是将return替换成了yield；但是return只能返回一次值，函数就终止了；而yield能返回多次值，每次返回都会将函数暂停，下次调用会从上一次暂停的位置继续执行

## 定义
* 生成器表达式

>类似列表推导式(简单理解为元组推导式)

```python
g = (x*x for x in range(10))
print(g)
```

* 生成器函数（使用yield返回的函数）

```python
def foo():
    print('first')
    yield 1
    print('second')
    yield 2
    print('third')
    yield 3
g = foo() #函数的执行，得到一个生成器【同时也是一个迭代器】
print(g)
next(g) #使用迭代器的next方法获取对象的元素（此处为执行函数，遇yield第一次返回）
print(next(g)) #执行函数，并打印返回值
```

## send方法
### send定义
* 生成器必须在执行一次next【也即yield一次，产生一个断点】才能接受send发送的值
* send是调用生成器的方法，在断点恢复同时向yield传递值，在下次yield处暂停
* next也是调用生成器的方法，但是不向yield传值，此时yield值为空

### send范例
```python
def echo(value=None):
    while 1:
        value = (yield value)
        print("The value is", value)
        if value:
            value = value + 1
g = echo(1)
print(next(g))
print(g.send(2))
print(g.send(5))
print(next(g))

# 输出
1
The value is 2
3
The value is 5
6
The value is None
None

# 步骤详解
定义生成器时value为1，返回value【1】
value重新赋值等于2【The value is 2】，value重新计算等于3，返回value【3】
value重新赋值等于5【The value is 5】，value重新计算等于6，返回value【6】
next调用，向yield中传递空值，value为空【The value is None】，value重新计算为空，返回空值【None】
```

### send范例详解
* 参考：<http://codingpy.com/article/python-generator-notes-by-kissg/>
* 上述代码既有yield value的形式，又有value = yield形式，看起来有点复杂。但以yield分离代码进行解读，就不太难了。第一次调用next()方法，执行到yield value表达式，保存上下文环境暂停返回1。第二次调用send(value)方法，从value = yield开始，打印，再次遇到yield value暂停返回。后续的调用send(value)或next()都不外如是。
* 在一次next()(非首次)或send(value)调用过程中，实际上存在2个yield，一个作为恢复点的yield与一个作为暂停点的yield。因此，也就有2个yield表达式。send(value)方法是将值传给恢复点yield;调用next()表达式的值时，其恢复点yield的值总是为None，而将暂停点的yield表达式的值返回。

## 使用范例
* 范例1：模拟tail -f a.txt|grep 'python'

```python
import time
def tail(filepath):
    with open(filepath, encoding='utf-8') as f:
        f.seek(0, 2)
        while True:
            line = f.readline().strip()
            if line:
                yield line
            else:
                time.sleep(0.2)
def grep(pattern, lines):
    for line in lines:
        if pattern in line:
            print(line)
grep('python', tail('a.txt'))
```

* 范例2：获取斐波那契数前n个值

>斐波那契数:除第一第二个数之外，其他每个数都由前俩个数相加得到结果

```python
def fib(max):
    n, a, b = 0, 0, 1
    while n < max:
        yield b
        a, b = b, a + b
        n = n + 1
g = fib(3)
for i in g:
    print(i)
```
