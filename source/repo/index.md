---
title: repo
date: 2019-06-24 21:54:01
---
* nginx文档
    - [淘宝tengine项目](http://tengine.taobao.org/)
    - [淘宝nginx文档](http://tengine.taobao.org/nginx_docs/cn/docs/)
* [shadowsocks一键安装](https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh)
* [jdk7下载](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html#jdk-7u80-oth-JPR)
* [BeyondCompare/xmind8/pycharm/sublimetext3破解](https://github.com/simple0426/key_store/blob/master/key.md)
* [jetbrains系列破解资源包(pycharm)](https://pan.baidu.com/s/1RAijfj89uf1oY7zazB1Umg)
* pypi阿里云配置
    - 命令行设置：-i https://mirrors.aliyun.com/pypi/simple/
    - 配置文件设置：

```
~/.pip/pip.conf
中添加或修改:

[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
```

* marven阿里云配置(conf/settings.xml)

```
<mirrors>
   <mirror>
     <id>nexus-aliyun</id>
     <mirrorOf>central</mirrorOf>
     <name>Nexus aliyun</name>
     <url>http://maven.aliyun.com/nexus/content/groups/public</url>
   </mirror>
 </mirrors>
```

* npm淘宝配置
    - 命令行临时设置：--registry=https://registry.npm.taobao.org
    - 永久设置：npm config set registry https://registry.npm.taobao.org
        + 验证：npm config get registry
* docker镜像地址(registry-mirrors)
    - 阿里云：https://2x97hcl1.mirror.aliyuncs.com
    - 163：http://hub-mirror.c.163.com
    - docker-cn：https://registry.docker-cn.com
    - daocloud：http://f1361db2.m.daocloud.io
