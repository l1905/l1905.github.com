---
title: golang的mysql-gorm操作指南
description: golang的mysql-gorm操作指南
categories:
 - golang开发
tags:
 - golang
 - golang-mysql-gorm
---


## golang的gorm操作指南

### 背景

首次尝试使用golang下的`orm`类库 `gorm`, 做一下记录

### 分析流程：


#### 构成

基于原有的go-sql-driver/mysql golang的mysql驱动做的封装， 但是兼容多种数据库源， 比如postgresl, sqlite等常见操作。


#### 调用方式

链式调用结构， 比如可以这样调用 db.select().find(), 和一般的orm操作方式基本一致。


#### 与model映射

可以通过映射关系，数据库查询字段和golang的数据结构体的字段进行一一映射绑定， 方便转化为结构体。

比如

```
type XzAutoServerConf struct {
    GroupZone string  `gorm:"column:group_zone"`
    ServerId  int `gorm:"column:server_id"`
    OpenTime  string `gorm:"coloumn:open_time"`
    ServerName string `gorm:"coloumn:server_name"`
    Status    int   `gorm:"coloumn:status"`
    Username string `gorm:"coloumn:username"`
}
```

gorm内部有一些约定的规则， 来简化日常操作， 只需学习一次记住，即可。

基本模型定义`gorm.Model`，包括字段`ID`，`CreatedAt`，`UpdatedAt`，`DeletedAt`，你可以将它嵌入你的模型，或者只写你想要的字段。

```
//gorm内部的Model基本模型的定义
type Model struct {
  ID        uint `gorm:"primary_key"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt *time.Time
}
```

//在我们的数据库，添加字段 `ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt`
type User struct {
  gorm.Model
  Name string
}

// 如果我们的结构体，只需要字段 `ID`, `CreatedAt`
type User struct {
  ID        uint
  CreatedAt time.Time
  Name      string
}

#### 表名复数

默认表名是结构体名称的复数形式

即我们写常用gorm语句时， 通过不会指定操作的表名， gorm会聪明的，认为我们操作的结构体的名称+s即是表名， 我认为是小聪明吧。刚开始入手时， 有些疑惑

```
type User struct {} // 默认表名是`users`

// 设置User的表名为`profiles`
func (User) TableName() string {
  return "profiles"
}

func (u User) TableName() string {
    if u.Role == "admin" {
        return "admin_users"
    } else {
        return "users"
    }
}

// 全局禁用表名复数， 即将结构体名作为表名
db.SingularTable(true) // 如果设置为true,`User`的默认表名为`user`,使用`TableName`设置的表名不受影响
```


我们可以修改全局的默认表名结构，比如统一给他们加前缀，或者后缀， 方式如下


```
您可以通过定义DefaultTableNameHandler对默认表名应用任何规则。
gorm.DefaultTableNameHandler = func (db *gorm.DB, defaultTableName string) string  {
    return "prefix_" + defaultTableName;
}
```

#### 列名映射

列名是字段名的蛇形小写.

更符合我们命名习惯， 即代码驼峰映射成 蛇形mysql数据字段，但是，新手依然会容易误导

```
type User struct {
  ID uint             // 列名为 `id`
  Name string         // 列名为 `name`
  Birthday time.Time  // 列名为 `birthday`
  CreatedAt time.Time // 列名为 `created_at`
}

// 重设列名
type Animal struct {
    AnimalId    int64     `gorm:"column:beast_id"`         // 设置列名为`beast_id`
    Birthday    time.Time `gorm:"column:day_of_the_beast"` // 设置列名为`day_of_the_beast`
    Age         int64     `gorm:"column:age_of_the_beast"` // 设置列名为`age_of_the_beast`
}


```

####  字段ID为主键

```


type User struct {
  ID   uint  // 字段`ID`为默认主键
  Name string
}

// 使用tag`primary_key`用来设置主键
type Animal struct {
  AnimalId int64 `gorm:"primary_key"` // 设置AnimalId为主键
  Name     string
  Age      int64
}
```
我的角度，禁止代码直接修改表结构，容易逃过审计操作记录。

####  字段CreatedAt用于存储记录的创建时间, 默认操作， 建议慎用

```
//创建具有`CreatedAt`字段的记录将被设置为当前时间
db.Create(&user) // 将会设置`CreatedAt`为当前时间

// 要更改它的值, 你需要使用`Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

####  字段`UpdatedAt`用于存储记录的修改时间， 默认操作建议慎用


```
//保存具有UpdatedAt字段的记录将被设置为当前时间
db.Save(&user) // 将会设置`UpdatedAt`为当前时间
db.Model(&user).Update("name", "jinzhu") // 将会设置`UpdatedAt`为当前时间
```

####  字段DeletedAt用于存储记录的删除时间，如果字段存在(不建议使用)

删除具有DeletedAt字段的记录，它不会冲数据库中删除，但只将字段DeletedAt设置为当前时间，并在查询时无法找到记录，


####  回调操作(Callbacks)：

即将curd事务化， 并且在事务过程中，预先埋好钩子hook， 用户实现钩子hook, 比如做一些特殊检查， 失败，即回滚整个事务。

操作复杂， 不建议使用


## 总结：

#### 优点：
1. 功能强大， 对golang原有mysql-driver，做了非常强大的上层封装

#### 缺点：
1.  不够精简，稍显复杂
2. 如果使用ORM操作，只能通过日志形式，获得原生sql语句， 无法进行埋点，进行调用链追踪。

实践代码如下：

```
package main

import (
    "fmt"
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/mysql"
    "log"
)

/**
定义连接实例
 */
func MyConn(user, password,host, db, port string) *gorm.DB {
    connArgs := fmt.Sprintf("%s:%s@(%s:%s)/%s?charset=utf8&parseTime=True&loc=Local",  user,password, host, port,db )
    db_instance, err := gorm.Open("mysql", connArgs)
    if err != nil {
        log.Fatal(err)
    }
    //查询单表， 禁止复数表存在
    db_instance.SingularTable(true)

    return db_instance
}

//实际数据表
/*CREATE TABLE `xz_auto_server_conf` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`group_zone` varchar(32) NOT NULL COMMENT '大区例如：wanba,changan,aiweiyou,360',
`server_id` int(11) DEFAULT '0' COMMENT '区服id',
`server_name` varchar(255) NOT NULL COMMENT '区服名称',
`open_time` varchar(64) DEFAULT NULL COMMENT '开服时间',
`service` varchar(30) DEFAULT NULL COMMENT '环境，test测试服，formal混服，wb玩吧',
`username` varchar(100) DEFAULT NULL COMMENT 'data管理员名称',
`submit_date` datetime DEFAULT NULL COMMENT '记录提交时间',
`status` tinyint(2) DEFAULT '0' COMMENT '状态，0未处理，1已处理，默认为0',
PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8;*/

//定义表结构体struct
type XzAutoServerConf struct {
    GroupZone string  `gorm:"column:group_zone"`
    ServerId  int `gorm:"column:server_id"`
    OpenTime  string `gorm:"coloumn:open_time"`
    ServerName string `gorm:"coloumn:server_name"`
    Status    int   `gorm:"coloumn:status"`
    Username string `gorm:"coloumn:username"`
}

func main() {
    user := "root"
    password := "l1905"
    host := "127.0.0.1"
    port := "3306"
    db_name := "UserDB"

    //建立连接
    db := MyConn(user,password,host, db_name, port)
    //打开调试模式，在日志中，能看到对应的执行sql
    db.LogMode(true)
    // 关闭数据库链接，defer会在函数结束时关闭数据库连接
    defer db.Close()

    //orm查询方式
    orm_op(db)

    //原生查询方式
    //raw_op(db)

}

func orm_op(db *gorm.DB)  {
    var rows []XzAutoServerConf

    //新增
    add_data := XzAutoServerConf{GroupZone: "20", ServerId: 811}
    db.Create(&add_data)

    //删除操作
    err := db.Model(&rows).Where("server_id=?", 81).Delete(&XzAutoServerConf{}).Error

    //查询操作
    db.Table("xz_auto_server_conf").Where("status=?", 0).Select([]string{"group_zone", "server_id", "open_time", "server_name", "username"}).Find(&rows)
    fmt.Println(rows)

    //排序order
    db.Table("xz_auto_server_conf").Where("status=?", 0).Select([]string{"group_zone", "server_id", "open_time", "server_name", "username"}).Order("server_id ASC").Find(&rows)

    //count操作
    var count int
    db.Table("xz_auto_server_conf").Where("status=?", 0).Count(&count)
    fmt.Println(count)


    //group操作
    type Result struct {
        ServerId  string
        Num int64
    }
    var new_results []Result
    db.Table("xz_auto_server_conf").Where("status=?", 0).Select("server_id, count(*) as num").Group("server_id").Scan(&new_results)
    fmt.Println("%+v", new_results)

    //更新操作
    err = db.Model(&rows).Where("server_id=?", 80).Update("status", 0).Error
    if err !=nil {
        fmt.Println(err)
    }

}

/**
类原生sql
 */
func raw_op(db *gorm.DB) {

    //新增
    sql_insert := `INSERT INTO xz_auto_server_conf (group_zone, server_id, open_time, server_name, status, username)
VALUES ('20', '81', 'Skagen 21', 'Stavanger', 2, 'Norway');`

    err := db.Exec(sql_insert).Error
    if err != nil {
        log.Fatal(err)
    }
    //exec操作的过程当中， 主要做哪些事情。
    //todo 获取最后一次插入ID

    //删除
    sql_delete := `delete from xz_auto_server_conf where server_id = 81 `
    err  = db.Exec(sql_delete).Error
    if err !=nil {
        log.Fatal(err)
    }

    //update操作
    err  = db.Exec(`update xz_auto_server_conf  set server_name="花好月圆" where server_id = 1`).Error
    if err !=nil {
        log.Fatal(err)
    }

    //update操作
    sql := fmt.Sprintf(`update xz_auto_server_conf  set server_name="花好月圆" where server_id = %d`, 1 )
    err = db.Exec(sql).Error

    if err !=nil {
        log.Fatal(err)
    }

    //查询多行
    var rows []XzAutoServerConf
    sql_query := fmt.Sprintf(`select* from  xz_auto_server_conf   where server_id = %d`, 80 )
    db.Raw(sql_query).Scan(&rows)

    //查询一行
    var row XzAutoServerConf
    db.Exec(sql_query).Row().Scan(&row)

    fmt.Println("hello world")
    //fmt.Printf("%+v", rows)

    fmt.Printf("%+v", row)
    //fmt.Println(rows[0].ServerName)
}

```

## 参考链接


1. https://jasperxu.github.io/gorm-zh/development.html
2. http://gorm.io/docs/sql_builder.html

