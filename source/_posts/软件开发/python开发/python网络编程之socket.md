---
title: python网络编程之socket
tags:
  - socket
  - 粘包
categories:
  - python
date: 2018-03-12 11:47:40
---

# 数据流处理
## 服务端
1. 绑定端口
2. 监听端口【udp无】
3. 接受tcp连接【udp无】
4. 处理连接
    1. 接收客户端数据
    2. 处理数据
    3. 向客户端返回数据
5. 关闭连接【udp无】

## 客户端
1. 建立连接【udp无】
2. 发送数据
3. 接收数据
4. 关闭连接【udp无】

# tcp编程
## 服务端
```python
from socket import *
import time, threading
# 建立socket
s = socket(AF_INET, SOCK_STREAM)
# 绑定端口
s.bind(('127.0.0.1', 9999))
# 监听端口
s.listen(5)
print('等待连接。。。')

# 处理连接
def tcplink(sock, addr):
    print('从%s:%s接受新连接' % addr)
    # 连接建立时首先向客户端发送信息
    sock.send('Welcome!'.encode('utf-8'))
    while True:
        # 接受数据
        data = sock.recv(1024)
        time.sleep(1)
        # 如果数据为空或收到exit关键字子则不再接受数据
        if not data or data.decode('utf-8') == 'exit':
            break
        sock.send(('你好,%s' % data.decode('utf-8')).encode('utf-8'))
    # 关闭连接
    sock.close()
    print('来自%s:%s的连接已经关闭' % addr)

while True:
    sock, addr = s.accept()
    t = threading.Thread(target=tcplink, args=(sock, addr))
    t.start()
```

## 客户端
```python
from socket import *

s = socket(AF_INET, SOCK_STREAM)
s.connect(('127.0.0.1', 9999))
print(s.recv(1024).decode('utf-8'))
for data in ['Michael', 'Tracy', 'Sarah']:
    s.send(data.encode('utf-8'))
    print(s.recv(1024).decode('utf-8'))

s.send('exit'.encode('utf-8'))
s.close()
```

# udp编程
## 服务端
```python
from socket import *
s = socket(AF_INET, SOCK_DGRAM)
s.bind(('127.0.0.1', 9999))
print('在udp端口9999开启服务')
while True:
    data, addr = s.recvfrom(1024)
    print('从%s:%s接受新连接' % addr)
    s.sendto(('你好,%s' % data.decode('utf-8')).encode('utf-8'), addr)
```

## 客户端
```python
from socket import *
s = socket(AF_INET, SOCK_DGRAM)
for data in ['Michael', 'Tracy', 'Sarah']:
    s.sendto(data.encode('utf-8'), ('127.0.0.1', 9999))
    print(s.recv(1024).decode('utf-8'))
s.close()
```

# socketserver编程
* SocketServer模块简化了编写网络服务程序的任务。同时SocketServer模块也 是Python标准库中很多服务器框架的基础。
* socketserver中包含了两种类，
    -  一种为服务类（server class），它提供了许多方法：像绑定，监听，运行…… （也就是建立连接的过程）
    -  一种为请求处理类（request handle class），则专注于如何处理用户所发送的数据（也就是事务逻辑）。
* 范例【服务端】

```python
import socketserver
import time

class Myhandler(socketserver.BaseRequestHandler):
    # 数据处理
    def handle(self):
        # while True:
        res = self.request.recv(1024)
        if res:
            msg = res.decode('utf-8')
            print('client:%s msg:%s' % (self.client_address, res))

if __name__ == '__main__':
    # 连接循环
    s = socketserver.ThreadingTCPServer(('127.0.0.1', 9999), Myhandler)
    s.serve_forever()
```

# 粘包与分包
>[参考][2]
## 粘包
* 简单描述：发送方发送两个字符串”hello”+”world”，接收方却一次性接收到了”helloworld”。
* 产生原因：有时候，TCP为了提高网络的利用率，会使用一个叫做Nagle的算法。该算法是指，发送端即使有要发送的数据，如果很少的话，会延迟发送。如果应用层给TCP传送数据很快的话，就会把两个应用层数据包“粘”在一起，TCP最后只发一个TCP数据包给接收端。

## 分包
* 简单描述：发送方发送字符串”helloworld”，接收方却接收到了两个字符串”hello”和”world”。
* 产生原因：由于[MTU和MSS][1]的长度限制

# 范例
## 思路解读【粘包处理】
* 设置数据的属性信息比如版本、md5、长度为报头信息
* 【发送】将报头信息的长度发送给接收方【struct】
    - struct用于python【str，int】和c【struct】之间的数据类型转换
    - 此处struct将要发送的【数据长度】打包成固定长度的bytes对象发送给接收方
* 【接收方】接收固定长度的数据【struct】，解析出报头长度
* 【发送】发送报头信息
* 【接收方】根据报头长度接收报头信息
* 【发送】发送数据
* 【接收方】根据报头信息接收数据信息

## 模拟远程执行命令服务端
```python
from socket import *
import threading
import subprocess
import struct
import hashlib
import json

s = socket(AF_INET, SOCK_STREAM)
s.bind(('0.0.0.0', 9999))
s.listen(5)
print('端口正在监听。。。')

def tcplink(sock, addr):
    while True:
        try:
            data = sock.recv(1024)
            print(data.decode('utf-8'))
            res = subprocess.Popen(data.decode('utf-8'), shell=True, stdout=subprocess.PIPE,
                                   stderr=subprocess.PIPE)
            res_err = res.stderr.read()
            res_std = res.stdout.read()
            if res_err:
                cmd_out = res_err
            elif not res_std:
                cmd_out = 'ok'.encode('utf-8')
            else:
                cmd_out = res_std
            head = {
                'ver': '1',
                'md5': hashlib.md5(cmd_out).hexdigest(),
                'lenth': len(cmd_out)
            }
            head_json = json.dumps(head)
            head_bytes = head_json.encode('utf-8')
            print(head_bytes)
            # 发送报头长度
            sock.send(struct.pack('i', len(head_bytes)))
            # 发送发送报头
            sock.send(head_bytes)
            # 发送数据
            sock.send(cmd_out)
        except Exception:
            break
    sock.close()
    print('来自%s:%s的连接已关闭' % addr)

if __name__ == '__main__':
    while True:
        sock, addr = s.accept()
        print('已接受来自%s:%s的连接' % addr)
        t = threading.Thread(target=tcplink, args=(sock, addr))
        t.start()
```
## 模拟远程执行命令客户端
```python
from socket import *
import struct
import json

t = socket(AF_INET, SOCK_STREAM)
t.connect(('10.10.10.100', 9999))

while True:
    cmd = input('>>>').strip()
    if not cmd:
        continue
    elif cmd == 'exit':
        break
    else:
        t.send(cmd.encode('utf-8'))
        # 接收报头长度
        head_info = t.recv(4)
        head_len = struct.unpack('i', head_info)[0]
        # 接收报头
        head_bytes = t.recv(head_len)
        head_json = head_bytes.decode('utf-8')
        head_dict = json.loads(head_json)
        data_len = head_dict['lenth']
        print(head_dict)
        # 接收数据
        data_buffers = bytes()
        recv_size = 0
        while recv_size < data_len:
            recv_data = t.recv(1024)
            data_buffers+=recv_data
            recv_size+=len(recv_data)
        print(data_buffers.decode('utf-8'))
t.close()
```

[1]:https://blog.csdn.net/keyouan2008/article/details/5843388
[2]: https://blog.csdn.net/yannanxiu/article/details/52096465