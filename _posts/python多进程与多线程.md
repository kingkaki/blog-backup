---
title: Python-多进程与多线程
url: 216.html
id: 216
categories:
  - python
  - 学习笔记
date: 2018-02-11 21:05:45
tags:
---

### 前言

对于一些扫描器、爬虫之类的，要实现高并发是很重要的，这就涉及到多进程与多线程了。 不得不说之前写的那篇多进程又臭又长，自己都懒得看，重写一遍，一切从简  

### 正文

##### 多进程

利用多核cpu的优势，充分利用每个核心的功能 使用方法（默认开启的进程数为）

from multiprocessing import Pool
p = Pool()
for i in task:
    p.apply_async(func-i,args=(…,…))  #加入进程
p.close() #关闭进程池才可以开始运行进程
p.join()  #等待进程运行完毕

##### 多进程

一个进程下分配多线程执行任务，适合I/O密集型操作 使用方法

import threading
t = threading.Thread(target=func, args=(…,))  #创建线程
t.start()   #启动进程
t.join()    #等待进程结束 

### 示例

如下是写的几个爬取wooyun镜像标题的脚本，分别用了几种方式实现 **初始脚本**

#encoding=utf8
import requests,re,time

url = 'http://localhost/wooyun/bugs.php'
header = {'User-Agent':'Mozilla/5.0(compatible;MSIE9.0;WindowsNT6.1;Trident/5.0'}
all_page = 295

def search_wybug():
	start_time = time.time()
	for i in range(1,all_page):
		with open('d:/bugs.txt','a+') as f:
			r = requests.get(url, params = {'page':i},headers=header)
			r.encoding = 'utf8'
			bugs = re.findall(r'(.*?)',r.text)
			for bug in bugs:
				f.write(bug.rstrip()+'\\n')
		print('done %d page' % i)
	end_time = time.time()
	print('All consume %0.2f second' % (end\_time - start\_time))

if \_\_name\_\_ == '\_\_main\_\_':
	search_wybug()

运行结果：耗时139.7s ![](http://blog.kingkk.com/wp-content/uploads/2018/02/search.png)   **多线程**

#encoding=utf8
import os ,time,requests ,re,threading

url = 'http://localhost/wooyun/bugs.php'
header = {'User-Agent':'Mozilla/5.0(compatible;MSIE9.0;WindowsNT6.1;Trident/5.0'}
thread_list = \[\]
thread_num = 4
all_page = 295

def search_wybug(start ,end):
	start_time = time.time()
	print('task thread (%s)...' % (threading.current_thread()))
	for i in range(start, end):
		with open('d:/bugs.txt','a+') as f:
			r = requests.get(url, params = {'page':i},headers=header)
			r.encoding = 'utf8'
			bugs = re.findall(r'(.*?)',r.text)
			for bug in bugs:
				f.write(bug.rstrip()+'\\n')
		print('done %d page' % i)
	end_time = time.time()
	print('Task %s runs %0.2f seconds.' % (threading.current\_thread(), (end\_time - start_time)))

def main():
	start_time = time.time()
	for i in range(thread_num):
		start = i*(all\_page//thread\_num)
		end = (i+1)*((all\_page//thread\_num)-1)
		t = threading.Thread(target=search_wybug, args=(start,end))
		thread_list.append(t)
	for t in thread_list:
		t.start()
	for t in thread_list:
		t.join()
	end_time = time.time()
	print("all consume %0.2f seconds" % (end\_time - start\_time))

if \_\_name\_\_ == '\_\_main\_\_':
	main()

运行结果：耗时57.84s![](http://blog.kingkk.com/wp-content/uploads/2018/02/thread.png)   **多进程**

#encoding=utf8
from  multiprocessing import Pool
import os ,time,requests ,re 

url = 'http://localhost/wooyun/bugs.php'
header = {'User-Agent':'Mozilla/5.0(compatible;MSIE9.0;WindowsNT6.1;Trident/5.0'}
core_num = 4
all_page = 295

def search_wybug(start ,end):
	start_time = time.time()
	print('task pid (%s)...' % (os.getpid()))
	for i in range(start, end):
		with open('d:/bugs.txt','a+') as f:
			r = requests.get(url, params = {'page':i},headers=header)
			r.encoding = 'utf8'
			bugs = re.findall(r'[(.*?)](bug_detail.*?)',r.text)
			for bug in bugs:
				f.write(bug.rstrip()+'\\n')
		print('done %d page' % i)
	end_time = time.time()
	print('Task %s runs %0.2f seconds.' % (os.getpid(), (end\_time - start\_time)))



def main():
	print('parent process %s' % os.getpid())
	start_time = time.time()
	p = Pool()
	for i in range(core_num):
		start = i*(all\_page//core\_num)
		end = (i+1)*((all\_page//core\_num)-1)
		p.apply\_async(search\_wybug,args=(start,end))
	print('waiting for all subprocesses done……')
	p.close()
	p.join()
	end_time = time.time()
	print('All consume %0.2f second' % (end\_time - start\_time))


if \_\_name\_\_ == '\_\_main\_\_':
	main()

运行结果：耗时53.52s ![](http://blog.kingkk.com/wp-content/uploads/2018/02/process.png)   **多线程+多进程**

#encoding=utf8
from  multiprocessing import Pool
import os ,time,requests ,re,threading

url = 'http://localhost/wooyun/bugs.php'
header = {'User-Agent':'Mozilla/5.0(compatible;MSIE9.0;WindowsNT6.1;Trident/5.0'}
thread_list = \[\]
core_num = 4
thread_num = 5
all_page = 295

def search_wybug(start ,end):
	start_time = time.time()
	print('task thread (%s)...' % (threading.current_thread()))
	for i in range(start, end):
		with open('d:/bugs.txt','a+') as f:
			r = requests.get(url, params = {'page':i},headers=header)
			r.encoding = 'utf8'
			bugs = re.findall(r'(.*?)',r.text)
			for bug in bugs:
				f.write(bug.rstrip()+'\\n')
		print('done %d page' % i)
	end_time = time.time()
	print('Task %s runs %0.2f seconds.' % (threading.current\_thread(), (end\_time - start_time)))

def sw_thread(start ,end):
	print('task pid (%s)...' % (os.getpid()))
	for i in range(thread_num):
		thread\_start = start+i*((end-start)//thread\_num)
		thread\_end = start+(i+1)*((end-start)//thread\_num)
		t = threading.Thread(target=search\_wybug, args=(thread\_start,thread_end))
		thread_list.append(t)
	for t in thread_list:
		t.start()
	for t in thread_list:
		t.join()

def main():
	print('parent process %s' % os.getpid())
	start_time = time.time()
	p = Pool()
	for i in range(core_num):
		start = i*(all\_page//core\_num)
		end = (i+1)*((all\_page//core\_num)-1)
		p.apply\_async(sw\_thread,args=(start,end))
	print('waiting for all subprocesses done……')
	p.close()
	p.join()
	end_time = time.time()
	print('All consume %0.2f second' % (end\_time - start\_time))


if \_\_name\_\_ == '\_\_main\_\_':
	main()

运行结果：耗时105.78s ![](http://blog.kingkk.com/wp-content/uploads/2018/02/mixed.png)  

### 总结：

单独使用多进程、多线程明显提升了程序运行的速度 然而使用多线程+多进程反而增加了cpu的开销，使得运行速度不如单独执行的快 **多进程**可以更好的利用cpu的多核，而且程序稳定性强，一个进程出错，其余进程可以继续运行 **多线程**更适合I/O密集型的操作，但是当一个线程出错时，操作系统就会强制停止整个进程，从而影响其他线程    

* * *

  补充一下，之前写的脚本有个小bug，原因是在取每个线程开始和结束范围的地方，当线程数或者进程数大了之后，就会漏掉几页，改进了下，start和end的取法如下

 start = i*round(port\_num/thread\_num)
 end = (i+1)*(round(port\_num/thread\_num))

测试过了，可以取到所有范围内的值