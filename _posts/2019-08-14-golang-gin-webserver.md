---
title: golang框架gin运行源码分析
description: golang框架gin运行源码分析
categories:
 - golang开发
tags:
 - golang
 - gin框架
---


## 解析`gin`框架中数据流转原理

### 分析流程

我们已经了解了http包中server的启动方法， 现在我们分析`gin`处理流程

gin框架号称 路由提速了40倍， 所以，到底他是哪里快呢？ 


我们还是看下他启动服务时做哪些事情

```
package main

import "github.com/gin-gonic/gin"

func main() {
    #生成gin实例
    r := gin.Default()
    #注册路由关系，定义具体的路由处理方法
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    #启动服务
    r.Run() // listen and serve on 0.0.0.0:8080
}
```

和我们刚才看到的http_server非常像， 但是路由管理器这块比较强大

较高级的语法糖， 可以直接指定HTTP方法名称， 内部封装json响应实现。

我们继续从启动方法`r.Run()`入手， 看下他的内部实现逻辑

```
#gin.go
func (engine *Engine) Run(addr ...string) (err error) {
    defer func() { debugPrintError(err) }()
    #解析地址
    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    #调用http包的ListenAndServe方法
    err = http.ListenAndServe(address, engine)
    return
}
```
我们可以发现， 这里仍然使用的是http包的`ListenAndServe`方法， 和我们上一章调用类似，主要区别是其第二个参数， 这里是gin框架生成gin实例。

并且我们知道， 第二个参数的主要意义是， 关联路由管器对象， 即gin框架承接了url和处理函数的路由管理。
其他http请求这一套， 继续复用原生包的`http`接口

我们截取下http包的server.go的代码
```
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    #获取路由管理对象, 这里即是我们调用`ListenAndServe`的第二个参数
    handler := sh.srv.Handler
    #如果没有设置，则使用默认的defaultServerMux
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    #进入路由管理对象的ServerHttp方法中，进行路由分发
    handler.ServeHTTP(rw, req)
}
```

从`handler.ServerHTTP`方法开始，gin框架开始接管http请求

因此我们看下gin.go中的`ServerHTTP`方法

```
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    #为减少gc重复回收， 这里使用sync.pool管理自定义Context对象
    c := engine.pool.Get().(*Context)
    c.writermem.reset(w)
    c.Request = req
    c.reset()

    #分发处理请求
    engine.handleHTTPRequest(c)

    #将Context对象放回
    engine.pool.Put(c)
}
```

我们看下上面代码， 主要做以下事情

1. 为减少gc重复回收， 这里使用sync.pool管理自定义Context对象
2. 将请求reqeust数据copy到Context对象中， 通过Context进行管理
2. 调用`engine.handleHTTPRequest` 进行路由分发

在这里引入自定义的`Context`对象， 其主要是用来管理数据流转过程时的，上下文数据， 比如response， request， 请求参数params,路径fullpath, 查询缓存, 错误管理， 主要的目的是:避免重复复制数据。 保证数据的一致性。这是gin最重要的数据结构体


我们着重看下 `engine.handleHTTPRequest`分发内部逻辑

```
func (engine *Engine) handleHTTPRequest(c *Context) {
    httpMethod := c.Request.Method
    rPath := c.Request.URL.Path
    unescape := false
    if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
        rPath = c.Request.URL.RawPath
        unescape = engine.UnescapePathValues
    }
    #获取请求路径
    rPath = cleanPath(rPath)

    // Find root of the tree for the given HTTP method
    t := engine.trees
    for i, tl := 0, len(t); i < tl; i++ {
        if t[i].method != httpMethod {
            continue
        }
        root := t[i].root
        // Find route in tree
        #根据路径，请求参数，找到对应的 路由处理函数
        value := root.getValue(rPath, c.Params, unescape)

        if value.handlers != nil {
            #更新Context对象属性，将路由地址管理的多个路由函数都交给Context管理
            c.handlers = value.handlers
            c.Params = value.params
            c.fullPath = value.fullPath
            #递归的执行关联的handler方法
            c.Next()
            c.writermem.WriteHeaderNow()
            return
        }
        if httpMethod != "CONNECT" && rPath != "/" {
            if value.tsr && engine.RedirectTrailingSlash {
                redirectTrailingSlash(c)
                return
            }
            if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
                return
            }
        }
        break
    }

    if engine.HandleMethodNotAllowed {
        for _, tree := range engine.trees {
            if tree.method == httpMethod {
                continue
            }
            if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
                c.handlers = engine.allNoMethod
                serveError(c, http.StatusMethodNotAllowed, default405Body)
                return
            }
        }
    }
    c.handlers = engine.allNoRoute
    serveError(c, http.StatusNotFound, default404Body)
}
```

上述代码完成完整路由，进行数据的最终响应， 我们把核心代码摘出来。

```
#根据路径，请求参数，找到对应的 路由处理函数， 和我们上一章节实现类似
value := root.getValue(rPath, c.Params, unescape)

#递归的执行关联的handler方法
c.Next()
```

我们先看 `root.getValue`方法， 其主要进行 路由操作，这里是采用的`基树Radix_tree`算法，实现较复杂，不再这里详细展开了。


我们再看下`c.Next()`方法，这个方法的核心，主要是方便接入中间件(Middleware)，使得代码模块化操作。

我们看下Next的具体实现

```
func (c *Context) Next() {
    c.index++
    for c.index < int8(len(c.handlers)) {
        #执行关联的中间件方法或者 实际路由处理函数
        c.handlers[c.index](c)
        c.index++
    }
}
```

描述相对抽象， 我们新建一个demo例子

```
import (
    "fmt"
    "github.com/gin-gonic/gin"
    "log"
    "time"
)

func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()

        //请求处理前 Set example variable
        c.Set("example", "12345")

        //请求下一个中间件
        c.Next()

        //请求处理之后
        latency := time.Since(t)
        log.Print(latency)

        // access the status we are sending
        status := c.Writer.Status()
        log.Println(status)

    }
}

func main() {
    r := gin.New()

    //引入Logger()中间件
    r.Use(Logger())

    r.GET("/test", func(c *gin.Context) {
        example := c.MustGet("example").(string)
        //打印关联的全部处理路由函数
        fmt.Println(c.HandlerNames())
        // it would print: "12345"
        log.Println(example)
    })

    // Listen and serve on 0.0.0.0:8080
    r.Run(":8080")
}

```
上面例子中， 我们引入一个Logger中间件， 在路由函数中打印全部路由函数名称，输出如下


```
[main.Logger.func1 main.main.func1]

```

即能看到，这里将我们新加入的Logger中间件，转换成了句柄函数，即`/test`URI对应的路由函数有两个， 这两个会按照先后顺序， 依次执行。

即先执行 `main.Logger.func1`, 后执行 `main.main.func1`, 结合我们上面的`Next`方法实现， 我们就能清楚的知道，其调用关系

实现了`Next`方法的伪代码，加深理解：

```
最外层index初始化为-1
    进入Next
        index++= 0
        调用handler方法，即Logger方法(如果index<总handler数)
            进入Logger方法
            //logger业务逻辑
            进入Next方法
                //index++ = 1
                这里还可以继续加入其他中间件
                //进入fun1方法
                //从fun1方法离开
                //index++ = 2
            离开Next方法

            //logger业务逻辑处理
        离开Logger方法
        index++ = 3
    离开Next方法
逻辑处理完毕

```

这里比较重要的概念是， 处理函数有先后执行关系， 并且处理函数可以通过调用`Abort`方法， 提前返回，不用递归调用到实际处理函数。


这些中间件，可以方便的使我们的业务代码接入`权限校验auth`，`日志管理`等其他功能模块。

### 总结

1. 核心亮点： 路由管理对象的实现+ 中间件的实现原理

### 参考文章：

https://www.kancloud.cn/liuqing_will/the_source_code_analysis_of_gin/616920

http://www.gorillatoolkit.org/pkg/
