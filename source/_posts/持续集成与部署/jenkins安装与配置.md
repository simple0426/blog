---
title: jenkins安装与配置
date: 2020-02-10 15:12:23
tags:
    - jenkins
categories: ['CICD']
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

# web设置
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
* 节点管理【添加slave节点(SSH Slaves插件)】
    - 远程工作目录
    - 设置启动方式（ssh）
    - 设置环境变量：JAVA_HOME

# 工具集成
## 集成maven
* 安装apache maven，并配置环境变量
* 插件：Maven Integration plugin
* 全局工具配置
    - maven配置--》settings文件
    - Maven--》Maven name、MAVEN_HOME设置
* jenkinsfile中使用（此处使用tools配置加载maven）
```
pipeline{
    agent any

    tools{
        maven 'maven3.6.3'
    }

    stages{
        stage('Build'){
            steps{
                sh 'echo Build stage...'
                sh 'java -version'
            }
        }

        stage('Test'){
            steps{
                sh 'echo Test stage...'
                sh 'mvn --version'
            }
        }
    }
}
```

## 集成docker
* 插件
    - Docker Commons Plugin：给各种docker插件提供基础的API
    - docker-build-step：在job中执行docker命令
* 配置(docker-build-step插件)：系统配置--》Docker Builder--》unix:///var/run/docker.sock【确保jenkinis用户对此socket有读权限】
* 使用方式-tools：通过在全局工具配置中定义特定版本的docker
```
pipeline {
  agent any
  tools {
    'org.jenkinsci.plugins.docker.commons.tools.DockerTool' 'docker1903'
  }
  stages {
    stage('foo') {
      steps {
        sh "docker version"
      }
    }
  }
}
```
* 使用方式-agent：加载PATH变量下默认的docker
```
pipeline{
    agent{
        docker{
            image 'maven:3.6.3-jdk-8'
            label 'agent01'
        }
    }
    stages{
        stage('Build'){
            steps{
                sh 'echo jdk version...'
                sh 'java -version'
            }
        }
        stage('Test'){
            steps{
                sh 'echo maven version...'
                sh 'mvn --version'
            }
        }
        stage('Deploy'){
            steps{
                sh 'echo Deploy stage...'
            }
        }
    }
}
```
* 并行使用多个docker镜像(parallel)
```
stage('Build'){
    parallel{
        stage('Front End Build: Angular'){
            agent{
                docker{
                    image 'liumiaocn/angular:8.3.8'
                }
            }
            steps{
                sh 'echo Front End Build stage...'
                sh 'ng --version'
            }
        }
        stage('Back End build: Marven'){
            agent{
                docker{
                    image 'maven:3.6.3-jdk-8'
                }
            }
            steps{
                sh 'echo Back End Build stage ...'
                sh 'mvn --version'
            }
        }
    }
}
```
* dockerfile中定义构建环境
    - 默认，需要在多分支pipeline或SCM-pipeline中使用
    - 为了手动测试，可以在jenkins宿主机的绝对目录定义dockerfile
```
agent{
    dockerfile{
        filename 'Dockerfile'
        dir '/tmp'
        label 'agent01'
    }
}
```
