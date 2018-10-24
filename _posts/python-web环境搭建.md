---
title: Python web环境搭建
url: 177.html
id: 177
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-21 11:04:27
tags:
---

### 安装pip

**centos6.x版本的话**

yum install python-pip.noarch

**centos7.x版本**

yum install python2-pip

### 安装virtualenv

用于独立每个python环境

pip install virtualenv

创建virtualenv

virtualenv pythonweb

切换virtualenv

\[root@localhost ~\]# source ~/pythonweb/bin/activate
(pythonweb) \[root@localhost ~\]#

推出virtualenv

deactivate

 

### 搭建flask

新建一个目录和文件

mkdir pythontest
vim ~/pythontest/index.py

写入如下helloworld程序

from flask import Flask
app=Flask(\_\_name\_\_)

@app.route('/')
def hello_world():
 return 'hello world!'

if \_\_name\_\_=='\_\_main\_\_':
 app.run(host='0.0.0.0')

通过主机浏览一下 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/409272fdca611e822875f4f22b755391.png)  

### 反向代理pythonweb

可以参考javaweb搭建中的反向代理，这里就暂时不复述