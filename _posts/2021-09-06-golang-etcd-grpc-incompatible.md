---
title: golang升级etcd解决 grpc兼容性问题
description: golang升级etcd解决 grpc兼容性问题
categories:
 - golang
tags:
 - grpc
 - etcd
---


# golang升级etcd解决 grpc兼容性问题(最新方案，不依赖replace）

## 背景

目前基于milvus-go-sdk开发项目， 其依赖grpc，最低版本要求 v1.27.0， 但框架中集成的etcd依赖的grpc版本是 v1.26.0

1. 升级grpc到v1.27.0, 导致etcd不好用
2. 维持grpc版本在v1.26.0, 导致milvus-go-sdk不好用。


## 报错现象

遇到了各种报错

```
`golang` 调用 `etcdv3` 报错 `undefined: balancer.PickOptions`
```

```
go: github.com/coreos/bbolt@v1.3.6: parsing go.mod etcd 3.5
```


## 过时方案

使用replace方法替换， 临时本地解决， 多项目依赖 replace，治标不治本！ 大部分解决方案都是基于此


## 最新方案

一、 升级etcd到最新版本， 修改etcd调用方式

`go get -u go.etcd.io/etcd/client/v3`

二、 代码中修改etcd包引用方式

```
import "go.etcd.io/etcd/clientv3" 修改为  import "go.etcd.io/etcd/client/v3"
```

## 参考文章

1. https://chunlife.top/2021/06/17/etcd%E7%BB%88%E4%BA%8E%E8%A7%A3%E5%86%B3%E5%86%B2%E7%AA%81%E4%BA%86/




