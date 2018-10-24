---
title: sqli-labs练习（less23-38）
url: 131.html
id: 131
categories:
  - sql注入
  - 学习笔记
date: 2017-12-17 17:16:48
tags:
---

### less-23

常规的单引号测试，出错 ![](http://www.kingkk.com/wp-content/uploads/2017/12/12c4c1430998a89b64074988c09851fe.png) 然后用#闭合，好像不行。。。编码一下。。好像也不行。。。试一下--+。。怎么还是不行。。。 ![](http://www.kingkk.com/wp-content/uploads/2017/12/21fae53becf20d4641ff9c9aea9a97b0.png) 查看源码之后发现，把注释符都改为空了，而不难发现之前的报错语句都是一样的 ![](http://www.kingkk.com/wp-content/uploads/2017/12/0867143cf0bc40ff04b228806fec6844.png) 于是可以用 or '1'='1闭合 ![](http://www.kingkk.com/wp-content/uploads/2017/12/5bff698b90434565d5fd5f665806b6b4.png) 这个没问题，可是好像获取不到数据。。于是得换一种注释方式（要提前测试字段数） ![](http://www.kingkk.com/wp-content/uploads/2017/12/2544862e6c33d041b37e6695d04cd1be.png) 然后就可以开始提取数据了 ![](http://www.kingkk.com/wp-content/uploads/2017/12/3c72e7aabb25fca70d03c7aedd7472e1.png) 还发现一种注入姿势，好像再多用一个单引号把后面引起来也可以，这样也可以执行 ![](http://www.kingkk.com/wp-content/uploads/2017/12/7687d6f9427c3123f81a01e89cef3342.png) 因为这里后面的语句是limit 0,1 对返回也没什么影响，所以是可以那么做 ![](http://www.kingkk.com/wp-content/uploads/2017/12/babf93d225499f934cfa739cc0e5564d.png)    

### less-24

一个完整的登录、修改、创建用户的网站 还是去审计下代码把。发现过滤还是蛮严格的 ![](http://www.kingkk.com/wp-content/uploads/2017/12/6169ddca1950849227be196c99d3e6bf.png) ![](http://www.kingkk.com/wp-content/uploads/2017/12/d9121aea61808219c5ad1a60b2547aac.png) 最后，pass_change中貌似有一个参数可以利用 ![](http://www.kingkk.com/wp-content/uploads/2017/12/48999e59429a52345ab4989756079135.png) session和cookie不一样，它保存在服务器端，不能直接进行修改，但是没事，可以创建一个这样包含sql注入语句的用户名，先审查下注入语句 ![](http://www.kingkk.com/wp-content/uploads/2017/12/1c05714c2869927cbc0569d150a37ca8.png) 那应该注册一个admin' #这样的用户名，先试试把，登录进入，然后改密码 ![](http://www.kingkk.com/wp-content/uploads/2017/12/be6ecddadab9b0edfcc41aabd6ff1c22.png) 看来思路没错 ![](http://www.kingkk.com/wp-content/uploads/2017/12/4da68fece45b5de116185c8f27b2da80.png)    

### less-25

题目很明显地说明了过滤了or和and，从报错中也能看出来，这样的话，那大不了就不用呗 ![](http://www.kingkk.com/wp-content/uploads/2017/12/f838726354a239642c882b16b8cd42a1.png) 看了下别人的思路。。他们貌似是用or '1'='1绕过的，那也就跟着学一手吧 ![](http://www.kingkk.com/wp-content/uploads/2017/12/63fb028e1525600992c4acf81f4ea20f.png)开启了大小写匹配，那么大小写的话估计是绕不过了 第一种的话是利用算术运算符了

### ![](http://www.kingkk.com/wp-content/uploads/2017/12/c276605d65c79baa2f22df7e12045512.png)

用&&的时候注意下要url编码 ![](http://www.kingkk.com/wp-content/uploads/2017/12/077e1317fe393a44e095bb396dcd93e3.png) 第二种方法就双写oorr,anandd,这种

### ![](http://www.kingkk.com/wp-content/uploads/2017/12/507aaee4ee51e902be3a06a3c5cf844b.png)

### less-25a

和上面一样，就是关了报错，而且没有引号引起来了 不过感觉没有报错，有点不是很清楚改怎么判断是双引号还是单引号还是没有引号这种了倒是      

### less-26

这里就有点酷炫 ![](http://www.kingkk.com/wp-content/uploads/2017/12/7f9475981636fd911b420b046bbe40e2.png) 貌似是过滤了空格，然后试一下常用的替代空格的多行注释/**/貌似也不行。 无奈看了下源码，过滤了蛮多东西的还是 ![](http://www.kingkk.com/wp-content/uploads/2017/12/b7cb0f743394db72a96272a065252f0d.png) 学着大佬写了个脚本尝试了下，什么字节能代替空格，脚本如下

#encoding=utf8
from time import time
import requests 

url = 'http://localhost/sqli-labs-master/Less-26/'

def to_hex(i):
	h = hex(i)\[2:\]
	if len(h)<2:
		h = '0'+h
	h = '%'+h
	return h

def main():
	for i in xrange(0,256):
		url = "http://localhost/sqli-labs-master/Less-26/?id=1'union{0}select{0}1,2,3 '".format(to_hex(i))
		r = requests.get(url)
		if 'Dumb' in r.text:
			print to_hex(i)


if \_\_name\_\_ == '\_\_main\_\_':
	main()

写脚本的时候出现过一个问题，就是不能将id参数通过params的方式传入，貌似会将其进行url编码，导致注入不了，只能直接跟在url的后面 ![](http://www.kingkk.com/wp-content/uploads/2017/12/4f29902ae75ee89de1486602b61f471a.png)跑出来就一个吧，那就用这个代替空格就好了 ![](http://www.kingkk.com/wp-content/uploads/2017/12/00efc8a6db36b1e026225a96c7aa8e6e.png)    

### less-26a

和前面一个类似，就是闭合的方式不一样， 附上payload `?id=1') union%a0select%a01,2,3 || '1'=('1` ![](http://www.kingkk.com/wp-content/uploads/2017/12/92d83288f8df63154f597f5c5bd7be78.png) 这里貌似因为括号的原因，直接再多用一个引号的方式不行 ![](http://www.kingkk.com/wp-content/uploads/2017/12/8eb86cd7358162ca1136bcb8121fbbc0.png) 对比下之前可以的语句

### ![](http://www.kingkk.com/wp-content/uploads/2017/12/0cbdbd8a4ec7447ed33fad0f02b256b5.png)

### less-27

这里说明一下判断过滤什么符号的方式把，就直接把怀疑要过滤的符号放到可以执行的参数之前，如下 ![](http://www.kingkk.com/wp-content/uploads/2017/12/1353a539ea7a30f59e067b193de197b8.png) 返回结果和只输入一个1时一样，就表示被过滤 这里就很惨了union、select都被过滤了，看了下源码 ![](http://www.kingkk.com/wp-content/uploads/2017/12/1725920e2cd1710912df6b3c190052f3.png) 好像没有开启i大小写匹配模式，那就可以大小写混合绕过 ![](http://www.kingkk.com/wp-content/uploads/2017/12/490c6b743dbc9ec43b5b288e60edaea6.png) 不得不说这个%a0还是蛮好用的    

### less-27a

和之前好像差不多，payload: `?id=55"unIOn%a0selEct%a01,2,3 "` ![](http://www.kingkk.com/wp-content/uploads/2017/12/4d0b164dd8c61dfda171f598047359f5.png)    

### less-28

这里遇到一个问题，直接加引号注释的方式，在遇到括号的时候会这个样子 ![](http://www.kingkk.com/wp-content/uploads/2017/12/8cc76416670201996bce0d1ff6085e7a.png) 所以当两个引号能注释成功时，并不能判定有没有括号，果然自己发现的方法还是有些缺陷，用 or '1'='1这种貌似就不会这样 附上上payload： `?id=55')union%a0select%a01,(select%a0password%a0from%a0users%a0limit%a00,1),(3)%a0or '1'=('1` （select与union之间要有空格，or前面也要有空格，反正不知道的话就多加几个好了） ![](http://www.kingkk.com/wp-content/uploads/2017/12/749cd04a0755a94cf039859e3e91c7d1.png)    

### less-28a

![](http://www.kingkk.com/wp-content/uploads/2017/12/e604629a1a75d301d6ae0f7851500648.png) 过滤了union select的组合，那就只能盲注入了 改一下less-7的脚本就行了 把id的值改一下即可`"1' and ascii(substr(({2}),{1},1))>{0} and '1'='1".format(x,i,sql)`     由于借鉴博客的那位大佬不更了，后面的都是自己写的，可能有点粗糙

### less-29

不是很懂题目什么意思，好像直接绕过就行 payload: `?id=aa'union select 1,(select password from users limit 0,1),3 %23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/afb9090a2593656b20dab27ed89374f9.png) 后来发现，好像除了index.php,还多了hacked.php、login.php两个页面，不妨先进login看看 测试下单引号 ![](http://www.kingkk.com/wp-content/uploads/2017/12/643a5d18639f4357b1dc82a333431e5a.png) 貌似被waf拦了下来，而且跳转到了hacked.php页面，那就来分析下这个waf吧 ![](http://www.kingkk.com/wp-content/uploads/2017/12/a09978287493a4cfd1b14a56073fab25.png) $\_SERVER\['QUERY\_STRING'\]这个函数取得就是这一部分 ![](http://www.kingkk.com/wp-content/uploads/2017/12/fa1608d7d6befc2ed555b107e2fc65e3.png) 然后传到java_implimentation这个函数里进行处理 ![](http://www.kingkk.com/wp-content/uploads/2017/12/56635b3ac8752b52b65286fda1026be5.png) 单纯的切片操作，通过&将传入的参数分离，然后前两个值为id的然后将后面的内容返回 返回后交给whirelist函数处理 ![](http://www.kingkk.com/wp-content/uploads/2017/12/40fd24380ef6508f5d60beee47d89a0d.png) 若不是纯数字，就跳转到hacked.php页面 这样的话java_implimentation这个函数对于参数的处理过于粗糙，就可以构造这样以一个payload: `?idd32&id=-1' union select 1,2,3 or '1'='1`来绕过这个waf 只要前两个值为id,第三个之后纯数字即可 ![](http://www.kingkk.com/wp-content/uploads/2017/12/81ef4f4cdb98f8f85b3eb11ae7298623.png)    

### less-30

无waf的页面把29的单引号改成双引号即可 payload: `?id=aa"union select 1,(select password from users limit 0,1),3 %23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/67312c3352f2a95e28f0aeefaf3f97b2.png) 进login.php 和猜想的一样，也是改引号即可 payload: `?idd32&id=-1" union select 1,2,3 or '1'="1` ![](http://www.kingkk.com/wp-content/uploads/2017/12/eb921b6df7e5539f914f9170393db14d.png)  

### less-31

测试了下双引号，然后报错 ![](http://www.kingkk.com/wp-content/uploads/2017/12/5db6e83f77dab0a1129933b8c6873b7f.png) 根据报错闭合括号就行 ![](http://www.kingkk.com/wp-content/uploads/2017/12/394a123dc8843bd0294041e080dfe794.png) 那么这样的话login.php也同理 ![](http://www.kingkk.com/wp-content/uploads/2017/12/71222bff781c717c94abcb3bf1ce48e7.png)      

### less-32

加入单引号测试时，发现将单引号转义 ![](http://www.kingkk.com/wp-content/uploads/2017/12/f6ba3381e5ae8daace6a228b6e0425b4.png) 查看下源码 ![](http://www.kingkk.com/wp-content/uploads/2017/12/748d59221fd7e48a8e49a11a8bbd2cd9.png) 其实自己看着也有点晕，反正意思就是说通过正则匹配，将单引号' 双引号" 反斜杠\ 都在加一个反斜杠进行转义 \\' \\" \\\ 这样的话，试试宽字节注入 宽字节注入原理，就是通过unicode编码的原理，根据前两位的值判断接下来字节的大小，这样就可以将转义后的反斜杠\\一并纳入字符中，使其消失转义的功能，通俗的说法就是吃掉一个反斜杠\ ![](http://www.kingkk.com/wp-content/uploads/2017/12/3301442d6eb0cada489ffc54b5d55083.png) 报错了，那就表示有效，那就继续 附上payload: `?id=-1%df'union select 1,2,3%23` ![](http://www.kingkk.com/wp-content/uploads/2017/12/e02dbb03a931237abac88bca06de25fc.png)    

### less-33

payload和之前的一样 ![](http://www.kingkk.com/wp-content/uploads/2017/12/23fb5a3f98092dda8d2f581f50c08d86.png) 看了下源码，![](http://www.kingkk.com/wp-content/uploads/2017/12/f18e433fb558c570a24f4ea48a8d87c3.png)其实这个过滤的方式和上一个差不多  

### less-34

利用宽字节在post里注入，原理其实都一样 [![](http://www.kingkk.com/wp-content/uploads/2017/12/94c97f7509f5e8d62d68acb957af4817.png)](http://blog.csdn.net/u012763794/article/details/51457142)    

### less-35

测试一下，发现好像不需要引号，那就连宽字节都不用了？可能没懂出题人意思 ![](http://www.kingkk.com/wp-content/uploads/2017/12/275ac365bb17e1637ca38b03246bc12d.png)    

### less-36

好像和之前一样？ ![](http://www.kingkk.com/wp-content/uploads/2017/12/1d2721f5115de2cd69b09fb0f10d19b7.png) 看了下源码，这里是通过mysql\_real\_escape_string这个函数过滤的 php官方手册对这个函数的解释是

**mysql\_real\_escape_string()** 调用mysql库的函数 mysql\_real\_escape_string, 在以下字符前添加反斜杠: _\\x00_, _\\n_, _\\r_, _\_, _'_, _"_ 和 _\\x1a_.

为了安全起见，在像MySQL传送查询前，必须调用这个函数（除了少数例外情况）。

但是这个函数依旧可以用宽字节吃掉转义的反斜杠    

### less-37

还是在post里面使用宽字节吃掉反斜杠 ![](http://www.kingkk.com/wp-content/uploads/2017/12/d14cbc108854ee4eae7a41e26527881a.png)      

### less-38

一样的意思 ![](http://www.kingkk.com/wp-content/uploads/2017/12/4731f0000686116081870f66ad17599e.png)         这里主要用了三个函数过滤 preg\_replace（），addslashes()，mysql\_real\_escape\_string() 都可以通过宽字节进行绕过，所以在写代码的时候光用这三个函数是远远不够的 不过宽字节貌似只有在gbk字符集下才能使用，所以在传入sql之前会进行字符集的指定![](http://www.kingkk.com/wp-content/uploads/2017/12/66d75caef81eb13a41c85be45f77d745.png) 否则的话 ![](http://www.kingkk.com/wp-content/uploads/2017/12/d7226981d9331e207304e4721c53b639.png)           依旧感谢 [http://blog.csdn.net/u012763794/article/details/51457142](http://blog.csdn.net/u012763794/article/details/51457142)