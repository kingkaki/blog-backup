---
title: 关于阿里云一键安装之后phpwind首页的修改
url: 31.html
id: 31
categories:
  - 搭建与配置
date: 2017-08-31 12:18:36
tags:
---

### 相信不少用阿里云一键部署web的人都出现过这种情况，网站的首页莫名其妙被导向了安装包中自带的phpwind。今天就来说一下重新自定义网站首页的方法**。（PS:只是想知道修改方法可以直接跳到第五部分）**

 

#### 1.首先一开始我也是各种百度和谷歌，发现网上介绍的大致如下两种方法：

1.  修改/alidata/server/httpd-2.4.10/conf/下的httpd.conf配置文件中下面那两行代码的文件位置。
    
    DocumentRoot "/alidata/www/"
    <Directory "/alidata/www/">
    
2.  修改/alidata/server/httpd-2.4.10/conf/vhosts 下的phpwind.conf文件。

 

#### 2.后来两种方法都试了下：

1.  第一种方法应该对于普通自己搭建的web服务可以修改网站的自定义首页，但是在阿里云的一键部署中并没有作用。
2.  第二种方法看网上的说法是有人成功了，可是我在本地试了之后,service httpd restart 重启服务却出现了如下的错误提示。![](http://106.14.162.156/wp-content/uploads/2017/08/1.png)

#### **3.于是后来就去目录里寻找，发现是/alidata/server/httpd-2.4.10/conf/extra下的http-vhosts.conf在启动的时候引用了phpwind.conf文件，然后打开后发现就仅仅只有一句引用的话，既然这样，于是做好备份后就把这个conf文件给删了**

![](http://106.14.162.156/wp-content/uploads/2017/08/2.png)

#### 4.再次重启，出现了如下提示

![](http://106.14.162.156/wp-content/uploads/2017/08/3.png)  

#### 5.于是再去/alidata/server/httpd-2.4.10/conf文件夹下找到httpd.conf文件，找到其中关于httpd-vhosts.conf引用的部分（即下面截图选中的部分），然后删掉这句话（为了让它不再引用这个配置文件），再次重启服务

![](http://106.14.162.156/wp-content/uploads/2017/08/4.png)  

#### 6.完美！成功的重启了服务，没有报错信息，再次登录站点，就是/www主目录，这时候再按照前面的修改httpd.conf配置文件的方法，就能把网站首页导向相应的文件夹了。![](http://106.14.162.156/wp-content/uploads/2017/08/5.png)

  相信不少用阿里云一键部署web安装的人都会遇到这个问题，希望可以帮到那些想更换的网站的首页人。 /*吐槽一下： 明明一键部署web的安装包还要收费，自己并非不会自己去配置lamp,但是图着省事也就购买了阿里云的一键部署安装包，自绑定了phpwind不说，还把网站首页这样的绑定在上面。 而且在阿里云把phpwind的安装也直接放在了web的部署里面，到后面教你安装wordpress的时候也就是直接把wordpress再放到phpwind文件夹里（这简直坑爹啊）。 本来是想着省事买的安装包，这样一来花了钱先不说，而且去找这些的解决方法也花了大把的时间。 */