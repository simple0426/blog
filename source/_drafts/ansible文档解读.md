---
title: ansible文档解读
tags:
categories:
---
# 功能介绍

* 配置管理：
    - 基于当前的架构理解，使用简单的语言描述操作过程
    - 基于状态的描述【幂等特性】，只需保证目标对象将要处于的状态，而不管目标对象的当前状态
    - 基于openssh进行安全通信、无需额外的端口、无需客户端

* 程序部署：
    - 基于playbook的强大功能
        + 可重复使用
        + 编写简单
    - 内置的模块
    - 不停机更新
    - 工作流程【workflow】管理
    - 云部署

* 持续交付【流程管理】：
    - 软件版本的快速迭代更新
    - 滚动更新【percentage】、零停机时间实现：
        + 负载均衡器
        + 监控系统
    - 多环境管理【多inventory】
    - 在持续集成环境【jenkins】中调用ansible执行playbook
    - devops：
        + devops：https://www.ansible.com/overview/devops
        + devops-tools：https://www.ansible.com/integrations/devops-tools

* 资源编排【架构编排】：
    - 架构编排
        + 资源的生成
        + 服务的操作顺序
    - 通过单一语法管理各种类型资源，比如：
        - 物理裸机：cobbler
        - 虚拟化：vagrant、docker
        - 云：公有云/私有云
        - 容器：docker/Kubernetes
        - 操作系统
            + 软件安装
            + 系统配置
        - 网络
        - 存储

# playbook

# 部署实践
* 发布
    - 灰度发布
    - 回滚
* 调试【debug】
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



