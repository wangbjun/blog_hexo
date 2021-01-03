---
title: Golang的HttpServer解析
date: 2020-06-07 22:00:00
tags: 
- Golang
- Http
category: Golang
---
Golang之所以非常适合用于网络编程的原因之一就是其自带网络库可以非常简单快速的建立一个基于http或者tcp的服务应用，以http服务为例，只需几行代码：
```go
package main

import "net/http"

func main() {
    http.HandleFunc("/", func(writer http.ResponseWriter, r *http.Request) {
        _, _ = writer.Write([]byte("Hello"))
    })
    err := http.ListenAndServe(":8888", nil)
    if err != nil {
        panic(err)
    }
}
```
上面这几行代码就启动了一个http服务，运行在8888端口，虽然说非常简陋，也不区分GET或者POST，但是其性能缺十分高效，主要是得益于其底层使用了协程，对每一个请求都会分配一个协程去处理，并发能力强。

<!--more-->

## 1.ISO网络模型
说到网络，不得不说一下这个模型，http本质上是一个基于tcp协议的应用层协议，而且http是超文本传输协议，注意这里的文本是指其通信协议是文本形式（具体表现就是请求头和响应头），其传输的内容并不一定是文本，可以是任何内容，图片等二进制内容都可以。

<img src="https://wangbjun.site/images/old/5f6e3e27ly1g371bhib22j20da0dn0t2.jpg" />

说到tcp就不得不说下socket网络编程，上面这张图基本上描述了tcp网络通信的一个流程，这和http有什么关系呢？

实际上，这种图里面```处理请求```这部分则是http服务应该做的东西，tcp只负责传输控制，至于内容，其协议可能是http，也可能是ftp，甚至有可能是自定义的协议。

网上借张http协议报文的图看一下：

<img src="/images/2020/2020-06-07_16-30.png" />

实际上，协议内容非常复杂，对于每一个字段代表的意思和应该出现的位置都有规定，有人可能说，我拿chrome F12打开控制台看到和这个不一样啊，那是因为浏览器把请求解析了一行行显示出来方便咱调试而已。

如果你理解了这2张图，你应该理解了一个http请求是怎么发起，怎么响应的，但是实际应用中，Go又是怎么做到解析协议并且响应结果的呢？

## 2.DefaultServeMux
让我们点开源码，看看这几行代码到底干了啥，首先看一下个 ```HandlFunc()```，Go里面关于http server的代码都在server.go这个文件里面，总共3000多行，有点多，这里摘取部分。
```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}
type muxEntry struct {
    h       Handler
    pattern string
}
// DefaultServeMux is the default ServeMux used by Serve.
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux

// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}

// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    if handler == nil {
        panic("http: nil handler")
    }
    mux.Handle(pattern, HandlerFunc(handler))
}
```
这里的核心是```ServeMux```这个结构体，按照注释的介绍，这个是一个http server的多路复用分发器，顾名思义，它是用来处理分发请求的，下面是它实现的一些方法：

<img src="/images/2020/2020-06-07_19-01.png" />

这个结构体有4个成员属性，其中mu是一个读写互斥锁；m是一个map，其key是一个string（实际上也是路由），value是一个muxEntry；这个muxEntry则是代表了handler和pattern，其中pattern就是咱说的路由，又叫请求path。

HandleFunc最终调用了Handle方法，其主要目的是把路由和handler函数做一个映射关系，简单说就是注册路由：
```go
// Handle registers the handler for the given pattern.
// If a handler already exists for pattern, Handle panics.
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()
    
    if pattern == "" {
        panic("http: invalid pattern")
    }
    if handler == nil {
        panic("http: nil handler")
    }
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }
    
    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    e := muxEntry{h: handler, pattern: pattern}
    mux.m[pattern] = e
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }
    
    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```
这里需要注意一点的是```ServeMux```里面包含一个es，它保存了一个有序的entry，是根据pattern从长到短排序，不知道有啥用。。。

总结，这里面的```DefaultServeMux```就是库里面自己已经初始化好的一个结构体，我们可以直接用，使用它的handle方法就可以注册路由，这时有人可能会问，那咱自己动手行不行呢？当然可以
```go
package main

import "net/http"

var myServer = new(http.ServeMux)

func main() {
    myServer.HandleFunc("/", func(writer http.ResponseWriter, r *http.Request) {
        _, _ = writer.Write([]byte("Hello"))
    })
    err := http.ListenAndServe(":8888", myServer)
    if err != nil {
        panic(err)
    }
}
```
上面这种写法就是自己new一个ServeMux，如果我们点开```ListenAndServe```这个方法，可以看到注释非常明确的写到如果第二个参数为nil则会默认使用```DefaultServeMux```，这也就解释了为什么第一种写法这个参数是nil。
```go
// ListenAndServe listens on the TCP network address addr and then calls
// Serve with handler to handle requests on incoming connections.
// Accepted connections are configured to enable TCP keep-alives.
//
// The handler is typically nil, in which case the DefaultServeMux is used.
//
// ListenAndServe always returns a non-nil error.

func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```
## 2.ListenAndServe
前面只是一些准备工作，真正的逻辑是在这个方法里面，首先，这个方法接受2个参数，一个是监听的地址，一个```Handler```，handler是一个interface，它只有一个方法需要实现：
```go
// A Handler responds to an HTTP request.
//
// ServeHTTP should write reply headers and data to the ResponseWriter
// and then return. Returning signals that the request is finished; it
// is not valid to use the ResponseWriter or read from the
// Request.Body after or concurrently with the completion of the
// ServeHTTP call.
//
// Depending on the HTTP client software, HTTP protocol version, and
// any intermediaries between the client and the Go server, it may not
// be possible to read from the Request.Body after writing to the
// ResponseWriter. Cautious handlers should read the Request.Body
// first, and then reply.
//
// Except for reading the body, handlers should not modify the
// provided Request.
//
// If ServeHTTP panics, the server (the caller of ServeHTTP) assumes
// that the effect of the panic was isolated to the active request.
// It recovers the panic, logs a stack trace to the server error log,
// and either closes the network connection or sends an HTTP/2
// RST_STREAM, depending on the HTTP protocol. To abort a handler so
// the client sees an interrupted response but the server doesn't log
// an error, panic with the value ErrAbortHandler.

type Handler interface {
    ServeHTTP(ResponseWriter, *Request) 
}
```
如果你留意了你会发现，之前那个```ServeMux```也是实现了这个方法，所以我们可以这么用。

哎，这时候你有一个大胆的想法，那如果我自己定义一个结构体去实现这个方法，是不是连```ServeMux```都不用new了？你别说还真是这样
```go
package main

import "net/http"

type myServer struct{}

func (myServer) ServeHTTP(writer http.ResponseWriter, r *http.Request) {
    _, _ = writer.Write([]byte("Hello"))
}

func main() {
    err := http.ListenAndServe(":8888", myServer{})
    if err != nil {
        panic(err)
    }
}
```
但是你这么写，就连基本的路由功能都没了。。。因为```ServeMux```本质上就是带了一个路由功能而已，相当于官方库实现的一个路由，虽然不够强大，但是基本够用，很多第三方框架甚至会自己实现更强大的路由功能。

继续看这个```ListenAndServe```源码，可以看到它初始化了一个```Server```，把地址和Handler传进去了，这个Server是最重要的一个结构体了，它里面的成员和方法特别多，这里就不列出了，直接看调用的方法。
```go
func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```
这里可以看到它调用了```net.Listen```监听了端口的tcp请求，这里我就不继续往下追了，因为我认为这里面往下都是属于tcp传输层的东西，不是本文研究的重点，有兴趣的童鞋可以继续追进去看一下。

最终把```TCPListener```转换成一个```tcpKeepAliveListener```调用了方法Serve：
```go
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each. The service goroutines read requests and
// then call srv.Handler to reply to them.
//
// HTTP/2 support is only enabled if the Listener returns *tls.Conn
// connections and they were configured with "h2" in the TLS
// Config.NextProtos.
//
// Serve always returns a non-nil error and closes l.
// After Shutdown or Close, the returned error is ErrServerClosed.

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
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```
其中比较核心的是for循环里面那一段，不断的```Accept```新的请求，然后通过```srv.newConn```创建一个新的连接，然后开启一个go协程处理这个请求。这个```srv.newConn```返回的是一个```conn```结构体，其成员和函数也非常之多，它代表的是服务的一个http连接。

下面这段代码是处理http请求协议内容的核心代码：
```go
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
后面的代码非常之多，不太好解析，这里捡个重点说说，其中解析http协议调用的是```readRequest```方法，里面做了很多操作，比如说解析请求头的一些属性，比如请求类型、协议版本、请求URI、缓存控制等等，把他们放入到```Request```对象里面，这个结构体也是我们日常开发中最常用到的，我们会从这里获取所需要的请求信息。
```go
// parseRequestLine parses "GET /foo HTTP/1.1" into its three parts.
func parseRequestLine(line string) (method, requestURI, proto string, ok bool) {
    s1 := strings.Index(line, " ")
    s2 := strings.Index(line[s1+1:], " ")
    if s1 < 0 || s2 < 0 {
        return
    }
    s2 += s1 + 1
    return line[:s1], line[s1+1 : s2], line[s2+1:], true
}
```
当然其中还有一个非常重要的操作，那就是调用我们之前注册的handler，这个操作是在解析完http请求之后的地方，然后后面就是一些收尾操作。

<img src="/images/2020/2020-06-07_21-46.png" />

## 3.总结
相信很多人看完之后还是一脸懵逼，我也差不多，虽然说看上去都是合情合理，但是很多细节并没有深入去研究，虽然说http协议是一个文本协议，但是其解析处理也绝非易事。对于很多使用Go做Web开发的人来说，有必要简单了解一下，有助于了解网络请求的处理过程，加深对协议的理解。
