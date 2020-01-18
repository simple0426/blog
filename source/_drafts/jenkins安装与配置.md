---
title: jenkins安装与配置
tags:
categories:
---
# 安装
## 配置java8运行环境
## jenkins安装
* centos安装
```
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
yum install jenkins -y
```
* ubuntu安装
```
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```
* 源码包安装
    - 软件源：https://mirrors.tuna.tsinghua.edu.cn/jenkins/【debian-stable、redhat-stable】
    - deb包安装失败
        + sudo apt-get update # 更新
        + sudo apt-get -f install # 解决依赖关系
        + sudo dpkg -i xxx.deb # 重新安装
    - 安装错误
        + 错误：ERROR: No Java executable found in current PATH: /bin:/usr/bin:/sbin:/usr/sbin
        + 解决：ln -s /usr/local/jdk1.8.0_241/bin/java /usr/bin/

## web界面操作
* 访问地址：http://JENKINS_URL:8080
* 使用国内镜像中心的步骤
    1. 更新简体中文插件到1.0.10+
    2. 点击页面右下角（确保浏览器的语言为中文）的Jenkins中文社区
    3. 点击使用按钮
    4. 修改更新中心的地址为https://updates.jenkins-zh.cn/update-center.json
    5. 在插件管理的高级页面，点击立即获取
* 邮件设置(Mailer插件)
    - 系统管理员帐户
    - SMTP服务器户
    - SMTP认证
    - 用户名
    - 密码
    - 发送测试【如果设置的邮件为163，则收件人和发件人一样；否则会显示为垃圾邮件不能发送】
    - 测试用户邮箱地址
