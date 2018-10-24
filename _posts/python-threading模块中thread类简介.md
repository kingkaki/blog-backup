---
title: Python threading模块中Thread类简介
url: 91.html
id: 91
categories:
  - python
  - 学习笔记
date: 2017-12-07 20:41:34
tags:
---

### Thread类

threading中的Thread类是主要的执行对象。有如下的对象属性以及方法。

###### Thread 对象数据属性

*   `name`  线程名
*   `ident`  线程的标识符
*   `daemon`  bool标识，表示该线程是否为守护线程

###### Thread 对象方法

*   `__init__(self, group=None, target=None, name=None, args=(), kwargs=None, verbose=None)`    实例化一个线程对象，主要参数为，target一个函数式，以及args 函数的参数
*   `start()`    开始执行该线程
*   `run()`    定义线程的功能（主要用于在子类中重载）
*   `join(timeout=None)`    直至启动的线程终止前一直挂起，除非给出了timeout（秒），否则将会一直堵塞

##### 三种使用Thread类创建线程的方式

（常用的为第三种，其次为第一种，第二种方案比较少见）

*   创建Thread实例，传给它一个函数
*   创建Thread实例，传给它一个可调用的类实例
*   创建Thread的子类，并创建子类的实例

 

###### 创建Thread的实例，传给它一个函数

#encoding=utf8
import threading
from time import sleep, ctime

loops = \[4,2\]

def loop(nloop,nsec):
 print "start loop",nloop,'at:',ctime()
 sleep(nsec)
 print 'loop',nloop,'done at:',ctime()

def main():
 print 'starting at:',ctime()
 threads = \[\]       #线程列表
 nloops = range(len(loops))

 for i in nloops:
  t = **threading.Thread(target=loop,args=(i,loops\[i\]))**  #创建线程
  threads.append(t)  #添加至线程列表

 for i in nloops:    #启动线程列表
  threads\[i\].start()

 for i in nloops:   #等待线程列表结束 
  threads\[i\].join()

 print 'all DONE at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
 main()

运行结果如下：

starting at: Thu Dec 07 19:38:24 2017
start loop 0 at: Thu Dec 07 19:38:24 2017
start loop 1 at: Thu Dec 07 19:38:24 2017
loop 1 done at: Thu Dec 07 19:38:26 2017
loop 0 done at: Thu Dec 07 19:38:28 2017
all DONE at: Thu Dec 07 19:38:28 2017。

**tips：**与thread.start\_new\_thread()一个主要区别就是，实例化Thread类时不会立即执行该线程，而是通过start()方法让他们开始执行。join()方法将等待线程结束。  

###### 创建Thread的实例，传给它一个可调用的类实例

仅需在之前的代码上稍作修改

#encoding=utf8
import threading
from time import sleep, ctime

loops = \[4,2\]

**class ThreadFunc(object):** 	def \_\_init\_\_(self,func,args,name=''):
		self.name = name
		self.func = func
		self.args = args

	**def \_\_call\_\_(self):**  #重载\_\_call\_\_方法
		self.func(*self.args)

def loop(nloop,nsec):
	print "start loop",nloop,'at:',ctime()
	sleep(nsec)
	print 'loop',nloop,'done at:',ctime()

def main():
	print 'starting at:',ctime()
	threads = \[\]     #线程列表
	nloops = range(len(loops))

	for i in nloops:
		t = **threading.Thread(target=ThreadFunc(loop,(i,loops\[i\]),loop.\_\_name\_\_))**  #创建线程
		threads.append(t)       #添加至线程列表

	for i in nloops:		#启动线程列表
		threads\[i\].start()

	for i in nloops:		#等待线程列表结束	
		threads\[i\].join()

	print 'all DONE at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
	main()

三次的运行结果都是相同的就不放上来了。 这种创建Thread类的主要方法就是重载类的\_\_call\_\_方法，然后将其入Thread的target参数中  

##### 派生Thread的子类，并创建子类的实例

也仅需在代码一上稍作修改

#encoding=utf8
import threading
from time import sleep, ctime

loops = \[4,2\]

**class MyThread(threading.Thread):** 	def \_\_init\_\_(self,func,args,name=''):
		threading.Thread.\_\_init\_\_(self) #调用基类构造函数
		self.name = name
		self.func = func
		self.args = args

	**def run(self):**  #重载Thread中run方法
		self.func(*self.args)

def loop(nloop,nsec):
	print "start loop",nloop,'at:',ctime()
	sleep(nsec)
	print 'loop',nloop,'done at:',ctime()

def main():
	print 'starting at:',ctime()
	threads = \[\]     #线程列表
	nloops = range(len(loops))

	for i in nloops:
		t = **MyThread(loop,(i,loops\[i\]),loop.\_\_name\_\_)**  #创建线程
		threads.append(t)           #添加至线程列表

	for i in nloops:		    #启动线程列表
		threads\[i\].start()

	for i in nloops:		    #等待线程列表结束	
		threads\[i\].join()

	print 'all DONE at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
	main()

此次实例化的主要方法是通过继承Thread类,并重载了它的run()方法。 在实例化时就可以直接通过MyThread实例化一个Thread对象。  

##### 一个简单的单线程与多线程执行应用与对比

#encoding=utf8
import threading
from time import sleep, ctime

class MyThread(threading.Thread):    #定义MyThread类
	def \_\_init\_\_(self,func,args,name=''):
		threading.Thread.\_\_init\_\_(self)
		self.name = name
		self.func = func
		self.args = args

	def getResult(self):
		return self.res 

	def run(self):
		print "starting",self.name,"at:",ctime()
		self.res = self.func(*self.args)
		print self.name,"finished at:",ctime()

def fib(x):
	sleep(0.005)
	if x<2:return 1
	return(fib(x-2)+fib(x-1))

def fac(x):
	sleep(0.1)
	if x<2:return 1
	return (x*fac(x-1))

def sum_(x):
	sleep(0.1)
	if x<2 :return 1
	return (x+sum_(x-1))

funcs = \[fib,fac,sum_\]     #函数式列表
n=12

def main():
	nfuncs = range(len(funcs))

	print u"*****单线程*****"  #单线程执行
	for i in nfuncs:
		print 'starting',funcs\[i\].\_\_name\_\_,'at:',ctime()
		print funcs\[i\](n)
		print funcs\[i\].\_\_name\_\_,'finished at:',ctime()

	print u'\\n*****多线程*****'  #多线程执行
	threads = \[\]
	for i in nfuncs:
		t = MyThread(funcs\[i\],(n,),funcs\[i\].\_\_name\_\_)
		threads.append(t)

	for  i in nfuncs:
		threads\[i\].start()

	for i in nfuncs:
		threads\[i\].join()
		print threads\[i\].getResult()
	print "all DONE"

if \_\_name\_\_ == '\_\_main\_\_':
	main()

执行结果如下图：

*****单线程*****
starting fib at: Thu Dec 07 20:32:23 2017
233
fib finished at: Thu Dec 07 20:32:26 2017
starting fac at: Thu Dec 07 20:32:26 2017
479001600
fac finished at: Thu Dec 07 20:32:27 2017
starting sum_ at: Thu Dec 07 20:32:27 2017
78
sum_ finished at: Thu Dec 07 20:32:28 2017

*****多线程*****
starting fib at: Thu Dec 07 20:32:28 2017
starting fac at: Thu Dec 07 20:32:28 2017
starting sum_ at: Thu Dec 07 20:32:28 2017
sum_ finished at: Thu Dec 07 20:32:30 2017
fac finished at: Thu Dec 07 20:32:30 2017
fib finished at: Thu Dec 07 20:32:31 2017
233
479001600
78
all DONE

可以明显看出多线程的执行速度远胜于单线程。 这里用sleep函数只是为了进行演示，想必也没有谁会在函数执行时增添sleep函数来拖慢运行时间。 虽然python的多线程可能对与计算密集型的应用没有什么太大的提升，此处仅用sleep函数来伪装成一个I/O密集型来测试python的多线程与单线程的比较。