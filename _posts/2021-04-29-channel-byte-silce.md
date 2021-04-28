---
title: 跨协程channel指针数据共享问题
description: 跨协程channel指针数据共享问题
categories:
 - golang
 - channel
tags:
 - golang
 - channel
---

# 跨协程channel指针数据共享问题

## 背景

目前项目基于uber的zap做debug日志， 现在输出到kafka中， 进而通过内部kibana平台检索日志。

因此自定义实现了`io.writer`接口， `Write`方法中做 `kafka`写日志操作逻辑。

因为写kafka，存在网络慢问题， 会影响主逻辑正常运行。 决定`Write`方法中先写入预先定义的channel, 再起单独协程，从channel中消费日志去写kafka

大致代码如下

```
type ZlogKafkaWriter struct {
	topic string
	dataChan chan []byte
}

// 生成输出到kafka的日志 writer
func InitZlogKafkaWriter(topic string) (writer io.Writer) {
	var zlogKafkaWriter *ZlogKafkaWriter
	zlogKafkaWriter = &ZlogKafkaWriter{
		topic:topic,
		dataChan: make(chan []byte, 1000),
	}
	go zlogKafkaWriter.HandleWrite()
	writer = zlogKafkaWriter
	return
}

func (zlogKafkaWriter *ZlogKafkaWriter) Write(p []byte) (n int, err error) {
	select {
		case zlogKafkaWriter.dataChan <- p:
			//
		default:
	}
	return
}

func (zlogKafkaWriter *ZlogKafkaWriter) HandleWrite() {
	var kafkaConn *zkafka.KafkaConn
	var err error
	for {
		select {
			case p := <- zlogKafkaWriter.dataChan:
				// 真正写入kafka逻辑
		}
	}
}
```

## 测试环境实际运行

从kinbana中检索数据，发现存在日志丢失情况， 明明打印日志了，但检索不到， 日志总条目却完全正确。 仔细review代码，发现代码并未有逻辑bug

反复debug排查，最后将打入channel 和从channel中消费的数据条目都做编号处理， 排查前后编号相同的数据是否吻合。

最终发现， 在从channel中消费到的数据存在脏读，读两遍的概率。


回头重新review代码， 怀疑可能是write方法传参`p []byte`存在 二次使用问题。去上层代码zap中确认到其 调用传参取自`buff` ,会重复使用， 并不是每次调用都生成新的`[]byte`

```
buf            *buffer.Buffer
```

真相大白， 因为channel 异步处理数据，导致我从channel中取数据时， 对应的`[]byte`已被上层zap重新写入新的日志内容，即原有日志内容被覆盖。

## 解决方案

从Write 参数中读取的`p []byte` 重新copy出新的 切片中即可解决， 对应方案代码:

```
func (zlogKafkaWriter *ZlogKafkaWriter) Write(p []byte) (n int, err error) {
	d := make([]byte, len(p))
	copy(d, p)
	select {
		case zlogKafkaWriter.dataChan <- d:
			//
		default:
	}
	return
}
```

## 深入思考

跨协程， channel通信时， 如果数据为指针类型， 一定要确认：

 1. 上游投递方协程是否在投递结束后， 仍然会修改此指针内容。  如果会修改，则导致下游协程脏读，因此
 2. 上下游协程都会修改指针内容的话， 需要加锁处理，否则数据协程不安全，导致数据写坏
 
 更省事的方案是， 将通信数据序列化为json字符串， 缺点是数据多个副本，导致存储浪费。
 
 


