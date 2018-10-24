---
title: Python之thread模块简介
url: 83.html
id: 83
categories:
  - python
  - 学习笔记
date: 2017-12-06 21:10:16
tags:
---

由于thread模块的局限性，以及如今以及普遍使用功能更加强大的threading模块替代thread模块进行多线程操作，所以就在此对thread模块进行简单的介绍，作为学习的记录。

#### thread 模块的函数

*   `start_new_thread(function, args[, kwargs])` 派生一个新的线程，并使用元组传递指定的参数来执行function函数
*   `allocate_lock()` 分配LockkType锁对象
*   `exit()` 给线程退出指令

#### LockType锁对象的方法

*   `acquire(wait=None)` 尝试获取锁对象，通俗的讲法为“将锁锁上”
*   `locked()` 如果获取了锁对象则返回True,否则，返回False
*   `release()` 释放锁

##### 使用thread模块，进行一次简单的多线程实例

#encoding=utf8
import thread
from time import sleep, ctime

def loop0():
 print u"loop0开始时间:",ctime()
 sleep(3)
 print u"loop0结束时间:",ctime()

def loop1():
 print u"loop1开始时间:",ctime()
 sleep(2)
 print u"loop1结束时间:",ctime()

def main():
 print u"开始时间：",ctime()
 thread.start\_new\_thread(loop0,()) #创建loop0线程
 thread.start\_new\_thread(loop1,()) #创建loop1线程
 sleep(5)                          #确保在主线程退出前子线程已经退出
 print u"结束时间：",ctime()

if \_\_name\_\_ == '\_\_main\_\_':
 main()

运行后的输出结果为：

开始时间： Wed Dec 06 20:12:41 2017
loop0开始时间:loop1开始时间: Wed Dec 06 20:12:41 2017Wed Dec 06 20:12:41 2017

loop1结束时间: Wed Dec 06 20:12:43 2017
loop0结束时间: Wed Dec 06 20:12:44 2017
结束时间： Wed Dec 06 20:12:46 2017

甚至可以很明显的发现loop0与loop1是并发的，甚至由于获取时间耗费的一点时间，两次打印都出现在了同一排。 并且loop1虽然在loop0之前运行，但是却提前loop0一秒结束。  

##### 通过使用LockType锁对象，更智能的将程序挂起。在所有线程全部完成执行后立即退出。

#encoding=utf8
import thread
from time import sleep, ctime

loops = \[3,2\]                          #记录sleep的时间列表

def loop(nloop,nsec,lock):             #从左至右依此为 循环编号，sleep时间，锁
 print 'start loop',nloop,'at:',ctime()
 sleep(nsec)
 print 'loop',nloop,'done at:',ctime()
 lock.release()

def main():
 print 'starting at:',ctime()
 locks = \[\] #创建锁列表
 nloops = range(len(loops))            #获取线程个数

for i in nloops:
 lock = thread.allocate_lock()         #分配锁对象
 lock.acquire()                        #上锁
 locks.append(lock) #添加到锁列表

for i in nloops:                       #循环创建线程
 thread.start\_new\_thread(loop,(i,loops\[i\],locks\[i\]))

for i in nloops:                       #检查每个锁是否被释放,即该线程是否结束
 while locks\[i\].locked():pass

print 'all done at:',ctime()

if \_\_name\_\_ == '\_\_main\_\_':
 main()

运行结果：

starting at: Wed Dec 06 21:00:58 2017
start loopstart loop 01 at:at: Wed Dec 06 21:00:58 2017Wed Dec 06 21:00:58 2017

loop 1 done at: Wed Dec 06 21:01:00 2017
loop 0 done at: Wed Dec 06 21:01:01 2017
all done at: Wed Dec 06 21:01:01 2017

依旧loop0与loop1并发，并且在最长的loop0（3秒）执行完毕后，主线程也退出了。