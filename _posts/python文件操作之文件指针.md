---
title: Python文件操作之文件指针
url: 45.html
id: 45
categories:
  - python
  - 学习笔记
date: 2017-09-06 23:25:05
tags:
---

最近在学习Python的文件操作，发现了一个比较有意思的东西，便随手记录一下。 首先先列举一下Python常用的文件打开方式

*   r      只读
*   w    只写，如果文件不存在，则创建，如果文件存在，则覆盖文件
*   a     追加写，如果文件不存在，则创建文件
*   r+   读、写
*   w+  读、写，如果文件不存在，则创建，如果文件存在，则覆盖文件
*   a+   追加打开文件，可读可写，如果文件不存在，则创建文件

上述就是一些常用的文件打开方式（mode），然后利用open(name\[, mode\[, buffering\]\])即可打开   然后就来介绍一下Python文件操作时的文件指针 首先，在d盘下创建一个demo.txt文件，然后随意输入点什么，关闭文件，然后在电脑中打开，发现文字是被顺利的写了进去

>>\> fp=open("d:\\\demo.txt","a+")
>>\> fp.write("This is a Python test")
>>\> fp.close()

![](http://106.14.162.156/wp-content/uploads/2017/09/HZ_2HXWYAM5NGNNNNXI.png) 然后再次打开，用read()函数连续两次读取demo.txt文件的内容

>>\> fp=open("d:\\\demo.txt","r+")
>>\> fp.read()
'This is a Python test'
>>\> fp.read()
''

发现仅有第一次有内容输出，第二次read的时候仅仅是''，然而close之后再次打开demo.txt依然是有内容的 这就是Python的**文件指针**所造成的 由于Python的底层是由C实现的，自然而然的会涉及到C语言中的精髓——指针 Python通过open()函数打开文件的时候，文件指针会指向打开文件的起始地址 通过read()函数对文件内容进行输出到屏幕的时候，文件指针会以此指向文件的内容，然后输出，直至内容结束 所以，当read()函数执行完毕的时候，文件指针便指向了文件的末尾 这也就解释了为什么当执行第二个read()函数的时候只会输出 ''   Python的文件处理中也有相应的函数可以**查询文件指针的位置——tell()**

>>\> fp=open("d:\\\demo.txt","r+")
>>\> fp.tell()
0L
>>\> fp.read()
'This is a Python test'
>>\> fp.tell()
21L

可以看到，正如前面所说的，一开始打开文件时，文件指针是指向文件开头的。在一次read()之后，文件指针指向了文件末尾，21也正好是这段文本的长度。   Python也提供了**移动文件指针的函数——seek()**

>>\> fp=open("d:\\\demo.txt","r+")
>>\> fp.seek(8)
>>\> fp.tell()
8L
>>\> fp.read()
'a Python test'

也很容易看出tell()所返回的指针位置正是seek()函数中所指定的参数位置，而且此时再次使用read()函数时，就仅会输出第七个字符之后的内容“a Python test”