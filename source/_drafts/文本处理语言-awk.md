---
title: 文本处理语言-awk
tags: awk
categories: linux
---
# 介绍
在给定的文本中，通过正则匹配(pattern)对行(记录)和列(字段)进行操作(action)

# awk命令
## awk命令语法
* awk [选项](#选项) -f [awk程序文件](#awk程序) 待处理的文本文件
* awk 选项 awk程序文本 待处理的文本文件

## 命令选项
* -f awk程序文件
* -F 定义分隔符【也可使用变量FS】，可以同时定义多个分隔符
    - `ifconfig eth0|awk -F'[ :]*' 'NR==2{print $3}'`【同时使用任意多个空格和任意多个冒号作为分隔符】
* -v var=val 定义变量【定义程序执行前的变量】
* -e awk程序文本【可省略】

# awk程序语法
* 定义：awk程序是由一系列的pattern-action以及可选的函数定义组成
* 格式：[pattern](#awk匹配语法) { [action](#awk执行语法) }
* 执行顺序
    - 命令行-v指定的变量
    - BEGIN指定的规则
    - 处理命令行下引用的每一个源文件【ARGV方式调用】
    - 使用pattern匹配每一个record，匹配成功则执行actions
    - 所有record处理完成后执行END规则
* pattern和action
    - awk是面向行的语言，先是模式(pattern)，后是行为(action)【action包含在花括号中】
    - pattern或action其中之一可能不存在，但不可能出现二者都缺失的情况
    - 假如pattern缺失，则action应用于每一行记录
    - 如果action缺失，则相当于action是{print}【即打印整行记录】
    - 可以使用分号分隔pattern-action下的action的多条语句，或者分隔pattern-actions本身

## awk匹配语法【正则表达式、关系表达式、条件表达式】
* BEGIN/END：
    - BEGIN模式不对输入进行测试【也即不需要源文件也能使用BEGIN和END】，BEGIN模式在读取输入之前执行
    - 所有输入都被处理完毕或执行exit语句后，才开始执行END规则
    - 所有BEGIN模式应该被写成单一规则，且BEGIN模式的action部分会合并执行；END模式也是如此。
    - BEGIN和END模式不与其他模式表达式组合
    - BEGIN和END模式必须有action部分
* BEGINFILE/ENDFILE：BEGINFILE是在读取命令行的每个文件中的第一行记录之前执行的模式，相应的，ENDFILE是读取命令行的每个文件中最后一行记录之后执行的模式
* /regular expression/：对于[正则表达式](#正则表达式)模式，关联的语句将在正则匹配的记录的每一行上都执行；这里使用的正则和egrep中的一样
* relational expression：关系表达式
    - pattern && pattern：两个表达式“与”关系
    - pattern || pattern：两个表达式“或”关系
    - ! pattern：表达式取反
    - pattern ? pattern : pattern：三元表达式【第一个为真则执行第二个，否则执行第三个】
    - (pattern)：括号可以变更逻辑连接顺序
    - pattern1, pattern2：范围表达式，使用pattern1模式测试行记录，接着使用pattern2测试行记录

## awk执行语法
* action语句整体由大括号包围
    - if条件中多个条件使用圆括号连接
    - if条件中的多个执行动作使用分号分隔，整体使用花括号包围
* action语句和大多数程序语言一样，由操作符、控制语句、输入输入语句、赋值、条件、循环语句构成

# awk程序构成
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
* ARGC：【count of arg】命令行参数数量
* ARGIND：【index in ARGV of file being processed 】当前处理的文件在ARGV中的索引
* ARGV：【arry of cmd arg】命令行参数数组
* ENVIRON：环境变量数组
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

## 数据类型
* 字符串常量
* 数字
* 字符串
* 数组

## 操作符【和表达式】
>优先级递减

1. (...)：分组(但是awk分组功能不支持后向引用)
2. $：字段或变量引用
3. `++ --`：递增或递减
4. ^：指数【也可以使用`**`，例如`**`=可以在赋值语句中使用】
5. +-!：加减，逻辑非
6. */%：乘除取余
7. 空格：字符串连接
8. |和|&：用于getline、print、printf的的管道符
9. `< > <= >= != ==`：关系运算符
10. ~ !~：正则匹配，正则否定；只在符号的右侧使用常量，比如【$1 ~ 'ceshi'】
11. in：在数组之中
12. && ||：逻辑“与”、“或”
13. ?:：三元运算符
14. = += -= *= /= %= ^=：赋值语句（包含绝对赋值和带有其他操作符的赋值）

## 流程控制
* if语句：if (condition) statement [ else statement ]
* 循环语句while：while (condition) statement
* 循环语句do-while：do statement while (condition)
* 循环语句for：for (expr1; expr2; expr3) statement
* 数组循环for：for (var in array) statement
* 跳出循环体break
* 跳出本次循环continue
* 删除数组成员：delete array[index]
* 删除数组：delete array
* 退出程序：exit [ expression ]{ statements }
* 多种选择switch：

    ```
    switch (expression) {
    case value|regex : statement
    ...
    [ default: statement ]
    }
    ```

## 输入输出
* close：关闭文件或管道
* [getline](#getline与next)：从下一个输入记录中设置$0，同时设置NF, NR, FNR, RT.
    - 对于模式匹配的记录直接跳过，直接处理相邻的下一个记录
* getline < file：从下一个文件记录中设置$0，设置NF，RT
* getline var：从下一个输入记录中设置变量，设置NR，FNR，RT
* getline var < file：从下一个文件记录中设置变量，设置RT
* command| getline `[var]`：执行命令，将结果保存在$0或变量var中以及RT中
* command|& getline `[var]`：执行命令，同时将结果保存在$0或变量var中以及RT中
* [next](#getline与next)：停止处理当前的输入记录，读取下一个输入记录并使用awk程序的第一个模式处理；在到达输入数据的末尾时，awk执行END规则
    - 对于第一个模式匹配的记录直接跳过，直接处理余下的全部记录
    - 因此需要至少一个匹配模式，next位于第一个匹配模式之后，且由花括号包围
* nextfile：停止处理当前的输入文件，读取的下一个记录来自下一个文件。FILENAME和ARGIND被更新，FNR设置为1，并使用awk程序的第一个模式处理；在到达输入数据的末尾时，awk执行END规则
* print：打印当前记录，输出记录以ORS定义的值结束
    - print的参数可以使变量、数值、字符串【字符串使用双引号引用】
    - 参数之间使用逗号分隔，没有分隔符时参数就“黏”在一起
    - 可以使用的特殊字符：`\t`：制表符 `\n`：换行符
    - 范例：
        + 默认格式(%.6g)：`awk 'BEGIN{a=1.23456789,print a}'`
        + 自定义格式：`awk 'BEGIN{OFMT="%.2f";a=1.23456789;print a}'`
* print expr-list：打印表达式，表达式直接以OFS分隔，输出记录以ORS定义的值结束
* print expr-list >file：打印表达式到文件，文件名使用双引号
* printf fmt, expr-list：格式化输出
    - 范例：
        + 显示日期的每一个字段：`date|awk '{for(i=1;i<=NF;i++)printf("%s\n",$i)}'`
        + 格式化数字：`awk 'BEGIN{a=123.456;printf("a is %.2f\n", a)}'`
* printf fmt, expr-list >file：格式化输出到文件
* system(cmd-line)：执行命令并返回退出状态【不适用于非unix系统】
* `fflush([file])`：刷新所有输出文件或管道的buffer，如果文件丢失或为空则刷新所有输出文件或管道
* print ... >> file：追加输出到文件
* print ... | command：输出写入管道
* print ... |& command：打印输出并写入管道

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
* `gsub(regx, str [, target_str])` ：在目标字符串中，每个与正则表达式匹配的子串都被替换，并返回替换的次数；如果目标字符串未提供则使用$0
* `sub(regx, str [, target_str])`：只替换第一个匹配的字符串
* `index(str, target)`：返回子字符串在字符串中的索引位置，从1开始
* `length([str]) `：返回字符串的长度或数组个数
* `match(str, regx [, arry])`：返回正则表达式在字符串中出现的位置，找不到则返回0
* `substr(str, index [, n])`：字符串截取，从字符串中的第index个索引位置截取n个字符
* `tolower(str)`：返回字符串的字母的小写形式
* `toupper(str)`：返回字符串中字母的大写形式
* `split(str, arry [, regx [, seps] ])`：使用正则表达式regx定义的分隔符将字符串str拆分成数组arry，如果regx未定义则使用FS
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

### 时间函数
* systime()：返回当前时间到epoch(1970-01-01 00:00:00 UTC)之间的秒数【即timestamp】
*  mktime(datespec)：将datespec转换成和systime一样的时间戳（timestamp），datespec形式为`YYYY MM DD HH MM SS[ DST]`
* strftime([format [, timestamp[, utc-flag]]])：将timestamp格式化为[指定格式](#时间格式)的字符串输出

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
# next
awk '/3/{next}{print $0}' 1.txt：直接过滤掉包含3的记录
# getline
awk '/3/{getline;print $0}' 1.txt：对包含3的记录进行处理：读取下一行内容后直接打印
awk '/3/{getline}{print $0}' 1.txt：此时和next用法一直：直接过滤掉包含3的记录
```

## 应用范例
* 将java项目的sql导出：`awk -F'Preparing: ' '/Preparing/{var=$2;sub("  .*$","",var);print var}' catalina.out-20190622`
    - 过滤包含“Preparing”的行【这样的行包含sql语句】
    - 将第一步过滤的记录以“Preparing: ”分隔取第二个字段
    - 删除【双空格及以后的内容】
* 统计sql及其执行次数：`awk '{++count[$0]}END{for(i in count)print i,"/",count[i]}' total.sql|sort -t'/' -rn -k2 > tongji.sql`
    - 以每行sql作为自加的变量
    - 重复的sql，变量会自加；新的sql会形成新的自加变量；多个不同的sql自加变量会形成数组，以此类推。
    - 在所有记录处理完成后打印变量以及变量的自加结果【以斜线分隔】
    - 对awk的处理结果再使用sort进行排序【以斜线作为分隔符，对第二个字段以数字方式进行倒序输出】
* 统计web服务器443端口外网tcp各种连接状态及数量【也即并发连接】
    - 连接状态统计并排序：`netstat -ant|awk -F '[ :]+' '$1 !~ /^Act|^Pro/{if(($5==443)&&($6!="0.0.0.0"))++count[$8]}END{for(i in count)print i,count[i]}'|sort -nr -k2`
        + 排除Act和Pro开头的字头行
        + 包含连接443端口的连接
        + 排除来自0.0.0.0的连接
        + 以tcp状态为作为自加数组变量，不同tcp状态共同构成数组
        + 打印tcp状态及数量
        + 以tcp状态的个数作为排序依据
    - 并发总数：`netstat -ant|awk -F '[ :]+' '$1 !~ /^Act|^Pro/{if(($5==443)&&($6!="0.0.0.0"))++count[$1]}END{print count[$1]}'`
        + 选取每行都不变的信息作为自加变量，最终统计并发连接
* 统计一天中某个域名下url访问排行榜【get去除参数】:`awk -F'[ ?]' '{++count[$7]}END{for(i in count)print i,count[i]}' access.log|sort -rn -k2|head -10`

---
# 附加内容
## 时间格式
| 格式符 |                            含义                           |           范例           |
|--------|-----------------------------------------------------------|--------------------------|
| %a     | 缩写的星期几                                              | Sun                      |
| %A     | 完整的星期几                                              | Sunday                   |
| %b     | 缩写的月份                                                | Mar                      |
| %B     | 完整的月份                                                | March                    |
| %c     | 日期和时间表示法                                          | Sun Aug 19 02:56:02 2012 |
| %d     | 一月中的第几天（01-31）                                   | 19                       |
| %H     | 24小时制的小时（00-23）                                   | 14                       |
| %I     | 12小时制的小时（01-12）                                   | 05                       |
| %j     | 一年中的第几天（001-366）                                 | 231                      |
| %m     | 十进制表示的月份（01-12）                                 | 08                       |
| %M     | 分（00-59）                                               | 55                       |
| %p     | AM或PM名称                                                | PM                       |
| %S     | 秒 （00-59）                                              | 02                       |
| %U     | 一年中的第几周，以第一个星期日为第一周的第一天（00-53）   | 33                       |
| %w     | 十进制表示的星期几，星期日表示为0（0-6）                  | 4                        |
| %W     | 一年中的第几周，以第一个星期一作为第一周的第一天（00-53） | 34                       |
| %x     | 日期表示法                                                | 08/19/12                 |
| %X     | 时间表示法                                                | 02:50:06                 |
| %y     | 年份的最后两个数字（00-99）                               | 01                       |
| %Y     | 年份                                                      | 2012                     |
| %Z     | 时区的名称或缩写                                          | CST                      |
| %%     | 一个%符号                                                 | %                        |

## 正则表达式
* `.`：匹配任何字符，包含换行符
* ^：匹配字符串开始
* $：匹配字符串结束
* `[abc...]`：字符列表，也可以使用短横线表示字符范围，如：`[a-g]`
* `[^abc]`：字符列表取反
* r1|r2：匹配r1或r2
* r1r2：先匹配r1，后匹配r2
* r+：匹配r 1次或多次
* r*：匹配r 0次或多次
* r?：匹配r 0次或1次
* (r)：分组后，匹配r，
* r{n,m}：匹配r至少n次，至多m次
* \y：匹配单词开头或结尾的空字符串
* \B：匹配单词中的空字符串
* `\<`：匹配单词开头的空字符
* `\>`：匹配单词末尾的空字符串
* \s：匹配空白字符
* \S：匹配非空白字符
* \w：匹配字母、数字、下划线
* \W：匹配非字母、数字、下划线字符
* `\``：匹配缓冲区开头空字符串
* \'：匹配缓冲区末尾空字符串
* `\`：转义字符
* `[:keyword:]`：POSIX标准字符类【与国家、地区的字符集有关，不建议使用】