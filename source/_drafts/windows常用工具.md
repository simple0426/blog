---
title: windows常用工具
date: 2019-05-11 12:24:02
tags:
    - git
    - beyondCompare
    - pycharm
    - sublimeText
    - xmind
categories:
    - skill
---
# 软件列表
## 办公类
* [beyondCompare](#beyondCompare):文本比较工具
* [git](#git):windows下git工具（一般作为windows下的linux命令行使用）
* [pycharm](#pycharm)：python集成开发工具
* [sublimeText3](#sublimetext3)：文本编辑工具
* [xmind8](#xmind8):思维导图工具
* secureCRT：linux系统远程连接工具
* Postman：接口调试工具
* navicat：数据库连接工具
* DockerToolbox：windows下docker使用环境
    - （默认安装）Oracle VM VirtualBox：虚拟化工具
    - （默认安装）Git MSYS-git UNIX tools：windows下linux命令行工具
* setupssh：windows cmd模式下下ssh工具
* shadowsocks：FQ工具
* wireshark：网络包分析工具

## 生活类
* 快播/QQ影音
* 印象笔记
* windows便签
* 迅雷极速版
* uiso：镜像制作工具
* 驱动精灵万能网卡版
* teamviewer：远程协作工具
* office
* DiskGenius：硬盘分区工具
* DriverStoreExplorer：驱动卸载工具

# beyondCompare
## 适用版本
BCompare-zh-3.3.13.18981
## key信息
```
--- BEGIN LICENSE KEY ---
***REMOVED***
***REMOVED***
***REMOVED***
--- END LICENSE KEY -----
```
# git
## 下载地址
https://git-scm.com/download/win
## 配置
在环境变量PATH中添加路径【C:\Git\bin；C:\Git\usr\bin】以方便适用其中的linux命令
# xmind8
破解参考：https://blog.csdn.net/qq_42863682/article/details/82416153
## 配置
1. 破解包XMindCrack.jar放置xmind安装根目录
2. 编辑XMind.ini，在末尾追加`-javaagent:自己的路径\XMindCrack.jar`
3. 打开xmind8输入序列号【帮助--》序列号】

```
***REMOVED***
***REMOVED***
***REMOVED***
***REMOVED***
```

# pycharm
## 使用版本
JetBrains PyCharm 2019.1.2 x64
## 激活参考
https://zhile.io/2018/08/26/jetbrains-license-server-crack.html

# sublimetext3
## 版本
Sublime Text Build 3143 x64 Setup
## 激活码
```
—– BEGIN LICENSE —–
TwitterInc
200 User License
EA7E-890007
***REMOVED***
***REMOVED***
***REMOVED***
***REMOVED***
***REMOVED***
***REMOVED***
***REMOVED***
***REMOVED***
—— END LICENSE ——
```
## 安装packagecontrol
* 地址(需要翻墙)：https://packagecontrol.io/installation#st3
* 执行命令

```
import urllib.request,os,hashlib; h = '6f4c264a24d933ce70df5dedcf1dcaee' + 'ebe013ee18cced0ef93d5f746d80ef60'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```
## 安装插件
>ctrl+shift+p >> install package >> xxxx

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/sublimeText3-plugins.jpg)
## 全局配置
```
{
    "color_scheme": "Packages/Color Scheme - Default/Monokai.tmTheme",
    "expand_tabs_on_save": true,
    "font_face": "YouYuan",
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
## 快捷键设置
```
[
    { "keys": ["ctrl+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} }
]
```

