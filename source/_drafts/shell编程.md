---
title: shell编程
tags:
categories:
---
# 管道命令
* cmd1|cmd2：将cmd1命令的标准输出传递给cmd2的标准输入
* cmd1|&cmd2：【|&等同于2>&1 |】将cmd1的标准输出和错误输出都传递给cmd2
* 管道中的每个命令都在单独的进程中执行【subshell】

# 命令列表
* &会将命令放入后台执行
* 以“;”分隔的命令会顺序执行，shell等待每个命令执行完成
* cmd1&&cmd2 逻辑“与”连接两个命令，cmd1执行成功后才执行cmd2
* cmd2||&&cmd2 逻辑“或”连接两个命令，cmd2执行不成功才执行cmd2

# 括号与测试命令
* (list)：开启一个子shell执行命令列表，$(list)表示list命令的输出结果（也可以使用反撇号执行命令），常用于a=$(list)这样的赋值语句
    - 其他开启子shell的方法：&（后台执行）、|（管道执行）
* ((expression))：括号里的进行的是数字运算(因此可以用+、-、*、/、>、<等算术运算符)，同样的，$((expression))表示计算的结果
* `[]`：条件表达式，其实是一个程序`/usr/bin/[`，相当于/usr/bin/test，后面多的那个`]`只是对称好看，所以`[`后要有空格
* `[[ expression ]]`：表示里面进行的是逻辑运算，可以使用`!、&&、||`逻辑运算符
* { list; }：内部分组，这个结构事实上创建了一个匿名函数。大括号内的命令不会开启一个子shell运行。括号内的命令用分号分隔，最后一个也要用分号，第一个命令和左括号之间必须要有一个空格。

## 数学计算
* (())：echo $((5+4))
* `expr 1 + 4`：空格分隔
* declare：`declare -i i=2;i=i+3;echo $i`
* let：`let a=3+2;echo $a`

## test命令
* -a、-o 逻辑与、或
* -n、-z 字符串长度不为0、为0
* =、!= 字符串相等、不等
* -ne、-eq、-lt、-le、-gt、-ge：数字上的大【等】于、小【等】于、【不】等于
* -e：文件存在

# 流程控制
* for列表循环：`for name [ [ in [ word ... ] ] ; ] do list ; done`：循环列表中的内容
* for自增循环：`for((expr1;expr2;expr3))`：条件循环；expr1为初始值，expr2为终止条件，expr为循环条件。
    - 范例：for((a=50;a>=0;a--));do echo $a;done
* select循环：`select name [in word];do list;done`：交互式选择列表中的项目
    - 范例：`select name in a b c;do echo $name;done `
* case循环：`case word in [ [(] pattern [ | pattern ] ... ) list ;; ] ... esac`：多条件选择

```
case variable in
value1 | value2)        // value1支持通配符和|作为or的关系
   command1
   ;;
value3)
   command2
   ;;
[vV]alue4)
   command3
   ;;
*)
   command4
   ;;
esac
```

* if语句：`if list; then list; [ elif list; then list; ] ... [ else list; ] fi`

```
if [ $1 -gt 5 ];then
    echo $1
elif [ $1 -eq 5 ];then
    echo 5
else
    echo "$1 is less than 5"
fi
```

* while循环：`while list-1; do list-2; done`：list-1满足条件则执行list-2

```
declare -i i=0
while ((i<=10));
do
    echo $i
    i=i+1
done
```
```
# 读取文件方式1
exec < FILE
while read line
do
    cmd
done
# 读取文件方式2
while read line
do
    cmd
done<FILE
# 范例
while read line
do 
    echo $line
    sleep 1
done < $1
```

* break和continue
    - 不能用于if语句
    - 单循环中：break终止循环；continue终止本次循环(不执行循环后语句)，继续下次循环
    - 双循环中：break跳出内循环，执行外循环语句；break 2直接跳出外循环。continue跳出内循环中的本次循环，继续下次内循环；continue 2跳出内循环，继续下次外循环。

# 函数
## 定义
* name () { cmd-list }
* function name `[()]` { cmd-list }

## 调用
func_name

# 引用机制
* 反斜杠：转移符，保留字面含义【当行尾有反斜杠时表示续行】
    - 特殊转义序列：`\t \n`
* 单引号：保留字符字面含义
* 双引号：可以解析变量及使用转移符

# 特殊语法
* `:`：冒号表示空语句
* `#`：注释内容

# 参数
## 位置参数
* $n表示shell脚本或shell函数的位置参数【n>=1】
* 当数字n大于9时应该这样表示 ${12}

## 特殊参数
* `$* $@`：所有位置参数
    - `$*`：相当于“$1空格$2空格$3...”，即空格分隔多个位置参数同时作为一个字符串
    - `$@`：相当于"$1" "$2"...，即空格分隔多个位置参数并作为多个独立的字符串
* $#：位置参数个数
* $?：上一个shell命令执行状态
* `$$`：当前shell的pid
* $!：上一个放入后台执行的命令的pid
* $0：shell脚本的名字

## 数组
### 定义数据
* 定义整个数组：name=(value1 ... valuen)
    - arry=(128 string http html)
* 定义数组的某个值：`name[subscript]=value`
    - `arry[4]="ceshi"`

### 数组使用
* 查看数组某个元素(下标从0开始)：`${arry[0]}`
* 查看全体元素：`${arry[*]}`
* 查看数组某个元素长度：`${#arry[0]}`
* 查看全体元素个数：`${#arry[*]}`
* 数组元素替换：`${arry[*]/string/jing}`
* 删除数组某个元素：`unset arry[0]`
* 删除数组整体：`unset arry`

### 范例
* 统计并发连接：`netstat -ant|awk -F '[ :]+' '$1 !~ /^Active|^Proto/ && $6 !~ /0.0.0.0|^172.16/{if($5==443)++ip[$6]}END{for(i in ip)print i,ip[i]}'|sort -rn -k2`

# 扩展用法
>可以拆分为多个单词后再命令行依次执行

* 大括号：
    - 扩展字母和数字：{1..n} {a..z}
    - 逗号分隔的扩展：{1,5,a}
* 波浪线：表示当前用户家目录
* 变量扩展
    - 赋值：将一个变量(值可能为空或不存在)的值赋值给另一个变量时的操作
    - 截取：
        + ${变量名:起始索引}            显示起始索引开始的字符串
        + ${变量名:起始索引:长度}   显示起始索引开始的指定长度字符串
        + ${#变量名}                         显示变量长度
        + ${变量名#样式}                 从行首开始截取符合样式的最短字符串：`${PATH#/*:}`
        + ${变量名##样式}               从行首开始截取符合样式的最长字符串：`${PATH##/*:}`
        + ${变量名%样式}                从行尾开始截取符合样式的最短字符串：`${PATH%:/*}`
        + ${变量名%%样式}             从行尾开始截取符合样式的最长字符串：`${PATH%%:/*}`
    - 替换和删除
        + ${变量名/样式/替换}          替换第一个样式字符串：`${PATH/sbin/Sbin}`
        + ${变量名//样式/替换}         替换所有的样式字符串：`${PATH//sbin/Sbin}`
        + ${变量名/样式/ }               删除第一个样式字符串：`${PATH/sbin/}`
        + ${变量名//样式/ }              删除所有的样式字符串：`${PATH//sbin/}`
