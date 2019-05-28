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

## ansible-inventory
资源查看工具

* --list：显示所有主机信息
    - --export ：与--list搭配使用，优化显示信息
* --host：显示特定主机信息
* -y/--yaml：以yaml显示输出【默认json格式】
    - 与list和host搭配使用
* --graph：以图表形式显示所有主机信息
    - --vars：在图表中显示变量信息

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
### ansible-galaxy
管理共享仓库中的ansible角色，默认的共享仓库是<https://galaxy.ansible.com.>

* info：查询已经安装的或在共享仓库中的角色详细信息
* search：从共享仓库搜索角色【全名搜索】
* list：查看本地已经安装的角色【全名搜索】
* remove：移除本地角色
* install：安装一个角色
    - -f 强制覆盖已存在的角色
    - --force-with-deps：强制覆盖已存在的角色和依赖信息
