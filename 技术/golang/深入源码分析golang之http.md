---
sr-due: 2024-01-06
sr-interval: 1
sr-ease: 230
---

#golang   #note 


# Client

基于HTTP构建的服务标准模型包括两个端，客户端(`Client`)和服务端(`Server`)。HTTP 请求从客户端发出，服务端接受到请求后进行处理然后将响应返回给客户端。所以http服务器的工作就在于如何接受来自客户端的请求，并向客户端返回响应。

一个典型的 HTTP 服务应该如图所示：

![http](https://img.luozhiyun.com/20210608210147.png)

## HTTP client

在 Go 中可以直接通过 HTTP 包的 Get 方法来发起相关请求数据，一个简单例子：

```go
func main() {
    resp, err := http.Get("http://httpbin.org/get?name=luozhiyun&age=27")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer resp.Body.Close()
    body, _ := ioutil.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

我们下面通过这个例子来进行分析。

HTTP 的 Get 方法会调用到 DefaultClient 的 Get 方法，DefaultClient 是 Client 的一个空实例，所以最后会调用到 Client 的 Get 方法：

![httpclient](https://img.luozhiyun.com/20210608210156.png)

### Client 结构体

```go
type Client struct { 
    Transport RoundTripper 
    CheckRedirect func(req *Request, via []*Request) error 
    Jar CookieJar 
    Timeout time.Duration
}
```

Client 结构体总共由四个字段组成：

**Transport**：表示 HTTP 事务，用于处理客户端的请求连接并等待服务端的响应；

**CheckRedirect**：用于指定处理重定向的策略；

**Jar**：用于管理和存储请求中的 cookie；

**Timeout**：指定客户端请求的最大超时时间，该超时时间包括连接、任何的重定向以及读取相应的时间；

### 初始化请求

```go
func (c *Client) Get(url string) (resp *Response, err error) {
    // 根据方法名、URL 和请求体构建请求
    req, err := NewRequest("GET", url, nil)
    if err != nil {
        return nil, err
    }
    // 执行请求
    return c.Do(req)
}
```

我们要发起一个请求首先需要根据请求类型构建一个完整的请求头、请求体、请求参数。然后才是根据请求的完整结构来执行请求。

#### NewRequest 初始化请求

NewRequest 会调用到 NewRequestWithContext 函数上。这个函数会根据请求返回一个 Request 结构体，它里面包含了一个 HTTP 请求所有信息。

**Request**

Request 结构体有很多字段，我这里列举几个大家比较熟悉的字段：

![httpclient2](https://img.luozhiyun.com/20210608210223.png)

**NewRequestWithContext**

```go
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error) {
    ...
    // parse url
    u, err := urlpkg.Parse(url)
    if err != nil {
        return nil, err
    }
    rc, ok := body.(io.ReadCloser)
    if !ok && body != nil {
        rc = ioutil.NopCloser(body)
    } 
    u.Host = removeEmptyPort(u.Host)
    req := &Request{
        ctx:        ctx,
        Method:     method,
        URL:        u,
        Proto:      "HTTP/1.1",
        ProtoMajor: 1,
        ProtoMinor: 1,
        Header:     make(Header),
        Body:       rc,
        Host:       u.Host,
    } 
    ...
    return req, nil
}
```

NewRequestWithContext 函数会将请求封装成一个 Request 结构体并返回。

### 准备 http 发送请求

![httpclientsend](https://img.luozhiyun.com/20210608210229.png)

如上图所示，Client 调用 Do 方法处理发送请求最后会调用到 send 函数中。

```go
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    resp, didTimeout, err = send(req, c.transport(), deadline)
    if err != nil {
        return nil, didTimeout, err
    }
    ...
    return resp, nil, nil
}
```

#### Transport

Client 的 send 方法在调用 send 函数进行下一步的处理前会先调用 transport 方法获取 DefaultTransport 实例，该实例如下：

```go
var DefaultTransport RoundTripper = &Transport{
    // 定义 HTTP 代理策略
    Proxy: ProxyFromEnvironment,
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
        DualStack: true,
    }).DialContext,
    ForceAttemptHTTP2:     true,
    // 最大空闲连接数
    MaxIdleConns:          100,
    // 空闲连接超时时间
    IdleConnTimeout:       90 * time.Second,
    // TLS 握手超时时间
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

![transport](https://img.luozhiyun.com/20210608210234.png)

Transport 实现 RoundTripper 接口，该结构体会发送 http 请求并等待响应。

```go
type RoundTripper interface { 
    RoundTrip(*Request) (*Response, error)
}
```

从 RoundTripper 接口我们也可以看出，该接口定义的 RoundTrip 方法会具体的处理请求，处理完毕之后会响应 Response。

回到我们上面的 Client 的 send 方法中，它会调用 send 函数，这个函数主要逻辑都交给 Transport 的 RoundTrip 方法来执行。

![transport2](https://img.luozhiyun.com/20210608210244.png)

RoundTrip 会调用到 roundTrip 方法中：

```go
func (t *Transport) roundTrip(req *Request) (*Response, error) {
    t.nextProtoOnce.Do(t.onceSetNextProtoDefaults)
    ctx := req.Context()
    trace := httptrace.ContextClientTrace(ctx) 
    ...  
    for {
        select {
        case <-ctx.Done():
            req.closeBody()
            return nil, ctx.Err()
        default:
        }

        // 封装请求
        treq := &transportRequest{Request: req, trace: trace, cancelKey: cancelKey} 
        cm, err := t.connectMethodForRequest(treq)
        if err != nil {
            req.closeBody()
            return nil, err
        } 
        // 获取连接
        pconn, err := t.getConn(treq, cm)
        if err != nil {
            t.setReqCanceler(cancelKey, nil)
            req.closeBody()
            return nil, err
        }

        // 等待响应结果
        var resp *Response
        if pconn.alt != nil {
            // HTTP/2 path.
            t.setReqCanceler(cancelKey, nil) // not cancelable with CancelRequest
            resp, err = pconn.alt.RoundTrip(req)
        } else {
            resp, err = pconn.roundTrip(treq)
        }
        if err == nil {
            resp.Request = origReq
            return resp, nil
        } 
        ...
    }
}
```

roundTrip 方法会做两件事情：

1.  调用 Transport 的 getConn 方法获取连接；
2.  在获取到连接后，调用 persistConn 的 roundTrip 方法等待请求响应结果；

### 获取连接 getConn

getConn 有两个阶段：

1.  调用 queueForIdleConn 获取空闲 connection；
2.  调用 queueForDial 等待创建新的 connection；

![getconn4](https://img.luozhiyun.com/20210608210250.png)

```go
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (pc *persistConn, err error) {
    req := treq.Request
    trace := treq.trace
    ctx := req.Context()
    if trace != nil && trace.GetConn != nil {
        trace.GetConn(cm.addr())
    }   
    // 将请求封装成 wantConn 结构体
    w := &wantConn{
        cm:         cm,
        key:        cm.key(),
        ctx:        ctx,
        ready:      make(chan struct{}, 1),
        beforeDial: testHookPrePendingDial,
        afterDial:  testHookPostPendingDial,
    }
    defer func() {
        if err != nil {
            w.cancel(t, err)
        }
    }()

    // 获取空闲连接
    if delivered := t.queueForIdleConn(w); delivered {
        pc := w.pc
        ...
        t.setReqCanceler(treq.cancelKey, func(error) {})
        return pc, nil
    }

    // 创建连接
    t.queueForDial(w)

    select {
    // 获取到连接后进入该分支
    case <-w.ready:
        ...
        return w.pc, w.err
    ...
}
```

#### 获取空闲连接 queueForIdleConn

成功获取到空闲 connection：

![getconn](https://img.luozhiyun.com/20210608210254.png)

成功获取 connection 分为如下几步：

1.  根据当前的请求的地址去**空闲 connection 字典**中查看存不存在空闲的 connection 列表；
2.  如果能获取到空闲的 connection 列表，那么获取到列表的最后一个 connection；
3.  返回；

获取不到空闲 connection：

![getconn2](https://img.luozhiyun.com/20210608210258.png)

当获取不到空闲 connection 时：

1.  根据当前的请求的地址去**空闲 connection 字典**中查看存不存在空闲的 connection 列表；
2.  不存在该请求的 connection 列表，那么将该 wantConn 加入到 **等待获取空闲 connection 字典**中；

从上面的图解应该就很能看出这一步会怎么操作了，这里简要的分析一下代码，让大家更清楚里面的逻辑：

```go
func (t *Transport) queueForIdleConn(w *wantConn) (delivered bool) {
    if t.DisableKeepAlives {
        return false
    }

    t.idleMu.Lock()
    defer t.idleMu.Unlock() 
    t.closeIdle = false

    if w == nil { 
        return false
    }

    // 计算空闲连接超时时间
    var oldTime time.Time
    if t.IdleConnTimeout > 0 {
        oldTime = time.Now().Add(-t.IdleConnTimeout)
    }
    // Look for most recently-used idle connection.
    // 找到key相同的 connection 列表
    if list, ok := t.idleConn[w.key]; ok {
        stop := false
        delivered := false
        for len(list) > 0 && !stop {
            // 找到connection列表最后一个
            pconn := list[len(list)-1] 
            // 检查这个 connection 是不是等待太久了
            tooOld := !oldTime.IsZero() && pconn.idleAt.Round(0).Before(oldTime)
            if tooOld { 
                go pconn.closeConnIfStillIdle()
            }
            // 该 connection 被标记为 broken 或 闲置太久 continue
            if pconn.isBroken() || tooOld { 
                list = list[:len(list)-1]
                continue
            }
            // 尝试将该 connection 写入到 w 中
            delivered = w.tryDeliver(pconn, nil)
            if delivered {
                // 操作成功，需要将 connection 从空闲列表中移除
                if pconn.alt != nil { 
                } else { 
                    t.idleLRU.remove(pconn)
                    list = list[:len(list)-1]
                }
            }
            stop = true
        }
        if len(list) > 0 {
            t.idleConn[w.key] = list
        } else {
            // 如果该 key 对应的空闲列表不存在，那么将该key从字典中移除
            delete(t.idleConn, w.key)
        }
        if stop {
            return delivered
        }
    } 
    // 如果找不到空闲的 connection
    if t.idleConnWait == nil {
        t.idleConnWait = make(map[connectMethodKey]wantConnQueue)
    }
  // 将该 wantConn 加入到 等待获取空闲 connection 字典中
    q := t.idleConnWait[w.key] 
    q.cleanFront()
    q.pushBack(w)
    t.idleConnWait[w.key] = q
    return false
}
```

上面的注释已经很清楚了，我这里就不再解释了。

#### 建立连接 queueForDial

![getconn3](https://img.luozhiyun.com/20210608210303.png)

在获取不到空闲连接之后，会尝试去建立连接，从上面的图大致可以看到，总共分为以下几个步骤：

1.  在调用 queueForDial 方法的时候会校验 MaxConnsPerHost 是否未设置或已达上限；
    1.  检验不通过则将当前的请求放入到 connsPerHostWait 等待字典中；
2.  如果校验通过那么会异步的调用 dialConnFor 方法创建连接；
3.  dialConnFor 方法首先会调用 dialConn 方法创建 TCP 连接，然后启动两个异步线程来处理读写数据，然后调用 tryDeliver 将连接绑定到 wantConn 上面。

下面进行代码分析：

```go
func (t *Transport) queueForDial(w *wantConn) {
    w.beforeDial()
    // 小于零说明无限制，异步建立连接
    if t.MaxConnsPerHost <= 0 {
        go t.dialConnFor(w)
        return
    }

    t.connsPerHostMu.Lock()
    defer t.connsPerHostMu.Unlock()
    // 每个 host 建立的连接数没达到上限，异步建立连接
    if n := t.connsPerHost[w.key]; n < t.MaxConnsPerHost {
        if t.connsPerHost == nil {
            t.connsPerHost = make(map[connectMethodKey]int)
        }
        t.connsPerHost[w.key] = n + 1
        go t.dialConnFor(w)
        return
    }
    //每个 host 建立的连接数已达到上限，需要进入等待队列
    if t.connsPerHostWait == nil {
        t.connsPerHostWait = make(map[connectMethodKey]wantConnQueue)
    }
    q := t.connsPerHostWait[w.key]
    q.cleanFront()
    q.pushBack(w)
    t.connsPerHostWait[w.key] = q
}
```

这里主要进行参数校验，如果最大连接数限制为零，亦或是每个 host 建立的连接数没达到上限，那么直接异步建立连接。

**dialConnFor**

```go
func (t *Transport) dialConnFor(w *wantConn) {
    defer w.afterDial()
    // 建立连接
    pc, err := t.dialConn(w.ctx, w.cm)
    // 连接绑定 wantConn
    delivered := w.tryDeliver(pc, err)
    // 建立连接成功，但是绑定 wantConn 失败
    // 那么将该连接放置到空闲连接字典或调用 等待获取空闲 connection 字典 中的元素执行
    if err == nil && (!delivered || pc.alt != nil) { 
        t.putOrCloseIdleConn(pc)
    }
    if err != nil {
        t.decConnsPerHost(w.key)
    }
}
```

dialConnFor 会调用 dialConn 进行 TCP 连接创建，创建完毕之后调用 tryDeliver 方法和 wantConn 进行绑定。

**dialConn**

```go
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {
    // 创建连接结构体
    pconn = &persistConn{
        t:             t,
        cacheKey:      cm.key(),
        reqch:         make(chan requestAndChan, 1),
        writech:       make(chan writeRequest, 1),
        closech:       make(chan struct{}),
        writeErrCh:    make(chan error, 1),
        writeLoopDone: make(chan struct{}),
    }
    ...
    if cm.scheme() == "https" && t.hasCustomTLSDialer() {
        ...
    } else {
        // 建立 tcp 连接
        conn, err := t.dial(ctx, "tcp", cm.addr())
        if err != nil {
            return nil, wrapErr(err)
        }
        pconn.conn = conn 
    } 
    ...

    if s := pconn.tlsState; s != nil && s.NegotiatedProtocolIsMutual && s.NegotiatedProtocol != "" {
        if next, ok := t.TLSNextProto[s.NegotiatedProtocol]; ok {
            alt := next(cm.targetAddr, pconn.conn.(*tls.Conn))
            if e, ok := alt.(http2erringRoundTripper); ok {
                // pconn.conn was closed by next (http2configureTransport.upgradeFn).
                return nil, e.err
            }
            return &persistConn{t: t, cacheKey: pconn.cacheKey, alt: alt}, nil
        }
    }

    pconn.br = bufio.NewReaderSize(pconn, t.readBufferSize())
    pconn.bw = bufio.NewWriterSize(persistConnWriter{pconn}, t.writeBufferSize())
    //为每个连接异步处理读写数据
    go pconn.readLoop()
    go pconn.writeLoop()
    return pconn, nil
}
```

这里会根据 schema 的不同设置不同的连接配置，我上面显示的是我们常用的 HTTP 连接的创建过程。对于 HTTP 来说会建立 tcp 连接，然后为连接异步处理读写数据，最后将创建好的连接返回。

### 等待响应

这一部分的内容会稍微复杂一些，但确实非常的有趣。

![response](https://img.luozhiyun.com/20210608210309.png)

在创建连接的时候会初始化两个 channel ：writech 负责写入请求数据，reqch负责读取响应数据。我们在上面创建连接的时候，也提到了会为连接创建两个异步循环 readLoop 和 writeLoop 来负责处理读写数据。

在获取到连接之后，会调用连接的 roundTrip 方法，它首先会将请求数据写入到 writech 管道中，writeLoop 接收到数据之后就会处理请求。

然后 roundTrip 会将 requestAndChan 结构体写入到 reqch 管道中，然后 roundTrip 会循环等待。readLoop 读取到响应数据之后就会通过 requestAndChan 结构体中保存的管道将数据封装成 responseAndError 结构体回写，这样 roundTrip 就可以接受到响应数据结束循环等待并返回。

**roundTrip**

```go
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    ...
    writeErrCh := make(chan error, 1)
    // 将请求数据写入到 writech 管道中
    pc.writech <- writeRequest{req, writeErrCh, continueCh}

    // 用于接收响应的管道
    resc := make(chan responseAndError)
    // 将用于接收响应的管道封装成 requestAndChan 写入到 reqch 管道中
    pc.reqch <- requestAndChan{
        req:        req.Request,
        cancelKey:  req.cancelKey,
        ch:         resc,
        ...
    }
    ...
    for {
        testHookWaitResLoop()
        select { 
        // 接收到响应数据
        case re := <-resc:
            if (re.res == nil) == (re.err == nil) {
                panic(fmt.Sprintf("internal error: exactly one of res or err should be set; nil=%v", re.res == nil))
            }
            if debugRoundTrip {
                req.logf("resc recv: %p, %T/%#v", re.res, re.err, re.err)
            }
            if re.err != nil {
                return nil, pc.mapRoundTripError(req, startBytesWritten, re.err)
            }
            // 返回响应数据
            return re.res, nil
        ...
    }
}
```

这里会封装好 writeRequest 作为发送请求的数据，并将用于接收响应的管道封装成 requestAndChan 写入到 reqch 管道中，然后循环等待接受响应。

然后 writeLoop 会进行请求数据 writeRequest ：

```go
func (pc *persistConn) writeLoop() {
    defer close(pc.writeLoopDone)
    for {
        select {
        case wr := <-pc.writech:
            startBytesWritten := pc.nwrite
            // 向 TCP 连接中写入数据，并发送至目标服务器
            err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
            ...
        case <-pc.closech:
            return
        }
    }
}
```

这里会将从 writech 管道中获取到的数据写入到 TCP 连接中，并发送至目标服务器。  
**readLoop**

```go
func (pc *persistConn) readLoop() {
    closeErr := errReadLoopExiting // default value, if not changed below
    defer func() {
        pc.close(closeErr)
        pc.t.removeIdleConn(pc)
    }()
    ... 
    alive := true
    for alive {
        pc.readLimit = pc.maxHeaderResponseSize()
        // 获取 roundTrip 发送的结构体
        rc := <-pc.reqch
        trace := httptrace.ContextClientTrace(rc.req.Context())

        var resp *Response
        if err == nil {
            // 读取数据
            resp, err = pc.readResponse(rc, trace)
        } else {
            err = transportReadFromServerError{err}
            closeErr = err
        }

        ...  
        // 将响应数据写回到管道中
        select {
        case rc.ch <- responseAndError{res: resp}:
        case <-rc.callerGone:
            return
        }
        ...
    }
}
```

这里是从 TCP 连接中读取到对应的请求响应数据，通过 roundTrip 传入的管道再回写，然后 roundTrip 就会接受到数据并获取的响应数据返回。

# Server

在实现上面我先用一张图进行简要的介绍一下：

![server2](https://img.luozhiyun.com/20210608210316.png)

其实我们从上面例子的方法名就可以知道一些大致的步骤：

1.  注册处理器到一个 hash 表中，可以通过键值路由匹配；
2.  注册完之后就是开启循环监听，每监听到一个连接就会创建一个 Goroutine；
3.  在创建好的 Goroutine 里面会循环的等待接收请求数据，然后根据请求的地址去处理器路由表中匹配对应的处理器，然后将请求交给处理器处理；


## 一、快速搭建web服务器

> 在Golang仅需要几行代码，便可以建立一个简单的 Web 服务。如下，运行程序后在浏览器输入 [http://localhost:8888/hello](https://link.zhihu.com/?target=http%3A//localhost%3A8888/hello) 你将会在浏览器中看到 hello world，证明该简易版的web服务器搭建成功。

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/hello", func(writer http.ResponseWriter, request *http.Request) {
		fmt.Fprintf(writer, "hello world")
	})

	http.ListenAndServe(":8888", nil)
}
```

## 二、结构体分析

```go
type Server struct { 
	// Addr可以选择指定服务器侦听的TCP地址
        // 形式为“ host:port” 如果为空，则使用“:http”（端口80）
	Addr string
 
        // http server的处理程序，如果为nil 则默认为http.DefaultServeMux
        // 实现了ServeHTTP方法的接口。
	Handler Handler 
  
	// TLSConfig可选地提供TLS配置供使用,通过ServeTLS和ListenAndServeTLS
	TLSConfig *tls.Config
 
	// ReadTimeout的时间计算是从连接被接受(accept包括tcp连接时间)到request body完全被读取
	ReadTimeout time.Duration

	// ReadHeaderTimeout 的时间计算是从连接被接受(accept包括tcp连接时间)到request header完全被读取
	ReadHeaderTimeout time.Duration

        // WriteTimeout的时间计算正常是从request header(包括tcp连接时间)的读取结束开始，到response write结束为止
	WriteTimeout time.Duration

	// IdleTimeout是一个空闲连接等待的最长时间
	IdleTimeout time.Duration

	// MaxHeaderBytes 控制服务器取解析请求标头的键和值的最大字节数，不包括请求body
        // 默认为DefaultMaxHeaderBytes字节
	MaxHeaderBytes int

	// 禁用HTTP/2的程序可以通过将Transport.TLSNextProto（对于客户端或Server.TLSNextProto（对于服务器）设置为非零空映射来实现。
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)

	// ConnState指定一个可选的回调函数，当客户端连接更改状态时调用
	ConnState func(net.Conn, ConnState)

	// ErrorLog，处理错误请求以及异常请求的记录器，潜在的FileSystem错误
        // 如果为nil，则通过日志包的标准记录器完成记录。
	ErrorLog *log.Logger

	// 指定一个返回函数，返回服务器上下文。
        // 如果BaseContext为nil，则默认值为context.Background()
        // 如果为非nil，则必须返回非nil上下文
	BaseContext func(net.Listener) context.Context

        // ConnContext可选地指定一个修改函数，用于新连接的上下文c。提供的ctx
        // 从基本上下文派生，并具有ServerContextKey值
	ConnContext func(ctx context.Context, c net.Conn) context.Context

        // 当服务器关闭时为true
	inShutdown atomicBool 
        // 原子访问。
	disableKeepAlives int32   
        // http2.0 协议相关
	nextProtoOnce     sync.Once 
        // 如果使用了http2.ConfigureServer的结果
	nextProtoErr      error   
	// 互斥锁
	mu         sync.Mutex
        // 监听器map
	listeners  map[*net.Listener]struct{}
        // 连接map
	activeConn map[*conn]struct{}
        //接收关闭连接channel
	doneChan   chan struct{} 
        // 中止连接的函数切片
	onShutdown []func()
}
```

## 三、源码分析

> 搭建web服务器的重要代码： `http.ListenAndServe(":8888", nil)`，下面来源码分析下所经历的步骤。

### 1、注册路由

> `net/http`包暴露的注册路由API很简单，利用`http.HandleFunc` 来完成注册路由功能。`http.HandleFunc`是一个函数类型，并实现了Handler接口，当通过调用HandlerFunc(),把pattern强转为HandlerFunc类型时，就意味着pattern函数也实现了ServeHttp方法。

```go
type ServeMux struct {
	mu    sync.RWMutex //读写锁
	m     map[string]muxEntry //用来管理所有注册路由的map
	es    []muxEntry //按照pattern长度降序排列，记录值均以/结尾
	hosts bool       // 是否存在hosts, 即不以'/'开头的pattern，http://localhost/hello
}

type muxEntry struct {
	h       Handler // handler处理
	pattern string // 路由
}

var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	// 加锁
        mux.mu.Lock()
	defer mux.mu.Unlock()
        // 判断路由匹配
	if pattern == "" {
		panic("http: invalid pattern")
	}
        // 判断handler函数
	if handler == nil {
		panic("http: nil handler")
	}
        // 多次注册了 
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}
        // 初始化map
	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
        //添加路由
	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
        // 以斜杠结尾的pattern将存入es切片并按pattern长度降序排列
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}
        // 不以"/"开头的模式将视作存在hosts
	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

### 2、监听端口

> `ListenAndServe`实际上，是初始化一个server对象，调用了 server 的 ListenAndServe 方法。追进去发现，该方法调用了`net.Listen`方法，表示创建了一个监听端口。之后调用`src.Serve`进入到下一步请求接收过程。

```go
func ListenAndServe(addr string, handler Handler) error {
        // 初始化 Server事例
	server := &Server{Addr: addr, Handler: handler}
        // 调用 server的ListenAndServe方法
	return server.ListenAndServe()
}
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() { // 服务突然中断或者关闭返回 http: Server closed
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
        // 监听tcp请求，主要调用了 func (lc *ListenConfig) Listen，
        // 代码位于 /src/net/dial.go
	ln, err := net.Listen("tcp", addr) 
	if err != nil {
		return err
	}
	return srv.Serve(ln) // for循环接收请求
}
```

### 3、接收请求

> 服务器在创建监听器成功之后，阻塞的去接收客户端的连接请求 `l.Accept` ，为每一个客户端发起的请求初始化一个`Conn`连接，并设置为New状态，之后为每个连接创建一个goroutine 用来真正的处理客户端请求。

```go
func (srv *Server) Serve(l net.Listener) error {
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // 钩子函数
	}

	origListener := l 
        // onceCloseListener包裹了net.listener监听器，用以保护它免受多个关闭调用。
        // 结构体里面维护了一个sync.Once保证一次性执行
	l = &onceCloseListener{Listener: l}
	defer l.Close()
        // http2.0相关配置
	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}
        // trackListener将net.Listener添加或删除到被跟踪的集合中
        // true是添加，false表示删除
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
        // 用完删除掉
	defer srv.trackListener(&l, false)
        // 返回服务器的上下文，此处为根的context
	baseCtx := context.Background()
	if srv.BaseContext != nil { //不为空
		baseCtx = srv.BaseContext(origListener) //返回根context
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}
        // 接受失败要睡眠多长时间
	var tempDelay time.Duration 
        // 在baseCtx的基础上创建一个传递key为ServerContextKey，value为srv的上下文
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
                // 阻塞接受客户端连接请求
		rw, err := l.Accept()
		if err != nil { // 连接出现错误
			select {
			case <-srv.getDoneChan(): //监听doneChan channel
				return ErrServerClosed
			default:
			}
            // 判断tcp连接的错误码，并且是临时错误，进入重试机制
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay) 
				continue
			}
			return err //反之直接返回错误信息
		}
	    // 传递给处理连接的goroutine的上下文
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0 // 将睡眠时间置为0
		c := srv.newConn(rw) // 创建一个新连接
		c.setState(c.rwc, StateNew) // 标记为新连接，添加到map[*conn]struct{}中
		go c.serve(connCtx) // 为每个请求开启一个goroutine
	}
}
```

### 4、读取请求

> `readRequest` 便是读取数据，解析请求的地方，包括解析请求的header、body，和一些基本的校验，比如header头信息，请求method等。最后将请求的数据赋值到Request，并初始化Response对象，供业务层调用。

```go
func (c *conn) serve(ctx context.Context) {
        // 获取远程地址以及新建一个父级为ctx的context上下文
        // key为LocalAddrContextKey，value为c.rwc.LocalAddr()为本地网络地址
	c.remoteAddr = c.rwc.RemoteAddr().String()
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
        
        // 恐慌处理
	defer func() {
                // ErrAbortHandler：中止处理程序的恐慌值
		if err := recover(); err != nil && err != ErrAbortHandler {
			const size = 64 << 10
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() { //连接是否被劫持，hijackedv由mu保护
			c.close() //关闭连接
			c.setState(c.rwc, StateClosed) //删除这个链接
		}
	}()

        // 处理https的
	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
                // 设置读超时时间
		if d := c.server.ReadTimeout; d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
                // 设置写超时时间
		if d := c.server.WriteTimeout; d != 0 {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
                // 握手协议失败，记录错误信息
		if err := tlsConn.Handshake(); err != nil {
                        // https记录头无效，
			if re, ok := err.(tls.RecordHeaderError); ok && re.Conn != nil && tlsRecordHeaderLooksLikeHTTP(re.RecordHeader) {
				io.WriteString(re.Conn, "HTTP/1.0 400 Bad Request\r\n\r\nClient sent an HTTP request to an HTTPS server.\n")
				re.Conn.Close()
				return
			}
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
                // 记录tls连接信息
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState()
                // 验证alpn协议
		if proto := c.tlsState.NegotiatedProtocol; validNextProto(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initALPNRequest{ctx, tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h)
			}
			return
		}
	}

	// http1.x
        // 创建一个用于取消的上下文，父级为ctx
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()
        // 初始化一个基于连接的读取器
	c.r = &connReader{conn: c}
        // 读缓冲区
	c.bufr = newBufioReader(c.r)
        // 写缓冲区
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
                // 从conn中读取一个请求
		w, err := c.readRequest(ctx)
		if c.r.remain != c.server.initialReadLimitSize() {
			// 标记处于活动状态
			c.setState(c.rwc, StateActive)
		}
		if err != nil {
			const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection: close\r\n\r\n"

			switch {
			case err == errTooLarge: // 请求过大
				const publicErr = "431 Request Header Fields Too Large"
				fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
				//刷新所有未完成的包，并发送FIN数据包，然后暂停一会，
                                //希望客户端处理随后它发出来的RST数据包
                                c.closeWriteAndWait()  
				return

			case isUnsupportedTEError(err): //非nil错误类型
				code := StatusNotImplemented // 服务端501，发送请求关闭连接
				fmt.Fprintf(c.rwc, "HTTP/1.1 %d %s%sUnsupported transfer encoding", code, StatusText(code), errorHeaders)
				return
                        // 读取失败，直接返回
			case isCommonNetReadError(err):
				return 
                        // 默认400 bad request
			default:
				publicErr := "400 Bad Request"
				if v, ok := err.(badRequestError); ok {
					publicErr = publicErr + ": " + string(v)
				}

				fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
				return
			}
		}

		// 100-continue
		req := w.req
		if req.expectsContinue() { // 头部 Expect:100-continue
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// 将Body读取器与对连接有答复的一个包装在一起
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
				w.canWriteContinue.setTrue()
			}
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed() 
			return
		}
		c.curReq.Store(w) //response中存储一个请求
		if requestBodyRemains(req.Body) {
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			w.conn.r.startBackgroundRead()
		}
                // http不能同时有多个活动请求
                // 在服务器回复此请求之间，它无法读取其他请求
                // 路由处理
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx() 
		if c.hijacked() {
			return
		}
		w.finishRequest() // 响应请求
                // 是否可以重用基础的TCP连接
		if !w.shouldReuseConnection() { 
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				c.closeWriteAndWait()
			}
			return
		}
		c.setState(c.rwc, StateIdle) //设置连接为完成状态
		c.curReq.Store((*response)(nil))
                //没有开启keepalive，则关闭连接    
		if !w.conn.server.doKeepAlives() {
			return
		}

		if d := c.server.idleTimeout(); d != 0 {
                        //设置读超时时间为idleTimeout
			c.rwc.SetReadDeadline(time.Now().Add(d))
                        //阻塞读，待idleTimeout超时后，返回错误
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
		}
                //接收到新的请求，清空ReadTimeout
		c.rwc.SetReadDeadline(time.Time{})
	}
}


func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
	if c.hijacked() {
		return nil, ErrHijacked
	}
	var (
		wholeReqDeadline time.Time 
		hdrDeadline      time.Time 
	)
	t0 := time.Now()
        // 读取server配置的从建立连接到读取请求头的时间
        // 如果没有设置，则获取到server配置的从建立连接到读取请求体的时间
	if d := c.server.readHeaderTimeout(); d != 0 {
		hdrDeadline = t0.Add(d)
	}
        // 读取server配置的从建立连接到读取请求体的时间
	if d := c.server.ReadTimeout; d != 0 {
		wholeReqDeadline = t0.Add(d)
	}
        // SetReadDeadline设置将来的Read调用的截止日期，
        // t的值为零表示读取不会超时。
	c.rwc.SetReadDeadline(hdrDeadline)
        // 如果serve配置的数据响应时间
	if d := c.server.WriteTimeout; d != 0 {
		defer func() {
            // 如果不为0，设置数据响应时间
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}()
	}
        // 设置读请求的大小
	c.r.setReadLimit(c.server.initialReadLimitSize())
	if c.lastMethod == "POST" {
		peek, _ := c.bufr.Peek(4) 
		c.bufr.Discard(numLeadingCRorLF(peek))
	}
        // 读取request
	req, err := readRequest(c.bufr, keepHostHeader)
	if err != nil {
                // http: request too large
		if c.r.hitReadLimit() {
			return nil, errTooLarge
		}
		return nil, err
	}
        // unsupported protocol version
	if !http1ServerSupportsRequest(req) {
		return nil, badRequestError("unsupported protocol version")
	}

	c.lastMethod = req.Method
	c.r.setInfiniteReadLimit()
        // 读取头部信息
	hosts, haveHost := req.Header["Host"]
	isH2Upgrade := req.isH2Upgrade()
	if req.ProtoAtLeast(1, 1) && (!haveHost || len(hosts) == 0) && !isH2Upgrade && req.Method != "CONNECT" {
		return nil, badRequestError("missing required Host header")
	}
	if len(hosts) > 1 {
		return nil, badRequestError("too many Host headers")
	}
	if len(hosts) == 1 && !httpguts.ValidHostHeader(hosts[0]) {
		return nil, badRequestError("malformed Host header")
	}
	for k, vv := range req.Header {
		if !httpguts.ValidHeaderFieldName(k) {
			return nil, badRequestError("invalid header name")
		}
		for _, v := range vv {
			if !httpguts.ValidHeaderFieldValue(v) {
				return nil, badRequestError("invalid header value")
			}
		}
	}
	delete(req.Header, "Host")

	ctx, cancelCtx := context.WithCancel(ctx)
	req.ctx = ctx
	req.RemoteAddr = c.remoteAddr
	req.TLS = c.tlsState
	if body, ok := req.Body.(*body); ok {
		body.doEarlyClose = true
	}
        // 如有必要，调整读取期限
	if !hdrDeadline.Equal(wholeReqDeadline) {
		c.rwc.SetReadDeadline(wholeReqDeadline)
	}
        
        // 构造response响应结构体
	w = &response{
		conn:          c,
		cancelCtx:     cancelCtx,
		req:           req,
		reqBody:       req.Body,
		handlerHeader: make(Header),
		contentLength: -1,
		closeNotifyCh: make(chan bool, 1),
		wants10KeepAlive: req.wantsHttp10KeepAlive(),
		wantsClose:       req.wantsClose(),
	}
	if isH2Upgrade {
		w.closeAfterReply = true
	}
	w.cw.res = w
        // response bufio
	w.w = newBufioWriterSize(&w.cw, bufferBeforeChunkingSize)
	return w, nil
}
```

### 5、路由处理

> 在路由注册过程中，每次调用 `http.HandleFunc()`函数 ，做的事情都很简单，就是维护一个内部的哈希表 `m`, 并没有涉及到如何处理路由，真正开始处理路由的应该是在读取完请求之后调用的 `serverHandler{c.server}.ServeHTTP(w, w.req)`这里来实现路由处理。

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	// 传入实现ServeHTTP方法的handler
        handler := sh.srv.Handler 
	if handler == nil { 
                // 默认的处理handler
		handler = DefaultServeMux
	}
        // 响应options * 请求
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
        // DefaultServeMux实现了ServeHTTP方法
	handler.ServeHTTP(rw, req)
}
// ServeMux 实现了 ServeHTTP
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
                // 400 bad request
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
        //根据不同情况处理路由，RedirectHandler，Handler，NotFoundHandler都实现了ServeHTTP
	h.ServeHTTP(w, r) 
}

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	// connect请求未规范
	if r.Method == "CONNECT" {
                // 规范路由中的“/”
		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
		       // 返回重定向处理程序
                       return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
		}
                // handler和pattern
		return mux.handler(r.Host, r.URL.Path)
	}

	host := stripHostPort(r.Host) // 返回不带port的host
	path := cleanPath(r.URL.Path) // cleanPath返回p的规范路径，消除了.和..元素。
        // 规范路由中的“/”
	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
		// 返回重定向处理程序
               return Redirec tHandler(u.String(), StatusMovedPermanently), u.Path
	}
        // 规范后的path跟url.path不一致
	if path != r.URL.Path {
		_, pattern = mux.handler(host, path)
		url := *r.URL
		url.Path = path
                // 返回重定向处理程序
		return RedirectHandler(url.String(), StatusMovedPermanently), pattern
	}
        // handler和pattern
	return mux.handler(host, r.URL.Path)
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()
	// 特定于主机的模式优先于通用模式
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
                // 404 no found ， HandlerFunc(NotFound)
		h, pattern = NotFoundHandler(), ""
	}
	return
}

func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// 检查完全匹配的路由.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}
        // 检查最长的有效匹配项，mux.es包含所有模式
        // 的结尾/从最长到最短排序。
	for _, e := range mux.es {
                // 前缀匹配
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
 
```

# 关于为什么要Close
  
## 前言

首先来看看一个最常出错的地方，net/http包中，Response结构体Body的处理方法不当所导致的内存泄露问题

##  为什么要Close？

首先来看一段代码：

```go
package main  
  
import (  
	"fmt"  
	"io/ioutil"  
	"net"  
	"net/http"  
	"time"  
)  
  
func PrintLocalDial(network, addr string) (net.Conn, error) {  
	dial := net.Dialer{  
		Timeout:   30 * time.Second,  
		KeepAlive: 30 * time.Second,  
	}  
  
	conn, err := dial.Dial(network, addr)  
	if err != nil {  
		return conn, err  
	}  
  
	fmt.Println("connect done, use", conn.LocalAddr().String())  
  
	return conn, err  
}  
  
func doGet(client *http.Client, url string, id int) {  
	resp, err := client.Get(url)  
	if err != nil {  
		fmt.Println(err)  
		return  
	}  
	// io.Copy(ioutil.Discard, resp.Body)  
	// fmt.Println("copy")  
	buf, err := ioutil.ReadAll(resp.Body)  
	fmt.Printf("%d: %s -- %v\n", id, string(buf[0:1]), err)  
	if err := resp.Body.Close(); err == nil {  
		fmt.Println("close")  
	}  
}  
  
func main() {  
	client := &http.Client{  
		Transport: &http.Transport{  
			Dial: PrintLocalDial,  
		},  
	}  
	const URL = "https://www.baidu.com/"  
  
	for {  
		go doGet(client, URL, 1)  
		go doGet(client, URL, 2)  
		time.Sleep(2 * time.Second)  
	}  
}  
```

这段代码是标准的写法，首先发出Get请求，然后读Body中的数据，最后将Body用Close方法关闭

相信有不少人在首次编写类似程序的时候，一定有人告诉过你，一定要注意将Body关闭，不然会导致内存泄露的问题,那么，事实是否真的是这样呢？

我们来看看源码（Go 1.13）

首先我们来看看Do方法，只挑重点的代码段：
```go
func (c *Client) Do(req *Request) (*Response, error) {  
	return c.do(req)  
}  
```

可以看到直接调用了私有方法do，再看看这个：
```go
func (c *Client) do(req *Request) (retres *Response, reterr error) {  
	if testHookClientDoResult != nil {  
		defer func() { testHookClientDoResult(retres, reterr) }()  
	}  
	if req.URL == nil {  
		req.closeBody()  
		return nil, &url.Error{  
			Op:  urlErrorOp(req.Method),  
			Err: errors.New("http: nil Request.URL"),  
		}  
	}  
  
	var (  
		deadline      = c.deadline()  
		reqs          []*Request  
		resp          *Response  
		copyHeaders   = c.makeHeadersCopier(req)  
		reqBodyClosed = false // have we closed the current req.Body?  
  
		// Redirect behavior:  
		redirectMethod string  
		includeBody    bool  
	)  
	uerr := func(err error) error {  
		// the body may have been closed already by c.send()  
		if !reqBodyClosed {  
			req.closeBody()  
		}  
		var urlStr string  
		if resp != nil && resp.Request != nil {  
			urlStr = stripPassword(resp.Request.URL)  
		} else {  
			urlStr = stripPassword(req.URL)  
		}  
		return &url.Error{  
			Op:  urlErrorOp(reqs[0].Method),  
			URL: urlStr,  
			Err: err,  
		}  
	}  
	for {  
		// For all but the first request, create the next  
		// request hop and replace req.  
		if len(reqs) > 0 {  
		  //省略了，不重要，看下面.......  
               }  
	//重点  
	reqs = append(reqs, req)  
		var err error  
		var didTimeout func() bool  
		//重点是下面的send方法  
		if resp, didTimeout, err = c.send(req, deadline); err != nil {  
			// c.send() always closes req.Body  
			reqBodyClosed = true  
			if !deadline.IsZero() && didTimeout() {  
				err = &httpError{  
	// TODO: early in cycle: s/Client.Timeout exceeded/timeout or context cancellation/  
		      err:     err.Error() + " (Client.Timeout exceeded while awaiting headers)",  
		      timeout: true,  
				}  
			}  
			return nil, uerr(err)  
		}  
  
		var shouldRedirect bool  
	redirectMethod, shouldRedirect, includeBody = redirectBehavior(req.Method, resp, reqs[0])  
		if !shouldRedirect {  
			return resp, nil  
		}  
  
		req.closeBody()
```

可以看到，在调用do方法时，首先会调用send方法，我们再来看看send方法
```go
// didTimeout is non-nil only if err != nil.  
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {  
	if c.Jar != nil {  
		for _, cookie := range c.Jar.Cookies(req.URL) {  
			req.AddCookie(cookie)  
		}  
	}  
	//调用了send函数  
	resp, didTimeout, err = send(req, c.transport(), deadline)  
	if err != nil {  
		return nil, didTimeout, err  
	}  
	if c.Jar != nil {  
		if rc := resp.Cookies(); len(rc) > 0 {  
			c.Jar.SetCookies(req.URL, rc)  
		}  
	}  
	return resp, nil, nil  
}
```
又再次调用了send函数，继续：
```go
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {  
	req := ireq // req is either the original request, or a modified fork  
  
	if rt == nil {  
		req.closeBody()  
		return nil, alwaysFalse, errors.New("http: no Client.Transport or DefaultTransport")  
	}  
	//省略  
        //重点  
	stopTimer, didTimeout := setRequestCancel(req, rt, deadline)  
	//调用了rt的RoundTrip方法  
	resp, err = rt.RoundTrip(req)  
	if err != nil {  
		stopTimer()  
		if resp != nil {  
			log.Printf("RoundTripper returned a response & error; ignoring response")  
		}  
		if tlsErr, ok := err.(tls.RecordHeaderError); ok {  
			// If we get a bad TLS record header, check to see if the  
			// response looks like HTTP and give a more helpful error.  
			// See golang.org/issue/11111.  
			if string(tlsErr.RecordHeader[:]) == "HTTP/" {  
				err = errors.New("http: server gave HTTP response to HTTPS client")  
			}  
		}  
		return nil, didTimeout, err
```

可以看到，send函数实际上调用了RoundTripper这个interface的RoundTrip方法，也就是上面c.transport()类型所实现的RoundTrip方法，那么c.transport()返回值是什么类型呢？来看看：
```go
func (c *Client) transport() RoundTripper {  
	if c.Transport != nil {  
		return c.Transport  
	}  
	return DefaultTransport  
}  
var DefaultTransport RoundTripper = &Transport{  
	Proxy: ProxyFromEnvironment,  
	DialContext: (&net.Dialer{  
		Timeout:   30 * time.Second,  
		KeepAlive: 30 * time.Second,  
		DualStack: true,  
	}).DialContext,  
	ForceAttemptHTTP2:     true,  
	MaxIdleConns:          100,  
	IdleConnTimeout:       90 * time.Second,  
	TLSHandshakeTimeout:   10 * time.Second,  
	ExpectContinueTimeout: 1 * time.Second,  
}
```

我们发现是Transport类型，那来看看Transport类型是如何实现RoundTrip方法的：
```go
// roundTrip implements a RoundTripper over HTTP.  
func (t *Transport) roundTrip(req *Request) (*Response, error) {  
	t.nextProtoOnce.Do(t.onceSetNextProtoDefaults)  
	ctx := req.Context()  
	trace := httptrace.ContextClientTrace(ctx)  
  
	if req.URL == nil {  
		req.closeBody()  
		return nil, errors.New("http: nil Request.URL")  
	}  
	if req.Header == nil {  
		req.closeBody()  
		return nil, errors.New("http: nil Request.Header")  
	}  
	scheme := req.URL.Scheme  
	isHTTP := scheme == "http" || scheme == "https"  
	if isHTTP {  
		for k, vv := range req.Header {  
			if !httpguts.ValidHeaderFieldName(k) {  
				return nil, fmt.Errorf("net/http: invalid header field name %q", k)  
			}  
			for _, v := range vv {  
				if !httpguts.ValidHeaderFieldValue(v) {  
		return nil, fmt.Errorf("net/http: invalid header field value %q for key %v", v, k)  
				}  
			}  
		}  
	}  
  
	if t.useRegisteredProtocol(req) {  
		altProto, _ := t.altProto.Load().(map[string]RoundTripper)  
		if altRT := altProto[scheme]; altRT != nil {  
			if resp, err := altRT.RoundTrip(req); err != ErrSkipAltProtocol {  
				return resp, err  
			}  
		}  
	}  
	if !isHTTP {  
		req.closeBody()  
		return nil, &badStringError{"unsupported protocol scheme", scheme}  
	}  
	if req.Method != "" && !validMethod(req.Method) {  
		return nil, fmt.Errorf("net/http: invalid method %q", req.Method)  
	}  
	if req.URL.Host == "" {  
		req.closeBody()  
		return nil, errors.New("http: no Host in request URL")  
	}  
  
	for {  
		select {  
		case <-ctx.Done():  
			req.closeBody()  
			return nil, ctx.Err()  
		default:  
		}  
  
		// treq gets modified by roundTrip, so we need to recreate for each retry.  
		treq := &transportRequest{Request: req, trace: trace}  
		cm, err := t.connectMethodForRequest(treq)  
		if err != nil {  
			req.closeBody()  
			return nil, err  
		}  
  
		// Get the cached or newly-created connection to either the  
		// host (for http or https), the http proxy, or the http proxy  
		// pre-CONNECTed to https server. In any case, we'll be ready  
		// to send it requests.  
		//重点是这个函数getConn  
		pconn, err := t.getConn(treq, cm)  
		if err != nil {  
			t.setReqCanceler(req, nil)  
			req.closeBody()  
			return nil, err  
		}  
  
		var resp *Response  
		if pconn.alt != nil {  
			// HTTP/2 path.  
			t.setReqCanceler(req, nil) // not cancelable with CancelRequest  
			resp, err = pconn.alt.RoundTrip(req)  
		} else {  
			resp, err = pconn.roundTrip(treq)  
		}  
		if err == nil {  
			return resp, nil  
		}  
		if http2isNoCachedConnError(err) {  
			t.removeIdleConn(pconn)  
		} else if !pconn.shouldRetryRequest(req, err) {  
			// Issue 16465: return underlying net.Conn.Read error from peek,  
			// as we've historically done.  
			if e, ok := err.(transportReadFromServerError); ok {  
				err = e.err  
			}  
			return nil, err  
		}  
		testHookRoundTripRetried()  
  
		// Rewind the body if we're able to.  
		if req.GetBody != nil {  
			newReq := *req  
			var err error  
			newReq.Body, err = req.GetBody()  
			if err != nil {  
				return nil, err  
			}  
			req = &newReq  
		}  
	}  
}
```

可以看到主要是调用了getConn方法返回一个\*persistConn类型的变量，继续跟进看看getConn是如何实现的
```go
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (pc *persistConn, err error) {  
	req := treq.Request  
	trace := treq.trace  
	ctx := req.Context()  
	if trace != nil && trace.GetConn != nil {  
		trace.GetConn(cm.addr())  
	}  
  
	w := &wantConn{  
		cm:         cm,  
		key:        cm.key(),  
		ctx:        ctx,  
		ready:      make(chan struct{}, 1),  
		beforeDial: testHookPrePendingDial,  
		afterDial:  testHookPostPendingDial,  
	}  
	defer func() {  
		if err != nil {  
			w.cancel(t, err)  
		}  
	}()  
	//尝试去空闲的连接池中寻找  
	//client针对每个host最多可以分别复用两个连接  
	// Queue for idle connection.  
	if delivered := t.queueForIdleConn(w); delivered {  
		pc := w.pc  
		if trace != nil && trace.GotConn != nil {  
			trace.GotConn(pc.gotIdleConnTrace(pc.idleAt))  
		}  
		// set request canceler to some non-nil function so we  
		// can detect whether it was cleared between now and when  
		// we enter roundTrip  
		t.setReqCanceler(req, func(error) {})  
		return pc, nil  
	}  
  
	cancelc := make(chan error, 1)  
	t.setReqCanceler(req, func(err error) { cancelc <- err })  
	//当没有在连接池中找到可用连接时，新建一个  
	// Queue for permission to dial.  
	t.queueForDial(w)  
       // Wait for completion or cancellation.  
	select {  
	case <-w.ready:  
		// Trace success but only for HTTP/1.  
		// HTTP/2 calls trace.GotConn itself.  
	if w.pc != nil && w.pc.alt == nil && trace != nil && trace.GotConn != nil {  
		trace.GotConn(httptrace.GotConnInfo{Conn: w.pc.conn, Reused: w.pc.isReused()})  
		}  
		if w.err != nil {  
			// If the request has been cancelled, that's probably  
			// what caused w.err; if so, prefer to return the  
			// cancellation error (see golang.org/issue/16049).  
			select {  
			case <-req.Cancel:  
				return nil, errRequestCanceledConn  
			case <-req.Context().Done():  
				return nil, req.Context().Err()  
			case err := <-cancelc:  
				if err == errRequestCanceled {  
					err = errRequestCanceledConn  
				}  
				return nil, err  
			default:  
				// return below  
			}  
		}  
		return w.pc, w.err  
	case <-req.Cancel:  
		return nil, errRequestCanceledConn  
	case <-req.Context().Done():  
		return nil, req.Context().Err()  
	case err := <-cancelc:  
		if err == errRequestCanceled {  
			err = errRequestCanceledConn  
		}  
		return nil, err  
	}  
}
```

当我们新建连接时，若没有在Client结构体所维护的连接池中找到可用的连接，那么就调用queueForDial方法，这里需要注意，当Transport没有设置MaxIdleConnsPerHost这个值的情况下，client针对每个不同的host所能维持的connection默认值为2。

接下来看看queueForDial方法：
```go
func (t *Transport) queueForDial(w *wantConn) {  
	w.beforeDial()  
	if t.MaxConnsPerHost <= 0 {  
		go t.dialConnFor(w)  
		return  
	}  
  
	t.connsPerHostMu.Lock()  
	defer t.connsPerHostMu.Unlock()  
  
	if n := t.connsPerHost[w.key]; n < t.MaxConnsPerHost {  
		if t.connsPerHost == nil {  
			t.connsPerHost = make(map[connectMethodKey]int)  
		}  
		t.connsPerHost[w.key] = n + 1  
		go t.dialConnFor(w)  
		return  
	}  
  
	if t.connsPerHostWait == nil {  
		t.connsPerHostWait = make(map[connectMethodKey]wantConnQueue)  
	}  
	q := t.connsPerHostWait[w.key]  
	q.cleanFront()  
	q.pushBack(w)  
	t.connsPerHostWait[w.key] = q  
}
```
跟进dialConnFor方法：
```go
func (t *Transport) dialConnFor(w *wantConn) {  
	defer w.afterDial()  
  
	pc, err := t.dialConn(w.ctx, w.cm)  
	delivered := w.tryDeliver(pc, err)  
	if err == nil && (!delivered || pc.alt != nil) {  
		// pconn was not passed to w,  
		// or it is HTTP/2 and can be shared.  
		// Add to the idle connection pool.  
		t.putOrCloseIdleConn(pc)  
	}  
	if err != nil {  
		t.decConnsPerHost(w.key)  
	}  
}
```

继续跟dialConn方法：
```go
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {  
	// 生成一个*persistConn对象  
	pconn = &persistConn{  
		t:             t,  
		cacheKey:      cm.key(),  
		reqch:         make(chan requestAndChan, 1),  
		writech:       make(chan writeRequest, 1),  
		closech:       make(chan struct{}),  
		writeErrCh:    make(chan error, 1),  
		writeLoopDone: make(chan struct{}),  
	}  
	trace := httptrace.ContextClientTrace(ctx)  
	wrapErr := func(err error) error {  
		if cm.proxyURL != nil {  
			// Return a typed error, per Issue 16997  
			return &net.OpError{Op: "proxyconnect", Net: "tcp", Err: err}  
		}  
		return err  
	}  
	//下面是连接代码  
	if cm.scheme() == "https" && t.DialTLS != nil {  
		var err error  
		pconn.conn, err = t.DialTLS("tcp", cm.addr())  
		if err != nil {  
			return nil, wrapErr(err)  
		}  
		if pconn.conn == nil {  
		return nil, wrapErr(errors.New("net/http: Transport.DialTLS returned (nil, nil)"))  
		}  
		if tc, ok := pconn.conn.(*tls.Conn); ok {  
			// Handshake here, in case DialTLS didn't. TLSNextProto below  
			// depends on it for knowing the connection state.  
			if trace != nil && trace.TLSHandshakeStart != nil {  
				trace.TLSHandshakeStart()  
			}  
			if err := tc.Handshake(); err != nil {  
				go pconn.conn.Close()  
				if trace != nil && trace.TLSHandshakeDone != nil {  
					trace.TLSHandshakeDone(tls.ConnectionState{}, err)  
				}  
				return nil, err  
			}  
			cs := tc.ConnectionState()  
			if trace != nil && trace.TLSHandshakeDone != nil {  
				trace.TLSHandshakeDone(cs, nil)  
			}  
			pconn.tlsState = &cs  
		}  
	} else {  
		conn, err := t.dial(ctx, "tcp", cm.addr())  
		if err != nil {  
			return nil, wrapErr(err)  
		}  
		pconn.conn = conn  
		if cm.scheme() == "https" {  
			var firstTLSHost string  
			if firstTLSHost, _, err = net.SplitHostPort(cm.addr()); err != nil {  
				return nil, wrapErr(err)  
			}  
			if err = pconn.addTLS(firstTLSHost, trace); err != nil {  
				return nil, wrapErr(err)  
			}  
		}  
	}  
	//省略一部分不重要的代码  
  //重点来了  
	pconn.br = bufio.NewReaderSize(pconn, t.readBufferSize())  
	pconn.bw = bufio.NewWriterSize(persistConnWriter{pconn}, t.writeBufferSize())  
	//起了俩goroutine，一个负责在conn上读，一个负责写  
	go pconn.readLoop()  
	go pconn.writeLoop()  
	return pconn, nil
```

兜兜绕绕终于来到了关键的地方，在connection成功建立之后，可以看见启动了两个goroutine来负责读写  
我们挑readloop来看看：
```go
func (pc *persistConn) readLoop() {  
	closeErr := errReadLoopExiting // default value, if not changed below  
	defer func() {  
		pc.close(closeErr)  
		pc.t.removeIdleConn(pc)  
	}()  
	tryPutIdleConn := func(trace *httptrace.ClientTrace) bool {  
		if err := pc.t.tryPutIdleConn(pc); err != nil {  
			closeErr = err  
			if trace != nil && trace.PutIdleConn != nil && err != errKeepAlivesDisabled {  
				trace.PutIdleConn(err)  
			}  
			return false  
		}  
		if trace != nil && trace.PutIdleConn != nil {  
			trace.PutIdleConn(nil)  
		}  
		return true  
	}  
  
	// eofc is used to block caller goroutines reading from Response.Body  
	// at EOF until this goroutines has (potentially) added the connection  
	// back to the idle pool.  
	eofc := make(chan struct{})  
	defer close(eofc) // unblock reader on errors  
  
	// Read this once, before loop starts. (to avoid races in tests)  
	testHookMu.Lock()  
	testHookReadLoopBeforeNextRead := testHookReadLoopBeforeNextRead  
	testHookMu.Unlock()  
  
	alive := true  
	for alive {  
	pc.readLimit = pc.maxHeaderResponseSize()  
		_, err := pc.br.Peek(1)  
  
		pc.mu.Lock()  
		if pc.numExpectedResponses == 0 {  
			pc.readLoopPeekFailLocked(err)  
			pc.mu.Unlock()  
			return  
		}  
		pc.mu.Unlock()  
  
		rc := <-pc.reqch  
		trace := httptrace.ContextClientTrace(rc.req.Context())  
  
		var resp *Response  
		if err == nil {  
			resp, err = pc.readResponse(rc, trace) //从connection中读response  
		} else {  
			err = transportReadFromServerError{err}  
			closeErr = err  
		}  
  
		if err != nil {  
			if pc.readLimit <= 0 {  
				err = fmt.Errorf("net/http: server response headers exceeded %d bytes; aborted", pc.maxHeaderResponseSize())  
			}  
  
			select {  
			case rc.ch <- responseAndError{err: err}:  
			case <-rc.callerGone:  
				return  
			}  
			return  
		}  
		pc.readLimit = maxInt64 // effictively no limit for response bodies  
  
		pc.mu.Lock()  
		pc.numExpectedResponses--  
		pc.mu.Unlock()  
  
		bodyWritable := resp.bodyIsWritable()  
		hasBody := rc.req.Method != "HEAD" && resp.ContentLength != 0  
  
		if resp.Close || rc.req.Close || resp.StatusCode <= 199 || bodyWritable {  
			// Don't do keep-alive on error if either party requested a close  
			// or we get an unexpected informational (1xx) response.  
			// StatusCode 100 is already handled above.  
			alive = false  
		}  
  
		if !hasBody || bodyWritable {  
			pc.t.setReqCanceler(rc.req, nil)  
  
			// Put the idle conn back into the pool before we send the response  
			// so if they process it quickly and make another request, they'll  
			// get this same conn. But we use the unbuffered channel 'rc'  
			// to guarantee that persistConn.roundTrip got out of its select  
			// potentially waiting for this persistConn to close.  
			// but after  
			alive = alive &&  
				!pc.sawEOF &&  
				pc.wroteRequest() &&  
				tryPutIdleConn(trace)  
  
			if bodyWritable {  
				closeErr = errCallerOwnsConn  
			}  
  
			select {  
			case rc.ch <- responseAndError{res: resp}: //利用chan将respone数据发送到上层  
			case <-rc.callerGone:  
				return  
			}  
  
			// Now that they've read from the unbuffered channel, they're safely  
			// out of the select that also waits on this goroutine to die, so  
			// we're allowed to exit now if needed (if alive is false)  
			testHookReadLoopBeforeNextRead()  
			continue  
		}  
  
		waitForBodyRead := make(chan bool, 2)  
		//这里是最关键的地方，close函数实际上就是调用了earlyCloseFn函数，后面详细讲  
		body := &bodyEOFSignal{  
			body: resp.Body,  
			earlyCloseFn: func() error {  
				waitForBodyRead <- false  
				<-eofc // will be closed by deferred call at the end of the function  
				return nil  
  
			},  
			fn: func(err error) error {  
				isEOF := err == io.EOF  
				waitForBodyRead <- isEOF  
				if isEOF {  
					<-eofc // see comment above eofc declaration  
				} else if err != nil {  
					if cerr := pc.canceled(); cerr != nil {  
						return cerr  
					}  
				}  
				return err  
			},  
		}  
  
		resp.Body = body //利用bodyEOFSignal封装body  
		if rc.addedGzip && strings.EqualFold(resp.Header.Get("Content-Encoding"), "gzip") {  
			resp.Body = &gzipReader{body: body}  
			resp.Header.Del("Content-Encoding")  
			resp.Header.Del("Content-Length")  
			resp.ContentLength = -1  
			resp.Uncompressed = true  
		}  
  
		select {  
		case rc.ch <- responseAndError{res: resp}:  
		case <-rc.callerGone:  
			return  
		}  
  
		// Before looping back to the top of this function and peeking on  
		// the bufio.Reader, wait for the caller goroutine to finish  
		// reading the response body. (or for cancellation or death)  
		select {  
		//内存泄露的元凶  
		case bodyEOF := <-waitForBodyRead:  
			pc.t.setReqCanceler(rc.req, nil) // before pc might return to idle pool  
			alive = alive &&   
				bodyEOF &&  
				!pc.sawEOF &&  
				pc.wroteRequest() &&  
				tryPutIdleConn(trace) 	//尝试将connection放回连接池  
			if bodyEOF {  
				eofc <- struct{}{}  
			}  
		case <-rc.req.Cancel:  
			alive = false  
			pc.t.CancelRequest(rc.req)  
		case <-rc.req.Context().Done():  
			alive = false  
			pc.t.cancelRequest(rc.req, rc.req.Context().Err())  
		case <-pc.closech:  
			alive = false  
		}  
  
		testHookReadLoopBeforeNextRead()  
	}  
}
```

我们可以看到，负责读response的goroutine会以alive为条件不断循环，这时想让此goroutine退出，需要将alive置为false，但是我们可以看到最后有一个select语句，在一切正常的情况下（下面的几个错误信号不来的情况下）应当是由waitForBodyRead这个chan来推进整个goroutine，但是从上方代码我们可以看到，想让此chan中被写入数据，只有两个回调函数earlyCloseFn和fn，那我们来看看这两个函数应当如何被触发：
```go
func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {  
	es.mu.Lock()  
	closed, rerr := es.closed, es.rerr  
	es.mu.Unlock()  
	if closed {  
		return 0, errReadOnClosedResBody  
	}  
	if rerr != nil {  
		return 0, rerr  
	}  
  
	n, err = es.body.Read(p)  
	if err != nil {  
		es.mu.Lock()  
		defer es.mu.Unlock()  
		if es.rerr == nil {  
			es.rerr = err  
		}  
		err = es.condfn(err)  
	}  
	return  
}  
  
func (es *bodyEOFSignal) Close() error {  
	es.mu.Lock()  
	defer es.mu.Unlock()  
	if es.closed {  
		return nil  
	}  
	es.closed = true  
	if es.earlyCloseFn != nil && es.rerr != io.EOF {  
		return es.earlyCloseFn()  
	}  
	err := es.body.Close()  
	return es.condfn(err)  
}  
  
// caller must hold es.mu.  
func (es *bodyEOFSignal) condfn(err error) error {  
	if es.fn == nil {  
		return err  
	}  
	err = es.fn(err)  
	es.fn = nil  
	return err  
}
```

终于看到了我们心心念念的Close方法，可以看到Close方法在
`es.earlyCloseFn != nil && es.rerr != io.EOF`
的情况下会触发earlyCloseFn方法（这里有很多情况，后面详细讨论）

而当这一条件不满足时，Close方法将会调用condfn方法，仔细看，其实这一方法在上面的Read方法中也会调用，而Read方法实际和io里Reader有关，也就是说Read方法实际在我们读resp.Body的时候会触发（相当于我们在读出Body中数据的过程中会触发这个方法）

那么也就是说，如果我们既不读Body中的数据也不调用Close方法，那么这个goroutine和writeloop的goroutine将无法退出（因为writeloop是当readerloop这个goroutine退出时才会退出，可看readloop开头的defer语句），这就将导致大量的goroutine阻塞在select语句无法退出，而且占用了此persistconn，导致连接池中一直无法有空闲的persistconn放回，这会导致HTTP1.1直接退化到1.0（一个http请求一个tcp connection），相关资源也无法被gc回收，最终导致内存泄露的问题。

不过从代码上来看，实际上不仅仅是一定要调用Close方法才可以防止内存泄露，实际上如果能将Body数据读完也是可以的，但建议最好还是调用Close方法，因为也许有许多异常状态会发生（其实如果正常读写，并在读完后调用Close方法，类似于我最开始那个正确的程序，那么Close方法的判断条件中es.rerr会一直等于io.EOF，所以不是说调用了Close方法就会关闭读写的goroutine(实际在既读又close的情况下，只要connection不断开，Close方法只是关闭了io.Reader而已，而且connection的异常实际上也不是由Close方法负责，而是由readloop方法中的resp.Close来进行判断，所以实际上Close方法并非是我们所认为的关闭connection，释放资源（除了io.Reader出现错误时，或者没有读body时才是扮演这一角色），绝大多数情况下，Close方法只是用来关闭读取Body的io.Reader,与connection异常并无关联，也不会实际上执行退出goroutine以供gs释放资源的行为）

所以在正常情况下（尤其是既Close又read的情况下），调用Close方法实际就是关闭了io.Reader后调用了condfn方法，而且正常情况下，condfn方法中的es.fn其实一直是nil，直到Body中的数据被读完，此时Read方法再次调用condfn方法时，es.fn才会被赋值为之前在readloop中定义的回调函数的地址(猜想这也是为了保证先等上层读完数据之后才继续接着取数据），从而将goroutine不阻塞在select，并把connection重新放回连接池备用（此时goroutine也不会退出，因为err是io.Eof,alive并不会变为false导致退出循环）），当Read操作做完之后，Close方法关闭Reader，调用condfn方法，但这时候es.fn又变为了nil（读body的操作已经做完，es.fn重新变为nil,其实可以它看作驱动器，只有当上层读完数据，需要读写goroutine进行下一次循环时，才会被赋值，执行回调函数，推进整个循环结构的进行，防止阻塞)，故而不会执行任何其余操作了。

简而言之，每一个connection都有一对goroutine负责读写，在读写完一次之后，readloop这个routine会负责将这个connection放回连接池，并且俩goroutine都不会退出，等到connection上有新的request时，这两个goroutine将会再次被激活，如此往复。

# KeepAlive机制

继续上文，那么什么时候Close函数会被调用并使得goroutine退出呢？那就是当我一开始的代码改为下面这样的时候：
```go
func doGet(client *http.Client, url string, id int) {  
	resp, err := client.Get(url)  
	if err != nil {  
		fmt.Println(err)  
		return  
	}  
	//io.Copy(ioutil.Discard, resp.Body)  
	//fmt.Println("copy")  
	// buf, err := ioutil.ReadAll(resp.Body)  
	// fmt.Printf("%d: %s -- %v\n", id, string(buf[0:1]), err)  
	if err := resp.Body.Close(); err == nil {  
		fmt.Println("close")  
	}  
}
```

此时我们不读Body中的数据，直接关闭，那么这个时候，Close函数中的es.rerr会变为nil，从而触发earlyClosefn函数，使得读写goroutine全部退出，connection关闭

在这个问题上，我还想讨论一个机制，那就是keepalive的机制，我们都知道keepalive是旨在让connection更多的承载http信息，而不用一个http创建一个tcp，从而节省资源。

在go中，我们可以通过Transport的Disablekeepalive选项来选择关闭或者启用，当我们关闭了keepalive机制的时候，那么每一个http请求都会新建一个tcp连接，而每个连接将都不会放入client的连接池中，这意味着所有连接都不可复用，两个负责读写的goroutine也是只处理一个request后立即退出

**这里有一点要注意，那就是如果你启用了keepalive选项，并不代表你的所有连接真的可以复用了，因为我在实际测试中发现，如果对方服务器的返回头当中Connection这一项置空，那么哪怕你自己启用了，实际上还是有可能无法复用的，这一点很奇怪，因为我看了不少资料表示http1.1应该是默认keepalive，除非显式指明Connection头为close才会导致不启用复用的情况，但是在实际测试中我发现这在一定程度上不是很准确，并不是所有的都默认启用，有些特例是不会的，例如python的SimpleHttpServer，就是默认不启用，返回的头中也没有keealive，所以具体还是看实现**

**做了个实验，证实了就算服务端包头没有connection alive也可能会keepalive，主要还是看服务器具体如何实现的（绝大多数都是默认支持keep的，不管返回包是否写明，因为http1.1默认实现），附上一张抓包图**

[![capture](https://ph4ntonn.github.io/image/Sourcecode/fake-not-keepalive.png)](https://ph4ntonn.github.io/image/Sourcecode/fake-not-keepalive.png)

**可以看见虽然服务器返回头当中没有keepalive（客户端request有keepalive），但是仍然保持了长连接并没有断开，说明还是默认实现了keepalive机制**

注：我这里举的例子是在两边均同意复用的情况下才称为keepalive机制启用，vice versa。

keepalive关闭时分四种情况：

首先每次都会新建连接，读写goroutine也都是新的

（1）如果我们既读Body数据，也close了body，那么最后导致读写goroutine退出的将是因为下方代码中resp.Close变为true（因为这个标志表示连接已经断开，不可读，并且由于这句判断语句是在goroutine读取了connection之后判断的，所以实际上在第一次的request读完之后就已经变为了false）->Read方法->condfn方法->goroutine解除阻塞，此时由于alive已经变为false，从而使得两个goroutine在下个循环之前退出。
```go
if resp.Close || rc.req.Close || resp.StatusCode <= 199 || bodyWritable {  
			// Don't do keep-alive on error if either party requested a close  
			// or we get an unexpected informational (1xx) response.  
			// StatusCode 100 is already handled above.  
			alive = false  
		}
```

（2）如果我们没有读数据，直接Close了Body，那么最后导致读写goroutine退出的将是由Close方法中es.rerr等于nil导致earlyCloseFn方法被触发，从而直接退出goroutine（er.rerr正常情况下应该是io.EOF，表示body在被正常读取）：
```go
if es.earlyCloseFn != nil && es.rerr != io.EOF {  
		return es.earlyCloseFn()  
	}
```

（3）如果我们只读数据，不Close，那么情况和第一种一致,但是如果不Close，那么读body的io.Reader将不会被关闭，没关闭的多了，也可能会内存泄露（因为Close方法代码里有err := es.body.Close()),详情可看下面，写在后面了

（4）如果我们不读也不Close，这里由于keepalive本身就被关闭，connection在一次resquest/response后也失效了，但是由于goroutine一直阻塞，无法退出，所以占用的资源一直无法释放,最后会导致内存泄露

keepalive开启时也分四种情况：

首先每次连接只要连接池中有足够的空闲connection，则不需要新建，可以直接复用，默认情况下每个host对应最多两个connection

（1）如果我们既读Body数据，也close了body，那么读写goroutine在keepalive到期之前将不会退出，一直会重复处理connection数据，将connection放回连接池，处理connection数据。。。这样的循环操作，直到keepalive到期，然后就和keepalive关闭时第一种情况一样了

（2）如果我们不读数据，直接close，那么keepalive将会失效（其实不是真的失效，只是因为不读数据的情况下，es.rerr为nil，导致goroutine被退出，无法放入连接池以供复用，所以看起来好像是失效了，实际只是我们强行关闭了）
```go
func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {  
	es.mu.Lock()  
	closed, rerr := es.closed, es.rerr  
	es.mu.Unlock()  
	if closed {  
		return 0, errReadOnClosedResBody  
	}  
	if rerr != nil {  
		return 0, rerr  
	}  
  
	n, err = es.body.Read(p)  
	if err != nil {  
		es.mu.Lock()  
		defer es.mu.Unlock()  
		if es.rerr == nil {  
              //因为没有调用read方法，所以es.rerr一直是nil，无法被赋新值，故而close函数中条件被满足了  
	           es.rerr = err  
		}  
		fmt.Println("wo zai read han shu zheer")  
		err = es.condfn(err)  
	}  
	return  
}
```
（3）如果我们只读数据，不close，和第一种一致,但是如果不Close，那么读body的io.Reader将不会被关闭，没关闭的多了，也可能会内存泄露（因为Close方法代码里有err := es.body.Close()）
```go
func (es *bodyEOFSignal) Close() error {  
	fmt.Println("wori...")  
	es.mu.Lock()  
	defer es.mu.Unlock()  
	if es.closed {  
		return nil  
	}  
	es.closed = true  
      //es.rerr应该就是记录reader的上一个状态，如果是EOF，那么说明是正常的读写完成，无需退出goroutine  
	if es.earlyCloseFn != nil && es.rerr != io.EOF {  
		fmt.Print("i am close")  
		return es.earlyCloseFn()  
	}  
	//正常读数据的话会走到这里而不是在上面的if就返回  
        //下面这行应该就是关闭了读Body的io.Reader  
	err := es.body.Close()  
	fmt.Println("err is:", err)  
	return es.condfn(err)  
}
```
（4）如果我们既不读也不close，那么keepalive将会失效，因为读写goroutine被阻塞，无法将connection放入连接池，导致后续数据传输无法复用connection，只能一个http请求一个tcp连接，最终导致内存泄露。

# 另一个小坑点

切记不要在每个请求里都新建一个client结构体。。。也就是说下面的代码是错误的：
```go
package main  
  
import (  
	"net/http"  
	"time"  
)  
  
func main() {  
	for {  
		test()  
	}  
}  
func test() {  
	transport := http.Transport{  
		DisableKeepAlives: true,  
	}  
  
	client := &http.Client{  
		Transport: &transport,  
		Timeout:   10 * time.Second,  
	}  
	request, _ := http.NewRequest("GET", target, nil)  
  
	resp, err := client.Do(request)  
  
}
```

这样会导致内存疯长，实测。。。

而且在源码中，作者也有注释：
```go
// A Client is an HTTP client. Its zero value (DefaultClient) is a  
// usable client that uses DefaultTransport.  
//  
// The Client's Transport typically has internal state (cached TCP  
// connections), so Clients should be reused instead of created as  
// needed. Clients are safe for concurrent use by multiple goroutines.  
//  
// A Client is higher-level than a RoundTripper (such as Transport)  
// and additionally handles HTTP details such as cookies and  
// redirects.
```

因为client结构底层维护了一个连接池，所以不需要每次都新建，正确代码如下：
```go
package main  
  
import (  
	"net/http"  
	"time"  
)  
  
var (  
	transport = http.Transport{  
		DisableKeepAlives: true,  
	}  
  
	client = &http.Client{  
		Transport: &transport,  
		Timeout:   10 * time.Second,  
	}  
)  
  
func main() {  
	for {  
		test()  
	}  
}  
func test() {  
  
	request, _ := http.NewRequest("GET", target, nil)  
  
	resp, err := client.Do(request)  
  
}
```
  

# 结语

其实看似简单的功能背后也有很多玄机，随便看看都是一个个大坑（笑）

# Reference
https://zhuanlan.zhihu.com/p/367694474
https://ph4ntonn.github.io/Go-source-code
https://www.luozhiyun.com/archives/561