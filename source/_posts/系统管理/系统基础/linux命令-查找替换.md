---
title: linux命令-查找替换
tags:
  - find
  - sort
  - grep
  - sed
categories:
  - linux
date: 2019-06-30 10:40:22
---

# 命令列表
* [sed](#sed)
* [grep](#grep)
* [find](#find)
* wc：统计字节数、字符数、单词数、行数
* [sort](#sort)

# sort
默认以空白(空格、制表符等)为分隔符，默认以第一个字段的第一个字符排序，其他参数如下

* -u：删除重复的行
* -f：忽略大小写
* -b：忽略数据开始的空格部分
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
* -B n：显示每个匹配行及相邻的前n行内容
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

# sed
## 简介
* 含义：用于文本过滤和替换的流编辑器【输入数据源可以是文件或管道】
* 原理：sed把当前处理的行存在临时缓冲区【模式空间pattern space】中，一旦sed完成对模式空间的处理，模式空间中的行就被送到屏幕输出；行被处理完毕后，被模式空间移除，程序读入下一行进行处理
* 语法：sed [命令选项](命令选项) ‘脚本([行选择](行选择)+[操作命令](#操作命令))’ 待操作源文件

## 命令选项
* -n 【--silent --quit】抑制模式空间(pattern space)自动输出，只显示匹配行的内容
* -e 【--expression】script：添加命令将要执行的脚本文本【允许多次处理文本行】
  - 范例：`sed -e '1,3d' -e 's/Hemenway/Jones/' testfile `【删除1到3行后，替换Hemenway为Jones】
* -f 【--file】script-file：添加命令将要执行的脚本文件
* -i`[suffix]` 【--in-place`[=suffix]`】：在模式空间中编辑文件，
  - 如果提供suffix后缀，则会使用suffix后缀对文件备份后再进行编辑
* -r【--regexp-extended】：在脚本中使用扩展的正则表达式
* -u【-unbuffered】：从输入文件加载最少量的数据并更频繁地刷新输出缓冲区

## sed脚本
### 行选择
* 没有行定位信息表示对所有输入行进行处理
* `!`用于地址【或地址范围】之后、命令之前，用于对不匹配的行进行操作
* 数字：仅匹配指定行号的内容【命令行-s选项会跨文件累加行号，此时不适用于行号匹配】
* first~step：从first开始，每次显示第step行内容
  - 范例：sed -n 1~2p ：从第一行开始，只显示奇数行内容
* $：显示最后一行
  - 范例：seq 10|sed -n '3,$p'【显示第3行到末尾行的内容】
* /regexp/：匹配正则表达式匹配的行
* \cregexpc：匹配正则表达式匹配的行：c可以是任意字符
* 以逗号分隔的2个地址【addr1,addr2】表示对2个地址之间的行进行处理
  - addr1,addr2
    + addr1会一直起作用，即使第addr2选择的行早于第1个
    + 假如addr2是正则表达式，则不会对addr1匹配的行本身进行测试
  - addr1,+N：将匹配addr1及以后的N行
  - addr1,~N：将匹配addr1及以后到N的倍数的行
  - 0,addr1：addr1只能使用正则表达式，表示从第一行开始到匹配行的内容
  - 1,addr1：addr1可以是任意形式地址，表示从第一行开始到匹配行的内容

### 操作命令
>`{ }`：一个命令块，命令之间以分号分隔

* s/regexp/replacement/：使用replacement内容替换正则表达式匹配的内容，
  - regexp分组后，replacement语句可以使用\1..\9进行分组引用【`sed -n '/west/s/\(Charles\)/\1jingqi/p' testfile`】
  - 默认只替换第一个出现的字符串，使用g标志可对行内进行全部替换【s/regexp/replacement/g】
  - s后面的字符一定是分隔搜索字符串和替换字符串的分隔符，默认为斜杠；但是在s命令使用的情况下可以改变。不论什么字符紧跟着s命令都认为是新的分隔符。这个技术在搜索含斜杠的模板时非常有用
  - 范例：sed -n '/west/s/Charles/jingqi/p' testfile 【过滤包含west的行后，将Charles替换为jingqi，最后显示该行内容】
* `&`：引用替换命令【s/regexp/replacement/】中正则匹配到的部分
  - 范例：sed -n '/west/s/Charles/&jingqi/p' testfile 【将Charles替换为Charlesjingqi】
* a\ text 在文本行之后的新行添加内容
* i\ text 在文本行之前的新行添加内容
* c\ text：使用文本内容替换文本行
* d：删除模式空间内容，开始下一个循环
  - 范例：seq 10|sed '3d'【删除第3行，其他行默认输出屏幕】
* D：如果模式空间不包含换行符，则和d一样；如果包含换行符，则删除到第一个换行符之间的内容，然后启动循环而不读取新的内容。
* p：打印当前模式空间内容
* P：打印模式空间第一个换行符之前的内容
* = 打印当前行号
* l：显示当前行内容【包含隐藏内容，如换行符】
* l width：显示当前行内容，以指定行宽度显示【超过宽度的行被截断为新行】
* n/N：读取或追加下一行内容到模式空间
  - 范例：`sed -n '/^north /{n;s/central/ceshi/p}' testfile`【定位到north 开始行的下一行，替换字符串后并显示】
* h/H：复制或追加模式空间内容到暂存区
* g/G：使用暂存区内容替换模式空间内容，或在模式空间后追加暂存区内容
* r filename：将从filename文件读取的全部行内容追加到文本行之后
  - 范例：`sed '/^north/r update.txt' testfile`【在以north开始的行后插入update.txt中的内容】
* R filename：将从filename文件读取的一行内容追加到文本行之后，每次从filename中读取的行内容为上次读取行的下一行
* w filename：将模式空间内容写入filename文件
  - 范例：`sed -n '/north/w newfile' testfile `【将包含north的行写入newfile文件中，同时只显示包含north的行】
* W filename：将模式空间的第一行内容写入filename文件
* x：交换模式空间和暂存区内容
* y/source/dest/：将目标中出现的字符替换为源中存在的字符【不能使用正则表达式】
* q：立即退出sed脚本而不处理新行，但如果没有禁用自动打印功能，则会输出模式空间内容
  - 范例：`sed '2q' testfile`【打印到第2行后退出脚本】
* Q：立即退出sed脚本而不处理新行

## 范例
* h与G：`sed -e '/^western/h' -e '$G;w newfile' testfile`
  - 在第一个编辑模式中，将north开始的行从模式空间放入暂存区
  - 在第二个编辑模式中，将暂存区中的内容追加到末尾行的模式空间，然后写入newfile中
* sed中使用shell：里层使用双引号，外层使用单引号
  - `sed -n '/eastern/s/4.5/'"$(date +%F)"'/p' testfile`：将4.5替换为当前时间
  - `sed -n '/eastern/s/Savage/'"$USER"'/p' testfile`：将Savage替换为当前用户
* 获取eth0网卡地址：
  - sed：`ifconfig eth0|sed -n 's/^.*addr:\(.*\)\sBcast.*$/\1/p'`
  - awk：`ifconfig eth0|awk -F '[: ]+' '/inet/{print $4}'`
* 文本修改：将文本中的ver版本号替换为2.0.16，filename版本号替换为2.0.17_40

  ```
  Data ver="2.0.16" filenName="feng1_360_HD_2.0.15_39.apk"
  Down ver="2.0.16" filenName="feng2_360_HD_2.0.15_39.apk"
  Data ver="2.0.14" filenName="feng3_360_HD_2.0.14_38.apk"
  Data ver="2.0.13" filenName="feng4_360_HD_2.0.16_40.apk"
  ```

  - sed方法：`sed -i-bak 's/2.0..*\" /2.0.16\" /;s/HD_.*\./HD_2.0.17_40./' 360test_apk.xml`
  - awk方法：`awk '{gsub("\"2.0.[0-9][0-9]","\"2.0.16");gsub("HD_2.0.1[0-9]_[0-9][0-9]", "2.0.17_40");print}' 360test_apk.xml`

* 文本修改：将文本1修改为文本2
  - 文本1：【修改前】

    ```
    <html>
    <title>Firest web</title>
    <body>Hello the World<body>
    h1helloh1
    h2helloh2
    h3helloh3
    </html>
    ```

  - 文本2：【修改前】
    
    ```
    <html>
    <title>Firest web</title>
    <body>Hello the World<body>
    <h1>hello</h1>
    <h2>hello</h2>
    <h3>hello</h3>
    </html>
    ```

  - 执行命令：`sed 's/^\(h[123]\)/<\1>/;s#\(h[123]\)$#</\1>#' test.txt`
