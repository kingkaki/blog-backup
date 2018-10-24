---
title: Mysql搭建
url: 155.html
id: 155
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-18 19:29:41
tags:
---

### Mysql安装

1、由于我是CentOs6.5的版本，直接用yum就可以了

yum install mysql-server

假如是CentOs7及以上的版本，默认数据库为mariadb，要先移除mariadb，然后再进行安装

yum remove mariadb-libs.x86_64     #版本号视具体版本而定
wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
yum localinstall mysql57-community-release-el7-8.noarch.rpm
yum install mysql-community-server

2、启动mysql

service mysqld start

3、登录mysql，我这密码默认为空

mysql -u root -p

若是centos7安装的社区版的话，可能初始密码不为空，那起始密码保存在/var/log/mysqld.log中，可通过如下命令查看

cat /var/log/mysqld.log |grep password

 

### Mysql的一些基本操作

##### 修改密码

进入mysql终端后输入

mysql> SET PASSWORD = PASSWORD('123456');     #将密码设置为123456

##### 设置远程连接

1、先进入mysql终端 2、选择mysql库，并修改user表

mysql> use mysql;
mysql> update user set host = '%' where user = 'root' and host='localhost';

3、刷新权限

mysql> flush privileges;

这时候应该就可以通过navicat等工具远程连接上了，假如还不行，试试关闭防火墙  

##### 新建用户

mysql> create user 'kingkk'@'%' identified by '123456';

create user 用户名@主机 identified by‘密码’  

##### 赋予权限

可以赋予的权限有 all / select / insert /delete / update      *.*  处表示 库.表

mysql> grant all privileges on *.* to 'kingkk'@'%' identified by '123456' with grant option;

grant 权限 privileges on 库.表 to '用户名'@'主机' identified by '密码' with grant option; 刷新权限

mysql> flush privileges;

##### 移除权限

mysql> revoke all privileges on *.* from kingkk

revoke 权限 privileges on 库.表 from kingkk 刷新权限

mysql> flush privileges;