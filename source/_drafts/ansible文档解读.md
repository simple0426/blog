---
title: ansible文档解读
tags:
categories:
---
# 安装
安装参考：<https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installation-guide>
# 前言
## 远程连接
* ansible默认使用原生openssh用于远程连接，这将开启ControlPersist 功能
* 但由于RHEL6和centos6的openssh版本过老，在这些系统上将会使用paramiko来替代
* 有时会出现一个设备不支持sftp，这时候就需要使用scp来替代【比如使用跳板机】

# 阿里云
ansible与阿里云：<https://docs.ansible.com/ansible/latest/scenario_guides/guide_alicloud.html>

# ansible-config
配置查看工具

* ansible-config dump --only-changed：查看ansible.cfg中的自定义配置【非默认配置】
* ansible-config view：查看当前配置文件ansible.cfg

# ansible-pull
# 动态inventory
# 自定义模块
# 自定义插件