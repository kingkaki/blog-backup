---
title: Python-requests库
url: 229.html
id: 229
categories:
  - python
  - 学习笔记
date: 2018-02-14 16:49:55
tags:
---

### 前言

requests库真的特别好用，一开始进行网络编程的时候就使用的requests，封装的特别好，编程风格也很优雅 只是之前一直都是零零散散的学习一些基本操作，今天看了一遍官方文档，做一些总结，希望能更好的运用这个库  

### 正文

 

##### 请求方式

*   **requests.get()**
*   **requests.post()**
*   **requests.head()**
*   **requests.options()**
*   **requests.put()**
*   **requests.delete()**
*   **requests.patch()**

几乎涵盖了http协议的请求方式，但其实用的最多就还是get、post  

##### response状态

*   **r.text                                     解码后的文本**
*   **r.content                              gzip之类的解码后的二进制响应数据**
*   **r.url                                       请求的网址**
*   **r.status_code                      状态码**
*   **r.encoding                           指定编码**
*   **r.headers                             响应头，返回一个字典**
*   **r.cookies                              cookie，返回一个RequestsCookieJar，类似字典**
*   **r.history                               重定向的记录，返回一个从老到最近的response列表**
*   **r.request                              请求的包，即自己发送给服务器的数据包，有headers等属性**

 

##### 请求参数

*   **params = {'key1': 'value1', 'key2': 'value2'}                                      get方式的请求参数** **params = {'key1': 'value1', 'key2': \['value2', 'value3'\]}                     **
*   **data = {'key1': 'value1', 'key2': 'value2'}                                           post表单的参数** **data = {'key':\['value1','value2'\]}**
*   **json = {'some': 'data'}                                                                            自动编码json**
*   **files = {'file': open('report.xls', 'rb')}                                                  添加上传文件（需以二进制打开）**
*   **stream=True                                                                                           以流的方式打开**
*   **timeout=1                                                                                                等待响应的最长时间（单位sec）**
*   **headers = {'user-agent': 'my-app/0.0.1'}                                           添加头部**
*   **cookies = {'cookies_are': 'working'}                                                   添加cookie**
*   **proxies = {                                                                                                添加代理** **"http": "http://127.0.0.1:8080",** **"https": "http://**127.0.0.1**:1080",** **}**

 

### 一些常用操作

##### requests.session()

用以keep-alive保持连接，并保持会话cookie

s = requests.Session()
s.get('http://www.baidu.com')

关闭连接

s.close()
#更优雅的方式,上下文管理器
with requests.Session() as s:
     s.get('http://www.baidu.com')

  tip:正确写法应该是requests.Session()，但是通用  

##### 下载图片等资源

r = requests.get(url, stream=True)
with open(filename, 'wb') as f:
     for chunk in r.iter_content(1024):    #文件不大时可以直接for chunk in r:
         f.write(chunk)