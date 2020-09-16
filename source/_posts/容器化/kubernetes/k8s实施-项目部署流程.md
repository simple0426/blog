---
title: k8s实施-项目部署流程.md
tags:
  - wordpress
  - 运维
categories:
  - kubernetes
date: 2020-07-29 00:01:14
---


# 运维架构

传统部署方式

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/deploy-os.jpg)

k8s部署方式

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/deploy-k8s.jpg)

# k8s部署流程

1. 制作镜像
2. 控制器管理pod
3. pod数据持久化
4. 暴露、发布应用：service、ingress
5. 监控、日志：prometheus+granfa、elk

## 制作镜像

* 镜像分类
  * 基础镜像：ubuntu、centos
  * 环境镜像：程序运行环境，如：java、python、php、tomat
  * 项目镜像：代码打包进环境镜像
* 制作镜像
  * 编写Dockerfile
  * build构建
  * 推送到镜像仓库

## pod控制器

* Deployment：无状态部署，如web、微服务、API
* Statefulset：有状态部署，例如数据库集群、zk、etcd
* Daemonset：守护进程部署，例如监控agent、日志agent
* job&cronjob：批处理，例如数据库备份、邮件通知

## pod数据持久化

* 初始化数据、配置文件
* 临时数据：临时缓存、多个容器共享数据
* 业务数据：运行过程中产生的数据

## 应用发布

* 内部暴露：service
* 外部暴露：ingress、集群外的nginx/haproxy

# 部署范例-wordpress

wordpress需要同时用到nginx和php-fpm进行解析，所以采用一个pod中包含nginx和php(也包含wordpress源码)两个容器的方式在k8s中部署；根据使用的php镜像不同，可分两种形式部署：

* 直接使用docker hub中的wordpress[wordpress:fpm]镜像
  * 此种方式可以直接将wordpress的运行目录[/var/www/html]通过共享存储的形式挂载到nginx容器
* 下载wordpress源码，将源码打包进[php:fpm]镜像【以下也为此种形式部署】
  * 此种方式在包含源码php-fpm镜像启动容器后(postStart)，需要将源码复制到共享存储，然后nginx容器将项目发布出去

## 下载并解压wordpress

修改wp-config.php中的数据库配置，范例如下

```
// ** MySQL 设置 - 具体信息来自您正在使用的主机 ** //
/** WordPress数据库的名称 */
define( 'DB_NAME', 'wp' );

/** MySQL数据库用户名 */
define( 'DB_USER', 'root' );

/** MySQL数据库密码 */
define( 'DB_PASSWORD', 'QMnY1CjjOm' );

/** MySQL主机 */
define( 'DB_HOST', 'java-demo-db-mysql.default' );
```

## 制作镜像

```
FROM php:5.6.36-fpm
RUN docker-php-ext-install mysqli pdo pdo_mysql
COPY wordpress /app
```

docker build -t 'wordpress:v3' .

## 创建资源文件

* namespace

  ```
  apiVersion: v1
  kind: Namespace
  metadata:
    name: php
  ```

* configmap【配置nginx访问php-fpm】

  ```
  kind: ConfigMap
  apiVersion: v1
  metadata:
    name: nginx-conf
    namespace: php
  data:
    default.conf: |
      server {
          listen       80;
          listen  [::]:80;
          server_name _;
          root /var/www/html;
          index index.php index.html;
          location / {
            try_files $uri $uri/ /index.php =404;
          }
          location ~ \.php$ {
              root  /var/www/html;
              fastcgi_pass   127.0.0.1:9000;
              fastcgi_index  index.php;
              include        fastcgi_params;
              fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
          }
      }
  ```

* deployment

  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: php-demo
    namespace: php
  spec:
    replicas: 1
    selector:
      matchLabels:
        project: www
        app: php-demo
    template:
      metadata:
        labels:
          project: www
          app: php-demo
      spec:
        nodeName: "node01"
        volumes:
        - name: app-volume
          emptyDir: {}
        - name: config-volume
          configMap:
            name: nginx-conf
            items:
            - key: default.conf
              path: default.conf
        containers:
        - name: nginx 
          image: nginx:latest
          ports:
          - containerPort: 80
            name: web
            protocol: TCP
          resources:
            requests:
              cpu: 0.5
              memory: 256Mi
            limits:
              cpu: 1
              memory: 512Mi
          volumeMounts:
          - mountPath: /var/www/html
            name: app-volume   
          - mountPath: /etc/nginx/conf.d
            name: config-volume
        - name: wordpress 
          image: wordpress:v3
          ports:
          - containerPort: 9000
            name: web
            protocol: TCP       
          resources:
            requests:
              cpu: 0.5
              memory: 256Mi
            limits:
              cpu: 1
              memory: 512Mi
          lifecycle:
            postStart:
              exec:
                command: ["sh", "-c", "cp -a /app/* /var/www/html"]
          volumeMounts:
          - mountPath: /var/www/html
            name: app-volume 
  ```

* service

  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: php-demo 
    namespace: php
  spec:
    selector:
      project: www
      app: php-demo
    ports:
    - name: web
      port: 80
      targetPort: 80
  #  type: NodePort
  ```

* ingress

  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: php-demo 
    namespace: php
  spec:
    rules:
      - host: php.ctnrs.com
        http:
          paths:
          - path: /
            backend:
              serviceName: php-demo 
              servicePort: 80
  ```

## 安装及访问

  http://php.ctnrs.com

