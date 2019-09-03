---
title: 文本处理语言-awk
tags: awk
categories: linux
date: 2019-07-06 01:42:52
---

# 介绍
* awk是用于文本检索和处理的语言
* awk程序是由一系列的pattern-action以及可选的函数定义组成
* 可以在命令行中输入短程序文本(通常使用单引号括起来以避免被shell解释)，也可以使用-f选项从文件中读取长的awk程序
* 读取的数据源可以是命令行的文件列表或标准输入
* 输入的数据被记录分隔符RS(RS默认为”\n“)切分为多条记录，每个记录都会与每个模式进行比较，如果匹配则执行{action}的程序文本

# awk命令
## 命令语法
* awk [选项](#选项) -f [awk程序文件](#awk程序) 待处理的文本文件
* awk 选项 awk程序文本 待处理的文本文件

## 命令选项
* -f awk程序文件
* -F 定义分隔符【也可使用变量FS】，可以同时定义多个分隔符
    - 范例：`ifconfig eth0|awk -F'[ :]*' 'NR==2{print $3}'`【同时使用任意多个空格和任意多个冒号作为分隔符】
* -v var=val 定义变量【定义程序执行前的变量】
* -e awk程序文本【可省略】

## 执行顺序
- 命令行-v指定的变量
- BEGIN指定的规则
- 处理命令行下引用的每一个源文件【ARGV方式调用】
- 使用pattern匹配每一个record，匹配成功则执行actions
- 所有record处理完成后，执行END规则

# awk程序
* awk程序是由一系列的[pattern](#awk模式) { [action](#awk执行) }以及可选的函数定义组成
* awk是面向行的语言，先是模式(pattern)，后是行为(action)，并且pattern和action是捆绑在一起的【模式控制的动作从紧挨的第一个花括号开始到第一个花括号结束】
* pattern或action其中之一可能不存在，但不可能出现二者都缺失的情况；假如pattern缺失，则action应用于每一行记录；如果action缺失，则相当于action是{print}【即打印整行记录】

## awk模式
* BEGIN/END：
    - BEGIN和END模式不对输入进行测试【也即不需要源文件也能使用BEGIN】，BEGIN在读取输入之前执行，END在所有输入都被处理完毕后执行
    - BEGIN模式通常被用来改变内建变量，如OFS，RS，FS等，也可以用于初始化自定义变量值，或打印输出标题
    - BEGIN和END模式必须有action部分
    - 所有BEGIN模式的action部分会合并执行，END模式也是如此；但是BEGIN和END模式不与模式匹配中的表达式合并。
* BEGINFILE/ENDFILE：BEGINFILE是在读取命令行的每个文件中的第一行记录之前执行的模式；相应的，ENDFILE是读取命令行的每个文件中最后一行记录之后执行的模式
* 模式匹配部分：由表达式(可以是记录、字段、内置变量、字符串、数字、正则表达式)和[操作符](#操作符)构成
    - 正则表达式
        + 语法：expr ~ /r/
        + 含义：expr用来和正则表达式【和egrep一样的正则】进行匹配测试
        + 说明：【/r/ { action }】和【$0 ~ /r/ { action }】都是相同的意思，都是用一行记录和正则进行匹配测试
    - 条件表达式
        + 范例：'NR>=2&&NR<=10{print $3}'【取2到10行中第3个字段中的内容】
    - 关系表达式：使用[操作符](#操作符)【&&、||、!、三元操作符、括号(改变连接顺序)，逗号(先后进行匹配测试)】连接多个表达式

## awk执行
* action语句和大多数程序语言一样，由控制语句、输入输入语句、赋值、操作符构成
* 循环体或action部分被大括号`{...}`分块，块内语句由分号或换行符分隔，块内最后一条语句不需要终止符
* 可以使用`\`来继续长语句；相反的，在逗号、左大括号、&&、do、else、if/while/for语句的右括号、自定义函数的右括号之后，可以在没有反斜杠的情况下断开语句

# awk语法
## 记录
* 记录通常由换行符分隔【也即一行内容为一个记录】，也可以由内置变量RS定义分隔符
* 只有单字符或正则表达式可以作为分隔符
* 如果RS设置空，则由空行(\n\n)作为记录分隔符；此时，换行符(\n)作为字段分隔符【FS】

## 字段
* 默认使用空格分隔记录为多个字段
* 使用FS定义字段分隔符，FS可以是单字符或正则表达式
* 如果设定FIELDWIDTHS，即每个字段相同宽度，则会覆盖FS设置
* 记录中的每个字段都可以使用$1,$2...$N进行引用，$0表示整个记录，不能使用负数进行字段引用
* 变量NF表示记录中的总字段数
* 引用不存在的字段将会产生空字符串

## 内置变量
* ARGC：【count of arg】命令行参数数量【不包括选项和awk程序】
* ARGIND：【index in ARGV of file being processed 】当前处理的文件在ARGV中的索引
* ARGV：【arry of cmd arg】命令行参数数组
    - ARGV[0]为awk解释器
    - ARGV[1]...ARGV[ARGC-1]为待处理的文件
* ENVIRON：环境变量数组【环境变量是由a父进程shell传递给awk程序的】
    - 范例：`awk 'BEGIN{print ENVIRON["HOME"]}'`
* FIELDWIDTHS：固定宽度分隔字段
* FILENAME：当前输入的文件名，在BEGIN阶段则是未定义
* NR【number of record】：到目前为止的输入记录总数
* FNR：【number record of current file】正在处理的记录在当前文件的行号
    - NR与FNR：由于awk可以一次处理多个文件，而FNR每打开一个文件都会重置为0，所以总是存在NR>=FNR【在一个文件中NR和FNR相等】
    - 范例(打印5到10行之间的内容)：awk 'NR>=5&&NR<=10{print $0}' access.log
* FS【field separator】：输入字段分隔符【默认空格】
* OFS【output field separator】：输出字段分隔符【默认空格】
* RS【record separator】：输入记录分隔符【默认换行符】
* ORS【output record separator】：输出记录分隔符【默认换行符】
* NF【number of field】：输入记录的字段总数
* OFMT【output format】：数字输出格式，默认"%.6g"
* RT【record terminator】：记录终止符，awk将RT设置为与RS相匹配的字符或正则表达式匹配的文本
* IGNORECASE：在正则表达式和字符串操作中关闭大小写敏感【当IGNORECASE内置变量的值为非0时，表示在进行字符串操作和处理正则表达式时关闭大小写敏感。】

## 数据类型
* 字符串常量
    - `\n`：换行符
    - `\t`：制表符
    - `\\`：反斜杠
* 字符串：由双引号包围
* 数字：在awk中变量无须定义即可使用，变量在赋值时即已经完成了定义。变量的类型可以是数字、字符串。根据使用的不同，未初始化变量的值为0或空白字符串" "，这主要取决于变量应用的上下文。
* 数组
    - 删除数组成员：delete array[index]
    - 删除数组：delete array

## 操作符
* (...)：分组(但是awk分组功能不支持后向引用)
* $：字段或变量引用
* ^：指数【也可以使用`**`，例如`**`=可以在赋值语句中使用】
* `+ - * / % ++ --`：算数运算符：加、减、乘、除、取余、递增、递减
* |和|&：用于getline、print、printf的的管道符
* `< > <= >= != ==`：比较运算符
* ~ !~：正则匹配，正则否定；只在符号的右侧使用常量，比如【$1 ~ 'ceshi'】
* in：在数组之中
* && || !：逻辑运算符：“与”、“或”、“非”
* ?:：三元运算符【第一个为真则执行第二个，否则执行第三个】
* = += -= *= /= %= ^=：赋值语句（包含绝对赋值和带有其他操作符的赋值）

## 流程控制
* if语句：if (condition) statement [ else statement ]
    - 多个条件语句使用圆括号分组后使用逻辑操作符连接
    - 多个执行语句使用分号分隔，整体使用花括号包围
* 循环语句while：while (condition) statement
* 循环语句do-while：do statement while (condition)
* 循环语句for：for (expr_st; expr_end; expr_incre) statement
* 数组循环for：for (var in array) statement
* 跳出循环体：break
* 跳出本次循环：continue
* 退出awk程序【但是不会跳过END模块】：exit [ expression ]{ statements }
* 多种选择switch：

    ```
    switch (expression) {
    case value|regex : statement
    ...
    [ default: statement ]
    }
    ```

## 输入输出
### 输出重定向
* 使用shell的通用重定向符号“>”完成awk的输出重定向
* 使用">"时，被打开的文件先被清空；文件会持续打开，直到文件被明确关闭(close)或awk程序结束
* 使用">>"时，重定向的输出只是添加到文件末尾

### 输入重定向
* awk对于输入重定向是通过getline函数完成的，
* getline可以从标准输入、管道、当前正在处理的文件之外的其他文件获取

### 语法
* [getline](#getline与next)：从下一个输入记录中设置$0，同时设置NF, NR, FNR, RT.
    - 对于模式匹配的记录直接跳过，直接处理相邻的下一个记录
* getline < file：从下一个文件记录中设置$0，设置NF，RT
* getline var：从下一个输入记录中设置变量，设置NR，FNR，RT
* getline var < file：从下一个文件记录中设置变量，设置RT
* command| getline `[var]`：执行命令，将结果保存在$0或变量var中以及RT中
    - 范例【命令输出保存为变量并打印】：`awk 'BEGIN{"date"|getline d;print d}'`
* command|& getline `[var]`：执行命令，同时将结果保存在$0或变量var中以及RT中
* print：打印当前记录【即$0】到标准输出
* print expr-list：打印表达式到标准输出
    - 表达式之间以OFS分隔，输出记录以ORS定义的值结束
    - 表达式可以时变量、数值、字符串【字符串使用双引号引用】、字符串常量【如`\t`：制表符 `\n`：换行符】
    - 参数之间使用逗号分隔；没有分隔符时，参数就“黏”在一起
    - 范例：
        + 默认格式(%.6g)：`awk 'BEGIN{a=1.23456789,print a}'`
        + 自定义格式：`awk 'BEGIN{OFMT="%.2f";a=1.23456789;print a}'`
* print expr-list >file：打印表达式到文件【文件名使用双引号】
* print ... >> file：打印表达式到文件末尾
* printf fmt, expr-list：格式化输出
    - 范例：
        + 显示日期的每一个字段：`date|awk '{for(i=1;i<=NF;i++)printf("%s\n",$i)}'`
        + 格式化数字：`awk 'BEGIN{a=123.456;printf("a is %.2f\n", a)}'`
* printf fmt, expr-list >file：格式化输出到文件
* print ... | command：输出写入管道
* print ... |& command：打印输出并写入管道
* [next](#getline与next)：停止处理当前的输入记录，读取下一个输入记录并使用awk程序的第一个模式处理；在到达输入数据的末尾时，awk执行END规则
    - 对于第一个模式匹配的记录直接跳过，直接处理余下的全部记录
    - 因此需要至少一个匹配模式，next位于第一个匹配模式之后，且由花括号包围
* nextfile：停止处理当前的输入文件，读取的下一个记录来自下一个文件。FILENAME和ARGIND被更新，FNR设置为1，并使用awk程序的第一个模式处理；在到达输入数据的末尾时，awk执行END规则
* close：关闭文件或管道
* system(cmd-line)：执行命令并返回退出状态【不适用于非unix系统】
    - `awk 'BEGIN{system("date")}'`
* `fflush([file])`：刷新所有输出文件或管道的buffer，如果文件丢失或为空则刷新所有输出文件或管道

### 特殊文件
当从print或printf进行i/o重定向到文件，或使用getline从文件中进行i/o重定向时，awk程序可以从内部识别某些特殊的文件名。  
这些文件名允许通过继承自awk的父进程【一般为shell】来进行访问  
这些文件名也可以直接在命令行【即在shell】进行访问  

* -：标准输入
* /dev/stdin：标准输入
* /dev/stdout：标准输出
* /dev/stderr：标准错误输出

## 内置函数
### 字符串函数
* `gsub(regx, substr [, target_str])` ：在目标字符串中，每个与正则表达式匹配的子串都被替换，并返回替换的次数；如果目标字符串未提供则使用$0
    - 范例：`awk '{gsub(":1","--");print}' test.txt`【将记录中的":1"替换为"--"】
* `sub(regx, substr [, target_str])`：只替换第一个匹配的字符串
* `index(str, substr)`：返回子字符串在字符串中的索引位置，从1开始
    - 范例：`awk 'BEGIN{print index("1a2b3","2b")}' test.txt`【返回"2b"第一个字符在"1a2b3"中的索引位置】
* `length([str]) `：返回字符串的长度或数组个数
* `match(str, regx [, arry])`：返回正则表达式在字符串中出现的位置，找不到则返回0
* `substr(str, index [, n])`：字符串截取，从字符串中的第index个索引位置截取n个字符
* `tolower(str)`：返回字符串的字母的小写形式
* `toupper(str)`：返回字符串中字母的大写形式
* `split(str, arry [, regx [, seps] ])`：使用正则表达式regx定义的分隔符将字符串str拆分成数组arry，如果regx未定义则使用FS
    - 范例：`awk 'BEGIN{split("1a 2b 3c",a)}END{for(i=0;i<length(a);i++)print a[i]}' test.txt`
    - awk的切分算法：用split函数将字符串切分为数组，用RS分隔符将文件切分为记录，用FS分隔符将记录切分为字段
* `strtonum(str)`：字符串转化我数字
* `sprintf(fmt, expr-list)`：使用指定格式输出表达式

### 格式化输出printf
* %c：单字符
*  %d, %i：十进制数（整数部分）
*  %e, %E：`[-]d.dddddde[+-]dd`格式的浮点数【科学表示法】
*  %f, %F：`[-]ddd.dddddd`格式浮点数
*  %g, %G：取%e或%f中较短的内容，并抑制无意义0的输出
*  %o：无符号八进制数【整数】
*  %u：无符号十进制数【整数】
*  %s：字符串
*  %x, %X：无符号十六进制数【整数】，使用ABCDEF代替abcdef
*  %%：单字符%
*  count$：格式化字符串时指定位置参数
*  -：表达式应该在字段内左对齐
*  空格：对于数字转化，应该使用空格为正值添加前缀，使用减号为负值添加前缀
*  +：数字转化时，为宽度修饰符提供前缀符号
*  #：对数字系列的控制字符提供转化形式
*  0：前导0用于数字格式化的填充
*  width：定义输出宽度
*  .prec：定义输出精度控制

### 数学函数
* int(expr)：截断为整数
* rand()：返回0和1之间的随机数N【0 ≤ N < 1】
* `srand([expr])`：使用expr作为随机数生成器的新种子，如果未提供expr则使用时间作为种子
    - 范例：`awk 'BEGIN{srand();printf("%d\n", rand()*10)}'`

### 时间函数
* systime()：返回当前时间到epoch(1970-01-01 00:00:00 UTC)之间的秒数【即timestamp】
*  mktime(datespec)：将datespec转换成和systime一样的时间戳（timestamp），datespec形式为`YYYY MM DD HH MM SS[ DST]`
* strftime([format [, timestamp[, utc-flag]]])：将timestamp格式化为指定格式的字符串输出
    - 范例：`awk 'BEGIN{print strftime("%Y%m%d", systime())}'`

### 类型函数
isarray(x)：如果x是数组则返回为真

## 用户自定义函数
* 语法:function name(parameter list) { statements }
* 使用：
    - 可以在pattern或action中调用函数
    - 函数调用时，左括号需要紧跟函数名【但这不适用于内置函数】
    - 可以在函数中使用return表达式来返回值，假如没有提供值则返回值显示为未定义

# awk范例
## getline与next
```
# 源文本
1
2
3
00
23
4

5
6
7
11
12
# next用法示例
awk '/3/{next}{print $0}' 1.txt：直接过滤掉包含3的记录
# getline用法示例
awk '/3/{getline;print $0}' 1.txt：对包含3的记录进行处理：读取下一行内容后直接打印
awk '/3/{getline}{print $0}' 1.txt：此时和next用法一直：直接过滤掉包含3的记录
```

##  getline
* 循环获取命令输出并打印：`awk 'BEGIN{while("ls"|getline)print}'`
* getline交互式

```
awk 'BEGIN{print "what is your name?";getline name < "/dev/pts/0"}\
$1 ~ name{print "Found " name "on line", NR}\
END{print"See ya," name}' test.txt

# BEGIN部分：打印标题【what is your name?】，从pts终端获取输入并赋值给name
# pattern-action部分：如果文件中的某一行记录的第一个字段是name变量的值，则打印内容
# END部分：打印name变量相关内容
```

## 应用范例
* SQL执行统计【将java项目运行过程中的sql导出】：`awk -F'Preparing: ' '/Preparing/{var=$2;sub("  .*$","",var);print var}' catalina.out-20190622`
    - 过滤包含“Preparing”的行【这样的行包含sql语句】
    - 将第一步过滤的记录以“Preparing: ”分隔取第二个字段
    - 删除【双空格及以后的内容】
* TCP状态统计：【统计服务器443端口外网tcp各种连接状态及数量】
    - 连接状态统计并排序：`netstat -ant|awk -F '[ :]+' '$1 !~ /^Act|^Pro/{if(($5==443)&&($6!="0.0.0.0"))++count[$8]}END{for(i in count)print i,count[i]}'|sort -nr -k2`
        + 排除Act和Pro开头的字头行
        + 包含连接443端口的连接
        + 排除来自0.0.0.0的连接
        + 以tcp状态为作为自加数组变量，不同tcp状态共同构成数组
        + 打印tcp状态及数量
        + 以tcp状态的个数作为排序依据
    - 连接总数【并发】：`netstat -ant|awk -F '[ :]+' '$1 !~ /^Act|^Pro/{if(($5==443)&&($6!="0.0.0.0"))++count[$1]}END{print count[$1]}'`
        + 选取每行都不变的信息作为自加变量，最终统计并发连接
* WEB访问统计：【nginx日志中：第1个字段为访问者ip，第7个字段为访问的url【含get参数】，第10个字段为访问url响应体大小】
    - 统计每天访问量最大的资源【get去除参数】TOP10:`awk -F'[ ?]' '{++url[$7]}END{for(i in url)print i,url[i]}' access.log|sort -rn -k2|head -10`
    - 统计每天占用带宽最大的资源TOP10：`awk '{sub("?.*$","",$7);url[$7]+=$10}END{for(i in url)print i,url[i]}' access.log|sort -rn -k2`
        + sub("?.*$","",$7)：get去除参数，获取真正的资源
        + `url[$7]+=$10`：对同一个资源，参数不同，获取的响应体大小也不一样；因此需要将相同资源每次返回的响应体大小累加
    - 统计每天访问量最大的ip TOP10：`awk '{++ip[$1]}END{for(i in ip)print i,ip[i]}' access.log|sort -rn -k2|head -10`

## shell脚本中的awk
```
read -p "pls input a ip:" ip
egrep -v "localhost|^$" /etc/hosts|awk '{if($1=="'$ip'")print $2}'
```

---
# 参考
* centos下的gawk man文档
* ubuntu下的mawk man文档
