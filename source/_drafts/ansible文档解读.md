---
title: ansible文档解读
tags:
categories:
---
# playbook
官方示例：https://github.com/ansible/ansible-examples

## playbook使用建议
* 使用git等版本控制系统保留playbook及变更事项

# 部署实践
* 发布
    - 灰度发布
    - 回滚
* docker
* jenkins
* vagrant
* 阿里云：<https://docs.ansible.com/ansible/latest/scenario_guides/guide_alicloud.html>
* web界面awx【社区版tower】：https://github.com/ansible/awx【官方只支持容器部署】
    - 基于角色的访问控制
    - 作业调度
    - 集成通知
    - 图形化主机资源
    - centos7安装：http://yallalabs.com/devops/how-to-install-ansible-awx-without-docker-centos-7-rhel-7/

# 自定义开发
## 动态inventory
## 自定义模块
## 自定义插件
## 编写可重用的roles

解读位置：https://docs.ansible.com/ansible/latest/plugins/plugins.html
