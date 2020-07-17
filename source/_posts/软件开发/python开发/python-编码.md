---
title: python-编码
tags:
  - 编码
categories:
  - python
date: 2020-07-10 22:45:43
---

# 通用编码
* ASCII[1个字节] 只支持英文字母和一些常用的符号
* Unicode[2个字节]
* UTF-8 把一个Unicode字符根据不同的数字大小编码成1-6个字节
    - 常用的英文字母被编码成1个字节，汉字通常是3个字节，只有很生僻的字符才会被编码成4-6个字节
    - ASCII编码实际上可以被看成是UTF-8编码的一部分

# 中文编码
* gb2312
    - GB2312(1980年)一共收录了7445个字符，在windows中的代码页是CP936
    - 原来的CP936和GB 2312-80一模一样
* gbk
    - GBK最初是由微软对GB2312的扩展，也就是CP936字码表 (Code Page 936)的扩展
    - gbk并非国家正式标准
* GB18030
    - GB18030取代了GBK1.0的正式国家标准。该标准收录了27484个汉字，同时还收录了藏文、蒙文、维吾尔文等主要的少数民族文字。
    - 现在的PC平台必须支持GB18030，GB18030在windows中的代码页是CP54936。

# python文件设置
python3默认使用utf-8编码读取文件，支持读取中文  
python2默认使用ASCII编码读取文件，不支持读取中文；在文件中设置`#-*- coding:utf8 -*-`可以使python2使用UTF-8编码方式读取文件，从而支持读取中文  

# python2与python3
* python3
    - str即为unicode
    - bytes即为str.encode()的结果
* python2
    - str是bytes
    - u'str'为unicode

# python3下encode与decode
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/python_encode_decode.jpg)

* 数据在内存中，数据类型为str，统一使用unicode编码
* 数据在网络传输或磁盘存储，数据类型为bytes，使用UTF-8编码
* 内存中的数据转换为网络或磁盘上的数据时(str==>bytes)，使用encode方法
* 从网络或磁盘读取数据到内存(bytes==>str)，需要使用decode方法

# 单字符的编码
* ord(ordinals)：获取单字符的unicode编码(整数)
    - print(ord('中'))
* chr(character)：把Unicode编码(整数)转换为对应的字符
    - print(chr(20013))