---
title: php的伪随机数
url: 206.html
id: 206
categories:
  - php
date: 2018-02-07 22:35:55
tags:
---

### 前言

之前打CTF的时候就有一道题，一个关键的点就是预测随机数，根据前一次给的随机数，预测出下一次的随机数。今天在看《白帽子》的时候就重点看了下，就顺便记录一下  

### 正文

php里两个重要的随机函数，

*   **rand()   //不指定参数时，范围0-32767**
*   **md_rand()   //不指定参数时，范围0-2^32-1**
*   **srand()   //给rand()函数播种**
*   **mt\_srand()  //给mt\_srand()函数播种**

php是基于C开发的，C中生成随机数时，需要自己去一个种子，相同的种子产生的随机数是相同的，php中也一样

> 自 PHP 4.2.0 起，不再需要用 **srand()** 或 mt_srand() 给随机数发生器播种 ，因为现在是由系统自动完成的 ————取自php官方手册

    看如下例子，在给mt_rand函数播下一个固定值时 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/fd7dd919da74099e400efcdaef764b77.png) 无论刷新，还是换不同浏览器打开，都是相同的随机数，而且接下来几次的变化也都相同，从而产生了可预测性

#### 所以，当seed一定时，接下来几次的随机数是固定的

我记得之前那题的播种时取一个0-99999的随机数进行播种，本地测试代码如下 **test.php**

<?php
 $seed = rand(0,99999);
 echo "seed is :".$seed."<br>";
 mt_srand($seed);
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
?>

取一次的输出情况 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/d646246566e64d322c5e681d3f2d15a5.png) 在我们可以知道第一个随机数(1240422887)时，从而求出seed，进而预测接下来的随机数，代码如下 **1.php**

 <?php

function get\_seed($rand\_num){
    for($i=0;$i<100000;$i++){
        mt_srand($i);
        if(mt\_rand()==$rand\_num){
            return $i;
        }
    }
    return False;
 }

 $rand_num = 1240422887;
 $seed = get\_seed($rand\_num);
 echo "the seed is:".$seed."<br>";
 mt_srand($seed);
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";
 echo mt_rand()."<br>";

?>

访问这个1.php ![](http://blog.kingkk.com/wp-content/uploads/2018/02/f7756b5a9bd1c934777ac39d061835f8.png) 成功求出了所播的seed，也预测了接下来的值  

* * *

这里刻意缩小了seed的范围，然而在php4.2之后，是不需要手动赋值的，这时系统赋予的seed值范围是在0-2^32（32位服务器上） 爆破时间应该也不算特别长，特地测试了一下，我在本地电脑上100000跑的话0.687s，2147483648的话，emmm大概四个小时，换成性能好的计算机的话，可能能快很多，总之，时间没有长的那么可怕。

* * *

下面演示下正紧的渗透测试环节，这里需要两个页面，test.php和test2.php。 test.php用来播种，test2.php用来产生随机数，访问时要保重keep-alive，这样才能保证种子至始至终都是同一个   首先  访问test.php获得seed并播种，然后执行模拟业务处理，转跳至具体业务处理页面test2.php

<?php
 mt_srand((double)microtime()*1000000);
 header("Location: http://localhost/code/test2.php");
?>

test2.php中获取到生成的随机数值 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/47ff1716d2d1dd94ee3caf479980acb1.png) 此时，就可以通过之前的函数求出seed，并预测接下来，其余未重新播种的业务将产生的随机数 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/c16997d537dab151be279dfad0e127eb.png) 刷新几次test2.php页面，验证下 ![](http://blog.kingkk.com/wp-content/uploads/2018/02/ae75cef8d1402a16d7c2eecf5d3a7a6d.png)![](http://blog.kingkk.com/wp-content/uploads/2018/02/010e4e2552c8028e677a7cd2c9707e7e.png)![](http://blog.kingkk.com/wp-content/uploads/2018/02/325874b632179796018e436fe2026a5b.png) bingo，成功的预测到了将要生成的随机数

##### **注意：**

要保持keep-alive的连接，不然这个进程断掉了，种子就不是之前test.php中的那个了 我这是用firefox的插件，增加了keep-alive的头部 其次，若是后面被重新播种，就得重新计算     假如不是人工播种的话，应该也是可以求出那个seed，只是范围变大了很多，需要耗费的时间增加，并且这一切也要建立在keep-alive的基础上（未验证，机子太渣）    

* * *

#### 总结

这样的可预测性，导致了所谓的随机数不够随机，并非正真的随机数，而是个伪随机