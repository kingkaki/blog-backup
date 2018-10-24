---
title: RPO攻击
url: 262.html
id: 262
categories:
  - CTF
  - javascript
date: 2018-03-27 21:49:31
tags:
---

# 前言

这次强网杯真的见识到了大佬们的可怕，知识面和深度的双重不足，是得抓紧时间赶紧把基础给补上，以后才能有提升的机会。 
分析了以下sharemind这道题，又见识了一种新的攻击方式——RPO攻击  

# 正文

RPO攻击主要是利用服务器和浏览器，也就是服务端和客户端对url的解析不同，造成的漏洞攻击。 

测试环境

*   Chrome浏览器
*   nginx websever

* * *

先看一个简单的例子 文件结构如下 
```
WWW 
|——test 
|    |——test.php 
|    |——style.css(bule) 
| 
|——index.php 
|——style.css(red) 
```
index.php引用相对路径，引入WWW目录下的style.css(red)，对字符进行渲染
```html
<link rel="stylesheet" href="**style.css**" style="css" />
<h1>Index!</h1>
```
对index.php进行访问，其渲染的是红色字体 
![](http://blog.kingkk.com/wp-content/uploads/2018/03/e52b8f8e492adc14774f29df2aebe17a.png) 
当我们访问如下url时，会出现一个奇怪的现象
```
http://localhost/test/test.php%2f..%2f..%2findex.php
```
![](http://blog.kingkk.com/wp-content/uploads/2018/03/a3578f4116e0b0b7531b18509c5fab42.png) 
请求的页面是index.php的页面，但是加载的css却是test页面下的css 
这是因为 对服务器而言，会对%2f进行解析，转义成/，所以之前的url转义后就变成了
```
http://localhost/test/test.php/../../index.php
```
等价于
```
http://localhost/index.php
```
然而对于浏览器而言，在对页面进行渲染的时候，对于路径的解析并不会转义%2f，它是将`test.php%2f..%2f..%2findex.php`当作一个文件名，目录路径还是以`localhost/test/`当作目录路径(貌似apache需要指定的版本才能解析，本地的apache不会转义%2f) 
![](http://blog.kingkk.com/wp-content/uploads/2018/03/80b948bea3c962f02d185889cba10858.png) 
这一点也能从浏览器的查看源码中看出 **从而产生了浏览器和服务器对路径的理解不同** 所以，当加载的不是一个css，而是js的时候，能找到一个可以控制的文本，然后指定对应的地址，就能够产生xss攻击  

* * *

接下来看下强网杯的那道share mind 漏洞发生在index.php页面上 !
[](http://blog.kingkk.com/wp-content/uploads/2018/03/e07bf2bc14b156691dd9c83523936fe8.png) 
这里单独使用了相对路径，也表名了出题的意图 这里我们可以控制一个留言界面,为了保证字符的纯净（因为js遇到无法解析的语句时就不会继续执行），所以我们这里不输入标题，只输入文章内容 
![](http://blog.kingkk.com/wp-content/uploads/2018/03/2f8dcfb2fad149a6a17dc2441788c314.png) 
可以获得一个没有坏字符的纯文档 
![](http://blog.kingkk.com/wp-content/uploads/2018/03/477c4b5b435be14b3ab677f33fa58771.png) 
此时访问如下页面
```
http://39.107.33.96:20000/index.php/view/article/1571/..%2f..%2f..%2f
```
![](http://blog.kingkk.com/wp-content/uploads/2018/03/2059e24133261e229bad5a2461fe4895.png) 
成功弹出xss 服务器获取的页面即为
```
http://39.107.33.96:20000/index.php
```
此时该index.php引用了相对路径下的 `static/js/jquery.min.js`  
此时被解释为了`http://39.107.33.96:20000/index.php/view/article/1571/static/js/jquery.min.js` 
![](http://blog.kingkk.com/wp-content/uploads/2018/03/36da43e70ae09b450cbcd9c96dcf04c3.png) 
然而真正访问`http://39.107.33.96:20000/index.php/view/article/1571/static/js/jquery.min.js`路径时，解析为`http://39.107.33.96:20000/index.php/view/article/1571` 
![](http://blog.kingkk.com/wp-content/uploads/2018/03/b7392d4b2e1ab4e6ff513715612216e1.png) 
即我们之前插入的文章内容，因为`http://39.107.33.96:20000/index.php/view/article/1571`之后的字符串，以一种伪静态传参的方式被解析成参数了，所以访问的文件即是我们之前写好的恶意html，被当作js文件解析了。从而引发xss攻击

* * *

后面的问题就是绕过一些转义的实体编码，打xssbot的cookie，获取falg。没学过js，目前不是很了解。总之主要的思路就是利用了这个RPO攻击。（逃……）