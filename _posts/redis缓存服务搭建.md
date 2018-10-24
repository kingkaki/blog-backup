---
title: Redis 缓存服务搭建
url: 161.html
id: 161
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-19 12:08:48
tags:
---

### 安装 Redis

1、从官网获取下载连接 [http://download.redis.io/releases/redis-4.0.6.tar.gz](http://download.redis.io/releases/redis-4.0.6.tar.gz) 并通过wget方式下载,并解压

wget http://download.redis.io/releases/redis-4.0.6.tar.gz
tar -xzvf redis-4.0.6.tar.gz

2、安装依赖包

yum install gcc

3、切换到其目录下编译安装

cd  redis-4.0.6
make MALLOC=libc
make install

4、切换目录，启动redis

cd src
./redis-server

 

### Redis 操作

大致看了下redis的一些操作，由于对redis其实并不熟悉，决定先暂时放下这块内容，后期对它有学习需求时再来学习