---
title: linux编辑器-vim
tags:
categories:
---
# vim
## 选中多行内容  
1. 先按__Shift + v__后,使用上下键选中多行操作  
2. 先按__ctrl + v__后，使用上下左右键选中区块操作  

## 多行注释与删除  
1. ctrl+v进入块编辑模式  
2. 使用上下键选中行首  
3. 【大写i】插入#等字符进行注释或d键删除  
4. Esc退出，wq保存

## tab和空格设置
```
"显示行号
set nu
"tab长度为4个空格
set ts=4
"把tab显示成空格
set expandtab
"自动缩进
set autoindent
"搜索高亮
set hlsearch
```
