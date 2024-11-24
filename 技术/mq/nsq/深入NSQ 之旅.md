#nsq  #note 

# 介绍

[NSQ](http://www.oschina.net/p/nsq)是一个实时的分布式消息平台。它的设计目标是为在多台计算机上运行的松散服务提供一个现代化的基础设施骨架。这篇文章介绍了 基于go语言的NSQ的内部架构，它能够为高吞吐量的网络服务器带来 性能的优化，稳定性和鲁棒性。可以说， 如果不是因为我们在[bitly](https://bitly.com/)使用go语言，NSQ就不会存在。这里既会讲NSQ的功能也会涉及语言提供的特征。当然，语言会影响思维，这次也不例外。现在回想起来，选择使用go语言已经收到了十倍的回报。由语言带来的兴奋和社区的积极反馈为这个项目提供了极大的帮助。

# 概要

NSQ是由3个进程组成的：

-   [nsqd](http://bitly.github.io/nsq/components/nsqd.html)是一个接收、排队、然后转发消息到客户端的进程。负责接收消息,存储消息,分发消息给客户端，nsqd可以单独部署,也可以多节点部署,主要监听了两个端口,一个用来服务客户端(4150),一个用来提供api服务(4151).当然也可以配置监听https(4152)端口
-   [nsqlookupd](http://bitly.github.io/nsq/components/nsqlookupd.html) 管理拓扑信息并提供最终一致性的发现服务。 主要负责服务发现,nsqd的心跳、状态监测，给客户端、nsqadmin提供nsqd地址与状态,主要监听端口(4160)服务客户端,端口(4161)提供api服务.
-   [nsqadmin](http://bitly.github.io/nsq/components/nsqadmin.html)用于实时查看集群的统计数据（并且执行各种各样的管理任务）。是nsq的web后台管理,比如节点管理,topic管理,实时消息状态等,主要监听端口(4171)提供web服务

NSQ中的数据流模型是由_streams_和_consumers_组成的tree。topic是一种独特的stream。channel是一个订阅了给定topic的consumers 逻辑分组。

![](http://static.oschina.net/uploads/img/201401/03081429_evAT.gif)    

单个nsqd可以有多个topic，每个topic可以有多个channel。channel接收这个topic所有消息的副本，从而实现多播分发，而channel上的每个消息被分发给它的订阅者，从而实现负载均衡。这些基本成员组成了一个可以表示各种[简单和复杂拓扑结构](http://bitly.github.io/nsq/deployment/topology_patterns.html)的强大框架。有关NSQ的设计的更多信息请参见[设计文档](http://bitly.github.io/nsq/overview/design.html)。

-   NSQ推荐通过 **_nsqd_** 实例使用协同定位 **_producer_**，这意味着即使面对网络分区，消息也会被保存在本地，直到它们被一个 **_consumer_**读取。更重要的是， **_producer_**不必去发现其他的 **_nsqd_**节点，他们总是可以向本地 **_nsqd_**实例发布消息。  
    
-   一个 **_producer_**向它的本地 **_nsqd_**发送消息，要做到这点，首先要先打开一个连接( NSQ 提供 **_HTTP API_** 和 **_TCP 客户端_** 等2种方式连接到 **_nsqd_**)，然后发送一个包含 **_topic_**和消息主体的发布命令(pub/mpub/publish)，在这种情况下，我们将消息发布到 **_topic_**上，消息会采用多播的方式被拷贝到各个 **_channel_**中, 然后通过多个 **_channel_**以分散到我们不同需求的 **_consumer_**中。
   **_channel_**起到队列的作用。 多个 **_producer_**产生的 **_topic_**消息在每一个连接 **_topic_**的 **_channel_**上进行排队。  
    
-   每个 **_channel_**的消息都会进行排队，直到一个 **_consumer_**把他们消费，如果此队列超出了内存限制，消息将会被写入到磁盘中。 **_nsqd_**节点首先会向 **_nsqlookup_** 广播他们的位置信息，一旦它们注册成功， **_consumer_**将会从 **_nsqlookup_** 服务器节点上发现所有包含事件 **_topic_**的 **_nsqd_**节点。  
      
-   每个 **_consumer_**向每个 **_nsqd_**主机进行订阅操作，用于表明 **_consumer_**已经准备好接受消息了。这里我们不需要一个完整的连通图，但我们必须要保证每个单独的 **_nsqd_**实例拥有足够的消费者去消费它们的消息，否则 **_channel_**会被队列堆着。  


## 一些需要注意的问题
> Producer与nsqd是1:1关系,nsqd与topic是1:N关系,nsqlookupd与nqsd关系可以是M:N  
> Consumer可以指定单个nsqd,或者多个nsqd地址,也可以通过单个或者多个nsqlookupd去匹配对应的topic+channel的nsqd去消费消息  
> Consumer第一次通过nsqlookupd,会重试三次, 然后就是根据LookupdPollInterval(60s)时间轮询或者防抖动算法LookupdPollJitter(0.3默认)去进行重试查询  
> Producer发布消息没有topic会新建,如果先Consumer使用nsqlookupd去寻找消费,也就是没有创建topic之前会无法连接,重试3.3步骤  
> Consumer退出,channel不会自动删除, 多个nsqd服务都有相同的topic的时候,需要修改默认的config.MaxInflight才能连接,代表一次性可以接受多少条消息.  
> channel和topic的命名都有限制,正则匹配如下`^[\.a-zA-Z0-9_-]+(#ephemeral)?$`  
> 多个Consumer消费channel数据是随机的,无序. 消息至少投递一次,可能会重复投递  
> nsqd之间不会扩散消息. 但是topic的消息会分发给下面所有channel,但一个channel如果有多个消费者，消息会随机发送给其中一个消费者  
> nsq延时消息最长是一小时(60min)

有赞自研版Nsq
>  有赞的自研版 NSQ 在高可用性以及负载均衡方面进行了改造，自研版的 nsqd 中引入了数据分区以及副本，副本保存在不同的 nsqd 上，达到容灾目的。此外，自研版 NSQ 在原有 Protocol Spec 基础上进行了拓展，支持基于分区的消息生产、消费，以及基于消息分区的有序消费，以及消息追踪功能。

# NSQ架构和关键概念
## 架构
![](https://pic3.zhimg.com/80/v2-b266260a17702af4509f6c7d2ba40d4a_720w.jpg)

## topic
-   **topic** 是 **NSQ** 消息发布的 **逻辑关键词** ，可以理解为人为定义的一种消息类型。当程序初次发布带 **topic** 的消息时,如果 **topic** 不存在,则会在 **_nsqd_**中创建。

## producer
-   **producer** 通过 **HTTP API** 将消息发布到 **nsqd** 的指定 **topic** ，一般有 **pub/mpub** 两种方式， **pub** 发布一个消息， **mpub** 一个往返发布多个消息。

-   **producer** 也可以通过 **nsqd客户端** 的 **TCP接口** 将消息发布给 **nsqd** 的指定 **topic** 。

-   当生产者 **producer** 初次发布带 **topic** 的消息给 **nsqd** 时,如果 **topic** 不存在，则会在 **nsqd** 中创建 **topic** 。

## channel
-   当生产者每次发布消息的时候,消息会采用多播的方式被拷贝到各个 **_channel_** 中, **_channel_** 起到队列的作用。
-   **_channel_** 与 **_consumer(消费者)_** 相关，是消费者之间的负载均衡,消费者通过这个特殊的channel读取消息。
-   在 **_consumer_** 想单独获取某个 **topic** 的消息时，可以 **_subscribe(订阅)_**一个自己单独命名的 **_nsqd_**中还不存在的 **_channel_**, **_nsqd_**会为这个 **_consumer_**创建其命名的 **_channel_**
-   **_Channel_** 会将消息进行排列，如果没有 **_consumer_**读取消息，消息首先会在内存中排队，当量太大时就会被保存到磁盘中。可以在配置中配置具体参数。
-   一个 **_channel_** 一般会有多个 **_consumer_** 连接。假设所有已连接的 **_consumer_** 处于准备接收消息的状态，每个消息将被传递到一个随机的 **_consumer_**。
-   Go语言中的channel是表达队列的一种自然方式，因此一个NSQ的topic/channel，其核心就是一个存放消息指针的Go-channel缓冲区。缓冲区的大小由 --mem-queue-size 配置参数确定。

## consumer消息的消费者

-   **_consumer_** 通过 **TCP****_subscribe_** 自己需要的 **_channel_**  
    **_topic_** 和 **_channel_** 都没有预先配置。 **_topic_** 由第一次发布消息到命名 **_topic_** 的 **_producer_** 创建 **_或_** 第一次通过 **_subscribe_** 订阅一个命名 **_topic_** 的 **_consumer_** 来创建。 **_channel_** 被 **_consumer_** 第一次 **_subscribe_** 订阅到指定的 **_channel_** 创建。
-   多个 **_consumer_**  **_subscribe_**一个 **_channel_**，假设所有已连接的客户端处于准备接收消息的状态，每个消息将被传递到一个 **随机** 的 **_consumer_**。
-   NSQ 支持延时消息， **_consumer_** 在配置的延时时间后才能接受相关消息。
-   Channel在 **_consumer_** 退出后并不会删除，这点需要特别注意。

# Topics 和 Channels

Topics 和 channels，是NSQ的核心成员，它们是如何使用go语言的特点来设计系统的最好示例。Go的channels（为防止歧义，以下简称为“go-chan”）是表达队列的一种自然方式，因此一个NSQ的topic/channel，其核心就是一个存放消息指针的go-chan缓冲区。缓冲区的大小由  --mem-queue-size 配置参数确定。

读取数据后，向topic发布消息的行为包括：
-   实例化消息结构 (并分配消息体的字节数组)
-   read-lock 并获得 Topic
-   read-lock 并检查是否可以发布
-   发送到go-chan缓冲区

为了从一个topic和它的channels获得消息，topic不能按典型的方式用go-chan来接收，因为多个goroutines在一个go-chan上接收将会_**分发**_消息，而期望的结果是把每个消息_**复制**_到所有channel(goroutine)中。此外，每个topic维护3个主要goroutine。第一个叫做 router，负责从传入的go-chan中读取新发布的消息，并存储到一个队列里（内存或硬盘）。

第二个，称为 messagePump, 它负责复制和推送消息到如上所述的channel中。

第三个负责 DiskQueue IO，将在后面讨论。

Channels稍微有点复杂，它的根本目的是向外暴露一个单输入单输出的go-chan（事实上从抽象的角度来说，消息可能存在内存里或硬盘上）；

![](http://static.oschina.net/uploads/img/201401/03081430_kzH6.png)   

另外，每一个channel维护2个时间优先级队列，用于延时和消息超时的处理（并有2个伴随goroutine来监视它们）。并行化的改善是通过管理每个channel的数据结构来实现，而不是依靠go运行时的全局定时器。

注意：在内部，go运行时使用一个优先级队列和goroutine来管理定时器。它为整个time包（但不局限于）提供了支持。它通常不需要用户来管理时间优先级队列，但一定要记住，它是一个有锁的数据结构，有可能会影响 GOMAXPROCS>1 的性能。请参阅[runtime/time.goc](http://golang.org/src/pkg/runtime/time.goc?s=1684:1787#L83)。

## Backend / DiskQueue

NSQ的一个设计目标是绑定内存中的消息数目。它是通过DiskQueue(它拥有前面提到的的topic或channel的第三个goroutine)透明的把消息写入到磁盘上来实现的。

由于内存队列只是一个go-chan，没必要先把消息放到内存里，如果可能的话，退回到磁盘上：
```go
for msg := range c.incomingMsgChan {
    select {
    case c.memoryMsgChan <- msg:
    default:
        err := WriteMessageToBackend(&msgBuf, msg, c.backend)
        if err != nil {
            // ... handle errors ...
        }
    }
}
```

利用go语言的select语句，只需要几行代码就可以实现这个功能：上面的default分支只有在memoryMsgChan 满的情况下才会执行。

NSQ也有临时channel的概念。临时channel会丢弃溢出的消息（而不是写入到磁盘），当没有客户订阅后它就会消失。这是一个Go接口的完美用例。Topics和channels有一个的结构成员被声明为Backend接口，而不是一个具体的类型。一般的 topics和channels使用DiskQueue，而临时channel则使用了实现Backend接口的DummyBackendQueue。

# 减少垃圾回收的压力

在任何带有垃圾回收的环境里，你都会多多少少感受到吞吐量（工作有效性）、延迟（响应能力）、驻留集大小（内存使用量）的压力。就 Go 1.2 而言，垃圾回收有标记-清除（并发的）、不再生、不紧凑、阻止一切运行、大体精准的特点。大体精准是因为剩下的工作没有及时的完成（这是 Go 1.3 的计划）。Go 的垃圾回收机制当然会持续改进，但普遍的真理是：创建的垃圾越少，回收垃圾的时间越少。

首先，理解垃圾回收是如何在实际的工作负载中运行的是非常重要的。为此，**nsqd** 以 [statsd](https://github.com/etsy/statsd/) 的格式 (与其它内部指标一起) 发布垃圾回收的统计信息。**nsqadmin** 显示这些指标的图表，可以让你深入了解它在频率和持续时间两方面产生的影响：

![](http://static.oschina.net/uploads/img/201401/03081430_QIHU.png)

为了减少垃圾，你需要知道它们是在哪生成的。再次回到Go的工具链，它提供的答案如下：

-   使用[testing](http://golang.org/pkg/testing/)包和go test -benchmen来基准测试热点代码路径。它配置了每个迭代分配的数字（基准的运行可与[benchcmp](http://golang.org/misc/benchcmp)进行比较）。
-   使用 go build -gcflags -m 创建，将会输出[逃逸分析](http://en.wikipedia.org/wiki/Escape_analysis)的结果。

除此之外，它还提供了**nsqd** 的如下优化：

-   避免把[]byte 转化为字符串类型.
-   重复使用缓存或者对象（有时也许是[sync.Pool](https://groups.google.com/forum/#%21topic/golang-dev/kJ_R6vYVYHU)又称为[issue4720](https://code.google.com/p/go/issues/detail?id=4720)）.
-   预分配切片（特别是make的能力）并总是知晓链中各个条目的数量和大小。
-   提供各种配置面板（如消息大小）的限制。
-   避免封装（如使用interface{}）或者不必要的包装类（例如 用一struct给一个多值的go-chan）.
-   在热代码路径（它指定的）中避免使用defer。

## TCP 协议

[NSQ的TCP协议](http://bitly.github.io/nsq/clients/tcp_protocol_spec.html)是一个闪亮的会话典范，在这个会话中垃圾回收优化的理论发挥了极大的效用。

协议的结构是一个有很长的前缀框架，这使得协议更直接，易于编码和解码。
![[Pasted image 20221101174044.png]]

因为框架的组成部分的确切类型和大小是提前知道的，所以我们可以规避了使用方便的编码二进制包的Read()和Write()封装（及它们外部接口的查找和会话）反之我们使用直接调用 [binary.BigEndian](http://golang.org/pkg/encoding/binary/#ByteOrder)方法。

为了消除socket 输入输出的系统调用，客户端net.Conn被封装了[bufio.Reader](http://golang.org/pkg/bufio/#Reader)和[bufio.Writer](http://golang.org/pkg/bufio/#Writer)。这个Reader通过暴露[ReadSlice()](http://golang.org/pkg/bufio/#Reader.ReadSlice)，复用了它自己的缓冲区。这样几乎消除了读完socket时的分配，这极大的降低了垃圾回收的压力。这可能是因为与数据相关的大多数命令并没有逃逸（在边缘情况下这是假的，数据被强制复制）。

在更低层，MessageID 被定义为 [16]byte，这样可以将其作为 map 的 key（slice 无法用作 map 的 key)。然而，考虑到从 socket 读取的数据被保存为 []byte，胜于通过分配字符串类型的 key 来产生垃圾，并且为了避免从 slice 到 MessageID 的支撑数组产生复制操作，unsafe 包被用来将 slice 直接转换为 MessageID：
```go
id := *(*nsq.MessageID)(unsafe.Pointer(&msgID))
```

**注意:** 这是个技巧。如果编译器对此已经做了优化，或者 [Issue 3512](https://code.google.com/p/go/issues/detail?id=3512) 被打开可能会解决这个问题，那就不需要它了。[issue 5376](https://code.google.com/p/go/issues/detail?id=5376) 也值得通读，它讲述了在无须分配和拷贝时，和 string 类型可被接收的地方，可以交换使用的“类常量”的 byte 类型。

类似的，Go 标准库仅仅在 string 上提供了数值转换方法。为了避免 string 的分配，**nsqd** 使用了 [惯用的十进制转换方法](https://github.com/bitly/nsq/blob/master/util/byte_base10.go#L7-L27)，用于对 []byte 直接操作。

这些看起来像是微优化，但 TCP 协议包含了一些最热的代码执行路径。总体来说，以每秒数万消息的速度来说，它们对分配和系统开销的数量有着显著的影响：
![[Pasted image 20221101174533.png]]

# HTTP

NSQ的HTTP API是基于 Go's [net/http](http://golang.org/pkg/net/http/) 包实现的. _就是_ 常见的HTTP应用,在大多数高级编程语言中都能直接使用而无需额外的三方包。 简洁就是它最有力的武器，Go的 HTTP tool-chest最强大的就是其调试功能.  [net/http/pprof](http://golang.org/pkg/net/http/pprof) 包直接集成了HTTP server，可以方便的访问CPU, heap,    goroutine, and OS 进程文档 .gotool就能直接实现上述操作:
```bash
$ go tool pprof http:``//127``.0.0.1:4151``/debug/pprof/profile
```

这对于调试和_实时_监控进程非常有用！

此外，/stats端端返回JSON或是美观的文本格式信息，这让管理员使用命令行实时监控非常容易:
```bash
watch -n 0.5 'curl -s http://127.0.0.1:4151/stats | grep -v connected'
```

 打印出的结果如下:

![](http://static.oschina.net/uploads/img/201401/03081431_orZv.png)

此外, Go 1.2 还有很多监控指标[measurable HTTP performance gains](https://github.com/davecheney/autobench/blob/master/linux-amd64-x220-go1.1.2-vs-go1.2.txt#L156-L181). 每次更新Go版本后都能看到性能方面的改进，真是让人振奋！

# 依赖关系

源于其它生态系统，使用GO（理论匮乏）语言的依赖管理还得花点时间去适应

NSQ 就并不是单一的整个 repo库, 通过 _relative imports_ 而无需区别内部的包资源, 最终产生结构化的依赖管理。

主流的观点有以下两个:

-   **Vendoring**:拷贝应用需要的正确版本号到本地仓库并修改import 路径到本地库地址
    
-   **Virtual Env**: 列出构建是需要的版本信息，创建包含相关信息的GOPATH环境变量 
    

**Note:** 这仅仅应用于二级制包，对于可导入的包版本不起作用

NSQ使用 [godep](https://github.com/kr/godep)提供 (2) 中的实现.

它的实现原理是复制依赖关系到 [Godeps](https://github.com/bitly/nsq/blob/master/Godeps)文件中, 之后生成GOPATH环境变量。构建时，它使用Go环境中的工具链 来完成工作。 Godeps就是json格式，可以手动修改。

它还支持go的get. 例如，构建一个 NSQ版本:
```bash
godep get github.com/bitly/nsq/...
```


# NSQD
## nsqd基本结构
![[Pasted image 20221101165418.png]]

> 利用svc框架来启动服务, Run 时, 先后调用svc框架的 Init 和 Start 方法 ，然后开始不断监听退出的信号量, 最后调用 svc框架的Stop 方法来退出。  
>   
> svc框架的Start方法从本地文件读取数据初始化topic和channel，然后调用功能入口Main方法。Main方法利用waitGroup框架来启动4个服务线程，至此启动完毕。  
>   
> WaitGroup来自sync包，用于线程同步，单从字面意思理解，wait等待的意思，group组、团队的意思，WaitGroup就是等待一组服务执行完成后才会继续向下执行，涉及到WG个数的操作都使用原子操作来保证线程安全。

## nsqd详细流程图
![[Pasted image 20221101170513.png]]
## nsqd源码概述
> **_nsqd_** 服务开启时启动 **_TCP_** 服务供客户端连接，启动 **_HTTP_** 服务，提供 **_HTTP API_**

```go
//nsqd/nsqd.go:238
    tcpServer := &tcpServer{ctx: ctx}
    n.waitGroup.Wrap(func() {
        protocol.TCPServer(n.tcpListener, tcpServer, n.logf)
    })
    httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
    n.waitGroup.Wrap(func() {
        http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf)
}   )
```
> **_TCP_** 接收到客户端的请求后，创建protocol实例并调用nsqd/tcp.go中IOLoop()方法

```go
//nsqd/tcp.go:31
    var prot protocol.Protocol
    switch protocolMagic {
    case "  V2":
        prot = &protocolV2{ctx: p.ctx}
    default:
        protocol.SendFramedResponse(clientConn, frameTypeError, []byte("E_BAD_PROTOCOL"))
        clientConn.Close()
        p.ctx.nsqd.logf(LOG_ERROR, "client(%s) bad protocol magic '%s'",
            clientConn.RemoteAddr(), protocolMagic)
        return
    }
    err = prot.IOLoop(clientConn)
```
> protocol的IOLoop接收客户端的请求，根据命令的不同做相应处理。同时nsqd/protocol_v2.go中IOLoop会起一个goroutine运行messagePump()，该函数从该client订阅的channel中读取消息并发送给client( **_consumer_** )

```go
//nsqd/protocol_v2.go:41
    func (p *protocolV2) IOLoop(conn net.Conn) error {
        ...

        clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
        client := newClientV2(clientID, conn, p.ctx)

        // synchronize the startup of messagePump in order
        // to guarantee that it gets a chance to initialize
        // goroutine local state derived from client attributes
        // and avoid a potential race with IDENTIFY (where a client
        // could have changed or disabled said attributes)
        messagePumpStartedChan := make(chan bool)
        go p.messagePump(client, messagePumpStartedChan)
        <-messagePumpStartedChan

        ...
    }
```
```go
//nsqd/protocol_v2.go:200
    func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
        ...
        var memoryMsgChan chan *Message
        var backendMsgChan chan []byte
        var subChannel *Channel
        ...

        select {
                ...
            case b := <-backendMsgChan:
                ...
                msg, err := decodeMessage(b)
                ...
                subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
                client.SendingMessage()
                err = p.SendMessage(client, msg)
                ...
            case msg := <-memoryMsgChan:
                ...
                subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
                client.SendingMessage()
                err = p.SendMessage(client, msg)
                ...
            }
        ...
```

> 然后我们看下memoryMsgChan，backendMsgChan是如何产生的。我们知道producer通过TCP或HTTP来发布消息。我们重点看下TCP时的处理过程。首先protocol的IOLoop会根据producer的不同请求做相应处理，Exec方法判断请求的参数，调用不同的方法。

```go
//nsqd/protocol_v2.go:165
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
    ...
    switch {
    case bytes.Equal(params[0], []byte("FIN")):
        return p.FIN(client, params)
    case bytes.Equal(params[0], []byte("RDY")):
        return p.RDY(client, params)
    case bytes.Equal(params[0], []byte("REQ")):
        return p.REQ(client, params)
    case bytes.Equal(params[0], []byte("PUB")):
        return p.PUB(client, params)
    case bytes.Equal(params[0], []byte("MPUB")):
        return p.MPUB(client, params)
    case bytes.Equal(params[0], []byte("DPUB")):
        return p.DPUB(client, params)
    case bytes.Equal(params[0], []byte("NOP")):
        return p.NOP(client, params)
    case bytes.Equal(params[0], []byte("TOUCH")):
        return p.TOUCH(client, params)
    case bytes.Equal(params[0], []byte("SUB")):
        return p.SUB(client, params)
    case bytes.Equal(params[0], []byte("CLS")):
        return p.CLS(client, params)
    case bytes.Equal(params[0], []byte("AUTH")):
        return p.AUTH(client, params)
    }
    return nil, protocol.NewFatalClientErr(nil, "E_INVALID", fmt.Sprintf("invalid command %s", params[0]))
}
```
> 我们重点看下“PUB”时的运行过程。调用了p.pub(client, params)。从TCP中读到messageBody，然后处理后调用topic.PutMessage(msg)发送给topic。  
>   
> topic.PutMessage（）首先对topic加一个锁，通过t.put(m)方法将消息m发送memoryMsgChan中，然后释放锁。如果memoryMsgChan满了，申请一个buff，把消息写到Backend，后期被backendMsgChan接收

```go
//nsqd/protocol_v2.go:757
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
    ...
    topic := p.ctx.nsqd.GetTopic(topicName)
    msg := NewMessage(topic.GenerateID(), messageBody)
    err = topic.PutMessage(msg)
    ...
}

//nsqd/topic.go:197
func (t *Topic) put(m *Message) error {
    select {
    case t.memoryMsgChan <- m:
    default:
        b := bufferPoolGet()
        err := writeMessageToBackend(b, m, t.backend)
        bufferPoolPut(b)
        t.ctx.nsqd.SetHealth(err)
        if err != nil {
            t.ctx.nsqd.logf(LOG_ERROR,
                "TOPIC(%s) ERROR: failed to write message to backend - %s",
                t.name, err)
            return err
        }
    }
    return nil
}
```

> 了解了消息的产生，现在看下消息的传递。在nsqd/topic.go中有一个NewTopic()。其中又调用了messagePump()，注意这个和上面IOLoop的messagePump()不一样。

```go
//nsqd/topic.go:44
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
    t := &Topic{
        name:              topicName,
        channelMap:        make(map[string]*Channel),
        memoryMsgChan:     make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
        exitChan:          make(chan int),
        channelUpdateChan: make(chan int),
        ctx:               ctx,
        pauseChan:         make(chan bool),
        deleteCallback:    deleteCallback,
        idFactory:         NewGUIDFactory(ctx.nsqd.getOpts().ID),
    }

    ...

    t.waitGroup.Wrap(func() { t.messagePump() })

    t.ctx.nsqd.Notify(t)

    return t
}
```

> 看下t.messagePump()。topic的messagePump函数会不断从memoryMsgChan/backend队列中读消息，并将消息每个复制一遍，发送给topic下的所有channel。

```go
//nsqd/topic.go:220
func (t *Topic) messagePump() {
    ...
    for {
        select {
        case msg = <-memoryMsgChan:
        case buf = <-backendChan:
            msg, err = decodeMessage(buf)
            ...
        case <-t.channelUpdateChan:
            ...
        case pause := <-t.pauseChan:
            ...
        case <-t.exitChan:
            goto exit
        }

        for i, channel := range chans {
            chanMsg := msg
            // copy the message because each channel
            // needs a unique instance but...
            // fastpath to avoid copy if its the first channel
            // (the topic already created the first copy)
            if i > 0 {
                chanMsg = NewMessage(msg.ID, msg.Body)
                chanMsg.Timestamp = msg.Timestamp
                chanMsg.deferred = msg.deferred
            }
            if chanMsg.deferred != 0 {
                channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
                continue
            }
            err := channel.PutMessage(chanMsg)
            ...
            }
        }
    }
    ...
}
```

> channel的PutMessage方法和topic类似的，也是调用了put,首先写入memoryMsgChan，满了写入backend。  
>   
> 最后由一开始介绍的protocol实例的messagePump方法从memoryMsgChan或backendMsgChan读取消息并通过p.SendMessage(client, msg)发送到客户端 ，消息写入client.Writer。

```go
//nsqd/channel.go:220
func (c *Channel) put(m *Message) error {
    select {
    case c.memoryMsgChan <- m:
    default:
        b := bufferPoolGet()
        err := writeMessageToBackend(b, m, c.backend)
        bufferPoolPut(b)
        c.ctx.nsqd.SetHealth(err)
        if err != nil {
            c.ctx.nsqd.logf(LOG_ERROR, "CHANNEL(%s): failed to write message to backend - %s",
                c.name, err)
            return err
        }
    }
    return nil
}

//nsqd/protocol_v2.go:258
        select {
               ...
        case b := <-backendMsgChan:
            ...
            subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
            client.SendingMessage()
            err = p.SendMessage(client, msg)
            ...
        case msg := <-memoryMsgChan:
            ...
            subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
            client.SendingMessage()
            err = p.SendMessage(client, msg)
            ...
        }
```
# 测试

Go语言提供了内置的测试和基线。由于其简单的并发操作建模，在测试环境里加入**nsqd** 实例轻而易举。但是，在测试初始化的时候会有个问题：全局状态。最明显的就是引用运行态**nsqd** 实例的全局变量 i.e.var nsqd *NSQd.

于是某些测试就无可避免的使用局部变量去保存该值i.e.nsqd := NewNSQd(...).这也就意味着全局状态并未指向运行态的值，使测试失去了意义。

应对这个问题，Context结构体被引入以保存配置项metadata和实时**nsqd**的父类。所有全局状态的子引用都通过访问该Context来安全的获取相应值（主题，渠道，协议处理等等），这样测试起来也更有保障。

# 可靠性

一个系统，如果在面对变幻的网络环境和不可预知的事件时不具备可靠性，将不会是一个表现良好的分布式生产环境。NSQ的设计和实现方式，使它能容忍错误并以一种始终如一的，可预期的和稳定的方式来运行。它的首要的设计哲学是快速失败，认为错误都是致命的，并提供一种方式来调试遇到的任何问题。不过，为了能有所行动，你必须要能够检测异常环境...

## 心跳检测和超时

NSQ的TCP协议是需要推送的.在经过建立连接，三次握手，客户在aRDYstate的订阅数被置为0.当准备接受消息时，通过更新RDYstate来控制将要接受的消息数目。NSQ 客户端libraries将在后台持续管理这一环节，最终形成相应的消息流。 周期性的， **nsqd** 会发送心跳检测连接状态.客户端可以设置这个间隔时间但**nsqd**需要在发送下调指令前收到上条请求的回复。 应用层面的心跳检测和RDYstate组合能够避免 [head-of-line blocking](http://en.wikipedia.org/wiki/Head-of-line_blocking),它会是心跳检测失效 (i.e.如果用户等待处理消息前OS的缓存已满，则心跳检测失效).

为了确保进程的正常工作，所有的网络IO都会依据心跳检测的间隔时间来设置边界.这意味着你甚至可以断开客户端和 **nsqd** 的网络连接，而不必担心问题被发现并恰当的处理。一旦发现致命错误，客户连接将被强关。发送中的消息超时并从新加入新的客户端接受队列。最后，错误日志会被保存并增加内部评价矩阵内容。


## 管理Goroutines

启用goroutines很简单，但后续工作却不是那么容易弄好的。避免出现死锁是一个挑战。通常都是因为在排序上出了问题，goroutine可能在接到上游的消息前就收到了go-chan的退出信号。为啥提到这个？简单，一个未正确处理的goroutine就是内存泄露。更深入的分析，**nsqd** 进程含有多个激活的goroutines。从内部情况来看，消息的所有权是不停在变得。为了能正确的关掉goroutines，实时统计所有的进程信息是非常重要的。虽没有什么神奇的方法，但下面的几点能让工作简单一点...

### WaitGroups

[sync](http://golang.org/pkg/sync/) 包提供了 [sync.WaitGroup](http://golang.org/pkg/sync/#WaitGroup), 它可以计算出激活态的goroutines数（比提供退出的平均等待时间）

为了使代码简洁**nsqd** 使用如下wrapper：
```go
type WaitGroupWrapper struct { 
    sync.WaitGroup 
} 
  
func (w *WaitGroupWrapper) Wrap(cb func()) { 
    w.Add(1) 
    go func() { 
        cb() 
        w.Done() 
    }() 
} 
 // can be used as follows: wg := WaitGroupWrapper{} 
wg.Wrap(func() { n.idPump() }) // ... wg.Wait()
```

### 退出信号

在含有多个子goroutines中触发事件最简单的办法就是用一个go-chan，并在完成后关闭。所有当中暂停的动作将被激活，这就无需再向每个goroutine发送相关的信号了

```go
type WaitGroupWrapper struct { 
    sync.WaitGroup 
} 
  
func (w *WaitGroupWrapper) Wrap(cb func()) { 
    w.Add(1) 
    go func() { 
        cb() 
        w.Done() 
    }() 
} 
 // can be used as follows: wg := WaitGroupWrapper{} 
wg.Wrap(func() { n.idPump() }) // ... wg.Wait()
```

### 同步退出

想可靠的，无死锁，所有路径都保有信息的实现是很难的。下面是一些提示：

-   理想情况下，在go-chan发送消息的goroutine也应为关闭消息负责.
-   如果消息需要保留，确保相关go-chans被清空（尤其是无缓冲的！），以保证发送者可以继续进程.
-   另外，如果消息不再是相关的，在单个go-chan上的进程应该转换到包含推出信号的select上 （如上所述）以保证发送者可以继续进程.

一般的顺序应该是：
-   停止接受新的连接（停止监听）
-   向goroutines发出退出信号（见上文）
-   等待WaitGroup的goroutine中退出（见上文）
-   恢复缓冲数据
-   剩下的部分保存到磁盘

### 日志

最后，最重要的工作是记录你的Go例程的入口和出口日志！这使得它更容易识别死锁或泄漏的情况。**nsqd**日志行包括信息Go例程与他们的兄弟姐妹（和父母），如客户端的远程地址或主题/渠道名。日志是冗长的，但还不至于到接受不了的程度。这个是有两面性的，但**nsqd**倾斜当故障发生时向日志中放入更多的信息，，而不是为了避免繁琐而降低日志定位问题的有效性。

# FAQ

## 1、什么是nsqd的推荐拓扑？

-   **强烈建议在生成消息的任何服务旁边运行nsqd**。
-   nsqd是一个相对轻量级的进程，具有有限的内存占用，这使得它非常适合“与他人玩得很好”。
-   好处：有助于将消息流结构化为消费问题而不是生产问题。
-   好处：它基本上为给定主机上的该主题形成了一个独立的，分片的数据仓库。

## 2、为什么生产者不能使用nsqlookupd来查找要发布给的客户端？

-   NSQ推出了一种**消费者端发现模型**，可以减轻必须告诉消费者在哪里找到所需主题的前期配置负担。
-   但是，它没有提供任何方法来解决服务应该发布到的问题。 这是鸡和蛋的问题，在第一次发布之前主题不存在。
-   通过共同定位**nsqd**，完全回避了这个问题（服务只是发布到本地nsqd）并允许NSQ的运行时发现系统自然地工作。

## nsqd适合在单机中作为工作队列使用？

nsqd 也可以用在单机做**工作队列**使用，不过定位是分布式环境。

## 应该启动多少个nsqlookupd ？

-   只有少数几个，具体取决于您的群集大小，nsqd节点和使用者的数量，以及您所需的容错能力。
-   3或5对于涉及多达数百个主机和数千个消费者的部署非常有效。

#### 1、是否需要客户端库来发布消息？ 

不需要。直接通过http来发布订阅消息

#### 2、为什么强制客户端处理对TCP协议的PUB和MPUB命令的响应？ 

NSQ的默认操作模式应优先考虑安全性，我们希望协议简单且一致。

#### 3、什么时候会发布或订阅失败？ 

-   topic名称格式不正确（对字符/长度限制）。【[请看](https://nsq.io/clients/tcp_protocol_spec.html#notes)】
-   消息太大（此限制作为nsqd的参数公开）。
-   该topic正在被删除。
-   nsqd正处于干净利落的状态。
-   发布期间与客户端连接相关的任何故障（断开连接…）。 （1,2点是编程出错。3和4比较罕见。5是TCP连接中的问题）

#### 如何避免发布消息时，该主题正被删除[](http://liangjf.top/2019/05/26/72.nsq%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BA/#%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E5%8F%91%E5%B8%83%E6%B6%88%E6%81%AF%E6%97%B6%E8%AF%A5%E4%B8%BB%E9%A2%98%E6%AD%A3%E8%A2%AB%E5%88%A0%E9%99%A4)

删除topic是一种相对不常见的操作。 如果您需要删除topic，在删除后经过足够长的时间后再发布该topic创建的时间（这时不会造成歧义，会爆发布失败）。

### 设计与理论 

#### 1、如何命名主题（topic）和频道（channel） 

-   topic名称应描述流中的数据。如：encodes, decodes, api_requests, page_views
-   channel名称应描述其消费者所执行的工作。如analytics_increment,spam_analysis

#### 2、单个nsqd可以支持的主题和通道数量是否有任何限制？ 

没有内置限制。 它仅受nsqd运行的主机的内存和CPU的限制

#### 3、如何向群集发布新主题？ 

一个主题的第一个PUB或SUB将在nsqd上创建主题。 然后，主题元数据将传播到配置的nsqlookupd。 其他消费者将通过定期查询nsqlookupd来发现此主题。

#### 4、把nsq作为rpc？ 

可以，但不推荐


# Reference
http://www.oschina.net/translate/day-22-a-journey-into-nsq
https://www.cnblogs.com/shanyou/p/4296181.html

  https://zhuanlan.zhihu.com/p/37081073