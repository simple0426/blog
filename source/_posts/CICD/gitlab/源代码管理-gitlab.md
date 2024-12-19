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

> 初始密码（用户root）：cat /etc/*gitlab*/initial_root_password

### [docker方式](https://docs.gitlab.com/omnibus/docker/)

* 使用默认的端口

  ```
  sudo docker run --detach \
    --hostname gitlab.example.com \
    --env GITLAB_OMNIBUS_CONFIG="external_url 'http://gitlab.example.com'; gitlab_rails['lfs_enabled'] = true;" \
    --publish 443:443 --publish 80:80 --publish 22:22 \
    --name gitlab \
    --restart always \
    --volume $GITLAB_HOME/config:/etc/gitlab \
    --volume $GITLAB_HOME/logs:/var/log/gitlab \
    --volume $GITLAB_HOME/data:/var/opt/gitlab \
    --shm-size 256m \
    gitlab/gitlab-ee:<version>-ee.0
  ```

* 使用自定义端口【非80/443/22】

>容器端口配置：http服务在宿主机和容器都暴露在9999（新版本要求一致） 、 ssh服务在宿主机暴露于9998
>
>gitlab配置文件同步：external_url定义http服务地址(192.168.31.160为宿主机ip)，gitlab_shell_ssh_port定义ssh端口信息【此处配置重启可能丢失，需要在配置文件持久化】

```
docker run -d \
  --name gitlab \
  -p 9999:9999 \
  -p 9998:22 \
  -v $PWD/config:/etc/gitlab \
  -v $PWD/logs:/var/log/gitlab \
  -v $PWD/data:/var/opt/gitlab \
  -v /etc/localtime:/etc/localtime \
  --restart=always \
  -e GITLAB_OMNIBUS_CONFIG="external_url 'http://192.168.31.160:9999/'; gitlab_rails['gitlab_shell_ssh_port'] = 9998" \
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

* 服务器地址配置【将启动的配置持久化】  
  `external_url "http://192.168.31.160:9999/"`

* ssh端口配置

  `gitlab_rails['gitlab_shell_ssh_port'] = 9998`

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
git config --global   user.name simple0426
git config --global   user.email perfect_0426@qq.com
```

* 生产秘钥并上传至gitlab  
`ssh-keygen -t rsa -C "perfect_0426@qq.com"`
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
# 兼容新版本master--》main
git branch -m main
git push -u origin main
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