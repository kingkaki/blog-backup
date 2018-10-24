---
title: php5.6 安装
url: 167.html
id: 167
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-20 20:52:55
tags:
---

一开始想试着装下php7.0的，后来编译安装搞得有点头大，转向5.6了，直接开始吧 由于yum中自带的时php5.3，要重新配置下yum源

### 配置yum源

**追加CentOS 6.5的epel及remi源。**

\# rpm -Uvh http://ftp.iij.ad.jp/pub/linux/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm
\# rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm

**以下是CentOS 7.0的源。**

\# yum install epel-release
\# rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm

**使用yum list命令查看可安装的包(Packege)。**

\# yum list --enablerepo=remi --enablerepo=remi-php56 | grep php

 

### 安装php

**yum源配置好了，下一步就安装PHP5.6。**

\# yum install --enablerepo=remi --enablerepo=remi-php56 php php-opcache php-devel php-mbstring php-mcrypt php-mysqlnd php-phpunit-PHPUnit php-pecl-xdebug php-pecl-xhprof

**用PHP命令查看版本。**

\# php --version
PHP 5.6.0 (cli) (built: Sep 3 2014 19:51:31)
Copyright (c) 1997-2014 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2014 Zend Technologies
 with Zend OPcache v7.0.4-dev, Copyright (c) 1999-2014, by Zend Technologies
 with Xdebug v2.2.5, Copyright (c) 2002-2014, by Derick Rethans

若成功就说明安装完成了 **安装PHP-fpm(增加php并发量的，可选)**

yum install --enablerepo=remi --enablerepo=remi-php56 php-fpm

**启动php-fpm**

service php-fpm start

 

### 检测

在搭建好apahce或者nginx的情况下 在网站目录下新建个phpinfo.php，写入如下

<?php
phpinfo();
?>

访问，出现如下就说明可以正常解析php语言了![](http://blog.kingkk.com/wp-content/uploads/2018/01/5938f0a38454b7f53fb8b8e5ce1cf5ba.png)    

* * *

来源于网上…… 有一些教程貌似行不通，我也不知道是什么原因，至少如上教程在我电脑中可以成功运行