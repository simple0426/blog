---
title: ansible文档解读
tags:
categories:
---
# 功能介绍

# 安装
* yum
* apt
* pip
* 其他参考：<https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installation-guide>

# 远程连接设置
## 免密登陆
## 跳板机设置

## 远程连接
* ansible默认使用原生openssh用于远程连接，这将开启ControlPersist 功能
* 但由于RHEL6和centos6的openssh版本过老，在这些系统上将会使用paramiko来替代
* 有时会出现一个设备不支持sftp，这时候就需要使用scp来替代【比如使用跳板机】

# 配置文件设置

# 资源文件设置

# 命令行
## ansible-pull
## ansible-config
配置查看工具

* ansible-config dump --only-changed：查看ansible.cfg中的自定义配置【非默认配置】
* ansible-config view：查看当前配置文件ansible.cfg

# playbook

# 部署实践
* 灰度发布
* 回滚
* 调试【debug】
* docker
* jenkins
* vagrant
* 阿里云：<https://docs.ansible.com/ansible/latest/scenario_guides/guide_alicloud.html>
* ansible tower与web界面

# 自定义开发
## 动态inventory
## 自定义模块
## 自定义插件
## galaxy与社区roles
