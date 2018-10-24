---
title: Python-闭包与装饰器
url: 213.html
id: 213
categories:
  - python
  - 学习笔记
date: 2018-02-10 15:45:14
tags:
---

### 闭包

讲闭包前先说一下在函数内部，返回一个函数

def lazy_sum(*args):
    def sum():
        ax = 0
        for n in args:
            ax = ax + n
        return ax
    return sum

f = lazy_sum(1,3,5,7,9)

利用返回的函数，可以在后期使用

f()
>>>25

这里我们可以看到，lazy_sum中传入的值，可以在sum()函数中，使用，并且不用再次赋值，返回sum函数时，那些变量都暂时保存在返回的函数中，这被称为**闭包**

*   每次调用都会返回一个新的函数
*   返回的函数并没有立刻执行，而是直到调用了f()才执行

* * *

#### 关于闭包的重新解读

由于最近在读fluent python这本书，里面也有提到闭包，其实之前对闭包的概念也有点模糊，今天重新解读了下。 先看两段代码示例 ![](http://blog.kingkk.com/wp-content/uploads/2018/03/b68ebeff95290ece093e4496cb8a208b.png)![](http://blog.kingkk.com/wp-content/uploads/2018/03/6ba7a95e7fac5d996820e3673ccd354c.png) 这里两段代码的区别就在于t这个集合的位置不同 当t放在B函数之中时（示例1），两次调用c都会初始化t这个集合，重新开始添加元素 然而当t在函数B之外，与B一同在A函数下时（示例2），第二次调用t这个集合就会在之前第一次调用的基础上，进行添加 **因为在后期的执行c函数时，是不断执行A()所返回的B函数，A不会再次执行，而B会在每次调用时反复执行。** B函数的闭包延伸到了B函数的作用域之外，使得**仍能反复使用一些曾经操作过的变量（如t）**

### ![](http://blog.kingkk.com/wp-content/uploads/2018/03/42373f70cbbc15ec35a98182d25f8d3e.png)

### 装饰器

装饰器的本意是为了不改变原有函数，并在其基础上增添额外功能 例如：

def now():
    import time
    print(time.ctime())

Sat Feb 10 15:03:48 2018

在装上某个我们自定义装饰器之后

@log
def now():
    import time
    print(time.ctime())

func's name is now():
Sat Feb 10 15:06:38 2018

如下为之前事例中log函数的定义

def log(func):
    def wrapper(\*args, \*\*kw):
        print('func\\'s name is %s():' % func.\_\_name\_\_)
        return func(\*args, \*\*kw)
    return wrapper

通过闭包，返回一个函数，并在该函数之前添加我们所想要执行的代码 在函数前加上**@log**，等价于执行了 ** now = log(now)**   若想要增加的功能需要输入变量，则需要三层嵌套的闭包

def log(myname):
	def wrapper1(func):
	    def wrapper2(\*args, \*\*kw):
	        print('\[call %s\]func\\'s name is %s():' % (myname,func.\_\_name\_\_))
	        return func(\*args, \*\*kw)
	    return wrapper2
	return wrapper1

@log(log.\_\_name\_\_)
def now():
    import time
    print(time.ctime())

now()

\[call log\]func's name is now():
Sat Feb 10 15:19:08 2018

添加**@log(log.\_\_name\_\_)**等价于执行了**now =log(log.\_\_name\_\_).(now)**   需要注意的是log函数最后返回的其实是wrapper2，所以一些自身的属性，如\_\_name\_\_等也变成wrapper2了，所以增加如下语句

**import functools**
def log(func):
    **@functools.wraps(func)
**    def wrapper(\*args, \*\*kw):
        print('func\\'s name is %s():' % func.\_\_name\_\_)
        return func(\*args, \*\*kw)
    return wrapper

**import functools**
def log(myname):
	def wrapper1(func):
		**@functools.wraps(func)** 		def wrapper2(\*args, \*\*kw):
			print('\[call %s\]func\\'s name is %s():' % (myname,func.\_\_name\_\_))
			return func(\*args, \*\*kw)
		return wrapper2
	return wrapper1