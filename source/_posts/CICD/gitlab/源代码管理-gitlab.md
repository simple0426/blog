---
title: 源代码管理-gitlab
date: 2020-07-06 14:48:16
tags:
  - gitlab
categories:
  - CICD
---
# gitlab服务
## [软件安装][gitlab-repo]
### [docker方式](https://docs.gitlab.com/omnibus/docker/)
>由于gitlab的访问端口由external_url中决定，如下配置中  
>容器中的gitlab服务暴露在80端口，宿主机以外通过http://172.17.8.53:9999/访问gitlab的web服务

```
docker run -d \
  --name gitlab \
  -p 8443:443 \
  -p 9999:80 \
  -p 9998:22 \
  -v $PWD/config:/etc/gitlab \
  -v $PWD/logs:/var/log/gitlab \
  -v $PWD/data:/var/opt/gitlab \
  -v /etc/localtime:/etc/localtime \
  --restart=always \
  -e GITLAB_OMNIBUS_CONFIG="external_url 'http://172.17.8.53/';gitlab_rails['gitlab_shell_ssh_port'] = 9998" \
  gitlab/gitlab-ce:latest
```
### ubuntu16.04
* 软件源

```
sudo curl https://packages.gitlab.com/gpg.key 2> /dev/null | sudo apt-key add - &>/dev/null
echo "deb https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu xenial main" > /etc/apt/sources.list.d/gitlab-ce.list
```

* 安装

```
sudo apt-get update
sudo apt-get install gitlab-ce
```
### Centos7
* 软件源

```
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
gpgcheck=0
enabled=1
```

* 安装

```
sudo yum makecache
sudo yum install gitlab-ce
```
## 软件配置
>/etc/gitlab/gitlab.rb

* 服务器地址配置【在邮件通知中会使用此地址】  
`external_url 'http://10.10.10.164'`
* 备份保留时间配置[以s为单位]  
`gitlab_rails['backup_keep_time'] = 86400`
* 邮箱配置

```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "gitlab@abc.com"
gitlab_rails['smtp_password'] = "Git@abc123"
gitlab_rails['smtp_domain'] = "exmail.qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = "gitlab@abc.com"
```

* [邮箱功能测试][gitlab-mail]
    - 进入控制台：`gitlab-rails console`
    - 测试：`Notify.test_email('destination_email@address.com', 'Message Subject', 'Message Body').deliver_now`

## 常用命令
* 查看gitlab版本：`cat /opt/gitlab/version-manifest.json` 
* 变更配置：`sudo gitlab-ctl reconfigure`
* 创建备份：`gitlab-rake gitlab:backup:create`
* 恢复备份(BACKUP后为在备份目录【默认/var/opt/gitlab/backups/】产生文件名中数字)：
`gitlab-rake gitlab:backup:restore BACKUP=1507879010_2017_10_13`
* 服务状态操作(查/起/停)：`gitlab-ctl status|start|stop`

# git客户端配置
* web界面注册账号
* 本地配置账号

```
git config --global   user.name=simple0426
git config --global   user.email=perfect_0426@qq.com
```

* 生产秘钥并上传至gitlab  
`ssh-keygen -t rsa -C "perfect_0426@qq.com`
* 测试连通性  
`ssh -T git@github.com`
* 根据gitlab范例创建项目  

```
mkdir blog-comments
cd blog-comments
git init
echo "# blog-comments" >> README.md
git add README.md
git commit -m "first commit"
# 本地库关联远程库
git remote add origin git@github.com:simple0426/blog-comments.git
git push -u origin master
```
# 多仓库设置
## 产生密钥对
* ssh-keygen.exe -t rsa -P "" -f "C:\Users\simple\\.ssh\gitee"
* ssh-keygen.exe -t rsa -P "" -f "C:\Users\simple\\.ssh\github"

## 在两个平台添加公钥
## 配置ssh的config文件
>windows下：C:\Users\simple\\.ssh\config

```
Host gitee.com
  ProxyCommand none
  IdentityFile C:\Users\simple\\.ssh\gitee
  ServerAliveInterval 60
  StrictHostKeyChecking no
  TCPKeepAlive yes
Host github.com
  ProxyCommand none
  IdentityFile C:\Users\simple\\.ssh\github
  ServerAliveInterval 60
  StrictHostKeyChecking no
  TCPKeepAlive yes
```
## ssh登录测试
* ssh -T git@github.com
* ssh -T git@gitee.com

## 配置项目config文件
>.git\config

```
[remote "github"]
        url = git@github.com:simple0426/blog.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[remote "gitee"]
        url = git@gitee.com:simple0426/simple0426.git
        fetch = +refs/heads/*:refs/remotes/origin/*
```
## 客户端命令
* git pull github master
* git push github master

[gitlab-repo]: https://mirrors.tuna.tsinghua.edu.cn/help/gitlab-ce/
[gitlab-mail]: https://docs.gitlab.com/omnibus/settings/smtp.html#testing-the-smtp-configuration