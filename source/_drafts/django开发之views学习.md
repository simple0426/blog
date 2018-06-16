---
title: django开发之views学习
tags:
categories:
---
# request请求内容
* request.method：请求方法
* request.GET：对收到的get请求进行参数解析
* request.POST：对收到的post请求进行参数解析
    - 仅当Content-Type：application/x-www-form-urlencoded
* request.Meta：请求的元数据信息
* request.body：请求体内容
    - get请求体为空
    - post请求体范例：
    `b'csrfmiddlewaretoken=SP3GO7LEfUb0QqWwE4Rq0r0W&user=he&passwd=12'`

# 类视图
## 简单介绍
* 使用类方式实现通用视图，和使用函数方式相比，类能更方便的实现继承和mixins
* 类视图在URLconf中的实现：
    - 调用类的as_view方法，比如`url(r'^login', views.LoginView.as_view())`
    - 接受request并实例化，返回实例的dispatch方法
    - 实例的dispatch方法根据请求的类型返回同名的处理函数
* 类视图在增加功能时【比如授权登录】，可以采取两种方式：
    - [类的多重继承方式](#基础功能类)
    - [装饰器方式](#基础功能装饰器)
        + 当类视图使用装饰器时，必须使用django内置的装饰器方法【比如method_decorator】
        + 在类视图使用csrf装饰器时，必须在dispatch方法前使用
        + 类视图装饰器可以放在类前【必须指明要装饰的具体函数名称】，也可以放在类下的方法前

## 应用范例
### 基础功能类
```
class BaseView(View):
    def dispatch(self, request, *args, **kwargs):
        if request.session.get('username'):
            response = super(BaseView, self).dispatch(request, *args, **kwargs)
            return response
        else:
            return redirect('login.html')
```
### 基础功能装饰器
```、
def auth(func):
    def wrapper(request, *args, **kwargs):
        if request.session.get('username'):
            obj = func(request, *args, **kwargs)
            return obj
        else:
            return redirect('login.html')
    return wrapper
```
### 主功能组件
```
from django.utils.decorators import method_decorator
class LoginView(View):
    @method_decorator(csrf_exempt)
    def dispatch(self, request, *args, **kwargs):
        response = super(LoginView, self).dispatch(request, *args, **kwargs)
        return response
    def get(self, request, *args, **kwargs):
        return render(request, 'login.html')
    def post(self, request, *args, **kwargs):
        user = request.POST.get('user')
        pwd= md5.encrypt(request.POST.get('passwd'))
        obj = UserInfo.objects.filter(username=user, password=pwd).first()
        if obj:
            request.session['username'] = user
            return redirect('ok.html')
        return render(request, 'login.html', {'msg': '用户或密码错误'})

# class OkView(BaseView, View): # 可以使用类的多重继承凡是增加功能【比如登录授权】
@method_decorator(auth, name='get')
# 在类前使用装饰器必须指明要装饰的具体函数名称，比如get、post等
class OkView(View):
    def get(self,request,*args, **kwargs):
        return render(request, 'ok.html', {'user': request.session['username']})

# class LogoutView(BaseView, View):
class LogoutView(View):
    # 装饰器也可放在类下的方法前
    @method_decorator(auth)
    def get(self, request, *args, **kwargs):
        del request.session['username']
        return redirect('login.html')
```
