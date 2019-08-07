---
title: 博客应用hexo
date: 2018-2-26 13:43:05
tags: ['hexo', 'Next']
categories: ['skill']
---
# 安装
[参考](https://hexo.io/zh-cn/docs/)
## 安装前提
* Node.js
* Git

## 安装命令
`npm install -g hexo-cli`

# 建站
## hexo建站
* hexo init blog
* cd blog
* npm install

## 使用git版本控制
* 目录切换：cd blog
* 初始化：git init
* 关联远程：git remote add origin git@github.com:simple0426/blog.git
* 保持同步：git pull origin master

# 目录介绍
* scaffolds：模版文件夹。Hexo的模板是指在新建的markdown文件中默认填充的内容。
* source：资源文件夹是存放用户资源的地方。
    * 除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。
    * Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。
* themes：主题 文件夹。Hexo 会根据主题来生成静态页面。

# 标签插件
相当于在markdown文件中使用非markdown语法，  
它是的hexo私有语法，不能被markdown解析，但可以被hexo解析  
## 引入其他文章
语法：{% raw %}{% post_link path/to/file [title] %}{% endraw %}  
范例：{% raw %}{% post_link 系统管理/系统基础/shell编程 shell %}{% endraw %}  
注意：post_link使用的路径为相对于source/posts的路径  
## 引入代码
语法：{% raw %}{% include [title] [lang:language] path/to/file %}{% endraw %}  
范例：{% raw %}{% include_code python登录 lang:python login.py %}{% endraw %}  
目录设置：  

* 代码目录【code_dir】在主站config.yml中设置
* 【code_dir：code】表示路径为source/code
* 上述login.py表示的实际路径为source/code/login.py

# Next主题
## 下载
* cd blog
* git clone https://github.com/iissnan/hexo-theme-next themes/next

## 配置
### 站点配置
* theme: next【主题】
* language: zh-Hans【语言】
* order_by: -updated【以文章修改时间排序】

### 主题配置
* scheme: Mist【主题风格】
* 文章缩略显示

```
auto_excerpt:
  enable: true
```

* 菜单配置【主题配置】

```
 menu:
  home: / || home
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
```

## 分类与标签
* 创建categories和tags目录
* [分类页][2]
* [标签页][3]

## 搜索
### 安装
`npm install hexo-generator-searchdb`
### 配置
>主题配置

```yaml
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
# Local search
local_search:
  enable: true
```

# hexo命令
* hexo init：新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。
* hexo new 文章标题：新建一篇文章
* hexo new draft 文章标题：新建一篇草稿【或私密文章】
  - 在source/`_`drafts目录下建立相应文件
* hexo generate|g：生成静态文件。
* hexo server|s 
    - -p, --port    重设端口【默认4000】
    - -s, --static  只使用静态文件
    - -l, --log 启动日记记录，使用覆盖记录格式
    - --drafts 将草稿也显示【默认不显示草稿】
* hexo deploy|d：部署网站。
    - -g, --generate    部署之前预先生成静态文件
* hexo clean：清除缓存文件 (db.json) 和已生成的静态文件 (public)。
* hexo publish 文章标题：将草稿发布为文章

# FAQ
## 文章插入图片
* 主站配置：post_asset_folder:这个选项设置为true
* 安装插件：npm install hexo-asset-image
* 使用：
    - 运行hexo n “xxxx”来生成md博文时，/source/_posts文件夹内除了xxxx.md文件还有一个同名的文件夹 
    - 把图片复制进入文件夹内
    - markdown文章内引入`![](xxxx目录/图片)`

## github-pages配置
>使用gh-pages分支方式部署

### 安装插件
npm install hexo-deployer-git

### 主站配置
```
url: http://blog.unlazy.cn/
root: /
deploy:
- type: git
  repository: git@github.com:simple0426/blog.git
  branch: gh-pages
```

### 自定义域名
* 在source文件夹下建立CNAME文件
* CNAME文件内容：blog.unlazy.cn

### 部署github
`hexo d -g`

## public的压缩bug
* 现象：将windows下的public文件夹使用zip压缩后，传输至linux系统，文章标题为中文的的文件或目录名乱码
* 原因：这是zip格式的缺陷，所以目前并没有很完美的解决办法。[参考][1]
* 解决：使用tar方式传输public文件夹

[1]:http://blog.csdn.net/u012260238/article/details/52718416
[2]: https://github.com/iissnan/hexo-theme-next/wiki/%E5%88%9B%E5%BB%BA%E5%88%86%E7%B1%BB%E9%A1%B5%E9%9D%A2
[3]: https://github.com/iissnan/hexo-theme-next/wiki/%E5%88%9B%E5%BB%BA%E6%A0%87%E7%AD%BE%E4%BA%91%E9%A1%B5%E9%9D%A2

## 特殊字符处理
>特别注意的是小括号 ( ) 大括号 { } ,如果不小心写了,你执行hexo s –debug可能报一些莫名其妙的错误! 

```
! &#33; — 惊叹号Exclamation mark 
” &#34; &quot; 双引号Quotation mark 
# &#35; — 数字标志Number sign 
$ &#36; — 美元标志Dollar sign 
% &#37; — 百分号Percent sign 
& &#38; &amp; Ampersand 
‘ &#39; — 单引号Apostrophe 
( &#40; — 小括号左边部分Left parenthesis 
) &#41; — 小括号右边部分Right parenthesis 
* &#42; — 星号Asterisk 
+ &#43; — 加号Plus sign 
< &#60; &lt; 小于号Less than 
= &#61; — 等于符号Equals sign 
> &#62; &gt; 大于号Greater than 
? &#63; — 问号Question mark 
@ &#64; — Commercial at 
[ &#91; --- 中括号左边部分Left square bracket 
\ &#92; --- 反斜杠Reverse solidus (backslash) 
] &#93; — 中括号右边部分Right square bracket 
{ &#123; — 大括号左边部分Left curly brace 
| &#124; — 竖线Vertical bar 
} &#125; — 大括号右边部分Right curly brace 
```