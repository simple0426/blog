---
title: IPC与MQ
date: 2018-02-28 13:54:15
tags: ['MQ']
categories: ['python']
---
# IPC通信
进程间的通讯IPC (Inter-process communication))：指至少两个进程或线程间传送数据或信号的一些技术或方法。
## 原因
* 数据传输：一个进程需要将它的数据发送给另一个进程，发送的数据量在一个字节到几M字节之间
* 共享数据：多个进程想要操作共享数据，一个进程对共享数据的修改，别的进程应该立刻看到。
* 通知事件：一个进程需要向另一个或一组进程发送消息，通知它（它们）发生了某种事件（如进程终止时要通知父进程）。
* 资源共享：多个进程之间共享同样的资源。为了作到这一点，需要内核提供锁和同步机制。
* 进程控制：有些进程希望完全控制另一个进程的执行（如Debug进程），此时控制进程希望能够拦截另一个进程的所有陷入和异常，并能够及时知道它的状态改变。

## 类型
* 本地过程调用(LPC)LPC用在多任务操作系统中，使得同时运行的任务能互相会话。这些任务共享内存空间使任务同步和互相发送信息。
* 远程过程调用(RPC)RPC类似于LPC，只是在网上工作。RPC开始是出现在Sun微系统公司和HP公司的运行UNIX操作系统的计算机中。


## 方式
* 管道(pipe)：管道可用于具有亲缘关系的进程间的通信，是一种半双工的方式，数据只能单向流动，允许一个进程和另一个与它有共同祖先的进程之间进行通信。
* 命名管道(named pipe)：命名管道克服了管道没有名字的限制，同时除了具有管道的功能外（也是半双工），它还允许无亲缘关系进程间的通信。命名管道在文件系统中有对应的文件名。命名管道通过命令mkfifo或系统调用mkfifo来创建。
* 信号（signal）：信号是比较复杂的通信方式，用于通知接收进程有某种事件发生了，除了进程间通信外，进程还可以发送信号给进程本身；linux除了支持Unix早期信号语义函数sigal外，还支持语义符合Posix.1标准的信号函数sigaction（实际上，该函数是基于BSD的，BSD为了实现可靠信号机制，又能够统一对外接口，用sigaction函数重新实现了signal函数）。
* 消息队列：消息队列是消息的链接表，包括Posix消息队列system V消息队列。有足够权限的进程可以向队列中添加消息，被赋予读权限的进程则可以读走队列中的消息。消息队列克服了信号承载信息量少，管道只能承载无格式字节流以及缓冲区大小受限等缺点
* 共享内存：使得多个进程可以访问同一块内存空间，是最快的可用IPC形式。是针对其他通信机制运行效率较低而设计的。往往与其它通信机制，如信号量结合使用，来达到进程间的同步及互斥。
* 内存映射：内存映射允许任何多个进程间通信，每一个使用该机制的进程通过把一个共享的文件映射到自己的进程地址空间来实现它。
* 信号量（semaphore）：主要作为进程间以及同一进程不同线程之间的同步手段。
* 套接字（Socket）：更为一般的进程间通信机制，可用于不同机器之间的进程间通信。起初是由Unix系统的BSD分支开发出来的，但现在一般可以移植到其它类Unix系统上：Linux和System V的变种都支持套接字。

# Message Queue
* 提到消息中间件，那么首先就必须理解一下所谓的Message Queue。
* 在平常的开发中，应用开发人员完全可以通过发送和接受消息的方式来方便的与应用程序进行可靠的通信，并且消息的处理为我们提供了方便的消息传递和许多业务处理的可靠的防止故障的方法。
* 但消息传递与传统的应用程序交互又有区别？最明显的区别就是实时性了。Message Queue不适合实时性要求比较高的场景，因为Message Queue通过异步的方式与server端进行交互，不用担心server端的长时间处理过程。
* 举个例子，平常的应用程序之间的调用大部分都是通过暴露接口的形式进行相互的调用，一旦业务复杂起来，接口之间的管理也会很麻烦。这时候如果通过Message Queue的方法的话，只要在需要的时候把消息发送到Queue Manage就可以，这时候Message Queue就成了嫁接各个系统之间的桥梁。
* 上面也提到了实时性的问题，比如一个生成报表的功能，为了给用户更好的体验度，我们不可能让用户等待很长的时间，这时可以把对报表的处理使用MQ，客户端只需要把必要的报表请求和一些必要的报表条件放置到Queue中处理，处理好后，再给用户发送一个消息即可。

#  消息
两台计算机之间传送的数据单位，例如字符串、文本等
# 消息队列
消息的容器，用于在消息传递的过程中保存消息的容器，充当消息源和目标之间的中间桥梁。队列的主要目的就在于提供路由保证消息的传递。
# 消息中间件
上面对消息队列有了一定的了解，那么消息中间件利用高效可靠的消息传递机制进行平台无关的数据交流，并基于数据通信来进行分布式系统的集成。通过提供消息传递和消息排队模型，可以再分布式环境下扩展进程间的通信。  
那么对于消息中间件，常见的角色大致也就有Producer（生产者）、Consumer（消费者）、Broker（中转角色），有这么几个主要的角色，那么消息中间件能为我们带来那些功能呢？
## Message Priority
Producer把消息发送给Broker来存储，那么我们就可以在消息队列中对我们的消息来进行排序，实现不同的优先级。从而满足我们复杂的业务需求。
## Message Order
消息排序，有的消息的处理是需要按照一定的顺序进行处理的，比如用户的创建订单、订单付款、订单完成。那么对于消费者也需要按照这个流程来消费，否则就没有意义了。
## Message Filter
在消息队列中，也可以对我们的消息进行过滤，比如按照消息类型等条件来过滤
## Message Persistence
消息的持久化，一般有以下几种方式

* 持久化到数据库，比如MySQL
* 持久化到KV存储，比如Redis
* 文件形式持久化

消息的持久化，防止了系统挂掉后，仍然能够从之前备份中恢复出来。
## Broker的Buffer满了
我们知道Broker是用来存储需要处理的消息，如果消息过多，导致Buffer满了怎么办？  
这时候就会采取一定的策略来丢弃已有的消息。  
## 事务的支持
正如上面所谈到的订单的操作，因此消息中间件中也会提供对分布式事务的支持。
## 定时消息
 在实际应用中，有时也会需要定时消费的功能，因此中间件中，也会对消息进行排序，然后实现定时发送或者消费消息的业务需求。
## 消息重试
考虑一下这个问题，如果消息消费失败后，怎么办，是等待处理这个消息呢？还是让消费者再次消费一次呢？通常情况下，采取后者的形式，因为大多数情况下，消费失败的原因在于该消息本身的原因，如果再次消费这个消息的话，还是会出现失败的情况，因此通常采取再次发送消息的方式。
## 回溯消费
什么是回溯消费呢？对于已经消费成功的消息，是不是在Broker中就丢弃该消息呢？显而易见是不可能的，因此需要中间件对该功能支持，支持已经消费的信息进行时间段内的存储，等待某一刻内该消息会被重新消费的可能。

# 范例
## 队列-Queue
* q=Queue(3) 定义队列及其大小
* q.put() 向队列放置元素
* q.full() 队列是否已满
* q.empty() 队列是否为空
* q.put_nowait() 不能放置元素直接报异常
* q.put('4', block=False) 不能放则报异常【默认block为True】
* q.put('4', timeout=2) 放置时最多等待2s，超时则报异常

## 队列-JoinableQueue
>和Queue方法类似，但是JoinableQueue消费者消费完时必须向生产者发送信号，生产者必须等待消费者消费完成时发送的信号

* JoinableQueue.task_done()：向生产者发送已消费的信号
* JoinableQueue.join()：等待消费者发送已消费的信号

```python
from multiprocessing import Process, Queue, Pool, Manager, JoinableQueue
import os, time, random

def write(seq, q):
    print('Process to write: %s' % os.getpid())
    for value in seq:
        time.sleep(random.randint(1, 3))
        print('put %s to queue...' % value)
        q.put(value)
    # (JoinableQueue下）等待消费者发送已消费的信号
    q.join()


def read(q):
    print('Process to read: %s' % os.getpid())
    while True:
        time.sleep(random.randint(1, 3))
        value = q.get(True)
        # （JoinableQueue下）向生产者发送已消费的信号
        q.task_done()
        print('get %s from queue' % value)

# 进程模式下队列使用【使用原生的queue】
if __name__ == '__main__':
    #q = Queue()
    q = JoinableQueue()
    data = list(range(5))
    pw = Process(target=write, args=(data, q))
    pr = Process(target=read, args=(q,))
    # 当主进程结束时子进程也跟着结束，子进程的daemon属性必须先于start设置
    pr.daemon = True
    pw.start()
    pr.start()
    pw.join()
    # pr.join()

# 进程池模式下队列使用【使用Manager的queue】
#if __name__ == '__main__':
#    manager = Manager()
#    q = manager.Queue()
#    p = Pool()
#    p.apply_async(write, args=(q,))
#    p.apply_async(read, args=(q,))
#    p.close()
#    p.join()
```

## 共享内存空间-Manager
```python
from multiprocessing import Process, Manager
import os

def work(l, d):
    l.append(os.getpid())
    d[os.getpid()] = os.getpid()


if __name__ == '__main__':
    m = Manager()
    # 由manager创建的列表和字典对象可以被所有进程操作
    l = m.list(['init'])
    d = m.dict({'name': 'egon'})
    p_l = []
    for i in range(5):
        p = Process(target=work, args=(l, d))
        p_l.append(p)
        p.start()
    for p in p_l:
        p.join()
    print(d, l)
```
