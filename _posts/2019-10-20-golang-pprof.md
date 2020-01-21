---
title: 基于pprof工具分析性能问题
description: 基于pprof工具分析性能问题
categories:
 - golang
tags:
 - golang
 - pprof
 - 性能分析
---

# 基于pprof工具分析性能问题

## 背景

线上部分服务希望用golang替换PHP，承接数据中间层，提高并发。  但是经过性能压测， 发现在同等配置下， PHP的QPS远高于golang , 且golang的web服务在较低的并发下， 打高CPU， 因此需要借助性能分析工具，对CPU打高的原因，一探究竟。已正golang清白。


## 服务简介

目前golang承接的业务逻辑主要有以下逻辑

1. 查询redis， 通过hgetall (redis key较大)
2. 拼装Redis， 其中一个字段，需要进行`bizp2`解密(PHP在设置redis key时，采用的`bzcompress`加密)
3. json形式返回客户端


显而易见， 基本没有什么逻辑。 但是CPU被打高， 一定是上面的流程出现了问题。 我们猜测可能是 `bizp2`解密 或者 golang`json编码`的问题。 基于猜测，我们希望能有莫得见，看得着的性能指标去验证。 我们这里使用 `pprof`工具。

## pprof


### 代码中埋点方式

```go
import "runtime/pprof"
// ...
// 指定生成的数据文件
cpuProfile, _ := os.Create("cpu_profile")
defer cpuProfile.Close()

// 开始进行CPU性能分析
pprof.StartCPUProfile(cpuProfile)

// 停止性能分析
defer pprof.StopCPUProfile()
```

内存性能分析方式

```go
... (业务逻辑)
// 需放到进程结尾处，放到程序开头，不生效
// 内存分析
//memProfile, _ := os.Create("mem_job_profile")
//defer memProfile.Close()
//runtime.GC()
//pprof.WriteHeapProfile(memProfile)
```


### 将性能指标可视化

```
go tool pprof -http=":8081" [binary] [profile]
```

会自动在浏览器打开 `http://localhost:8081/ui/`, 数据可视化。

我们可以通过， 图， 火焰图， top图， peek图进行各个类别的分析, 更倾向于使用或沿途分析问题


#### Top 概念截图

* flat: 是指该函数执行耗时, 程序总耗时 570ms, main.NewNode 的 200ms 占了 35.09%
* sum: 当前函数与排在它上面的其他函数的 flat 占比总和, 比如 35.09% + 12.28% = 47.37%
* cum: 是指该函数加上在该函数调用之前累计的总耗时, 这个看图片格式的话会更清晰一些.


#### 流程图 Graph

从上到下， 展示系统调用时间,时间依次递减


方块越大，占用时间越长

#### 火焰图 Flame Graph

总体思想： 统计函数调用栈的总耗时， 聚合函数内部的方法的调用总时间。 能非常容易发现是具体哪个方法调用花费时间价高。

本次问题发现， 即通过火焰图，发现解压缩bzip2 占用较高， 并涉及到大内存分配


### 参考资料

1. https://xguox.me/go-profiling-optimizing.html/
2. https://cizixs.com/2017/09/11/profiling-golang-program/

