---
title: golang中http包webserver运行源码分析
description: golang中http包webserver运行源码分析
categories:
 - golang开发
tags:
 - golang
 - golang-http包源码
---

## 科普golang中http包server服务运行


### 分析流程

常见使用用例：

```
package main

import (
    "log"
    "net/http"
)

func main() {
    #定义路由管理对象
    mux := http.NewServeMux()

    #定义具体handler回调方法
    rh := http.RedirectHandler("http://www.baidu.com", 307)
    mux.Handle("/foo", rh)

    log.Println("Listening...")

    #拉起http服务，监听3000端口， 关联路由管理对象mux
    http.ListenAndServe(":3000", mux)
}

```
利用http包， 我们轻松的拉起一个http服务。

我们这里详细追踪一下http服务拉起过程中， http包会帮我门做什么

```
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```
我们看到会生成server对象， 并且执行 `ListenAndServe`方法， 进入该方法，再一探究竟

```
func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    #实际监听端口
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    #进行实际逻辑处理
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```

主要是在tcp层面监听端口， 获得监听端口具柄， 在`srv.Serve`做具体的逻辑处理


```
func (srv *Server) Serve(l net.Listener) error {
    if fn := testHookServerServe; fn != nil {
        fn(srv, l) // call hook with unwrapped listener
    }

    l = &onceCloseListener{Listener: l}
    defer l.Close()

    if err := srv.setupHTTP2_Serve(); err != nil {
        return err
    }

    if !srv.trackListener(&l, true) {
        return ErrServerClosed
    }
    defer srv.trackListener(&l, false)

    var tempDelay time.Duration     // how long to sleep on accept failure
    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        #accpet数据，建立连接具柄
        rw, e := l.Accept()
        if e != nil {
            select {
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
            }
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return e
        }
        tempDelay = 0
        #生成新连接
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        #起协程，处理具体连接过程， 每次的业务请求，都单独再此协程序中处理
        go c.serve(ctx)
    }
}
```
上面代码，主要是server端监听端口， 然后获取新的连接， 开新协程， 完成业务逻辑处理。 下面我们看下`c.serve(ctx)`方法里，都是具体如何操作.


```
// Serve a new connection.
func (c *conn) serve(ctx context.Context) {
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    defer func() {
        if err := recover(); err != nil && err != ErrAbortHandler {
            const size = 64 << 10
            buf := make([]byte, size)
            buf = buf[:runtime.Stack(buf, false)]
            c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
        }
        if !c.hijacked() {
            c.close()
            c.setState(c.rwc, StateClosed)
        }
    }()

    if tlsConn, ok := c.rwc.(*tls.Conn); ok {
        if d := c.server.ReadTimeout; d != 0 {
            c.rwc.SetReadDeadline(time.Now().Add(d))
        }
        if d := c.server.WriteTimeout; d != 0 {
            c.rwc.SetWriteDeadline(time.Now().Add(d))
        }
        if err := tlsConn.Handshake(); err != nil {
            // If the handshake failed due to the client not speaking
            // TLS, assume they're speaking plaintext HTTP and write a
            // 400 response on the TLS conn's underlying net.Conn.
            if re, ok := err.(tls.RecordHeaderError); ok && re.Conn != nil && tlsRecordHeaderLooksLikeHTTP(re.RecordHeader) {
                io.WriteString(re.Conn, "HTTP/1.0 400 Bad Request\r\n\r\nClient sent an HTTP request to an HTTPS server.\n")
                re.Conn.Close()
                return
            }
            c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
            return
        }
        c.tlsState = new(tls.ConnectionState)
        *c.tlsState = tlsConn.ConnectionState()
        if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
            if fn := c.server.TLSNextProto[proto]; fn != nil {
                h := initNPNRequest{tlsConn, serverHandler{c.server}}
                fn(c.server, tlsConn, h)
            }
            return
        }
    }

    // HTTP/1.x from here on.

    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx()

    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

    for {
        #读取请求数据
        w, err := c.readRequest(ctx)
        if c.r.remain != c.server.initialReadLimitSize() {
            // If we read any bytes off the wire, we're active.
            c.setState(c.rwc, StateActive)
        }
        if err != nil {
            const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection: close\r\n\r\n"

            if err == errTooLarge {
                // Their HTTP client may or may not be
                // able to read this if we're
                // responding to them and hanging up
                // while they're still writing their
                // request. Undefined behavior.
                const publicErr = "431 Request Header Fields Too Large"
                fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
                c.closeWriteAndWait()
                return
            }
            if isCommonNetReadError(err) {
                return // don't reply
            }

            publicErr := "400 Bad Request"
            if v, ok := err.(badRequestError); ok {
                publicErr = publicErr + ": " + string(v)
            }

            fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
            return
        }

        // Expect 100 Continue support
        req := w.req
        if req.expectsContinue() {
            if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
                // Wrap the Body reader with one that replies on the connection
                req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
            }
        } else if req.Header.get("Expect") != "" {
            w.sendExpectationFailed()
            return
        }

        c.curReq.Store(w)

        if requestBodyRemains(req.Body) {
            registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
        } else {
            w.conn.r.startBackgroundRead()
        }

        // HTTP cannot have multiple simultaneous active requests.[*]
        // Until the server replies to this request, it can't read another,
        // so we might as well run the handler in this goroutine.
        // [*] Not strictly true: HTTP pipelining. We could let them all process
        // in parallel even if their responses need to be serialized.
        // But we're not going to implement HTTP pipelining because it
        // was never deployed in the wild and the answer is HTTP/2.
        #实际处理请求数据
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
        if c.hijacked() {
            return
        }
        w.finishRequest()
        if !w.shouldReuseConnection() {
            if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
                c.closeWriteAndWait()
            }
            return
        }
        c.setState(c.rwc, StateIdle)
        c.curReq.Store((*response)(nil))

        if !w.conn.server.doKeepAlives() {
            // We're in shutdown mode. We might've replied
            // to the user without "Connection: close" and
            // they might think they can send another
            // request, but such is life with HTTP/1.1.
            return
        }

        if d := c.server.idleTimeout(); d != 0 {
            c.rwc.SetReadDeadline(time.Now().Add(d))
            if _, err := c.bufr.Peek(4); err != nil {
                return
            }
        }
        c.rwc.SetReadDeadline(time.Time{})
    }
}
```

业务逻辑较多， 我们只关注于主数据的逻辑处理， 一共两点

```
#读取请求数据
w, err := c.readRequest(ctx)

...
#获取server管理句柄， 调用其ServeHTTP方法
serverHandler{c.server}.ServeHTTP(w, w.req)

```
核心是获取server管理句柄， 下面再此进入到`ServeHTTP`看其内部实现。



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

这一步主要是将请求数据dispatch分发给路由管理对象， 并且进行实际的路由分发

```
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    #获取请求处理对象， 这里即是路由器操作
    h, _ := mux.Handler(r)
    #进行serverHttp操作, 这里即执行我们预先定义的方法
    h.ServeHTTP(w, r)
}
```
这里的`serverHttp`方法相对陌生， 我们最开始定义程序时， 是没见到该 对象方法的。
我们定义的句柄函数是 `redirectHandler`, 我们看下他的定义，发现同样实现了`ServeHTTP`方法， 因此处理函数的底层生成都必须定义`ServeHTTP`方法

```
// RedirectHandler returns a request handler that redirects
// each request it receives to the given url using the given
// status code.
//
// The provided code should be in the 3xx range and is usually
// StatusMovedPermanently, StatusFound or StatusSeeOther.
func RedirectHandler(url string, code int) Handler {
    return &redirectHandler{url, code}
}

func htmlEscape(s string) string {
    return htmlReplacer.Replace(s)
}

// Redirect to a fixed URL
type redirectHandler struct {
    url  string
    code int
}

#这里即实现了我们对应的`ServerHttp`方法
func (rh *redirectHandler) ServeHTTP(w ResponseWriter, r *Request) {
    Redirect(w, r, rh.url, rh.code)
}
```




我们看下`mux.Handler`都做了哪些事情

```
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {

    // CONNECT requests are not canonicalized.
    if r.Method == "CONNECT" {
        // If r.URL.Path is /tree and its handler is not registered,
        // the /tree -> /tree/ redirect applies to CONNECT requests
        // but the path canonicalization does not.
        if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
            return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
        }

        return mux.handler(r.Host, r.URL.Path)
    }

    // All other requests have any port stripped and path cleaned
    // before passing to mux.handler.
    #获取请求host
    host := stripHostPort(r.Host)
    #获取请求路径
    path := cleanPath(r.URL.Path)

    // If the given path is /tree and its handler is not registered,
    // redirect for /tree/.
    if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
        return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
    }

    if path != r.URL.Path {
        _, pattern = mux.handler(host, path)
        url := *r.URL
        url.Path = path
        return RedirectHandler(url.String(), StatusMovedPermanently), pattern
    }
    #根据请求host, path获取具体的 处理函数
    return mux.handler(host, r.URL.Path)
}
```

从这里我们可以看到， 我们从请求中解析出host, path, 然后调用`mux.handler`来获取映射的处理函数

```
// handler is the main implementation of Handler.
// The path is known to be in canonical form, except for CONNECT methods.
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    // Host-specific pattern takes precedence over generic ones
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}
```


这里不再详细追踪， 即是通过`mux.match`方法获取到 handler函数和匹配模式


### 总结

到这里我们完成对http包 server的流程 昨晚分析， 做个总结

项目拉起时，主要做以下事情

1. 注册多路路由器， 即建立 路径和hander处理函数的映射关系
2. 拉起tcp服务， 监听端口


当一次请求到达时：

1. 建立新连接， 起新的连接协程
2. 通过多路路由选择器找到对应的 处理函数
3. 进行业务处理


其他的函数方法，无非是一些闭包，一些语法糖， 万变不离其宗。


参考文章：

1. [golang http包理解](https://studygolang.com/articles/9467 )




