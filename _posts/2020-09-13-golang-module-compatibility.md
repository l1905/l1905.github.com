---
title: (翻译)golang项目兼容适配讨论
description: (翻译)golang项目兼容适配讨论
categories:
 - golang
 - 模块开发
 - 架构
tags:
 - golang
 - 模块开发
---


## (翻译)golang项目兼容适配讨论


## 背景

原文地址： https://blog.golang.org/module-compatibility

此文为翻译文章， 主要是怎么解决`已对外发布模块的持续兼容`问题


## 正文

随着时间进行， 你的项目代码添加了很多新特性，同样改变了代码运行的方式， 项目处于持续更新迭代中。

因此需要重新考虑项目公共部分，在上一章节(将项目升级到V2)[https://blog.golang.org/v2-go-modules], 重要特性改变需要发布V1+版本。

但是， 对用户来说，使用新版本存在诸多问题。 用户必须找到新api接口调用， 并且调整他们的业务代码， 还有些用户从来不升级类库， 因此你需要持续维护两个版本的项目代码。  因此在已有项目上持续迭代，保持代码适配，才是更好的选择。

### 增加函数

经常的， 当添加重要特性时， 需要修改函数签名(包含函数的名字+函数参数)， 给其加上新的参数， 下面，我们会展示一些方法去解决这种变化。
首先我们来看一下不好的例子

我们采用一个比较巧妙的方法， 来添加新的参数， 即我们新添加的参数作为可变参数来传递。对下面的函数进行扩展:

```
func Run(name string)
```

增加额外的`size`参数， 默认值为0, 可能如下改变:

```
func Run(name string, size ...int)
```

我们所有的函数调用，都还能很好的运行。 但是下面代码会导致报错

```
package mypkg
var runner func(string) = yourpkg.Run
```

第一个函数没问题， 主要是因为 他的类型是 `func(string` 但是新函数的类型是 `func(string, ...int)`, 所以在编译阶段就报错失败了。

这个例子表明， 函数适配不是完全的隐式适配。事实上， 对函数签名不存在完美的隐式适配。

既然更改原有函数的的传参存在问题， 那我们转换思路： 增加新的函数。像下面的例子， 标准库中引入`context`包后， 函数首个参数 传递`context`作为一种普遍的尝试， 但是 稳定对外发布的APIs不能直接让第一个参数传递`context`, 因为会改变调用者的传参方式。

因此我们选择新增加一个函数， `database/sql`包的查询`Query`方法签名是：

```
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

当`context`包被引入后， GO开发团队在`database/sql`包中，新添加了如下方法

```
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
```

为了避免同时维护两套代码逻辑， 老方法的内部实现逻辑调整为:

```
func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
    return db.QueryContext(context.Background(), query, args...)
}
```

增加一个新方法，可以让我们很方便的将其融入到新API， 因为这些方法比较相似，并且被排列在一起， `Context`这个字符在新方法的名字中有体现， 因此， 对`database/sql` API的扩充， 并不会降低包的可读性。


如果你能预见到， 在不久的将来， 函数需要更多的传参， 你可以在传参劣币走中，预先加上可选参数。 最简单的方式是在参数重添加一个简单的结构体参数。 就像 [crypto/tls.Dial](https://pkg.go.dev/crypto/tls?tab=doc#Dial)函数一样：

```
func Dial(network, addr string, config *Config) (*Conn, error)
```

`Dial`函数 执行 TLS 握手操作， 需要`network` 和地址， 但是他有很多其他参数，都是使用默认值。 `config`传递`nil`即是使用默人值， 传递一个`Config` 结构体, 其包含几个字段， 就会覆盖掉默人值， 以后， 添加新的TLS配置参数， 只需要在`Config`结构体中添加一个新字段， 就可以隐式的兼容适配。


有时， 参数结构体对象中，增加的一个新函数， 和增加的配置参数， 他们被组合在一起， 并且参数结构体还作为一个方法的接收者， 思考下 `net`包的能力进化，从监听一个网络地址。  Go 1.11版本之前  `net`包，提供 `Listen`函数

```
func Listen(network, address string) (Listener, error)
```

在GO 1.11 版本中， 两个新特性被加入到 `net`监听中：

1. 传递`context 参数
2. 允许调用者提供一个"控制函数" 在创建之后，绑定之前，去调整 原生连接(raw connection) 


因此 需要一个新函数，接收`context`， 网络，地址 和一个控制函数。包作者增加了一个 [ListenConfig](https://pkg.go.dev/net@go1.11?tab=doc#ListenConfig) 结构体, 以后可以方便增加更多的可选配置， 而不需要重新定义冗长的新的对外函数， 为ListenConfig增加一个`Listen`方法。

```
type ListenConfig struct {
    Control func(network, address string, c syscall.RawConn) error
}

func (*ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error)
```

另一种方式是提供新选项模式， 即选项作为参数进行传递， 每个选项都是一个可以改变选项值状态的函数， 此模式出自[Self-referential functions and the design of options](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)
被广泛使用的例子是 [google.golang.org/grpc](https://pkg.go.dev/google.golang.org/grpc?tab=doc)的[DialOption](https://pkg.go.dev/google.golang.org/grpc?tab=doc#DialOption)

可选类型和结构题选项再函数参数中扮演同样的角色，他们是可扩展的一种方式， 即修改配置行为。选择那种作为主流方式， 考虑下面grpc的例子

```
grpc.Dial("some-target",
  grpc.WithAuthority("some-authority"),
  grpc.WithMaxDelay(time.Second),
  grpc.WithBlock())
```

同样可以使用结构体选项方式

```
notgrpc.Dial("some-target", &notgrpc.Options{
  Authority: "some-authority",
  MaxDelay:  time.Minute,
  Block:     true,
})
```

函数式的可选项方式有一些缺点： 

1. 在每个可选项调用前， 都需要写包名， 这里即是需要写`grpc.WIth`
2. 逐渐增长的包命名空间
3. 如果同一个选项参数被提供多次，比如两次， 会造成逻辑不清晰

结构体选项方式同样也有一些缺点：

1. 可选项结构体可能在大部分情况下一直是`nil`
2. 类型为零值， 可能有固定的含义， 并且可选项参数有默认值，是非常不方便，通常需要一个指针或者其他bool类型

每个人有足够的理由去选择适当的方式，来确保未来模块API的可扩展性。

### 针对接口interface开发

有时候， 新特性需要改变对外暴露的接口， 比如 一个接口需要扩展其新的方法， 直接在接口中添加，是一个重大改变，那么，我们怎么来支持新方法放在对外暴露的接口中？

我们首先想到的是： 定义一个新接口， 在新接口中有新方法， 当老接口在使用时，动态检查我们提供的类型是新接口， 还是老接口

我们继续举例， [archive/tar](https://pkg.go.dev/archive/tar?tab=doc) 包中
[tar.NewReader](https://pkg.go.dev/archive/tar?tab=doc#NewReader)接收 `ioReader`接口， 后来 GO团队意识到，更有效的办法是，跳过文件头，我们可以使用`seek`方法， 但是在`io.writer` 中不能加入`seek`方法，因为需要调整所有实现`io.writer`的地方。

另一种方法是改变`tar. NewReader`,  让他接收[io.ReadSeeker](https://pkg.go.dev/io?tab=doc#ReadSeeker), 而不再接收`io.writer` 主要是io.readseeker

直接改变函数签名传参方式， 也是一个重大调整。


因此他们保持`tar.NewReader` 函数签名不被改变。但是可以在检查 ```io.Seeker ```和 `tar.Reader`

```
package tar

type Reader struct {
  r io.Reader
}

func NewReader(r io.Reader) *Reader {
  return &Reader{r: r}
}

func (r *Reader) Read(b []byte) (int, error) {
  if rs, ok := r.r.(io.Seeker); ok {
    // Use more efficient rs.Seek.
  }
  // Use less efficient r.r.Read.
}
```
点击查看详细的[代码reader.go ](https://github.com/golang/go/blob/60f78765022a59725121d3b800268adffe78bde3/src/archive/tar/reader.go#L837)

当你想在已存在的接口上，新添加新的方法时, 可以这样操作：创建一个新接口， 新接口里有你的新方法。或者确认已存在的接口， 实现这个新方法。下一步， 实现以上关联的所有的函数。检查是否是第二个接口， 如果是，则使用第二个接口的方法。

(意外收获: 即接口 可以类型推断，是否是另一个接口实现， 这里即是类型推断接口的原始的数据结构。而不是接口)


但是这种实现 有局限性， 即老接口没办法再实现此方法， 限制未来模块的扩展性。

为了更好的避免这类问题， 设计构造方法， 返回具体类型， 具体类型可以让我们在不打扰用户调用，直接添加新方法，这样，再未来，扩展新方法会更简单。

小提示： 

如果你打算使用接口， 但是并不希望用户实现此接口，你可以增加一个不可导出接口，这样会阻止在包外实现此接口， 即不可扩展。这样的话， 后期你你在接口上添加新方法， 不会影响到用户的调用， 向下面的例子[testing.TB's private() function](https://github.com/golang/go/blob/83b181c68bf332ac7948f145f33d128377a09c42/src/testing/testing.go#L564-L567)

```
type TB interface {
    Error(args ...interface{})
    Errorf(format string, args ...interface{})
    // ...

    // A private method to prevent users implementing the
    // interface and so future additions to it will not
    // violate Go 1 compatibility.
    private()
}
```

这个主题，在其他地方被详细的讨论过. ([视频](https://www.youtube.com/watch?v=JhdL5AkH-AQ), [PPT](https://github.com/gophercon/2019-talks/blob/master/JonathanAmsterdam-DetectingIncompatibleAPIChanges/slides.pdf))

### 增加配置方法

到目前为止， 我们讨论了重大新特性改变， 改变一个类型或函数将会让用户的代码编译失败，对外行为性改变将会影响到用户，即使用户代码持续去改变。
举个例子， 很多用户希望 [json.Decoder](https://pkg.go.dev/encoding/json?tab=doc#Decoder) 忽略JSON中不再结构体中的字段， 当golang 开发团队想要返回一个err报错时， 需要特别小心。 当没有可选项时， 将会让很多正常的用户收到此报错。

相对于改变所有用户的行为，增加一个配置方法[Decoder.DisallowUnknownFields](https://pkg.go.dev/encoding/json?tab=doc#Decoder.DisallowUnknownFields),  调用这个配置选项的用户，将会获得新特性， 其他用户继续保持原有返回输出。

### 维持结构体的兼容适配

我们可以看到， 改变函数签名，都是重大改变， 使用结构体将是更好的选择。

如果你有一个对外发布的结构体类型， 你可以一直 增加或者删除未导出的字段， 而不需要做重大版本发布，当增加一个字段， 确保零值是有意义的，并且还可以保持原有老的行为，这样老代码即使不设置新字段，这样可以运行。

在 1.11版本 `net`包的作者增加`ListenConfig`结构体，他觉得增加更多选项是为了将来做更多打算， 事实证明，他是对的， 在1.13版本，增加了[KeepAlive field](https://pkg.go.dev/net@go1.13?tab=doc#ListenConfig), 允许禁止keep-alive , 或者更改对应时间， 默认零值，将保证原有行为继续使用默人的响应时间。

当在结构体中增加一个新字段，将会导致不可预期的细微变化，如果在结构体中所有的字段都是可比较类型（即字段类型对应的值，可以使用== 和!= )并且可以作为map[key]使用，结构体类型也是一样， 在这种情况下， 增加一个不可比较的字段类型，将会导致结构体也不可以比较(打破了结构体中所有的字段都可比较)。

为了保持结构体的可比较性， 不要增加不可比较字段， 你可以写个小demo测试下或者使用即将发布的[gorelease](https://pkg.go.dev/golang.org/x/exp/cmd/gorelease?tab=doc)

为了提前禁止比较特性，可以提前在结构体中增加不可比较字段类型，比如增加一个 切片，map， 函数类型，除了他们都是可比较类型的，你可以像下面一样操作

```
type Point struct {
        _ [0]func()
        X int
        Y int
}
```

函数类型是不可比较的， 数组零长度不占空间， 我们可以定义一个类型，让其更清晰


```
type doNotCompare [0]func()

type Point struct {
        doNotCompare
        X int
        Y int
}
```

在你的结构体中， 你应该使用 `doNotCompare`嘛？

如果你已经定义了一个结构体， 并且当作指针使用，

可能存在一个指针方法，或者有一个 NewXXXX构造函数返回指针，增加`doNotCompare`，会稍微有些过度小心， 用户的指针类型， 可以理解为每个字段的值都是独立的，如果想要比较两个值，应该直接比较这两个指针。

如果你定义了一个结构体，打算直接将其作为值使用， 像我们举例子的`Point`, 你可能更注重是否可比较，在不普遍的例子中，你有一个值类型结构体，但是你不需需要可比较特性，然后增加一个`doNotCompare`字段，将让你更加随心所欲， 将来想要改变结构体时，不需要再考虑打破可比较性，

不可比性的缺点就是：没办法作为map的key值了

### 总结

当计划发布API， 无比要小心考虑API的扩展性，其可能会成为新的改变。

当你需要增加新特性时， 记得以下规则： 

1. 增加
2. 不需要改变或者删除
3. 牢记异常情况: 接口，方法参数，返回值不能被添加，因为都不是兼容的


如果你想要动态改变API， 如果API开始时，很多特性被加入， 那么可能需要一段时间，来创建新版本，在大部分情况下， 可适配，兼容性的改变将会避免用户的痛苦。




















