---
title: golang中channel常见用法
description: golang中channel常见用法
categories:
 - golang
tags:
 - golang
 - select
 - channel
---

## golang channel 使用

## 理解

实践源码： https://github.com/l1905/golang_daily_tools/blob/master/channel_and_select/main.go


Channel可以作为一个先入先出(FIFO)的队列。 可以将channel 细化为以下三个流程。

1. 定义channel
2. 向channel发送(publish）数据
3. 从channel中获取数据进行消费


下面，我们分类细化讲解

## 定义channel

Channel类型的定义格式如下：

```
ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType .
```

它包括三种类型的定义。可选的`<-`代表channel的方向。如果没有指定方向，那么Channel就是双向的，既可以接收数据，也可以发送数据。

```
chan T          // 可以接收和发送类型为 T 的数据, 定义时使用
chan<- float64  // 只可以用来发送 float64 类型的数据， 在函数参数中使用， 这样可以限定chan使用
<-chan int      // 只可以用来接收 int 类型的数据, 在函数参数中使用， 这样可以限定chan使用
```

`<-`总是优先和`最左边`的类型结合。(The <- operator associates with the leftmost chan possible)

```
chan<- chan int    // 等价 chan<- (chan int)
chan<- <-chan int  // 等价 chan<- (<-chan int)
<-chan <-chan int  // 等价 <-chan (<-chan int)
chan (<-chan int)
```

#### 初始化容量和类型

使用`make`初始化Channel类型,并且可以设置容量:

```
make(chan int, 100)
```

容量(capacity)代表Channel容纳的最多的元素的数量，代表Channel的缓存的大小。类似队列可支持的最大容量。

channel从容量上可以分为以下两种

#### 无缓存，即没有设置容量，或者设置容量为0

只有sender和receiver都准备好了后它们的通讯(communication)才会发生(Blocking)

#### 有缓存容量

send: send开始时不阻塞， 只有buffer满了后 send才会阻塞， 

receive: 只有缓存空了后receive才会阻塞


## 向channel发送(publish）数据

send语句用来往Channel中发送数据， 如`ch <- 3`

定义如下：

```
SendStmt = Channel "<-" Expression .
Channel  = Expression .
```

在通讯(communication)开始前channel和expression必选先求值出来(evaluated)，比如下面的(3+4)先计算出7然后再发送给channel。


```
c := make(chan int)
defer close(c)
go func() { c <- 3 + 4 }()
i := <-c
fmt.Println(i)
```

send被执行前(proceed)通讯(communication)一直被阻塞着。

#### channel无缓存队列

无缓存的channel只有在receiver准备好后send才被执行

#### channel缓存未满

send会被执行

#### channel缓存已满
send会阻塞，直到缓存被消费，变成不满

#### channel已经被close

继续发送数据会导致`run-time panic`。

#### channel被设置为nil

继续发送数据会`一直阻塞`, 

(todo 这里需要验证，配合select-default)


## 从channel中获取数据进行消费

```
<-ch
```

#### channel无缓存队列或者缓存列表为空

表达式会一直被block,直到有数据可以接收

#### channel缓存有数据

持续进行消费，获取数据

#### channel已被close

从一个被close的channel中接收数据不会被阻塞，而是立即返回，接收完已发送的数据后会返回元素类型的零值(zero value)。

这里可以根据零值判断， channel是否关闭

```
var x, ok = <-ch
如果OK 是false，表明接收的x是产生的零值，这个channel被关闭了或者为空。
```


#### channel被设置为nil

从一个nil channel中接收数据会一直被block。


#### 通过range进行消费

`for …… range` 语句可以处理Channel。

```
func main() {
    go func() {
        time.Sleep(1 * time.Hour)
    }()
    c := make(chan int)
    go func() {
        for i := 0; i < 10; i = i + 1 {
            c <- i
        }
        close(c)
    }()
    for i := range c {
        fmt.Println(i)
    }
    fmt.Println("Finished")
}
```

`range c`产生的迭代值为Channel中发送的值，它会一直迭代直到channel被关闭。上面的例子中如果把close(c)注释掉，程序会一直阻塞在`for …… range`那一行


## 通过select进行发送和消费

`select`语句选择一组可能的`send`操作和`receive`操作去处理。它类似`switch`,但是`只能`用来处理通讯(communication)操作。

它的case必须是send语句，receive语句，default语句中的一个

最多允许有一个`default case`,它可以放在case列表的任何位置，尽管我们大部分会将它放在最后。

```
import "fmt"
func fibonacci(c, quit chan int) {
    x, y := 0, 1
    for {
        select {
        case c <- x:
            x, y = y, x+y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}
func main() {
    c := make(chan int)
    quit := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
```

如果有同时多个case去处理,比如同时有多个channel可以接收数据，那么Go会伪随机的选择一个case处理(pseudo-random)。


`非阻塞情况`：存在`default`, 没有case需要处理, 则会选择`default`去处理

`阻塞情况`：没有`default`, 直到某个case需要处理。


需要注意的是，`nil channel`上的操作会一直被阻塞，如果没有`default case`,只有`nil channel`的`select`会一直被阻塞。(todo 需要验证)


`select`语句和`switch`语句一样，它不是循环，它只会选择一个case来处理，如果想一直处理`channel`，你可以在外面加一个无限的`for`循环：

```
for {
    select {
    case c <- x:
        x, y = y, x+y
    case <-quit:
        fmt.Println("quit")
        return
    }
}
```


## timeout超时处理

`select`有很重要的一个应用就是超时处理。 因为上面我们提到，如果没有case需要处理，select语句就会一直阻塞着。这时候我们可能就需要一个超时操作，用来处理超时的情况。

下面这个例子我们会在2秒后往channel c1中发送一个数据，但是select设置为1秒超时,因此我们会打印出`timeout 1`,而不是`result 1`。

```
import "time"
import "fmt"
func main() {
    c1 := make(chan string, 1)
    go func() {
        time.Sleep(time.Second * 2)
        c1 <- "result 1"
    }()
    select {
    case res := <-c1:
        fmt.Println(res)
    case <-time.After(time.Second * 1):
        fmt.Println("timeout 1")
    }
}

todo 需要验证， 这里的超时时间，是怎么计时的
```
其实它利用的是`time.After`方法，它返回一个类型为`<-chan Time`的单向的channel，在指定的时间发送一个当前时间给返回的channel中。



## Timer和Ticker 使用

我们看一下关于时间的两个Channel。
timer是一个定时器，代表未来的一个单一事件，你可以告诉timer你要等待多长时间，它提供一个Channel，在将来的那个时间那个Channel提供了一个时间值。下面的例子中第二行会阻塞2秒钟左右的时间，直到时间到了才会继续执行。


```
timer1 := time.NewTimer(time.Second * 2)
<-timer1.C
fmt.Println("Timer 1 expired")
```

当然如果你只是想单纯的等待的话，可以使用`time.Sleep`来实现。

你还可以使用`timer.Stop`来停止计时器。

```
timer2 := time.NewTimer(time.Second)
go func() {
    <-timer2.C
    fmt.Println("Timer 2 expired")
}()
stop2 := timer2.Stop()
if stop2 {
    fmt.Println("Timer 2 stopped")
}
```


`ticker`是一个定时触发的计时器，它会以一个间隔(interval)往Channel发送一个事件(当前时间)，而Channel的接收者可以以固定的时间间隔从Channel中读取事件。下面的例子中ticker每500毫秒触发一次，你可以观察输出的时间。

```
ticker := time.NewTicker(time.Millisecond * 500)
go func() {
    for t := range ticker.C {
        fmt.Println("Tick at", t)
    }
}()
```

类似`timer`, `ticker`也可以通过`Stop`方法来停止。一旦它停止，接收者不再会从channel中接收数据了。


## 同步操作

channel可以用在goroutine之间的同步。
下面的例子中`main goroutine`通过`done channel`等待worker完成任务。 worker做完任务后只需往channel发送一个数据就可以通知`main goroutine`任务完成。

```
import (
    "fmt"
    "time"
)
func worker(done chan bool) {
    time.Sleep(time.Second)
    // 通知任务已完成
    done <- true
}
func main() {
    done := make(chan bool, 1)
    go worker(done)
    // 等待任务完成
    <-done
}
```


## 参考文章：

https://colobu.com/2016/04/14/Golang-Channels/

https://lg1024.com/post/go_timer_ticker.html








