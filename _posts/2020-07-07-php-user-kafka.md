---
title: PHP使用kafka
description: PHP使用kafka
categories:
 - kafka
 - PHP
tags:
 - kafka
 - PHP
---

## PHP的kafka 接入使用文档

后端首次大规模使用kafka，因此这里先介绍下kafka的基本使用原理

## kafka基本概念
 
 大数据处理标配。
 
 它提供了发布和订阅功能，使用者可以发送数据到Kafka中，也可以从Kafka中读取数据（以便进行后续的处理）。Kafka具有高吞吐、低延迟、高容错等特点， 主要是磁盘顺序存储
 
### 基本概念：
 
 * Broker  
 消息队列中常用的概念，在Kafka中指部署了Kafka实例的服务器节点。简单理解即服务器数

* Topic  (简单理解即队列)。  
用来区分不同类型信息的主题。比如应用程序A订阅了主题t1，应用程序B订阅了主题t2而没有订阅t1，那么发送到主题t1中的数据将只能被应用程序A读到，而不会被应用程序B读到。

* Partition  
每个topic可以有一个或多个partition（分区）。分区是在物理层面上的，不同的分区对应着不同的数据文件。Kafka使用分区支持物理上的并发写入和读取，从而大大提高了吞吐量。(上面topic是逻辑概念， 这里的分区是物理上，队列数据通过某种hash算法，将数据均匀的分布在几个分区上)

* Record  
实际写入Kafka中并可以被读取的消息记录。每个record包含了key、value和timestamp。
(即消息内容)

* Producer  
生产者，用来向Kafka中发送数据（record）。
* Consumer  
消费者，用来读取Kafka中的数据（record）。(普通消费者)
* Consumer Group  
一个消费者组可以包含一个或多个消费者。使用多分区+多消费者方式可以极大提高数据下游的处理速度。 (高级消费者)

### 基本原理

topic(消息队列)  同一类别的消息记录（record）的集合。在Kafka中，一个主题通常有多个订阅者。对于每个主题，Kafka集群维护了一个分区数据日志文件结构如下

每个partition都是一个有序并且不可变的消息记录集合。


### 生产流程

当新的数据写入时，就被追加到partition的末尾。在每个partition中，每条消息都会被分配一个顺序的唯一标识，这个标识被称为offset，即偏移量。注意，Kafka只保证在同一个partition内部消息是有序的，在不同partition之间，并不能保证消息有序。

所以如果要求强顺序， kafka不满足要求；如果只是希望同一个用户的消息保持顺序， 可以通过`key` 将此用户的消息，都分配到同一个分区，保证此用户数据的顺序性。

### 消费流程

每个消费者保存的唯一的元数据就是它所消费的数据日志文件的偏移量。

偏移量是由消费者来控制的，通常情况下，消费者会在读取记录时线性的提高其偏移量。不过由于偏移量是由消费者控制，所以消费者可以将偏移量设置到任何位置，比如设置到以前的位置对数据进行重复消费，或者设置到最新位置来跳过一些数据。

偏移量存储在kafka-server中，需要消费者提交到kafka-server中



## PHP的客户端调研选择

主要是拿github中排名前两位的类库做对比。

### kafka-php

[github地址](https://github.com/weiboad/kafka-php)

PHP包， 代码最后更新时间在2年前， 大量issue积压。
试用存在包兼容性问题

### php-rdkafka

[github地址](https://github.com/arnaud-lb/php-rdkafka)

C扩展实现， 基于C++得rdkafka类库封装，功能比较完备。 核心团队持续维护(最后更新代码在2天前)

### 最终选择php-rdkafka

综合类库代码质量，以及代码的高效性， 最终选择php-rdkafka扩展


## php-rdkafka安装流程+使用流程


*  首先安装依赖的C++ librdkafka包(https://github.com/edenhill/librdkafka)
*  安装php-rdkakfa扩展(https://github.com/arnaud-lb/php-rdkafka#installation)
*  为了能更好将扩展方法在IDE中类型提示， 需要安装对应的类型提示包(https://github.com/kwn/php-rdkafka-stubs)


## 常用的使用

### 发布消息

需要必备的字段 

* broke_list : 服务器节点
* topic_name: 发布的topic队列名称
* msg: 消息主体
* partition: 分区索引， 即将消息打到哪个分区。不填则默认
*  key: 消息key， 即相同key会打到同一个分区

重要的参数设置

*  socket.timeout.ms  网络链接超时时间 
*  queue.buffering.max.ms 发送本地缓消息自动提交时间， 调试次时间，可以增加消费者发送消息吞吐
*  request.required.acks  
   消费者发送消息 server端确认方式，0=不发送任何 response/ack 给客户端, 1=只有 leader broker 需要 ack 消息, -1 or all=broker 阻塞直到所有的同步备份(ISRs)或in.sync.replicas设置的备份返回消息提交的确认应答

常见问题： 

如何提高发布吞吐：  

1. 客户端批量发布多条， 即调整queue.buffering.max.ms
2. 调整server端确认方式，即调整request.required.acks  

如何计数统计发送成功次数，失败次数

类库提供对应回调方法

* setDrMsgCb 投递消息成功回调函数
* setErrorCb 调用失败回调函数
* setOffsetCommitCb 设置offsetcommit 回调函数
* setRebalanceCb  reblance重新平衡函数

需要`flush`操作, 或者主动调用`poll`方法， 否则回调函数不会执行

### 低级消费
需要必备的字段 :

* broke_list : 服务器节点
* topic_name: 发布的topic队列名称
* group.id : 组ID, 默认是 zdmDefaultConsumerGroup
* partition: 分区索引， 即消费哪个分区的消息

必须指定group.id和partition， 因为消费偏移量是 和二者直接关联

重要的参数设置:

*  socket.timeout.ms  网络链接超时时间 
*  auto.offset.reset 如果偏移量存储还没有初始化或偏移量超过范围时的处理方式, earliest 最早偏移地址，latest最晚偏移地址
*  enable.auto.commit 是否开启自动上报偏移量， 默认true
* enable.auto.offset.store 将自动提交偏移量存储到内存中, 默认开启
* enable.auto.commit 是否开启自动提交
* auto.commit.interval.ms 自动提交的时间间隔
* max.poll.interval.ms 高级消费者模式下，消费最大时长， 如果此次消费时长太长，则server端会剔除该group成员， 重新reblance 消费端分区

常见问题：

1. 高级消费和低级消费的区别？   
高级消费即不需要指定消费的分区， sdk自动帮你选择消费的分区

2. 希望重新消费数据，需要怎么操作   
  可以重设消费偏移量 `lowLevelConsume`方法的option参数传递 `consume_start_offset`. 
 *   RD_KAFKA_OFFSET_BEGINNING 从最开始开始消费
 *   RD_KAFKA_OFFSET_STORED 从上次消费的位置开始消费
 *   RD_KAFKA_OFFSET_END 从最新位置开始消费
 *   rd_kafka_offset_tail(xxx) 从最新偏移量的某个位置开始消费

3. 是否支持批量消费.  
   sdk支持，但我们的调用方法没有封装， [具体使用参考](https://arnaud.le-blanc.net/php-rdkafka-doc/phpdoc/rdkafka-consumertopic.consumebatch.html)


### 高级消费

需要必备的字段 :

* 不需要指定分区，其他和低级消费参数一致

重要的参数设置：

和低级消费参数一致

常见问题:

1. 如何重设消费者偏移量， 或者手动设置消费偏移量？   

 ```
 先将自动enable.auto.offset.store存储设置false， 然后每次调用 https://arnaud.le-blanc.net/php-rdkafka-doc/phpdoc/rdkafka-kafkaconsumertopic.offsetstore.html
 ```


### 业务监控程序运行(封装调用类)

1. 获取全部客户端配置参数， 排查问题使用

```
$kafka->kafka_conf->dump()
```


2. 获取当前队列的起始offset

```
$kafka = $this->load->kafka("user");
$topic = $kafka->newTopic("user");
$list = $kafka->getMetadata(false, $topic, 1000);
//        var_dump($list->getTopics()->current()->topic);
$high =  0;
$low = 0;
// 获取水位线，即当前partion的 最高，最低值, 分区offset的最低，最高
$array = call_user_func_array(array($kafka, 'queryWatermarkOffsets'), array("user", 0, &$high, &$low, 1000));
var_dump($high);
var_dump($low);
```

3. 获取当前group对于分区的 偏移量

```
$kafka->createClient(["group.id" => "test0000010001"]);
$topicPartition = new RdKafka\TopicPartition('user', 0);
$offsets = $kafka->high_level_consumer_client->getCommittedOffsets([$topicPartition], 5000);
var_dump($offsets);
```


### 运维配置

新建topic需要提前联系运维操作

topic默认是3分区，2副本。

运维后期可以调整分区个数。

### 开发环境和mq对比压测(简单版)

结论：
    1. 生产端， kafka和mq性能较高， 没有明显区别
    2. 消费端，kafka比mq的qps高两个数量级

kafka(三个节点， 目前运维较忙，还没给详细配置)

* 生产  
  1.4万qps(一次批量生产100s)
  
  响应时间 5ms
  
  响应时间 50ms(一次生产一条)

* 消费
   空跑：2.7万qps

rabbitmq（16c 16G 140G）无镜像备份机

* 生产    
  1.2万qps
  
  响应时间 5ms
  
  响应时间(46ms)

* 消费
   空跑：640qps 
	

### 剩下需要重点注意

1. 催促运维完善，kafka的grafana监控， 目前监控指标较少，不方便排查问题
2. 评估kafka使用场景， 对消费严谨，只能消费一次，涉及订单，金钱等，rabbitmq首选，其他电商点击，最后一次登录时间等，都可以使用kafka








