---
title: python常用模块
date: 2018-02-28 10:35:35
tags: ['subprocess']
categories: ['python']
---
# 包和模块
## 简介
* 在python中，一个py文件就是一个模块
* 使用模块还可以避免函数名和变量名冲突，相同的函数和变量完全可以分别存在不同的模块中；我们自己在编写模块时，不必考虑名字与其他模块冲突，但是尽量不要和内置函数名冲突。
* 为了避免模块名冲突，python又引入了按目录来组织模块的方法，称之为包。
* 每一个包目录下都会有`__init__.py`文件，在python2下这个文件是必须存在的【python3中可以不存在】，否则python就会把这个目录当成普通目录,而不是一个包；`__init__.py`可以是空文件，也可以有python代码；因为`__init__.py`本身就是一个模块，而它的模块名就是目录名。

## 模块与脚本
* 当python文件当做脚本直接运行时，`__name__ = '__main__'`;
* 当python文件作为模块导入时，`__name__='文件名'`;
* 因此可以在`if __name__ == '__main__':`下定义一些只作为脚本使用时的操作，而作为模块不执行。

## 导入方式
* import a as b：导入模块a并设置为别名b，实际为导入该包下的init.py文件
* from A.C import B：B为具体的模块名或函数名
* 包内导入：
    - 绝对导入【最上层使用 包名】：from A.C import B
    - 相对导入【使用逗号表示目录层级】：from ..C import B

## 搜索路径
>sys.path显示模块搜索路径（使用列表的append方法可以修改）

* 当前目录
* PYTHONPATH定义路径
* python默认安装路径
 
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

# logging
>记录日志的模块

## 参数
>日志级别：critical>error>warning>info>debug>noset

|      参数     |                含义               |
|---------------|-----------------------------------|
| level         | 大于某等级的日志                  |
| %(asctime)s   | 创建时间                          |
| %(filename)s  | 当前执行的文件                    |
| %(lineno)d    | 当前行号                          |
| %(levelname)s | 当前日志级别名称                  |
| %(message)s   | 当前日志信息                      |
| datefmt       | 时间格式                          |
| filename      | 记录的文件名                      |
| filemode      | 文件打开模式（'w'：写 'a'：追加） |

## 范例
```python
import logging
logging.basicConfig(level=logging.INFO,
            format='%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s',
            datefmt='%a, %d %b %Y %H:%M:%S',
            filename='/tmp/apiserver.log',
            filemode='w')
logging.debug('this is debug message')
logging.info('this is info message')
logging.warning('this is warn message')
```

# 系统管理
## os
* `os.path.isfile(__file__)`：判断文件是否存在
* os.environ：获取系统环境变量
* `os.path.abspath(__file__)`:当前文件的绝对路径
* os.mkdir('ceshi'):创建目录
* os.rmdir('ceshi')：删除空目录
* os.remove('./ceshi/1.py')：删除文件

## shutil
* shutil.rmtree('ceshi')：删除目录
* shutil.copy('module.py', 'ceshi.txt')：复制文件
* shutil.copytree('11', 'ww')：复制目录
    - shutil.copytree('11', 'ww', ignore=shutil.ignore_patterns('12'))：排除指定的文件
* shutil.make_archive('22', 'gztar', root_dir='11')：打包文件或目录

## sys
* sys.argv：命令行参数【列表形式】
* sys.stdout：进度条实现

```python
for i in range(30):
    # 向终端输出字符
    sys.stdout.write('%s' % '#'*i)
    # 刷新缓存立即显示
    sys.stdout.flush()
    time.sleep(0.5)
    # \r光标回到行首
    sys.stdout.write('\r')
sys.stdout.write('\n')
```

## argparse
命令行选项与参数解析

```python
import argparse
import os

def nslookup(url):
    print(os.system('nslookup %s' % url))

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    # 设置需要解析的选项参数
    parser.add_argument('--hostname', '-n', help="url info")
    # 从命令行获取选项参数，并将其转换为字典
    args = vars(parser.parse_args())

    if args['hostname']:
        nslookup(args['hostname'])
    else:
        # 打印帮助信息
        print(parser.print_help())
```

# time
```python
import time
# 获取当前系统时间
time1 = time.localtime()
# 获取当前系统时间的子项
print(time1.tm_year)
# 获取格式化时间
print(time.strftime('%Y-%m-%d %H:%M:%S'))
print(time.strftime('%X'))
# 格式化时间
time2 = time.strptime('2018-05-12 10:04:55', '%Y-%m-%d %H:%M:%S')
print(time2)
# 暂停3s
time.sleep(3)
```

# datetime
## 语法
```python
from datetime import datetime, timedelta
# 获取当前系统时间
now = datetime.now()
```
## timestamp
我们把1970年1月1日 00:00:00 UTC+00:00时区的时刻称为epoch time，记为0；  
timestamp的值与时区毫无关系，因为timestamp一旦确定，其UTC时间就确定了；  
转换到任意时区的时间也是完全确定的，这就是为什么计算机存储的当前时间是以timestamp表示的，  
因为全球各地的计算机在任意时刻的timestamp都是完全相同的 datetime.timestamp(now)
## 时间格式
|       函数      |             含义            |                              范例                             |
|-----------------|-----------------------------|---------------------------------------------------------------|
| timestamp()     | datetime转换为timestamp     |                                                               |
| fromtimestamp() | timestamp转换为本地datetime |                                                               |
| strptime()      | str转换为datetime           | datetime.strptime('2015-12-20 11:11:11', '%Y-%m-%d %H:%M:%S') |
| strftime()      | datetime转换为str           | datetime.now().strftime('%y-%d-%m')                           |

```python
# timestamp
tm = datetime.timestamp(now)
print(datetime.fromtimestamp(tm))
# strf
print(now.strftime('%Y-%m-%d'))
# strp
time = '1988-04-26 00:00'
time1 = datetime.strptime(time, '%Y-%m-%d %H:%S')
print(type(time1))
```
## 时间计算
```python
# timedelta
Tom = now + timedelta(hours=1)
print(Tom)
```
# random
返回随机数
## 方法
```python
# 0~1之间的随机小数
a = random.random()
# 1~5之间的随机整数
a = random.randint(1, 5)
# 1~5之间的随机整数【不包含结束】
a = random.randrange(1, 5)
# 从列表中随机选择一个数字
a = random.choice(range(1, 10))
# 从列表中国随机选择2个数字组成新的结果
a = random.sample(range(1, 10), 2)
# 1~3之间的随机小数
a = random.uniform(1, 3)
# 乱序重排
a = [1, 3, 8, 2, 4]
random.shuffle(a)
```
## 范例
>生成由数字和大写字母组成的n位字符串

```python
def v_code(n):
    res = ""
    for i in range(n):
        # chr 数字转字母
        str1 = chr(random.randint(65, 90))
        int1 = str(random.randint(0, 9))
        res1 = random.choice([str1, int1])
        res += res1
    return res
print(v_code(4))
```
# hashlib
* 摘要算法又称哈希算法、散列算法。它通过一个函数，把任意长度的数据转换为一个长度固定的数据串（通常用16进制的字符串表示）
* 摘要算法不是加密算法，不能用于加密（因为无法通过摘要反推明文），只能用于防篡改；但是它的单向计算特性决定了可以在不存储明文口令的情况下验证用户口令
* 通过在原始口令中“加盐”并配合用户名，可以实现不同用户名相同口令但是在数据库中以不同的摘要存储

```python
import hashlib
md5 = hashlib.md5()
md5.update("how to use python".encode("utf-8"))
print(md5.hexdigest())

user = 'hejingqi'
password = '123456'
salt = '123eedcdwsx'
sha1 = hashlib.sha1()
sha1.update((user + password + salt).encode('utf-8'))
print(sha1.hexdigest())
```

# urllib
## get请求
```python
from urllib import request

req = request.Request('http://www.douban.com/')
req.add_header('User-Agent', 'Mozilla/6.0 (iPhone; CPU iPhone OS 8_0 like Mac OS X) \
    AppleWebKit/536.26 (KHTML, like Gecko) Version/8.0 Mobile/10A5376e Safari/8536.25')
with request.urlopen(req) as f:
    print('status: %s %s' % (f.status, f.reason))
    for k, v in f.getheaders():
        print('%s: %s' % (k, v))
    print('Data: %s' % f.read().decode('utf-8'))
```

## 下载
`request.urlretrieve('http://www.521609.com/uploads/allimg/140717/1-140GF92J7.jpg', 'a.jpg')`
