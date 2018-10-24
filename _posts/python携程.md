---
title: Python-携程
url: 227.html
id: 227
categories:
  - python
  - 学习笔记
date: 2018-02-13 14:55:18
tags:
---

### 简介

通俗的讲，就是在几个函数之间切换 在某个函数阻塞时，切换到另一个等待运行的函数之中。阻塞停止时，再从之前阻塞的地方继续执行。以此提升cpu的利用率 携程是在同一个线程之下进行的，不需要开辟新的线程，减少了线程直接切换的cpu消耗  

### 正文

python中完成携程是通过生成器来实现的，顺带提一下几个函数的作用

*   `**generator.send(args)**`   使yield x拥有一个返回值args
*   `**yield from generator**`   类似于一个装饰器,在原有生成器基础上，增添一些头尾功能，返回新的生成器

 

###### pyton3.4引入asyncio标准库，提供对异步I/O的支持

import asyncio

@asyncio.coroutine  #将一个生成器标记为协同类型
def hello():
    print('Hello world!')
    yield from asyncio.sleep(1)    #调用异步的asyncio.sleep(1)
    print('Hello again!')

loop = asyncio.get\_event\_loop()    #生成event_loop
tasks = \[hello(), hello()\]
loop.run\_until\_complete(asyncio.wait(tasks))    #将携程放入event_loop执行
loop.close()

###### ![](http://blog.kingkk.com/wp-content/uploads/2018/02/7e8c9b95fecd62cfef6b6ef28d2ffb4a.png)

###### python3.5新增的async/await对携程进行了优化

import asyncio

**async** def hello():
    print("Hello world!")
    r = **await** asyncio.sleep(1)
    print("Hello again!")

loop = asyncio.get\_event\_loop()
tasks = \[hello(), hello()\]
loop.run\_until\_complete(asyncio.wait(tasks))
loop.close()

相当于async替换了@asyncio.coroutine， await替换了yield from    

### 总结

我后来尝试了下，自定义个一个生成器，然后加载await后面，但是并不能运行成功 具体原因好像是因为自定义的生成器并不是异步的，假如真的要自己写异步那工作量就实在有点大了 然后python提供的支持异步的操作除了asyncio下的一些函数，还有aiohttp库所提供的函数 不过不能自定义确实很蛋疼，而且提供的函数并没有很全面，导致感觉甚至这个携程有些鸡肋 总感觉python的携程也在发展当中，毕竟3.4才内置的异步I/O支持，希望未来能有些好用的库吧，携程就当了解了解了