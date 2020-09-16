---
title: jenkins自动化-gitlab
tags:
  - jenkins
  - gitlab
categories:
  - CICD
date: 2020-02-11 22:07:48
---

# 创建项目
* 类型：pipeline
* 名称：spring-hello-demo

# 触发器设置
* 以webhook方式
* 设置token信息，如：123456
* webhook触发的URL(gitlab中设置)：JENKINS_URL/job/spring-hello-demo/build?token=TOKEN_NAME
    - 如：http://192.168.2.162:7005/job/spring-hello/build?token=123456

# pipeline设置
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/pipeline-scm.jpg)

* 方式：pipeline script from SCM
* SCM：git
* Repository URL：包含项目源码、Jenkinsfile的源码库地址，[示例代码](https://github.com/simple0426/spring-hello-demo.git)
* Credentials：jenkins连接gitlab的认证信息，可使用【用户名+密码】形式
* Branches to build：构建的分支
* 脚本路径：Jenkinsfile文件位置

# jenkins设置
* 在用户(如root)中创建API Token，此token用于：gitlab连接jenkins
* 全局安全设置：
    - 访问控制-》授权策略-》【启用】匿名用户具有读权限
    - 跨站请求伪造保护：【禁用】防止跨站点请求伪造

# gitlab设置
* 设置：settings-》integrations
    - URL：jenkins中的hook地址，如：http://192.168.2.162:7005/job/spring-hello-demo/build?token=123456
    - Secret Token：jenkins中创建的API Token
    - 【启用】Push events
    - 【禁用】Enable SSL verification
    - 点击【add webhook】按钮
* 测试：Project Hooks-》test-》push events
* 错误处理
    - 错误：Url is blocked: Requests to the local network are not allowed
    - 解决【admin area】
        + 位置：settings-》network-》Outbound requests
        + 设置：【启用】Allow requests to the local network from web hooks and services
        + 设置：允许哪些局域网的webhook请求

# 代码详解
## Jenkinsfile
```
pipeline{
    agent any
    stages{
        stage('build'){
            steps{
                script{
                    echo "WORKSPACE: ${env.WORKSPACE}" //工作目录
                    echo "NODE_NAME: ${env.NODE_NAME}" //节点名称
                    if ("${env.NODE_NAME}" == "master"){ //如果是在master上，则执行脚本
                       sh "sh build-prod.sh"
                    }
                }
            }
        }
    }
}
```
## build-prod.sh
```
#!/bin/sh

# 服务端口
targetPort=8080
# 镜像版本号
vendor=1.0.0
# 项目名称
projectName=spring-hello-demo

# 软件打包
cd $WORKSPACE
mvn clean package -D skipTests

# 删除基于镜像的所有容器
if [ $(docker ps -aqf "name=$projectName") ];then 
    docker rm -f $projectName
fi

# 删除旧镜像
if [ $(docker images -qf "reference=$projectName:$vendor") ];then 
    docker rmi -f $projectName:$vendor
fi
 
# 创建镜像
docker build -t $projectName:$vendor .
 
# 基于镜像启动容器
docker run --name $projectName -d -p $targetPort:$targetPort $projectName:$vendor
```
## Dockerfile
```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
