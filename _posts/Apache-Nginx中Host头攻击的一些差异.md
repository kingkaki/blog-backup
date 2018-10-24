---
title: Apache/Nginx中Host头攻击的一些差异
tags:
  - Host
date: 2018-10-12 18:14:22
---


# Host header

> 服务器的域名(用于虚拟主机 )，以及服务器所监听的传输控制协议端口号。如果所请求的端口是对应的服务的标准端口，则端口号可被省略。
>
> 自超文件传输协议版本1.1（HTTP/1.1）开始便是必需字段。

以上是维基百科中对于Host头部的说明。可以看到Host头部并非是用于区别发送到哪台主机的字段。而是用于区分一台主机上不同的虚拟主机。（可以在Apache和Nginx中配置相应Host对应的虚拟主机。

利用PHP可以在`$_SERVER['HTTP_HOST']`字段中获取到该字段的值

```php
var_dump($_SERVER['HTTP_HOST']);
```

![](Apache-Nginx中Host头攻击的一些差异\1.png)

当没有配置虚拟主机或者匹配不到对应的`Host`主机时，webserver就会发送请求到默认的目录中去解析

从而`Host`头部就变成了一个可控的字段，当一些应用程序没有对`$SERVER['HTTP_HOST']`字段进行处理，过分信任时，就会产生一些安全问题

![](Apache-Nginx中Host头攻击的一些差异\2.png)

  

# Diff of win/linux

这里用的测试环境分别是

 - win10下的 phpstudy
 - ubuntu 18.04中默认安装的nginx和apache2

在1.php中会输出`$_SERVER['HTTP_HOST']`变量

## Windows

### Apache

可以任意修改apache中的值，服务器都会默认接受

![](Apache-Nginx中Host头攻击的一些差异\10.png)

甚至可以插入两个`Host`头部，在win下会用`,`将两个头部相连

```
Host: localhost
Host:123468
```

![](Apache-Nginx中Host头攻击的一些差异\11.png)

### Nginx

nginx中对于任意脏字符都是允许修改的

![](Apache-Nginx中Host头攻击的一些差异\12.png)

但是在插入两个`Host`头部的时候只会获取到第二个`Host`值

```
Host: localhost
Host: 'select user();
```

![](Apache-Nginx中Host头攻击的一些差异\13.png)

## Linux

但是在linux中又会有些不同

### Apache

linux下的apache就有着较为严格的限制，只允许修改对应的ip数字

![](Apache-Nginx中Host头攻击的一些差异\15.png)

插入一些脏字符就会返回400的错误

```
Host: 192.168.85.145'
```

![](Apache-Nginx中Host头攻击的一些差异\16.png)

两个头部也是一样的情况，即使头部是合法的

```
Host: 192.168.85.145
Host: 192.168.85.145
```

![](Apache-Nginx中Host头攻击的一些差异\17.png)

### Nginx

linux下nginx中对于`Host`的处理类似于win中的就不过多介绍

![](Apache-Nginx中Host头攻击的一些差异\14.png)

  

从而网上有一些防御手段就变成了设置虚拟主机（virtual host），从而可以确保`Host`字段不会被更改，因为一旦修改了`Host`字段就不会解析到对应的目录当中。

# Virtual Host

但即使配置了虚拟主机之后，所接受的`Host`字段就是可以信任的么？

在`Apache`、`Nginx`中会呈现不同的状态

测试环境为

 - ubuntu 18.04
 - Apache/2.4.29 (Ubuntu)
 - nginx/1.14.0 (Ubuntu)
 - PHP 7.2.10-0ubuntu0.18.04.1

至于虚拟主机配置的部分就不过多介绍，在网上可以找到很多配置的文章

至于域名，可以买一个域名修改解析，或者直接修改host文件

然后在`/var/www`目录下新建了三个文件夹

 - `html` 默认主目录
 - `apache` Apache的虚拟主机目录
 - `nginx` Nginx的虚拟主机目录

## Apache

可以看到，在配置了虚拟主机之后，访问对应的域名就会访问到对应的目录中

```
Host: localhost.ba123.top
```

![](Apache-Nginx中Host头攻击的一些差异\3.png)

当遇到一个不认识的域名的时候，就会解析到默认目录下（当然，假如默认目录下没有`1.php`这个文件就会返回一个404

```
Host: localhost.ba123.to
```

![](Apache-Nginx中Host头攻击的一些差异\4.png)

那想攻击Apache中的虚拟主机时`Host`字段就不能添加别的脏字符了么？目前找到的可以添加的就只有通过冒号`:`分割开的端口号

只要是一个合法的端口就可以发起正常的请求

```
Host: localhost.ba123.top:23333
```

![](Apache-Nginx中Host头攻击的一些差异\5.png)

但是当想插入一些字符或者端口号过大的时候，都会拒绝请求，返回一个400，更不会允许两个`Host`头部的请求

```
Host: localhost.ba123.top:23333'select
```

![](Apache-Nginx中Host头攻击的一些差异\6.png)

总体来说，Apache对于`Host`字段的限制还是比较严格的，在开启了虚拟主机之后，几乎比较难以插入脏字符。

## Nginx

在nginx中似乎有着更多的攻击手法

对于未知的主机名，处理方式还是与apache中一致，会解析到默认目录中

```
Host: localhost.ba123.to'select
```

![](Apache-Nginx中Host头攻击的一些差异\7.png)

但是这里假如利用冒号`:`的形式分割之后，冒号后面的`port`部分会直接被nginx给抛弃，从而可以插入任意的字符

```
Host: localhost.ba123.top'select sleep(5);
```

![](Apache-Nginx中Host头攻击的一些差异\8.png)

甚至在nginx中可以传入两个`Host`头部，Nginx将以第一个为准传送到对应的虚拟主机处理，而PHP-FPM将以第二个为准给`$_SERVER['HTTP_HOST']`赋值。

```
Host: localhost.ba123.top
Host:'select sleep(5);
```

![](Apache-Nginx中Host头攻击的一些差异\9.png)

但是在apache中传入两个Host头部的时候，就会直接返回400

由此可见，在ngin中对于`Host`的并没有做过多的限制，从而可以比较容易的进行Host头部攻击。

  

windows上的虚拟主机原谅我确实没怎么遇到过，就没测试了，感兴趣的可以自己测试下。



# Summary

可以看到`Host`头部，在不同的情况下会呈现出不同的状态，因此在编写程序的时候一定要将`Host`这个值设置为不可信任的，对其进行一些正则或者转义的一些过滤。否则极易引发安全问题。

在windows下对于这种头部的限制比较松，apache和nginx都能比较轻松的绕过去。

而在两个服务器相比，Apache对于数据的解析更加严谨，尤其是在linux的环境中，极大的限制了非法的请求。

Nginx则甚至会出现`Nginx`与`PHP-FPM`解析不一致的情况，有时这种解析不一样甚至会带来更多的危害。

# Reference

https://www.leavesongs.com/PENETRATION/some-tricks-of-attacking-lnmp-web-application.html

