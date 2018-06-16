---
title: django开发之中间件学习
tags: ['cookie', 'session', '缓存', '中间件', '信号']
categories: ['django']
---
# cookie
## 简单介绍
* 保存在浏览器端的键值对
* cookie依附在请求头或响应头中出现
* 向网站发送请求时，会自动携带网站的cookie信息
* 缺点
    - 明文传输，容易被劫持和篡改
    - 不能用于存储敏感信息
    - 客户端存储，因此cookie可能被拒绝

## 使用方法
* 查看cookie：request.COOKIES、request.COOKIES['id']
* 设置cookie：request.set_cookie
* 测试客户端是否可以存储cookie：request.test_cookie_worked
* cookie参数：
`set_cookie(self, key, value='', max_age=None, expires=None, path='/',domain=None, secure=False, httponly=False)`

|   参数   | 默认值 |           含义          |
|----------|--------|-------------------------|
| max_age  | None   | cookie有效期，单位s     |
| expires  | None   | 指定时间点cookie过期    |
| path     | /      | 只在某个url下cookie有效 |
| domain   | None   | 只在某个域有效          |
| secure   | False  | 为True时只在https下有效 |
| httponly | False  | 只在http下有效          |
## 应用范例
```
def test_cookie(request):
    if request.COOKIES and 'id' in request.COOKIES:
        return HttpResponse('hello %s' % request.COOKIES['id'])
    else:
        ip = request.META['REMOTE_ADDR']
        response = HttpResponse('hello world')
        response.set_cookie('id', ip, max_age=10)
        return response
```

# session
## 简单介绍
* 视图的第一个参数request包含session属性，它是一个字典对象
* 它是基于cookie实现
    - 检测客户端cookie是否可用：request.session.test_cookie_worked()
    - 设置测试cookie：request.session.set_test_cookie()
    - 删除测试的cookie：request.session.delete_test_cookie()      
* 用户登录时，django产生随机字符串作为session的key，将用户设置的session的key/value对进过“加工”后作为session的value在数据库中存储；同时设置一个cookie键值对，key为“sessionid”，value为session的key返回给客户端存储
* 当用户再次登陆时，服务端从cookie中取出sessionid的值作为session的key，根据key在数据库中取出session的value，从而可以验证用户是否已登录

## 使用方法
### 开启session
>项目settings.py

```
INSTALLED_APPS = [
    'django.contrib.sessions',
]
MIDDLEWARE = [
    'django.contrib.sessions.middleware.SessionMiddleware',
]
```
### session配置
* 默认配置
    - django.conf.global_settings中有默认设置，可在settings中覆盖
    - 默认使用数据库存储
    ```
    SESSION_ENGINE = 'django.contrib.sessions.backends.db'   # 引擎（默认）
    SESSION_COOKIE_NAME ＝ "sessionid"                       # Session的cookie保存在浏览器上时的key，即：sessionid＝随机字符串（默认）
    SESSION_COOKIE_PATH ＝ "/"                               # Session的cookie保存的路径（默认）
    SESSION_COOKIE_DOMAIN = None                             # Session的cookie保存的域名（默认）
    SESSION_COOKIE_SECURE = False                            # 是否Https传输cookie（默认）
    SESSION_COOKIE_HTTPONLY = True                           # 是否Session的cookie只支持http传输（默认）
    SESSION_COOKIE_AGE = 1209600                             # Session的cookie失效日期（2周）（默认）
    SESSION_EXPIRE_AT_BROWSER_CLOSE = False                  # 是否关闭浏览器使得Session过期（默认）
    SESSION_SAVE_EVERY_REQUEST = False                       # 是否每次请求都保存Session，默认修改之后才保存（默认）
    ```

* 其他存储方式
    - 使用redis存储[django-redis-sessions]
    ```
    SESSION_REDIS_DB = 5
    SESSION_REDIS_HOST = 127.0.0.1
    SESSION_REDIS_PORT = 6379
    SESSION_ENGINE = 'redis_sessions.session'
    ```
    - 使用cache系统存储
    ```
    SESSION_ENGINES= "django.contrib.sessions.backends.cache"
    SESSION_CACHE_ALIAS = "default"
    ```
    - [其他存储方式](#http://www.cnblogs.com/wupeiqi/articles/5246483.html)

## 应用范例
```
def auth(func):
    def wrapper(request, *args, **kwargs):
        if not request.session.get('yyyy'):
            return redirect('login.html')
        return func(request, *args, **kwargs)
    return wrapper

@auth
def ok(request):
    user = request.session.get('yyyy')
    return render(request, 'ok.html', context={'user':user})

@csrf_exempt
def login(request):
    if request.session.get('yyyy'):
        return redirect('ok.html')
    if request.method == 'POST':
        if request.session.test_cookie_worked():
            request.session.delete_test_cookie()
            user = request.POST.get('user')
            pwd = md5.encrypt(request.POST.get('passwd'))
            obj = UserInfo.objects.filter(username=user, password=pwd).first()
            if obj:
                request.session['yyyy'] = user
                return redirect('ok.html')
            else:
                return render(request, 'login.html', {'msg': '用户名或密码错误'})
        else:
            return HttpResponse('请开启cookie')
    request.session.set_test_cookie()
    return render(request, 'login.html')

@auth
def logout(request):
    del request.session['yyyy']
    return redirect('login.html')
```
## 扩展
### csrf装饰器
* 导入：from django.views.decorators.csrf import csrf_exempt, csrf_protect
* csrf_exempt：被装饰的函数不使用csrf【csrf中间件开启时，全站使用csrf验证】
* csrf_protect：被装饰的函数使用csrf【csrf中间件屏蔽时，全站禁用csrf验证】

### 查看session存储内容
```
from django.contrib.sessions.models import Session
session = Session.objects.get(session_key='90nv2o99bykw3qtqotrlt947c2pnx75f')
# 解码session内容
session.get_decoded()
# 查看session过期时间
session.expire_date
# 查看session内容
session.session_data
```