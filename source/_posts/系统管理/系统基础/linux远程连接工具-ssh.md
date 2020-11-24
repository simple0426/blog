---
title: linux远程连接工具-ssh
tags: ssh
categories:
  - linux
date: 2019-08-07 13:35:14
---

# 简介
>Secure shell protocol：安全的shell协议

* 服务端软件：openssh
* 客户端软件：securecrt、putty、openssh-client
* 登录方式：
    - 口令：交互式输入用户名和密码
    - 秘钥：使用秘钥文件
        + 产生秘钥对：ssh-keygen -t rsa -P "" -f ./bastion
        + 传送公钥到远端：ssh-copy-id -i ~/.ssh/bastion.pub

# 配置文件
## 服务端配置
>/etc/ssh/sshd_config

```
PermitRootLogin no # 是否允许root登录
PermitEmptyPasswords no # 是否允许无密码登录
Port 22 # 端口设置
ClientAliveInterval 60 #server每隔60秒发送一次请求给client，然后client响应，从而保持连接
ClientAliveCountMax 3 #server发出请求后，client没有响应次数达到3，就自动断开连接，一般client会响应。
```

## 客户端配置
* 配置文件：/etc/ssh/ssh_config（全局）、~/.ssh/config（用户级）
* 用途1：通过ssh代理转发实现跳板机功能（主机A通过B登录C）

```
Host ops
  User muker #连接跳板机使用的用户名
  HostName 47.99.78.151 #跳板机ip
  ProxyCommand none
  BatchMode yes #跳板机模式
  IdentityFile ~/keyfiles/bastion #本地连跳板机时使用的私钥
  StrictHostKeyChecking no #首次登录时禁止秘钥检查
Host 172.16.0.*  # 目标主机网络
  ServerAliveInterval 60 
  TCPKeepAlive        yes
  ProxyCommand ssh -qaY -i ~/keyfiles/bastion muker@ops 'nc -w 14400 %h %p' # ssh代理转发
  IdentityFile    ~/keyfiles/internal #跳板机使用私钥internal连接目标主机
  StrictHostKeyChecking no
```

* 用途2：实现快速登录

```
Host *
     IdentityFile ~/.ssh/feidao
     PreferredAuthentications publickey,keyboard-interactive,password
     ForwardAgent yes
     StrictHostKeyChecking no
     ServerAliveInterval 300
     ServerAliveCountMax 24
Host gitlab
     User root
     Hostname 10.0.124.6
     Port 22
```

# ssh命令

ssh 选项 user@host `command`
## 命令选项
* -a：禁用转发身份验证代理连接
* -Y：启用可信的X11转发
* -q：抑制警告和诊断信息输出
* -F configfile：客户端配置文件
* -i identity_file：私钥文件
* -p port：远程ssh服务端口
* -v：开启连接详细输出
    - 主要用于故障调试
    - 最多使用3个v

## scp命令
* -P：设置连接端口
* -p：保留文件时间戳
* -r：递归复制目录下内容

# 常见问题
* 只允许秘钥登录
  - 现象：Permission denied (publickey)
  - 解决：变更配置sshd_config 【PasswordAuthentication yes】
* 不允许root登录
  - 现象：ssh使用root登录时，密码正确但是拒绝登陆
  - 解决：变更配置sshd_config【PermitRootLogin yes】

# 其他参考
* [Centos 6.5升级openssh到7.9p1](https://blog.csdn.net/qq_25934401/article/details/83419849)
