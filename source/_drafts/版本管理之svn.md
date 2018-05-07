---
title: 版本管理之svn
tags: ['svn']
categories:
---
# 安装与部署
[参考][1]

# 配置范例
## authz
```
[groups]
member = xianing,chengle,liuzuowei
[repos:/]
@member = rw
```
## passwd
```
[users]
xianing = svn1234
chengle = svn1234
liuzuowei = svn1234
```
## svnserve.conf
```
[general]
anon-access = none
auth-access = write
password-db = passwd
authz-db = authz
```
## hooks
```shell
#!/bin/sh
export LANG="en_US.UTF-8"
export LANGUAGE="en_US:en"
REPOS="$1"
REV="$2"
SVN_PATH=/usr/bin/svn
WEB_PATH=/usr/share/nginx/html
LOG_PATH=/tmp/svn_update.log
#此行已注释
#/usr/bin/svn update --username user --password password $WEB_PATH --no-auth-cache
echo "\n\n\n##########开始提交 " `date "+%Y-%m-%d %H:%M:%S"` '##################' >>$LOG_PATH
echo `whoami`,$REPOS,$REV >> $LOG_PATH
#注意将此行user和password改为你具体的user和password
sudo $SVN_PATH update --username like --password svn1234 $WEB_PATH --no-auth-cache >> $LOG_PATH
sudo chown -R www-data:www-data $WEB_PATH
```

[1]:https://yq.aliyun.com/articles/38802
