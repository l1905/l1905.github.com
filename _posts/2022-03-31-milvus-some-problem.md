---
title: milvus使用总结
description: milvus使用总结
categories:
 - milvus
tags:
 - milvus
---


# milvus项目应用中问题总结



一、版本选择



目前miluvs官方区分1.0,2.0版本

1.0: 版本基于C开发， 属于稳定版本

2.0: 基于go开发， 目前属于开发中， 还未发布稳定版



线上业务， 基于稳妥， 建议选择1.1.1稳定版本

## 二、  内存和磁盘的用量估算



采用官方工具：<https://zilliz.com/sizing-tool>, 可快速计算出内存和磁盘占用

影响内存和磁盘的主要参数有

1. Number of vector： 向量总数
2. Dimensions： 向量的维度(创建index时指定)
3. nlist: 聚类时划分向量桶的总数(创建index时指定)
4. m: pq索引时需要指定
5. Segment file size： 段文件大小， 默认1GB, 提高该参数，可以提升检索性能

## 三、数据量评估



较小数据集: 百万级以下

较大数据集: 百万级别以上

## 四、 分布式架构选择



单机版本无法很好的支持高并发请求，因此选择分布式架构

官方提供的分布式架构: Mishards方案， 但建议在较大数据集 采用。

经压测， 如果Mishards在较小数据集下使用， 横向加机器，效果不明显。

较小数据集方案： nginx(负载均衡) + 横向多台mivlus查询服务+ 通过nfs共享数据盘方案。

如果需要扩容，横向加milvus查询服务。

## 五、 索引选择



较小数据集： FLAT索引即可， 100%召回

较大数据集(内存磁盘有限)： 可选择 [IVF_FLAT](https://milvus.io/cn/docs/v1.1.1/index.md#IVF_FLAT)，[IVF_SQ8](https://milvus.io/cn/docs/v1.1.1/index.md#IVF_SQ8) 等其他索引， 但并不是100%召回

## 六、 根据 ID获取向量方法get_entity_by_id 谨慎使用



在压测召回服务压测过程中， 如果采用get_entity_by_id 方法， 会导致延迟很大100ms+

和官方确认：该方法运行时，会读取磁盘，导致相应较慢。

官方确认方案： 将文章ID， milvus映射关系，存储到第三方库Redis， Mysql，Mongo中。

miluvs主要是做查询服务

## 七、 连接选用长连接



压测高并发时， 查询短连接会延迟较大， 因此需考虑长连接提供服务



## 八、 数据共享磁盘 nfs



nfs性能较渣， 避免直接查询磁盘。 建议内存>= 全部向量大小。



## 九、参考资料：

1. 官方文档： 

   <https://milvus.io/cn/docs/v1.1.1/overview.md>



2.   <https://cloud.tencent.com/developer/article/1607298>














