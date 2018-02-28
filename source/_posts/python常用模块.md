---
title: python常用模块
date: 2018-02-28 10:35:35
tags: ['subprocess']
categories: ['python']
---
# subprocess
## 作用
* 开启一个子进程，在其中执行系统命令
* 直接和进程通信获取执行结果，或者获取他们的返回码【成功或失败】
* 替代一些旧的模块，比如os.system、os.spawn*

## call方法
* 执行命令，并返回执行状态
* 其中shell参数为False时，命令需要通过列表的方式传入，当shell为True时，可直接传入命令

```python
import subprocess
print('$ nslookup')
r = subprocess.call(['nslookup', 'www.python.org'])  #开启子进程，传入命令行参数，捕获返回码
print('Exite code:', r)
```

## Popen方法
执行命令，获取命令的执行结果
### 参数
* args：shell命令，可以是字符串，或者序列类型，如list,tuple。
* stdin,stdout,stderr：分别表示程序的标准输入，标准输出及标准错误
* shell：与上面方法中用法相同
* cwd：用于设置子进程的当前目录
* env：用于指定子进程的环境变量。如果env=None，则默认从父进程继承环境变量
* universal_newlines：不同系统的的换行符不同，当该参数设定为true时，则表示使用\n作为换行符

### read/write
直接和管道通信，进行读写操作

```python
obj = subprocess.Popen(["python"], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
obj.stdin.write(b"print('test')")
obj.stdin.close()
cmd_out = obj.stdout.read()
obj.stdout.close()
cmd_error = obj.stderr.read()
obj.stderr.close()
print(cmd_out.decode())
```

### communicate
和管道通信，获取结果

```python
import subprocess
#打开一个管道对象，并把命令行执行结果保存到管道中
df = subprocess.Popen(["df", "-P", "-k"], stdout=subprocess.PIPE) 
#和管道通信，获取数据
output = df.communicate()[0]           
#将获取的管道数据读到内存中【解码过程】
print(output.decode("utf-8"))
```

### 进程间通信
将一个子进程的输出，作为另一个子进程的输入

```python
child1 = subprocess.Popen('cat /etc/passwd', shell=True, stdout=subprocess.PIPE)
child2 = subprocess.Popen('grep "0:0"', shell=True, stdin=child1.stdout, stdout=subprocess.PIPE)
out = child2.communicate()
print(out[0].decode())
```

### 其他进程方法
```python
import subprocess
child = subprocess.Popen('sleep 60',shell=True,stdout=subprocess.PIPE)
child.poll()    #检查子进程状态
child.kill()     #终止子进程
child.send_signal()    #向子进程发送信号
child.terminate()   #终止子进程
```
