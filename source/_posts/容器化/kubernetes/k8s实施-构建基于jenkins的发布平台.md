---
title: k8s实施-构建基于jenkins的发布平台
tags:
  - jenkins
  - k8s
categories:
  - kubernetes
date: 2020-08-03 19:35:10
---

# 发布流程设计

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/k8s-cicd.png)

1. 开发人员从镜像仓库harbor中拉取基础镜像，对应用进行容器化开发
2. 开发人员提交代码到代码仓库gitlab中
3. gitlab收到代码提交请求后通过webhook方式触发jenkins
4. jenkins收到请求后对代码进行打包，生产可执行程序(如，marven打包)
5. 打包完成后，基于dockerfile生成docker images
6. docker images产生后上传到镜像仓库harbor
7. 通过jenkins的pipeline在kubernetes的测试、生产环境拉取镜像、部署应用

# 构建基础环境

* k8s(ingress controller、storage class)
* helm v3【helm方式部署k8s服务】
* gitlab
* jenkins
* harbor，并启用chart存储功能
* mysql（微服务数据库）
* eureka（微服务注册中心）

# jenkins master部署

## 在k8s中部署jenkins

* 自定义资源文件方式：https://github.com/simple0426/sysadm/tree/master/kubernetes/jenkins
* helm安装方式：helm search repo jenkins

## 安装插件

> 修改使用国内镜像地址后重启jenkins【删除pod】
>
> 安装插件后重启jenkins，使部分插件生效(如kubernetes插件)

* [Git](https://plugins.jenkins.io/git)：拉取代码
* [Git Parameter](https://plugins.jenkins.io/git-parameter)：git参数化构建，支持在项目中使用分支、tag、revision拉取代码
* [Pipeline](https://plugins.jenkins.io/workflow-aggregator)：流水线
* [Kubernetes](https://plugins.jenkins.io/kubernetes)：在k8s中动态创建agent
* [Config File Provider](https://plugins.jenkins.io/config-file-provider)：支持在web界面加载配置文件，并传递给job workspace
* [Extended Choice Parameter](https://plugins.jenkins.io/extended-choice-parameter)：支持多项选择【内置choice插件只支持单选】

## jenkins设置

* Credentials中新建
  * 连接gitlab的认证信息
  * 连接harbor的认证信息

* Config File Management（config provider插件功能）中新增kubeconfig文件

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

此处即为增加特定任务镜像，包含的工具如下：

```
1. 项目构建：maven、jdk
2. 镜像构建：docker build/push【docker通过在pod中挂载宿主机docker的二进制、socket完成】
3. 资源文件部署：kubectl、helm
```

此处将镜像设置为公开镜像，方便动态创建pod时无需认证即可拉取镜像

```
FROM centos:7
LABEL maintainer perfect_0426@qq.com
RUN rm -rf /etc/yum.repos.d/* && \
    curl -Lo /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo && \
    curl -Lo /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo && \
    sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo && \
    yum clean all && yum makecache && \ 
    yum install -y java-1.8.0-openjdk maven curl git libtool-ltdl-devel && \ 
    yum clean all && \
    rm -rf /var/cache/yum/* && \
    mkdir -p /usr/share/jenkins
COPY settings.xml /etc/maven/settings.xml
COPY kubectl helm /usr/bin/
```

# 部署k8s中的服务-kubectl

项目地址：https://github.com/simple0426/tomcat-java-demo，需要注意的内容如下：
* 部署所需资源文件在k8s-yaml目录
* 需要保证有相关的命名空间【dev、test、prod】
* 部署的命名空间中含有连接harbor的secret信息
* 将ingress域名配置到主机hosts中

部署所需Jenkinsfile如下：

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

# 部署k8s中的微服务-helm

## 微服务部署要点

k8s中不同微服务的不同点

```
1. 命名空间
2. 服务名称
3. 服务端口
4. 标签
5. 镜像地址
6. ingress域名
7. 代码地址
8. 副本数
9. 环境变量
```

通过模板参数化适配不同微服务

* jenkins创建的pipeline，能够接收参数匹配多个项目
* helm创建的chart模板，可以接收自定义变量匹配多个项目

## Jenkinsfile部署

项目地址：https://github.com/simple0426/simple-microservice，需要注意的内容如下

* 在harbor的chartmuseum中导入模板ms：https://github.com/simple0426/simple-microservice/tree/dev3/ms
* 需要先部署数据库、注册中心Eureka；此处的jenkins只负责gateway、order、portal、product、stock项目部署
* 将ingress域名配置到主机hosts中

部署项目的Jenkinsfile

```
#!/usr/bin/env groovy
// 所需插件: Git Parameter/Git/Pipeline/Config File Provider/kubernetes/Extended Choice Parameter
// 公共
def registry = "www.unlazy.cn"
def git_url = "http://192.168.31.250:9080" 
// 项目
def project = "microservice"
def app_name = "microservice"

def git_address = "${git_url}/${project}/${app_name}.git"
def gateway_domain_name = "gateway.ctnrs.com"
def portal_domain_name = "portal.ctnrs.com"
// 认证
def image_pull_secret = "registry-pull-secret"
def harbor_registry_auth = "d0d9b572-80e3-4347-a627-0486038b99f9"
def git_auth = "32c9f547-1a26-4fbc-9e30-617e217334b6"
// ConfigFileProvider ID
def k8s_auth = "9b502bf9-4d4d-405d-8559-8f9c691e072e"

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
    image: www.unlazy.cn/library/agent-k8s:v1
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
    parameters {
        gitParameter branch: '', branchFilter: '.*', defaultValue: 'master', description: '选择发布的分支', name: 'Branch', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'PT_BRANCH'        
        extendedChoice defaultValue: 'none', description: '选择发布的微服务', \
          multiSelectDelimiter: ',', name: 'Service', type: 'PT_CHECKBOX', \
          value: 'gateway-service:9999,portal-service:8080,product-service:8010,order-service:8020,stock-service:8030'
        choice (choices: ['ms', 'demo'], description: '部署模板', name: 'Template')
        choice (choices: ['1', '3', '5', '7'], description: '副本数', name: 'ReplicaCount')
        choice (choices: ['ms'], description: '命名空间', name: 'Namespace')
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
        stage('代码编译') {
            // 编译指定服务
            steps {
                sh """
                  mvn clean package -Dmaven.test.skip=true
                """
            }
        }
        stage('构建镜像') {
          steps {
              withCredentials([usernamePassword(credentialsId: "${harbor_registry_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
                sh """
                 docker login -u ${username} -p '${password}' ${registry}
                 for service in \$(echo ${Service} |sed 's/,/ /g'); do
                    service_name=\${service%:*}
                    image_name=${registry}/${project}/\${service_name}:${BUILD_NUMBER}
                    cd \${service_name}
                    if ls |grep biz &>/dev/null; then
                        cd \${service_name}-biz
                    fi
                    docker build -t \${image_name} .
                    docker push \${image_name}
                    cd ${WORKSPACE}
                  done
                """
                configFileProvider([configFile(fileId: "${k8s_auth}", targetLocation: "admin.kubeconfig")]){
                    sh """
                    # 添加镜像拉取认证
                    kubectl create secret docker-registry ${image_pull_secret} --docker-username=${username} --docker-password=${password} --docker-server=${registry} -n ${Namespace} --kubeconfig admin.kubeconfig |true
                    # 添加私有chart仓库
                    helm repo add  --username ${username} --password ${password} myrepo https://${registry}/chartrepo/${project}
                    """
                }
              }
          }
        }
        stage('Helm部署到K8S') {
          steps {
              sh """
              common_args="-n ${Namespace} --kubeconfig admin.kubeconfig"
              
              for service in  \$(echo ${Service} |sed 's/,/ /g'); do
                service_name=\${service%:*}
                service_port=\${service#*:}
                image=${registry}/${project}/\${service_name}
                tag=${BUILD_NUMBER}
                helm_args="\${service_name} --set image.repository=\${image} --set image.tag=\${tag} --set replicaCount=${replicaCount} --set imagePullSecrets[0].name=${image_pull_secret} --set service.targetPort=\${service_port} myrepo/${Template}"

                # 判断是否为新部署
                if helm history \${service_name} \${common_args} &>/dev/null;then
                  action=upgrade
                else
                  action=install
                fi

                # 针对服务启用ingress
                if [ \${service_name} == "gateway-service" ]; then
                  helm \${action} \${helm_args} \
                  --set ingress.enabled=true \
                  --set ingress.host=${gateway_domain_name} \
                   \${common_args}
                elif [ \${service_name} == "portal-service" ]; then
                  helm \${action} \${helm_args} \
                  --set ingress.enabled=true \
                  --set ingress.host=${portal_domain_name} \
                   \${common_args}
                else
                  helm \${action} \${helm_args} \${common_args}
                fi
              done
              # 查看Pod状态
              sleep 10
              kubectl get pods \${common_args}
              """
          }
        }
    }
}
```

