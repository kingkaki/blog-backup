---
title: 关于Apache的一些安全配置
url: 254.html
id: 254
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-03-11 20:28:37
tags:
---

### 前言：

开学也一星期了，也有十多天没更新博客了。开学这两天忙也是有点忙，但还是抽空记录下比较好。

##### 环境：

*   centos  6/7差别不是很大
*   httpd-2.2.34

 

### 正文：

下载源码包

wget http://apache.website-solution.net/httpd/httpd-2.2.34.tar.gz
tar xf httpd-2.2.34.tar.gz && cd httpd-2.2.34

修改apache版本信息（伪装成iis）

vim include/ap_release.h

#define AP\_SERVER\_BASEVENDOR "IIS Software Foundation"
#define AP\_SERVER\_BASEPROJECT "IIS HTTP Server"
#define AP\_SERVER\_BASEPRODUCT "Microsoft-IIS/7.5"
#define AP\_SERVER\_MAJORVERSION_NUMBER 7
#define AP\_SERVER\_MINORVERSION_NUMBER 5
#define AP\_SERVER\_PATCHLEVEL_NUMBER 0
#define AP\_SERVER\_DEVBUILD_BOOLEAN 0

修改服务器信息（否则linux装iis就很假）

vi os/unix/os.h

#define PLATFORM "Win32"

编译安装

\# yum install zlib zlib-devel gcc-c++ -y  一些依赖库
\# ./configure --prefix=/usr/local/httpd-2.2.34 \
--enable-deflate \
--enable-expires \
--enable-headers \
--enable-modules=most \
--enable-so \
--with-mpm=worker \
--enable-rewrite

\# make && make install

创建伪用户，以httpd伪用户登录

# useradd -s /sbin/nologin -M httpd

修改配置文件，自定义网站目录

\# vim /usr/local/httpd-2.2.34/conf/httpd.conf

ServerRoot "/usr/local/httpd-2.2.34"  # 定义apache的安装目录
Listen 80       # apache默认监听的web端口,没有指定ip的情况下,默认是监听在0.0.0.0
Listen 8080     # 在后面配置基于端口的虚拟主机时需要先在此定义好监听端口
Listen 8888
<IfModule !mpm\_netware\_module>
<IfModule !mpm\_winnt\_module>
User httpd   	# apache的服务用户
Group httpd  	# apache的服务用户组
</IfModule>
</IfModule>
ServerAdmin admin@kingkk.com   # 管理员邮箱地址
ServerName 127.0.0.1:80	     # 解决FQDN的问题
<Directory />		       # 对系统根目录的权限设置
    Options FollowSymLinks
    AllowOverride None
    Order deny,allow	     # 禁止用户直接访问系统根目录
    Deny from all			
</Directory>
<IfModule dir_module>
    DirectoryIndex index.html index.php  # 设置主页索引文件
</IfModule>
<FilesMatch "^\\.ht">	# 当匹配到以.ht开头的文件apache要做什么样的反应
    Order allow,deny
    Deny from all
    Satisfy All
</FilesMatch>
ErrorLog "logs/error_log" 	# apache自身的错误日志存放位置 
LogLevel warn			# 定义apache日志的报告级别,默认是警告
<IfModule log\_config\_module>	# 定义访问日志格式
    # 在下面combined格式日志中还要多记录一个字段\`X-Forwarded-For\`,用于记录客户端真实ip
    LogFormat "%h %l %u %t \\"%r\\" %>s %b \\"%{Referer}i\\" \\"%{User-Agent}i\\" \\"%{X-Forwarded-For}i\\"" combined 
    LogFormat "%h %l %u %t \\"%r\\" %>s %b" common
    <IfModule logio_module>
      LogFormat "%h %l %u %t \\"%r\\" %>s %b \\"%{Referer}i\\" \\"%{User-Agent}i\\" %I %O" combinedio
    </IfModule>
    CustomLog "logs/access_log" common
</IfModule>
DefaultType text/plain
<IfModule headers_module>
    RequestHeader unset Proxy early
</IfModule>
<IfModule mime_module>	# mime类型设定,也就是说,遇到什么类型的文件就做什么样的操作
    TypesConfig conf/mime.types
    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz	
    # 意思就是说,当apache遇到.php这样的后缀就丢给libphp5模块去处理,也就是说,如果这里是.jpg,它依然会被当做php去解析,如,图片马...
    AddType application/x-httpd-php .php 
</IfModule>
<IfModule ssl_module>
SSLRandomSeed startup builtin
SSLRandomSeed connect builtin
</IfModule>
Include conf/extra/httpd-mpm.conf  # 包含apache扩展配置文件
Include conf/extra/httpd-vhosts.conf	  
Include conf/extra/httpd-default.conf
<Directory "/var/html/80">
    # 我们在Indexes前面加上了'-',意思就是禁止目录遍历,防止敏感文件泄露,此项非常重要,另外,关闭CGI,SSI,以及Follow Symbolic Links
    Options -Indexes -Includes -ExecCGI –FollowSymLinks 
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
<Directory "/var/html/8080">
    Options -Indexes FollowSymLinks
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
<Directory "/var/html/8888">
    Options -Indexes FollowSymLinks
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
LoadModule php5_module   modules/libphp5.so	# 该模块在你编译安装完php以后会自动生成并自动加入该配置文件
TraceEnable off 	# 禁用trace方法,防止xss

在我自己安装的时候，不知道为什么一直无法生成libphp5.so这个so文件，导致无法解析php文件 可能还是对linux这个操作系统没有那么熟练，编译安装这些不是很懂，后期懂了再加回来把 网站目录和文件就自己创建，这里不多说，这时候可以尝试启动httpd

\# cd /usr/local/httpd/bin
\# ./httpd -k start

被防火墙关了的话就开一下端口 成功启动 ![](http://blog.kingkk.com/wp-content/uploads/2018/03/f870d58912430b9f6f45a91dd2a7f09a.png) 主机访问一下 ![](http://blog.kingkk.com/wp-content/uploads/2018/03/7b6267fa48347e452c1765a9d3f72e84.png) 前期的安装阶段已经差不多了，后面来讲解一下配置的部分   尽可能的缩小权限，将网站目录所有者改为root，将除了上传目录以外的目录权限改为755，文件改为644

# chown -R root.root /var/html/bwapp/80/
# find /var/html/80 -type f | xargs chmod 644
# find /var/html/80 -type d | xargs chmod 755
# chown -R httpd.httpd /var/html/80/upload

  禁止上传目录中执行动态脚本，以减少上传漏洞带来的危害

<Directory "/var/html/80/upload">
<FilesMatch "\\.(?i:php|php3|php4|php5)$">
 Order allow,deny
 Deny from all
</FilesMatch>
</Directory>

或者将其以文本形式解析

<Directory "/var/html/bwapp/bWAPP/upload">
    AddType text/html .php
</Directory>

  将后台或指定目录限制指定ip段

<Directory "/var/html/80/admin_login">
 Order deny,allow
 deny from all
    allow from 192.168.3.0/24
</Directory>

  还有一些伪静态的配置和输入密码验证的就之后需要的时候再具体配置把，毕竟不是具体搞运维的，只是学习一下加固的思路和手段，以及方式。  

### 总结

总之服务器能做的还是比较有限，但是可以尽可能的限制危害的范围。最主要的还是代码层面，但是难免会疏漏，服务器层也是一层保障。 最后，感谢 https://klionsec.github.io/2017/11/26/apache-sec/