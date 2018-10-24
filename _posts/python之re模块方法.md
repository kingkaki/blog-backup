---
title: Python之re模块方法
url: 63.html
id: 63
categories:
  - python
  - 学习笔记
date: 2017-12-04 21:51:30
tags:
---

#### match()  `match(pattern, string, flags=0)`

match对字符串进行正则匹配，匹配成功返回一个匹配对象，匹配失败，返回None 匹配成功时：

In \[14\]: m = re.match('foo','foo')
In \[15\]: m
 Out\[15\]: <\_sre.SRE\_Match at 0x6bd8870>

In \[16\]: if m is not None:print m.group()
 foo

匹配失败：

In \[5\]: m = re.match('foo','seafoo')

In \[7\]: m is None
Out\[7\]: True

tips:若匹配失败再使用group等方法，会抛异常，所以使用前最好加上 is not None判断  

#### search()  `search(pattern, string, flags=0)`

对给定的正则表达式搜索第一次出现的匹配情况。如果搜索成功，返回一个匹配对象；否则，返回None match失败，search成功:

In \[12\]: m = re.search('foo','seafoo')

In \[15\]: if m is not None:print m.group():
foo

search失败:

In \[16\]: m = re.search('foo','sea')

In \[17\]: m is None
Out\[17\]: True

 

#### findall() finditer()  `findall(pattern, string, flags=0)`

返回值为一个列表。匹配失败返回一个空列表，匹配成功返回成功匹配部分的列表（顺序为出现的顺序） 注意存在分组时返回的为一个元组列表

In \[101\]: re.findall('car','catt')  #返回空列表
Out\[101\]: \[\]

In \[102\]: re.findall('car','scary') 
Out\[102\]: \['car'\]

In \[103\]: re.findall('car','carry the barcardi to the car')
Out\[103\]: \['car', 'car', 'car'\]

In \[105\]: re.findall(r'(th\\w+) and (th\\w+)','this and that,the and those')   #**对于存在分组的返回元组列表**
Out\[105\]: \[('this', 'that'), ('the', 'those')\]

finditer()类似于findall()，只是返回的是一个迭代器

In \[108\]: \[i.group(1) for i in re.finditer(r'(th\\w+) and (th\\w+)','this and that,the and those')\]
Out\[108\]: \['this', 'the'\]

   

#### sub() subn()替换   `sub(pattern, repl, string, count=0, flags=0)`

sub()和subn()功能一样。但sub返回替换后的字符串，subn返回替换后的字符串与替换总数作为元组返回

In \[111\]: re.sub('\[ae\]','X','abcdef')
Out\[111\]: 'XbcdXf'

In \[112\]: re.subn('\[ae\]','X','abcdef')
Out\[112\]: ('XbcdXf', 2)

还可以使用\\N编号的方式进行更多替换

In \[119\]: re.sub(r'(\\d{1,2})/(\\d{1,2})/(\\d{2}|\\d{4})',
 ...: r'\\2/\\1/\\3', '2/20/91')     #将美式日期MM/DD/YY{,YY}转换为常用的DD/MM/YY{,YY}
Out\[119\]: '20/2/91'

 

#### split()`split(pattern, string, maxsplit=0, flags=0)`

以正则模式分割字符串，并以列表返回

In \[121\]: re.split(':','str1;str2;str3')  
Out\[121\]: \['str1;str2;str3'\]