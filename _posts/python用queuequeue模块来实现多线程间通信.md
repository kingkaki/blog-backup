---
title: Python 用Queue/queue模块来实现多线程间通信
url: 98.html
id: 98
categories:
  - python
  - 学习笔记
date: 2017-12-09 18:00:57
tags:
---

使用Queue模块（2.x版本，3.x版本中重命名为queue），通过创建一个队列来提供线程间通信的机制，从而让线程之间可以互相分享数据。 先列举一下Queue模块中的一些属性

##### Queue/queue模块的类

*   `Queue(maxsize=0)` 创建一个先入先出队列。若指定maxsize，则在队列满时阻塞；否则则为无限队列
*   `LifoQueue(maxsize=0)` 创建一个后入先出队列（个人感觉更应该叫堆栈>_<）。若制定maxsize，则在队列满时阻塞，否则为无限队列
*   `PriorityQueue(maxsize=0)` 创建一个优先级队列。若指定maxsize，则在队列满时阻塞；否则则为无限队列

##### Queue/queue异常

*   `Empty` 当空队列调用get()方法时抛出异常
*   `Full` 当已满队列调用put()方法时抛出异常

##### Queue/queue对象方法

*   `qsize()` 返回队列大小
*   `empty()` 队列为空时返回True，不为空返回False
*   `full()` 队列已满，返回True，否则返回False
*   `put(item,block=True,timeout=None)` 将item放入队列
*   `put_nowait(item)` 和put(item,False)相同
*   `get(block=True,timeout=None)` 从队列中取得元素
*   `get_nowait()` 和get(False)相同
*   `task_done()` 用于表示队列中的某个元素已经执行完成，该方法会被join()使用
*   `join()` 在队列中所有元素执行完毕并调用上面的task_done()之前，保持阻塞

顺便提一下**阻塞（block）**这个概念（毕竟自己之前一直不知道阻塞什么意思，就看书看的也一脸懵） 通俗的说就是等待，比如该资源被另一个线程所使用，当该线程想使用该线程时就会处于一种阻塞状态，就是等待别人使用完该资源，或者是说等待有资源可以被自己使用之后再来使用该资源。 当block参数为False，就处于不阻塞的状态，此时，当资源被别的线程占据时，该线程就不会再去等待别人使用完线程，而是选择直接抛出一个异常。 put与get中的block就是设定该命令在资源无法使用时是否进行阻塞等待，timeout则是指定等待的时间. 如下是一个示例：

#encoding=utf8
from random import randint
from time import sleep,ctime
from Queue import Queue
import threading


class MyThread(threading.Thread):    #之前定义过了的一个MyThread类
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

def writeQ(queue):
	print 'producing object form Q...',
	queue.put('xxx',1),
	print 'size now',queue.qsize()

def readQ(queue):
	val = queue.get(1)
	print 'consumed object from Q ... size now ',queue.qsize()

def writer(queue,loops):
	for i in range(loops):
		writeQ(queue)
		sleep(randint(1,3))

def reader(queue,loops):
	for i in range(loops):
		readQ(queue)
		sleep(randint(2,5))

funcs = \[writer,reader\]

def main():
	nloops = randint(2,5)
	q = Queue(32)
	threads = \[\]
	for i in range(len(funcs)):
		t = MyThread(funcs\[i\],(q,nloops),funcs\[i\].\_\_name\_\_)
		threads.append(t)

	for i in range(len(funcs)):
		threads\[i\].start()

	for i in range(len(funcs)):
		threads\[i\].join()

	print 'ALL DONE'

if \_\_name\_\_ == '\_\_main\_\_':
	main()

简单解释下上面代码，对writer与reader随机创建2-4次循环，writer负责往队列中放入元素，reader负责往对列中取出元素。write随机sleep1-3秒，reader随机sleep2-5秒。（尽可能使放入比取出操作更快） 运行结果如下

starting writer at: Sat Dec 09 17:48:56 2017
producing object form Q... size now 1
starting reader at: Sat Dec 09 17:48:56 2017
consumed object from Q ... size now 0
producing object form Q... size now 1
consumed object from Q ... size now 0
producing object form Q... size now 1
producing object form Q... size now 2
consumed object from Q ... size now 1
producing object form Q... size now 2
writer finished at: Sat Dec 09 17:49:05 2017
consumed object from Q ... size now 1
consumed object from Q ... size now 0
reader finished at: Sat Dec 09 17:49:16 2017
ALL DONE

若使取出操作比放入更快时，仅需修改随机sleep的部分，输出如下

starting writer at: Sat Dec 09 17:57:04 2017
producing object form Q... size now 1
starting reader at: Sat Dec 09 17:57:04 2017
consumed object from Q ... size now 0
producing object form Q... size now 1
consumed object from Q ... size now 0
producing object form Q... size now 1
consumed object from Q ... size now 0
producing object form Q... size now 1
consumed object from Q ... size now 0
writerreader finished at:finished at: Sat Dec 09 17:57:17 2017Sat Dec 09 17:57:17 2017

**当没有资源可以消耗时，reader会处于阻塞状态，每当write填充一个资源时，reader会立即随之执行**