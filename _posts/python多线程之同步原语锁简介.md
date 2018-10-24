---
title: Python 多线程之同步原语--锁与信号量简介
url: 93.html
id: 93
categories:
  - python
  - 学习笔记
date: 2017-12-07 21:23:02
tags:
---

锁
-

先介绍一下锁引入的原因以及锁的主要是为了结尾哪一类的问题。先看如下的试例：

#encoding=utf8
from atexit import register
from random import randrange
from threading import Thread, currentThread
from time import sleep, ctime

class CleanOutputSet(set): #重载了\_\_str\_\_方法的set
	def \_\_str\_\_(self):  
		return ','.join(x for x in self)

loops = (randrange(2,5) for x in xrange(randrange(2,5)))
remaining = CleanOutputSet()  #剩余进程的列表

def loop(nsec):
	myname = currentThread().name
	remaining.add(myname)
	print '\[%s\] started %s '%(ctime(),myname)
	sleep(nsec)
	remaining.remove(myname)
	print '\[%s\] completed %s (%d secs)'%(ctime(),myname,nsec)
	print '(remaining:%s)'%(remaining or 'NONE')  #remaining不为空则打印remaining,否则打印‘NONE’

def main():
	for pause in loops:		#创建并启动所有进程
		Thread(target=loop,args=(pause,)).start() 

@register
def _atexit():
	print 'all DONE at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
	main()

简单介绍一下上面代码，主要目的就是为了随机生成2-4个线程，每个线程随机执行2-4秒，然后每个线程结束后打印剩下的线程。 执行之后的一次结果如下：

\[Thu Dec 07 21:04:47 2017\] started Thread-1 
\[Thu Dec 07 21:04:47 2017\] started Thread-2 
\[Thu Dec 07 21:04:47 2017\] started Thread-3 
\[Thu Dec 07 21:04:49 2017\] completed Thread-1 (2 secs)
(remaining:Thread-3,Thread-2)
\[Thu Dec 07 21:04:51 2017\] completed Thread-3 (4 secs)\[Thu Dec 07 21:04:51 2017\] completed Thread-2 (4 secs)

(remaining:NONE)(remaining:NONE)

all DONE at: Thu Dec 07 21:04:51 2017

可以很明显的发现Thread-3与Thread-2同时结束并打印出了提示语句，并且同时对remaining这个集合进行remove以及打印的操作。 导致了打印语句出现在了同一排，并且两个换行符号同时打印出来，而且所打印的remaining集合都为空  

##### 这就涉及到了一个问题，**两线程在同时修改一个变量**

这时，也就引出了**锁**的概念。**用以防止多个线程同时进入临界区**。 用threading中的Lock函数，对上面的代码进行改进，得到如下代码

#encoding=utf8
from atexit import register
from random import randrange
from threading import Thread, currentThread, Lock
from time import sleep, ctime
lock = Lock()

class CleanOutputSet(set): 
	def \_\_str\_\_(self):  
		return ','.join(x for x in self)

loops = (randrange(2,5) for x in xrange(randrange(2,5)))
remaining = CleanOutputSet()  

def loop(nsec):
	myname = currentThread().name
	**lock.acquire()**
	remaining.add(myname)
	print '\[%s\] started %s '%(ctime(),myname)
	**lock.release()** 	sleep(nsec)
	**lock.acquire()** 	remaining.remove(myname)
	print '\[%s\] completed %s (%d secs)'%(ctime(),myname,nsec)
	print '(remaining:%s)'%(remaining or 'NONE') 
	**lock.release()**

def main():
	for pause in loops:		
		Thread(target=loop,args=(pause,)).start() 

@register
def _atexit():
	print 'all DONE at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
	main()

通过对输入框以及对集合操作的上锁，即使多次运行该程序，也不会出现之前在多线程处理同一块数据的问题

\[Thu Dec 07 21:16:37 2017\] started Thread-1 
\[Thu Dec 07 21:16:37 2017\] started Thread-2 
\[Thu Dec 07 21:16:37 2017\] started Thread-3 
\[Thu Dec 07 21:16:39 2017\] completed Thread-3 (2 secs)
(remaining:Thread-2,Thread-1)
\[Thu Dec 07 21:16:40 2017\] completed Thread-2 (3 secs)
(remaining:Thread-1)
\[Thu Dec 07 21:16:41 2017\] completed Thread-1 (4 secs)
(remaining:NONE)
all DONE at: Thu Dec 07 21:16:41 2017

**tips:** 这里更推荐大家使用python2.5及之后版本推出的**上下文管理器**对锁进行操作。 也只需将

 **lock.acquire()**
 remaining.remove(myname)
 print '\[%s\] completed %s (%d secs)'%(ctime(),myname,nsec)
 print '(remaining:%s)'%(remaining or 'NONE') 
 **lock.release()**

改成

**with lock:**
  remaining.remove(myname)
  print '\[%s\] completed %s (%d secs)'%(ctime(),myname,nsec)
  print '(remaining:%s)'%(remaining or 'NONE')

以减少上锁（acquire）以及解锁（release）的重复操作，也使代码更加易懂    

信号量
---

信号量的产生主要用于多线程之间竞争有限个资源的情况。 相当于一个计数器，记录着实时资源的个数，当资源个数为0时，便无法使用该资源，当资源个数超过最大值时，也无法往其中加载资源。 如下为一个糖果机示例，糖果意味着资源，糖果的购买与填充，相当于资源的使用与返回。购买与填充同时使用着糖果这个资源池。

#encoding=utf8
from atexit import register
from random import randrange
from threading import BoundedSemaphore, Lock, Thread
from time import sleep, ctime

lock = Lock()
MAX = 5
candytray = BoundedSemaphore(MAX)  #设置资源计数器，计数器的值不会超过设定的MAX

def refill():
	with lock:
		print 'Refilling candy...',
		try:
			candytray.release()  #尝试从资源池中释放一个资源
		except ValueError:
			print 'full,skipping'
		else:
			print 'OK'

def buy():
	with lock:
		print 'Buying candy...',
		if candytray.acquire(False): #检测资源是否被消耗完，传入Flase,时调用不阻塞（默认阻塞），可以直接进入else语句
			print 'OK'
		else:
			print 'empty, skipping'

def producer(loops):
	for i in xrange(loops):
		refill()
		sleep(randrange(3))

def consumer(loops):
	for i in xrange(loops):
		buy()
		sleep(randrange(3))


def main():
	print 'starting at:',ctime()
	nloops = randrange(2,6)
	print 'THE CANDY MACHINE (full with {0} bars)!'.format(MAX)
	Thread(target=consumer,args=(randrange(nloops,nloops+MAX+2),)).start() 		
					#创建购买该糖果线程（为了出现糖果为空，购买次数应尽量大于填充次数）
	Thread(target=producer,args=(nloops,)).start()  	#创建填充糖果线程

@register
def _atexit():
	print 'all DONE at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
	main()

其中一次运行结果如下

starting at: Sat Dec 09 16:23:46 2017
THE CANDY MACHINE (full with 5 bars)!
Buying candy... OK
Buying candy... OK
Buying candy... OK
Refilling candy... OK
Buying candy... OK
Buying candy... OK
Refilling candy... OK
Buying candy... OK
Refilling candy... OK
Buying candy... OK
Refilling candy... OK
Refilling candy... OK
all DONE at: Sat Dec 09 16:23:54 2017