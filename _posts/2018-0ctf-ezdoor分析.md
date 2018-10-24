---
title: 2018 0ctf-ezdoor分析
url: 265.html
id: 265
categories:
  - CTF
date: 2018-04-06 21:45:22
tags:
---

# 前言

这道题主要是利用了php的缓存技术——opcache，本意应该是上传一个特制的.bin文件，覆盖缓存文件，导致文件覆盖。 
还有一种解法是一种比较清奇，至今不是很懂的骚操作。总之，很强👍  

# 正文

来到首页，可以看到源码
```php
    <?php
    
    error_reporting(0);
    
    $dir = 'sandbox/' . sha1($_SERVER['REMOTE_ADDR']) . '/';
    if(!file_exists($dir)){
      mkdir($dir);
    }
    if(!file_exists($dir . "index.php")){
      touch($dir . "index.php");
    }
    
    function clear($dir)
    {
      if(!is_dir($dir)){
        unlink($dir);
        return;
      }
      foreach (scandir($dir) as $file) {
        if (in_array($file, [".", ".."])) {
          continue;
        }
        unlink($dir . $file);
      }
      rmdir($dir);
    }
    
    switch ($_GET["action"] ?? "") {
      case 'pwd':
        echo $dir;
        break;
      case 'phpinfo':
        echo file_get_contents("phpinfo.txt");
        break;
      case 'reset':
        clear($dir);
        break;
      case 'time':
        echo time();
        break;
      case 'upload':
        if (!isset($_GET["name"]) || !isset($_FILES['file'])) {
          break;
        }
    
        if ($_FILES['file']['size'] > 100000) {
          clear($dir);
          break;
        }
    
        $name = $dir . $_GET["name"];
        if (preg_match("/[^a-zA-Z0-9.\/]/", $name) ||
          stristr(pathinfo($name)["extension"], "h")) {
          break;
        }
        move_uploaded_file($_FILES['file']['tmp_name'], $name);
        $size = 0;
        foreach (scandir($dir) as $file) {
          if (in_array($file, [".", ".."])) {
            continue;
          }
          $size += filesize($dir . $file);
        }
        if ($size > 100000) {
          clear($dir);
        }
        break;
      case 'shell':
        ini_set("open_basedir", "/var/www/html/$dir:/var/www/html/flag");
        include $dir . "index.php";
        break;
      default:
        highlight_file(__FILE__);
        break;
    }
    
```
大致意思就是，可以上传一个后缀名不含‘p’的文件，可以运行的文件为index.php。 
还有一些可以查看时间戳，沙箱路径，清理文件的功能。 
大体思路就是，通过上传，控制index.php文件，读取flag目录下的文件，从而获取flag。

* * *

可以先查看以下phpinfo。可以看到开启了opcache缓存技术 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/59877c27f3fb0bf7a0580a6545d075af.png) 

开启file\_cache之后，在apache上运行的文件都会在/tmp/cache（配置文件中自我指定）
下生成对应文件目录下的bin文件。 具体关于opcache的file\_cache攻击原理可以参考如下文章 
[http://gosecure.net/2016/04/27/binary-webshell-through-opcache-in-php-7/](http://gosecure.net/2016/04/27/binary-webshell-through-opcache-in-php-7/) 
或者一些国内的译文 [https://www.exehack.net/3272.html](https://www.exehack.net/3272.html) 
控制缓存文件文件主要需要获得两个参数，**timestamp**以及**opcache_id** **timestamp **.bin文件建立时的时间戳（需要与源文件最后更改的时间戳一致时，才能运行.bin，否则会重新生成） 
**opcache_id**主要是根据php\_version、zend\_extension\_id、zend\_bin_id这三个参数计算得出的。（

这三个参数都可以从phpinfo中获取） 计算公式：

system\_id = hashlib.md5((php\_version+zend\_extension\_id+zend\_bin\_id).encode('utf8')).hexdigest()

**System ID** : 7badddeddbd076fe8352e80d8ddf3e73 
**获取时间戳**：获取reset时会暂时删除index.php文件，再次访问就会重新生成，即可获得此时的时间戳
```python
import requests
print(requests.get('http://202.120.7.217:9527/index.php?action=time').text)
print(requests.get('http://202.120.7.217:9527/index.php?action=reset').text)
print(requests.get('http://202.120.7.217:9527/index.php?action=time').text)
```

为了准确，当第一次输出和第二次输出的时间戳相等时，即是准确时间 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/2d99a593c91fbae94aaf697f0eab7114.png) 
其余的字节码，需要通过在本地模拟一个类似的opcache file_cache缓存机制来获取 
这里不做过多介绍（tips:/tmp/cache目录需要自己提前创建，而且一些文件夹可能没有权限，用户无法访问，自行赋予权限）

* * *

将生成的index.php.bin文件在本地用二进制编辑器打开 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/83c513e6e3c465034d71d68ad6008ba1.png) 
这时候可以借助一个opcache的项目，来对文件进行分析 https://github.com/GoSecure/php7-opcache-override 
导入对应的模板，再运行 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/9723c390f4a94fcc10da19eb853eff51.png) 
主要需要修改的就是这两个参数，改成我们之前计算好的数据
![](http://blog.kingkk.com/wp-content/uploads/2018/04/fe62148acc2a7643b9ed98132e8fa853.png) 
另存为一个shell.bin

* * *

编写对应的upload.html文件，上传shell.bin文件

```html
<html>
<body>
 <form action="http://202.120.7.217:9527/?action=upload&name=../../../../../tmp/cache/7badddeddbd076fe8352e80d8ddf3e73/var/www/html/sandbox/e46d53933685bb7ffd339c5f0fda1c8055302d5f/index.php.bin" method="post" enctype="multipart/form-data">
 <input type="file" name="file" />
 <input type="submit" value="upload" />
 </form>
</body>
</html>
```
成功覆盖 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/6ad9d6426dd9fe1183e04522ca5f5e0d.png) 
后面就可以尝试去读取一些数据（官方将一些特定函数禁用了，因此只能用一些特定的函数） 
获取/var/www/html/flag下的目录 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/3e342b8719723ec362c935ff400fd332.png) 
读取文件内容（是个二进制流文件，防止乱码可以通过base64加密一下） 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/8fa3203277c5337447997f780273bfdc.png) 
然后写入文件，然后是一些字节码，好像是进行逆向分析把，在通过一些解密计算，得出flag（web狗表示到这已经是极限了）  

* * *

# 另一种骚操作

然后还有看到一个杭电大佬，在上传文件的路径上，通过一种奇异的方式绕过了，虽然不是很清楚原理，但是还是想记录一下 原文如下 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/2811d18bdeb616b57a15d0130973c63e.png) 
这样，就不需要那么复杂的上传.bin文件进行覆盖，也不需要获得timestamp、opcache等一系列乱七八糟的东西，就可以直接覆盖index.php文件 
（假如题意是这样的话，也就完全不用opcache，以及提供time,phpinfo这些东西了）

 这种绕过的大体思路就是:

通过一个不存在文件夹，再返回其上层文件，获取文件时就可以无视后面的/.获取文件，但是解析时依旧会以最后的.当作文件名
造成判断文件名和操纵文件产生差异

附上一些截图把

![](http://blog.kingkk.com/wp-content/uploads/2018/04/d7ca48bd628d817518001653bfd50adc.png) 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/bbbb860a022b1c56f3853b5d74e61c34.png) 
而且这个x文件夹一定要不存在，存在时依旧无法绕过。。 对这个目录文件的一些解析 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/a1294d6d595bb849727dcc4c55b173ba.png) 
后缀名为空，文件名为. 从而就绕过了后缀名含'p'的字母，然后又能覆盖到对应的文件 具体原理，目前网上好像没找到相关信息，希望以后能看下具体的原理  

# 总结

研究了整整一天，思路很骚气，0ctf的题目质量实在是高，怕是清明那几道web都消化不完…  

* * *

更新一下，看到了一个介绍关于 `x/../index.php/.`  绕过的原理介绍的文章 
![](http://blog.kingkk.com/wp-content/uploads/2018/04/2811d18bdeb616b57a15d0130973c63e.png) 
丢一下链接 [http://pupiles.com/%E7%94%B1%E4%B8%80%E9%81%93ctf%E9%A2%98%E5%BC%95%E5%8F%91%E7%9A%84%E6%80%9D%E8%80%83.html](http://pupiles.com/%E7%94%B1%E4%B8%80%E9%81%93ctf%E9%A2%98%E5%BC%95%E5%8F%91%E7%9A%84%E6%80%9D%E8%80%83.html) 
里面讲的很详细了，这里我就自己总结概括一下 `index/.`  时，操作文件流的文件可以进行文件的创建（但不能覆盖），以及绕过后缀名 `x/../index.php/.`   
则可以进行文件的创建、覆盖，以及绕过后缀名的一些检测 在`file_get_contents`以及`move_uploaded_file`这类需要打开文件流的函数时 
文件名非标准格式时（例如包含相对路径等），php要对文件名进行一些处理 其中就用到了`tsrm_realpath` 这个函数，会对去除文件名后面的 /. 将其转化为一个标准路径 
然后调用`lstat`函数，去判定文件是否存在 
存在，则不写入文件（具体原理可以看链接中的源码分析） 
不存在，则调用write函数进行文件写入 然而利用  `x/../index.php/.`这样的文件名时 
会判定改文件不存在，绕过了`lstat`函数 从而继续调用write函数，进行写文件，从而导致文件覆盖