---
title: docker-k8s的CICD实践预览
tags:
  - gitlab
  - harbor
  - jenkins
categories:
  - CICD
date: 2020-01-18 12:10:00
---

# [docker/k8s的CICD实现](https://blog.csdn.net/AIfurture/article/details/100668771)
![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/docker-k8s-cicd.png)

1. 开发人员从镜像仓库harbor中拉取基础镜像，对应用进行容器化开发
2. 开发人员提交代码到代码仓库gitlab中
3. gitlab收到代码提交请求后通过webhook方式触发jenkins
4. jenkins收到请求后对代码进行打包，生产可执行程序(如，marven打包)
5. 打包完成后，基于dockerfile生成docker images
6. docker images产生后上传到镜像仓库harbor
7. 通过jenkins的pipeline在kubernetes的测试、生产环境拉取镜像、部署应用
