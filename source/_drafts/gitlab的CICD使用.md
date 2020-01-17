---
title: gitlab的CICD使用
tags:
    - gitlab
    - runner
categories: ['CICD']
---
# gitlab-ci安装与配置
gitlab CI服务最好不要部署于gitlab服务器上【因为自动化构建会很消耗资源】
## 安装docker
docker是自动化构建的基础环境，当然自动化构建也可以直接利用CI服务器的shell环境

## [安装gitlab-runner](https://docs.gitlab.com/runner/install/linux-repository.html)
>gitlab-runner是部署于gitlab CI服务器上的代理服务，用于接收gitlab服务器上发送的构建指令并执行构建操作
    
- curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
- sudo apt-get install gitlab-runner
- gitlab-runner status

## 设置权限
使gitlab-runner运行的用户可以执行docker命令

## 将ci-runner注册到gitlab
* 执行命令：gitlab-runner register
* 注册输入内容
    - gitlab服务器地址【管理员界面--》runners】
    - 注册秘钥【管理员界面--》runners】
    - 描述信息：说明这个runner用途
    - 标签信息：标识这个runner，可以让.gitlab-ci.yml中的job使用指定的runner运行任务
    - 执行环境：命令执行的环境，可以是docker【python27、python34、maven等环境】、shell等

# gitlab-ci使用
* 项目源代码的根目录下创建.gitlab-ci.yml文件
```
# 定义构建的步骤
stages:
  - build
  - test

# 定义执行的任务
job1:
  stage: test   # 任务和构建步骤绑定
  tags:             # 定义构建的基础环境(即runner的标签信息)
    - shell 
  script:           # 定义构建执行的命令
    - echo "I am job1"
    - echo "I am in test stage"

job2:
  stage: build
  tags:
    - shell
  script:
    - echo "I am job2"
    - echo "I am in build stage"
```
* 默认，项目代码库有变动，CICD--》pipeline就会执行

# gitlab-ci应用范例
* [学习示例代码](https://github.com/imooc-course/docker-cloud-flask-demo)
* gitlab-ci.yml示例
```
stages:
  - style
  - test
  - deploy
  - release

pep8:
  stage: style
  script:
    - pip install tox
    - tox -e pep8               #执行python pep8代码检查
  tags:
    - python2.7
  except:                            #代码库有版本(tag)变动时不执行
    - tags

unittest-py27:
  stage: test
  script:
    - pip install tox
    - tox -e py27               #执行python2.7环境下的单元测试
  tags:
    - python2.7
  except:
    - tags

unittest-py34:
  stage: test
  script:
    - pip install tox
    - tox -e py34               #执行python3.4环境下的单元测试
  tags:
    - python3.4
  except:
    - tags

docker-deploy:
  stage: deploy
  script:
    - docker build -t flask-demo .
    - if [ $(docker ps -qf "name=web") ];then docker rm -f web;fi
    - docker run -id -p 5000:5000 --name web flask-demo 
  tags:
    - shell
  only:                             #仅当master分支有变动时才执行此任务
    - master

docker-build:
  stage: release
  script:
    - docker build -t registry.cn-hangzhou.aliyuncs.com/simple00426/flask-demo:$CI_COMMIT_TAG . 
    - docker push registry.cn-hangzhou.aliyuncs.com/simple00426/flask-demo:$CI_COMMIT_TAG
  tags:
    - shell
  only:                         #仅当代码库有版本(tag)变动时才执行此任务
    - tags
```

# gitlab-ci实践建议
* 保护master分支，只允许其他分支merge，不允许直接push
* 分支合并到master时，必须通过pipeline检测
* 在项目的readme文件中添加项目的pipeline状态信息【settings-》CICD--》General pipelines--》Pipeline status】
