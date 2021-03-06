---
title: django开发之models学习
tags:
  - models
  - ORM
  - admin
categories:
  - django
date: 2018-06-14 16:38:58
---

# ORM介绍
## ORM介绍
* Object Relational Mapping(对象关系映射)：是一种程序技术，用于实现面向对象编程语言里不同类型系统的数据之间进行转换。
* 在django中主要实现方式为：在modles文件中定义类，通过映射关系和相关命令转换为对数据库中对应的对象进行操作。

## django中的映射关系
|     程序     |  数据库  |
|--------------|----------|
| 类名         | 表名     |
| 属性         | 字段名   |
| 类实例化对象 | 数据记录 |

## ORM功能
* [创建数据表](#创建数据表)
* [操作数据](#数据操作)

# 前置配置
* 创建数据库，设置相应的用户和权限
* 安装python连接数据库的引擎【如连接mysql的pymysql】
* 在settings文件中配置[数据库相关选项](/2018/06/12/django开发之基本设置/#数据库设置)

# 创建数据表
## 数据库建模
* 作者（Author）和作者详情（AuthorDetail）为一对一关系（OneToOneField）
* 出版社（Publisher）和书籍（Book）为一对多关系（ForeignKey）
* 书籍（Book）和作者（Author）为多对多关系（ManyToManyField）

## 表结构范例
>modles.py文件

```
class Publisher(models.Model):
    name = models.CharField(max_length=30, verbose_name='名称')
    address = models.CharField("地址", max_length=50)
    city = models.CharField("城市", max_length=60)
    state_province = models.CharField(max_length=30)
    country = models.CharField(max_length=50)
    website = models.URLField()

    class Meta:
        verbose_name = '出版商'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name

class Author(models.Model):
    name = models.CharField(max_length=30)

    def __str__(self):
        return self.name

class AuthorDetail(models.Model):
    sex = models.BooleanField(max_length=1, choices=((0, '男'), (1, '女')))
    email = models.EmailField()
    address = models.CharField(max_length=50)
    birthday = models.DateField()
    author = models.OneToOneField(Author)

class Book(models.Model):
    title = models.CharField(max_length=100)
    authors = models.ManyToManyField(Author)
    publisher = models.ForeignKey(Publisher)
    publication_date = models.DateField()
    price = models.DecimalField(max_digits=5, decimal_places=2, default=10)

    def __str__(self):
        return self.title
```
## 数据库建表
* 创建执行SQL的脚本：python manage.py makemigrations
* 查看建表SQL语句：python manage.py sqlmigrate app01 0001
* 执行建表SQL脚本：python manage.py migrate

# 数据操作
## 增加
### 基础操作方式
* create方法：
    `Author.objects.create(name='he')`
* save方法
    ```
    author = Author(name='jing')
    author.save()
    ```

### 一对多关系操作方式
>含有外键的表

* 获取外键字段对象【出版社】  
    `pub_obj=Publisher(name='河大出版社',address='保定',city='保定',state_province='河北',country='China',website='http://www.hbu.com')`
* 在添加数据时绑定外键字段
    `Book.objects.create(title='php', publication_date='2017-7-7', price=99, publisher=pub_obj)`

### 多对多关系操作方式
>实质为关系表数据增加
#### 正向操作
>以外键所在表为操作对象，例如book

* 获取作者对象：`author = Author.objects.filter(id__gt=1)`
* 获取书籍对象：`book = Book.objects.get(id=1)`
* 在书籍处添加作者【一本书多个作者】：`book.authors.add(*author)`
    - 删除作者：`book.authors.remove(*author)`

#### 反向操作 
>从外键所映射的主键所在的表为操作对象，例如author

* 获取书籍对象：`book1 = Book.objects.filter(id__gt=1)`
* 获取作者对象：`author1 = Author.objects.get(id=2)`
* 作者处添加书籍【一个作者有多本书】：`author1.book_set.add(*book1)`
    - 删除书籍：`author1.book_set.remove(*book1)`

## 删除
* Modle对象删除：Author.objects.get(id=1).delete()
* Queryset对象删除：Author.objects.filter(id=1).delete()

## 修改
* update方法
    - 只能用于queryset对象【比如filter、all的结果】
    - update方法只对变更的属性赋值，效率较高
    `Author.objects.filter(id=1).update(name='yang')`
* save方法
    - 对modle对象进行操作【比如get的结果】
    - save方法会对所有属性重新赋值，效率较低
    ```
    author = Author.objects.get(id=2)
    author.name = 'meng'
    author.save()
    ```

## 查询
### 查询类API
* filter：获取所有匹配的queryset对象
* all：获取所有的queryset对象
* get：获取指定条件的modle对象【没有或多于1个时则报错】

### 过滤型API
* value：获取由指定字段组成的queryset对象
    `Book.objects.filter(price__gt=12).values('price')`
* exclude：排除指定条件的数据
    `Book.objects.all().exclude(title='java')`
* count：返回指定条件的结果数量
* `[]`(切片)：返回指定数量的结果
    `Book.objects.all().order_by('title')[:2]`
* first/last：返回结果集中的第一条或最后一条数据
* exists：判断返回的结果中是否有数据【True或False】
* order_by：根据指定字段对结果排序
    - 字段名前加减号【-】为反向排序，字段可以有多个
* reverse：反向排序
* distinct：结果去重

### 下划线语法
#### 条件查询
* contains：包含指定字符串的数据【icontains不区分大小写】
    `Book.objects.filter(title__contains='php')`
* regex：正则查询【iregex不区分大小写】
    `Book.objects.filter(title__regex='^p')`
* lt/gt：数值比较
    `Author.objects.filter(id__gt=1)`
* range/in：范围与区间查询
    `Author.objects.filter(id__in=[1, 3])`
    `Author.objects.filter(id__range=[1, 3])`

#### 跨表使用
* 正向查询【从外键所在的表查询主键所在表中的数据，跨表时使用表内的外键字段名】
    `Book.objects.filter(title__icontains='php').values('publisher__name')`
* 反向查询【从主键所在的表查询外键所在表中的数据，跨表时使用另一个表的表名】
    `Publisher.objects.filter(id=1).values('book__title')`

### 聚合和分组
>from django.db.models import Avg, Sum

* aggregate：对返回的结果进行聚合运算【平均值，最大值，最小值】
    `Book.objects.all().aggregate(avg=Avg('price'))`
* annotate：对返回的结果的每一个分组分别进行聚合运算
    `Book.objects.values('authors__name').annotate(Sum('price'))`

### F和Q查询
>from django.db.models import F, Q

* F查询：保存中间状态值
    `Book.objects.all().update(price=F('price') + 10)`
* Q查询：根据`&|~`【与或非】逻辑组合查询条件
    `Book.objects.filter(Q(id__gt=1)&Q(title='python'))`

# admin site
主要功能是：在web页面操作Django ORM
## 前置条件
* settings中默认配置【INSTALLED_APPS和MIDDLEWARE_CLASSES】
* urls中保持默认配置【存在admin路由】
* 数据库中存在django相关表
* 后台汉化
    - 'django.middleware.locale.LocaleMiddleware'【SessionMiddleware之后添加】

## 新建用户
python manage.py createsuperuser

## 注册模块
### 显示逻辑
* list_display：可显示的字段
* list_display_links：可点击编辑的字段
* search_fields：可搜索字段
* list_filter：在字段列表右侧显示可用于过滤的字段
* date_hierarchy：在字段列表上方显示日期过滤参数
* ordering：默认排序方式【优先级左右依次降低】
* filter_horizontal：设置多对多字段使用水平的可选控件
* raw_id_fields：设置外键使用可搜索的文本框形式而非列表【降低列表加载带来的系统开销】
* actions：执行函数功能

### 范例
```
from app01.models import *
class BookAdmin(admin.ModelAdmin):
    list_display = ['title', 'price', 'publisher', 'publication_date']
    list_display_links = ['title']
    search_fields = ['title', 'publisher']
    list_filter = ['publication_date']
    date_hierarchy = 'publication_date'
    ordering = ['-publication_date']
    filter_horizontal = ['authors']
    raw_id_fields = ['publisher']
    def func(self, request, queryset):
        print(self, request, queryset)
        print(request.POST.getlist('_selected_action'))
    func.short_description = '显示结果'
    actions = [func,]

admin.site.register(Book, BookAdmin)
admin.site.register(Author)
```
## 登录后台
http://localhost/admin

# 多数据库联用
## settings设置
```
# 数据库设置
DATABASES = {
    # default为默认数据库，具有特殊含义
    'default': {
        'NAME': 'app_data',
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'USER': 'postgres_user',
        'PASSWORD': 's3krit'
    },
    'users': {
        'NAME': 'user_data',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'mysql_user',
        'PASSWORD': 'priv4te'
    }
}
# 数据库路由设置[类名]
DATABASE_ROUTERS = ['utils.dbroute.DBRouter']
```
## 路由设置
```
class DBRouter:
    # 指定读数据库路由
    def db_for_read(self, model, **hints):
        app_label = model._meta.app_label
        if app_label in ('order', 'payment'):
            return 'order'
        if app_label in ('xcass_passport', 'store_manage'):
            return 'xcass'
        return 'default'
    # 指定写数据库路由
    def db_for_write(self, model, **hints):
        app_label = model._meta.app_label
        if app_label in ('order', 'payment'):
            return 'order'
        if app_label in ('xcass_passport', 'store_manage'):
            return 'xcass'
        return 'default'
    # 如果obj1 和obj2 之间应该允许关联则返回True，如果应该防止关联则返回False，
    # 如果路由无法判断则返回None
    def allow_relation(self, obj1, obj2, **hints):
        if obj1._meta.app_label == obj2._meta.app_label:
            return True
    # 定义迁移操作是否允许在别名为db的数据库上运行。如果操作应该运行则返回True，
    # 如果不应该运行则返回False，如果路由无法判断则返回None
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if app_label in ('order', 'payment'):
            return db ==  'order' 
        if app_label in ('xcass_passport', 'store_manage'):
            return db == 'xcass'
        return db  ==  'default' 
```
## 数据对象操作
* queryset对象：Book.objects.using('default').all()
* Models对象
    ```
    author = Author(first_name='yashuai', last_name='gao')
    author.save(using='default')
    ```

## 数据导入导出
* 数据同步【建表】：python manage.py migrate --database=test
* 数据导出：python manage.py dumpdata app1 --database=db1 > app1_fixture.json
* 数据导入：python manage.py loaddata app1_fixture.json --database=db1
