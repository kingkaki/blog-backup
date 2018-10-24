---
title: Python2、3的兼容并切换
url: 180.html
id: 180
categories:
  - python
  - 搭建与配置
date: 2018-01-21 19:02:48
tags:
---

如今，python3已经逐渐在替代以及成为未来Python的一个大势所趋，然而由于时间问题，python2又是许多好用的，前人所开发的语言，一时半会儿又丢弃不了。所以经常会出现python2、python3都需要的场景，一直更换环境变量，总是会很麻烦。其实python官方也有考虑到这个问题，所以还是有办法同时切换python2、3的，只是需要一些小小的操作

#### 首先要将Python2、3都添加进环境变量

![](http://blog.kingkk.com/wp-content/uploads/2018/01/ecf44165ff4ab02d0aa3e5d641a06968.png)

#### 运用py

**`py -2`**时调用python 2 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/0c2979f5403cff66b0edc8c2713b0ac4.png) **`py -3`**时调用python 3 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/86a19ef49ebae7c3956d9186ada4e08e.png)  

#### 运行脚本时

python2 的脚本需要在脚本前加上

#! python2

python3 的脚本则需要加上

#! python3

###### 运行时都用py xxx.py即可

 

#### pip安装

分别要在不同环境下安装库时 当需要python2的pip时，只需

py -2 -m pip install xxx

当需要python3的pip时，只需

py -3 -m pip install xxx

 

        转载自[https://www.cnblogs.com/shabbylee/p/6792555.html](https://www.cnblogs.com/shabbylee/p/6792555.html)