---
title: python环境管理
tags:
  - virtualenv
  - pip
categories:
  - python
date: 2018-05-30 17:10:00
---

# virtualenv
创建一个隔离的python运行环境

## 安装
pip3 install virtualenv
## 创建独立环境
* --no-site-packages 不复制系统已存在的软件包
* -p指定解释器，否则使用默认解释器

virtualenv -p /usr/bin/python3 venv
## 进入虚拟环境
source venv/bin/activate
## 退出虚拟环境
deactive

# pip
python包管理工具
## 安装
1. 更新系统软件库：apt-get update
2. 安装python-pip：apt-get install python-pip
3. 升级python-pip：pip install pip --upgrade
4. 使用easy_install替换pip为最新版本：easy_install pip

## 使用
* install：安装：pip install xxx [-f requirements.txt]
* uninstall：卸载
* list：安装包显示
* freeze：安装包显示与导出：pip freeze >requirements.txt
* search：软件包在线搜索

## 软件源设置
>文件位置：~/.pip/pip.conf

```
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
```
