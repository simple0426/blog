---
title: 消息队列RabbitMQ
tags:
  - rabbitmq
  - celery
  - pika
categories:
  - 消息队列
date: 2019-11-01 18:31:24
---

# 使用对比
## 与python中的queue
* 线程queue：同一个进程下进程间的交互
* 进程Queue：父子进程交互，或同属于一个进程下的多个子进程间交互
* 不同程序间的交互选择：
    - socket
    - disk
    - 消息中间件（比如rabbitmq）

## 同类产品对比
* RabbitMQ
    - Mozilla出品，由高并发语言Erlang编写
    - 基于AMQP协议实现
    - 为了保证消息的可靠性，有消息确认机制、支持事务，不支持批量操作
* Kafka
    - linkedin开源的产品，目前归属apache基金会
    - 追求吞吐量，内部采用消息的批量处理
    - 数据的存储和获取是本地磁盘顺序批量操作
* RocketMQ：阿里巴巴基于kafka，使用java重写的产品

# 基本概念
* broker：简单来说就是消息队列服务器实体
* Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列；目前支持的规则类型
    - Direct：精确匹配，完全根据key进行消息投递
    - Topic：模式匹配，对key进行模式匹配后进行消息投递（符 号"#"匹配一个或多个词，符号"*"匹配正好一个词。例如"abc.#"匹配"abc.def.ghi"，"abc.*"只匹配"abc.def"）
    - fanout：广播功能，不需要key，此时会把所有发送到该exchange的消息路由到与它绑定的Queue中
* Queue：消息队列载体，每个消息都会被投入到一个或多个队列
* Binding：绑定，它的作用就是把exchange和queue按照路由规则绑定起来
* Routing Key：路由关键字，exchange根据这个关键字进行消息投递
* vhost：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离
* producer：消息生产者，就是投递消息的程序
* consumer：消息消费者，就是接受消息的程序
* channel：消息通道，在客户端的每个连接里，都可以建立多个channel，每个channel代表一个会话任务

# 使用过程
* 客户端连接到消息队列服务器（vhost、账号、密码），打开一个channel
* 客户端声明一个exchange，并设置相关属性
* 客户端声明一个queue，并设置相关属性
* 客户端使用routing key，在exchange和queue之间建立绑定关系
* 客户端投递消息到exchange

# 消息持久化
rabbitMQ支持消息持久化，持久化包含三个部分

* exchange：声明时指定durable 
* queue：声明时指定durable
* message：在投递时指定delivery_mode =》2（1是非持久化）

只有exchange和queue都是持久化的，那么他们的binding才是持久化的；如果两者，一个持久化，一个非持久化，就不允许建立绑定。

# 安装
* ubuntu/centos仓库安装：install rabbitmq-server
* 自定义版本安装
    - 安装erlang语言
    - 下载指定格式rabbitmq-server软件包

# 管理
* rabbitmqctl add_vhost celery     添加虚拟机
* rabbitmqctl add_user ever01 Theistyle     添加用户
* `rabbitmqctl set_permissions -p celery ever01 ".*" ".*" ".*"`   对虚拟机进行用户授权【后面三个".*"代表ever01用户拥有对celery的配置、写、读全部权限】
* rabbitmqctl list_queues --vhost celery：查看队列信息及数量

## console管理
* 开启console：rabbitmq-plugins enable rabbitmq_management
* 建立管理用户：rabbitmqctl add_user full_access qaz123
* 管理用户设置权限：rabbitmqctl set_user_tags full_access administrator
* url访问：http://127.0.0.1:15672

# python客户端-pika
* 生产者producer

```
import pika, sys

# 建立链接
auth = pika.credentials.PlainCredentials('ever01', 'Theistyle')
conn_para = pika.ConnectionParameters(
    host='127.0.0.1',
    port=5672,
    virtual_host='celery',
    credentials=auth
)
conn = pika.BlockingConnection(parameters=conn_para)

# 创建channel
channel = conn.channel()
# direct模式
# channel.exchange_declare( # 创建exchange
#     exchange='direct_log',
#     exchange_type='direct',
#     durable=True)
# topic模式
# channel.exchange_declare( # 创建exchange
#     exchange='topic_log',
#     exchange_type='topic',
#     durable=True)
channel.exchange_declare( # 创建exchange
    exchange='fanout_log',
    exchange_type='fanout',
    durable=True)

# 定义发送的key和内容
keys = sys.argv[1] if len(sys.argv) > 1 else 'anonymous.info' # 定义key
message = ''.join(sys.argv[2:]) or 'Hello World!' # 定义message

# 发布消息
channel.basic_publish(
    # exchange='direct_log',
    # exchange='topic_log',
    exchange='fanout_log',
    routing_key=keys,
    body=message,
    properties=pika.BasicProperties(delivery_mode=2)
)
print("[x] Sent %r:%r" % (keys, message))
conn.close()
```

* 消费者

```
import pika, sys

# 建立链接
auth = pika.credentials.PlainCredentials('ever01', 'Theistyle')
conn_para = pika.ConnectionParameters(
    host='127.0.0.1',
    port=5672,
    virtual_host='celery',
    credentials=auth
)
conn = pika.BlockingConnection(parameters=conn_para)

# 创建channel
channel = conn.channel()
channel.queue_declare(queue='hello', durable=True) # 创建queue
# direct模式
# channel.exchange_declare( # 创建exchange
#     exchange='direct_log',
#     exchange_type='direct',
#     durable=True)
# topic模式
# channel.exchange_declare( # 创建exchange
#     exchange='topic_log',
#     exchange_type='topic',
#     durable=True)
channel.exchange_declare( # 创建exchange
    exchange='fanout_log',
    exchange_type='fanout',
    durable=True)

# 定义接收的key，并将queue和exchange绑定【建立通道接收消息】
# key = sys.argv[1] # 命令行接收routing_key
# if not key:
#     sys.stderr.write("Usage:%s key \n" % sys.argv[0])
#     sys.exit(1)
channel.queue_bind(
    # exchange='direct_log',
    # exchange='topic_log',
    exchange='fanout_log',
    queue='hello',
    # routing_key=key # fanout模式下无需绑定绑定key
)

# 消费消息
def callback(ch, method, properties, body): # 消息的具体功能
    print("[x] Received %r:%r" % (method.routing_key, body))

channel.basic_consume(
    queue='hello',
    on_message_callback=callback,
    auto_ack=True
)
print('[*] Waiting for logs.To exit press CTRL+C')
channel.start_consuming() # 开启阻塞式消费，持续接收发布的消息
```
# python客户端-celery
## celery架构
celery+rabbitmq+redis

* celery worker：任务执行单元【单独开启进程提供任务执行】
* rabbitmq：消息中间件【传递消息和定义消息队列】
* redis：存储worker执行结果

## 软件安装和配置
* 安装rabbitmq-server、redis-server
* 配置mq和redis
* 安装python模块celery、celery-with-redis

## 开启一个任务
```
from celery import Celery

app = Celery('task01', backend='redis://127.0.0.1:6379/0', broker='amqp://ever01:Theistyle@127.0.0.1:5672/celery')

@app.task
def add(x,y):
    return x + y
```

开启任务：celery -A tasks worker -l info【-A 指定一个模块，即文件名】

## 客户端测试
```
from celery import Celery
from tasks import add
import time

result = add.delay(4, 4)
print("waiting result...")
while not result.ready():
    time.sleep(2)
print("Result:", result.get())
```

测试：python3 test.py
## 信息监控
* 查看celery worker输出

```
[2019-11-01 18:15:07,708: INFO/MainProcess] Received task: tasks.add[05d41d34-9e5d-4e59-bb15-23ddc74f2e08]  
[2019-11-01 18:15:07,715: INFO/ForkPoolWorker-2] Task tasks.add[05d41d34-9e5d-4e59-bb15-23ddc74f2e08] succeeded in 0.00482514314353466s: 8
```

* 查看redis存储

```
127.0.0.1:6379> keys *
1) "celery-task-meta-05d41d34-9e5d-4e59-bb15-23ddc74f2e08"
127.0.0.1:6379> type celery-task-meta-05d41d34-9e5d-4e59-bb15-23ddc74f2e08
string
127.0.0.1:6379> get celery-task-meta-05d41d34-9e5d-4e59-bb15-23ddc74f2e08
"{\"date_done\": \"2019-11-01T10:15:07.710361\", \"task_id\": \"05d41d34-9e5d-4e59-bb15-23ddc74f2e08\", \"status\": \"SUCCESS\", \"result\": 8, \"children\": [], \"traceback\": null}"
```
