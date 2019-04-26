---
title: django开发之views学习
tags:
  - CBV
  - Form
  - request
categories:
  - django
date: 2018-06-29 12:43:53
---

# request内容
## 请求路径
* request.path：除域名以外的请求路径，以斜杠开头
* request.get_host()：主机名（比如通常所说的域名）
* request.get_full_path()：请求路径，可能包含查询字符串
* request.is_secure()：请求方法是否是https，是则返回True

## 请求内容
* request.method：请求方法
* request.GET：对收到的get请求进行数据【来自form或url中的查询串】解析，它是一个类字典对象
* request.POST：对收到的post请求进行数据【来自html中的form】解析，他是一个类字典对象
    - 仅当Content-Type：application/x-www-form-urlencoded
* request.body：请求体内容
    - get请求体为空
    - post请求体范例：
    `b'csrfmiddlewaretoken=SP3GO7LEfUb0QqWwE4Rq0r0W&user=he&passwd=12'`
* request.Meta：它是一个python字典，包含了本次请求的所有header信息，可以通过get请求获得【防止异常退出】
    - HTTP_REFERER，进站前链接网页，如果有的话。
    - HTTP_USER_AGENT，用户浏览器的user-agent字符串，如果有的话。
    - REMOTE_ADDR 客户端IP

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
```
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
# Form表单
* 自动生成html标签，并对字段进行校验
* 表单框架最主要的用法是：为HTML下每一个将要处理的表单定义一个Form类

## Form对象
### 表单要素
* 字段类型
    - CharField：文本框
    - DateField：日期选择
    - DecimalField：数字
    - MultipleChoiceField：多选框
    - ChoiceField：单选框
    - FileField：文件选择框
* 字段参数
    - label：标签显示内容
    - required：是否必选
    - initial：初始值
    - min_length/max_length：最大最小长度
    - error_messages：错误提示
    - widget：插件
    - choices：可选项【choice相关字段类型】
    - validators：简单的验证规则
* 常用插件
    - TextInput：默认插件
    - RadioSelect：单选插件
    - CheckboxSelectMultiple：多选插件
    - Textarea：多行文本框
    - PasswordInput：密码类型输入框

### 使用范例
```
class BookForm(Form):
    def __init__(self, *args, **kwargs):
        super(BookForm, self).__init__(*args, **kwargs)
        # 初始化时从数据库获取信息
        self.fields['authors'].choices = Author.objects.values_list('id', 'name')
        self.fields['publisher'].choices = Publisher.objects.values_list('id', 'name')

    title = fields.CharField(
        required=True,
        label='书名',
        widget=widgets.Input(
            attrs= {'class': 'c1', 'style': 'color: red'},
        )
    )
    publication_date = fields.DateField(
        required=True,
        label='出版日期',
        widget=widgets.SelectDateWidget(
            years=range(2018, 1970, -1),
        )
    )
    price = fields.DecimalField(
        required=True,
        label='价格'
    )
    authors = fields.MultipleChoiceField(
        label='作者',
        choices=[],
    )
    publisher = fields.ChoiceField(
        label='出版社',
        choices=[],
    )
```



## Form规则验证
### 使用方法
 |           位置           |                                           使用方法                                          |    验证级别    |           是否有返回值           |      优先级      |
 |--------------------------|---------------------------------------------------------------------------------------------|----------------|----------------------------------|------------------|
 | 类字段下的validators参数 | 1. RegexValidator对象【只支持正则判断】<br> 2.  自定义函数 【支持逻辑判断，错误时抛出异常】 | 字段级别       | 有错误时抛出异常                 | 在默认规则前生效 |
 | 类下的clean_字段         | 类下的clean_字段【支持逻辑判断和与数据库联动】                                              | 字段级别       | 返回字段信息，同时有错误抛出异常 | 在默认规则后生效 |
 | 类下的clean方法          | 类下的clean方法                                                                             | 表单级别多字段 | 返回表单数据，同时有错误抛出异常 | 在默认规则后生效 |

### 使用范例
```
from django.core.exceptions import ValidationError
from django.core.validators import RegexValidator

def validate_username(msg):
    if 'test' in msg:
        raise ValidationError('关键词不能包含test')

class UserForm(Form):
    username = fields.CharField(
        min_length=6,
        max_length=20,
        required=True,
        error_messages={
        "required": '用户名不能为空',
        },
        validators=[RegexValidator(r'^(\D)', '不能数字开头'),
                    validate_username]
    )
    password = fields.CharField(
        required=True,
        error_messages={
            "required": '密码不能为空',
        },
        widget=widgets.PasswordInput(render_value=True),
    )
    password_confirm = fields.CharField(
        required=True,
        error_messages={
            "required": '密码不能为空',
        },
        widget=widgets.PasswordInput(render_value=True),
    )
    email = fields.EmailField(
        required=True,
        error_messages={
            "required": '邮箱不能为空',
            "invalid": '邮箱格式错误',
        },
    )
    user_type = fields.ChoiceField(
        required=True,
        choices=(('1', u'普通用户'), ('2', u'超级用户')),
    )
    # 字段级别验证
    def clean_username(self):
        username = self.cleaned_data['username']
        if not re.match(r'^(\D)', username):
            raise ValidationError('不能数字开头')
        if 'test' in username:
            raise ValidationError('关键词不能包含test')
        return username
    # 表单级别多字段验证
    def clean(self):
        # 调用父类初始化
        cleaned_data = super().clean()
        # get方法避免取空值
        password = cleaned_data.get('password')
        password_confirm = cleaned_data.get('password_confirm')
        if password == password_confirm:
            return self.cleaned_data
        else:
            # 可以将表单验证结果绑定在特定字段
            self.add_error('password_confirm', ValidationError('密码输入不一致'))
            return self.cleaned_data
```

## Form对象实例化
### 使用要点
* 初始化：data = UserForm(initial=xxx)
* 实例化：data = UserForm(data=request.POST)
* 调用验证规则：data.is_valid()
* 获取校验后的数据：data.cleaned_data
* 返回的错误信息：data.errors/data.errors.字段.字段错误索引

### 使用范例
```
def book_admin(request, book_id=None):
    # 图书信息变更
    if request.method == 'POST':
        data = BookForm(data=request.POST)
        if data.is_valid():
            data = data.cleaned_data
            # 设置书籍出版社【一对多关系】
            data['publisher'] = Publisher.objects.get(id=data['publisher'])
            # 获取作者id列表
            authors_id_list = data.pop('authors')
            # 获取作者对象列表
            authors_list = Author.objects.filter(id__in=authors_id_list)
            # 编辑图书
            if book_id is not None and Book.objects.filter(id=int(book_id)).exists():
                query = Book.objects.filter(id=int(book_id))
                # 更新其他信息
                query.update(**data)
                # 更新作者信息
                book_obj = query.first()
                book_obj.authors.clear()
                book_obj.authors.add(*authors_list)
                return HttpResponseRedirect('book_admin_%s' % book_id)
            # 添加书籍
            else:
                # 添加书籍
                new_book = Book.objects.create(**data)
                # 书籍对象处添加作者【多对多关系】
                new_book.authors.add(*authors_list)
                book_id = str(new_book.id)
                return HttpResponseRedirect('book_admin_%s' % book_id)
    # 显示书籍
    elif book_id is not None:
        book_obj = Book.objects.filter(id=int(book_id)).first()
        if book_obj:
            authors_list = book_obj.authors.values_list('id')
            authors_id_list = list(zip(*authors_list))[0]
            book_info = {
                'title': book_obj.title,
                'publication_date': book_obj.publication_date,
                'price': book_obj.price,
                'authors': authors_id_list,
                'publisher': book_obj.publisher.id
            }
            data = BookForm(initial=book_info)
        else:
            return HttpResponseRedirect('book_admin')
    # 默认显示空表格
    else:
        data = BookForm()
    return render(request, 'book_admin.html', {'form': data})
```
## HTML渲染
### 使用要点
* `<form method="post" novalidate>`：novalidate关闭浏览器验证
* form.as_p：循环模式下生产p标签，由于标签格式固定不利于css渲染
* form.username：获取表单字段
* form.errors.username.0：获取表单字段的错误信息

### 错误提示
* 表单内容较多时
    - 此时表单验证使用前台提交(url跳转)方式【验证时页面刷新】
    - 由于页面刷新，所以可以在页面中内置错误变量，刷新后显示错误
* 表单内容较少时
    - 如场景：模态对话框，此时表单验证使用ajax方式【验证时页面无刷新】
        + 验证成功时，前台处理url跳转
    - 由于页面无刷新，页面内的错误变量无法被渲染，所以此时需要使用DOM新建错误标签后显示

### ajax范例
```
<form>
    {% csrf_token %}
    <p>书名：{{ form.title }} </p>
    <p>出版日期：{{ form.publication_date }}</p>
    <p>价格：{{ form.price }}</p>
    <p>作者: {{ form.authors }} </p>
    <p>出版社：{{ form.publisher }}</p>
    <p><input type="button" value="提交" class="submit"></p>
</form>
<script>
    $('.submit').click(function () {
        $.ajax({
            url: 'book_ajax',
            type: 'POST',
            data: $("form").serialize(),
            dataType: "json",
            success:function (data) {
                if(data.status){
                    // ajax处理页面跳转
                    location.href = 'book_admin_3';
                }else{
                    $.each(data.Error, function (k, v) {
                        var tag = document.createElement('span');
                        tag.innerHTML = v[0];
                        tag.className = 'error';
                        $('input[name="'+k+'"]').after(tag);
                    })
                }
            }
        })
    })
</script>
```
```
def book_ajax(request):
    if request.method == 'POST':
        response = {'status': True, 'Error': None}
        data = BookForm(request.POST)
        if data.is_valid():
            data = data.cleaned_data
            # ajax跳转在前台处理，此处redirect无用
        else:
            response['status'] = False
            response['Error'] = data.errors
        return HttpResponse(json.dumps(response))
    else:
        data = BookForm()
        return render(request, 'book_ajax.html', {'form': data})
```
