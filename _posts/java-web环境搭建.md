---
title: java web环境搭建
url: 172.html
id: 172
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-20 23:51:58
tags:
---

**系统环境centos6.5**

### 安装java包

yum install java-1.8.0-openjdk*

有点大，等一下就好了，好像有点简单啊。。 **检验下是否安装完毕**

\[root@localhost ~\]# **java -version**
openjdk version "1.8.0_161"
OpenJDK Runtime Environment (build 1.8.0_161-b14)
OpenJDK Server VM (build 25.161-b14, mixed mode)

 

### 安装tomcat

1、weget下载tomcat包

wget http://mirrors.shu.edu.cn/apache/tomcat/tomcat-8/v8.5.24/bin/apache-tomcat-8.5.24.tar.gz

2、解压进入bin目录

tar -xzvf apache-tomcat-8.5.24.tar.gz
cd ~/apache-tomcat-8.5.24/bin

3、启动tomcat

sh startup.sh

这时候就启动了一个tomcat进程，默认端口为8080，通过浏览器访问一下，出现如下画面，就说明启动成功 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/cb88847b15efe92c8d39fe12894be169.png)    

### Apache 反向代理tomcat

1、确认如下模块，并开启，我这里是默认开启的

*   mod_proxy.so
*   mod\_proxy\_ajp.so
*   mod\_proxy\_balancer.so
*   mod\_proxy\_connect.so
*   mod\_proxy\_http.so

2、在httpd.conf文件末尾，加入如下代码（也可以采用include的方式）

<VirtualHost *>
   ServerName www.centos.com
   ServerAlias www.centos.com
   ProxyPass / http://localhost:8080/
   ProxyPassReverse / http://localhost:8080/
</VirtualHost>

没设置过host的需要设置一下host 3、重启httpd

service httpd restart

运气好的话，重新打开该网页，就可看到tomcat被反向代理了 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/6260057703beb1b26904f9176d11ca19.png) 像我这种运气不好一点的，打开之后就是503，折腾了好久，反向时selinux中的一个设置不对，坑爹呀。。

/usr/sbin/setsebool -PV httpd\_can\_network_connect=1

这样开启httpd\_can\_network_connect之后，就能正常访问了    

* * *

nginx的反向代理，那个webserver搭建中有类似的，就不复述