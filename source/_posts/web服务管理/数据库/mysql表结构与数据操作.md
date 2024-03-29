---
title: mysql表结构与数据操作
tags:
  - mysql
  - 索引
  - 存储过程
categories:
  - mysql
date: 2018-05-14 16:13:01
---

# 库操作
* 查看存在的库：show databases;
* 建数据库：create database olgboy_utf8 default character set utf8 collate utf8_general_ci;
    - 不存在再建立：create database if not exists oldboy_default; 
* 进入数据库：use db1
* 删除数据库： drop database db1;
    - 存在才删除：drop database if exists test1;

# 数据类型
## 数字
* tinyint：1字节
    - 无符号：(0，255)
    - 有符号：(-128，127)
* smallint：2字节
    - 无符号：(0，65 535)
    - 有符号：(-32 768，32 767)
* int：4字节
    - 无符号：(0，4 294 967 295)
    - 有符号：(-2 147 483 648，2 147 483 647)
* bigint：8字节
* float：4字节
* double：8字节
* decimal【精确的小数】

## 字符串
* char：定长存储，最长255
* varchar：不定长存储，最长65535
* text：存储文本，最长(2**16-1)65535
* mediumtext：最长（2**24-1）
* longtext：最长4GB（2**32-1）

## 时间
* date【日期】：1000-01-01/9999-12-31
* time【时间】：'-838:59:59'/'838:59:59'
* datetime【日期 时间】：1000-01-01 00:00:00/9999-12-31 23:59:59
* year：1901/2155
* timestamp【时间戳、4字节存储】：开始时间0表示1970-01-01 00:00:00，结束时间是第 2147483647 秒【北京时间 2038-1-19 11:14:07（格林尼治时间 2038年1月19日 凌晨 03:14:07）】

# 新建表
## 范例
```
CREATE TABLE subject_comment_manager (
  subject_comment_manager_id bigint(12) NOT NULL auto_increment COMMENT '主键',
  subject_type tinyint(2) NOT NULL COMMENT '素材类型',
  subject_primary_key varchar(255) NOT NULL COMMENT '素材的主键（词条是名称，文章是iden，组图是id，视频是MD5）',
  subject_title varchar(255) NOT NULL COMMENT '素材的名称',
  edit_user_nick varchar(64) default NULL COMMENT '修改人',
  edit_user_time timestamp NULL default NULL COMMENT '修改时间',
  edit_comment varchar(255) default NULL COMMENT '修改的理由',
  state tinyint(1) NOT NULL default '1' COMMENT '0代表关闭，1代表正常',
  PRIMARY KEY  (subject_comment_manager_id),
  KEY IDX_PRIMARYKEY (subject_primary_key(32)), 
  KEY IDX_SUBJECT_TITLE (subject_title(32)),
  KEY index_nick_type (edit_user_nick(32),subject_type)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```
```
create table tb1(
    id int not null auto_increment primary key, 
    name char(20),
    age int default 18,
    gender char(1)
    )engine=innodb default charset='utf8' comment='跟进表（基于客户、联系人、询盘）';
```
## 字段
| 字段名 |                    数据类型                   |   是否为空  | 是否为[主键](#主键) |   是否有默认值  | 是否[自增](#自增) | [外键约束](#外键) |
|--------|-----------------------------------------------|-------------|---------------------|-----------------|-------------------|-------------------|
| -      | [数字](#数字)/[字符串](#字符串)/[时间](#时间) | 【not】null | primary key         | default 【xxx】 | auto_increment    | constraint        |

### 自增
* 关键字：auto_increment
* 限制：一个表只能有一个自增列

### 主键
* 约束：不能为空，不能重复
* 索引：对字段建立索引可以加速查找

### 外键
* 约束：只能是某个表中已经存在的字段
* 语法：constraint 约束名称 foreign key(本表字段名) references 外表(外表字段名)

## 批量建表
>使用like语法创建结构一样的表：create table tb_name1 like tb_name2

```
import pymysql

tb_list = ['c_fu', 'c_fu_file', 'e_email', 'e_email_body', 'e_email_file', 'e_email_to', 'e_email_track', 'f_mem_upload']
conn = pymysql.connect(host='127.0.0.1', user='user', password='pass_str', database='crm_big3', charset='utf8')
for index in range(2001, 2003):
    for table in tb_list:
        cursor = conn.cursor(cursor=pymysql.cursors.Cursor)
        cursor.execute('create table %s_%s like %s_2000' % (table, index, table))
        cursor.close()
conn.close()
```

# 修改表
## 表名修改
alter table 表名 rename 新表名;
## 字段类型修改
* 修改字段类型：alter table 表名 modify 列名 列类型;
* 修改字段名及字段类型：alter table 表名 change 列名 新列名 列类型;

## 增加字段
* 增加字段到其他字段末尾【默认】：alter table 表名 add 列名 列类型 
* 添加字段为第一个字段：alter table 表名 add 列名 列类型 FIRST
* 在指定的字段后添加字段：alter table 表名 add 列名 列类型  AFTER col_name

## 删除字段
alter table 表名 drop 列名;

## 默认值
* 设置字段默认值：ALTER TABLE tbl_name ALTER col_name SET DEFAULT literal
* 删除字段默认值：ALTER TABLE tbl_name ALTER col_name  DROP DEFAULT

# 删除表
* 删除空表：drop table tb1;
* 删除表数据：delete from tb1;
    - 自增列会继续之前的ID
* 清空表：truncate table tb1;
    - 物理删除，速度快，重新计算ID

# 查看表
* 查看存在的表：show tables;
* 表结构：desc tb_name;
* 建表语句：show create table tb_name;

# 多表范例
## 一对多关系
```
create table department(
    id int not null auto_increment primary key,
    title char(20)
    );
create table userinfo(
    id int not null auto_increment primary key, 
    name char(20),
    age int default 18,
    gender char(1),
    department_id int,
    constraint user_depar foreign key (department_id) references department(id)
    )engine=innodb default charset='utf8';
```
## 多对多关系
```
create table boy(
    id int not null auto_increment primary key,
    name char(32)
    )engine=innodb default charset='utf8';
create table girl(
    id int not null auto_increment primary key,
    name char(32)
    )engine=innodb default charset='utf8';
create table b2g(
    id int not null auto_increment primary key,
    bid int,
    gid int,
    constraint to_boy foreign key (bid) references boy(id),
    constraint to_girl foreign key (gid) references girl(id)
    )engine=innodb default charset='utf8';
```

# 数据操作
## 增加
* 单行数据：`insert into tb1(name,age,gender) values('hejingqi',30,'0');`
* 多行数据：`insert into tb1(name,age,gender) values('yuanshuo', 24, '1'),('yangxiaomeng', 29, '1');`

## 删除
* 精确删除：`delete from tb1 where name='hejingqi';`

## 修改
* 单字段修改：`update tb1 set age=18 where id=1;`
* 多字段修改：`update tb1 set id=2,age=30 where name='yangxiaomeng';` 

# 数据查询
## 排序（order by）
* 从大到小：select * from score order by number desc;
* 从小到大：select * from score order by number asc;

## 限定数量（limit）
* 输出前2行：select * from score limit 2;
* 从第1个之后取2个：select * from score limit 1,2;

## 模糊匹配（like）
* 单字符匹配（`_`）：select * from class where caption like '_三一班';
* 多字符匹配（`%`）:select * from class where caption like '三%';

## 左右连表（join）
* INNER JOIN即为JOIN，只保留2个表均有的数据
* LEFT JOIN以左边的表为主输出数据

```
SELECT
    student.sname AS '姓名', # '指明表和列'
    class.caption AS '班级'
FROM 
    student #查的主表
LEFT JOIN class ON student.class_id = class.cid;
# LEFT JOIN要连接的表
# ON 2个表的桥接
```
```
SELECT
    student.sname
FROM
    student
LEFT JOIN score ON score.student_id = student.sid
LEFT JOIN cource ON score.course_id = cource.cid # 多表关联
WHERE cource.cname = '体育'; # 连表之后条件过滤
```
## 分组（group by）
* 先分组，再过滤

```
SELECT
  class.caption as '班级',
    count(class_id) as '数量' #对重复数据的聚合处理：count|max|min|avg
FROM
    student
LEFT JOIN class ON class.cid = student.class_id
GROUP BY # 分组依据
    student.class_id
HAVING count(class_id) > 1; #先分组，然后对分组进行二次过滤
```

* 先过滤，再分组

```
SELECT
  class.caption as '班级',
    count(class_id) as '数量' #对重复数据的聚合处理
FROM
    student
LEFT JOIN class ON class.cid = student.class_id
WHERE class_id > 1          #先使用条件过滤再进行分组
GROUP BY # 分组依据
    student.class_id;
```

## 上下连表（union）
```
select sid,sname from student union select tid,tname
 from teacher;
```

## 其他用法
* 结果无重复的查询（distinct ）：select distinct 列名 from 表名 where 条件；
* 拼接查询结果（concat）：select concat (id, name, score) as info from tt2;
* 别名设置（as）
* 聚合函数：count、sum、max、min、avg

# 操作符
* 逻辑操作
    - or
    - and
    - in
    - not
    - between 。。。and。。。
* 比较操作
    - `>`
    - `>=`
    - `<`
    - `<=`
    - `=`

# 索引
## 介绍
* 一般对select查询的where条件列建立索引
* 可以对多个字段、一个字段、一个字段的前n个字符建立索引
* 由于复合索引的前缀特性，索引内的字段顺序很重要【一般将常用的列放在前边】
* 不应当建立索引的情况
  - 频繁更新的字段【数据插入时，也需要更新索引，这会降低更新操作性能】
  - 数据量小的表，或字段内容较少的字段（如性别、旗标）
  - 字段值为空【此时，查询不走索引】
  - 字段为主键【主键创建时默认会创建索引】

## 管理
* 建立索引：
  - create table test1(id int not null,name char(20),sex char(3),primary key(id),index k_name(name(2))); 
  - alter table test add index index_name(column1,column2...);
  - create index index_name on test(name); 
* 删除索引
  - alter table test DROP INDEX index_name
  - drop index index_Sage on student;
* 查询是否有索引：show crate table tb_name
* 查询是否使用索引：explain select \* 。。。

# 存储过程
>批量建表的范例

```
delimiter // -- 定义分界符
drop procedure if exists create_batch_table; -- 删除已经存在的存储过程
create procedure create_batch_table() -- 创建存储过程
begin -- 开始存储过程
declare i int; -- 定义变量类型
set i = 2001; -- 设置变量
while i < 2005 do -- 循环开始
  set @create=CONCAT('create table c_fu_', i , ' like c_fu_2000;'); -- 定义创建表的语句
  select @create; -- 显示建表语句
   prepare tmt from @create; -- 预编译sql语句
   execute tmt; -- 执行sql语句
   deallocate prepare tmt;  -- 收回sql游标cursor
  set i = i + 1; -- 循环自增
end while; -- 循环结束
end // -- 结束存储过程
call create_batch_table(); -- 调用存储过程
```
# python使用
>pymysql模块

```
import pymysql

conn = pymysql.connect(host='127.0.0.1', user='kingold', password='zjht098_kingold',
                       database='db1', charset='utf8')
cursor = conn.cursor(cursor=pymysql.cursors.DictCursor)
# 查询
cursor.execute('select name,age,gender from tb1')
# 取所有结果
res1 = cursor.fetchall()
# 取一个结果
res2 = cursor.fetchone()
# 取指定数目结果
res3 = cursor.fetchmany(2)
# 插入
cursor.execute('insert into tb1(name, age, gender) values(%s, %s, %s)', ('hejingqi', 30, '0'))
# 删除
cursor.execute('delete from tb1 where name=%s', ('yangxiaomeng'))
cursor.close()
# 增删改数据时提交变更
conn.commit()
conn.close()
```
