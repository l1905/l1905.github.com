---
title: 日常开发之php-mongo连接数打高踩坑记录
description: 日常开发之php-mongo连接数打高踩坑记录
categories:
 - 日常开发
tags:
 - PHP
---


### 遇到的问题

目前PHP项目把一些消息日志存储从Mysql迁移到Mongo上， 但是服务一上线, 很快Mongo 连接数打满。 这里分析下具体原因

### 原因分析

排查文档， 发现 PHP的mongo扩展是主要元凶。 主要体现在

1. 每个PHP-FPM进程维护一个 mongo持久连接
2. 用户无法主动关闭 mongo持久连接， 除非把进程杀掉


查阅官方文档， 发现很多用户都想要实现php-mongo的短连接， 但是开发者坚持认为mongo连接池非常有必要， 能提高性能。 官方不支持php-mongo的短连接， 如果用户强烈需要， 可以自己修改扩展源码。

> https://github.com/mongodb/mongo-php-driver/issues/732

> https://github.com/mongodb/mongo-php-driver/issues/969

但是，如果PHP大规模开发， 每个机器上起1024个 worker进行， 10个机器， 非常轻松的就是1万个mongo连接数。 可以断定，开发者没有大流量使用php-mongo的场景


### 解决方案

1. 下沉使用mongo的PHP服务， 比如之前50个web机直接连mongo, 现在是下沉出5个web机，专门处理mongo相关业务
2. 也是下沉mongo的PHP服务，但是使用golang来重写，因为golang高性能。支持连接池，高并发