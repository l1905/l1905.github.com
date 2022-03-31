---
title: golang泛型白话篇
description: golang泛型白话篇
categories:
 - 泛型
tags:
 - golang开发
 - 泛型
---


# golang泛型



## 背景



目前golang中，最新版本1.8版本中新增了方法泛型。泛型的合理使用， 能减少我们写很多类似的代码。

一些成熟的语言， 比如JAVA支持泛型(接口泛型、类泛型、方法泛型)， 或者模版， 可以减少写重复的代码，这里做一些泛型的基础介绍。

## 一、例子先行





业务环境经常有场景：判断某个元素 是否在数组或列表中。

比如判断某个 数字， 是否在 列表中。

```
func IntContains(rows []int, key int) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```



比如 判断某个字符串， 是否在列表中



```
func StrContains(rows []string, key string) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```



我们发现， IntContains 和 StrContains的代码逻辑结构，基本一致， 只有传参不一样。 我们可能希望， 如果既能传 【数字数组、数字元素】【字符串数组、字符串元素】就好了。

但现实情况是， golang是强类型语言， 传参类型需要固定。 

有的同学可能会说， 可以传万能类型interface, 比如传数组interface类型 \[]inerface{}，那我们可以试下



```
func InterfaceContains(rows []interface{}, key interface{}) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```

前台研发后端内部空间 > golang泛型实践 > image2022-3-31_8-56-27.png



会报错提示， \[]int{1,2,3} \[]int类型不能转换为\[]interface{}类型。所以此路不通。

那我们的最佳方案呢？

即是写多个方法IntContains、 StrContains、FloatContains....



## 二、引入泛型



在新的1.8版本发布，增加泛型后， 我们可以怎么使用。我们用泛型重写我们刚才的方法。

```
func zdmContains[K int|string](rows []K, key K) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```

我们发现，在方法名字后面， 出现了新语法中括号\[], 中括号里有个变量K， 其类型为 int 或者string,  也是新语法， 这个变量必然既可以是int类型，也可以是string类型， 神奇！

我们继续看方法的参数列表， 发现 参数的变量类型变成了， 我们前面中括号中定义的变量K了， 这样就可以既支持int类型，也可以支持string类型了嘛?

```
fmt.Println(zdmContains[string]([]string{"1", "2", "3"}, "1"))
fmt.Println(zdmContains[int]([]int{1, 2, 3}, 6))

true
false

```



验证后确实如此， 但我们使用的方法的时候， 也添加了一个中括号， 中括号里为数组的类型。 每次调用方法的时候， 都需要加上数组的类型， 稍显麻烦， 有没有更好的方案？有的， 在调用的时候， 可以省略数组类型， 编译器会自动类型推断。



```
fmt.Println(zdmContains([]string{"1", "2", "3"}, "1"))
fmt.Println(zdmContains([]int{1, 2, 3}, 6))

```

这样，用起来，就会非常的舒服。 如果我们还想要增加支持float64类型呢？ 简单， K类型后面用或｜符号追加 float64类型即可



```
func zdmContains[K int|string|float64](rows []K, key K) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```



如果我们还想增int32、int8类型呢？

那我们继续在方法中括号中，追加其他类型即可

```
func zdmContains[K int|string|float64|int32|int8](rows []K, key K) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```

但参数太长了， 如果写多个泛型方法，都要写这么一串方法，不通用， 这该怎么解决？

那我们把那一长串 `“int|string|float64|int32|int8“`抽出来，共用一下， 看来是个好主意，

```
type Itype interface {
   int | string | float64 |int32 |int8
}

func zdmContains[K Itype](rows []K, key K) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}

```

如果我们并不关心参数类型呢， 希望能传任何参数， 自己枚举类型，总会有漏的情况？

这里引入了 comparable关键字，是一种特殊的变量类型， 它代表， 一切可以进行比较（可以进行==或者！=操作)的类型。 我们改写下我们的方法，和调用

```
func zdmContains[K comparable](rows []K, key K) bool {
   for _, item := range rows {
      if item == key {
         return true
      }
   }
   return false
}


fmt.Println(zdmContains([]string{"1", "2", "3"}, "1"))
fmt.Println(zdmContains([]int{1, 2, 3}, 6))

```

现在使用泛型非常熟练了， 那换一种复杂的场景做个练习。

我们期望计算map中value之和， value是数字类型。 map比如是map\[int]int, 或者map\[string]int64类型。 我们仍然可以先写两个常规方法， 看一下是否有相似之处



```
func mapSum1(data map[string]int) int  {
   var s int
   for _, v := range data {
      s += v
   }
   return s
}

func mapSum2(data map[int]int64) int64  {
   var s int64
   for _, v := range data {
      s += v
   }
   return s
}

```

发现传参中， map的key类型是一种变量(string,int)， value类型是另一种变量(int, 64), 返回值同value变量类型一致。  我们再进行以下分析。

key并没有参与到计算， 可以定义为comparable 任意类型

value和返回值 因为需要参与计算 +=， 所以得是能支持计算的数字类型， 



并且泛型中多个参数写？

```
func zdmMapSum[K comparable, V int | float64 | int32](data map[K]V) V {
   var s V
   for _, v := range data {
      s += v
   }
   return s
}

```



```
fmt.Println(zdmMapSum(map[string]int{"a": 1, "b": 2}))
fmt.Println(zdmMapSum(map[int]int32{101: 1, 102: 2}))

```





这里的V 应该可以通过抽象出来定义一种类型来表达， 这里有兴趣的可以实践操作。



## 三、官方建议



建议在合理的地方使用

## 四、参考文档

https\://www.sunzhongwei.com/go-version-118-new-features-and-upgrade-steps?from=bottom

https\://go.dev/doc/tutorial/generics

https\://juejin.cn/post/7042344142281637895






