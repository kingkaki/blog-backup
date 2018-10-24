---
title: sqli-labs练习（less11-22）
url: 125.html
id: 125
categories:
  - sql注入
  - 学习笔记
date: 2017-12-14 21:02:07
tags:
---

之前less1-10的主要通过get注入，less11-20几乎通过POST注入

### less-11

先检测出是post方式提交的数据（最简单的方法就看有没有增加参数嘛） 然后可以通过抓包或者审查元素的方式获取到post的参数，如下三个

*   uname=1
*   passwd=1
*   submit=Submit

然后就可以去构造然后去post发包啦 首先在uname字段构造个万能密码，然后通过#注释后面的内容 ![](http://www.kingkk.com/wp-content/uploads/2017/12/812f47dda83bb6f2db8c90ac2e0855fd.png) 或者也可以尝试闭合后面的引号 ![](http://www.kingkk.com/wp-content/uploads/2017/12/2c3b30334ec87f172d21dddf134aafa7.png) 可以看到，登录失败了，原因在于or语句之后还有一句and语句，因为并没有password=1这个字段，所以该语句为假，返回为空。 至于网上说的and优先级比or高的说法其实并不认同，个人认为两者运算优先级应当是相同的，在没有括号的情况下严格按照从左至右的顺序进行判断，也可以测试得出 ![](http://www.kingkk.com/wp-content/uploads/2017/12/ceec7a095e9ed737e9861a6097a4b152.png) 然后就有两种方式可以注入：

*   在uname字段注释后面内容然后进行注入
*   在passwd字段注释后面内容，或者构造or永真语句。

具体注入方法和之前类似，就不赘述 ![](http://www.kingkk.com/wp-content/uploads/2017/12/622bde285122550c995c6342c29afd96.png)  

### less-12

进行简单的测试，发现双引号引起报错，并且 ![](http://www.kingkk.com/wp-content/uploads/2017/12/ef68cac5200acbc84a3905c74891cbe9.png) 闭合括号与双引号即可， 然后和之前一样，开始注入就行了（这里正好前面没有uname=1和passwd=1的字段，所以刚好就返回union select的这一行字段） ![](http://www.kingkk.com/wp-content/uploads/2017/12/0b7c92657b3320ee2a2b99d8c3b7c732.png)  

### less-13

单引号测试，报错 ![](http://www.kingkk.com/wp-content/uploads/2017/12/c0ad3fe79e73d14f94360ce24087df26.png) 闭合下括号，即可，但是，不显示数据。。。 ![](http://www.kingkk.com/wp-content/uploads/2017/12/03520e48c2325d3e07e5c2753dad65f3.png) 没办法了，布尔盲注吧，至少比基于时间稍微快点。附上payload `passwd=1') or ascii(substr((select database()),1,1))>10 #`    

### less-14

一样了，闭合用双引号即可 payload:`passwd=1" or ascii(substr((select database()),1,1))<10 #`  

### less-15

没有sql报错语句，只能通过加入永真语句，然后观察登录与否来判断 ![](http://www.kingkk.com/wp-content/uploads/2017/12/ef69ceb31209ea50e2ff5891afac06b2.png) ok,单引号注入 然后继续布尔盲注： `passwd=1' or ascii(substr((select database()),1,1))>10#`    

### less-16

测试`passwd=1")or 1=1#`登录成功， 然后，也一样了，布尔大法 `passwd=1")or ascii(substr((select database()),1,1))<10#`    

### less-17

这个就有类点类似于一个修改密码的选框 输入指定的username，以及要更新的password，通过update语句，更新数据库中的password中对应username的password字段。反正这个也是看了别人写的才看懂的把。 先分析下源码。![](http://www.kingkk.com/wp-content/uploads/2017/12/8c0edcf52402c7126f0f53764beefc6c.png)这里对uname做了较为严格的过滤，注入只好从passwd入手 这里就要涉及到另一个函数updatexml，虽然这个函数与xml有关，但其实使用它注入时并不怎么需要了解xml的知识，只是通过输入错误的第二个参数，然后报错返回我们需要的数据结果即可(还有个前提，要提前知道某个用户名存在，就猜测admin了..) `passwd=a ' or 1=updatexml(1,concat(0x2a2a2a2a2a,(select database())),1)#`测试了下，0x2a2a2a中2a的个数为基数时数据才显示完整，具体原因不知==、 ![](http://www.kingkk.com/wp-content/uploads/2017/12/6f90bc5e193dca8e590425aa78499c37.png) 这样即可通过xpath的报错，来获取想要的数据（tips:第一个参数一定要加，否则报错不会把数据呈现出来，第三个参数随意） 然后按照老套路注入，然后到最后获取数据的时候，会出现一个小问题 ![](http://www.kingkk.com/wp-content/uploads/2017/12/279807dc6bc377328bfedfba583595e4.png) 说是不能获取同一列中的数据，然后参考了下别的大佬的方法（具体原理不是很懂） 把sql语句改为`select password from (select password from users where username='admin')a` 就是说在加一层select 然后再给他随便取一个别名即可 ![](http://www.kingkk.com/wp-content/uploads/2017/12/4237a7294f98e5b9b97e4311f0b5469e.png) 其实感觉假如不怕费时的话还是盲注大法好，不过就是会毁数据库。。不过这也算多学了一招，但是条件也比较苛刻，必须有报错语句返回    

### less-18

这道题的本意应该是让我们通过已经登录的身份，然后在对uagent字段进行注入，因为源码中uname和passwd都进行了过滤 ![](http://www.kingkk.com/wp-content/uploads/2017/12/8ac549564be742ce5fca62524a5c69c1.png) 研究下sql语句 ![](http://www.kingkk.com/wp-content/uploads/2017/12/0ed3f95b44b72f39ca099929f8a83d28.png) 本来想用burpsuite抓包的，但是不知道为什么我这burpsuite无法往localhost发送数据包，然后自己用python随手写一个脚本来发包把， 先附上payload：`Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:56.0) Gecko/20100101 Firefox/56.0 **'or updatexml(0,concat(0x2a2a2a,version()),0),'0','0') # **`（后面两个'0'，以及一个括号是为了闭合语句的，不然会报sql错误） ![](http://www.kingkk.com/wp-content/uploads/2017/12/4963a23fa5d0222a4e4d8643075249b8.png)然后能愉快的看到数据了，然后就修改那个version部分来读数据吧。 ![](http://www.kingkk.com/wp-content/uploads/2017/12/7ea32f6136f80e1bc24a8f00cb06ffee.png)    

### less-19

登录进去之后会显示你的referer信息，大概意思就是说通过http头中referer进行注入 观察下sql语句，![](http://www.kingkk.com/wp-content/uploads/2017/12/f0da8518998283c01fb9e420a9ebcb5e.png) 然后开始注入，本来是想用hackbar自带的referer进行注入的，后来发现好像会转义你的单引号等字节，所以还是用了py ![](http://www.kingkk.com/wp-content/uploads/2017/12/35859111319eebf7297d2288f8c63af7.png) 和之前的基本差不多，就是把注入点改了下 再介绍下另一个函数extractvalue，其实目的和使用方法与updatexml差不多，只是参数变成了两个 ![](http://www.kingkk.com/wp-content/uploads/2017/12/88b21e03e1a4b253be51aa640bf82d2a.png)      

### less-20

分析下源码 大概意思就是说，你第一次登录之后会帮你设置一个uname的cookie，然后保存你的信息。然后只要这个cookie存在，就显示你的信息。 这里依旧uname和passwd都被过滤了，唯一一个点就是![](http://www.kingkk.com/wp-content/uploads/2017/12/3066851b8fe84cece1595d8c757e4c50.png)这个cookie值 然后修改cookie中uname的值进行注入 ![](http://www.kingkk.com/wp-content/uploads/2017/12/ef9feac935e8fa037c85e1346873bdc7.png)    

### less-21

这里对uname的值进行了一次base64的加密，于是，只要改一下之前的payload（闭合个括号）在用base64加密一下即可 ![](http://www.kingkk.com/wp-content/uploads/2017/12/e73458ca6f787b63276c9babdecef044.png)    

### less-22

感觉没什么好讲的，变成双引号闭合而已 ![](http://www.kingkk.com/wp-content/uploads/2017/12/6a9e6cfae0859176d3aeb304b739064c.png)  

### 总结

这里主要修改了注入的点，比如POST数据,http头，以及cookie，不过感觉更麻烦的就是黑盒测试，不清楚到底哪里是注入点，以及改如何闭合语句，感觉还是需要多加锻炼。 终于也算把基础的刷完了，了解了注入的大概思路，后面就继续进阶advanced injection吧，学习些更骚气的姿势         最后，感谢 [http://blog.csdn.net/u012763794/article/details/51361152](http://blog.csdn.net/u012763794/article/details/51361152)