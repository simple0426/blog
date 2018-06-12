---
title: django开发之基本设置
tags:
  - URLconf
categories:
  - django
date: 2018-06-12 14:27:26
---

# MTV模型
MTV即是通用的web开发模型MVC在django中的实现，  此外django中还有一个URLconf。

* M【models】：建立和操作数据库
* T【templates】：建立和渲染html
* V【views】：连接models和templates，进行逻辑操作
* [URLconf](#URLconf)：匹配相关的url请求，交于后端的views处理

# 项目创建
* 创建project：django-admin startproject webapp
* 创建app：python manage.py startapp books
* 项目目录简介：
    - webapp：项目的根目录
        + webapp：项目的默认app【主要用于项目各种配置】
            * settings.py：项目的主配置文件
            * urls.py：项目的主url配置入口
        + manage.py：项目的命令行工具入口
        + books：项目下books app目录
            * models.py：books app的模型文件
            * views.py：books app的视图文件

# 命令行
* 语法：python manage.py [CMD]
* 可选子命令

|          命令[CMD]          |                      含义                     |
|-----------------------------|-----------------------------------------------|
| runserver 0.0.0.0:80        | 运行django内建的web服务器[默认8000端口]       |
| makemigrations              | 根据models生成表结构                          |
| sqlmigrate                  | 查看用于建表的语句                            |
| migrate                     | 将表结构写入数据库                            |
| dumpdata books > books.json | 导出books的数据【不加app默认导出所有app数据】 |
| flush                       | 清空数据库                                    |
| loaddata books.json         | 导入books数据【不需要设置app】                |
| shell                       | 进入django项目终端环境                        |
| dbshell                     | 根据settings设置进入相应数据库                |
| collectstatic               | 把静态文件收集到 STATIC_ROOT 目录         |
| check                       | 检验数据模型（models）有效性                  |

# settings设置
* DEBUG = True：开启调试模式
* ALLOWED_HOSTS = ['*']：当DEBUG=False时，此值必须设置
* INSTALLED_APPS：此处可以让django自动在app下的templates子目录中查找模板文件，在static子目录中查找静态文件
* BASE_DIR：项目根目录

## 模板目录设置
* TEMPLATES：设置模板目录
    - `'DIRS': [os.path.join(BASE_DIR, 'templates').replace('\\', '/'),],`：设置公有模板目录
    - `'APP_DIRS': True,`：开启app私有的模板目录

## 静态目录设置
* STATIC_URL = '/static/'：使用STATIC_ROOT中的静态文件时使用的url前缀
* STATICFILES_DIRS = (os.path.join(BASE_DIR, "static"),)：设置app公有的静态目录
* STATIC_ROOT = os.path.join(BASE_DIR,'static') ：这个目录配置只在运行collectstaitc时才会用到

## 数据库设置
```
# 安装数据库引擎
pip install pymysql
# pip install psycopg2
# 使用pymysql作为mysql数据库引擎
import pymysql
pymysql.install_as_MySQLdb()
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        # 'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mysite',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'USER': 'root',
        'PASSWORD': '123456',
    }
}
```
## 邮件设置
```
DEFAULT_FROM_EMAIL = 'zabbix@abc.com'
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.exmail.qq.com'
EMAIL_HOST_USER = 'zabbix@abc.com'
EMAIL_HOST_PASSWORD = '******'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
#django邮件发送
from django.core.mail import send_mail
send_mail(subject, body, from_address, list_to_address)
```
## 日志设置
>在进行ORM操作时，在日志中还原为原始SQL

```
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console':{
            'level':'DEBUG',
            'class':'logging.StreamHandler',
        },
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'propagate': True,
            'level':'DEBUG',
        },
    }
}
```
# URLconf
## 语法
url(url正则匹配，views视图函数，参数，别名)
## url正则匹配
### 不分组url
>此时按位置将参数传递给视图函数

```
# urls.py
urls.py：
    url(r'^time/plus/(\d)/$', hours_ahead),
views.py
    def hours_ahead(request, offset):
        now = datetime.now()
        new_date = now + timedelta(hours=int(offset))
        return HttpResponse(str(new_date))
```
### 分组url
>此时按关键词方式将参数传递给视图函数

```
urls.py：
    url(r'^image/getimage/(?P<file_id>\w+)_(?P<image_width>\d+)_(?P<image_height>\d+)', imgview.get_image, name='get_image')
views.py：
    def get_image(request, file_id, image_width, image_height):
        pass
```
## 参数设置
```
urls.py：
    url(r'^foo/$', views.foobar_view, {'template_name': 'template1.html'}),
views.py：
    def foobar_view(request, template_name):
        m_list = MyModel.objects.filter(is_new=True)
        return render_to_response(template_name, {'m_list': m_list})
```

### views参数优先级
* URLconf设置的参数
* 正则匹配捕获的参数【命名分组》无名分组】
* 参数默认值

## 别名设置
### views中使用
```
urls.py
        url(r'^calc/(\d+)/(\d+)/$', calc, name='calc'),
        url(r'^add/(?P<v1>\d+)/(?P<v2>\d+)/$', calc2, name='add'),
views.py
        def calc(request, v1, v2):
            return HttpResponseRedirect(reverse('add', args=(v1, v2)))
        def calc2(request, v1, v2):
            result = int(v1) + int(v2)
            return HttpResponse(str(result))
解释：
1）访问calc时重定向url
2）reverse组合要跳转的url，第一个参数为视图名称【非视图函数名称】，args、kwargs为捕获或设置的参数
```
### template中使用
```
urls.py：
    url(r'^register', views.index, name='reg')
html：
    <form action="{% url "reg" %}">
```
## 路由分发
```
urls.py：
    url(r'^time/', include('contact.urls')),
contact.urls.py：
    url(r'^(?P<year>\d{4})/(?P<month>\d{1,2})/(?P<day>\d{1,2})/', views.current_datetime),
eg：
    http://127.0.0.1:8000/time/2014/12/1/
```

* 主URLconf使用include关键词包含子URLconf
* 主URLconf捕获url的第一部分time，子URLconf对time之后的部分进行匹配
* 当使用include时，主URLconf捕获的命名参数和手动设置的参数将传递给子URLconf的每一行

# Pycharm中运行django
* 创建django项目
* 设置django项目使用的语言解释器【使用virtualenv】
    - 设置位置：file -》settings -》project simplesite-》project Interpreter
* 设置django项目的框架配置
    - 设置位置：file -》settings -》languages & frameworks -》django
    - 开启django支持：enable django support
        + 设置django项目根目录：django project root
        + 设置django项目配置：settings
* 设置server运行配置
    - 设置位置：run -》Edit Configuration
        + 添加一个django server
            * 设置django server名称
            * 设置django server使用的端口
