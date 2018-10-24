---
title: 令人头大的CRLF注入
url: 200.html
id: 200
categories:
  - php
date: 2018-02-07 13:50:19
tags:
---

由于在重温《白帽子讲web安全》这本书，在注入攻击一章发现了这个新奇的crlf注入攻击，甚至可以改变http的报文，并将http头的部分当成httpbody输出。先大致解释下原理把。 就是在可以控制http的一些地方，插入%0d%0a，转义后就是\\r\\n，也就是换行符，从而破坏http报文。 例如在set-cookie后插入1%0d%0a%0d%0a<scritp>alert(/xss/)</script>，在返回的报文中就可以加j将set-cookie之后的http头当作body部分输出，甚至可以在这之后加上xss，造成反射性xss攻击
```
HTTP/1.1 200 OK
Date: Wed, 07 Feb 2018 05:41:05
GMT Server: Apache/2.4.9 (Win64) PHP/5.5.12
X-Powered-By: PHP/5.5.12
Set-Cookie: name=1 

<script>alert(/xss/)</script>
Content-Length: 5
Connection: close
Content-Type: text/html
```
当时想着，这玩意也太牛逼了把，现在本地测试一下。 然而和我想象的根本不一样。。测试代码如下

<?php
 setcookie('name',$_GET\['name'\]);
?>

然而![](http://blog.kingkk.com/wp-content/uploads/2018/02/dc9b3bdb7f43e8d023036069a304951d.png)。。 根本就是原样输出。。。 然后想着是不是函数的问题，于是再测试了下header函数

<?php
 header('Location: '.$_GET\['name'\]);
?>

抛出了个warning，说只能添加一行 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/f29b3f149c8dafd203cde4675232eafd.png) 后来也试着换过nginx，换过各种浏览器，也还是这个样子。 今天仔细看了下php的官方手册，貌似php官方对函数进行了安全过滤 如下是header的官方文档 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/K3_@1SQ9NANAT8YDN12.png) 官方都出手了，这还能怎么办。。。