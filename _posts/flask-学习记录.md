---
title: flask-学习记录
date: 2018-06-21 16:35:53
tags: python flask
---

# 前言
等待漏洞审核和国赛名单真是个难熬的事情，目前已经等到麻木了  
期间就决定学习一下flask,了解一下python web  
之前学过一些Django，但是那时候还懵懵懂懂，而且Django封装了太多东西，那时候也就只能照着教程来依葫芦画瓢  
然后这回打算从更加轻量级的flask入手，来学习python web  
  
# 正文
不知道写啥就写博客系统呗，于是就花了两个星期左右时间写了一个博客系统  
还是先上下项目的[github地址](https://github.com/kingkaki/Flask-blog)吧  

主要的功能在github里已经说清楚了，写博客主要是为了理清flask的运行逻辑，以及一些相互关系  
  
## 目录结构
先上一下整个项目的目录结构
```
├─app
│  │  models.py
│  │  __init__.py
│  │
│  ├─admin
│  │  │  forms.py
│  │  │  views.py
│  │  │  __init__.py
│  │
│  ├─blog
│  │  │  forms.py
│  │  │  views.py
│  │  │  __init__.py
│  │
│  ├─member
│  │  │  forms.py
│  │  │  views.py
│  │  │  __init__.py
│  │
│  ├─static
│  │  ├─css
│  │  ├─fonts
│  │  ├─img
│  │  │  │
│  │  │  └─avatars
│  │  └─js
│  │
│  ├─templates
│     │  404.html
│     │  base.html
│     │
│     ├─admin
│     │      admin.html
│     │      adminlog.html
│     │      article.html
│     │      base.html
│     │      index.html
│     │      login.html
│     │      password.html
│     │      user.html
│     │
│     ├─blog
│     │      article.html
│     │      base.html
│     │      index.html
│     │
│     └─member
│             articles.html
│             base.html
│             create.html
│             index.html
│             login.html
│             member.html
│             modify.html
│             password.html
│             register.html
│             update.html
│             userlog.html
└─manage.py
```

## app __init__.py
重点在app的__init__.py中  
不妨先来看下
```python
from flask import Flask, render_template
from flask_sqlalchemy import SQLAlchemy
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = "mysql://root:@127.0.0.1/flask-blog"
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
app.config['SECRET_KEY'] = 'cc1df2e4-fd25-4a33-a7ae-9555f5e361b0'
app.config['UPLOAD_FOLDER'] = os.path.join(os.path.dirname(__file__),'static','img')
app.debug = True
db = SQLAlchemy(app)

from app.member import member
from app.admin import admin
from app.blog import blog

app.register_blueprint(member, url_prefix='/member')
app.register_blueprint(admin, url_prefix='/admin')
app.register_blueprint(blog, url_prefix='/')


@app.errorhandler(404)
def page_not_found(error):
	return render_template('404.html'), 404
```
一开始是导入一些必要的库，然后用`Flask(__name__)`生成一个flask app  
之后就是对app的一些配置信息  
  
`db = SQLAlchemy(app)`将数据库和app绑定  
  
`app.register_blueprint(member, url_prefix='/member')`  
蓝图注册，将不同模块注册到不同的蓝图下（也就是所谓的目录），然后配置目录的前缀  
如这里memebr模块下的url都是在`/memeber`下的  
  
然后定义了一个app的404页面


## models.py
models.py中主要是定义了数据类型  
  
取一小部分看下
```python
from . import db

class Userlog(db.Model):
	__tablename__ = 'userlog'
	id = db.Column(db.Integer, primary_key = True)
	user_id = db.Column(db.Integer, db.ForeignKey('member.id'))
	ip = db.Column(db.String(50)) 
	addtime = db.Column(db.DateTime, index=True, default=datetime.now)

	def __repr__(self):
		return "<Userlog %r>" % self.id
```
导入了__init__.py中的db变量  
继承了db.Model类，定义了一个数据类，对应了mysql中的一张表  
然后`__tablename__`也就是表名，然后比变量对应着字段  
  
可以重写一些函数，或者添加一写数据处理的函数  

models.py也就对应着mvc中的整个Model数据层
  
  
## modules __init__.py
然后接下来来看下模块中的`__init__.py`  
由于三个模块都类似，取blog的看一下  
```python
from flask import Blueprint

blog = Blueprint('blog', __name__)

import app.blog.views
```
内容很简单，就三行  
主要是调用了Blueprint蓝图方法，创建了一个blog变量，供__init___.py中注册全局蓝图，以及views.py中注册子url  
然后调用views.py中的内容
  

## views.py
作为mvc中的Controller（其实我也不知道为什么要叫成views）是后台代码量的主要部分，处理着主要业务逻辑  
  

取一小部分看一下
```python
from . import blog
from flask import render_template, redirect, url_for, session, flash, request
from app.models import *

@blog.route('/')
def index():
	articles = Article.query.all()
	return render_template('blog/index.html', articles=articles)
```
从__init__.py中导入blog蓝图变量，用于注册url`@blog.route('/')`  
  
然后从flask中导入一些常用的变量
- render_template 用于返回视图
- redirect 进行重定向
- url_for 动态构建url
- session 就是服务器cookie
- flash 消息闪现
- request 请求变量，存放一些url，get、post参数之类
  
然后从models.py中导入全部数据类型

接下来就是注册url为根url  
然后从Article数据类型中筛选出全部字段  
然后返回templates目录下blog/index.html，并注册articles变量  
  
## 视图层
flask视图层自带着jinja2
语法也是特别好用  
  
常用语法
```
- {% extends 'base.html' %} 页面继承
- {% if  %} {% elif %} {% else %} {% endif %}  if判断语句
- {% for i in xxx %} {% endfor %} for循环
- {{ variable }} 变量输出
- {{ variable|safe }} 变量输出(不html编码)
```
  
然后在views.py中引入了的函数名也可以直接引用
  

## forms.py
涉及到和表单打交道时就免不了数据校验  
flask中单独把数据校验这块拿了拿出（django中似乎也是）  

forms.py中会继承wrforms中的Form，新建一个自定义表单类  
```python
class LoginForm(Form):
	username = TextField('username', [validators.Required()])
	password = PasswordField('password', [validators.Required()])

	def validate_username(self, field):
		user = Admin.query.filter_by(username=field.data).count()
		if user == 0:
			raise ValidationError("username does not exist.")
```
或者可以添加自定义校验函数，实现更复杂的数据校验  
  
在views.py中会对post或者get的数据传入这个LoginForm中`form = LoginForm(request.form)`  
然后调用form的validate()方法，通过数校验时返回True，否则返回False  
失败时也可以通过`form.errors`获取具体的错误原因  
  
## bootstrap
这个本来属于前端的东西，我这里要说一下的原因就在于，我也是通过写这个小系统才熟悉的bootstrp  
  
bootstrap只需要在html开头导入几个文件，即可利用class标签，生成对应的css和js样式，极其方便
  
整个html主要是以div标签为切块，然后利用col-md-x进行栅格排列，还有col-md-offset-x进行位移  
使得整个页面的布局变得简单，然后还有一些`container` `btn` `danger` `info`之类的标签，使得央视的更改也变得简易  
  
更多的详情见官方手册好了，毕竟那么多标签我也记不住（逃。。。  
  
## 启动
启动app的工作就在manage.py中，其实也很简单  
```python
from app import app

if __name__ == '__main__':
	app.run()
```
只要运行app变量的run方法即可，实际上线时记得关闭debug模式即可

