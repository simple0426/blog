---
title: gitbook简单使用
date: 2018-02-26 15:52:05
tags: ['gitbook']
categories: ['skill']
---
# 介绍

* [范例参考](http://gitbook.zhangjikai.com)
* 可以将markdown书写的文章转换为html文档直接在网页上浏览
* 可以将markdown书写的文章导出为电子书
* 可以搭配github做文章版本管理

# 安装

* [安装nodejs](http://nodejs.cn/download/)
* 使用nmp安装gitbook-cli  
`npm install gitbook-cli -g`

# 目录设置

* 创建电子书目录\[同时也是github库\]
* 创建SUMMARY.md文件,此文件主要描述电子书的目录结构

![](http://simple0426-blog.oss-cn-beijing.aliyuncs.com/summary.png)

* 创建book.json文件，此文件主要用于描述gitbook提供web服务时使用的插件及配置

```json
{
    "plugins": [
        "-lunr",
        "-search",
        "-livereload",
        "expandable-chapters-small",
        "anchors",
        "anchor-navigation-ex",
        "edit-link"
    ],
    "pluginsConfig": {
        "edit-link": {
            "base": "https://github.com/simple0426/blog/edit/master",
            "label": "修改本文"
        }
    }
}
```

# 命令使用

* gitbook help 使用帮助
* gitbook install 根据book.json中的设置安装相应插件
* gitbook build 根据SUMMARY.md的设置在当前目录的\_book目录下生产用于提供web服务的html文件
* gitbook serve --port 80 在80端口提供浏览电子书的web服务
  * 默认4000端口
  * 此命令同时也会执行build的功能

# 导出电子书

* 需要安装[calibre](https://calibre-ebook.com/download)【使用其中的ebook-convert这个插件】
* 导出命令  
`gitbook pdf . my.pdf`

# 插件使用

* [官网地址](https://plugins.gitbook.com/)
* [中文参考](http://gitbook.zhangjikai.com/plugins.html#search-plus)
