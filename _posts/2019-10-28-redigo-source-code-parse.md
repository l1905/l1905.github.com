---
title: redigo源码分析
description: redigo源码分析
categories:
 - golang
tags:
 - golang
 - redis
 - 性能分析
---

## redigo源码分析

主要关注点在于 `连接池pool`和 `连接实例`的管理。

### 我们首先分析 连接池管理

连接池的主要动作包括

1. 初始化连接池 init
2. 获取一次连接 get
3. 释放连接 put

我们可以看下连接池对象的具体参数


```go
type Pool struct {
    // Dial is an application supplied function for creating and configuring a
    // connection.
    //
    // The connection returned from Dial must not be in a special state
    // (subscribed to pubsub channel, transaction started, ...).
    // 建立连接回调操作， 让用户主动去建立连接
    Dial func() (Conn, error)

    // DialContext is an application supplied function for creating and configuring a
    // connection with the given context.
    //
    // The connection returned from Dial must not be in a special state
    // (subscribed to pubsub channel, transaction started, ...).
    // 带着context上下文的 超时时间的建立连接操作
    DialContext func(ctx context.Context) (Conn, error)
    
    // TestOnBorrow is an optional application supplied function for checking
    // the health of an idle connection before the connection is used again by
    // the application. Argument t is the time that the connection was returned
    // to the pool. If the function returns an error, then the connection is
    // closed.
    TestOnBorrow func(c Conn, t time.Time) error

    // Maximum number of idle connections in the pool.
    // 最大闲置连接数
    MaxIdle int

    // Maximum number of connections allocated by the pool at a given time.
    // When zero, there is no limit on the number of connections in the pool.
    // 最多连接数， 
    MaxActive int

    // Close connections after remaining idle for this duration. If the value
    // is zero, then idle connections are not closed. Applications should set
    // the timeout to a value less than the server's timeout.
    // 闲置连接的closed时间， 即闲置连接，多长时间不用后， 自动清除， 是从当前连接【放回put】开始计时
    IdleTimeout time.Duration

    // If Wait is true and the pool is at the MaxActive limit, then Get() waits
    // for a connection to be returned to the pool before returning.
    // 当连接全部被使用完， 是否继续等待连接空出， 还是直接返回，告知客户端，无可用的连接
    Wait bool

    // Close connections older than this duration. If the value is zero, then
    // the pool does not close connections based on age.
    // 最大连接生命周期, 是指从【创建开始】计时
    MaxConnLifetime time.Duration
    
    // 是否已经初始化，最大连接总数
    chInitialized uint32 // set to 1 when field ch is initialized

    // 锁管理， 管理以下字段的修改
    mu           sync.Mutex    // mu protects the following fields
    closed       bool          // set to true when the pool is closed.
    active       int           // the number of open connections in the pool
    // 活跃连接池对象， 从此队列获取名额
    ch           chan struct{} // limits open connections when p.Wait is true
    // 闲置连接列表
    idle         idleList      // idle connections
    // 等待连接总数
    waitCount    int64         // total number of connections waited for.
    // 新连接等待时长
    waitDuration time.Duration // total time waited for new connections.
}
``` 


获取连接具体操作

```go
func (p *Pool) get(ctx context.Context) (*poolConn, error) {
    // 1. 检查是否需要等待空闲连接， 如果需要等待， 则需要等待
    // 2. 从后往前，检查闲置连接链表是否超时，IdleTimeout， 超时则释放连接
    // 3. 从前往后， 检查闲置连接表是否有没有超时的连接， 如果有，则返回当前连接， 否则的话，释放连接
    // 4. 创建连接， 如果失败， 将本次申请的名额，重新放回channel

    // Handle limit for p.Wait == true.
    var waited time.Duration
    // 如果已经处于等待wait状态， 并且 最大连接数大于0
    if p.Wait && p.MaxActive > 0 {
        // 进行懒加载操作， 初始化channel
        p.lazyInit()

        // wait indicates if we believe it will block so its not 100% accurate
        // however for stats it should be good enough.
        // 判断实际上是否需要等待， 如果未消费总数==0, 则确定一定需要等待
        wait := len(p.ch) == 0
        var start time.Time
        if wait {
            start = time.Now()
        }
        // 如果没有ctx， 则直接从p.ch中，获取连接实例
        if ctx == nil {
            <-p.ch
        } else {
            // 阻塞， p.ch, ctx.Done 看谁先完成
            select {
            case <-p.ch:
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
        if wait {
            // 计算等待时长
            waited = time.Since(start)
        }
    }

    p.mu.Lock()

    // 等待统计
    if waited > 0 {
        // 等待次数+1
        p.waitCount++
        // 等待时长累加
        p.waitDuration += waited
    }

    // Prune stale connections at the back of the idle list.
    if p.IdleTimeout > 0 {
        n := p.idle.count
        // 从后往前检查， 释放已经到达过期时间的闲置连接 实例
        for i := 0; i < n && p.idle.back != nil && p.idle.back.t.Add(p.IdleTimeout).Before(nowFunc()); i++ {
            pc := p.idle.back
            p.idle.popBack()
            p.mu.Unlock()
            pc.c.Close()
            p.mu.Lock()
            p.active--
        }
    }

    // Get idle connection from the front of idle list.
    // 从idlelist 的前面，获取闲置连接总数
    for p.idle.front != nil {
        pc := p.idle.front
        p.idle.popFront()
        p.mu.Unlock()
        if (p.TestOnBorrow == nil || p.TestOnBorrow(pc.c, pc.t) == nil) &&
            (p.MaxConnLifetime == 0 || nowFunc().Sub(pc.created) < p.MaxConnLifetime) {
            // 成功获取连接
            return pc, nil
        }
        // 连接已经超时， 需要重新创建连接
        pc.c.Close()
        p.mu.Lock()
        p.active--
    }

    // Check for pool closed before dialing a new connection.
    // 检查连接池是否关闭
    if p.closed {
        p.mu.Unlock()
        return nil, errors.New("redigo: get on closed pool")
    }

    // Handle limit for p.Wait == false.
    // 不需要等待，并且连接已经用完， 则直接报错
    if !p.Wait && p.MaxActive > 0 && p.active >= p.MaxActive {
        p.mu.Unlock()
        return nil, ErrPoolExhausted
    }
    // 再活连接+1
    p.active++
    p.mu.Unlock()
    c, err := p.dial(ctx)
    // 创建连接失败
    if err != nil {
        c = nil
        p.mu.Lock()
        p.active--
        if p.ch != nil && !p.closed {
            // 本次创建连接失败， 将本次创建连接名额，重新送回
            p.ch <- struct{}{}
        }
        p.mu.Unlock()
    }
    return &poolConn{c: c, created: nowFunc()}, err
}
```

释放返回连接
```go
// 将连接放回的 连接池
func (p *Pool) put(pc *poolConn, forceClose bool) error {
    p.mu.Lock()
    // 先检查连接是否关闭
    if !p.closed && !forceClose {
        pc.t = nowFunc()
        p.idle.pushFront(pc)
        // 如果闲置连接已经达到最大总数， 则从后往前弹出，一个
        if p.idle.count > p.MaxIdle {
            pc = p.idle.back
            p.idle.popBack()
        } else {
            pc = nil
        }
    }
    // 释放从后往前弹出的连接
    if pc != nil {
        p.mu.Unlock()
        pc.c.Close()
        p.mu.Lock()
        p.active--
    }

    //
    if p.ch != nil && !p.closed {
        // 交还连接实例到 channel
        p.ch <- struct{}{}
    }
    p.mu.Unlock()
    return nil
}

// 当close 再活连时， 真正放回
func (ac *activeConn) Close() error {
    pc := ac.pc
    if pc == nil {
        return nil
    }
    ac.pc = nil

    if ac.state&connectionMultiState != 0 {
        pc.c.Send("DISCARD")
        ac.state &^= (connectionMultiState | connectionWatchState)
    } else if ac.state&connectionWatchState != 0 {
        pc.c.Send("UNWATCH")
        ac.state &^= connectionWatchState
    }
    if ac.state&connectionSubscribeState != 0 {
        pc.c.Send("UNSUBSCRIBE")
        pc.c.Send("PUNSUBSCRIBE")
        // To detect the end of the message stream, ask the server to echo
        // a sentinel value and read until we see that value.
        sentinelOnce.Do(initSentinel)
        pc.c.Send("ECHO", sentinel)
        pc.c.Flush()
        for {
            p, err := pc.c.Receive()
            if err != nil {
                break
            }
            if p, ok := p.([]byte); ok && bytes.Equal(p, sentinel) {
                ac.state &^= connectionSubscribeState
                break
            }
        }
    }
    pc.c.Do("")
    ac.p.put(pc, ac.state != 0 || pc.c.Err() != nil)
    return nil
}
```

这里比较核心的点是， 管理闲置连接，是通过双联表来操作的， 具体定义如下

```go
双联表：
prev<——————conn1----->next
            prev<-----front----->next
            
// pushFront 操作  双链表， 压到栈头
pushFront(pc *poolConn)
    // 初始化pc操作, 修改pc 的next指针
    1. pc.next = l.front
    2. pc.prev = nil

    // 设置idleList 
    如果idleList 长度 <0 
        l.back = pc
    否则
        l.front.prev = pc

    l.front = pc
    l.count++

双链表，弹出
func (l *idleList) popFront() {
    // 获取栈首元素
    pc := l.front
    //总数减1
    l.count--
    // 如果元素已经全取完， 清空指针
    if l.count == 0 {
        l.front, l.back = nil, nil
    } else {
        pc.next.prev = nil // 断开链表
        l.front = pc.next  // 断开链表
    }
    pc.next, pc.prev = nil, nil
}
// pop出最深入的一个
func (l *idleList) popBack() {
    pc := l.back
    l.count--
    if l.count == 0 {
        l.front, l.back = nil, nil
    } else {
        pc.prev.next = nil
        l.back = pc.prev
    }
    pc.next, pc.prev = nil, nil
}
```

对连接的管理，定如下

```go
// 底层连接 实例管理, 一个连接，维护，读writer, 写reader
type conn struct {
    // Shared 
    // 网络连接实例
    mu      sync.Mutex
    pending int
    err     error
    conn    net.Conn

    // Read
    // 读writer
    readTimeout time.Duration
    br          *bufio.Reader

    // Write
    // 写writer
    writeTimeout time.Duration
    bw           *bufio.Writer

    // Scratch space for formatting argument length.
    // '*' or '$', length, "\r\n"
    // 格式化参数
    lenScratch [32]byte

    // Scratch space for formatting integers and floats.
    // 格式化参数
    numScratch [40]byte
}
```

其他即是按照redis协议进行通信即可

redis协议参考：


1. http://redisdoc.com/topic/protocol.html
2. https://juejin.im/post/5b69cf08f265da0f6a037dea
3. https://xargin.com/redis-protocal/


### 其他

使用`go-callvis`展示的redigo 可视化调用关系

使用 `jquery.graphviz.svg` 可以实现调用图的动态交互

简单实用原生协议实现一个`set` `get`方法， 加深下协议理解

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "net"
    "time"
)

// 简单构造redis协议
func main() {
    // 1. 建立请求
    proto := "tcp"
    host := "127.0.0.1:6379"
    fmt.Println("hello world")
    // 建立连接
    conn, err := net.Dial(proto, host)
    defer conn.Close()

    var br *bufio.Reader
    var bw *bufio.Writer

    if err != nil {
        log.Println(err)

    } else {
        // 设置读写实例
        br = bufio.NewReader(conn)
        bw = bufio.NewWriter(conn)
        // set redigo foobar
        // 构造写命令 set redigo "foobar" *总数\r\n{set长度}\r\nset\r\n{redigo长度}\r\n{foobar长度}\r\nfoobar\r\n
        bw.WriteString("*3\r\n$3\r\nset\r\n$6\r\nredigo\r\n$6\r\nfoobar\r\n")
        // 刷到网络
        bw.Flush()

        // 先不解析返回值， 只是打印返回值
        data, err := br.ReadSlice('\n')
        if err != nil {
            fmt.Print(err)
        } else {
            fmt.Print(string(data))
            // +OK
        }

        // 构造读取方法
        bw.WriteString("*2\r\n$3\r\nget\r\n$6\r\nredigo\r\n")
        bw.Flush()

        data, err = br.ReadSlice('\n')
        if err != nil {
            fmt.Print(err)
        } else {
            // 获取长度$6
            fmt.Print(string(data))

            // 获取第二个字符
            // foobar
            data, err = br.ReadSlice('\n')
            fmt.Println(string(data))
            // +OK
        }

    }

    // 两秒后自动退出
    time.Sleep(time.Second*2)

}

```




