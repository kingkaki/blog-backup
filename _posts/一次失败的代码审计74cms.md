---
title: 一次失败的代码审计-74cms
url: 193.html
id: 193
categories:
  - 代码审计
  - 学习笔记
date: 2018-02-02 20:07:55
tags:
---

哎，弄得有点头大。。弄了一整个下午，虽然还是失败了，但还是记录一下吧   emmm，一开始，想从user这个文件夹下入手，大体瞟了下那些基础的类文件，当然没看全， 然后配合seay的代码审计工具，发现了一个点 **74cms\\user\\connect\_qq\_client.php**

elseif ($act=='login_go')
{
	
	$\_SESSION\["openid"\] = trim($\_GET\['openid'\]); 
	$\_SESSION\["accessToken"\] = trim($\_GET\['accessToken'\]);
	if (empty($_SESSION\["openid"\]))
	{
		showmsg('登录失败！openid获取不到',0);
	}
	else
	{
		require\_once(QISHI\_ROOT_PATH.'include/mysql.class.php');
		$db = new mysql($dbhost,$dbuser,$dbpass,$dbname);
		unset($dbhost,$dbuser,$dbpass,$dbname);
		require\_once(QISHI\_ROOT\_PATH.'include/fun\_user.php');
		var\_dump($\_SESSION\["openid"\]);
		$user=get\_user\_inqqopenid($_SESSION\["openid"\]);
		……

重点为加粗的两句，发现$user是根据$_SESSION\["openid"\]来的，然而$_SESSION\["openid"\]是直接从$\_GET\['openid'\]上获取（仅加了个trim函数），然后跟进get\_user_inqqopenid（）这个函数看看 **74cms\\include\\fun_user.php**
```php
function get_user_inqqopenid($openid)
{
	global $db;
	if (empty($openid))
	{
	return false;
	}
	$sql = "select * from ".table('members')." where qq_openid = '{$openid}' LIMIT 1";
	return $db->getone($sql);
}
```
啊哈，$openid直接传入了sql语句，没有做什么过滤。当时就有点开心，然后去找下调用这个函数的页面 
哎。。好像页面被改的有点改不回来了，最后调试的输出页面就如下面 
输入如下url： http://localhost/74cms/user/connect\_qq\_client.php?act=login_go&openid=1 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/1f73e77372d7a9413db2b40f492e4419.png) 试着注入，发现过滤了很多关键字。。 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/67b0c76abf397d575f95d00cb990cbe9.png) 最后找到一个没过滤的payload
```
openid=1%df' || ascii(substr((user()),1,1))>10 limit 1%23
```
然而。。。 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/737cc5c9d89b36d986618b989eac2627.png) 
语句都是没问题的，可是返回的却是两个false 把语句放入数据库执行也是OK的 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/a5e942d0b0e7c78988750bc46da32da8.png) 
当时就是很懵逼的，各种输出变量，调试，最后找到了问题的原因 在执行mysql\_fetch\_array的时候，貌似自动检测单引号前面的字符是否为宽字节，是的话就不允许通过 
![](http://blog.kingkk.com/wp-content/uploads/2018/02/918c4f8a17e2d3e86d298e6e30bd6a0d.png) 
然后把那个宽字节的字改成别的话 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/deb2ab1fc55d1ac7cf4cc59e8a6a1055.png)   
万脸懵逼，而且那个$_GET也是，至今没找到过滤函数 整整弄了一个下午。。。哎，有点头大  

* * *

  来个分割线，昨天问了下大佬，今天大佬指点了几句，果然大佬还是大佬 重新回到那个connect\_qq\_client.php的页面 文件一开头就有如下的包含语句

require\_once(dirname(\_\_FILE__).'/../include/plus.common.inc.php');

跟进这个**plus.common.inc.php**文件，果然对$\_GET以及$\_POST等获得的数据一开始就做了处理

if (!empty($_GET))
{
$\_GET = addslashes\_deep($_GET);
}
if (!empty($_POST))
{
$\_POST = addslashes\_deep($_POST);
}
$\_COOKIE = addslashes\_deep($_COOKIE);
$\_REQUEST = addslashes\_deep($_REQUEST);

找下**addslashes_deep**这个函数的定义 **common.fun.php**下

function addslashes_deep($value)
{
    if (empty($value))
    {
        return $value;
    }
    else
    {
		if (!get\_magic\_quotes_gpc())
		{
		$value=is\_array($value) ? array\_map('addslashes\_deep', $value) : mystrip\_tags(addslashes($value));
		}
		else
		{
		$value=is\_array($value) ? array\_map('addslashes\_deep', $value) : mystrip\_tags($value);
		}
		return $value;
    }
}

返回的是个**mystrip_tags**这个过滤后的值，再继续跟进去看看把

function **mystrip_tags**($string)
{
	$string = new\_html\_special_chars($string);
	$string = remove_xss($string);
	return $string;
}
function **new\_html\_special_chars**($string) {
	$string = str_replace(array('&', '"', '<', '>'), array('&', '"', '<', '>'), $string);
	$string = strip_tags($string);
	return $string;
}
function **remove_xss**($string) { 
    $string = preg_replace('/\[\\x00-\\x08\\x0B\\x0C\\x0E-\\x1F\\x7F\]+/S', '', $string);

    $parm1 = Array('javascript', 'union','vbscript', 'expression', 'applet', 'xml', 'blink', 'link', 'script', 'embed', 'object', 'iframe', 'frame', 'frameset', 'ilayer', 'layer', 'bgsound', 'title', 'base');

    $parm2 = Array('onabort', 'onactivate', 'onafterprint', 'onafterupdate', 'onbeforeactivate', 'onbeforecopy', 'onbeforecut', 'onbeforedeactivate', 'onbeforeeditfocus', 'onbeforepaste', 'onbeforeprint', 'onbeforeunload', 'onbeforeupdate', 'onblur', 'onbounce', 'oncellchange', 'onchange', 'onclick', 'oncontextmenu', 'oncontrolselect', 'oncopy', 'oncut', 'ondataavailable', 'ondatasetchanged', 'ondatasetcomplete', 'ondblclick', 'ondeactivate', 'ondrag', 'ondragend', 'ondragenter', 'ondragleave', 'ondragover', 'ondragstart', 'ondrop', 'onerror', 'onerrorupdate', 'onfilterchange', 'onfinish', 'onfocus', 'onfocusin', 'onfocusout', 'onhelp', 'onkeydown', 'onkeypress', 'onkeyup', 'onlayoutcomplete', 'onload', 'onlosecapture', 'onmousedown', 'onmouseenter', 'onmouseleave', 'onmousemove', 'onmouseout', 'onmouseover', 'onmouseup', 'onmousewheel', 'onmove', 'onmoveend', 'onmovestart', 'onpaste', 'onpropertychange', 'onreadystatechange', 'onreset', 'onresize', 'onresizeend', 'onresizestart', 'onrowenter', 'onrowexit', 'onrowsdelete', 'onrowsinserted', 'onscroll', 'onselect', 'onselectionchange', 'onselectstart', 'onstart', 'onstop', 'onsubmit', 'onunload','style','href','action','location','background','src','poster');
	
	$parm3 = Array('alert','sleep','load_file','confirm','prompt','benchmark','select','update','insert','delete','alter','drop','truncate','script','eval','outfile','dumpfile');

    $parm = array_merge($parm1, $parm2, $parm3); 

	for ($i = 0; $i < sizeof($parm); $i++) { 
		$pattern = '/'; 
		for ($j = 0; $j < strlen($parm\[$i\]); $j++) { if ($j > 0) { 
				$pattern .= '('; 
				$pattern .= '(&#\[x|X\]0(\[9\]\[a\]\[b\]);?)?'; 
				$pattern .= '|((\[9\]\[10\]\[13\]);?)?'; 
				$pattern .= ')?'; 
			}
			$pattern .= $parm\[$i\]\[$j\]; 
		}
		$pattern .= '/i';
		$string = preg_replace($pattern, '****', $string); 
	}
	return $string;
}

嗯，这也就解释了为什么sql语句的很多关键字会被过滤，哎，原来之前输出的值就是已经被过滤了的$_GET参数，怪不得一直不对 至于宽字符注入不了的话，我查看了下mysql.log文件。不知道为什么wamp默认是没开的，所以以前也一直找不到，顺带这里提一下如何查看sql.log文件
```bash
mysql> **SHOW VARIABLES LIKE '%general_log%';**
+------------------+----------------------------------------------------------------------+
| Variable_name    | Value                                                                |
+------------------+----------------------------------------------------------------------+
| general_log      | ON                                                                   |
| general\_log\_file | D:\\Program Files\\wamp\\bin\\mysql\\mysql5.6.17\\data\\DESKTOP-S9PSGAM.log |
+------------------+----------------------------------------------------------------------+
```
就可以看到文件的开启状态和log文件位置，开启的方式

**SET GLOBAL general_log = 'ON';**

然后看下日志文件

180203 21:33:41 23 Connect root@localhost on 
   23 Query SET NAMES gbk
   23 Query SET sql_mode = ''
   23 Query **SET character\_set\_connection=gbk, character\_set\_results=gbk, character\_set\_client=binary**
   23 Init DB 74cms
   23 Query select * from qs_crons WHERE (nextrun<1517664821 OR nextrun=0) AND available=1 LIMIT 1
   24 Connect root@localhost on 
   24 Query SET NAMES gbk
   24 Query SET sql_mode = ''
   24 Query SET character\_set\_connection=gbk, character\_set\_results=gbk, character\_set\_client=binary
   24 Init DB 74cms
   23 Quit 
   24 Query **select * from qs\_members where qq\_openid = '1xDf\\' or 1=1 limit 1' LIMIT 1**
   24 Quit

重点就是加粗那两句，我猜应该是设置了**character\_set\_client=binary** 的原因，导致虽然编码是gbk但是依然无法宽字节注入，从sql的语句中也能看出来，那个反斜线并没有被吃掉 emm具体原因的话，网上是那么说的，具体原理，暂时没找到。。

解决方法：就是在初始化连接和字符集之后，使用SETcharacter\_set\_client=binary来设定客户端的字符集是二进制的。如：

mysql\_query(”SET character\_set_client=binary”);

总之，这是一个目前广泛使用的防止宽字节注入的方式，再具体也就不深究了 总之，这个注入思路，到这里可以算是gg了，单引号目前是逃不出来了     虽然过程有点曲折，但也算学到了一些，那些common.inc.php之类的文件还是要好好去看看