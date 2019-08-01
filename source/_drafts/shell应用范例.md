---
title: shell应用范例
tags:
categories:
---
# linux技巧
## 批量修改文件
```
touch {a..g}.html
for file in $(ls);do mv $file ${file/html/HTML};done
```
## 判断变量是否为数字
* grep过滤：`grep -E -w "[0-9]+"`
* expr相加：`expr $a + 1`
* 变量删除：`${b//[0-9]/}`

## 菜单制作
```
#!/bin/bash
menu_1 () {
    echo -e "
    ====================
    1.\033[32minstall lamp\033[0m
    2.\033[32minstall lnmp\033[0m
    3.\033[31mexit\033[0m
    please install your choice:
    "
}
menu_1
read num1
echo $num1
```
## 文件md5检测
>通过使用md5sum命令检测目录下文件内容是否有变化

```
#!/bin/bash
export PATH=/bin:/sbin:/usr/bin:/usr/sbin:~/bin
. /etc/init.d/functions

check_dir=$1
dir_name=$(echo $check_dir|sed 's#/#_#g')

usage () {
    echo "Usage:$0 <dir> [create|recreate|check]"
    exit 1
}

create_md5 () {
    find $check_dir -type f|xargs md5sum > /tmp/md5${dir_name}.db    
}

check_md5 () {
    md5sum --quiet -c /tmp/md5${dir_name}.db >/tmp/error${dir_name} 2>/dev/null
    err_num=$(wc -l /tmp/error${dir_name}|awk '{print $1}')
    if [ ${err_num} -ne 0 ];then
        action "$(cat /tmp/error${dir_name})" /bin/false
    else
        action "All files is the right!" /bin/true
    fi
}

if test $# -ne 2 -o ! -e $check_dir;then
    usage
    exit
fi

case $2 in
    create|recreate)
        create_md5
        if [ -f /tmp/md5${dir_name}.db ];then
            action "Md5 data is $2" /bin/true
        else
            action "DB file not created" /bin/false
        fi
    ;;
    check)
        if [ -f /tmp/md5${dir_name}.db ];then
            check_md5
        else
            action "DB file not exists" /bin/false
        fi
    ;;
    *)
        usage
        exit 1
    ;;
esac
```
## 1+...+100求和
* shell下for循环：for((i=0;i<=100;i++));do ((sum+=i));done;echo $sum
* awk下for循环：awk 'BEGIN{{for(i=0;i<=100;i++)sum+=i}print sum}'
* seq与bc：seq -s + 100|bc
* seq与awk：seq 100|awk '{sum+=$0}END{print sum}'

## 产生随机数
>输入英文单词，产生1-100之间的随机数

```
#!/bin/bash
while true # 输入循环
do
    read -p "Please input your name:" name    # 交互式输入
    echo "$name"|grep -E -q -w "^exit|^quit" && exit # 定义退出码
    echo "$name"|grep -E -q -w "[a-zA-Z]+" || continue # 非字母则继续输入循环
    [ ${#name} -eq 0 ] && continue # 为空继续输入循环
    grep -E -w "$name" random.list 2>/dev/null && continue # 内容已存在则继续输入循环
    while true # 随机数循环
    do
        random=$(awk 'BEGIN{srand();val=int(rand()*100);print val}') # 产生随机数
        grep -E -w -q "$random" random.list 2>/dev/null && continue # 随机数存在则继续随机数循环产生新随机数
        printf "$name\t$random\n"
        printf "$name\t$random\n" >> random.list && break # 保存输入和随机数
    done
done
```
## 99乘法表
```
#!/bin/bash
# 一、处理顺序：第1行1-9列 第2行1-9列 。。。
# 二、1-外层循环:行  2-内层循环:列
# 三、显示：列位置*行位置=乘积结果
for ((i=1;i<=9;i++)) # 行
do
    for ((j=1;j<=9;j++)) # 列
    do
        if [ $j -lt $i ];then # 列小于行则输出
            printf "$j*$i=$((j*i))\t"
        elif [ $j -eq $i ];then  # 列等于行则换行
            printf "$j*$i=$((j*i))\n"
        else # 列大于行退出循环
            break
        fi
    done
done
```
