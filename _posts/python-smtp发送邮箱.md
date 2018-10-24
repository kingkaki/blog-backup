---
title: Python-smtp发送邮箱
url: 225.html
id: 225
categories:
  - python
  - 学习笔记
date: 2018-02-12 18:08:36
tags:
---

### 前言

主要就记录下最常用的发送邮件的方式吧，复杂的，有需要的话再去学。一切从简…一切从简……  

### 正文

主要有两种smtp形式，一种是没有ssl加密的（如新浪、163等），还有就是有ssl加密的（如腾讯、谷歌等） 要使用smtp的话要在设置里开启smtp服务，具体不会可以自行百度，然后就以新浪和腾讯的邮箱为例 **新浪邮箱**

#encoding=utf8
import smtplib
from email.mime.text import MIMEText

msg_from ='*****@sina.cn'         #新浪邮箱
passwd ='******'                  #新浪密码
msg_to = '****@**.**'             #收件人邮箱
                            
subject = "*********"             #主题     
content = "**********"            #正文

msg = MIMEText(content, 'plain', 'utf-8')          #用MINMEText格式化文本
msg\['Subject'\] = subject          
msg\['From'\] = msg_from
msg\['To'\] = msg_to

s = smtplib.SMTP("smtp.sina.cn", 25)               #邮件服务器及端口号
s.login(msg_from, passwd)                      
s.sendmail(msg\_from, \[msg\_to\], msg.as_string())

s.quit()

**腾讯邮箱**

#encoding=utf8
import smtplib
from email.mime.text import MIMEText
 
msg_from = '*********@qq.com'             #腾讯邮箱
passwd = '**********'                     #腾讯邮箱的授权码
msg_to = '***@**.**'                      #收件人邮箱
                            
subject = "*********"                     #主题     
content = "*********"                     #正文 
msg = MIMEText(content, 'plain', 'utf-8')          #用MINMEText格式化文本 
msg\['Subject'\] = subject
msg\['From'\] = msg_from
msg\['To'\] = msg_to

s = smtplib.SMTP_SSL("smtp.qq.com", 465)           #邮件服务器及端口号
s.login(msg_from, passwd)                      
s.sendmail(msg\_from, \[msg\_to\], msg.as_string())

s.quit()

    emmm，发送html，发送附件其实也就改下格式，然后格式化下文本，但其实可能用的不多，以后用到再去看看吧