---
title: python之文件操作
tags:
categories: ['python']
---
# 文件操作open
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
>文本编辑器（vim、sublime、记事本等）和python解释器（如open函数）,默认使用系统的编码方式保存和打开文件,
>如windows的gbk，linux的utf-8;当然也可以指定这俩种方式下的编码方式。

* 文件的默认操作模式：mode='r'
* 文件对象的读方法
    - read：一次读取整个文件，或指定字节数
    - readline：一次读取一行
    - readlines：将读取的整个文件保存为列表输出

## 写操作
* 文件操作模式：mode='w'[覆盖写]和mode='a'[追加写]
* 文件对象的写方法
    - write：一次写入整个内容
    - writelines：将列表或元组写入文件【自己处理换行】

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

## 文件对象操作
