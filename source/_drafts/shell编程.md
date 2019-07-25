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

# 复合命令
* (list)：开启一个子shell执行命令列表，$(list)表示list命令的输出结果，常用于a=$(list)这样的赋值语句
    - 其他开启子shell的方法：&（后台执行）、|（管道执行）
* ((expression))：括号里的进行的是数字运算(因此可以用+、-、*、/、>、<等算术运算符)，同样的，$((expression))表示计算的结果
* `[]`：条件表达式，其实是一个程序`/usr/bin/[`，相当于/usr/bin/test，后面多的那个`]`只是对称好看，所以`[`后要有空格
* `[[ expression ]]`：表示里面进行的是逻辑运算，可以使用`!、&&、||`逻辑运算符
* { list; }：内部分组，这个结构事实上创建了一个匿名函数。大括号内的命令不会开启一个子shell运行。括号内的命令用分号分隔，最后一个也要用分号，第一个命令和左括号之间必须要有一个空格。
* `for name [ [ in [ word ... ] ] ; ] do list ; done`：循环列表中的内容
* `for((expr1;expr2;expr3))`：条件循环；expr1为初始值，expr2为终止条件，expr为循环条件。
    - 范例：for((a=50;a>=0;a--));do echo $a;done
* `select name [in word];do list;done`：交互式选择列表中的项目
    - 范例：`select name in a b c;do echo $name;done `
* `case word in [ [(] pattern [ | pattern ] ... ) list ;; ] ... esac`：多条件选择

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

* `if list; then list; [ elif list; then list; ] ... [ else list; ] fi`

```
if [ $1 -gt 5 ];then
    echo $1
elif [ $1 -eq 5 ];then
    echo 5
else
    echo "$1 is less than 5"
fi
```

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
