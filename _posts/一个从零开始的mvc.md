---
title: 一个从零开始的mvc
url: 337.html
id: 337
categories:
  - 学习笔记
  - 生活随记
date: 2018-05-12 15:06:15
tags:
---

# 前言

仔细算来，已经差不多有20天没有更新过博客了，自己的网站页差不多有十几天没来过了。好吧，其实是去coding去了，感觉没什么好记录也没什么需要记录的，也就没有写博客了。

# 正文

emm，身为半个野路子出来的人，没有做过开发，从小迪开始，零星的收集着知识点，用着一些大佬写好的工具，抓几个包，肆意的跑着。 
实际渗透起来，总觉得像是一个无头的苍蝇，乱飞，费力而且找不到想要的东西。 这学期一开学差不多当了两个月的赛棍，从n1ctf一直到最后参加的ciscn。国际赛，全国赛，学校看重的比赛，乱七八糟的比赛都参加了。中间writeup写了不少，也学了不少骚操作。。 
可是连续打到了上一个月底的时候，在写writeup的我总感觉有哪里不是很顺畅。说是没有从ctf中学到什么，那也确实不是，也学到了很多新姿势。 
 **知识太碎片化了**。 感觉还是基础太差，对整个web没有更加深入的理解，所以对一个网站，或者一次渗透时，不能找到关键的点。或许就是说是没有感觉。 
 这种感觉应该是来自于对于网站、对于功能点的洞察力。有时候或许会把它归为运气，但是我觉得，这种洞察力，对于一名黑客来说更加的极为重要。 
 思路和洞察力是指导的方向，而后期的一些测试更多的可以基于自动化的工具来完成。而ctf带来的或许是更加巧妙地利用姿势，可更加重要的还是那种感觉，对于漏洞的敏感。在哪可能会发生漏洞的一种洞察力。后面，或许可以借助更多的骚操作来绕过限制，攻下这个漏洞。然而，光有骚操作是绝对不够的。 
 于是我想重新回到代码，去熟悉网站的结构，去了解开发的流程（当然也是LoRexxar大佬的建议，非常感谢）。就决定先从暑假草草而过的MVC框架开始，从框架到各个功能的实现，自己写了一个小型留言的站。暂无借助任何外来的包。 
 [https://github.com/kingkaki/simple_guestbook](https://github.com/kingkaki/simple_guestbook) 

 下面内容帖自该项目的README.md

### [](https://github.com/kingkaki/simple_guestbook#simple_guestbook)simple_guestbook

主要目的是为了学习mvc框架，熟悉一些开发的流程

### [](https://github.com/kingkaki/simple_guestbook#功能介绍)功能介绍

#### [](https://github.com/kingkaki/simple_guestbook#两个个主要控制器)两个个主要控制器

*   index
*   user

#### [](https://github.com/kingkaki/simple_guestbook#控制器中的方法页面)控制器中的方法/页面

##### [](https://github.com/kingkaki/simple_guestbook#index)index

*   index (主页面，用于显示全部留言)
*   page (留言详情页）
*   add (新增留言)
*   save (添加留言功能页)
*   del (删除留言功能页)
*   commentadd (添加评论功能页)
*   commentdel (删除评论功能页)

##### [](https://github.com/kingkaki/simple_guestbook#user)user

*   index (个人信息页)
*   login (登录页面)
*   logout (登出页面)
*   register (注册页面)
*   uploadavatar (上传头像功能页面)

### [](https://github.com/kingkaki/simple_guestbook#后续)后续

#### [](https://github.com/kingkaki/simple_guestbook#一些功能特色)一些功能特色

*   sql查询使用pdo的预编译形式，防止sql注入
*   权限分离，在进行操作前会进行权限认证
*   自编写xss_escape对注册的变量进行递归html编码
*   对上传的文件进行重命名，并对上传类型进行限制

#### [](https://github.com/kingkaki/simple_guestbook#学到的一些东西)学到的一些东西

*   文件上传处理
*   sql预编译
*   权限认证

#### [](https://github.com/kingkaki/simple_guestbook#总结)总结

算是对mvc框架有了一定的认知 从框架到实现，算是真正的从零自己动手写了一个php的站（虽然前端页面丑的不想话说） 对文件的处理，以及数据的查询动手操作之后也有更深入的理解

#### [](https://github.com/kingkaki/simple_guestbook#后期学习计划)后期学习计划

开始代码审计吧~