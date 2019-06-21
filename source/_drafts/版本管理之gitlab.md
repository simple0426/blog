---
title: 版本管理之gitlab
tags: gitlab
categories:
---
# [软件安装][gitlab-repo]
## ubuntu16.04
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
## Centos7
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
# 软件配置
>【/etc/gitlab/gitlab.rb】

* 服务器地址配置【在邮件通知中也会使用此地址】  
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

# [邮箱功能测试][gitlab-mail]
* 进入控制台：gitlab-rails console
* 测试：Notify.test_email('destination_email@address.com', 'Message Subject', 'Message Body').deliver_now

# 常用命令
* 查看gitlab版本：`cat /opt/gitlab/version-manifest.json` 
* 变更配置：`sudo gitlab-ctl reconfigure`
* 创建备份：`gitlab-rake gitlab:backup:create`
* 恢复备份(BACKUP后为在备份目录【默认/var/opt/gitlab/backups/】产生文件名中数字)：
`gitlab-rake gitlab:backup:restore BACKUP=1507879010_2017_10_13`
* 服务状态操作(查/起/停)：`gitlab-ctl status|start|stop`

# 客户端配置
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

[gitlab-repo]: https://mirrors.tuna.tsinghua.edu.cn/help/gitlab-ce/
[gitlab-mail]: https://docs.gitlab.com/omnibus/settings/smtp.html#testing-the-smtp-configuration