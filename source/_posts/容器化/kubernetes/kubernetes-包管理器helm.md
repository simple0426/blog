---
title: kubernetes-包管理器helm
date: 2020-03-08 12:21:44
tags:
  - helm
  - charts
categories: ['kubernetes']
---

# [helm简介](https://helm.sh/)

* 优点
  * 能够管理复杂的程序结构，将各种配置文件(service/deployment/configmap)作为整体管理
  * 使用模板后，资源文件可以复用、分享
  * 可以支持应用级别的版本管理：更新、回滚

* 核心概念
  * helm：helm二进制管理工具，主要用于k8s应用的创建、打包、发布、管理；类似linux上的apt/yum工具
  * Chart：描述一个应用的所有k8s资源文件集合；相当于linux中的的rpm、deb包
  * Release：Chart的部署实体，chart被部署后会生成一个对应的Release；相当于linux中的服务进程
  * 【v2版本】Tiller server：它是运行于k8s集群中的应用；helm的服务端程序，接收helm客户端请求，与kubernetes API Server交互，完成以下任务：
    - 监听来自helm客户端的请求
    - 安装应用，合并charts和Config为一个Release
    - 跟踪Release状态
    - 升级或卸载Release
  
* helm版本对比：
  * v2：有服务端tiller组件
  * v3：无服务端组件

* helm下载：

  * 官方版本信息：https://github.com/helm/helm/tags
  * 国内下载地址：https://mirrors.huaweicloud.com/helm/

## v2版本服务端安装

* rbac配置
  - `kubectl create serviceaccount --namespace kube-system tiller`
  - `kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`
* 初始化安装tiller server：`helm init --service-account tiller --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.1 --stable-repo-url https://apphub.aliyuncs.com --debug`
* 卸载：
  - `kubectl delete deploy tiller-deploy -n kube-system`
  - `helm reset`

## helm命令

* 集群认证：读取本地的kubeconfig用于k8s集群认证

* 管理命令

| 命令       | 详解                     | 范例                           |
| ---------- | ------------------------ | ------------------------------ |
| completion | 命令行补全               | source <(helm completion bash) |
| env        | 显示配置、数据和缓存目录 | helm env                       |
| plugin     | chart插件管理            | -                              |

# 应用管理

## [chart目录结构](https://helm.sh/docs/topics/charts/#the-chartyaml-file)

```
├── LICENSE # 许可证信息【可选】
├── README.md # README文件【可选】
├── Chart.yaml  # chart的元数据信息，包括名字、描述信息、版本、依赖等
├── charts/  # 当前chart依赖的其他chart文件
│   └── mysql-6.8.0.tgz
├── crds/ # 自定义资源
├── templates/  # 模板文件，用于生成kubernetes资源
│   ├── deployment.yaml
│   ├── _helpers.tpl  # 模板助手，会在模板文件中定义可公用的子模板
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt # 使用帮助文件；helm install部署后或helm status会显示此信息
│   ├── serviceaccount.yaml
│   └── service.yaml
└── values.yaml   # chart默认配置文件(模板文件中变量的值)，install/upgrade -f/--set会覆盖其中的值
```

### Chart.yaml

```
apiVersion: chart API版本 (必选项)
name: chart名称 (必选项)
version: chart项目版本 (必选项)
kubeVersion: 支持的k8s版本(可选项)
description: 项目描述信息 (可选项)
type: chart类型 (可选项)
keywords:
  - 项目涉及的关键词 (可选项)
home: 项目主页 (可选项)
sources:
  - 项目源代码 (可选项)
dependencies: # 项目依赖的其他chart (可选项)
  - name: chart名称 (如nginx)
    version: chart版本 ("1.2.3")
    repository: 仓库地址 ("https://example.com/charts") 或仓库别名 ("@repo-name")
    condition、tags、enabled: (可选项) 综合逻辑启用或停止此依赖
    alias: (可选项) chart别名，在多次使用chart依赖时可用
maintainers: # 项目维护信息(可选项)
  - name: 
    email: 
    url:
icon: 项目图标url (可选项).
appVersion: 应用版本信息 【比如redis、nginx的版本】(可选项)
deprecated: 是否被遗弃(可选项, boolean)
annotations:
  注解信息 (可选项).
```

* apiVersion版本（v1到v2）
  * v1的依赖定义使用requirements.yaml文件，v2的依赖使用chart.yaml文件中的dependencies字段
  * v2增加type字段，描述chart类型
* type：chart类型
  * application：这是一个应用，可以直接部署为Release
  * library：这是一个库，只能被application依赖使用

## chart管理

* 安装chart：helm install redis apphub/redis
* 下载chart(当前目录)：helm pull apphub/nginx      
* 查看chart内容：helm show all/chart/readme/values redis
* 管理chart依赖：helm dependency
  * 查看依赖：helm dependency list mychart
  * 下载依赖：helm dependency update mychart

## release管理

* 查看运行中的release列表：helm list
* 查看release部署历史：helm history web1
* 查看release详情：helm get  all/hooks/manifest/notes/values
* 查看release信息：helm status nginx
* 升级release：helm upgrade web1 --set image.tag="1.19" mychart
* 回滚到上一版本：helm rollback web1
* 卸载release：helm uninstall

## 安装和升级参数

* 使用方式：helm install/upgrade

  * 命令行传参(--set)：helm install db --set persistence.storageClass="example-nfs" microsoft/mysql
  * 配置文件方式(-f)：helm install db -f mysql.yal microsoft/mysql
    * 先使用show values获取chart默认值文件后修改：helm show values apphub/nginx > nginx.yaml

* 文件与命令使用形式对照

| yaml                    | set                      |
| ----------------------- | ------------------------ |
| name:value              | --set name=value         |
| name:<br>-a<br>-b<br>-c | --set name={a,b,c}       |
| servers:<br>-port:80    | --set servers[0].port=80 |
| image:<br>tag:"1.16"    | --set image.tag="1.16"   |

# chart仓库

## 公有hub

> 可使用web搜索

* 官方：https://hub.helm.sh/
  * 代码地址(开发者)：https://github.com/helm/charts
  * charts仓库：
    * stable：`https://kubernetes-charts.storage.googleapis.com`
    * incubator：`https://kubernetes-charts-incubator.storage.googleapis.com`
  * 微软镜像charts仓库【适合国内】
    * stable：`http://mirror.azure.cn/kubernetes/charts/`
    * incubator：`http://mirror.azure.cn/kubernetes/charts-incubator/`
* kubeapps：https://hub.kubeapps.com/
  * 代码地址(开发者)：https://github.com/kubeapps/kubeapps
  * charts仓库：`https://charts.bitnami.com/bitnami`
* 阿里云：https://developer.aliyun.com/hub/
  * 代码地址(开发者)：https://github.com/cloudnativeapp/charts
  * charts仓库：`https://apphub.aliyuncs.com`【适合国内】

## 命令使用

* 显示仓库列表：helm repo list
* 添加仓库：helm repo add name URL
* 更新仓库元数据：helm repo update
* 删除仓库：helm repo remove name
* 搜索仓库：helm search repo chart_name
* 搜索hub：helm search hub chart_name 
  * --endpoint 指定要搜索的hub地址（默认官方）
  * 可以搜索由[Monocular](https://github.com/helm/monocular)渲染的charts hub

## charts仓库-原生实现

* 创建索引文件：helm repo index my-repo --url http://49.232.17.71:8900

  * my-repo为多个chart压缩包所在目录
  * url为helm下载chart使用的主机地址

* 将chart压缩包目录(包含索引文件和chart压缩包)使用web服务发布出去

  ```
  docker run -d --name=chart-repo --restart=always \
  -v $(pwd)/my-repo:/usr/share/nginx/html \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf \
  -p 8900:80 nginx
  ```

  可以在nginx配置文件中开启文件目录索引，这样使用者就可以在web界面查看charts仓库包含的chart

  ```
  autoindex on;
  autoindex_exact_size off;
  autoindex_localtime on;
  charset utf-8;
  ```


## 开源chart仓库-[ChartMuseum](https://github.com/helm/chartmuseum)

> 此处使用在harbor中集成的chartmuseum仓库

### harbor集成chartmuseum功能

```fallback
sudo ./install.sh --with-chartmuseum
```

### [helm添加chartmuseum仓库](https://goharbor.io/docs/2.0.0/working-with-projects/working-with-images/managing-helm-charts/#working-with-chartmuseum-via-the-helm-cli)

* 不指定项目名称：将charts推送到library项目

  ```
  helm repo add --ca-file ca.crt --username=admin --password=Passw0rd myrepo https://xx.xx.xx.xx/chartrepo
  ```

  * --ca-file：指定自签名ca证书
  * --username/--password：指定harbor认证信息
  * chartrepo为harbor的chartmuseum接口

* 指定项目名称：将charts推送到指定项目

  ```
  helm repo add --ca-file ca.crt --username=admin --password=Passw0rd myrepo https://xx.xx.xx.xx/chartrepo/myproject
  ```

### 安装[helm-push插件](https://github.com/chartmuseum/helm-push)

* 在线方式：helm plugin命令

  ```
  helm plugin install https://github.com/chartmuseum/helm-push
  ```

* 离线方式：下载二进制文件并放在合适的位置

  * helm env：查看插件默认位置，如：HELM_PLUGINS="/root/.local/share/helm/plugins"
  * 创建默认位置目录：mkdir -p /root/.local/share/helm/plugins/helm-push
  * [下载二进制文件](https://github.com/chartmuseum/helm-push/releases)并解压，最终目录结构如下

  ```
  plugins/
  └── helm-push
      ├── bin
      │   └── helmpush
      ├── LICENSE
      └── plugin.yaml
  ```

### push推送charts到chartmuseum

* push .tgz压缩包：helm push mychart-0.1.0.tgz myrepo
* push chart目录：helm push mychart myrepo
* _push参数_：
  * --username/--password：认证信息
  * --ca-file=ca.crt：指定自签ca证书
  * --force/-f：强制推送【覆盖】
  * --version：自定义版本
    * latest：helm push --version="latest" mychart myrepo 
    * git版本：`helm push mychart/ --version="$(git log -1 --pretty=format:%h)" chartmuseum`

### 从chartmuseum安装charts

* 本地更新chart仓库：helm repo update
* 搜索charts：helm search repo mychart
* 安装charts：helm install web myrepo/library/mychart
  * --version：自定义版本

# [chart模板语法与开发](https://helm.sh/docs/chart_template_guide/)

* 创建一个chart：helm create mychart 
* 检查chart语法：helm lint mychart
* 渲染模板并输出：helm template sample mychart
* 打包chart为压缩包：helm package mychart
* 推送chart到chartmuseum【安装push插件】：helm push mychart myrepo

## 内置变量

> 内置值始终以大写字母开头

* `.Release.*`：引用Release中的内容

  ```
  {{ .Release.Name }}
  {{ .Release.Namespace }}
  ```

* `.Values.*`：引用values.yaml文件中的内容【也包括安装/升级时-f、--set设置的值】

  ```
  {{ .Values.image.repository }}
  ```

  ```
  --set image.repository="tomcat"
  ```

* `.Chart.*`：引用Chart.yaml文件中的内容

  ```
  {{ .Chart.Name }}-{{ .Chart.Version }} #chart名称和版本
  ```

* `.Capabilitie.*`：获取k8s信息

  ```
  {{ .Capabilities.KubeVersion }} #k8s版本
  ```

* `.Template.*`：模板信息

  ```
  {{ .Template.Name }} #当前模板文件路径
  ```

## [函数和管道](https://helm.sh/docs/chart_template_guide/function_list/)

* quote：将变量值加双引号

  ```
  {{ quote .Values.image.repository }}
  {{ .Values.image.repository|quote }}
  ```

* repeat：重复

  ```
  {{ .Template.Name|repeat 4 }}
  ```

* default：设置默认值

  ```
  {{ .Template.Name1|default "test" }}
  ```

* indent：缩进，左边空出空格【nindent，创建新行并缩进】

  ```
    toppings: |
    {{- range .Values.pizzaToppings }}
    {{ indent 2 .|title }}
    {{- end}}
  ```

## 流程控制

* if-else：本例中同时在左括号旁边使用短横线来消除语句左边的空白

  ```
    {{- if .Template.Name1 }}
    mug: true
    {{- else }}
    mug: false
    {{- end }}
  ```

* with：限制使用范围【本例中限定在Chart文件中，可以直接使用Name(等效于全局使用.Chart.Name)】

  ```
    {{- with .Chart }}
    name: {{  .Name }}
    {{- end }}
  ```

* range：循环【循环values文件中pizzaToppings】

  ```
    toppings: |
      {{- range .Values.pizzaToppings }}
      - {{ .|title|quote }}
      {{- end}}
  ```

* toYaml：将当前变量值(字典)转换为yaml格式【toYaml . | nindent 8】

  ```
  values文件定义
  # test1: {"a1": "cest", "b1": "dfsdfs"}
  test1:
    a1: cest
    b1: dfsdfs
  
  引用：
    test_dict:
      {{- toYaml .Values.test1|nindent 4 }}
  ```

## 变量

* with中引用其他变量的值

    ```
      {{- $relname := .Release.Name -}} #定义变量$relname
      {{- with .Chart }}
      name: {{  .Name }}
      relname: {{ $relname }} #引用变量
      {{- end }}
    ```

* 循环列表产生索引和变量值

  ```
    toppings: |
    {{- range $index,$value := .Values.pizzaToppings }}
      {{ $index }}: {{ $value }}
    {{- end }}
  ```

* 循环字典产生key和value

  ```
    {{- range $key,$val := .Values.service }}
    {{ $key }}: {{ $val|quote }}
    {{- end}}
  ```

## 命名模板

* define：

  * 在_helpers.tpl中定义命名模板【下划线开始的文件都不是k8s资源清单，可以定义模板】

  * 模板名称定义规范：chart_name.func_name，以chart名称为前缀

  * 模板功能注释：

    ```
    {{/* generate basic lables */}}
    {{- define "mychart.labels" }}
      labels:
        generator: helm
        date: {{ now|htmlDate }}
        chart: {{ .Chart.Name }}
        version: {{ .Chart.Version }}
    {{- end }}
    {{- define "mychart.app" }}
    app_name: {{ .Chart.Name }}
    app_version: "{{ .Chart.Version }}"
    {{- end }}
    ```

* template：引用子模板内容【使用点定义子模板可以继承当前模板内容，默认子模板没有作用域】

  ```
  metadata:
    name: {{ .Release.Name }}-configmap
    {{- template "mychart.labels" . }}
  ```

* include：引用子模板并调用其他函数进行格式化处理【相比template更推荐使用】

  ```
  metadata:
    labels:
  {{- include "mychart.app" .|indent 4 }}
  ```

