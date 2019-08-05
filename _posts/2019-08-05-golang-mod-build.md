---
title: golang module 开发基本流程
description: golang module 开发基本流程
categories:
 - golang开发
tags:
 - golang
 - mod
 - 迁移文章
---


# golang module 开发基本流程


## 背景

目前在mac上开发, golang版本为 `go version go1.12.5 darwin/amd64`

需要提前设置全局变量

```
export GO111MODULE=on
```

这是自己总计的开发流程， 可能跟其他地方描述有出入，需灵活参考


## 操作流程

### 创建module项目

即初始化 仓库， 我的申请仓库地址

`https://github.com/l1905/golang_module_demo`


### 本地clone 下来，进行项目初始化

项目初始化不需要在`$GOPATH`目录中， 任意其他目录即可

```
go mod init github.com/l1905/golang_module_demo
```
这里的`github.com/l1905/golang_module_demo` 对应我们的包名，也是我们的github地址

这里会新建 `go.mod`文件，  大致输出如下

```
module github.com/l1905/golang_module_demo

go 1.12
```

### 初始化文件目录， 引入三方依赖包。


```
.
├── README.md
├── cmd
│   └── main.go //启动文件, 放main函数
├── country
│   ├── city.go //country包
│   └── city_test.go //测试文件
├── go.mod
├── go.sum
└── talk
    ├── speak.go
    └── speak_test.go
```

### speak.go 文件中 使用第三方包

```

package talk

import "rsc.io/quote"

func SayHello() string {
    quote.Hello()
    return "Hello, world."
}

```

### 通过 `go build ` 或者 `go test ` ，引入三方包 

```
go test

如果报以下错误
`can't load package: package github.com/l1905/golang_module_demo: unknown import path "github.com/l1905/golang_module_demo": cannot find module providing package github.com/l1905/golang_module_demo`

可能是需要引入环境变量 
export GO111MODULE=on
```

这时候 go会自动帮我们把三方包下载到 `$GOPATH`目录中， 具体路径为 ` $GOPATH/pkg/mod`, 目录内容如下

```
.
├── cache
│   ├── download
│   ├── lock
│   └── vcs
├── github.com
│   └── l1905
├── golang.org
│   └── x
└── rsc.io
    ├── quote
    ├── quote@v1.5.2
    ├── sampler@v1.3.0
    ├── sampler@v1.3.1
    └── sampler@v1.99.99
```

### GOLANG设置

如果goland需要识别 `module`,  则需要去 `Preferences>Go->Go Modules(vgo)` 勾选，启用 gomodule, 否则的话， 无法引入本地的包， main.og 如下

```
package main

import "fmt"
import "github.com/l1905/golang_module_demo/talk"

func main() {
    fmt.Println(talk.SayHello())
}

```

### 可以将依赖包 都到到当前文件夹`vendor`中，

```
go mod vendor
```

如果依赖`vendor`目录中的三方包， 则需要 编译时 指定

```
go build -mod vendor cmd/main.go
```



### 开发完成，提交到 github上， 在github上打tag `v0.0.1`

下面主要来测试怎么调整依赖包



### 新建测试项目(`golang_module_test`)， 引入我们刚才的包(`golang_module_demo`)

```
package main

import (
    "fmt"

)
import "github.com/l1905/golang_module_demo/talk"

func main() {
    fmt.Println(talk.SayHello())
}
```


通过 `go mod` 查看所有版本

```
> go list -m -versions github.com/l1905/golang_module_demo

`github.com/l1905/golang_module_demo v0.0.1 v0.0.2` 


```

### 我们发现已经有新的版本发布， 我们指定更新这个包文件

```
go get github.com/l1905/golang_module_demo@v0.0.2
```

发现包依赖文件已经调整，

<font color="#dd0000">重点：go get 默认拉取latest 最新 V1.x.x 或者 最新的 v0.x.x, 不会自动拉取v2, 或者 v3</font>: 

### 更新包 以及间接包到最新版本

```
go get -u github.com/l1905/golang_module_demo
```


### 列出所有的依赖包

对比同样可以参考 `go.mod`

```
➜  golang_module_test git:(master) ✗ go list -m all
github.com/l1905/golang_module_test
github.com/l1905/golang_module_demo v1.0.5
golang.org/x/crypto v0.0.0-20190605123033-f99c8df09eb5
golang.org/x/net v0.0.0-20190603091049-60506f45cf65
golang.org/x/sync v0.0.0-20190423024810-112230192c58
golang.org/x/sys v0.0.0-20190602015325-4c4f7f33c9ed
golang.org/x/text v0.3.2
golang.org/x/tools v0.0.0-20190606050223-4d9ae51c2468
rsc.io/quote v1.5.2
rsc.io/sampler v1.99.99
```
### 清除无用依赖

```
go mod tidy
```

### 引入代理 国内源

```
参考文章：https://goproxy.io/

# Enable the go modules feature
export GO111MODULE=on
# Set the GOPROXY environment variable
export GOPROXY=https://goproxy.io

```

### 参考文章

1. https://www.4async.com/2019/03/using-go-modules/
2. https://blog.golang.org/using-go-modules
3. https://github.com/golang/go/wiki/Modules#how-do-i-use-vendoring-with-modules-is-vendoring-going-away
