---
title: 部署工具fabric
tags:
  - fabric
date: 2018-04-20 16:52:38
categories: ['linux']
---
# 命令行选项
* -l 显示任务名称
* -f 指定fab入口文件，默认为fabfile.py
* -H 指定目标主机，多台主机以逗号分隔【当fab文件中只定义排除的主机列表时有效】
* -x 排除指定主机【当fab文件中定义主机列表时有效，且列表形式必须一致，比如主机名，ip或同时包含用户名】
* -u 目标主机用户名
* -p 目标主机密码
* -I 打开连接目标主机时的交互式提示
* -P 以异步并行方式运行多主机任务，默认为串行运行
* -R 指定role（角色），以角色名区分不同业务组设备
* --skip-bad-hosts 跳过不可达主机
* -t 设置目标主机连接超时时间（秒）
* -T 设置远程主机命令执行超时时间（秒）
* -w 当命令执行失败，发出警告，而非默认中止任务。
* --set=KEY=VALUE,...   以逗号分隔设置环境变量
* -z 设置并行执行时的进程数量

# env设置
* env.hosts 定义目标主机列表，可以是ip或主机名
    - 此时可以只定义主机列表，形如：['172.17.20.12', '172.17.20.13']，此时所有主机将使用已定义好的用户名和密码连接目标主机
    - 当各主机的用户名不同时，在hosts里定义用户名，形如：['zj-ops@172.17.20.12', 'zj-ops@172.17.20.13']
* env.exclude_hosts 排除指定主机
* env.user 定义目标主机用户名
    - 仅限于所有主机使用相同用户名
    - 当没有定义user变量，同时hosts里也没有user选项时，将设置user为本地用户
* env.password 定义目标主机密码
    - 仅限于所有主机使用相同密码
    - 当没有定义password变量，同时passwords里没有定义密码时，将出现交互式的密码输入提示
* env.passwords 定义目标主机密码组
    - passwords变量中的key中user、host、port必须同时设置，否则将依然提示输入密码
    - 范例：env.passwords = {'user1@172.17.20.12:22': '123', 'user2@172.17.20.13:22': '456', 'zj-ops@2.2.2.2:22': 'abc'}
* env.gateway 定义网关设备【堡垒机、跳板机】，
    - 网关设备的认证信息在passwords变量中定义
    - 当堡垒机的用户名与应用主机不同时，应当在此添加，如：env.gateway = 'zj-ops@2.2.2.2'
* env.roledefs 将主机分组
    - 定义：env.roledefs = {'test1': ['zj-ops@1.1.1.1'], 'test2': ['zj-web@1.1.1.2'], 'test3': ['zj-ops@1.1.1.1', 'zj-web@1.1.1.2']}
    - 使用：使用roles装饰器，表明仅某个主机组执行此任务，例如：@roles('test1')

# 常用API
## 命令
* with 上下文管理，与python的with类似
    - 导入：from fabric.context_managers import *
* run 运行远程命令
* cd 远程切换目录
* local 运行本地命令
* lcd 本地切换目录
* sudo 使用sudo执行远程命令
* put 上传本地文件至远程主机
* get 下载远程文件到本地主机
* prompt  获得用户输入信息，如：prompt('please input user password:')
    - 导入：from fabric.contrib.console import confirm, prompt
* confirm  获得提示信息确认，如：confirm('Test failed,Continue[Y/N]?')
    - 导入：from fabric.contrib.console import confirm, prompt
* yellow|green|red 输出带颜色字体
    - 导入：from fabric.colors import *

## 装饰器
* @task 定义任务可以通过命令行【fab -l】看到
* @runs_once 定义任务仅运行一次
* @roles('test1', 'test2') 定义任务仅被主机组执行
* @hosts('zj-ops@172.17.20.12') 定义任务仅被此主机执行
* @parallel 在多个主机并行执行任务【默认串行】

# 范例
```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-
# @Time    : 2018/4/16 17:08
# @Author  : simple0426
# @Email   : istyle.simple@gmail.com
# @File    : fabfile.py
# @Software: PyCharm
# @desc    : 功能如下
# 1.查看系统信息
# 2.部署nginx代码
# 3.回滚nginx代码
# 4.更新nginx配置【更新成功后自动重启，更新失败自动回滚配置】
# 5.查看nginx日志【分主机分项目】
# 6.登陆nginx主机【分主机】

from fabric.api import *
from fabric.context_managers import *
from fabric.contrib.console import prompt
from fabric.colors import *
import os, shutil
import json

env.hosts = ['zj-ops@1.1.1.1', 'zj-web@1.1.1.2']
env.gateway = 'zj-ops@2.2.2.2'
env.passwords = {
    'zj-ops@1.1.1.1:22': '1213456',
    'zj-web@1.1.1.2:22': '1213456',
    'zj-ops@2.2.2.2:22': '1213456',
}
env.roledefs = {
    'web1': ['zj-ops@1.1.1.1'],
    'web2': ['zj-web@1.1.1.2'],
    'webserver': ['zj-ops@1.1.1.1', 'zj-web@1.1.1.2']
}
project_dict = {
    'fop': {'name': 'fop', 'log': 'fop.abc.cn.log'},
    'kop': {'name': 'kop', 'log': 'kop.abc.cn.log'},
    'check': {'name': 'cloud-h5', 'log': 'check.fff.cn.log'},
    'open': {'name': 'cloud', 'log': 'open.fff.cn.log'},
    'www': {'name': 'kingold-web', 'log': 'www.fff.cn.log'},
}


@task
@parallel
def df():
    '''查看所有主机的磁盘信息'''
    print yellow(run('df -h'))

@task
@roles('web1')
def free():
    '''只用于查看web1的内存'''
    run('free -m')

@task
@hosts('zj-web@1.1.1.2')
def uptime():
    '''只用于查看1.1.1.2的负载信息'''
    print green(run('uptime'))

def delete_files(filename):
    '''如果文件或目录存在则删除'''
    dest_file = os.path.join(os.path.expanduser('~'), 'deploy', filename)
    if os.path.exists(dest_file) and os.path.isdir(dest_file):
        shutil.rmtree(dest_file)
    elif os.path.exists(dest_file) and os.path.isfile(dest_file):
        os.remove(dest_file)
    else:
        pass

@runs_once
def tar_code():
    '''本地代码打包'''
    project = prompt('请输入要部署的项目:').strip()
    print yellow('本地代码打包')
    if project not in project_dict.keys():
        abort('请输入正确的项目名称')
    project_name = project_dict[project]['name']
    delete_files(project_name)
    delete_files('%s.tar.gz' % project_name)
    with lcd('~/deploy'):
        local('git clone git@10.10.10.164:kingold_app/%s.git' %  project_name, capture=True)
        local('tar czf %s.tar.gz %s' % (project_name, project_name), capture=True)
    return project

def backup_code(project):
    '''远程备份代码'''
    print yellow('远程备份代码')
    if project not in project_dict.keys():
        abort('请输入正确的项目名称')
    project_name = project_dict[project]['name']
    with cd('~/nginx'):
        with settings(warn_only=True):
            result = run('rsync -az --delete %s/ %s.bak/' % (project_name, project_name), quiet=True)
        if result.succeeded:
            print green('代码备份成功')

def push_code(project):
    """上传代码【传输压缩包】"""
    print yellow('本地上传代码')
    if project not in project_dict.keys():
        abort('请输入正确的项目名称')
    project_name = project_dict[project]['name']
    result = put('~/deploy/%s.tar.gz' % project_name, '~/nginx/%s.tar.gz' % project_name)
    if result.failed:
        abort("上传代码失败！")
    else:
        with cd('~/nginx'):
            run('tar xzf %s.tar.gz' % project_name, quiet=True)
            print green('代码部署成功')

@task
def updateconf():
    '''更新配置文件目录'''
    # 备份
    with cd('nginx'):
        run('rsync -az --delete conf/ conf.bak/')
    # 上传
    result = put('~/deploy/conf', '~/nginx')
    with settings(warn_only=True) and cd('nginx'):
        if result.failed:
            # 回滚
            run('rsync -az --delete conf.bak/ conf/')
            abort('上传失败')
        else:
            res_conf = sudo('~/nginx/sbin/nginx -t', quiet=True)
            # 配置文件有效，重启nginx
            if res_conf.succeeded:
                nginx_count = run('ps aux|grep nginx|grep -v grep|wc -l')
                if nginx_count.stdout == '0':
                    sudo('~/nginx/sbin/nginx')
                    print green('nginx启动成功')
                else:
                    sudo('~/nginx/sbin/nginx -s reload')
                    print green('nginx重启成功')
            else:
                # 配置文件无效，回滚配置
                run('rsync -az --delete conf.bak/ conf/')
                print red(res_conf.stdout)

@task
def rollback():
    '''服务回滚'''
    project = prompt('请输入要回滚的项目:').strip()
    print yellow('代码回滚')
    if project not in project_dict.keys():
        abort('请输入正确的项目名称')
    project_name = project_dict[project]['name']
    with settings(warn_only=True):
        result = run('rsync -az --delete %s.bak/ %s/' % (project_name, project_name), quiet=True)
        if result.succeeded:
            print green('代码回滚成功')

@task
def deploy():
    '''服务部署'''
    project = tar_code()
    backup_code(project)
    push_code(project)

@task
@runs_once
def host():
    '''设置登陆主机'''
    print green(json.dumps(env.hosts, indent=4))
    host = prompt('请输入要登陆的主机')
    env.exclude_hosts.extend(env.hosts)
    for hostname in env.hosts:
        if host in hostname:
            env.exclude_hosts.remove(hostname)

@task
def login():
    '''远程登陆[需设置登陆主机]'''
    open_shell()

@task
def log():
    '''查看日志[需设置登陆主机]'''
    print green(json.dumps(project_dict.keys()))
    project = prompt('请输入要部署的项目:').strip()
    project_log = project_dict[project]['log']
    with cd('~/nginx/logs'):
        run('tail -10f %s' % project_log)
```

# QAQ
* 安装：pip install fabric
* [参考][1]

[1]:http://www.cnblogs.com/aslongas/p/5961144.html
