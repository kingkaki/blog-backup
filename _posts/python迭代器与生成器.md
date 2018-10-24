---
title: Python-迭代器与生成器
url: 211.html
id: 211
categories:
  - php
  - 学习笔记
date: 2018-02-10 13:48:05
tags:
---

### 迭代器（Iterable）

可以被for循环，但是不能被next调用 如：list、tuple、set、dict以及生成器等

### 生成器（Iterator）

可以被for循环，并可以通过next调用下一个值 如：range()等可以被next()调用的对象  

##### tips:生成器属于迭代器，但迭代器不一定是生成器

 

* * *

### 详解生成器：

**通过yield不断生成数据，并通过next不断调用**

def myfun():    
    for x in range(5):
         yield i

mf = myfun()

>>\> next(k)
0
>>\> next(k)
1
>>\> next(k)
2
>>\> next(k)
3
>>\> next(k)
4
>>\> next(k)
5
>>\> next(k)
Traceback (most recent call last):
 File "<stdin>", line 1, in <module>
StopIteration

通过next()不断获取下一个值，直至获取完毕时，抛出一个StopIteration异常 本质是每调用一次next()函数之后，生成器函数开始执行，碰到yield函数时生成一个量，然后停止，再次next时，从上次停止的地方开始执行，直至遇到下一个yield

>>\> def myfun():
...    yield "step:1"
...    yield "step:2"
...    yield "step:3"

>>\> mf = myfun()

>>\> next(mf)
'step:1'
>>\> next(mf)
'step:2'
>>\> next(mf)
'step:3'

##### 用“列表生成式”生成一个生成器

g = (x**2 for x in range(10))

for x in g:
   print(x)

0
1
4
9
16
25
36
49
64
81