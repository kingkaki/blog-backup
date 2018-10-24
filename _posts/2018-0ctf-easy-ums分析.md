---
title: 2018 0ctf-easy ums分析
url: 269.html
id: 269
categories:
  - CTF
date: 2018-04-06 23:22:26
tags:
---

# 前言

这题主要是利用条件竞争（Race condition）进行解题 
由于webserver一般都是多线程或者多进程运行，所以当代码未对数据操作上锁时，多个进程或者线程同时修改数据，导致数据呈现非预期的形式。简称条件竞争（自我翻译的）  

# 正文

题目注册账号时，需要输入一个ip地址，或者是网址 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/f377bb483f04c2b60b4c4cebd0698019.png) 
之后，他会往你的服务器上发送验证码
```
202.120.7.196 - - \[06/Apr/2018:23:01:15 +0800\] "HEAD /?af984075209abbeedd362234db0d4e8a HTTP/1.1" 200
```
通过验证后，你的地址就会变成之前验证的服务器 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/2e215aa00b682e3692e3b32e4d5380aa.png) 
题意也很明显了，要将服务器的地址变成8.8.8.8的，也就是谷歌的服务器（显然不可能拿到谷歌的服务器的） 
中间一些猜测和思维性推理就直接先略过了，直接说正经的解法把 服务器的数据库列表猜测大致如下

*   **mobile**（当前手机）：1.2.3.4
*   **verified**（是否已验证）：T/F
*   **code**（当前验证码）：a1b2c3…

条件竞争主要就是通过同时发送请求包，两个线程同时修改verified和mobile两个字段的值 
由于服务器做了针对phpseesionid的限制，同一个sessionid的请求强制顺序执行 
那就开两个浏览器进行抓包重发

* * *

先获取一次正常请求的验证码 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/b8c461fdb6c1f0b8265b5aec43780131.png) 
再抓取两个数据包，一个是发送验证码的数据包（修改是否验证通过数据），还有一个是更改ip地址的包（修改mobile数据） 

当两个数据同时被修改时，就会产生8.8.8.8的地址被通过 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/332b59b68072d2c2e687c0aabfbde77a.png) 
通过fiddler抓取，并同时发送 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/82fdb8c6f678a11acb297ea8337377ad.png) 
成功获取flag    

#  总结

说实在的，很多都是看着别人的writeup来的，但是考虑到这是自己第一次遇到条件竞争的题目，还是有必要记录一下。 
虽然看着似乎很简单，但是让自己来做可能还是不好想把。 
而且条件竞争好像很多还是很有用的，就当一次初识了…