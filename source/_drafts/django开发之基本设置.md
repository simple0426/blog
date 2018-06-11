---
title: django开发之基本设置
tags: ['URLconf']
categories: ['django']
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
