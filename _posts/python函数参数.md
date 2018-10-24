---
title: Python-函数参数
url: 209.html
id: 209
categories:
  - python
  - 学习笔记
date: 2018-02-09 19:02:31
tags:
---

### 前言

之前python学的有些零散与碎片化，近期就打算系统的开始学下Python，今天就去了解了下之前一直搞不太懂的可变参数，与\*arg,\*\*kw这些，今天就记录下  

### 正文

几种Python中的传参方式

*   **位置参数**
*   **默认参数**
*   **可变参数**
*   **关键字参数**

 

##### 位置参数

就是最普通的一种传参方式：传入有且唯一的参数

def mysum(x,y):
    return x+y

mysum(1,2)
>>>3

 

##### 默认参数

给指定的参数，传入默认值  (默认参数放在位置参数后面)

def mysum(x,y=3):
    return x+y

mysum(1,2)
>>>3
mysum(1)
>>>4

 

##### 可变参数

给出任意数量的参数，以**tuple**的方式传入

def mysum(x,*arg):
    if arg:
        for i in arg:
            x=x+i
    return x

mysum(1,2,3)
>>>6
mysum(1,2,3,4)
>>>10

可以在set、tuple前加上星号（*）将其转为可变参数传入

a=(1,2,3,4)
b=set(\[1,2,3,4,5\])

mysum(1,*a)
>>>11
mysum(1,*b)
>>>16

 

##### **关键字参数**

给出任意数量的键值对，以**dict**的方式传入

def show(name,**kw):
    print('my name : {},other : {}'.format(name,kw))

show('kingkk',city='zhejiang',job='student')
>>>my name : kingkk,other : {'city': 'zhejiang', 'job': 'student'}

也可以将一个dict，在变量前加上两个星号（**）将其转变为关键字参数传入

other = {'city':'zhejiang','job':'student'}

show('kingkk',**other)
>>>my name : kingkk,other : {'city': 'zhejiang', 'job': 'student'}

 

##### 受限关键字参数

只接受指定的关键字参数（*之后的参数为关键字参数）

def show(name,*,city,job):
   print('my name : {},city : {},job :{}'.format(name,city,job))

show('kingkk',city='zhejiang',job='student')
>>>my name : kingkk,city : zhejiang,job :student

只传入一个参数时，会抛出异常

show('kingkk',city='zhejiang')
>>>Traceback (most recent call last):
 File "D:\\Code\\test.py", line 12, in <module>
 show('kingkk',city='zhejiang')
TypeError: show() missing 1 required keyword-only argument: 'job'

 

* * *

常用的组合形式

def show(name,\*args,\*\*kw):
    print('my name : {},args : {} ,kw : {}'.format(name,args,kw))

show('kingkk',1,2,3,a='A',b='B',c='C')
>>>my name : kingkk,args : (1, 2, 3) ,kw : {'a': 'A', 'b': 'B', 'c': 'C'}