---
title: python之函数学习
tags:
  - 函数
categories:
  - python
date: 2018-05-09 16:23:13
---

# 函数定义
## 名称空间
* 内置名称空间：python解释器自带的名字所在的空间
* 全局名称空间：文件级别定义的名字所在的空间
* 局部名称空间：函数级别定义的名字所在的空间

## 作用域
* 全局作用域：内置名称空间、全局名称空间
    - 查看全局作用域的名字：globals
* 局部作用域：局部名称空间
    - 查看局部作用域的名字：locals
* 名字查找顺序：局部名称空间》全局名称空间》内置名称空间

## 定义
* 空函数pass
    - 空函数可以作为占位符，在某部分代码没想好之前，可以先让函数运行起来
* 函数定义阶段只检查语法定义错误
* 定义函数时，需要确定函数名和参数个数；如有必要，可以先对参数的数据类型做检查。

## 返回值
* 函数可使用return随时返回值
* 没有return语句时，默认返回None
* 当返回多个值时，以元组形式组成
    - 多个值的解包：a, b, c, d, e = t
    - 只取某些值【下划线占位】：  `a, _, b, _, c = t`
    - 只取首尾的值：`x, *_, y = t`

## 参数
>位置参数必须在关键词参数之前
### 位置参数
* 是必选参数
* 按位置赋值

```python
def foo(x, y):
    return x / y
print(foo(4, 2))
```

### 关键词参数
* 以key-value形式赋值
* 范例：`print(foo(y=4, x=2))`

### 默认参数
* 默认参数在非默认参数之后
* 默认参数只在定义时赋值一次
* 默认参数需要定义为不可变类型
* 范例：
    - 错误使用范例
    
    ```python
    def add_end(L=[]):
    L.append('END')
    return L
    # print(add_end([1,2,3]))
    print(add_end())
    print(add_end())
    # 输出
    ['END']
    ['END', 'END']
    ```

    - 正确使用

    ```python
    def add_end(L=None):
    if L is None:
        L = []
    L.append('END')
    return L
    # print(add_end([1,2,3]))
    print(add_end())
    print(add_end())
    ```

    - 对比解释：
        + python函数在定义的时候，默认参数L的值就被计算出来，即[],因为默认参数L也是一个变量，它指向对象[]
        + 每次调用该函数，如果改变了L的内容，则下次调用时，默认参数的内容就变了，不再是定义时的[]的了。
        + 所以定义默认参数要牢记一点：默认参数必须指向不变对象！

### 可变长参数*args
* *会把溢出的按位置定义的实参都接收，并以元组的形式赋值给args

```python
def foo(*args):
    total = 0
    for i in args:
        total+=i
    return total
print(foo(1,2,3))
```

* 在list或tuple前面加一个*号，把list或tuple的元素变成可变参数传进去

```python
def foo(*args):
    print(args)
li = ['a', 'b', 'c']
foo(*li)
```

### 可变长参数**kwargs
**会把溢出的按关键词定义的实参都接收，并以字典的形式赋值给kwargs 

```python
def test_kw(**kwargs):
    print(kwargs)
dic = {'a': 1, 'b': 2}
test_kw(**dic)
```

### 命名关键字参数
* \*后为命名关键字参数，必须传值，且以关键字形式赋值
* 参数搭配使用及顺序：位置参数/默认参数/\*,命名关键字参数

```python
def foo(x, y=1, *, z):
    print(x, y, z)
foo(2, z=3)
```

### 参数顺序
* 严格的顺序：位置参数、默认参数、可变长参数*args，[\*，命名关键字参数]，可变长参数\**kwargs
* 一般使用：位置参数、默认参数、可变长参数\*args、可变长参数\**kwargs

```python
def bar(a, b='x', *args, **kwargs):
    print(a, b, args, kwargs)
bar(1,2,3,4, x='a', y='g')
```

# 内置函数
## 递归函数
### 语法特点
* 递归函数的优点是定义简单，逻辑清晰。理论上所有递归函数都可以写成循环的方式。但循环的逻辑不如递归清晰
* 使用递归函数需要防止栈溢出。在计算机中，函数调用是通过栈【stack】这种数据结构实现的，每当进入一个函数调用，栈就会增加一层栈帧。每当函数返回，栈就会减少一层栈帧。由于栈的大小不是无限的，所以递归调用的次数过多就会导致栈溢出。
* 尾递归可以解决递归栈溢出，但由于python解释器没有对尾递归做优化，依然会存在栈溢出，只要保证函数嵌套不超过100一般就没问题。

### 范例（汉诺塔）
把圆盘从下面开始按大小顺序重新摆放到另一根柱子上。并且规定，在小圆盘上不能放大圆盘，在三根柱子之间一次只能移动一个圆盘。

```python
def hanoi(n,x,y,z):
    if n==1:
        print(x,'-->',z)
    else:
        hanoi(n-1,x,z,y) #将前n-1个盘子从x移动到y上
        hanoi(1,x,y,z) #将最底下的最后一个盘子从x移动到z上
        hanoi(n-1,y,x,z) #将y上的n-1个盘子移动到z上
hanoi(5,'x','y','z')
```

## lambda
### 语法
* 语法：lambda 参数：表达式
* 匿名函数有个限制，就是只能有一个表达式，不用写return，返回值就是该表达式的结果

### 范例
```python
f = lambda x, y: x if y > x else y
print(f(3, 2))
```

## map
### 语法
* map函数接收两个参数，一个是函数，一个是迭代器
* map将传入的函数依次作用到序列的每个元素，并把结果作为新的迭代器返回

### 范例
```python
result = map(lambda x:x*x, [1, 3, 5])
print(list(result))
```

## reduce
### 语法
* 在python3中，reduct函数已经被从全局名字空间中移除了，它被放置在functools模块里，用的话需要先引入
* reduce函数接收两个参数，一个是函数且必需有两个参数，一个是迭代器
* reduct会把相邻的两个元素使用函数处理后，再把处理的结果和相邻的元素进行处理，以此类推。

```python
from functools import reduce
result = reduce(lambda x,y: x + y, range(1, 10))
print(result):
```
### 范例
将数字字符串转换为数字

```python
from functools import reduce
def str2int(s):
    def char2num(s):
        return dict(zip('0123456789', range(0,10)))[s]
    def fn(x, y):
        return 10 * x + y
    return reduce(fn, map(char2num, s))
print(str2int('123'))
```

## filter
### 语法
* filter也接收一个函数和一个迭代器。
* 和map不同的是，filter把传入的函数依次作用于每个元素，然后根据返回值是True还是False决定保留还是丢弃该元素

```python
def is_odd(n):
    return n % 2 == 1
result = list(filter(is_odd, [1,2,3,6]))
print(result)
```

### 范例
* 质数：在大于1的自然数中，除了1和它本身之外不再有其他因数
* 删除1-100内的质数
    - 利用filter函数，只显示有返回值的函数结果【是质数，不返回；不是质数则返回】
    - 单个数除以比它小但大于2的数，如果余数为0说明不是质数，直接返回。

```python
def not_prime(n):
    flag = False
    for i in range(2, n):
        if n % i == 0:
            flag = True
    return flag
print(list(filter(not_prime, range(1, 101))))
```

## sorted
* 第一个参数接收可迭代的序列
    - list的sort属性只可用于list类型
* key为只接收一个参数的函数，以此定义排序依据
* reverse为是否逆序排列
* 结果：生成新序列的副本
    - list的sort属性变更list生成新列表

```python
student_tuple = (('john', 'A', 15), ('jane', 'B', 12), ('dave', 'B', 10))
print(sorted(student_tuple, key=lambda student:student[2]))
```

# 闭包
* 定义在函数内部的函数，它包含对外部作用域而非全局作用域的引用，该内部函数就是闭包函数
* 一个闭包就是你调用了函数A，这个函数A返回了一个函数B给你，这个返回的函数B就是闭包；调用函数A传递的参数就是自由变量

```python
def funcA():
    x = 1
    def funcB():
        print(x) #可以引用外部作用域的变量x
    return funcB
f = funcA()
f()
```
```python
from urllib.request import urlopen
def index(url):
    def get():
        return urlopen(url).read()
    return get
res = index('https://blog.unforget.cn')
print(res().decode('utf-8'))
```

# 装饰器
## 作用
符合开闭原则：对源代码修改封闭，对功能扩展开放
## 语法
>多个装饰器，从上往下依次执行，从下往上依次装饰

|           写法          |                                      使用decorator                                       |                                          不使用decorator                                          |
|-------------------------|------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| 单个Decorator，不带参数 | @dec<br>def method(args):<br>&emsp;pass                                                  | def method(args):<br>&emsp;pass<br>method = dec(method)                                           |
| 多个Decorator，不带参数 | @dec_a<br>@dec_b<br>@dec_c<br>def method(args):<br>&emsp;pass                            | def method(args):<br>&emsp;pass<br>method = dec_a(dec_b(dec_c(method)))                           |
| 单个Decorator，带参数   | @dec(params)<br>def method(args):<br>&emsp;pass                                          | def method(args):<br>&emsp;pass<br>method = dec(params)(method)                                   |
| 多个Decorator，带参数   | @dec_a(params1)<br>@dec_b(params2)<br>@dec_c(params3)<br>def method(args):<br>&emsp;pass | def method(args):<br>&emsp;pass<br>method = dec_a(params1)(dec_b(params2)(dec_c(params3)(method))) |

## 范例
### 无参装饰器
```python
import time, functools
def timmer(func):
    # 添加此装饰器后可保持原函数属性不变
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        func(*args, **kwargs)  #index()
        end_time = time.time()
        print('run time is %s' % (end_time - start_time))
    return wrapper

@timmer  #timmer(index)
def index():
    time.sleep(3)
    print('welcome!')

@timmer
def index1(args):
    time.sleep(3)
    print('%s' % args)
index()
index1('test')
```
### 有参装饰器
```python
import time
def log(text):
    def decorator(func):
        def wrapper(*args, **kwargs):
            print('%s %s' % (text, func.__name__))
            return func(*args, **kwargs)
        return wrapper
    return decorator
@log('execute')
def now():
    print(time.time())
now()
print(now.__name__)
```
### 装饰器的叠加
```python
def decorator_a(func):
    print('Get in decorator_a')
    def inner_a(*args, **kwargs):
        print('Get in inner_a')
        return func(*args, **kwargs)
    return inner_a

def decorator_b(func):
    print('Get in decorator_b')
    def inner_b(*args, **kwargs):
        print('Get in inner_b')
        return func(*args, **kwargs)
    return inner_b

@decorator_a
@decorator_b
def f(x):
    print('Get in f')
    return x * 2
# 函数定义阶段，decorator_b将f作为参数，取得“Get in decorator_b”
# decorator_a把decorator_b作为参数，取得"Get in decorator_a"
f(1)
# 函数调用阶段，执行等同于decorator_a(decorator_b(f(1))),
# 依次执行decorator_a，decorator_b，f,取得“Get in inner_a”、“ Get in inner_b”“Get in f”
```


