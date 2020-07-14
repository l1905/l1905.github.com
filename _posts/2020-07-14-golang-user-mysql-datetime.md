---
title: golang-mysql时间转化
description: golang-mysql时间转化, datetime时间类型转化
categories:
 - golang
 - mysql
tags:
 - golang
 - mysql
 - datetime
---


## 背景

目前`golang` 操作mysql， 通过原生sql进行操作。

当mysql数据表中字段类型为`datetime`即 `Y-m-d H:i:s`类型
golang结构体对应字段为string， 解析格式跟数据库类型不一致。

mysql中存储 ` 2012-03-07 13:02:47`

golang解析为 `2012-03-07T13:02:47+08:00`, 额外添加了`T``+` 还有毫秒字段。


如果golang结构体对应字段为time.time,  序列化输出json时， 字段类型也是跟上面一致``2012-03-07T13:02:47+08:00``

两种方式，都没有达到自己的预期。

所以妥协方案是， 字段定义为time.time类型 字段A， 新添加一个字段B，定义为string类型， 通过A的time.format中间转化一次， 相对比较麻烦。

```
s.A := time.Now()

s.B := s.A.format("2006-01-02 15:04:05")
```

所以我这次主要解决上面遇到的麻烦事

### 分析具体问题

为什么结构体中 字段`string` 类型时间， 和数据库不吻合？

我们发现，在连接mysql时， 有一个`parseTime`参数

```
fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=True&loc=%s&readTimeout=%dms&writeTimeout=%dms&timeout=%dms",
		"db", "pwd", "db", 3306, "DB", loc, 1000, 1000, 1000)
```

翻阅官方文档

```
Type:           bool
Valid Values:   true, false
Default:        false

```

`parseTime=true` 改变 数据库`DATE`， `DATETIME`值的输出类型， 从 `[]byte/string`类型被改变为`time.Time`类型， 像`0000-00-00 00:00:00`会被转化为time.Time的零值

再看下我们的配置， `parseTime`参数为true， 即我们的`datetime`被转化`time.time`,

但我们定义的是字符串， 即又被转化为了字符串， 具体转化方案是这样：

`DATETIME(Mysql)` ----->`time.Time`(golang) ------> `string(golang)`, 即这里的time.time到字符串 这一步出现了转化差异。


如果将`parseTime`参数置为false， 转化方式为

`DATETIME(Mysql)` ----->`string(golang)` ， 转化无问题。

在大部分程序中， 我们还是需要将时间类型设置为time.Time, 方便我们进行时间到字符串，时间比较等一些操作。

所以我们的问题区间缩小了， 变成了

### `time.Time`类型，怎么无差异的序列化为`json`?

已知，在json序列化时， 如果字段 拥有`MarshalJSON`, 则使用该方法序列化此字段。

但是`time.Time`是内置函数，我们没办法加上自己的`MarshalJSON`。 因此我们考虑别名定义方式， 给自己的变量类型添加 `MarshaJSON`方法。

具体操作如下

```
type ztime time.Time
func (t ztime) MarshalJSON() ([]byte, error) {
	var stamp = fmt.Sprintf("\"%s\"", time.Time(t).Format("2006-01-02 15:04:05"))
	return []byte(stamp), nil
}

// 获取time.Time类型，方便拓展方法
func (t ztime) Time() (time.Time) {
	return time.Time(t)
}

// 格式化
func (t ztime) Format() (string) {
	return  time.Time(t).Format("2006-01-02 15:04:05")
}
// 简单格式化
func (t ztime) FormatSimple() (string) {
	return  time.Time(t).Format("2006-01-02")
}

使用方式

type Account struct {
	CreationDate ztime  `gorm:"column:creation_date" json:"creation_date"`
}

```

这样就提高了我们转化时间字段的效率。


## 参考

1. https://github.com/go-sql-driver/mysql
