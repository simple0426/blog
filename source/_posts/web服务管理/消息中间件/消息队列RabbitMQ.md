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
* 线程queue：同一个进程下不同线程间的交互
* 进程Queue：父子进程交互，或同属于一个进程下的多个子进程间交互
* 消息中间件(rabbitmq)：不同程序间的交互

## 同类产品对比
* RabbitMQ
    - 由高并发语言Erlang编写
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
    - Topic：模式匹配，对key进行模式匹配后进行消息投递（符 号"#"匹配一个或多个词，符号"\*"匹配正好一个词。例如"abc.#"匹配"abc.def.ghi"，"abc.\*"只匹配"abc.def"）
    - fanout：广播功能，不需要key，此时会把所有发送到该exchange的消息路由到与它绑定的Queue中
* Queue：消息队列载体，每个消息都会被投入到一个或多个队列
* Binding：绑定，它的作用就是把exchange和queue按照路由规则绑定起来
* Routing Key：路由关键字，exchange根据这个关键字进行消息投递
* vhost：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离
* producer：消息生产者，就是投递消息的程序
* consumer：消息消费者，就是接受消息的程序
* channel：消息通道，在客户端的每个连接里，都可以建立多个channel，每个channel代表一个会话任务

# 高可用安装

高可用架构，需要同时使用cluster和队列的镜像模式：

* [cluster](https://www.rabbitmq.com/clustering.html)：在所有节点同步对broker的管理和状态信息，因此可以连接任意节点对集群操作
* [镜像模式](https://www.rabbitmq.com/ha.html)：使用镜像模式对exchange、queue做多副本管理，消除默认情况下exchange、queue单节点存储的危险

集群共使用3个节点【192.168.31.231、192.168.31.232、192.168.31.233】

## 集群操作

> 默认在3个节点操作

* 节点hosts配置

  ```
  192.168.31.231 rabbitmq-1
  192.168.31.232 rabbitmq-2
  192.168.31.233 rabbitmq-3
  ```

* 安装rabbitmq软件

  ```
  yum install -y rabbitmq-server
  systemctl start rabbitmq-server # 单机模式启动，产生cookie信息
  ```

* 统一节点cookie【既用于客户端认证，也用于集群节点间认证】

  ```
  # 停止rabbitmq服务
  rabbitmqctl stop
  
  # 将rabbitmq-1的cookie复制到rabbitmq-2、rabbitmq-3
  scp /var/lib/rabbitmq/.erlang.cookie rabbitmq-2:/var/lib/rabbitmq/.erlang.cookie
  scp /var/lib/rabbitmq/.erlang.cookie rabbitmq-3:/var/lib/rabbitmq/.erlang.cookie
  
  # 更改cookie权限
  chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
  chmod 400 /var/lib/rabbitmq/.erlang.cookie
  ```

* 独立方式启动rabbitmq

  ```
  rabbitmq-server -detached
  rabbitmqctl cluster_status # 查看状态
  ```

* 创建集群【在节点rabbitmq-2、rabbitmq-3上，将节点加入rabbitmq-1创建的集群】

  ```
  rabbitmqctl stop_app
  rabbitmqctl reset
  rabbitmqctl join_cluster rabbit@rabbitmq-1
  rabbitmqctl start_app
  rabbitmqctl cluster_status # 查看状态
  ```

## 镜像操作

设置：所有queue、exchange自动在所有节点进行镜像备份

```
rabbitmqctl set_policy ha-all ".*" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
```

更详细的镜像模式介绍如下：

* 镜像操作类型

  * exactly：集群中共保留几个副本【主+镜像数】
  * all：在集群的其他所有节点都保留副本
  * nodes：在集群指定的节点保留副本

* 操作范例

  * exactly：名字以two开始的queue，自动在集群中保留2个副本

    ```
    rabbitmqctl set_policy ha-two "^two\." \
      '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
    ```

  * all：名字以ha开始的queue，在集群其他所有节点都保留副本

    ```
    rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
    ```

  * nodes：名字以nodes开头的queue，在rabbit@nodeA、rabbit@nodeB保留副本

    ```
    rabbitmqctl set_policy ha-nodes "^nodes\." \
    '{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}'
    ```

## 客户端使用

* 如果客户端支持连接多个rabbitmq实例（如pika），可以直接使用
* 如果客户端不支持连接多个rabbitmq实例（如celery），则需要使用nginx对多个mq实例进行负载均衡代理

# 命令行操作

* 虚拟机管理

  ```
  add_vhost <vhostpath>
  delete_vhost <vhostpath>
  list_vhosts [<vhostinfoitem> ...]
  set_permissions [-p <vhostpath>] <user> <conf> <write> <read>
  clear_permissions [-p <vhostpath>] <username>
  list_permissions [-p <vhostpath>]
  list_user_permissions <username>
  
  # 范例
  添加：rabbitmqctl add_vhost celery
  列表：rabbitmqctl list_vhosts
  授权：rabbitmqctl set_permissions -p celery ever01 ".*" ".*" ".*"
  ```

* 用户管理

  ```
  add_user <username> <password>
  delete_user <username>
  change_password <username> <newpassword>
  clear_password <username>
  set_user_tags <username> <tag> ...
  list_users
  
  # 范例
  添加用户： rabbitmqctl add_user ever01 Theistyle     
  开启用户web功能：rabbitmqctl set_user_tags ever01 administrator
  ```

* 镜像模式设置

  ```
  set_policy [-p <vhostpath>] [--priority <priority>] [--apply-to <apply-to>] 
  <name> <pattern>  <definition>
  clear_policy [-p <vhostpath>] <name>
  list_policies [-p <vhostpath>]
  ```

* 信息统计

  ```
  list_queues [-p <vhostpath>] [<queueinfoitem> ...]
  list_exchanges [-p <vhostpath>] [<exchangeinfoitem> ...]
  list_bindings [-p <vhostpath>] [<bindinginfoitem> ...]
  list_connections [<connectioninfoitem> ...]
  list_channels [<channelinfoitem> ...]
  list_consumers [-p <vhostpath>]
  status
  environment
  report
  ```

# web管理功能

* 开启web功能：
  * 启用功能：rabbitmq-plugins enable rabbitmq_management
  * 重启rabbitmq：systemctl restart rabbitmq-server
  * web访问【默认guest/guest】：http://127.0.0.1:15672
* 自定义web管理用户：
  * 建立用户：rabbitmqctl add_user full_access qaz123
  * 开启用户管理员功能：rabbitmqctl set_user_tags full_access administrator

# 持久化设置

rabbitMQ支持消息持久化，持久化包含三个部分：  
* exchange：声明时指定durable 
* queue：声明时指定durable
* message：在投递时指定delivery_mode =》2（1是非持久化）

只有exchange和queue都是持久化的，那么他们的binding才是持久化的；如果两者之一不能持久化，就不允许建立绑定。

# 使用方式
* 客户端连接到消息队列服务器（vhost、账号、密码），打开一个channel
* 客户端声明一个exchange，并设置相关属性
* 客户端声明一个queue，并设置相关属性
* 客户端使用routing key，在exchange和queue之间建立绑定关系
* 客户端投递消息到exchange

# python客户端-pika
* 生产者producer

```
import pika, sys

# 建立链接
auth = pika.credentials.PlainCredentials('ever01', 'Theistyle')
# 连接单个mq实例
# conn_para = pika.ConnectionParameters(
#     host='127.0.0.1',
#     port=5672,
#     virtual_host='celery',
#     credentials=auth
# )

# 连接多个mq实例
conn_para = (
    pika.ConnectionParameters(
        host='192.168.31.231',
        port=5672,
        virtual_host='celery',
        credentials=auth,
        connection_attempts=3,
        retry_delay=1
    ),
    pika.ConnectionParameters(
        host='192.168.31.232',
        port=5672,
        virtual_host='celery',
        credentials=auth,
        connection_attempts=3,
        retry_delay=1
    ),
    pika.ConnectionParameters(
        host='192.168.31.233',
        port=5672,
        virtual_host='celery',
        credentials=auth,
        connection_attempts=3,
        retry_delay=1
    ),
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
# 连接单个mq实例
# conn_para = pika.ConnectionParameters(
#     host='127.0.0.1',
#     port=5672,
#     virtual_host='celery',
#     credentials=auth
# )

# 连接多个mq实例
conn_para = (
    pika.ConnectionParameters(
        host='192.168.31.231',
        port=5672,
        virtual_host='celery',
        credentials=auth,
        connection_attempts=3,
        retry_delay=1
    ),
    pika.ConnectionParameters(
        host='192.168.31.232',
        port=5672,
        virtual_host='celery',
        credentials=auth,
        connection_attempts=3,
        retry_delay=1
    ),
    pika.ConnectionParameters(
        host='192.168.31.233',
        port=5672,
        virtual_host='celery',
        credentials=auth,
        connection_attempts=3,
        retry_delay=1
    ),
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

## 开启一个任务调度进程
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
