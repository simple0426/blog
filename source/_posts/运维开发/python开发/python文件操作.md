---
title: python文件操作
tags:
  - open
  - json
  - xml
  - ini
categories:
  - python
date: 2018-05-11 16:49:14
---

# open函数
## with使用
>上下文管理

* 可以一次打开多个文件进行操作
* 不用自己处理文件关闭操作

```python
with open('test.txt', encoding='utf-8') as f1,open('b.txt', mode='w', encoding='utf-8') as f2:
    print(f1.read())
    f2.write('ok')
```

## 读操作
>文本编辑器(vim、记事本等)和python解释器(如open函数)默认使用系统的编码方式(windows的gbk，linux的utf-8)保存和打开文件
>
>当然也可以指定这俩种方式下的编码方式(encoding)

* 文件的默认操作模式：mode='r'
* 文件对象的读方法
    - read：一次读取整个文件，或指定字符数
    - readline：一次读取一行
    - readlines：将读取的整个文件保存为列表输出

## 写操作
* 文件写模式：
    * mode='w'(覆盖写)
    * mode='a'(追加写)
* 文件对象的写方法
    - write：一次写入整个内容
    - writelines：将列表或元组写入文件（列表或元组的元素必须是字符串，此时相当于将多个字符串拼接为单个字符串后写入文件）

```python
shuzi = list(range(0,10))
kexie = [str(n) + '\n' for n in shuzi]
with open('shuzi.txt', mode='w', encoding='utf-8') as f:
    f.writelines(kexie)
```

## 文件打开模式
* r+ 读写
* w+ 写读
* a+ 追加读
* b 以bytes形式读取【默认str】

```python
# 复制的实现
with open('a.png', mode='wb') as w, open('test.png', mode='rb') as r:
    w.write(r.read())
```

## 文件对象操作-seek
* seek 移动光标；参数1为偏移量(字节数)，参数2为起始位置
    * 起始位置0表示数据流的开始(默认)、1表示当前光标位置、2表示数据流的末尾
    * str模式下
        * 起始位置为0，偏移量必须大于等于0
        * 起始位置为1、2，偏移量必须等于0
    * bytes模式下
        * 起始位置为0，偏移量必须大于等于0
        * 其实位置为1，偏移量可正可负
        * 起始位置为2，偏移量为负数
* tell 显示当前光标位置

## 范例
* 模拟tailf

```python
import time
with open('db.txt', encoding='utf-8') as f:
    f.seek(0,2)
    while True:
        data = f.read().strip()
        if data:
            print(data)
        time.sleep(0.5)
```

* 模拟文件修改

```python
import os
with open('shuzi.txt', 'r') as f1, open('shuzi.txt.tmp', 'w') as f2:
    lines = f1.readlines()
    for line in lines:
        if line.startswith('ce'):
            line = 'aa\n'
        f2.write(line)
os.remove('shuzi.txt')
os.rename('shuzi.txt.tmp', 'shuzi.txt')
```

# json与序列化
把变量从内存中取出变成可存储或传输的过程
## pickle
```python
import pickle
d = dict(name='Bob', age=20, score=88)
# 序列化（写）
with open('./dump.txt', 'wb') as f:
    pickle.dump(d, f)
# 反序列化（读）
with open('./dump.txt', 'rb') as f:
    print(pickle.load(f))
```
## json
* json表示的对象就是标准的javascript语言的对象。  
* json不仅是标准格式，并且比XML更快，而且可以直接在web页面中读取，非常方便；  

### json和python
|  json类型  |     python类型    |
|------------|-------------------|
| {}         | dict              |
| `[]`       | list              |
| "string"   | 'str'或u'unicode' |
| 123.45     | int或float        |
| true/flase | True/False        |
| null       | None              |
### 序列化
* dump生成一个类文件对象
* dumps生成一个字符串

```python
import json
d = dict(name='Bob', age=20, score=88)
with open('./1.json', 'w') as f:
    # json.dump(d, f, indent=4)
    f.write(json.dumps(d, indent=4))
```

### 反序列化
* load 解析类文件对象
* loads 解析json格式字符串

```python
with open('./1.json', 'r') as f:
    # print(json.load(f))
    print(json.loads(f.read()))
```

### 非标准类型
```python
import json
from json.encoder import JSONEncoder
from datetime import datetime
# https://docs.python.org/3/library/json.html?highlight=json#module-json
class CustJSONEncoder(JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.strftime('%Y-%m-%d  %H:%M:%S')
        return json.JSONEncoder.default(self, obj)
now = datetime.now()
res = json.dumps(now, cls=CustJSONEncoder)
print(res)
```

# xml解析
## 语法
```python
# 导入
try:
    import xml.etree.cElementTree as ET
except ImportError:
    import xml.etree.ElementTree as ET
# 创建解析器
tree = ET.parse('repositories.xml')
# 获取根节点
root = tree.getroot()
```

## 查找
```python
# 遍历子元素
for child in root:
    print(child.tag, child.text, child.attrib)

# 遍历后代元素
# 查看标签名称、文本、属性内容
for ele in root.iter():
    print(ele.tag, ele.text, ele.attrib)

# findall使用xpath查找子元素下所有特定标签【默认查找当前标签下的子元素】
code = root.findall('connection//code')

# find使用xpath查找子元素下第一个特定标签【默认查找当前标签下的子元素】
server = root.find('connection/server')
```

## 变更
```python
# 创建节点
nei = ET.Element('nei')
nei.text = 'hi'
nei.attrib['name'] = 'ceshi'
# 变更节点
par = root.find('connection/attributes')
for child in par:
    if child.find('code').text == 'IS_CLUSTERED':
        # 添加节点【父节点末尾添加】
        child.append(nei)
        # 修改节点相关值
        child.find('attribute').text = 'true'
        # 删除相关节点【父节点删除子节点】
        par.remove(child)
tree.write('res1.xml')
```

# ini解析
## 语法
```python
import configparser
# 创建解析器
config = configparser.ConfigParser()
# 读取ini文件
config.read('12.txt')
```
## 查找
```python
# 查看所有块标题
print(config.sections())
# 查看特定标题下的所有key
print(config.options('persistent_connection'))
# 获取指定标题下key的值
print(config.get('defaults', 'log_path'))
print(config.getint('persistent_connection', 'connect_timeout'))
```
## 变更
```python
# 添加块
config.add_section('aa')
# 添加或修改块的key
config.set('aa', 'name', 'jingqi')
config.write(open('12.txt', 'w'))
# 删除块
config.remove_section('aa')
# 删除块的key
config.remove_option('persistent_connection', 'connect_timeout')
config.write(open('34.txt', 'w'))
```
