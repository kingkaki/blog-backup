---
title: SQL注入之万能密码
url: 53.html
id: 53
categories:
  - sql注入
  - 学习笔记
date: 2017-09-12 20:25:51
tags:
---

' or 1='1

这估计是最最简单的一个万能密码了，原理也就不过多解释，今天就记录刚刚看到的一种万能密码（数据库类型为mysql,并为在其他数据库中进行测试）

username= 1’=’0 
password= 1’=’0

username=what’=’ 
password=what’=’

username:admin’=’ 
password:admin’=’

当or、and以及注释符号被严格过滤时，便可以选择这种万能密码 接下来讲述一下这个万能密码的原理，假设原SQL查询语句为

select * from table where username= 'username'and password='password’

以第一个万能密码为例（三个万能密码的原理相同），第一个万能密码插入SQL语句时，语句为

select * from table where username= '1'='0'and password='1'='0’

这段代码利用了连等符号，首先对username=‘1’进行判断，返回false,然后在将false与‘0’进行比较，当两种数据类型不同时，会进行强制的类型转换，所以‘0’的返回值也是false，两个false相比较，返回值为true。然后后面password的比较同理 所以，其实上面的这种万能密码还有很多的变换形式，将第一个值设为某个不存在的值，然后将第二个值设为能返回flase的值，如0，‘0’，‘’，flase都可。（下面附上在截图） ![](http://www.kingkk.com/wp-content/uploads/2017/09/EPG53J6XCL2YA0CZSP.png)