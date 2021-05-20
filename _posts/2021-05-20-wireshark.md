---
title: wireshark分析包入门
description: wireshark分析包入门
categories:
 - 网络
tags:
 - 抓包
 - wireshark
---


# wireshark分析包入门


## 背景

最近排查k8s中redis启动后， 新建连接和配置文件配置连接数不一致， 因此需要抓包验证场长连接是在哪里终端。 这里第一次使用wireshark排查问题， 现将wireshark的使用方式记录，供以后回忆


## wireshark 主要用途

既可以在线抓包做分析，也可以在服务器上抓包(tcpdump或tshark， 在wireshark上做分析。

分析主要从两个方面

1. 过滤筛选出待分析的网络包， 比如问题包 TCP-rst包
2. 按多种维度对抓包内容统计， 比如  对<会话>进行统计， 包长度进行统计



## wireshark 页面主要简介

从上到下依次有:

1. 顶部导航栏， 即wireshark会提供哪些功能
2. 主要功能区， 主要功能包括开始抓包， 结束抓包，抓包配置，打开抓包文件，查找包等快捷功能
3. 输出过滤内容区
4. 抓包列表， 会为我们展示过滤出的每条包内容， 可以在`view->
5. 底部包的详细内容: 从协议层面展示包的结构


## wireshark的使用指南

现在我们利用tcpdump从线上导出了抓包内容。
然后在 wireshark中做分析。

### 打开待分析的包文件

右键文件， 使用 wireshark打开即可

### 过滤我们想要的包

因为包的个数太多， 我们需要过滤出我们想要的包， 因此有以下几种过滤方式

1. input框输入

需要熟悉filter语法

* 过滤指定的协议， 直接输入即可 tcp/udp/arp
* 过滤是否包含内容 `contains`
* 过滤是否相等 `==`
* 过滤出含有某些字段：直接输出协议对应字段，不需要字段名字
* 简单的逻辑运算符: `and or (), gt, lt`

复杂的语法还是很难记住， 所以我们需要更便捷的筛选方式

2. 列表框中选中某一列字段

右击，可以看到有几个筛选项：

* apply as filter : 选中后，则直接进行此项筛选
* prepare as filter: 选中后， 则将筛选内容加入到筛选框中， 但还没执行， 方便再添加其他筛选命令
* Conversation filter： 按会话进行筛选， 这里选择ipv4, 或者ipv6进行筛选
* Follow: 这里会追踪指定的一次tcp，udp，或者http流，方便我们更佳便捷观察流量的往返交互

这里就不需要我们手动输入命令了，但是，我们还是有更复杂的筛选需求， 比如希望过滤出 tcp协议端口是xxx的网络， 因此并不能满足我们的需求

3. 底部协议消息展开后选中字段进行筛选


我们点击底部的协议内容，发现字段都可以展开。

右击，可以看到， 提供的菜单，和2中一样， 可以进行组合筛选

因为字段非常多， 所以我们能做的筛选范围也更广。

4. Ctrl+f筛选

有时候，我们想更快捷对包内容筛选，可以采用这种方式， 能快速帮助我们找到想要的包

5. 保存常用的查询语句

在输入框左侧，可以选择自己保存的常见命令， 在输入框的右侧，可以直接展示出常用的命令


### 宏观统计

主要在菜单栏Statistics 中一些选项


* 协议树中包的占比(protocal Hierarchy))

比如tcp包占比， 二次协议包占比等，一般用来分析包大小


* conversation回话

筛选出的包列表太多， 我们可以将其归纳为会话的方式，即多次往返包交互属于哪一次会话

ipv4： ip到ip之间的统计， 快速找到和哪些IP通信， 通信包大小，通信时长
tcp： 流的方式: 即 ip:port->ip:port方式， 可以方便的按照响应时间排序，包大小排序，可以先筛选，然后再看具体会话数量
udp： 同tcp
主要作用: 可以筛选回话，进而跟踪tcp流或者udp流

* endpoints端点

简化版本会话， 支持查看接收包信息，但是不支持追踪流和过滤，主要查看作用

* packet Lengths包长度

主要统计包的长度范围，可以进行筛选，明确包是否太大，还是太小，如果太大，会造成阻塞，太小会加大开销

* IO graphs 时序图

可以图形化按时间展示包大小等

* follow graph

可以按照时序图方式，展示之前的数据往返






