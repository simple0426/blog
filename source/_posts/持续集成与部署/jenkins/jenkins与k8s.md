---
title: jenkins与k8s
tags:
  - jenkins
  - k8s
categories:
  - CICD
date: 2020-08-03 19:35:10
---


# 在k8s中部署jenkins

* 自定义资源文件地址：https://github.com/simple0426/sysadm/tree/master/kubernetes/jenkins
* 使用helm安装：helm search repo jenkins

# 安装插件

> 修改使用国内镜像地址后重启jenkins【删除pod】
>
> 安装插件后重启jenkins，使部分插件生效(如kubernetes插件)

* [Git](https://plugins.jenkins.io/git)：拉取代码
* [Git Parameter](https://plugins.jenkins.io/git-parameter)：git参数化构建，支持在项目中使用分支、tag、revision拉取代码
* [Pipeline](https://plugins.jenkins.io/workflow-aggregator)：流水线
* [Kubernetes](https://plugins.jenkins.io/kubernetes)：在k8s中动态创建agent
* [Config File Provider](https://plugins.jenkins.io/config-file-provider)：支持在web界面加载配置文件，并传递给job workspace

# 在k8s中动态创建agent

* 插件：[Kubernetes](https://plugins.jenkins.io/kubernetes)

* jenkins master：可以运行在k8s内，也可以运行在k8s外

## kubernetes插件配置

* 配置位置：manage nodes and clouds==》kubernetes
* 配置选项：根据jenkins-master的部署位置不同，有如下方式：

### jenkins-master部署在k8s集群中

* 名称：任意值，用于区分不同的k8s集群
* Kubernetes地址：kubernetes service的地址，例如：https://kubernetes.default
* Kubernetes 命名空间：代理pod的运行的命名空间
* jenkins地址：jenkins的service地址，例如：http://jenkins.ops

### jenkins-master部署在k8s集群外

* 名称：任意值，用于区分不同的k8s集群

* Kubernetes地址：k8s集群外部地址，例如：https://192.168.31.201:6443

* Kubernetes 服务证书 key：kubernetes ca证书，例如：/etc/kubernetes/pki/ca.crt

* Kubernetes 命名空间：代理pod的运行的命名空间

* 凭据(credential)：

  * 设置证书为 secret text类型

  * 凭据的值为在k8s集群中创建的代理pod运行时使用的serviceaccount的token

  * token获取（代理pod-RBAC设置）

    ```
    # 创建pod服务账号
    kubectl create serviceaccount jenkins-slave -n ops 
    # 给pod服务账号绑定权限
    kubectl create clusterrolebinding jenkins-slave --clusterrole cluster-admin --serviceaccount=ops:jenkins-slave 
    # 获取pod服务账号的token
    kubectl describe secret -n ops $(kubectl get secret -n ops|awk '/jenkins-slave/{print $1}')|awk '/^token/{print $2}'
    ```

* Jenkins 地址：jenkins的web接口地址，例如：http://192.168.31.250:7080/

## 自定义agent镜像

agent pod中默认包含一个jnlp的容器用于和jenkins master通信，可以增加一个容器用于执行特定的任务 

此处即为增加特定任务镜像【包含maven和kubectl命令，以用于在k8s中部署资源】

```
FROM centos:7
RUN yum install -y java-1.8.0-openjdk maven curl git libtool-ltdl-devel && \ 
    yum clean all && \
    rm -rf /var/cache/yum/* && \
    mkdir -p /usr/share/jenkins
COPY settings.xml /etc/maven/settings.xml
COPY kubectl /usr/bin/
```

# 使用jenkins部署k8s服务

* jenkins的Credentials中新建
  * 连接gitlab的认证信息
  * 连接harbor的认证信息
* Config File Management（config provider插件功能）中新增kubeconfig文件

* k8s部署时
  * 部署所需资源文件k8s-yaml/\*在项目的根目录
  * 需要保证有相关的命名空间
  * 命名空间中含有连接harbor的secret信息

```
 // harbor地址
def registry = "www.unlazy.cn"
// gitlab地址
def git_url = "http://192.168.31.250:9080" 
// 项目分组
def project = "test"
// 项目名称
def app_name = "tomcat-java-demo"
// 项目打包生成的镜像，BUILD_NUMBER为jenkins的内置变量，表示构建序列号
def image_name = "${registry}/${project}/${app_name}:${BUILD_NUMBER}"
// 项目git地址
def git_address = "${git_url}/${project}/${app_name}.git"
// 部署deployment时需要使用的dockerregistry类型secret
def k8s_dockerregistry_secret = "registry-pull-secret"
// docker命令推送镜像使用的用户名和密码信息
def dockerregistry_auth = "3e8f2333-99a6-4367-8cf4-9b4bd5c2317f"
// 拉取git项目的用户名和密码信息
def git_auth = "e4cf739c-9ede-4469-822b-7e58d97c1869"
// 执行kubectl命令使用的kubeconfig文件
def k8s_kubeconfig = "35c55219-287f-41b9-b77b-2c4abaace04e"

pipeline {
  agent {
    // 此处使用k8s的动态agent执行构建任务
    kubernetes {
        yaml """
apiVersion: v1
kind: Pod
spec:
  containers: #特定任务容器
  - name: maven
    image: www.unlazy.cn/library/maven-kubectl:v1
    imagePullPolicy: Always
    command:
    - sleep
    args:
    - infinity
    volumeMounts: #容器内可以执行docker命令用于镜像构建
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-sock
        mountPath: /var/run/docker.sock
      - name: maven-cache
        mountPath: /root/.m2
  volumes:
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
    - name: maven-cache
      hostPath:
        path: /tmp/m2
"""
        defaultContainer 'maven'
        }
      
      }
      // 参数化构建
    parameters {    
        gitParameter branch: '', branchFilter: '.*', defaultValue: 'master', description: '选择发布的分支', name: 'Branch', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'PT_BRANCH'
        choice (choices: ['1', '3', '5', '7'], description: '副本数', name: 'ReplicaCount')
        choice (choices: ['dev','test','prod'], description: '命名空间', name: 'Namespace')
    }
    stages {
        stage('拉取代码'){
            steps {
                checkout([$class: 'GitSCM', 
                branches: [[name: "${params.Branch}"]], 
                doGenerateSubmoduleConfigurations: false, 
                extensions: [], submoduleCfg: [], 
                userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]
                ])
            }
        }

        stage('代码编译'){
           steps {
             sh """
                mvn clean package -Dmaven.test.skip=true
                """ 
           }
        }

        stage('构建镜像'){
           steps {
                withCredentials([usernamePassword(credentialsId: "${dockerregistry_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
                sh """
                  echo '
                    FROM tomcat:8.5.57
                    ADD target/*.war /usr/local/tomcat/webapps/ROOT.war
                  ' > Dockerfile
                  docker build -t ${image_name} .
                  docker login -u ${username} -p '${password}' ${registry}
                  docker push ${image_name}
                """
                }
           } 
        }
        stage('部署到K8S平台'){
          steps {
              configFileProvider([configFile(fileId: "${k8s_kubeconfig}", targetLocation: "admin.kubeconfig")]){
                sh """
                  sed -i 's#IMAGE_NAME#${image_name}#' k8s-yaml/deployment.yaml
                  sed -i 's#SECRET_NAME#${k8s_dockerregistry_secret}#' k8s-yaml/deployment.yaml
                  sed -i 's#REPLICAS#${ReplicaCount}#' k8s-yaml/deployment.yaml
                  kubectl apply -f k8s-yaml/ -n ${Namespace} --kubeconfig=admin.kubeconfig
                """
              }
          }
        }
    }
}
```

