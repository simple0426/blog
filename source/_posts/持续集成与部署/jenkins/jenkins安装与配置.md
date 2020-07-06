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
    - 软件源：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/` `debian-stable|redhat-stable`
    - deb包安装失败
        + sudo apt-get update # 更新
        + sudo apt-get -f install # 解决依赖关系
        + sudo dpkg -i xxx.deb # 重新安装
    - 安装错误
        + 错误：ERROR: No Java executable found in current PATH: /bin:/usr/bin:/sbin:/usr/sbin
        + 解决：ln -s /usr/local/jdk1.8.0_241/bin/java /usr/bin/

### docker部署
* 启动参数
```
docker run --name jenkins \
  -u root -d \
  -p 80:8080 -p 50000:50000 \
  -v /data/jenkins/:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  -v /usr/local/maven:/usr/local/maven \
  -v /usr/local/jdk:/usr/local/jdk \
  -v /etc/localtime:/etc/localtime \
  -e JAVA_OPTS="-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  --restart=always \
  jenkins/jenkins:lts
```
* 参数详解：
    - --name：设置容器名称
    - -u：以root启动，防止出现权限问题
    - -d：后台运行
    - -p 7005:8080：映射服务端口【宿主机7005-》容器8080】
    - -p 50000:50000：agent连接master的端口
    - -v /data/jenkins/:/var/jenkins_home：jenkins主目录持久化存储
    - -v /var/run/docker.sock:/var/run/docker.sock：确保jenkins容器内可以操作宿主机的docker
    - -v /usr/local/maven:/usr/local/maven：挂载宿主机maven到jenkins容器
    - -v /usr/local/jdk:/usr/local/jdk：挂载宿主机jdk到容器
    - -v /etc/localtime:/etc/localtime：设置容器时间
    - -e JAVA_OPTS="-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"：关闭csrf保护
* csrf设置：由于jenkins2.0版本默认开启CSRF且不能关闭，所以需要在启动jenkins服务时关闭csrf
    - [官方说明](https://www.jenkins.io/doc/upgrade-guide/2.222/#always-enabled-csrf-protection)
    - [网友设置](https://www.cnblogs.com/kazihuo/p/12937071.html)

# web设置
* 访问地址：http://JENKINS_URL:8080
    - 初始化密码：/var/jenkins_home/secrets/initialAdminPassword
* 使用国内镜像的步骤
    - 修改插件更新中心URL：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`
    - 修改json配置【/var/jenkins_home/updates/default.json】
    ```
    sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && \
    sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
    ```
    - 重启jenkins：`docker restart jenkins`
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
