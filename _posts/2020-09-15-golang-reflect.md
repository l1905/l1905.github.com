---
title: golang反射理解
description: golang反射理解
categories:
 - golang
 - 反射
tags:
 - golang
 - 反射
---

## golang反射

### 背景：

前两天组内同学分享反射， 正好，自己在这块有知识盲点， 听完分享， 结合自己操作一些demo， 现在写下自己的体会。


### 反射

在不同语言中都会遇到反射的概念， 反射，即是获取变量自身的结构类型， 根据自身的结构类型，做相应的操作。

在golang中， 反射相关操作集中在 `reflect`包中。 反射常用的场景是： 
1. 序列化解码(json)
2. 打印输出(print)

常用的两种反射类型对象

1. reflect.Type, 变量类型对象
2. reflect.Value， 变量值对象

因为一个变量(比如 data int), 一定包含确定的类型(int, string, slice），同样也包含确定的值大小。  

通常使用 `refleact.Value`对象， 因为可以根据Value对象获取到对应的Type对象

```
a := reflect.ValueOf(m)
a.Type() // 这里即是Type类型
```

通常会使用 `reflect.Type`对象的`kind`方法， 此方法将会获取到字段类型， 这里是获取的原始类型，比如` type myInt int` 这种， 最终会获取`int`类型， 而`name`方法， 则可以获取表层的类型名称。


reflect.Value类型有以下几种比较有意思的使用

1. Type() // 获取Type对象
2. CanSet() // 字段是否支持设置方法， 像有些复制变量，则不支持直接应用set方法
3. interface() // 将Value对象重新退化为interface 对象， 当赋值获取打印时， 会有此需求。
4. Elem(), 解引用， 比如 *int类型， 调用解应用后， 会转化为 int类型，
5. Addr(), 取地址，比如是int类型， 调用后，将会获取其指针地址类型 *int


参考文章：

1. https://blog.golang.org/laws-of-reflection
2. https://github.com/owenliang/go-share-reflect

