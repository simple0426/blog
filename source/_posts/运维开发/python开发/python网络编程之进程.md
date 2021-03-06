---
title: python网络编程之进程
date: 2018-03-02 14:25:00
tags: ['进程', '线程', 'IPC', '协程', 'gevent']
categories: python
---
# 进程与线程
## 概念
* 进程是资源集合，至少包含一个线程；线程是程序运行的最小单位
* 一个进程的多个线程共享内存空间
* 无论多进程或多线程，数量增加到一定程度，任务开销就会急剧上升，效率就会下降
* 计算密集型任务，更多使用cpu，注重代码的执行效率，更适合使用C语言开发；而如web应用等io密集型【磁盘和网络】，瓶颈不在cpu和内存，所以适合使用python这样代码少的语言提高开发效率
* 利用操作系统对异步io的支持，可以用单进程单线程来执行多任务，这种模型即为事件驱动模型，对应到python即为协程
* 要实现多任务，通常采用master-worker模式，master负责分配任务，worker负责执行任务

## 优缺点
|   -    |                            优点                           |                            缺点                           |                                应用范例                                |
|--------|-----------------------------------------------------------|-----------------------------------------------------------|------------------------------------------------------------------------|
| 多进程 | 稳定性高,一个进程的崩溃不会影响到其他进程                 | 创建进程的开销大,Unix下fork还好,windows下特别明显         | Apache默认采用进程模型                                                 |
| 多线程 | 通常线程模式比进程模式执行快,windows下iis就采用多线程模式 | 但是由于多线程共享内存,一个线程的崩溃会导致整个进程的崩溃 | 现在apache和iis都出现了多进程多线程混合模式,在保证稳定性的同时提高效率 |

# 进程
## Process方法
* Process.name：查看子进程名称
* Process.pid：子进程pid
* Process.is_alive()：进程是否存活
* Process.terminate()：终止进程
* Process.daemon：当主进程结束时子进程也跟着结束，子进程的daemon属性必须先于start设置
* Process.start：开启子进程
* Process.join：父进程等待子进程结束【当不设置join时，父进程不等待子进程结束】

```python
from multiprocessing import Process
import os, random, time

def run_proc(name):
    print('子进程 %s(%s)开始' % (name, os.getpid()))
    time.sleep(random.randint(3, 6))
    print('子进程 %s(%s)结束' % (name, os.getpid()))


if __name__ == '__main__':
    print('父进程%s开始' % os.getpid())
    # Process产生用于执行函数的子进程
    p1 = Process(target=run_proc, args=('test1',))
    p2 = Process(target=run_proc, args=('test2',))
    # 开启子进程,此时p1与p2并行执行
    p1.start()
    p2.start()
    # 等待子进程结束
    p1.join()
    p2.join()
    print('父进程%s结束' % os.getpid())

# 假如程序为如下顺序，则p1与p2串行执行
# 子程序1
p1.start()
p1.join()  
# 子程序2
p2.start()
p2.join()
```

# 进程池
## Pool方法
```python
from multiprocessing import Pool
import os, time, random

def long_time_task(name):
    print('运行任务 %s(%s)' % (name, os.getpid()))
    start = time.time()
    time.sleep(random.random() * 10)
    end = time.time()
    print('任务 %s 运行 %0.2f seconds' % (name, (end - start)))

if __name__ == '__main__':
    print('主进程 %s 开始' % os.getpid())
    # Pool默认产生和cpu核数相等的子进程
    p = Pool(5)
    for i in range(5):
    # apply_async用于产生并行子进程，一个进程结束又会有新的进程进入，维持进程总数不变
        p.apply_async(long_time_task, args=(i,))
    # 关闭进程池
    p.close()
    # 主进程阻塞，等待子进程的退出， join方法要在close或terminate之后使用。
    p.join()
    print('主进程 %s 结束' % os.getpid())
```

## Pool方法回调
```python
# 进程池开启n个进程，每个进程获取一个随机值并取这个随机值的平方
from multiprocessing import Pool
import random

def random_int(n):
    res = random.randint(1, 2*n)
    print('res is %s' % res)
    return res

def square(n):
    print(n**2)

if __name__ == '__main__':
    p = Pool(3)
    for i in range(1, 6):
        print('i is %d' % i)
        p.apply_async(random_int, args=(i,), callback=square)
    p.close()
    p.join()
```

# 线程
* 任何进程默认就会启动一个线程，我们把该线程称为主线程，主线程又可以启动新的线程 
* 多线程和多进程最大的不同在于，多进程中，同一个变量，各自有一份拷贝存在于每个进程中，互不影响，而多线程中，所有变量都由所有线程共享，所以，任何一个变量都可以被任何一个线程修改
* 为了保证一个变量不被多个线程同时修改，可以在一个线程执行前加锁，执行完后释放锁，然后其他线程再执行操作【 由于锁只有一个，无论多少线程，同一时刻最多只有一个线程持有该锁，所以，不会造成修改的冲突】
* GIL【global iterpretor lock】 Python的线程虽然是真正的线程，但解释器执行代码时，有一个GIL锁：Global Interpreter Lock，任何Python线程执行前，必须先获得GIL锁，然后，每执行100条字节码，解释器就自动释放GIL锁，让别的线程有机会执行。这个GIL 全局锁实际上把所有线程的执行代码都给上了锁，所以，多线程在Python中只能交替执行，即使100个线程跑在100核CPU上，也只能用到1个核。

## 多线程同步
由于CPython的python解释器在单线程模式下执行，所以导致python的多线程在很多的时候并不能很好地发挥多核cpu的资源。  
大部分情况都推荐使用多进程【多个Python进程有各自独立的GIL锁，互不影响。】。python的多线程的同步与其他语言基本相同，主要包含：

* [Lock&RLock](#Lock&RLock) ：用来确保多线程多共享资源的访问。
* [Semaphore](#Semaphore)： 用来确保一定资源多线程访问时的上限，例如资源池
* [Event](#Event)：是最简单的线程间通信的方式，一个线程可以发送信号，其他的线程接收到信号后执行操作。 

### Lock&RLock
* Lock对象的状态可以为locked和unlocked，使用acquire()设置为locked状态；使用release()设置为unlocked状态。
    - 如果当前的状态为unlocked，则acquire()会将状态改为locked然后立即返回。当状态为locked的时候，acquire()将被阻塞直到另一个线程中调用release()来将状态改为unlocked，然后acquire()才可以再次将状态置为locked。
    - Lock.acquire(blocking=True, timeout=-1),blocking参数表示是否阻塞当前线程等待，timeout表示阻塞时的等待时间 。如果成功地获得lock，则acquire()函数返回True，否则返回False，timeout超时时如果还没有获得lock仍然返回False。
* Rlock在acquire状态时，依然可以再次acquire，只要acquire与release成对出现即可，可以递归使用
    - RLock与Lock的区别是：RLock中除了状态locked和unlocked外还记录了当前lock的owner和递归层数
*  范例Lock【确保只有一个线程可以访问共享资源】

```python
import time, threading, random

balance = 0
lock = threading.Lock()

def change_it(n):
    global balance
    balance = balance + n
    print(balance)
    balance = balance - n
    print(balance)
    time.sleep(random.random() * 2)

def run_thread(n):
    for i in range(10):
        lock.acquire()
        try:
            change_it(n)
        finally:
            lock.release()

t1 = threading.Thread(target=run_thread, args=(5,))
t2 = threading.Thread(target=run_thread, args=(8,))
t1.start()
t2.start()
t1.join()
t2.join()
print('final balance is %d' % balance)
```

* 范例RLock

```python
import threading, time

lock1 = threading.RLock()
x = 0

def increase_5(lock, loop):
    global x
    for i in range(0, loop):
        lock.acquire()
        x += 5
        print('increase x is %d' % x)
        lock.release()

def decrease_4(lock, loop):
    global x
    for i in range(0, loop):
        lock.acquire()
        increase_5(lock, loop)
        x -= 4
        print('decrease x is %d' % x)
        lock.release()

t1 = threading.Thread(target=decrease_4, args=(lock1, 4))
t1.start()
t1.join()
print('x is %d' % x)
```
### Semaphore
* Semaphore管理一个内置的计数器，每当调用acquire()时内置计数器-1；调用release() 时内置计数器+1。
* 计数器不能小于0；当计数器为0时，acquire()将阻塞线程直到其他线程调用release()。 
* 范例【同时只有2个线程可以获得semaphore,即可以限制最大连接数为2】

```python
import threading, time

semaphore = threading.Semaphore(2)

def func(name):
    if semaphore.acquire():
        print( '%s get semaphore' % name)
        time.sleep(2)
        semaphore.release()
        print('%s release semaphore' % name)

for i in range(4):
    t1 = threading.Thread(target=func, args=(i,))
    t1.start()
```

### Event
* Event内部包含了一个标志位，初始的时候为false。
* 可以使用使用set()来将其设置为true，或者使用clear()将其重新设置为false。
* 可以使用is_set()来检查标志位的状态
* wait(timeout=None)，用来阻塞当前线程，直到event的内部标志位被设置为true或者timeout超时。
* 范例【线程间相互通信】

```python
import threading, time

event = threading.Event()

def mysql_status(e):
    print('mysql is starting')
    time.sleep(5)
    e.set()
    print('mysql is ready...')

def mysql_conn(e):
    while True:
        if e.is_set():
            print('%s connected to mysql' % threading.current_thread().getName())
            break
        else:
            e.wait(2)
            print('%s connecting to mysql' % threading.current_thread().getName())

t1 = threading.Thread(target=mysql_status, args=(event,))
t2 = threading.Thread(target=mysql_conn, args=(event,))
t3 = threading.Thread(target=mysql_conn, args=(event,))
t1.start()
t2.start()
t3.start()
```

## 线程隔离Local
* 全局变量local_school是一个threadlocal对象，每个线程都可以对它的student属性操作，而互不影响
* threadlocal常用于为每个线程绑定一个数据库连接、http请求、用户身份信息等

```python
import threading

local_school = threading.local()

def process_thread(name):
    local_school.student = name
    print('Hello, %s (in %s)' % (local_school.student, threading.current_thread().name))

t1 = threading.Thread(target=process_thread, args=('Alice',), name='Thread-A')
t2 = threading.Thread(target=process_thread, args=('Bob',), name='Thread-B')
t1.start()
t2.start()
t1.join()
t2.join()
```

# 协程
* 对于多线程应用，cpu通过切片的方式来切换线程间的执行，线程切换需要耗时【保存上下文】
* 协程则只使用一个线程，在一个线程中规定某个代码块的执行顺序

## 生成器实现
>生产者和消费者在同一进程中的使用范例

```python
import time

def consumer():
    r = ''
    while True:
        n = yield r
        if not n:
            return
        print('[消费者] 正在消费 %s...' % n)
        time.sleep(1)
        r = '200 OK'

def produce(c):
    next(c)
    n = 0
    while n < 5:
        n = n + 1
        print('[生产者] 正在生产 %s...' % n)
        r = c.send(n)
        print('[生产者] 消费返回：%s' % r)
    c.close()

if __name__ == '__main__':
    c = consumer()
    produce(c)

# 范例详解
生产者中：
n = 1
生产1
消费者中：
向yield传值1后，n为1
消费1
消费者返回r 为‘200 ok’
生产者中：
接受消费者的返回值r为‘200 ok’
```

## gevent实现
* gevent是第三方库，通过greenlet实现协程
* 当一个greenlet遇到IO操作时，比如访问网络，就会自动切换到其他的greenlet，等待IO操作完成，再在适当的时候切换回来继续执行
* 由于IO操作非常耗时，经常处于等待状态，有了gevent为我们自动切换协程，就保证总有greenlet在运行，而不是等待IO
* 由于gevent是基于IO切换的协程，所以最神奇的是，我们编写的Web App代码，不需要引入gevent的包，也不需要改任何代码，仅仅在部署的时候，用一个支持gevent的WSGI服务器，立刻就获得了数倍的性能提升。

```python
# 创建一个协程对象g1，spawn括号内第一个参数是函数名，如eat，后面可以有多个参数，可以是位置实参或关键字实参，都是传给函数eat的
g1=gevent.spawn(func,1,,2,3,x=4,y=5)
g2=gevent.spawn(func2)
# 等待g1结束
g1.join() 
# 等待g2结束
g2.join() 
# 或者上述两步合作一步：gevent.joinall([g1,g2])
# 拿到func1的返回值
g1.value
```

## gevent范例
* 基本使用

```python
from gevent import monkey;monkey.patch_all()
import gevent
import time

def eat(name):
    print('%s eat 1' % name)
    # 模拟IO阻塞
    ''' 
    在gevent模块里面要用gevent.sleep(2)表示等待的时间
    然而我们经常用time.sleep()用习惯了，那么有些人就想着
    可以用time.sleep()，那么也不是不可以。要想用，就得在
    最上面导入from gevent import monkey;monkey.patch_all()这句话
    如果不导入直接用time.sleep()，就实现不了单线程并发的效果了
    '''
    time.sleep(2)
    print('%s eat 2' % name)
    return 'eat'

def play(name):
    print('%s play 1' % name)
    time.sleep(1)
    print('%s play 2' % name)
    return 'play'

start = time.time()
g1 = gevent.spawn(eat, 'egon')
g2 = gevent.spawn(play, 'alex')
gevent.joinall([g1, g2])
print('主', time.time() - start)
print(g1.value)
print(g2.value)
```

* 爬虫的使用

```python
import time

def get_page(url):
    print('get :%s' % url)
    response = requests.get(url)
    if response.status_code == 200:
        print('%d bytes received from:%s' % (len(response.text), url))

start = time.time()
gevent.joinall([
    gevent.spawn(get_page, 'http://www.baidu.com'),
    gevent.spawn(get_page, 'https://www.yahoo.com'),
    gevent.spawn(get_page, 'https://github.com'),
])
stop = time.time()
print('run time is %s' % (stop - start))
```

# 参考
* [多线程同步][3]
[3]:https://www.cnblogs.com/itech/archive/2012/01/05/2312831.html
