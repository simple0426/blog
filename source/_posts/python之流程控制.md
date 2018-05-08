---
title: python之流程控制
tags:
  - 流程控制
categories:
  - python
date: 2018-05-08 17:44:47
---

# 流程语句
## if-else
```python
name = input('input names:').strip()
if name == 'oldboy':
    print('superadmin')
elif name == 'alex':
    print('admin')
elif name == 'jingqi':
    print('man')
else:
    print('other')
```

## while
* 当条件为真时永远执行
* 范例：1-2+3-4...+99

```python
result = 0
i = 1
while i < 100:
    if i % 2 == 0:
        result = result - i
    else:
        result = result + i
    i = i + 1
print(result)
```

## break
* 跳出循环体，不再执行循环
* break只能跳出单层循环，跳出多层循环需要在内循环设置变量，在外层循环捕捉变量再次跳出
* 范例：3次登陆失败则退出

```python
i = 0
while i < 3:
    user = input('input user:').strip()
    password = input('input password:').strip()
    if user == 'alex' and password == 'newb':
        print('welcome')
        break
    else:
        print('输入错误')
        i += 1
```

## continue
* 退出本次循环，继续下次循环
* continue只能跳出单层循环的本次循环
* 范例：1到10的循环，遇到5不执行任何操作

```python
for i in range(1, 10):
    if i == 5:
        continue
    else:
        print(i)
    print('ok')
```

## for-else|while-else
* 在循环体内没有break语句、没有return语句、或者没有异常都会正常执行else一次
* 范例
    - 正常执行else

    ```python
    for i in range(1, 4):
        print(i)
    else:
        print('everything is ok')
    ```

    - 不执行else

    ```python
    for i in range(1, 4):
        if i == 3:
            break
        else:
            print(i)
    else:
        print('everything is ok')
    ```

# 异常处理
## try-except
* 语法

```python
try：
          需要捕获的异常源码部分
except    期望捕获的异常类型：
          捕获异常类型后执行的操作
else：
          未捕获异常执行的操作
finally：
          无论上述代码执行结果都会执行此部分代码
```

* 需要注意的是，它不但捕获该类型错误，还把其子类也"一网打尽"
* 如果错误没有被捕获，它就一直往上抛，最后被解释器捕获，打印一个错误信息，然后程序退出
* [常见的错误类型和继承关系](https://docs.python.org/3/library/exceptions.html#exception-hierarchy)
* 范例

```python
try:
    print('try...')
    # a = 5 + 'a'
    r = 10 /0
except ZeroDivisionError as e:
    print('except:', e)
except BaseException as b:
    print('except:', b)
else:
    print('No error!')
finally:
    print('finally')
print('END')
```

## raise
* 更改抛出的错误类型【单写raise原样抛出】

```python
try:
    result = 10/0
except ZeroDivisionError as e:
    # raise ValueError('Error!')
    raise
```

* 抛出自定义错误

```python
class FooError(ValueError):
    pass

def foo(s):
    n = int(s)
    if n == 0:
        raise FooError('invalid value:%s' % s)
    print(10/n)
foo('0')
```

## assert
如果第一个条件为假时，抛出第二个语句

```python
def foo(s):
    n = int(s)
    # n!=0为假时，抛出后边的断言
    assert n !=0, 'n is zero'
    print(10/n)
foo('0')
```
