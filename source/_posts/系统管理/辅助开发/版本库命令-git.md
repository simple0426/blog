---
title: 版本库命令-git
date: 2018-02-26 16:18:04
tags: ['git']
categories: ['linux']
---
# 状态迁移
版本库=》工作区=》暂存区=》版本库

# clean
>从工作区中清理未追踪的文件或目录
## 参数
* -d 清除未追踪的目录
* -f 强制删除未追踪的文件或目录
* -x 删除为追踪的文件，使目录保持干净的原始状态
## 应用
清除未追踪的文件或目录：git clean -xdf 

# reset
## 指定路径或文件的回退
* 还原暂存区内容，即撤销 git add操作
* git reset HEAD 【file】

## 版本库整体回退
* 在git中用HEAD表示当前版本,上个版本HEAD^,往上100个版本HEAD~100
* 回退到上个版本：git reset --hard HEAD^
* 回退到特定版本：git reset --hard 2ff6b08 

# checkout
>切换分支或恢复工作区为版本库文件【撤销工作区变更】
## 分支切换
* 切换：git checkout 【branch_name】
* 创建+切换：git checkout -b 【branch_name】 【start_point】
    - 从某一分支创建新分支，不写默认为当前分支

# tag
* git tag：查看标签列表
    - git show tag_name：查看标签变更 
* git tag tag_name：使用当前commit打标签
* git tag -d tag_name：删除本地标签
* -m：tag注释
* git tag tag_name commit_id：针对某个commit打标签

# add/rm 
* add：提交工作区变更到暂存区
* rm：删除版本库文件

# config
>设置或获取版本库或全局配置
## 全局设置
* --global 
* 有此参数对此电脑下的所有仓库有效，形成的参数文件在用户主目录的.gitconfig文件中
* 无此参数，只对某个仓库有效，形成的参数文件在仓库.git/config文件中
## 配置别名
* git config --global alias.st status
* git config --global alias.last 'log -1'
## 设置用户及邮件
* git config --global user.name "何静奇" 
* git config --global user.email "hejingqi@quxiu8.com"

## 设置代理

```
// 查看当前代理设置
git config --global http.proxy

// 设置当前代理为 http://127.0.0.1:1080 或 socket5://127.0.0.1:1080
git config --global http.proxy 'http://127.0.0.1:1080'
git config --global https.proxy 'http://127.0.0.1:1080'
git config --global http.proxy 'socks5://127.0.0.1:1080'
git config --global https.proxy 'socks5://127.0.0.1:1080'

// 删除代理
删除 proxy git config --global --unset http.proxy
```

# push/pull

* push：推送本地版本库对象到远程
    - tag：
        + 推送指定标签：git push origin 1.4
        + 推送所有标签：git push origin --tags
        + 删除远程标签：git push origin --delete tag <tagname>
    - branch：
        + 推送分支：git push <远程主机名> <本地分支名>  <远程分支名>
            * 如果远程分支被省略，如上则表示将本地分支推送到与之存在追踪关系的远程分支
        + 删除远程分支：git push origin --delete 【branchName】
        + 范例：
            * git push origin master：origin为主机名，在配置文件中设置
            * git push -u origin master：如果当前分支与多个主机存在追踪关系，则可以使用 -u 参数指定一个默认主机，这样后面就可以不加任何参数使用git push
* pull：拉取远程版本库对象到本地

# other
* diff：对比工作区和版本库中某个文件的差异
* commit：提交暂存区至版本库
* status：版本库状态查看
* reflog：查看命令操作历史
* clone：克隆分支
    - 将远程库克隆至本地【默认克隆master分支】：git clone git@github.com:simple0426/Spoon-Knife.git
    - -b dev 克隆dev分支
* log：历史变更查看
    - --pretty=oneline
* branch：分支管理
    - git branch 查看分支
        + git branch -r 查看远程分支
    - git branch 【name】 创建分支
    - git branch -d 【name】 删除分支
    - git branch --set-upstream dev origin/dev  关联本地分支和远程分支
* merge：分支合并
    - 合并某分支到当前分支
    - git merge dev
* init：将普通目录初始化为版本库
* .gitignore 配置.gitignore文件忽略部分文件或目录的变更

# FAQ
## git中文数字类型乱码
git config –global core.quotepath false

## 强制放弃本地修改和增加的文件
>此时，本地修改“多于”远程库

* git checkout . 放弃修改
* git clean -xdf 删除未追踪文件

## 强制本地与远程版本库保持一致
>此时本地与远程库的均有修改

* git fetch --all  取回远程端所有【分支】修改，但不和本地合并
* git reset --hard origin/master 
    - 将版本强制设置到origin/master【远程master】这个分支的最新代码
    - 放弃本地所有修改等操作，保持与远程代码库的一致
* git pull 与远程分支保持同步

# remote
* 变更本地关联的远程库：git remote set-url origin URL

# 故障收集
* git bash乱码：git config --global core.quotepath false

# 清除污点提交或大文件
## 前言
* git库提交内容有时候会包含大文件或密码这样的隐私信息，当使用正常的git提交等措施删除文件或密码信息后，仍然可以从历史提交记录中找到这些内容。对于删除的大文件而言，它会占用仓库存储空间；对于密码等隐私内容来说，隐私内容依然存在。
* git自带的工具git-filter-branch可以清除历史提交信息，但是操作比较复杂
* 可以使用第三方工具[BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/)方便的处理文件和密码的历史提交信息

## 工具介绍
* bfg是一个jar文件，[下载地址](https://search.maven.org/classic/remote_content?g=com.madgag&a=bfg&v=LATEST)
* 以下命令bfg就是【java -jar bfg.jar】的别名

## 操作步骤
* 克隆仓库元数据：

    ```
    git clone --mirror git@github.com:my-account/my-repo.git
    ```

* 移除文件【可以是下列任意一个操作】：
    - 删除所有文件id_rsa或id_dsa：`bfg --delete-files id_{dsa,rsa}  my-repo.git`
    - 删除目录.git：`bfg --delete-folders .git --delete-files .git  --no-blob-protection  my-repo.git`
    - 删除大于1M的文件：`bfg --strip-blobs-bigger-than 1M  my-repo.git`
    - 删除密码等文本：`bfg --replace-text banned.txt  my-repo.git`
        + 在指定的文件【比如：banned.txt】中一行定义一个要删除的密码等隐私信息，执行删除后，密码信息会被【REMOVED】关键词替换
        + 比如在banned.txt中包含原来密码信息【426hx118】，则在执行操作时会将历史记录中=的【426hx118】替换为REMOVED
    
* 将本地修改推送到远程
    1. `cd my-repo.git`
    2. `git reflog expire --expire=now --all && git gc --prune=now --aggressive`
    3. git push
    
* 将远程元数据信息同步到本地
    1. `cd my-repo`
    2. `git pull`
