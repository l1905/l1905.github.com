---
title: golang使用Mongo驱动
description: golang使用Mongo驱动
categories:
 - golang
tags:
 - golang
 - mongo
---


## 背景

因为PHP进程持久化mongo连接池，导致Mongo连接池太多， 因此考虑使用基于golang的web服务，进行读写mongo。

这里考虑使用mongo的官方golang驱动来实现


## 实践过程中的关键点


### Mongo底层存储为BSON格式

`bson` 是一种类似`json`的格式， 但是比 `json`更节省存储空间。golang使用了下面四种结构体来实现`bson`格式


* D: A BSON document. This type should be used in situations where order matters, such as MongoDB commands. 类似json的对象，但是是有序的
 
* M: An unordered map. It is the same as D, except it does not preserve order. 类似D，但是无序

* A: A BSON array. 类似json中的列表，类似[1,2,2,2,3]

* E: A single element inside a D. 单个元素在D里面， 这里使用非常少


上述概念理解完后， 即可以快速操作`CURD`实现。

```
//插入数据
data := bson.D{
        {"message", msg},
        {"creation_time", setCreationTime},
    }

    result, err := collection.InsertOne(ctx, data)

//查找数据
filter := bson.D{
        {
            "_id", bson.D{
                {"$in", objectIDList},
            },
        },
    }
cur, err := collection.Find(ctx, filter)    
//更新数据

update := bson.D{
        {"$set", bson.D{
            {"message", newMsg},
        }},
    }
updateResult, _ := collection.UpdateOne(ctx, filter, update)
    
```

BSON 可以方便的和结构体， json 转换，方式为

定义结构体和json, bson的转换关系

```
type Notice struct {
    ID         primitive.ObjectID   `json:"_id"  bson:"_id"`
    Message       string   `json:"message" bson:"message"`
    CreationTime     time.Time `json:"-" bson:"creation_time"`

    //格式化数据
    FormatCreationTime string `json:"creation_time"`

}

row := Notice{}
//解析为结构体
err := cur.Decode(&row)
//解析为json
data, _ := json.Marshal(row)
```

### 主键_ID读取

Mongo底层存储的是 `primitive.ObjectID`

`primitive.ObjectIDFromHex(msgID)` 转存字符类型`_id` 为 `objectID`, 查询只能使用 `objectID`

`obj.Hex()` 将`objectID`转换为字符串类型。


### 存储时间转换


目前`mongo`支持根据时间来筛选， 但是必须是`UTC`格式。

我们看下常用的字符串时间转换形式

`time.Now().UTC()` 先将当前时间转换为本时区时间，然后再将时间转为UTC时间

`time.Parse(layout, creationTime)` 将传入的时间字符串默认直接转换为UTC时间，但是我们的需求是 先将时间转换为本时区时间， 再转换为UTC时间。


`time.ParseInLocation(layout, creationTime, time.Local)`, 这一步即对 `parse()`的完善， 转换为指定时区， 再进行转换到`UTC`时区


这里再额外总结golang 时间转换方式

```

#将字符串转成当前时区结构体 YYYY-mm-dd HH:ii:ss
layout := "2006-01-02 15:04:05"
time.ParseInLocation(layout, creationTime, time.Local)

#将当前时间结构体转换为  YYYY-mm-dd HH:ii:ss
{CreationTime}.In(time.Local).Format("2006-01-02 15:04:05")

#获取unix时间戳
{CreationTime}.Unix()

#转成纳秒时间戳
{CreationTime}.UnixNano()

#将unix时间戳转成time结构体
t := time.Unix(1494505756, 0)


```


### 参考资料

https://docs.mongodb.com/ecosystem/drivers/go/
