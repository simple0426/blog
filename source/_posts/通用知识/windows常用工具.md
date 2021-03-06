---
title: windows常用工具
tags:
  - git
  - beyondCompare
  - pycharm
  - sublimeText
  - xmind
categories:
  - skill
date: 2019-05-11 12:24:02
---

# 软件列表
## 办公类
* beyondCompare：文本比较工具
* [git](#git)：windows下git工具（一般作为windows下的linux命令行使用）
* [pycharm](#pycharm)：python集成开发工具
* [sublimeText3](#sublimetext3)：文本编辑工具
* xmind8：思维导图工具
* secureCRT：linux系统远程连接工具
* Postman：接口调试工具
* navicat：数据库连接工具
* DockerToolbox：windows下docker使用环境
    - （默认安装）Oracle VM VirtualBox：虚拟化工具
    - （默认安装）Git MSYS-git UNIX tools：windows下linux命令行工具
* shadowsocks：FQ工具
* wireshark：网络包分析工具

## 生活类
* 快播/QQ影音
* 印象笔记
* windows便签
* 迅雷下载极速版
* uiso：镜像制作工具
* 驱动精灵万能网卡版
* teamviewer：远程协作工具
* office
* DiskGenius：硬盘分区工具
* DriverStoreExplorer：驱动卸载工具
* 百度网盘
* firefox/chrome
* QQ浏览器：
    - qq登录
    - 书签同步
    - 网站账号云同步

# git
## 下载地址
https://git-scm.com/download/win
## 配置
在环境变量PATH中添加路径【C:\Git\bin；C:\Git\usr\bin】以方便适用其中的linux命令

# pycharm
设置py文件默认头部信息【file and code templates】

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# @Time    : ${DATE} ${TIME}
# @Author  : simple0426
# @Email   : istyle.simple@gmail.com
# @File    : ${NAME}.py
# @Software: ${PRODUCT_NAME}
```
# sublimetext3
## 安装packagecontrol
* 安装地址：http://packagecontrol.cn/installation#st3
* 执行命令：通过 ctrl+` 或 View > Show Console打开控制台，将Python代码粘贴到控制台，回车。

```
import urllib.request,os,hashlib; h = '6f4c264a24d933ce70df5dedcf1dcaee' + 'ebe013ee18cced0ef93d5f746d80ef60'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.cn/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

* 添加或修改PackageControl的channels【PackageControl-SettingsUser】：

```
"channels": [ "http://packagecontrol.cn/channel_v3.json" ]
```

## 插件
### 安装
>ctrl+shift+p >> install package >> xxxx

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/sublimeText3-plugins.jpg)
### Pretty JSON
按Ctrl+Alt+J就可格式化json数据
###  Table Editor   
* 通过`Ctrl+Shift+P->Table Editor: Enable for current view`开启  
* 先输入标题行,如`|id|name|age|`，回车后在第二行输入`|-`后，  
按tab键就将进入table编辑模式  
* 表格必须与前面输入的文字之间有空行，否则表格会被当成普通文字渲染 

### Markdown Editing 
* 设置：package setting-->markdown editing-->GFM setings User  

```json
{
    "draw_centered": false, //去除文字整体居中
    "color_scheme": "Packages/MarkdownEditing/MarkdownEditor-Dark.tmTheme", //主题与sublime保持一致
    "line_numbers": true, //显示行号
}
```

* 在全局设置中忽略默认的markdown

```json
    "ignored_packages":
    [
        "Markdown",
        "Vintage"
    ],
```

### Markdown preview  
* 按键绑定的设置：  

```
[
    { "keys": ["ctrl+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} }
]
```

## 全局配置
```
{
    "color_scheme": "Packages/Color Scheme - Default/Monokai.tmTheme",
    "expand_tabs_on_save": true,
    "font_face": "微软雅黑",
    "font_size": 12,
    "ignored_packages":
    [
        "Vintage"
    ],
    "pep8_ignore":
    [
        "E123",
        "E128",
        "E301",
        "E302",
        "E309",
        "E401",
        "E305"
    ],
    "tab_size": 4,
    "theme": "Default.sublime-theme",
    "translate_tabs_to_spaces": true,
    "update_check": false
}
```
