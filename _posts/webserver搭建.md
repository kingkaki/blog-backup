---
title: Webserver 搭建
url: 148.html
id: 148
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-18 18:06:37
tags:
---

emm因为在先前的学习中，发现对linux许多操作与配置极其不熟练，各种百度谷歌，还不一定弄得对，所以最近想系统学习下linux的操作，那就先从webserver服务开始。废话不多说，直接上吧。

#### 环境：

*   CentOs 6.5
*   Apache 2.2.15
*   虚拟机

tips:要想虚拟机上运行的服务可以在电脑本机上能够访问，需要设置为桥接模式  

### Apache

1、安装apache :

yum install httpd

2、如果防火墙是开着的，为了方便这里就直接把防火墙关了

service iptales stop

3、找到httpd.conf配置文件修改如下内容（默认路劲为`/etc/httpd/conf/httpd.conf`）

#ServerName www.example.com:80  -->  ServerName localhost:80

4、启动apache

service httpd start

这时候在主机访问一下，应该就可以进入apche的默认页面了    

### Apache虚拟主机

1、打开httpd.conf文件，找到`virtual host being defined` 这一行 2、在其下面加入类似如下格式（也可以改用Include方式）
```
<VirtualHost *:80>
    ServerName test1.centos.com
    DocumentRoot /var/www/html/test1
    <Directory "/var/www/html/test1">
        Options Indexes FollowSymLinks
        AllowOverride None 
    </Directory>
</VirtualHost>

<VirtualHost *:80>
    ServerName test2.centos.com
    DocumentRoot /var/www/html/test2
    <Directory "/var/www/html/test2">
        Options Indexes FollowSymLinks
        AllowOverride None
    </Directory>
</VirtualHost>
```
3、将#NameVirtualHost *:80的注释去掉（否则貌似只能配置一个虚拟主机）
```
#NameVirtualHost *:80 -->  NameVirtualHost *:80
```
4、关闭防火墙（或者设置下80端口，我这嫌麻烦直接关闭了）
```
service iptables stop
```
5、重启apache
```
service httpd restart
```
6、修改本机host文件，加入类似如下语句（假如是外网ip，直接设置ip解析）
```
192.168.1.104 test1.centos.com
192.168.1.104 test2.centos.com
```
接下来，应就可以解析到不同网站了 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/79a87f70562ec1bae26e9c8b97430651.png)
![](http://blog.kingkk.com/wp-content/uploads/2018/01/828b9b1e0e9d39222b96019245e2671a.png) 
而且我现在还没搞懂怎么将目录弄到别的目录，而且很多设置按照教程了也是不对的，很多还是自己一点一点碎片化的搜出来的，而且有些也跟版本有关。以后在细细琢磨吧，现在弄的有点头大。    

### Apache 伪静态

1、在http.conf中加载rewrite_module模块，我这里是默认加载了的，加入没有的话就自行加载一下

LoadModule rewrite\_module modules/mod\_rewrite.so

2、在虚拟主机的配置块下的 **<Directory>标签下**，加入如下语句（大意为将psp结尾的网页导向index.html）
```
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*).psp$ index.html
</IfModule>
```
3、重启httpd即可

service httpd restart

![](http://blog.kingkk.com/wp-content/uploads/2018/01/ffd363238a4307af8664f05563f03f66.png)  

### Nginx 安装

1、由于yum中没有内置nginx,所以不能通过yum方式下载安装，只能去官网下载安装包[https://nginx.org/download/](https://nginx.org/download/) 2、通过wget下载安装包，没有wget的可以通过yum下载一个（这里下载的是1.10.1版本）
```
wget -c https://nginx.org/download/nginx-1.10.1.tar.gz
```
3、解压

tar -xzvf nginx-1.10.1.tar.gz

4、下载一些依赖环境
```
yum install gcc-c++ 

yum -y install pcre*  

yum -y install openssl*
```
5、进入解压后文件夹，并配置安装目录
```
cd nginx-1.10.1
./configure --prefix=/usr/local/nginx
```
6、编译安装
```
make
make install
```
7、切换目录并启动

cd /usr/local/nginx/sbin
./nginx

![](http://blog.kingkk.com/wp-content/uploads/2018/01/98b4d5d6d42100ba4dacfc6090a963af.png)
为了日后使用的方便，可以将其添加进系统服务，便可以通过service进行操作 1
、新建文件
```
vim /etc/init.d/nginx
```
2、加入如下内容

#!/bin/sh 
\# 
\# nginx - this script starts and stops the nginx daemon 
\# 
\# chkconfig:   - 85 15 
\# description: Nginx is an HTTP(S) server, HTTP(S) reverse  
#               proxy and IMAP/POP3 proxy server 
\# processname: nginx 
\# config:      /etc/nginx/nginx.conf 
\# config:      /etc/sysconfig/nginx 
\# pidfile:     /var/run/nginx.pid 
 
\# Source function library. 
. /etc/rc.d/init.d/functions
 
\# Source networking configuration. 
. /etc/sysconfig/network
 
\# Check that networking is up. 
\[ "$NETWORKING" = "no" \] && exit 0
 
\# 这里要根据实际情况修改
nginx="/usr/local/nginx/sbin.nginx"
prog=$(basename $nginx)
 
\# 这里要根据实际情况修改
NGINX\_CONF\_FILE="/usr/local/nginx/conf/nginx.conf"
 
\[ -f /etc/sysconfig/nginx \] && . /etc/sysconfig/nginx 
 
lockfile=/var/lock/subsys/nginx 
 
start() {
    \[ -x $nginx \] || exit 5
    \[ -f $NGINX\_CONF\_FILE \] || exit 6
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX\_CONF\_FILE 
    retval=$?
    echo
    \[ $retval -eq 0 \] && touch $lockfile 
    return $retval
}
 
stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT 
    retval=$?
    echo
    \[ $retval -eq 0 \] && rm -f $lockfile 
    return $retval 
    killall -9 nginx
}
 
restart() {
    configtest || return $?
    stop 
    sleep 1
    start
}
 
reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP 
    RETVAL=$?
    echo
}
 
force_reload() {
    restart
}
 
configtest() {
    $nginx -t -c $NGINX\_CONF\_FILE
}
 
rh_status() {
    status $prog
}
 
rh\_status\_q() {
    rh_status >/dev/null 2>&1
}
 
case "$1" in
    start)
        rh\_status\_q && exit 0
        $1
        ;;
    stop)
        rh\_status\_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh\_status\_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh\_status\_q || exit 0
        ;;
    *)    
      echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac


3、然后就可以和之前一样，通过service的方式对ningx进行操作    

### Nignx 配置虚拟主机

1、找到并打开conf目录下的nginx.conf文件（我这路径为`/usr/local/nginx/conf`） 

2、在http的大框下写入如下语句
```
 server {
    listen 80;
    server_name www.centos.com;    #网址
    root /data/www;                #网页文件路径
    index index.html index.htm;    #起始页面
 }
```
3、修改本机的host文件 4、重新加载nginx

service nginx reload

![](http://blog.kingkk.com/wp-content/uploads/2018/01/37d213d49754184b08b6894fd0a13326.png) 此处也可以改用include的方式，在conf目录下新建vhost文件夹 在其中新建.conf文件，并加入之前的语句 在nginx中的http范围下`include vhost/*.conf;`      

### Nginx 伪静态

1、在先前的虚拟主机配置块中加入如下语句（大意为将.psp结尾的文件指向index.html）
```
 location / {
     rewrite ^(.*)\\.psp$ /index.html;
 }
```
2、重加载nginx

service nginx reload

加载成功 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/ccddd4d0b4460fa405caa94eac3aaa6a.png)    

### Nginx 反向代理

1、最简单的方式，在之前虚拟主机的配置中，加入如下语句
```
location / {
      proxy_pass http://www.kingkk.com;
 }
```
2、重新加载nginx
```
service nginx reload
```
成功反向代理到我的个人博客了 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/5b6c17ab366f89e0a67e0a668bd23dcb.png) 
麻烦一点的话就是 将添加的语句改为
```
location / {
     proxy_pass http://kingkk_host;   #在此处将其取一个别名
 }
```
然后再在其他地方写入
```
upstream kingkk_hosts{
     server www.kingkk.com;
}
```
可以做到将两者分离，使代码简洁易懂    

### Nginx 负载均衡

1、在之前反向代理的基础上 设置upstream kingkk_hosts
```
upstream kingkk_hosts{
    server 127.0.0.1:80;
    server www.kingkk.com;
}
```
2、重新加载服务
```
service nginx reload
```
![](http://blog.kingkk.com/wp-content/uploads/2018/01/63388be8d695c03915f82afc19e8889a.png) 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/468c79de21974f81d033d5e64ee82db5.png) 
不断刷新页面就可以发现在这两页面之间来回切换，也就意味着来回向这两个服务器请求资源   还可以设置权重，默认权重为1：1：1…
```
upstream kingkk_hosts{
   server 127.0.0.1:80 weight=3;
   server www.kingkk.com weight=1;
}
```
就可以达到3：1的这样一个切换效果