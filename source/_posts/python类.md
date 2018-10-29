---
title: python类
tags:
  - class
categories:
  - python
date: 2018-05-07 15:08:57
---

# 概念
## 面向对象
* 面向过程：面向过程的程序设计把计算机程序视为一系列的命令集合，即一组函数的顺序执行
* 面向对象：面向对象的程序设计把计算机程序视为一组对象的集合，而每个对象都可以接收并处理其他对象发送的消息，程序的执行即为一系列消息在各个对象之间的传递
    - 在python中一切皆为对象，对象包括数据（属性）和操作数据的函数（方法）
    - object是一个基类或元类，在python2中继承object为新式类，否则为经典类；python3中默认都是新式类（无论是否明写object）
    - 类的三大特性：数据[封装](#封装)、[继承](#继承)、[多态](#多态)

## 封装
* 类的属性和方法可以共用一套数据【例如下例中echo方法中可以调用age属性的值】
* 在python中，以双下划线开头同时以双下划线结尾的是特殊变量，可以直接访问
* 以下划线开头的是私有变量：单下划线开头的变量在导入类时会被忽略，双下划线是类的私有变量

```
class test:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    def echo(self):
        print('my age is %s' % self.age)

person1 = test('he', 27)
person1.echo()
```

## 多态
* 是指由继承而产生的相关的不同类，这些类的实例对象会对同一消息作出不同的响应
* 例如：狗和鸡都有“叫”这一方法，但是调用狗的“叫”，狗会吠叫；调用鸡的“叫”，鸡则会啼叫。

## 继承
```
class Animal():
    def run(self):
        print('Animal is running')

class Dog(Animal):
    def run(self):
        print("Dog is running")
# 当子类和父类都存在相同的run()方法时，我们说，子类的run()覆盖了父类的run()，在代码运行的时候，总是会调用子类的run()。
dog = Dog()
dog.run()
```
### 多重继承
在进行主线继承的同时，需要“混入”额外的功能，这可以通过多重继承来实现，这种设计模式即为Mixin

```
class Animal(object):
    pass

class Mammal(Animal):
    pass

class Runnable(object):
    def run(self):
        print("Running")

class Dog(Mammal, Runnable):
    pass
```
### MRO
* MRO即method resolution order，主要用于在多重继承时判断方法的调用路径（来自哪个类）
* 在新式类中，查找一个要调用的函数或属性时，采用广度优先原则。范例如下：

```
class D(object):
    def foo(self):
        print("class D")
class B(D):
    pass
class C(D):
    def foo(self):
        print("class C")
class A(B, C):
    pass

f = A()
f.foo()
输出：               class C
```
### Super
【只能用于新式类】从运行结果上看，普通继承和super继承是一样的。但是其实他们的内部运行机制不一样，这一点在多重继承时体现得很明显。在super机制里可以保证公共父类仅被执行一次，至于执行顺序，则是按照MRO进行。

```
class C(object):
    def __init__(self):
        print("Enter C")
        print('Leave C')
class B(C):
    def __init__(self):
        print("enter B")
        super(B, self).__init__()
        print("leave B")
b = B()
# 注解：
super(B, self).__init__()【python2中写法】等同于super().__init__()
调用父类进行初始化【__init__为父类方法】
self指代类的实例
```

## 开闭原则
* 对功能组件扩展开放、对源代码修改封闭
* 开闭原则是面向对象设计中“可复用设计”的基石

## 鸭子类型
* 鸭子类型是动态类型的一种风格。在这种风格中，一个对象的有效语义不是由继承自特定的类或实现特定的接口来决定，而是由当前的方法和属性的集合决定
* 鸭子类型通常得益于不测试方法和函数中参数的类型，而是依赖文档、清晰的代码和测试来保证正确使用
* Python的“file-like object“就是一种鸭子类型。对真正的文件对象，它有一个read()方法，返回其内容。但是，许多对象只要有read()方法，都被视为“file-like object“。许多函数接收的参数就是“file-like object“，你不一定要传入真正的文件对象，完全可以传入任何实现了read()方法的对象。

# 属性
```
class chinese(object):
    country = 'china'
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def talk(self):
        print('%s is chinese' % self.name)
```
## 类属性
* 在类中定义的变量,上例中country即为类属性
* 既可以在类中，也可以在实例中调用

## property
将类的方法变为属性

```
class Student():
    # 返回属性值
    @property
    def score(self):
        return self._score
    # 设置属性值
    @score.setter
    def score(self, value):
        if not isinstance(value, int):
            raise ValueError('score必须是整型')
        if value < 0 or value > 100:
            raise ValueError('socre必须在0和100之间')
        self._score = value
    # 删除属性值
    @score.deleter
    def score(self):
        del self._score
        # raise PermissionError
s = Student()
s.score = 60
del s.score
print(s.score)
```

## 动态属性
```
class person():
    # 限制给实例添加的属性
    __slots__ = ('name', 'age', 'score')
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def print_info(self):
        print('%s:%s' % (self.name, self.age))

p = person('he', 18)
# 1）使用setattr添加属性【getattr获取/hasattr判断/delattr删除】
# setattr(p, 'height', 16)
# 2）直接添加属性
p.score = 32
p.print_info()
```

# 方法
## 实例方法
* 在类中定义的函数，默认为实例方法
* 与实例进行绑定，函数的第一个参数是self，用于指代实例本身
    - 第一个参数如果不是self，则第一个参数会被当成self，最终解释器会报参数个数错误
* 类实例化之后才能调用此方法

## 类方法
* 使用classmethod装饰器
* 与类进行绑定，函数的第一个参数是cls
* 类和实例均可以直接调用

```
class test:
    @classmethod
    def tell(cls):
        print('ok')
test.tell()
```

## 静态方法
* 使用staticmethod装饰器
* 不与实例或类进行绑定（所以不需要第一个参数是self或cls）
* 类和实例均可以直接调用

```
class test:
    @staticmethod
    def tell(x, y):
        print(x, y)
test.tell(3, 5)
```

## 方法范例
```
import settings
import time
import hashlib

class Mysql():
    # 类实例化时绑定主机和端口
    def __init__(self, host, port):
        self.__id = self.create_id()
        self.__host = host
        self.__port = port

    def select(self):
        print('%s is selecting...' % self.__id)

    # 当类不实例化时，定义从配置文件中读取参数
    @classmethod
    def from_conf(cls):
        return cls(settings.HOST, settings.PORT)

    # 定义一个与类或实例无关的工具包
    @staticmethod
    def create_id():
        id = hashlib.md5(str(time.clock()).encode('utf-8'))
        return id.hexdigest()

con1 = Mysql('127.0.0.1', 3306)
con4 = Mysql.from_conf()
con1.select()
con4.select()
```
# 内置方法
```
class Foo:
    def __init__(self, name, age):
        self.__name = name
        self.__age = age

    def __del__(self):
        print('del---')

    def __str__(self):
        return 'name:%s age:%s' % (self.__name, self.__age)

    def __setitem__(self, key, value):
        print('setitem')
        self.__dict__[key] = value

    def __getitem__(self, item):
        print('getitem')
        return self.__dict__[item]

    def __delitem__(self, key):
        print('delitem')
        self.__dict__.pop(key)

test1 = Foo('he', 18)
test1['score'] = 35
print(test1['score'])
del test1['score']
print(test1)
```

## `__dict__`
* 包含类的名称空间【包含属性和方法的集合】

## `__init__`
* 通过`__init__`方法可以给实例绑定属性，而类的其他函数则给实例绑定方法
* 它是一个特殊的函数，在类实例化时首先执行，并且此函数不能有返回值
* self指代实例本身

## `__new__`
待续，参考[见][4]
## `__str__`
* 打印实例对象时触发此方法的执行
* 此方法只能返回字符串
* PS：在命令行直接显示变量，调用的是`__repr__`方法

## `__del__`
* 当实例对象被解释器回收或主动删除实例对象时触发此方法执行

## `__setitem__`
* 以字典形式设置（获取、删除）类的属性时触发方法的执行
* 这3种方法【setitem、getitem、delitem】和【setattr、getattr、delattr】函数具有相同功能

## `__setattr__`
** 以属性形式设置或删除类的属性 **

```
class Foo:
    def __init__(self, name, age):
        self.__name = name
        self.__age = age

    def __setattr__(self, key, value):
        print('setattr')
        self.__dict__[key] = value

    def __delattr__(self, item):
        print('delattr')
        del self.__dict__[item]

test1 = Foo('he', 18)
test1.score = 35
print(test1.score)
del test1.score
print(test1.score)
```

## `__getattr__`
** 只有在没有找到属性的情况下，才调用__getattr__,已有的属性，比如name，不会在__getattr__中查找 **

```
import datetime
class Open:
    def __init__(self, filepath, mode='r', encoding='utf-8'):
        self.filepath = filepath
        self.mode = mode
        self.encoding = encoding
        self.f = open(self.filepath, mode=self.mode, encoding=self.encoding)
    def write(self, msg):
        msg = str(datetime.datetime.now()) + '\t' + msg
        return self.f.write(msg)

    def __getattr__(self, item):
        return getattr(self.f, item)

obj = Open('a.txt', 'w')
obj.write('111\n')
obj.write('222\n')
obj.close()
```

* 以上为重写open的write方法，每次写操作均添加时间
* 对于其他方法由于没有重写，使用getattr直接获取内置open函数对应文件对象的属性和方法

## `__iter__|__next__`
* 如果一个类实现了iter和next方法，那么他就实现了迭代器协议，它就是一个迭代器
* 如下的类实现了迭代器协议【因此就可以被用于for...in循环】
* for循环通过next方法获取循环的下一个值，直到遇到StopIteration错误时退出循环

```
class Foo:
    def __init__(self):
        self.a, self.b = 0, 1

    def __iter__(self):
        return self

    def __next__(self):
        self.a, self.b = self.b, self.a + self.b
        if self.a > 1000:
            raise StopIteration()
        return self.a
for n in Foo():
    print(n)
```

## `__enter__|__exit__`
* enter和exit用于上下文管理（with）
* enter方法用于入口的处理，直接返回对象本身，当使用as后实现f=self.f效果
* exit方法用于出口的处理，exc_type为异常类型，exc_value异常的值，exc_tb为异常的追踪
* exit方法必须返回True以保证with代码块的异常不会影响其他代码的执行

```
class Open:
    def __init__(self, filepath, mode='r', encoding='utf-8'):
        self.filepath = filepath
        self.mode = mode
        self.encoding = encoding
        self.f = open(self.filepath, mode=self.mode, encoding=self.encoding)
    def __enter__(self):
        print('__enter__')
        return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        print('__exit__')
        print(exc_type)
        print(exc_val)
        print(exc_tb)
        self.f.close()
        return True
    def __getattr__(self, item):
        return getattr(self.f, item)

with Open('c.txt', 'w') as f: #f=self.f
    1/0
    f.write('abc\n')

print('ok')
```

## `__call__`
* 任何类，只要定义了call方法，就可以直接对类的实例进行调用
* 对实例进行调用就好比对一个函数进行调用，所以你完全可以把实例对象当成函数
* 因为任何自定义类都是元类type的实例，而可以直接调用自定义类就是因为元类有call方法

```
class Student:
    def __init__(self, name):
        self.name = name
    def __call__(self, *args, **kwargs):
        print('my name is %s.' % self.name)

s = Student('Michael')
s()
```

# 元类
* 元类是类的类，是用来创建类【对象】的
* 函数type实际上是一个元类。type就是python在背后用来创建所有类【包括自身】的元类。
* str是用来创建字符串对象的类，int时用来创建整数对象的类，而type就是用来创建类对象的类
* python中所有的东西都是对象，包括整数、字符串、函数、类，他们都是由一个类【元类】创建的

## 创建类
* 使用type创建类和直接写class完全一样，因为python解释器遇到class定义时，仅仅是扫描class定义的语法，然后调用type创建class
* 语法：type(name, bases, dict) -> a new type
    - class的名称，字符串形式
    - 继承的父类集合，由于python支持多重继承，所以此处为元组形式
    - 包含的属性或方法的字典

## 修改类
元类很复杂，对于非常简单的类可以不通过元类对类进行修改，可以通过其他两种两种技术实现

* Monkey patching
* class decorators

## exec函数
>读取字符串中的python代码并执行

* 表达式：exec(object[, globals[, locals]])
* 参数1：python代码形式字符串对象
* 参数2：语句执行的全局作用域【下例：复用已存在的globals获取全局变量字典】
* 参数3：语句执行的本地作用域【下例：将函数的执行结果使用字典接收】

## type与class对比
* 使用type

```
class_name = 'chinese'
class_base = (object,)
class_body = """
def __init__(self, name):
    self.name = name
def print_info(self):
    print(self.name)
"""
class_dict = {}
exec(class_body, globals(), class_dict)

chinese = type(class_name, class_base, class_dict)
p = chinese('he')
p.print_info()
```

* 使用class

```
class chinese:
    def __init__(self, name):
        self.name = name

    def print_info(self):
        print(self.name)
```

## 元类扩展阅读
* [python中type和object关系][1]
* [使用元类实现自定义ORM框架][2]
* [瞎驴讲解元类][3]

[1]:https://www.zhihu.com/question/38791962
[2]:https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014319106919344c4ef8b1e04c48778bb45796e0335839000#0
[3]: http://www.cnblogs.com/linhaifeng/articles/6204014.html
[4]: https://www.cnblogs.com/34fj/p/6358702.html
