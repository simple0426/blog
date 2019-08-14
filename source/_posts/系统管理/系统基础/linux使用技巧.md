---
title: linux使用技巧
date: 2019-07-02 14:27:29
tags:
    - swap
    - ubuntu
    - 全互联
categories: ['linux']
---
# 文件swap
## 创建空文件
*  fallocate -l 4G /home/swapfile 
*  truncate -s 2G /swapfile 
*  dd if=/dev/zero of=/swapfile bs=4096 count=512k  

## 格式化与挂载
```
chmod 600 /home/swapfile
mkswap /home/swapfile
echo "/home/swapfile none swap    sw              0       0" >> /etc/fstab
swapon -a
free -m
```
# ubuntu初始化
* 创建用户： useradd -s /bin/bash -mr -G sudo test  
* 安装常用软件： sudo apt-get install vim ntp htop tree lrzsz sysv-rc-conf -y
* 时间和时区设置

```
service ntp start
sysv-rc-conf ntp on
ntpdate -u time1.aliyun.com
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "Asia/Shanghai" > /etc/timezone
```

* 更改默认编辑器：update-alternatives --set editor /usr/bin/vim.basic
* 更改文件描述符大小

```
echo "*   hard   nofile  65536" >> /etc/security/limits.conf
echo "*   soft   nofile  65536" >> /etc/security/limits.conf
```

# 批量删除小文件
1. 建立空目录：mkdir /tmp/blank
2. 使用rsync删除： rsync --delete-before -d /tmp/blank/ /var/spool/postfix/maildrop/ 
3. rsync解释：-d 只同步目录【不递归同步】 --delete-before 同步前删除相关文件

# 批量修改文件
```
touch {a..g}.html
for file in $(ls);do mv $file ${file/html/HTML};done
```
# 判断变量是否为数字
* grep过滤：`grep -E -w "[0-9]+"`
* expr相加：`expr $a + 1`
* 变量删除：`${b//[0-9]/}`

# 1+...+100求和
* shell下for循环：for((i=0;i<=100;i++));do ((sum+=i));done;echo $sum
* awk下for循环：{% raw %}awk 'BEGIN{{for(i=0;i<=100;i++)sum+=i}print sum}'{% endraw %}
* seq与bc：seq -s + 100|bc
* seq与awk：seq 100|awk '{sum+=$0}END{print sum}'

# 随机字符串
* awk随机数：awk 'BEGIN{srand();print rand()}'
* uuid命令：uuidgen
* md5命令：md5sum
* openssl命令：openssl rand -base64 32

# [多主机全互联][ssh-conn]
0. [通过root创建普通用户][create-user]，并把它加到sudo中；修改ansible hosts使用普通用户连接目标主机。
1. 在目标主机产生秘钥： ansible all -m shell -a 'ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa' 
2. 将所有主机公钥取至本地： ansible all -m fetch -a "src=~/.ssh/id_rsa.pub dest=./tmp"  
3. [将所有公钥合并为一个文件][merge-pubkey]，并命名为authorized_keys
4. 将authorized_keys传送至目标主机： ansible all -m copy -a "src=authorized_keys dest=~/.ssh/ mode=0600" 
5. 修改目标主机ssh客户端配置，使首次登录的用户没有yes/no的询问【可选】： ansible all -m lineinfile -a "path=/etc/ssh/ssh_config line='\t\tStrictHostKeyChecking no'" -b 

# socat命令
* 类似于netcat的工具，名字来由是” Socket CAT”，可以看作是netcat的N倍加强版  
* 主要特点就是在两个数据流之间建立通道

## haproxy应用
* help信息：echo ""|sudo socat stdio /var/run/haproxy.sock
* 状态查看：echo "show stat"|sudo socat stdio /var/run/haproxy.sock
* 启停server：echo "enable server service_marketinfo/172.17.134.60"|sudo socat stdio /var/run/haproxy.sock

[ssh-conn]: https://github.com/simple0426/sysadm/tree/master/ansible/playbook/ssh_conn_all
[merge-pubkey]: https://github.com/simple0426/sysadm/blob/master/ansible/playbook/ssh_conn_all/join_sshkey.py
[create-user]: https://github.com/simple0426/sysadm/blob/master/ansible/playbook/ssh_conn_all/create_user.yml

