---
title: angular运行环境
tags:
  - angular
categories:
  - web
date: 2019-05-15 10:46:02
---

# 使用淘宝npm库
npm install -registry=https://registry.npm.taobao.org
# 安装nodejs
下载地址：https://nodejs.org/en/download/

node为免安装，解压后bin目录有node和npm，只需将bin目录添加到系统PATH下即可
# 安装Angularjs
angular是用typescript编写的，所以先安装typescript，再安装angularjs-cli
* npm install -g typescript typings
* npm install -g @angular/cli

# build项目
* 首先进入项目目录：npm install
* ng build --prod --output-hashing=none

# 错误
* npm install报错：Error: EACCES: permission denied, mkdir '/root/web/CRM-Website-ABKE/node_modules/wd/build'
    - 原因：如果npm是在root账户下执行的话，它会将uid改成当前账户，或者uid的值从user配置文件中获取，而默认情况下uid的值为nobody。所以在root账户下运行npm install时需要将unsafe-perm选项加上。
    - 解决npm install -registry=https://registry.npm.taobao.org --unsafe-perm
* ng build报错:  JavaScript heap out of memory
    - 原因：项目build时需要的内存不足
    - 解决：export NODE_OPTIONS=--max_old_space_size=4096

