---
title: shell编程
tags:
  - shell
categories:
  - linux
date: 2019-07-30 18:05:31
---

# 变量
## 变量定义
* 当变量值为数字或路径时，可以不适用引号
* 等号两边不能有空格
* 变量名的开头必须是字母

## 引用机制
>是用于删除shell中某些特殊的字符或单词

* 反斜杠：转移符，保留字面含义【当行尾有反斜杠时表示续行】
    - 特殊转义序列：`\t \n`
* 单引号：保留字符字面含义
* 双引号：可以解析变量及使用转移符

## 位置参数
* $n表示shell脚本或shell函数的位置参数【n>=1】
* 当数字n大于9时应该这样表示：${12}

## 特殊变量
* `$* $@`：所有的位置参数
    - `$*`：相当于“$1空格$2空格$3...”，即空格分隔多个位置参数同时整体作为一个字符串
    - `$@`：相当于"$1" "$2"...，即空格分隔多个位置参数并作为多个独立的字符串
* `$#`：位置参数个数
* `$?`：上一个shell命令执行状态
* `$$`：当前shell的pid
* `$!`：上一个放入后台执行的命令的pid
* $0：shell脚本的名字
* 波浪线(~)：表示当前用户家目录

## 数组
### 定义数据
* 定义整个数组：name=(value1 ... valuen)
    - arry=(128 string http html)
* 定义数组的某个值：`name[subscript]=value`
    - `arry[4]="ceshi"`

### 数组使用
* 查看数组某个元素(下标从0开始)：`${arry[0]}`
* 查看全体元素：`${arry[*]}`
* 查看数组某个元素长度：{% raw %}${#arry[0]}{% endraw %}
* 查看数组元素个数：{% raw %}${#arry[*]}{% endraw %}
* 数组元素替换：`${arry[*]/string/jing}`
* 删除数组某个元素：`unset arry[0]`
* 删除数组整体：`unset arry`

### 范例
* 统计并发连接：`netstat -ant|awk -F '[ :]+' '$1 !~ /^Active|^Proto/ && $6 !~ /0.0.0.0|^172.16/{if($5==443)++ip[$6]}END{for(i in ip)print i,ip[i]}'|sort -rn -k2`

## 变量扩展
- 赋值：将一个变量的值(值可能为空或不存在)赋值给另一个变量时的操作
- 截取：
    + `${变量名:起始索引}`            显示起始索引开始的字符串
    + `${变量名:起始索引:长度}`   显示起始索引开始的指定长度字符串
    + {% raw %}${#变量名}{% endraw %}                       显示变量长度
    + `${变量名#样式}`                从行首开始截取符合样式的最短字符串：`${PATH#/*:}`
    + `${变量名##样式}`               从行首开始截取符合样式的最长字符串：`${PATH##/*:}`
    + `${变量名%样式}`                从行尾开始截取符合样式的最短字符串：`${PATH%:/*}`
    + `${变量名%%样式}`             从行尾开始截取符合样式的最长字符串：`${PATH%%:/*}`
- 替换和删除
    + `${变量名/样式/替换}`          替换第一个样式字符串：`${PATH/sbin/Sbin}`
    + `${变量名//样式/替换}`         替换所有的样式字符串：`${PATH//sbin/Sbin}`
    + `${变量名/样式/ }`               删除第一个样式字符串：`${PATH/sbin/}`
    + `${变量名//样式/ }`              删除所有的样式字符串：`${PATH//sbin/}`

## 路径名匹配
>即bash下的通配符

- `*`：匹配任意字符串
- `?`：匹配任意单个字符
- `[...]`：
    + 匹配括号内任意字符
    + 包含横杆【-】时表示匹配字符范围内的字符
    + 首字符是`!`或`^`表示不匹配括号内字符
- 复合模式【由多个匹配模式构成】
    + ?(pattern-list)：匹配0或1个给定的模式
    + *(pattern-list)：匹配0或多个给定的模式
    + +(pattern-list)：匹配1或多个给定的模式
    + @(pattern-list)：匹配1个给定的模式
    + !(pattern-list)：不匹配模式列表中任何项

# 函数
## 定义
* name () { cmd-list }
* function name `[()]` { cmd-list }
* 定义局部变量：local

## 调用
func_name
## 范例
```
定义：
usage (){
    echo "usage: $0 <git branch name>"
}
调用：
usage
```

# 管道
* 使用方式1：cmd1|cmd2：将cmd1命令的标准输出传递给cmd2的标准输入
* 使用方式2：cmd1|&cmd2：【|&等同于2>&1 |】将cmd1的标准输出和错误输出都传递给cmd2
* 管道与进程：管道中的每个命令都在单独的进程中执行【subshell】

# 重定向
* 重定向标准输入：`<word`
* 重定向输出：
    - 标准输出：>word
    - 错误输出：2>word
* 追加重定向输出：`[n]`>>word
* 同时重定向标准输出和错误输出：`&>word`或`>word 2>&1` 
* 追加重定向标准输出和错误输出：`&>>word`或`>>word 2>&1`

## 重定向多行输入
```
cmd << 分隔符
文本
分隔符
```
```
cat << EOF > ceshi.txt
he
jing
qi
EOF
# cat为接收文本的命令
# EOF为分隔符【可以是任意的】
# >ceshi.txt为cat命令的重定向输出
# he jing qi 为输入文本
```

# 括号使用
* (list)：开启一个子shell执行命令列表，$(list)表示list命令的输出结果（也可以使用反撇号执行命令），常用于a=$(list)这样的赋值语句
* ((expression))：括号里的进行的是数字运算(因此可以用+、-、*、/、>、<等算术运算符)，同样的，$((expression))表示计算的结果
* `[]`：条件表达式，其实是一个程序`/usr/bin/[`，相当于/usr/bin/test，后面多的那个`]`只是对称好看，所以`[`后要有空格
* `[[ expression ]]`：表示里面进行的是逻辑运算，可以使用`!、&&、||`逻辑运算符
* { list; }：内部分组，这个结构事实上创建了一个匿名函数。大括号内的命令不会开启一个子shell运行。括号内的命令用分号分隔，最后一个也要用分号，第一个命令和左括号之间必须要有一个空格。它也支持如下扩展：
    - 扩展字母和数字：{1..n} {a..z}
    - 逗号分隔的扩展：{1,5,a}

# 流程控制
* for列表循环：`for name [ [ in [ word ... ] ] ; ] do list ; done`：循环列表中的内容
* for自增循环：`for((expr1;expr2;expr3))`：条件循环；expr1为初始条件，expr2为终止条件，expr为循环条件。
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
1. 读取文件方式1
exec < FILE
while read line
do
    cmd
done

2. 读取文件方式2
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
    - 只能用于for、while、until、select循环
    - 单循环中：break终止循环；continue终止本次循环(不执行循环后语句)，继续下次循环
    - 双循环中：break跳出内循环，执行外循环语句；break 2直接跳出外循环。continue跳出内循环中的本次循环，继续下次内循环；continue 2跳出内循环，继续下次外循环。
* exit `[n]`：以n退出脚本
* return `[n]`：以n作为返回值退出函数

# 数学运算
## 运算符
* id++，id--：变量后加减
* ++id，--id：变量先加减
* +-：加减
* & ^ | ~：位与、异或、同或、非
* `**`：幂函数
* */%：乘除取余
* `<< >>`：向左位移、向右位移
* `> >= <= <`：比较
* == !=：等于、不等于
* && ||  !：逻辑与、或、非
* expr?expr:expr：三元表达式

## 表达式
* (())：echo $((5+4))
* `expr 1 + 4`：空格分隔
* declare：`declare -i i=2;i=i+3;echo $i`
* let：`let a=3+2;echo $a`

# 条件表达式
>test或[命令使用

* -a、-o：逻辑与、或
* -n、-z：字符串长度不为0、为0
* =、!=：字符串相等、不等
* -ne、-eq、-lt、-le、-gt、-ge：数字上的大【等】于、小【等】于、【不】等于
* -e：文件存在

# 内置命令
* `:`：冒号表示空语句，什么也不做
* ./source：在当前shell环境执行脚本，变量值可以回传给shell
* sh/bash：开启子shell执行脚本，变量值不能回传给父shell
* declare：声明变量
    - declare -r variable=value     声明只读变量.
    - declare -a variable=(1 2 3 4)     声明数组
    - declare -i aa                    声明整数型变量
* eval text：将text文本转换为shell命令执行，如【eval "echo text"】即为执行echo text命令
* export：设置全局变量，如export PATH=/data/tomcat:$PATH
* getopts：解析shell脚本位置参数
* shift：解析shell脚本位置参数时用于移动参数位置
* printf：格式化输出
* set：显示所有shell变量
    - -n：读取命令但不执行，此用于检测脚本语法【bash/sh -n file】
    - -x：展开所有简单的命令，同时显示for、case、select的每一个循环，主要用于脚本调试
        + 脚本内使用：set -x
        + 命令行使用：sh -x file
    - +x：关闭命令的展开显示：set +x
* unset：删除变量或函数
* env：显示环境变量

# 命令执行
## 命令列表
* &会将命令放入后台执行
* 以“;”分隔的命令会顺序执行，shell等待每个命令执行完成
* cmd1&&cmd2 逻辑“与”连接两个命令，cmd1执行成功后才执行cmd2
* cmd2||&&cmd2 逻辑“或”连接两个命令，cmd2执行不成功才执行cmd2

# 应用范例
其他范例参考：https://github.com/simple0426/sysadm/tree/master/shell
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


