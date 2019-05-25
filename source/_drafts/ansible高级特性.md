---
title: ansible高级特性
tags:
    - vault
    - 加密
categories: ['ansible']
---
# ansible-vault
* ansible-vault是用于加密结构化数据[json或yaml]文件的命令
* ansible-vault命令参数
    - create 创建加密文件
    - edit 编辑加密文件
    - encrypt 加密文件
    - decrypt 解密文件
    - view 查看加密文件内容
    - rekey 变更加密密码
* 使用方式【命令ansible或ansible-playbook】
    - --ask-vault-pass 交互式出入加密密码
    - --vault-password-file=xx 提供加密密码文件
* 使用范例
    - ansible 127.0.0.1 -e "@vars.yml" -m debug  -a "msg={{ key3 }}" --ask-vault-pass
    - ansible-playbook test.yml --vault-password-file=password.txt
