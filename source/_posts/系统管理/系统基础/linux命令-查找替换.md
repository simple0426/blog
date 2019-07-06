---
title: linux命令-查找替换
tags:
  - find
  - sort
categories:
  - linux
date: 2019-06-30 10:40:22
---

# 命令列表
* sed
* [grep](#grep)
* [find](#find)
* wc
* [sort](#sort)
* tr
* cut

# sort
默认以空白(空格、制表符等)为分隔符，默认以第一个字段的第一个字符排序，其他参数如下

* -u：删除重复的行
* -r：反向排序【默认以字母从前往后排序】
* -o：覆盖性输出【可直接覆盖源文档】
* -n：按数字大小排序【默认从小到大】
* -t：定义分隔符【只能使用单字符作为分隔符】
* -k：指定开始和结束字段以及字段内的字符位置【与-t搭配使用】

# find
在指定的目录下搜索特定的文件，语法格式：find  path  option
## 选项参数
* -maxdepth n 最大查找深度【0为目录本身；1为当前目录下，依次类推；此选项需紧跟查找路径之后】
* -name 按照文件名查找文件【iname忽略大小写】
    - 当name后使用通配符*时必须将匹配模式使用引号括起来或进行转义
    - find . -name '*.c' -print 或 $ find . -name \*.c -print
* -size [±]n按大小匹配（大于、小于或等于n）
    - find . -maxdepth 1 -size +5k
* -type c按照文件类型匹配（f、d、p、s、b、c、l）
* -mtime [±]n按文件最后修改时间【以天计数】
* -mmin [±]n按文件最后修改时间【以分钟计数】
* -o 用于多个条件表达式的"或者"
* -a 用于多个条件表达式的"同时" 
    - find / -type f -size +50k -a -size -55k 2> /dev/null | xargs ls -lh | head -3
    - 查找根目录下大于50k小于55k的文件
* ! 逻辑取反
    - find / -type f -size +50k -a ! -user root
    - 查找根目录下大于50k且不属于root的文件

## 应用范例
### name的使用
* 表达式：find . -name *.log -mtime +60   
* 错误：paths must precede expression
* 原因：*在shell下会直接进行路径匹配
* 正确做法：find . -name '*.c' -print 或 $ find . -name \*.c -print

### 与xargs的搭配
find . -maxdepth 0 -type d -name "*8056" -mtime +119|xargs rm -rf {}

# grep
* 功能：显示匹配特定模式的行
* 语法
  - grep 选项 `[-e]` 模式匹配文本 待处理源文件
  - grep 选项  -f 模式匹配文件 待处理源文件
* 数据源：grep可以从标准输入或文件中读取数据

## 选项
* -E：使用扩展的正则表达式进行模式解析【等同于egrep，grep默认支持基本的正则】
* -F：关闭正则表达式功能，等同于fgrep
  - 范例：`grep -F ".*18159" hosts`【匹配字符串`.*18159`】
* -e pattern：可以多次定义pattern，从而进行多次匹配
* -f FILE：从文件中获取pattern，pattern文件每行定义一个pattern
* -i：忽略大小写进行匹配
* -v：反向匹配
* -w：精确匹配单词（而不是目标的子集）
* -x：精确匹配行（行内容完全相同）
  - 范例：`grep -x test1 hosts`【匹配行内容只有字符串"test1"内容的行】
* -c：显示匹配的行数
* --color=auto：对匹配的内容突出颜色显示
* -l：显示匹配到内容的文件名【常用于多文件搜索】
* -q：静默输出【不输出任何内容】
* -s：屏蔽错误输出
* -h：匹配到内容时，行首不显示文件名【单文件默认不显示】
* -H：匹配到内容时，行首显示文件名【多文件默认显示】
* -n：显示行号
* -A n：显示每个匹配行及相邻的后n行内容
* B n：显示每个匹配行及相邻的前n行内容
* -C m,n：显示每个匹配行相邻的前m、后n行内容
* -r：递归读取目录下的文件，但不包含链接
* -R：递归读取目录下的文件，包含链接
* --exclude=GLOB：跳过指定模式(掩码方式匹配)的文件，掩码可以是`* ? [..]`，`\`可以引用字面意思的掩码和通配符
  - 范例：`grep -r newpass group_vars/ --exclude=test*`【排除group_vars目录下test*文件】
* --exclude-from=FILE：跳过模式匹配文件FILE【和exclude一样，使用掩码方式】中匹配到的文件
* --exclude-dir=DIR：从递归搜索中跳过指定模式的目录
  - 范例：`grep git roles/ -R --exclude-dir={bigdata,web}`【跳过roles目录下web、bigdata目录(忽略目录层级)】
* --include=GLOB：仅搜索指定模式的文件【和exclude一样，使用掩码方式】

## 返回值
* 0：搜索到匹配的内容
* 1：没有搜索到匹配的内容
* 2：搜索时发生错误【可使用-qs屏蔽错误及静默输出】
