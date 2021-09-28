---
title: Milvus踩坑实践
description: Milvus踩坑实践
categories:
 - Milvus
tags:
 - vector
---


# 向量数据库Milvus数据库踩坑


## 背景

公司推荐业务想增加一路向量召回， 目前数据库选型Milvus。

Milvus的配置信息:

*  Milvus 1.1.1版本
*  采用集群方式， 数据通过nfs共享


业务使用方式为： 传一组ID进来： 

1.   先调用MIlvus的`GetEntityByID`接口获取向量列表
2.  根据获取到的向量ID调用Milvus的`Search`接口获取召回向量列表


线上进行压测后， 发现步骤1(GetEntityByID)耗时巨大, 在400ms+， 查询接口在30ms左右， 断定GetEntityByID必然存在问题。

## 定位原因


首先翻阅的官方文档issue, 但是说其已经优化过该接口， 性能已经很好了（https://github.com/milvus-io/milvus/issues/4756）， 但现实情况是，该接口确实很慢， 存疑。

翻阅代码（源码地址： DBImpl----> GetVectorsByIdHelper方法）， 查看`GetEntityByID`背后的具体原理， 其执行步骤有以下：

1. 遍历待查询的segement文件
2. 从程序缓存中读取文件对应的bloom块， 命名为bloom_from_cache
3. 判断ID是否在bloom块中
3. 从程序缓存中读取文件对应的 ID文件块, 命名为uid_from_cache
4. 从ID文件块中判断该ID的偏移量offset
5. 根据偏移量，加载指定向量LoadsSingleVector


这里重点看了下从缓存读取值的情况， 发现uid_from_cache, 未读到， 会文件， 但查完文件后， 并没有把数据重写到缓存中

```
auto LoadUid = [&]() {
            auto index = cache::CpuCacheMgr::GetInstance()->GetItem(file.location_);
            if (index != nullptr) {
                uids_ptr = std::static_pointer_cast<knowhere::VecIndex>(index)->GetUids();
                return Status::OK();
            }

            return segment_reader.LoadUids(uids_ptr); // 这个方法里面并没有回写到CpuCacheMgr中
        };
``` 
加上日志打印， 重新编译源码， 验证发现每次都是查文件， 并没有读缓存。但是读nfs文件，是比较费时的网络IO操作

所以这里我们大致判断为读文件问题。

然后我们线上进行对比压测， 将数据从nfs存储调整为硬盘存储。 重新压测， 发现响应时间确实降低很多。


然后去官方微信群里询问问题
 

```
环境：Milvus 1.1.1版本， 采用集群方式， 数据通过nfs共享
场景需求： 根据多个ID获取向量实体， GetEntityByID
遇到的问题：
1. GetEntityByID相比search接口， 性能差很多， 响应延迟200ms+
经过我们分析压测， 我们的nfs性能较差， 改用本地硬盘存储会好很多
然后翻了一下源码：
GetVectorsByIdHelper方法中的LoadUid(即文件索引)， 每次读缓存都没有命中， 而是直接查的文件，从文件查完后， 并没有将数据回写到缓存，想问下， 不写入缓存的原因是啥？

基于集群模式怎么调整，才能提升GetEntityByID性能？
```

官方答复：

```
如果要直接从内存里面去找原始向量，就需要把原始向量数据全部加载到内存里面，milvus缓存到内存中的segment数据是lru机制的。把原始向量数据加载到内存以后，就会把内存里的索引文件挤出去，会影响到search的性能。milvus会优先保证search的性能。
另外，在milvus1.1.1里面，已经对get_entity_by_id的性能做过优化了，如果数据是放在ssd上面的，现在取一条向量的时间应该在几毫秒。
```


## 最终解决方案


问题基本明了， 是为了保证索引性能， 才对get_entity_by_id 有性能牺牲。 且官方建议， 采用其他方式比如Mysql， redis，Mongo缓存 ID和向量的映射。避免在高并发下调用get_entity_by_id， 因为该接口读取文件性能不佳。


所以我们最终采用Redis缓存 ID和向量关系




