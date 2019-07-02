---
title: repo
date: 2019-06-24 21:54:01
---

* shadowsocks一键安装：<https://teddysun.com/342.html>
* jdk7下载：<https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html#jdk-7u80-oth-JPR>
* pypi阿里云配置
    - 命令行：-i https://mirrors.aliyun.com/pypi/simple/
    - 配置文件：

    ```
    ~/.pip/pip.conf
    中添加或修改:

    [global]
    index-url = https://mirrors.aliyun.com/pypi/simple/

    [install]
    trusted-host=mirrors.aliyun.com
    ```
* marven配置阿里云镜像
    - 文件：apache-maven-3.6.1/conf/settings.xml
    - 配置

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