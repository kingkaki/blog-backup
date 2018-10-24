---
title: php之MVC框架
url: 189.html
id: 189
categories:
  - php
  - 学习笔记
date: 2018-01-31 19:11:25
tags:
---

由于代码审计的需要，简略的学习了下，MVC大体的架构，希望对以后代码审计时一些结构逻辑能有帮助把。  

### MVC（摘自wikipedia）

> **MVC模式**（Model–view–controller）是[软件工程](https://zh.wikipedia.org/wiki/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B "软件工程")中的一种[软件架构](https://zh.wikipedia.org/wiki/%E8%BD%AF%E4%BB%B6%E6%9E%B6%E6%9E%84 "软件架构")模式，把软件系统分为三个基本部分：模型（Model）、视图（View）和控制器（Controller）

 

##### 以下是我自己对MVC三个部分一个粗略的理解

*   Model（模型）连接数据，取得，并且处理原始数据，取得想要的数据
*   View（视图）将数据通过想要的形式展示出来
*   Controller（控制器）控制模型和视图，取得model所产生的数据，然后交给view去展示

 

##### 工作流程：

1、从index.php文件进入，获取到要执行的控制器（controller）以及它的方法(method) 2、控制器（controller）调用对应的控制器类文件（controller.class）实例化出对应的类，并执行对应的方法（method） 3、在方法（method）中

*   a）调用对应的模型类文件（model.class）实例化出对应的类，并执行其对应的获取数据方法，获取数据
*   b）然后调用对应的视图类文件（view.class）实例化出对应的类，并传入模型取得的数据，执行对应的方法展示数据

 

##### 示例：

1、传入controller=test,method=show 2、实例化出testController,并执行testController->show() 3、在testController->show()中                      //其中的具体实例化的类由具体函数决定，并非动态传入

*   a）先实例化出testModel，执行testModel->get(),获取数据$data
*   b）然后实例化出testView,执行testView->show($data)

        算是一个大概上的认识，然后其余的一些网上教程的事例，零零散散看了一些，没看完，感觉不是很有用，偏开发，但是又有点鸡肋 顺便了解了下视图模板，由于之前学过django的开发，感觉还是有很多相似之处。