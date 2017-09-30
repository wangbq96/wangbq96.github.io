---
title: 记一个简单的wiki系统的开发过程
tags:
- python
- Django
- 已完结
categories:
- 技术
- 项目
author: 汪博全
date: 2017-02-27 20:29:00
---

> 已完结

## 1 前言
作为第一个较为完整的Django作品，还是有必要记录一下开发过程的。

<!-- more -->

## 2 环境
python 3.5.2
mysql 5.6
django 1.10
nginx 1.10.2
Markdown (2.6.7)
gunicorn (19.6.0)
pycharm 2016.2.3

## 3 开发
### 3.1 新建项目
打开pycharm，新建django项目，项目名为yourwiki，应用名（application）为wiki。得到如下的文件结构：
```
.
├── manage.py
├── wiki
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
└── yourwiki
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```

### 3.2 设置
修改settings.py进行项目的设置。
#### 3.2.1 数据库设置：
本系统采用mysql数据库
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'yourwiki',
        'USER': 'root',
        'PASSWORD': 'xxxxxxx',
        'HOST': '127.0.0.1',
        'PORT': '3306',
    }
}
```
同时在同目录下的init.py文件中添加：
```python
import pymysql
pymysql.install_as_MySQLdb()
```
参考：[python3+django 支持 mysql][2]
#### 3.2.2 静态资源设置
```python
STATIC_URL = '/static/'
```
#### 3.2.3 添加APP
```python
INSTALLED_APPS = [
    ...
    'gunicorn',
    ...
]
```
以下设置在本项目中并不需要，仅供了解拓展
#### 3.2.4 设置公共模板
```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')]
        ,
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

使用app中的模板不用特别设置

### 3.3 用户认证
使用django自带的用户认证系统
参考：[使用Django轻松几步搞定用户认证功能][3]
#### 3.3.1 准备工作
在settings.py中添加必要的中间件和APP
```python
INSTALLED_APPS = [
    # 添加以下app
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.sites', # 这里有个坑
    ...
]

MIDDLEWARE = [
    # 添加以下中间件
    'django.middleware.common.CommonMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    ...
]
```

#### 3.3.2 Django的User类介绍
User类是认证系统的核心。用户对象通常用来代表网站的用户，并支持例如访问控制、注册用户、关联创建者和内容等。在Django认证框架中只有一个用户类，例如超级用户('superusers’)或('staff')用户只不过是相同用户对象设置了不同属性而已。

| 属性名    | 说明    |
| -------- | ------ |
| username | 用户名,必需字段。30个字符或更少，可以包含 _, @, +, . 和 - 字符。 |
|first_name|可选。 30 characters or fewer.|
|last_name|可选。 30 characters or fewer.|
|email|邮箱,可选。 Email address.|
|password|密码,必需。Django不是以明文存储密码的，而是存储哈希值。|
|groups|用户组。Many-to-many relationship to Group|
|user_permissions|用户权限。Many-to-many relationship to Permission|
|is_staff|Boolean。决定用户是否可以访问admin管理界面。默认False。|
|is_active|Boolean。 用户是否活跃,默认True。一般不删除用户，而是将用户的is_active设为False。|
|is_superuser|Boolean。默认False。当设为True时，用户获得全部权限。|
|last_login|上一次的登录时间，为datetime对象，默认为当时的时间。|
| date_joined |用户创建的时间|

本项目中有且仅有一个超级用户，使用命令`python manage.py createsuperuser`创建，所以需要用到的字段只有username和password

参考资料：[Django用户认证系统(一)User对象][4]

#### 3.3.3 扩展User类
由于我们的用户模型有个人简介、积分等额外的字段，所以必须对django的user类进行扩展，扩展方法有两种：新建一个与user类为一对一关系的类，或新建一个继承自user的类。这里我们采用第一种方法。
```python
# model.py
from django.db import models
from django.contrib.auth.models import User
...
class UserProfile(models.Model):
    user = models.OneToOneField(User)
    nickname = models.CharField(max_length=45, null=False, default='')
    description = models.TextField()
    scope = models.IntegerField(default=0, null=False)

    @classmethod
    def create(cls, user, nickname, description):
        return cls(user=user, nickname=nickname, description=description)
...
```

参考资料：[Django中扩展User模型][5]
#### 3.3.4 注册
在view.py中创建一个注册函数
```python
...
def register_action(request):
    if request.method == 'POST':
        email = request.POST['email']
        password = request.POST['password']
        nickname = request.POST['nickname']
        description = ''
        user = User.objects.create_user(username=email, email=email, password=password)
        user.save()
        user_profile = UserProfile.create(user, nickname, description)
        user_profile.save()
        return redirect('/')
    else:
        return redirect('/error')
...
```

并在url.py中分配url（此处略），由于User对象的密码不是明文存储的，所以创建User对象时与通常的Model create不同，需用内置的create_user()方法。

#### 3.3.5 登录与登出
在view.py中创建登录和登出函数
```python
from django.contrib.auth import authenticate, login, logout

...

def login_action(request):
    if request.method == 'POST':
        username = request.POST.get('username', '')
        password = request.POST.get('password', '')
        user = authenticate(username=username, password=password) # 验证用户名和密码
        #
        if user is not None:
            if user.is_active:
                login(request, user) # 登录操作
                return redirect('/')
            else:
                # 不存在该用户
                # do something
        else:
            # 用户名或密码错误
            # do something
    else:
        return redirect('/error')

def logout_action(request):
    logout(request)      # 登出操作
    return redirect('/')

...
```

#### 3.3.6 认证
在每个Web请求中都提供一个 request.user 属性来表示当前用户。如果当前用户未登录，则该属性为AnonymousUser的一个实例，反之，则是一个User实例。

你可以通过is_authenticated()来区分，例如：
```python
if request.user.is_authenticated():
    # Do something for authenticated users.
else:
    # Do something for anonymous users.
```

如果想要禁止未登录的用户进行某些操作，可以使用以下两种方法
1、使用 request.user.is_authenticated()

```python
from django.shortcuts import redirect

def my_view(request):
    if not request.user.is_authenticated():
        return redirect('/login/?next=%s' % request.path)
    # ...
```


或者：
```python
from django.shortcuts import render

def my_view(request):
    if not request.user.is_authenticated():
        return render(request, 'myapp/login_error.html')
    # ...
```

2、使用装饰器login_required

`login_required([redirect_field_name=REDIRECT_FIELD_NAME, login_url=None]`
```python
from django.contrib.auth.decorators import login_required

@login_required
def my_view(request):
    ...
```

如果用户未登录，则重定向到 settings.LOGIN_URL，并将现在的url相对路径构成一个next做key的查询字符对附加到 settings.LOGIN_URL后面去。
在本项目中会被重定向到一个未登录错误页面，于是在settings.py中添加：
```python
...
LOGIN_URL = '/nologin/'
...
```

参考资料：[Django用户认证系统(二)Web请求中的认证][6]

### 3.4 markdown渲染
本项目可以使用markdown语法来创建词条，所以需要渲染markdown文本
在wiki文件夹下建立新文件夹templatetags，然后在templatetags中建立__init__.py，让文件夹可以被看做一个包。然后在文件夹中新建custom_markdown.py文件, 添加代码
```python
import markdown

from django import template
from django.template.defaultfilters import stringfilter
from django.utils.encoding import force_text
from django.utils.safestring import mark_safe

register = template.Library()  #自定义filter时必须加上

@register.filter(is_safe=True)  #注册template filter
@stringfilter  #希望字符串作为参数
def custom_markdown(value):
    return mark_safe(
                markdown.markdown(
                    value,
                    extensions = ['markdown.extensions.fenced_code', 'markdown.extensions.codehilite'],
                    safe_mode=True,
                    enable_attributes=False)
                )
```

然后只需要对需要进行markdown化的地方进行简单的修改,

```html
    {% extends "base.html" %}
    {% load custom_markdown %}
    <!-- 添加自定义的filter -->

    {% block content %}
    <div class="container">
        <p>
            <h1>{{ entry.title }}</h1>
        </p>
        <hr class="intro-divider">
        <p>
            {{ entry.content | custom_markdown }}
            <!-- 使用filter -->
        </p>
        <p><a href="/entry/edit/{{ entry.id }}/">编辑词条</a></p>
    </div>
    <div class="container">
    ...
    {% endblock %}
```

参考资料：[多说,markdown和代码高亮][7]
### 3.5 其他
其他功能均为Django最为基础的应用，在此不再赘述，说一下遇到的问题
1、crsf_token
为了防御csrf攻击，Django框架提供了csrf令牌的机制，在每一个表单中必须插入
2、DoesNotExist at /admin/
在访问管理员界面时报错，原因不明，解决方法为注释掉`INSTALLED_APPS`中的`django.contrib.sites`，参考：[DoesNotExist at /admin/][8]


## 4 部署
### 4.1 拓扑结构
1台负载均衡服务器，2台应用服务器，1台数据库服务器
### 4.2 环境
系统均使用CentOS 6.5 64bit，其他与开发环境一致
### 4.3 准备工作
#### 4.3.1 数据库服务器
安装mysql5.6
#### 4.3.2 应用服务器
**安装python3**
参考http://www.jianshu.com/p/6199b5c26725
**安装mysql依赖**
yum install mysql-devel
**安装python依赖包**
pip3 install Django
pip3 install markdown
pip3 install gunicorn
pip3 install pymysql
**安装nginx**
yum install nginx
#### 4.3.3 负载均衡服务器
**安装nginx**
yum install nginx

### 4 服务器配置
#### 4.1 应用服务器
在两台服务器上进行以下操作：
1、修改settings.py中数据库配置信息

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'yourwiki',
        'USER': 'root',
        'PASSWORD': 'xxxxxxx',
        'HOST': 'xxxxxxxxx', # 改为数据库服务器的ip
        'PORT': '3306',
    }
}
```

2、上传项目代码到 `/home/sites/`

3、配置nginx
进入/etc/nginx/conf.d，修改defualt.conf文件

```
server {
    listen 80;
    server_name 127.0.0.1;

    location /static {
        alias /home/sites/yourwiki/wiki/static;
    }

    location / {
        proxy_pass http://localhost:8000;
    }
}
```

重启nginx

```
service nginx restart
```

4、在其中一台服务器上运行下面两条命令以同步数据库

```
python manage makemigrations
python manage migrate
```

5、运行Gunicorn
进入项目目录，运行：

```
gunicorn yourwiki.wsgi:application
```

#### 4.2 负载均衡服务器

#### 4.3 可能遇到的问题
修改nginx配置文件后，重启报错：
nginx: [emerg] socket() [::]:80 failed (97: Address family not supported by protocol)
解决办法：
修改/etc/nginx/conf.d/default.conf，将

```
listen       80 default_server;
listen       [::]:80 default_server;
```

改为：

```
listen       80;
#listen       [::]:80 default_server;
```

参考：[nginx: [emerg] socket() [::]:80 failed (97: Address family not supported by protocol)][9]

[1]: https://docs.djangoproject.com/en/1.10/
[2]: http://blog.csdn.net/tanzuozhev/article/details/50483167
[3]: http://cache.baiducontent.com/c?m=9f65cb4a8c8507ed4fece7631046893b4c438014609684482e93d25f93130716017bb5e3747e4559c4c50a3052ee024bea867734604161a09bbecf0b8afb852859db60722a4bda0749824ae9901b79dc34d507a9f916b1e2a36ec6f3c5d2af4325cb44747b9780fb4d7311dd6e800340e9b1e83c022914ad9e36&p=8b2a970086cc41af5ba6c6601c0886&newp=8c49cd1985cc43ff57ed947d157a95231610db2151d6d2126b82c825d7331b001c3bbfb423231503d5c27a670aa5425febfa3d78310825a3dda5c91d9fb4c57479&user=baidu&fm=sc&query=django%D3%C3%BB%A7%C8%CF%D6%A4&qid=924042dc00021d6a&p1=2
[4]: http://www.cnblogs.com/linxiyue/p/4060213.html
[5]: http://blog.csdn.net/watsy/article/details/15506351
[6]: http://www.cnblogs.com/linxiyue/p/4060434.html
[7]: http://wiki.jikexueyuan.com/project/django-set-up-blog/markdown.html
[8]: http://blog.sina.com.cn/s/blog_700c93a80100soyp.html
[9]: http://blog.csdn.net/reblue520/article/details/52799500
