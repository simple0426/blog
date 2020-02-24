---
title: docker镜像仓库-harbor
date: 2018-03-20 16:39:21
tags: ['harbor']
categories: ['docker']
---
# [harbor架构][5]
## 存储
* 镜像的存储harbor使用的是官方的docker registry服务去完成；至于registry是用本地存储或者s3都是可以的，harbor的功能是在此之上提供用户权限管理、镜像复制等功能
* harbor镜像的复制是通过docker registry 的API去拷贝

## 架构
![](http://simple0426-blog.oss-cn-beijing.aliyuncs.com/schema.jpg)

* 主要组件包括proxy，他是一个nginx前端代理，主要是分发前端页面ui访问和镜像上传和下载流量，上图中通过深蓝色先标识；
* ui提供了一个web管理页面，当然还包括了一个前端页面和后端API，底层使用mysql数据库；
* registry是镜像仓库，负责存储镜像文件，当镜像上传完毕后通过hook通知ui创建repository，上图通过红色线标识，当然registry的token认证也是通过ui组件完成；
* adminserver是系统的配置管理中心附带检查存储用量，ui和jobserver启动时候回需要加载adminserver的配置，通过灰色线标识；
* jobsevice是负责镜像复制工作的，他和registry通信，从一个registry pull镜像然后push到另一个registry，并记录job_log，上图通过紫色线标识；
* log是日志汇总组件，通过docker的log-driver把日志汇总到一起，通过浅蓝色线条标识。

## harbor使用
### 用户管理
* 项目中的角色：
    - 访客(guest)：对项目只读
    - 开发者(developer)：对项目有读写权限
    - 维护者(master)：高于开发者的权限，同时拥有权限：扫描镜像、查看复制任务、删除镜像
    - 项目管理员（projectadmin）：用户新建项目时，默认具有的权限；项目管理员可以添加或删除项目成员
* 系统级别角色
    - 系统管理员（sysadmin）：具有最高权限
    - 匿名用户（anonymous）：未登录的用户；默认，只能对公开项目有读权限

### 项目管理
项目管理是系统最主要的一个功能模块，项目是一组镜像仓库的逻辑集合，是权限管理和资源管理的单元划分。一个项目下面有多个镜像仓库，并且关联多个不同角色的成员，镜像复制也是基于项目的，通过添加复制规则，可以将项目下面的镜像从一个harbor迁移到另一个harbor，并且可以通过日志查看复制过程，并有retry机制。

### 配置管理和日志查询
* 配置管理主要是配置harbor的认证模式，企业内部使用，通常都是对接到公司LDAP上面，当然harbor也支持数据库认证；还可以设置token的有效时间。用户对镜像的pull和push操作都可以被harbor记录下来，这样为排查文件提供了重要手段。 
* 当然harbor功能不止我上面说的，譬如harbor集成了clair镜像扫描功能，它是cereos开发的一款漏洞扫描工具，可以检查镜像操作系统以及上面安装包是否与已知不安全的包版本相匹配，从而提高镜像安全性

### harbor高可用
![](http://simple0426-blog.oss-cn-beijing.aliyuncs.com/ha.jpg)

# [安装][6]
* 安装前置条件
    - [docker][1]
    - [docker-compose][2]
* [下载离线安装包][3]
* 参数配置(harbor.yml)
    - hostname: 对外发布的主机名
    - harbor_admin_password：web管理界面的密码【默认：admin/Harbor12345】
    - database：本地数据库密码
        + password: root123
    - data_volume: 镜像等数据的存储位置
* hostname配置要点
    - 当配置hostname为内网地址时，虽然登陆地址可以通过地址映射使用外网地址；但是认证时，服务端回传客户端的认证地址，依然是内网地址造成无法登陆认证。
    - 当配置hostname为外网地址时，由于外网带宽限制，内网服务器不能高速的进行镜像传输
    - 当配置hostname为域名时，采用dns分离解析，内网dns解析使用自定义dns并解析为内网地址，外网dns解析使用公网dns并解析为公网ip。
* harbor安装
    - 包装命令：sudo ./install.sh
* web访问：http://ip:80

# [管理][4]
## harbor集群管理 
>主要使用docker-compose命令

* docker-compose up -d 创建和启动容器【后台】
* docker-compose down 停止和删除容器
* docker-compose start 启动容器及服务
* docker-compose stop 停止容器及服务

## 仓库使用
* 客户端设置私有镜像库地址(/etc/docker/daemon.json)：insecure-registries
* 添加客户端认证：docker login -u username -p password reg.mydomain.com
* 上传镜像
    - docker tag SOURCE_IMAGE[:TAG] harbor.zj-hf.cn/mytest/IMAGE[:TAG]
    - docker push harbor.zj-hf.cn/mytest/IMAGE[:TAG]
* 下载镜像
    - docker login
    - docker pull harbor.zj-hf.cn/mytest/registry:latest
* 删除镜像
    - web界面删除镜像
    - 设置垃圾清理（Garbage Collection）：删除文件系统上的数据

## 其他功能
* 默认开启自注册功能【但是密码重置功能需要设置邮件参数】
* 项目可以设置公开或私有，公开项目读写无需登录认证，私有项目读写需要认证
* 新建项目默认自己为管理员，可以添加其他成员为开发者【可以上传或下载镜像】或访问者【只能下载镜像】
* 非系统管理员用户登录，只能看到有权限的项目和日志
* 用户创建项目权限控制，默认是everyone（所有人），也可以设置为adminonly（只能管理员）

[1]:https://yq.aliyun.com/articles/110806
[2]: https://docs.docker.com/compose/install/
[3]: https://github.com/vmware/harbor/releases
[4]: https://github.com/vmware/harbor/blob/master/docs/user_guide.md
[5]: http://blog.csdn.net/u010278923/article/details/77941995
[6]: https://github.com/vmware/harbor/blob/master/docs/installation_guide.md
