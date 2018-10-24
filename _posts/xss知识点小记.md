---
title: xss知识点小记
date: 2018-08-23 15:47:42
tags:
	- xss
---

# 前言

最近在学习xss，记录一些比较重要的知识点。有时能从根本上解决一些问题的疑惑

# 同源策略

同源策略可以说是浏览器安全中最最基础也最为重要的部分了。同源策略限制了资源的任意加载，限制恶意请求。



## 何为同源

这个估计大家都比熟悉，简单的来说就是如下三点

- 协议相同（http/https）
- 端口相同
- host相同

![](xss知识点小记\1.png)



## 请求过程

跨域请求在html中的不同位置都会有发生，主要分为如下三类

### Cross-origin embedding 

嵌入资源，比如一些图片、视频、字体、css、js资源等

这种嵌入式的资源是可以跨域访问的

### Cross-origin write

例如form表单的提交，以及一些link的重定向

这种情况下是不会受到同源策略的影响，表单数据，以及链接地址也是可以发往任何域的

### Cross-origin read

例如利用ajax发送http请求，获取页面数据

这种情况下是严格受到同源策略的限制

> 需要强调的一点是，跨站请求的失败，并非是请求没有发起，而是请求已经发送到服务器，服务器也已经返回数据。只是在返回时被浏览器给拦截了

## CORS

为了获取一些非同域的数据，于是有了`CORS`策略，来允许跨域资源的访问，对跨域的资源进行了一些限制

主要是通过一系列`Access-Control-xxxx`格式的http头来完成这一项任务

### simple request

`simple request`主要是依据`Access-Control-Allow-Origin` 头来判别允许加载的源

例如当发起这样一段ajax请求时

```
xmlhttp=new XMLHttpRequest();
xmlhttp.onreadystatechange=function()
{
	if (xmlhttp.readyState==4 && xmlhttp.status==200)
{
	document.body.append(xmlhttp.response);
}
}
xmlhttp.open("GET","http://192.168.85.128");
xmlhttp.send();
```

由于对方服务器未设置相应的CORS头部，导致被同源策略阻拦，无法获取对应的数据

![](xss知识点小记\2.png)



在远程服务器上利用php发送一段`Access-Control-Allow-Origin`头，让所有源的请求可以获得该响应数据

```
header('Access-Control-Allow-Origin:*');
```

当返回的响应带上这个头之后，浏览器就可获取到返回的信息

![](xss知识点小记\3.png)



### preflight request

不符合以下要求的，会触发一个`preflight`机制（搬运。。

- GET 请求
-  HEAD 请求
-  Content-Type 为指定值的 POST 请求，包括`text/plain`，`multipart/form-data`以及`application/x-www-form-urlencode`
- HTTP 首部字段不能包含下列以外的值： 
  - Accept
  - Accept-Language
  - Content-Language
  - Content-Type
  - DPR
  - Downlink
  - Save-Data
  - Viewport-Width
  - Width

该机制会在请求前发送一个`OPTIONS`检查，询问跨域相关信息

```
OPTIONS /resources/post-here/ HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Origin: http://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-TEST, Content-Type


HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-TEST, Content-Type
Access-Control-Max-Age: 86400
Vary: Accept-Encoding, Origin
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

请求方发送了两个CORS头部

- Access-Control-Request-Method: POST   询问是否可以使用POST传输
- Access-Control-Request-Headers: X-TEST, Content-Type   询问是否允许自定义头部

服务器也响应了四个CORS头部

- Access-Control-Allow-Origin: http://foo.example  允许该源请求数据
- Access-Control-Allow-Methods: POST, GET, OPTIONS  允许的请求方式
- Access-Control-Allow-Headers: X-TEST, Content-Type   允许的自定义头部
- Access-Control-Max-Age: 86400  有效时间





# 浏览器解码

主要是为了解决为何有时候可以使用html绕过，以及一些标签优先级问题

## HTML五类元素

### 空元素(Void elements)

如`<area>`,`<br>`,`<base>`等等

不能容纳任何内容，因为不存在闭合标签，从而没有内容可以容纳至其中

### 原始文本元素(Raw text elements)

有`<script>`和`<style>` 可以容纳文本

### RCDATA元素(RCDATA elements)

有`<textarea>`和`<title>` 可以容纳文本和字符引用

### 外部元素(Foreign elements)

例如MathML命名空间或者SVG命名空间的元素

可以容纳文本、字符引用、CDATA段、其他元素和注释

### 基本元素(Normal elements)

即除了以上4种元素以外的元素，可以容纳文本、字符引用、其他元素和注释

## 解析顺序

浏览器的解码顺序为

```
html解码 ==> javascript解码 ==> url解码
```

正是由于这个原因，导致在`script`标签中，只要遇到`</script>`标签就会进行闭合



在加载完html代码之后，浏览器会先对整个html代码进行一次解析，在解析中会进入几个不同的状态

- `<`字符前 ： Data state
- `<`字符时 ：Tag open state
- 找到标签名： Tag name state
- 属性名 ：before attribute name state
- 属性值 ： Data state
- `>`字符时 ： 重新进入Date state



每遇到一个新的标签就会记录一个token，当一个标签完结的时候，就会释放掉那个token

在RCDATA元素标签中，会进入一种`RCDATA`状态，只认得`</textarea>`、`<title>`
从而在“`<textarea>`”和“`<title>`”的内容中不会创建标签，就不会有脚本能够执行。

`svg`外部标签，在解析它的时候，由于其支持xml协议，从而，`svg`还会对其内部数据进行一次xml解码，也就导致了`svg`标签内部可以使用html编码

因此HTML编码只有在如下几种情况下可以使用

- Data state

  - 在标签外部

  - 在属性值时
- svg标签内部



# CSP（Content Security Policy）

## 简介

> Content Security Policy （CSP）内容安全策略，是一个附加的安全层，有助于检测并缓解某些类型的攻击，包括跨站脚本（XSS）和数据注入攻击。
>
> CSP的特点就是他是在浏览器层面做的防护，是和同源策略同一级别，除非浏览器本身出现漏洞，否则不可能从机制上绕过。
>
> CSP只允许被认可的JS块、JS文件、CSS等解析，只允许向指定的域发起请求。

CSP不仅限制了js的引入，还限制了个各种静态文件资源的的引入。在默认情况下甚至限制了内联js的执行（就是在`<script>`标签中的js代码

CSP的从一定程度上**缓解**了XSS的危害，但由于其较强的限制性，网站要做较为详细的配置，若是配置不当，依旧容易引起XSS



启用

有两种启用的方式

1、 发送一个`CSP`的http头

```php
header("Content-Security-Policy: default-src 'self'; script-src 'self' https://www.kingkk.com;");
```

2、在meta标签种添加

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' https://www.kingkk.com;">
```

## 配置

可以自定义限制不同资源的加载

![](xss知识点小记\5.png)

加载策略（注意一些有引号和一些没引号的区别）

![](xss知识点小记\4.png)

例如之前的那个范例种的CSP策略

```php
header("Content-Security-Policy: default-src 'self'; script-src 'self' https://www.kingkk.com;");
```

就意为`script`标签只允许加载本站，以及`https://www.kingkk.com`上的资源，其余的`css`、`img`之类的资源只允许加载本站的



例如服务上的一端代码

```php
<?php header("Content-Security-Policy: default-src 'self' "); ?>
<img src="https://www.kingkk.com/2018/08/xss%E7%9F%A5%E8%AF%86%E7%82%B9%E5%B0%8F%E8%AE%B0/1.png">
```

访问时够看到chrome中的报错

![](xss知识点小记\6.png)

需要添加指定CSP源才能添加远程服务器的资源

```php
header("Content-Security-Policy: default-src 'self'; script-src 'self'; img-src 'self' https://www.kingkk.com ")
```

才能正确加载出资源

![](xss知识点小记\7.png)





## nonce script

由于默认是会限制内联脚本的执行，然而实际当中内联脚本又是必不可少的，于是推出了一种`nonce`属性

```php
header("Content-Security-Policy: default-src 'self'; script-src 'nonce-{random-str}' ");
```

只有当`script`标签中`nonce`属性值和CSP中的`nonce`属性值一样才能执行内联脚本

```
<script nonce="{random-str}">alert(1)</script>
```

这个随机数值需要由后端随机生成。从而减少内联script的xss危害



例如一段这样的代码

```php+HTML
<?php 
header("Content-Security-Policy: default-src 'self'; script-src 'self'");
?>
<script>
	alert(/xss/);
</script>
```

可以看到默认是进制这种内联方式的代码的

![](xss知识点小记\8.png)

一种解决方式就是在`script-src`后面添加`'unsafe-inline'`

还有就是利用`nonce`标签，只有值相同的script标签才能执行

```php+HTML
<?php 
function random_string( $length = 8 ) { 
  $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'; 
  $password = ''; 
  for($i = 0; $i < $length; $i++)
  { 
    $password .= $chars[ mt_rand(0, strlen($chars) - 1) ]; 
  } 
  return $password; 
} 

$random = random_string(12);

header("Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-$random' ");
?>
<script nonce=<?php echo $random; ?>>
	alert(/xss/);
</script>
```

这样就可以成功弹窗

但当这个两个值不匹配时

```html
<script nonce=13456>
	alert(/xss/);
</script>
```

就可以看到控制台的报错信息

![](xss知识点小记\10.png)



由于这个值每次是随机产生的，构造的难度极大提升，从而能较为有效的抵御xss








# Reference Link

http://pupiles.com/xss.html

https://lightless.me/archives/review-SOP.html

https://www.hackersb.cn/hacker/85.html

https://paper.seebug.org/423/#0x02-cspcontent-security-policy

http://www.ruanyifeng.com/blog/2016/09/csp.html