---
title: socket编程
tags:
  - socket
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

# 范例
## 模拟远程登录服务端
```python
from socket import *
import threading
import subprocess

s = socket(AF_INET, SOCK_STREAM)
s.bind(('127.0.0.1', 10000))
# s.setsockopt(SOL_SOCKET,SO_REUSEADDR,1)
s.listen(5)
print('端口正在监听。。。')

def tcplink(sock, addr):
    while True:
        try:
            cmd = sock.recv(1024)
            if not cmd:
                break
            # print(cmd.decode())
            res = subprocess.Popen(cmd.decode('utf-8'), shell=True,
                                   stderr=subprocess.PIPE, stdout=subprocess.PIPE)
            res_err = res.stderr.read()
            res_std = res.stdout.read()
            if res_err:
                # print('err')
                cmd_out = res_err
            # 假如执行命令结果为空
            elif not res_std:
                cmd_out = 'ok'.encode('utf-8')
            else:
                # print('std')
                cmd_out = res_std
            print(cmd_out)
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

## 模拟远程登录客户端
```python
from socket import *
import time
s = socket(AF_INET, SOCK_STREAM)
s.connect(('127.0.0.1', 10000))
while True:
    cmd = input('>>>').strip()
    if not cmd:
        continue
    elif cmd == 'exit':
        break

    s.send(cmd.encode('utf-8'))
    cmd_out = s.recv(1024)
    print(cmd_out.decode('utf-8'))
s.close()
```