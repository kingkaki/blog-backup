---
title: apache配置多站点
url: 141.html
id: 141
categories:
  - 搭建与配置
date: 2017-12-29 19:33:31
tags:
---

由于最近需要搭建一个自己的xss平台，想用一个xss.kingkk.com的子域名，由于之前自己也一直好奇，是怎么把一个服务器下的不同站点指向不同的域名。今天就花了一个下午的时间去弄了下这件事，顺便记录下。  

1.  1.  找到域名解析这里![](http://blog.kingkk.com/wp-content/uploads/2017/12/c85d3a23c68f4fc57c327d25ea296c61.png)
    2.  点击进去之后添加解析，主机记录填写自己想分配的子域名，记录值处填写你的ip地址![](http://blog.kingkk.com/wp-content/uploads/2017/12/2f9f4962b358123203ae97ba8ba0faa5.png)
    3.  接下来，连上自己的云服务器，我的是lamp的环境，所以找到`/alidata/server/httpd-2.4.10/conf/vhosts` 这个路径下，新建一个xxx.conf文件，写入如下代码（由于格式话不了，只能放截图，懒得敲字的可以直接[下载](http://blog.kingkk.com/wp-content/uploads/2017/12/xss_conf.txt)，然后改下后缀即可）ServerName为域名，ServerAlias为子域名，DirectoryIndex为默认首页，不设置就是index.php。MirectoryMatch以及DocumentRoot、Directory路径为网站的路径，都需要修改![](http://blog.kingkk.com/wp-content/uploads/2017/12/bf19c854beda3dad61376fa7a7e03905.png)
    4.  最后再在httpd.conf中修改一下配置，将网站根目录改为www目录，修改这两个地方就可以![](http://blog.kingkk.com/wp-content/uploads/2017/12/ec346d4fa787c7cdb608a11b28dd05fd.png)
    5.  最后再加上一句![](http://blog.kingkk.com/wp-content/uploads/2017/12/bebcb12c563aa012ca503c4abe560766.png)即可
    6.  最后service httpd restart 重启一下httpd服务就好了，就可以解析到不同的子域名了

![](http://blog.kingkk.com/wp-content/uploads/2017/12/81e11e52eb6f2eae6bd1ca92c6a460c9.png) 后期还要加不同的子域名的话，就修改另存为一下那个xss.conf文件，然后再将该文件Include进httpd的配置文件就好啦，是不是很方便