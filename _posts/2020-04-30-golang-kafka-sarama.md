---
title: golang连接kafka(sarama)内存暴涨问题记录
description: golang连接kafka(sarama)内存暴涨问题记录
categories:
 - golang
tags:
 - golang
 - kafka
 - sarama
---

# golang连接kafka(sarama)内存暴涨问题记录


### 问题背景

有同事反馈使用gin封装的框架， 使用kafka客户端类库(sarama）步发布消息， qps为100+， 上线后内存，cpu爆炸。

### 排查过程

1. 首先排查代码层面写的有逻辑bug， 比如连接未close， 排查无问题
2. 排查发布的消息较大， 导致golang频繁gc,  和同事确认，无频繁gc
3. 通过查看源码，发现每次http请求，操作kafka都是短链接， 即频繁的会新建短链接， 排查到这里，还是不能特别确认是因为短链接导致， 因为之前接入rabbitmq类库，也是使用的短链接。
4. 使用`pprof`打印出火焰图, profile, 和block的， 也没发现特别大的bug点。
5.  使用`sarama meory`搜索官方issue, 和谷歌查询。 得到出具体结论

### 分析具体问题

从[官方issue](https://github.com/Shopify/sarama/issues/1321) 得知， sarama类库自动依赖第三方的统计类库go-mertic, 主要是为了方便给`prometheus`统计数据。 sarama类库默认打开。导致该统计站的内存，迟迟未释放

因此使用sarama前， 将该统计关闭即可。

对应代码:

```
metrics.UseNilMetrics = true
```


### 问题回顾

发生此次问题，主要有以下原因 

1. sarama建议使用长连接 
2. pprof查看内存火焰图，需查看heap.

本次新收获的pprof使用方式

```
// 自动下载文件
go tool pprof http://localhost:8109/debug/pprof/profile?seconds=3
// 监听端口
go tool pprof -http=":8081" ~/Downloads/pprof.web-user-dc.samples.cpu.003.pb.gz
```


### 参考文章：

1. https://github.com/Shopify/sarama/issues/1358
2. https://github.com/Shopify/sarama/issues/1321
3. http://lday.me/2017/09/02/0012_a_memory_leak_detection_procedure/
4. https://zhuanlan.zhihu.com/p/37405836


