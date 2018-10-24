---
title: 代码审计练习篇-xdcms
url: 186.html
id: 186
categories:
  - 代码审计
  - 学习笔记
date: 2018-01-30 20:41:38
tags:
---

看完了《代码审计》的基础部分，然后想找几个实战练练手，毕竟不会操作都tm白搭，于是看了一期实战的视频，跟着一起学习了下审计操作流程。 
本次审计的cms为：XDcms_v2.0.8(中文的话叫旭东cms) 视频挖的是一个sql注入漏洞，我这里就不赘述他的挖掘过程，就记录下两个自己挖的过程

### SQL注入漏洞

一开始都是丢进seay代码审计跑一跑，跑出一堆可能有风险的函数，然后反向审计（回溯） 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/888dadf3f5a940a8fc40877f17b2a1b9.png) 
一堆。。。或者也可以按照视频里的，去全局搜索$\_GET和$\_POST，查看哪些未被过滤的值 找到一个这个 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/ffba836ddb0529df7e760ace3eef90e8.png) 
溯源回去 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/9138038a24526815c3e09b15e52c2b57.png) 
蛮多没有过滤的值，那就选择控制title\_weigt参数 看看哪里调用了这个函数add\_save() emmm应该就是这个 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/cbae44b0582c7e4fb6d45945003966a8.png) 
进入`index.php?m=xdcms&c=content&f=add_save`试试 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/4072587b915bd96c85a20268ff4f85e3.png) 
好像不能直接进入，那就去掉后面一个参数`index.php?m=xdcms&c=content` 尝试着发布内容然后抓包，但是每次提交上去都说我没填写完整，最后弄了半天，改了一个参数终于可以了，晕。。 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/7d21cf67e3407c0faac4d5eeaf1ac643.png)
把这个catid从原来的0改为1就可以成功上传了，这是程序的bug吧==、 因为之前看过这个程序封装好的执行sql语句的函数，会报错。。然后就可以尝试报错注入 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/3da1e2ff5468fd66c30f300d8c71952b.png) 
将title_weight设为

db' and 1=updatexml(1,concat(1,(select database())),1))#

![](http://blog.kingkk.com/wp-content/uploads/2018/01/17601afe45051b1bba9a24ceaddf1f6e.png) 
嘿嘿，数据就出来了。 后来想着假如没有报错呢，试试盲注？ 于是试着把payload改成了

db') ; select sleep(2);#

好像不行。。根本没有延时，尝试这样报错

db' and 1=updatexml(1,concat(1,(select database())),1));#')

貌似也不行，，而且我在我mysql终端执行都是没问题的，原因至今未知。。。 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/57ace8716ca4b5215086d43c1439e350.png) 
但是这样盲注入又可以

db' or sleep(3))#

![](http://blog.kingkk.com/wp-content/uploads/2018/01/8ebb3db442ce1459b5aab7e6b340169d.png) 
就当算学了一手备用。。    

### 文件上传

一开始想着，毕竟别人一开始挖的都是sql，那自己得挖点别的类型的，然后无意中发现了如下代码 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/d7fc6da99b062c69d1b3932f454fdee3.png) 
emmmm！这好像是个黑名单的过滤方式 于是开始不停的溯源。。真的是找了好久，有点头皮发麻，因为这个好像是给很多地方用的一个总的上传函数 可以用来过滤图片、媒体、以及自定义文件，然后每种方式会设置一些不同的参数，以进行不同的过滤 
最后，在一个地方看到了
![](http://blog.kingkk.com/wp-content/uploads/2018/01/fa7c27369a05b4d8722fe834918995c0.png) 
貌似要开启这个isopen参数，然后继续溯源。。全局搜索，然后 
![](http://blog.kingkk.com/wp-content/uploads/2018/01/cab36511bf62e30afaad048213140134.png) 
最后在后台找到了这个地方 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/aab87560d3b2f485b3fe401397bcef88.png) 
发现其实默认是开启的，但是有个文件上传格式限制，然后就加了个php3（避开之前黑名单的） 然后找了个地方上传文件 会员这新增了一个files的类
![](http://blog.kingkk.com/wp-content/uploads/2018/01/a4f8b0482cf420ee4629fc8b51c61c2a.png) 
然后![](http://blog.kingkk.com/wp-content/uploads/2018/01/75f2dbc2821cf920bca25a733245d11c.png)
![](http://blog.kingkk.com/wp-content/uploads/2018/01/da6f552fa46a4953b0194753d9b5a16f.png) 
后来测试了下，新增php类的话，他会把你的文件变成txt 然后文件位置是在\\uploadfile\\file\\20180130\\201801301257570.php3 实际渗透中可以python跑个脚本就可以知道文件名了 
然后上菜刀，bingo  
emmm至于为什么网上的phtml、pht之类的不解析我也不清楚 cer的话只有在iis上才解析这个倒是弄明白了，到时候再琢磨下吧      

### 总结

1.  目前的经验来说，不是很建议纯白盒测试，因为实在代码有点头晕，相互牵连的地方很多，可以进行适当的黑盒，进行辅助测试
2.  多看代码注释，可能是自己代码功底也差，看注释能省很多麻烦
3.  熟悉下图形界面，多用下这个网站功能，熟悉下操作，不然上来直接看源码真的头疼，而且很多时候就算找到了点，也还要回过头来抓数据包。
4.  测试的时候可以echo、var_dump输出一些变量的值，配合exit，来帮助我们了解值的变化
5.  sublime的ctags真的超好用，定位贼方便，然后反回溯的话善用ctrl+shift+f可以全局搜索
6.  是得学学MVC了，不是很懂大体框架逻辑，看着代码头晕T_T