# Go netpoller的价值

Go netpoller 的价值 通过前面对源码的分析，我们现在知道 Go netpoller 依托于 runtime scheduler，为开发者提供了一种强大的同步网络编程模式；然而，Go netpoller 存在的意义却远不止于此，Go netpoller I/O 多路复用搭配 Non-blocking I/O 而打造出来的这个原生网络模型，它最大的价值是把网络 I/O 的控制权牢牢掌握在 Go 自己的 runtime 里，关于这一点我们需要从 Go 的 runtime scheduler 说起，Go 的 G-P-M 调度模型如下：
![[Pasted image 20230426143329.png]]


G 在运行过程中如果被阻塞在某个 system call 操作上，那么不光 G 会阻塞，执行该 G 的 M 也会解绑 P(实质是被 sysmon 抢走了)，与 G 一起进入 sleep 状态。如果此时有 idle 的 M，则 P 与其绑定继续执行其他 G；如果没有 idle M，但仍然有其他 G 要去执行，那么就会创建一个新的 M。当阻塞在 system call 上的 G 完成 syscall 调用后，G 会去尝试获取一个可用的 P，如果没有可用的 P，那么 G 会被标记为 \_Grunnable 并把它放入全局的 runqueue 中等待调度，之前的那个 sleep 的 M 将再次进入 sleep。 

现在清楚为什么 netpoll 为什么一定要使用非阻塞 I/O 了吧？就是为了避免让操作网络 I/O 的 goroutine 陷入到系统调用从而进入内核态，因为一旦进入内核态，整个程序的控制权就会发生转移(到内核)，不再属于用户进程了，那么也就无法借助于 Go 强大的 runtime scheduler 来调度业务程序的并发了；而有了 netpoll 之后，借助于非阻塞 I/O ，G 就再也不会因为系统调用的读写而 (长时间) 陷入内核态，当 G 被阻塞在某个 network I/O 操作上时，实际上它不是因为陷入内核态被阻塞住了，而是被 Go runtime 调用 gopark 给 park 住了，此时 G 会被放置到某个 wait queue 中，而 M 会尝试运行下一个 \_Grunnable 的 G，如果此时没有 \_Grunnable 的 G 供 M 运行，那么 M 将解绑 P，并进入 sleep 状态。当 I/O available，在 epoll 的 eventpoll.rdr 中等待的 G 会被放到 eventpoll.rdllist 链表里并通过 netpoll 中的 epoll_wait 系统调用返回放置到全局调度队列或者 P 的本地调度队列，标记为 \_Grunnable ，等待 P 绑定 M 恢复执行。
  
# Go netpoller 的问题 
Go netpoller 的设计不可谓不精巧、性能也不可谓不高，配合 goroutine 开发网络应用的时候就一个字：爽。因此 Go 的网络编程模式是及其简洁高效的，然而，没有任何一种设计和架构是完美的， goroutine-per-connection 这种模式虽然简单高效，但是在某些极端的场景下也会暴露出问题：goroutine 虽然非常轻量，它的自定义栈内存初始值仅为 2KB，后面按需扩容；
海量连接的业务场景下， goroutine-per-connection ，此时 goroutine 数量以及消耗的资源就会呈线性趋势暴涨，虽然 Go scheduler 内部做了 g 的缓存链表，可以一定程度上缓解高频创建销毁 goroutine 的压力，但是对于瞬时性暴涨的长连接场景就无能为力了，大量的 goroutines 会被不断创建出来，从而对 Go runtime scheduler 造成极大的调度压力和侵占系统资源，然后资源被侵占又反过来影响 Go scheduler 的调度，进而导致性能下降。 

一般`Go` 的`TCP` 和`HTTP` 的程序都是每一个连接启动一个`goroutine` 去处理。在`Go` 的理念和设计中，`goroutine` 是**廉价**的，成千上万的`goroutine` 也是轻松的能够被创建，这点决不等同于进程或者线程。成千上万的`goroutine` 确实小case，但如果是上百万的连接呢，按照现在`Go原生网络库` 的`goroutine-per-connection` 模式意味着需要创建上百万的`goroutine` 去处理，实际上一个连接可能不止创建一个`gorituine` ，创建一个`goroutine` 要占用多少内存呢，答案是`2kb～8kb` （和操作系统以及`Go` 相关版本有关）不等，所以如果是数百万连接的话光内存占用可能会有十几个G之多，这也是一个不小的开销了。

### 调度压力问题

大量的`goroutine` 创建不仅仅会带来内存占用的问题，还会给`Go runtime`调度器的调度带来压力，从而导致调度效率的下降，最直接的表现就是吞吐率的下降、连接延时的增加。另外还记得在之前的`Go原生网络库`的文章中源码剖析中的`sync.once` `epoller` 嘛，也就是下面这一段：

serverInit.Do(runtime_pollServerInit)

这里`serverInit` 是一个`sync.Once` 类型，这么做可以保证只初始化一个`epoll` 实例。所以`Go原生网络库` 是一个单`event loop` ，也就是连接的处理（连接的建立和读写）和`IO`的处理都是被加入到一个`epoller` 实例的，在同一个`event loop` 里面处理，这样在海量连接的同时可能会导致性能瓶颈。

以上就是`Go` 原生`net` 库的问题，虽然平常很难遇到百万甚至千万级别的`goroutine`带来的调度压力和内存占用问题，即使遇到了最不济也是升级机器配置就可以解决，但真的遇到了这样的问题除了简单粗暴的升级机器配置之外还有什么别的办法呢？其实就是放弃`Go`原生`net`库的这种网络模式，采取更加高效的网络库模式，当然这也意味着放弃了`Go`煞费苦心封装后暴露给开发者稳定简单的网络API，需要开发者做更多额外的工作，付出更多的心智。下面就来看看有哪些高性能的网络库模式。

# Reactor 网络模型 
目前 Linux 平台上主流的高性能网络库/框架中，大都采用 Reactor 模式，比如 netty、libevent、libev、ACE，POE(Perl)、Twisted(Python)等。 Reactor 模式本质上指的是使用 I/O 多路复用(I/O multiplexing) + 非阻塞 I/O(non-blocking I/O) 的模式。 通常设置一个主线程负责做 event-loop 事件循环和 I/O 读写，通过 select/poll/epoll_wait 等系统调用监听 I/O 事件，业务逻辑提交给其他工作线程去做。而所谓『非阻塞 I/O』的核心思想是指避免阻塞在 read() 或者 write() 或者其他的 I/O 系统调用上，这样可以最大限度的复用 event-loop 线程，让一个线程能服务于多个 sockets。在 Reactor 模式中，I/O 线程只能阻塞在 I/O multiplexing 函数上（select/poll/epoll_wait）。 

Reactor 模式的基本工作流程如下： 
- Server 端完成在 bind&listen 之后，将 listenfd 注册到 epollfd 中，最后进入 event-loop 事件循环。循环过程中会调用 select/poll/epoll_wait 阻塞等待，若有在 listenfd 上的新连接事件则解除阻塞返回，并调用 socket.accept 接收新连接 connfd，并将 connfd 加入到 epollfd 的 I/O 复用（监听）队列。 
- 当 connfd 上发生可读/可写事件也会解除 select/poll/epoll_wait 的阻塞等待，然后进行 I/O 读写操作，这里读写 I/O 都是非阻塞 I/O，这样才不会阻塞 event-loop 的下一个循环。然而，这样容易割裂业务逻辑，不易理解和维护。 
- 调用 read 读取数据之后进行解码并放入队列中，等待工作线程处理。 工作线程处理完数据之后，返回到 event-loop 线程，由这个线程负责调用 write 把数据写回 client。 
accept 连接以及 conn 上的读写操作若是在主线程完成，则要求是非阻塞 I/O，因为 Reactor 模式一条最重要的原则就是：I/O 操作不能阻塞 event-loop 事件循环。实际上 event loop 可能也可以是多线程的，只是一个线程里只有一个 select/poll/epoll_wait。 
上面提到了 Go netpoller 在某些场景下可能因为创建太多的 goroutine 而过多地消耗系统资源，而在现实世界的网络业务中，服务器持有的海量连接中在极短的时间窗口内只有极少数是 active 而大多数则是 idle，就像这样（非真实数据，仅仅是为了比喻）： 
![[Pasted image 20230426143610.png]]
那么为每一个连接指派一个 goroutine 就显得太过奢侈了，而 Reactor 模式这种利用 I/O 多路复用进而只需要使用少量线程即可管理海量连接的设计就可以在这样网络业务中大显身手了： 
![[Pasted image 20230426143621.png]]

在绝大部分应用场景下，我推荐大家还是遵循 Go 的 best practices，使用原生的 Go 网络库来构建自己的网络应用。然而，在某些极度追求性能、压榨系统资源以及技术栈必须是原生 Go （不考虑 C/C++ 写中间层而 Go 写业务层）的业务场景下，我们可以考虑自己构建 Reactor 网络模型。

### Reactor 模式

`Reactor` 模式并非是`Go` 独有的，他应该被理解为一种网络处理的模式，是一种`Io 多路复用` + `非阻塞Io` 的处理模式。在`Go` 中理解起来就是将`event loop goroutine` 和`业务goroutine` 剥离开，让`event loop goroutine`专注于连接的建立和读写 ,读到的消息交由给`goroutine池 (workerpool)`去处理业务逻辑，最后再由`event loop gorotuine` 写回响应。

在`Reactor` 模式中`event loop goroutine` 只能阻塞在`select/poll/epoll` `wait` 上面，其他的`Io` 都要求是非阻塞的，否则阻塞了`event loop` 将会严重影响处理连接的性能。这样可以最大限度的复用`envent loop goroutine` ，让它更快更多的处理连接，包括`java Netty`、`python Twisted` 、`Go evio` 以及我们之后要讲解的`Go` 开源项目`gnet` 实际上都是`Reactor` 模式或者`Reactor` 模式的变种，是高性能的网络库模式。

![图片说明](https://uploadfiles.nowcoder.com/images/20201224/48172364_1608810173616/83E8BB493A169320A39E38B2554136CD "图片标题")

### Reactor with mutiple poller

其实这也是`Reactor` 模式，只是它和上面介绍的那种`Reator` 模式不同的地方在于它会有多个`poller` 在不同的`thread/goroutine` 里面来共同承担起`evnet loop` 的功能（连接的建立、读取、响应）,具体到实现方式可能有多种，但基本理念都是一样的，列举几种实现的方式：

-   多个`event loop goroutine` 对同一个`listener` 调用`accept` 方法来竞争获取连接然后交由各自`event thread/goroutine`的`poller` 处理，后面的处理方式就和上面经典的`Reactor` 模式一样了。
    
-   利用`linux` 的`reuseport` 用多个`goroutine` 同时监听一个端口，`linux 内核` 会负责`负载均衡` ，到了各个`event loop goroutine` 之后的处理方式就是一样了。
    
-   一个主`Reactor` 负责`accept connection` 之后通过一些自己实现的负载均衡算法(`随机`、`轮询`、 `最少连接数`)等负载均和算法来尽可能的将连接平均负载到各个`event loop goroutine` 。
    
    这些具体的实现方式虽然不一样，但本质和目的确是一致的，就是多个`event poller` 共同处理，提升性能（提高吞吐率，降低延时）一般来说这种模式会拥有更高的性能。
    
    ![图片说明](https://uploadfiles.nowcoder.com/images/20201224/48172364_1608810186367/364475932AD250249F49992772EA2739 "图片标题")
    

### prefork

`Prefork` 是`Apache`实现的一种服务方式。一个单一的控制进程启动的时候负责启动多个**子进程**，每个**子进程**都是独立的，使用单一的`goroutine`处理消息事件。**子进程**可以共享**父进程**打开的文件，这样我们就可以把`net.Listener`传给子进程，让所有的子进程共同监听这个端口。

## 总结

本文介绍了`Go` 原生网络库模式、`Go` 原生网络库一些可能面临的问题以及一些高性能网络库模式的原理，本文的目的或者观点并不是想阐述`Go` 原生网络库模式的实现不好或者说无法满足日常使用，相反，`Go` 原生网络库设计实现封装的相当优秀和完备，提供简单易用`API` 的同时性能也是杠杆滴，但编程世界里面没有银弹，任何事情都是取舍后的体现，`Go` 原生网络库也无法在所有场景中都是`MVP` 级别表现，它的这种模式在海量连接但是活跃连接少的情况下对资源的利用就不是那么的到位，而在这种情况下像`evio` `gnet` 这些库的`reactor` 模式实现的少量`goroutine` 代理大量连接在内存占用和延时上会有更好的表现。但也只是一些极少的情况下`Go` 原生网络库可能略显疲态，百分之九十五以上的场景`Go` 推崇的`goroutine-per-connection` 模式都是**best practice**，配合 强大的`Go runtime scheduler` 用起来就一个字：爽！之后的几篇文章我们将挑选一个优秀的`Go` 开源网络库`gnet` 来体会一下它的设计以及分析一下它的主要流程。

# 对于epoll的边沿触发和水平触发
事件
EPOLLIN ：fd可读  
EPOLLOUT：fd可写  
EPOLLPRI：fd有紧急事件数据到达  
EPOLLERR：fd发生错误  
EPOLLHUP：fd被挂断  
EPOLLET： 设置epoll为边沿触发，默认为水平触发  
EPOLLONESHOT：只监听一次事件

原生netpoller是边沿触发！！！！！

在上述提及的 Reactor 和 I/O Task 设计中，epoll 的触发方式会影响 I/O 和 buffer 的设计，大体来说分为两种方式：
1.  采用水平触发(LT)，则需要同步的在事件触发后主动完成 I/O，并向上层代码直接提供 buffer。
2.  采用边沿触发(ET)，可选择只管理事件通知(如 go net 设计)，由上层代码完成 I/O 并管理 buffer。

Level_triggered(水平触发)：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll_wait()时，它还会通知你在上没读写完的文件描述符上继续读写，当然如果你一直不去读写，它会一直通知你！！！如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率！！！

Edge_triggered(边缘触发)：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你！！！这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符！！！


两种方式各有优缺，netpoll 采用前者策略，水平触发时效性更好，容错率高，主动 I/O 可以集中内存使用和管理，提供 nocopy 操作并减少 GC。事实上一些热门开源网络库也是采用方式一的设计，如 easygo、evio、gnet 等。

但使用 LT 也带来另一个问题，即底层主动 I/O 和上层代码并发操作 buffer，引入额外的并发开销。比如：I/O 读数据写 buffer 和上层代码读 buffer 存在并发读写，反之亦然。为了保证数据正确性，同时不引入锁竞争，现有的开源网络库通常采取 同步处理 buffer(easygo, evio) 或者将 buffer 再 copy 一份提供给上层代码(gnet) 等方式，均不适合业务处理或存在 copy 开销。


另一方面，常见的 bytes、bufio、ringbuffer 等 buffer 库，均存在 growth 需要 copy 原数组数据，以及只能扩容无法缩容，占用大量内存等问题。因此字节跳动的netpoll希望引入一种新的 Buffer 形式，一举解决上述两方面的问题。


# gnet对于原生的netpoller的劣势
这个事情只能从字节跳动的netpoll的报告来表达：
在我们实践过程中，发现我们新写的 netpoll 虽然在 avg 延迟上表现胜于 Go 原生的 net 库，但是在 p99 和 max 延迟上要普遍略高于 Go 原生的 net 库，并且尖刺也会更加明显，如下图（Go 1.13，蓝色为 netpoll + 多路复用，绿色为 netpoll + 长连接，黄色为 net 库 + 长连接）：

![](https://static001.infoq.cn/resource/image/4d/a2/4d7ebc98254965053bfa5811c9e148a2.png)

我们尝试了很多种办法去优化，但是收效甚微。最终，我们定位出这个延迟并非是由于 netpoll 本身的开销导致的，而是由于 go 的调度导致的，比如说：
1.  由于在 netpoll 中，SubReactor 本身也是一个 goroutine，受调度影响，不能保证 EpollWait 回调之后马上执行，所以这一块会有延迟；
2.  同时，由于用来处理 I/O 事件的 SubReactor 和用来处理连接监听的 MainReactor 本身也是 goroutine，所以实际上很难保证在多核情况之下，这些 Reactor 能并行执行；甚至在最极端情况之下，可能这些 Reactor 会挂在同一个 P 下，最终变成了串行执行，无法充分利用多核优势；
3.  由于 EpollWait 回调之后，SubReactor 内是串行处理 I/O 事件的，导致排在最后的事件可能会有长尾问题；
4.  在连接多路复用场景下，由于每个连接绑定了一个 SubReactor，故延迟完全取决于这个 SubReactor 的调度，导致尖刺会更加明显。

由于 Go 在 runtime 中对于 net 库有做特殊优化，所以 net 库不会有以上情况；同时 net 库是 goroutine-per-connection 的模型，所以能确保请求能并行执行而不会相互影响。

对于以上这个问题，我们目前解决的思路有两个：
1.  修改 Go runtime 源码，在 Go runtime 中注册一个回调，每次调度时调用 EpollWait，把获取到的 fd 传递给回调执行；
    
2.  与字节跳动内核组合作，支持同时批量读/写多个连接，解决串行问题。另外，经过我们的测试，Go 1.14 能够使得延迟略有降低同时更加平稳，但是所能达到的极限 QPS 更低。希望我们的思路能够给业界同样遇到此问题的同学提供一些参考。

# gnet源码分析

接着上文的介绍，我们最后讨论了网络IO的几种实现模型，接下来我们有了理论基础，就可以分析一款实现reactor模型的网络框架，目前实现reactor的框架比较经典有netty、gnet。本文将重点分析gnet的网络实现。

## **1.gnet网络框架整体架构**

![](https://ask.qcloudimg.com/http-save/yehe-6570637/9d80e75fe41c72c90133e3fbc73755ba.png?imageView2/2/w/2560/h/7000)

## **2.gnet调用链分析**

![](https://ask.qcloudimg.com/http-save/yehe-6570637/53dda854c84f3f737e6d7dcd53519a9a.png?imageView2/2/w/2560/h/7000)

## **3.gnet核心模块梳理**

![](https://ask.qcloudimg.com/http-save/yehe-6570637/e0f6b5b5777b56704e392d9421ef2eec.png?imageView2/2/w/2560/h/7000)

### **3.1 gnet分析**

**gnet.go**

```javascript
// Serve starts handling events for the specified address.
//
// Address should use a scheme prefix and be formatted
// like `tcp://192.168.0.10:9851` or `unix://socket`.
// Valid network schemes:
//  tcp   - bind to both IPv4 and IPv6
//  tcp4  - IPv4
//  tcp6  - IPv6
//  udp   - bind to both IPv4 and IPv6
//  udp4  - IPv4
//  udp6  - IPv6
//  unix  - Unix Domain Socket
//
// The "tcp" network scheme is assumed when one is not specified.
func Serve(eventHandler EventHandler, protoAddr string, opts ...Option) (err error) {
   //加载配置
    options := loadOptions(opts...)

    if options.Logger != nil {
        logging.DefaultLogger = options.Logger
    }
    defer logging.Cleanup()

    // The maximum number of operating system threads that the Go program can use is initially set to 10000,
    // which should be the maximum amount of I/O event-loops locked to OS threads users can start up.
    if options.LockOSThread && options.NumEventLoop > 10000 {
        logging.DefaultLogger.Errorf("too many event-loops under LockOSThread mode, should be less than 10,000 "+
            "while you are trying to set up %d\n", options.NumEventLoop)
        return errors.ErrTooManyEventLoopThreads
    }

 //从tcp://127.0.0.1:12解析network和addr
    network, addr := parseProtoAddr(protoAddr)

    var ln *listener
    // 初始化listener
    // 1.fd=socket()
    // 2.bind(fd,port)
    // 3.listen(fd)
    if ln, err = initListener(network, addr, options.ReusePort); err != nil {
        return
    }
    defer ln.close()
   //真正监听
    return serve(eventHandler, ln, options, protoAddr)
}
```



**listener_unix.go**

```javascript

type listener struct {
    once          sync.Once
    fd            int
    lnaddr        net.Addr
    reusePort     bool
    addr, network string
}3
func initListener(network, addr string, reusePort bool) (l *listener, err error) {
    l = &listener{network: network, addr: addr, reusePort: reusePort}
    err = l.normalize()
    return
}

// 归一化，选择不同的协议，进行初始化socket
// socket
// noblocking
// bind
// reusePort
// listen

func (ln *listener) normalize() (err error) {
    switch ln.network {
    case "tcp", "tcp4", "tcp6":
        ln.fd, ln.lnaddr, err = reuseport.TCPSocket(ln.network, ln.addr, ln.reusePort)
        ln.network = "tcp"
    case "udp", "udp4", "udp6":
        ln.fd, ln.lnaddr, err = reuseport.UDPSocket(ln.network, ln.addr, ln.reusePort)
        ln.network = "udp"
    case "unix":
        _ = os.RemoveAll(ln.addr)
        ln.fd, ln.lnaddr, err = reuseport.UnixSocket(ln.network, ln.addr, ln.reusePort)
    default:
        err = errors.ErrUnsupportedProtocol
    }
    if err != nil {
        return
    }

    return
}
```



### **3.2 server分析**

**server_unix.go**

```javascript
func serve(eventHandler EventHandler, listener *listener, options *Options, protoAddr string) error {
    // Figure out the correct number of loops/goroutines to use.
    numEventLoop := 1
    if options.Multicore {
        numEventLoop = runtime.NumCPU()
    }
    if options.NumEventLoop > 0 {
        numEventLoop = options.NumEventLoop
    }

    svr := new(server)
    svr.opts = options
    svr.eventHandler = eventHandler
    svr.ln = listener

    // 选择负责均衡策略
    switch options.LB {
    case RoundRobin:
        svr.lb = new(roundRobinLoadBalancer)
    case LeastConnections:
        svr.lb = new(leastConnectionsLoadBalancer)
    case SourceAddrHash:
        svr.lb = new(sourceAddrHashLoadBalancer)
    }

    svr.cond = sync.NewCond(&sync.Mutex{})
    svr.ticktock = make(chan time.Duration, channelBuffer(1))
    svr.logger = logging.DefaultLogger
    svr.codec = func() ICodec {
        if options.Codec == nil {
            return new(BuiltInFrameCodec)
        }
        return options.Codec
    }()

    server := Server{
        svr:          svr,
        Multicore:    options.Multicore,
        Addr:         listener.lnaddr,
        NumEventLoop: numEventLoop,
        ReusePort:    options.ReusePort,
        TCPKeepAlive: options.TCPKeepAlive,
    }
    switch svr.eventHandler.OnInitComplete(server) {
    case None:
    case Shutdown:
        return nil
    }

    // 调用start，开启所有的reactor
    if err := svr.start(numEventLoop); err != nil {
        svr.closeEventLoops()
        svr.logger.Errorf("gnet server is stopping with error: %v", err)
        return err
    }
    defer svr.stop(server)

    serverFarm.Store(protoAddr, svr)

    return nil
}

func (svr *server) start(numEventLoop int) error {
    // udp或者端口重用的的话直接activateEventLoops
    if svr.opts.ReusePort || svr.ln.network == "udp" {
        // 这个里面每个actor都可以接收客户端的连接，具体在loop_bsd的handleEvent中有体现loopAccept
        return svr.activateEventLoops(numEventLoop)
    }

    return svr.activateReactors(numEventLoop)
}
```



**server_unix.go**

```javascript
func (svr *server) activateReactors(numEventLoop int) error {
    for i := 0; i < numEventLoop; i++ {
        if p, err := netpoll.OpenPoller(); err == nil {
            el := new(eventloop)
            // 这儿listener是同一个，没啥关系，因为其他的actor不会监听客户端连接
            el.ln = svr.ln
            el.svr = svr
            el.poller = p
            el.packet = make([]byte, 0x10000)
            el.connections = make(map[int]*conn)
            el.eventHandler = svr.eventHandler
            el.calibrateCallback = svr.lb.calibrate
            svr.lb.register(el)
        } else {
            return err
        }
    }

    // 开始所有的subReactor
    // Start sub reactors in background.
    svr.startSubReactors()

    // epoll_create()
    if p, err := netpoll.OpenPoller(); err == nil {
        el := new(eventloop)
        el.ln = svr.ln
        el.idx = -1
        el.poller = p
        el.svr = svr
        // 注册读事件，接收客户端连接
        _ = el.poller.AddRead(el.ln.fd)
        svr.mainLoop = el

        // 开始mainReactor
        // Start main reactor in background.
        svr.wg.Add(1)
        go func() {
            svr.activateMainReactor(svr.opts.LockOSThread)
            svr.wg.Done()
        }()
    } else {
        return err
    }

    return nil
}

func (svr *server) startSubReactors() {
    svr.lb.iterate(func(i int, el *eventloop) bool {
        svr.wg.Add(1)
        go func() {
            svr.activateSubReactor(el, svr.opts.LockOSThread)
            svr.wg.Done()
        }()
        return true
    })
}
```



subReactors,封装在reactor_bsd.go**

```javascript
func (svr *server) activateMainReactor(lockOSThread bool) {
    if lockOSThread {
        runtime.LockOSThread()
        defer runtime.UnlockOSThread()
    }

    defer svr.signalShutdown()
    // 调用epoll_wait阻塞，等待客户端连接
    err := svr.mainLoop.poller.Polling(func(fd int, filter int16) error { return svr.acceptNewConnection(fd) })
    svr.logger.Infof("Main reactor is exiting due to error: %v", err)
}


func (svr *server) activateSubReactor(el *eventloop, lockOSThread bool) {
    if lockOSThread {
        runtime.LockOSThread()
        defer runtime.UnlockOSThread()
    }

    defer func() {
        el.closeAllConns()
        if el.idx == 0 && svr.opts.Ticker {
            close(svr.ticktock)
        }
        svr.signalShutdown()
    }()

    if el.idx == 0 && svr.opts.Ticker {
        go el.loopTicker()
    }

    // 这个内部会调用epoll_wait方法，阻塞在这个地方
    err := el.poller.Polling(func(fd int, filter int16) error {
        // 取得当前的client 连接
        if c, ack := el.connections[fd]; ack {
            if filter == netpoll.EVFilterSock {
                return el.loopCloseConn(c, nil)
            }


            switch c.outboundBuffer.IsEmpty() {
            // Don't change the ordering of processing EVFILT_WRITE | EVFILT_READ | EV_ERROR/EV_EOF unless you're 100%
            // sure what you're doing!
            // Re-ordering can easily introduce bugs and bad side-effects, as I found out painfully in the past.

            // 如果写的buffer不为空，则说明有数据可写
            //  采用kqueue的状态来判断
            case false:
                if filter == netpoll.EVFilterWrite {
                    return el.loopWrite(c)
                }
                return nil
            //  如果写的buffer为空，没有写的数据，则处理读的事件
            case true:
                if filter == netpoll.EVFilterRead {
                    return el.loopRead(c)
                }
                return nil
            }
        }
        return nil
    })
    svr.logger.Infof("Event-loop(%d) is exiting normally on the signal error: %v", el.idx, err)
}

// udp和端口复用的话会走到这个
func (svr *server) activateEventLoops(numEventLoop int) (err error) {
    // Create loops locally and bind the listeners.
    for i := 0; i < numEventLoop; i++ {
        l := svr.ln
        // udp
        if i > 0 && svr.opts.ReusePort {
            // 多个listener，监听同一个地址和端口
            if l, err = initListener(svr.ln.network, svr.ln.addr, svr.ln.reusePort); err != nil {
                return
            }
        }

        var p *netpoll.Poller
        if p, err = netpoll.OpenPoller(); err == nil {
            el := new(eventloop)
            el.ln = l
            el.svr = svr
            el.poller = p
            el.packet = make([]byte, 0x10000)
            el.connections = make(map[int]*conn)
            el.eventHandler = svr.eventHandler
            el.calibrateCallback = svr.lb.calibrate
            _ = el.poller.AddRead(el.ln.fd)
            svr.lb.register(el)
        } else {
            return
        }
    }

    // Start event-loops in background.
    svr.startEventLoops()

    return
}

func (svr *server) startEventLoops() {
    // 遍历所有的subReactor，然后监听读写事件
    svr.lb.iterate(func(i int, el *eventloop) bool {
        svr.wg.Add(1)
        go func() {
            // 开始监听事件
            el.loopRun(svr.opts.LockOSThread)
            svr.wg.Done()
        }()
        return true
    })
}

// 开始event的阻塞
func (el *eventloop) loopRun(lockOSThread bool) {
    if lockOSThread {
        runtime.LockOSThread()
        defer runtime.UnlockOSThread()
    }

    defer func() {
        el.closeAllConns()
        el.ln.close()
        if el.idx == 0 && el.svr.opts.Ticker {
            close(el.svr.ticktock)
        }
        el.svr.signalShutdown()
    }()

    if el.idx == 0 && el.svr.opts.Ticker {
        go el.loopTicker()
    }

    // 阻塞在这个地方，handleEvent里面最后还调用了loopAccept
    err := el.poller.Polling(el.handleEvent)
    el.svr.logger.Infof("Event-loop(%d) is exiting due to error: %v", el.idx, err)
}
```



**激活mainReactor和

**下面的handleEvent方法是区分linux和unix分别实现的，因为linux是epoll，而mac是kqueue，读写事件的封装不同**

**loop_bsd.go**

```javascript
// 不仅处理客户端的读写，还处理接收客户端的请求
func (el *eventloop) handleEvent(fd int, filter int16) error {
    if c, ok := el.connections[fd]; ok {
        if filter == netpoll.EVFilterSock {
            return el.loopCloseConn(c, nil)
        }

        switch c.outboundBuffer.IsEmpty() {
        // Don't change the ordering of processing EVFILT_WRITE | EVFILT_READ | EV_ERROR/EV_EOF unless you're 100%
        // sure what you're doing!
        // Re-ordering can easily introduce bugs and bad side-effects, as I found out painfully in the past.
        case false:
            if filter == netpoll.EVFilterWrite {
                return el.loopWrite(c)
            }
            return nil
        case true:
            if filter == netpoll.EVFilterRead {
                return el.loopRead(c)
            }
            return nil
        }
    }
    // 如果连接不存在，则表示一个新的连接，因此接受的客户端的连接
    return el.loopAccept(fd)
}
```



**acceptor_unix.go**

```javascript
// 接收新的客户端连接
func (svr *server) acceptNewConnection(fd int) error {
    nfd, sa, err := unix.Accept(fd)
    if err != nil {
        if err == unix.EAGAIN {
            return nil
        }
        return errors.ErrAcceptSocket
    }
    // 设置非阻塞
    if err = os.NewSyscallError("fcntl nonblock", unix.SetNonblock(nfd, true)); err != nil {
        return err
    }

    netAddr := netpoll.SockaddrToTCPOrUnixAddr(sa)
    // 负载均衡，找到合适的subReactor
    el := svr.lb.next(netAddr)
    // 封装新的链接
    c := newTCPConn(nfd, el, sa, netAddr)

    // 最后将conn交给subReactor管理
    _ = el.poller.Trigger(func() (err error) {
        // 注册读事件
        if err = el.poller.AddRead(nfd); err != nil {
            return
        }
        // 添加到链接管理中
        el.connections[nfd] = c
        err = el.loopOpen(c)
        return
    })
    return nil
}
```



**connection_unix.go**

```javascript
// 封装新的连接
func newTCPConn(fd int, el *eventloop, sa unix.Sockaddr, remoteAddr net.Addr) (c *conn) {
    c = &conn{
        fd:             fd,
        sa:             sa,
        loop:           el,
        codec:          el.svr.codec,
        inboundBuffer:  prb.Get(),
        outboundBuffer: prb.Get(),
    }
    c.localAddr = el.ln.lnaddr
    c.remoteAddr = remoteAddr
    if el.svr.opts.TCPKeepAlive > 0 {
        // 设置长连接
        if proto := el.ln.network; proto == "tcp" || proto == "unix" {
            _ = netpoll.SetKeepAlive(fd, int(el.svr.opts.TCPKeepAlive/time.Second))
        }
    }
    return
}
```



### **3.3 poller分析**

**epoll_events.go**

```javascript
type eventList struct {
    size   int
    events []unix.EpollEvent
}

// 事件列表
func newEventList(size int) *eventList {
    return &eventList{size, make([]unix.EpollEvent, size)}
}

// 扩容
func (el *eventList) increase() {
    el.size <<= 1
    el.events = make([]unix.EpollEvent, el.size)
}

```



**epoll.go**

```javascript
// Poller represents a poller which is in charge of monitoring file-descriptors.
type Poller struct {
    fd            int    // epoll fd
    wfd           int    // wake fd
    wfdBuf        []byte // wfd buffer to read packet
    asyncJobQueue internal.AsyncJobQueue
}

// OpenPoller instantiates a poller.
func OpenPoller() (poller *Poller, err error) {
    poller = new(Poller)
    // epoll_create
    if poller.fd, err = unix.EpollCreate1(unix.EPOLL_CLOEXEC); err != nil {
        poller = nil
        err = os.NewSyscallError("epoll_create1", err)
        return
    }
    if poller.wfd, err = unix.Eventfd(0, unix.EFD_NONBLOCK|unix.EFD_CLOEXEC); err != nil {
        _ = poller.Close()
        poller = nil
        err = os.NewSyscallError("eventfd", err)
        return
    }
    poller.wfdBuf = make([]byte, 8)
    //
    if err = poller.AddRead(poller.wfd); err != nil {
        _ = poller.Close()
        poller = nil
        return
    }
    poller.asyncJobQueue = internal.NewAsyncJobQueue()
    return
}

// Close closes the poller.
func (p *Poller) Close() error {
    if err := os.NewSyscallError("close", unix.Close(p.fd)); err != nil {
        return err
    }
    return os.NewSyscallError("close", unix.Close(p.wfd))
}

// Make the endianness of bytes compatible with more linux OSs under different processor-architectures,
// according to http://man7.org/linux/man-pages/man2/eventfd.2.html.
var (
    u uint64 = 1
    b        = (*(*[8]byte)(unsafe.Pointer(&u)))[:]
)

// 唤醒阻塞在网络事件上的poller
// Trigger wakes up the poller blocked in waiting for network-events and runs jobs in asyncJobQueue.
func (p *Poller) Trigger(job internal.Job) (err error) {
    if p.asyncJobQueue.Push(job) == 1 {
        _, err = unix.Write(p.wfd, b)
    }
    return os.NewSyscallError("write", err)
}

// 阻塞网络事件
// Polling blocks the current goroutine, waiting for network-events.
func (p *Poller) Polling(callback func(fd int, ev uint32) error) error {
    el := newEventList(InitEvents)
    var wakenUp bool

    for {
        n, err := unix.EpollWait(p.fd, el.events, -1)
        if err != nil && err != unix.EINTR {
            logging.DefaultLogger.Warnf("Error occurs in epoll: %v", os.NewSyscallError("epoll_wait", err))
            continue
        }

        for i := 0; i < n; i++ {
            if fd := int(el.events[i].Fd); fd != p.wfd {
                // 调用回调函数
                switch err = callback(fd, el.events[i].Events); err {
                case nil:
                case errors.ErrAcceptSocket, errors.ErrServerShutdown:
                    return err
                default:
                    logging.DefaultLogger.Warnf("Error occurs in event-loop: %v", err)
                }
            } else {
                wakenUp = true
                _, _ = unix.Read(p.wfd, p.wfdBuf)
            }
        }

        if wakenUp {
            wakenUp = false
            switch err = p.asyncJobQueue.ForEach(); err {
            case nil:
            case errors.ErrServerShutdown:
                return err
            default:
                logging.DefaultLogger.Warnf("Error occurs in user-defined function, %v", err)
            }
        }

        if n == el.size {
            el.increase()
        }
    }
}

const (
    readEvents      = unix.EPOLLPRI | unix.EPOLLIN
    writeEvents     = unix.EPOLLOUT
    readWriteEvents = readEvents | writeEvents
)

// AddReadWrite registers the given file-descriptor with readable and writable events to the poller.
func (p *Poller) AddReadWrite(fd int) error {
    return os.NewSyscallError("epoll_ctl add",
        unix.EpollCtl(p.fd, unix.EPOLL_CTL_ADD, fd, &unix.EpollEvent{Fd: int32(fd), Events: readWriteEvents}))
}

// AddRead registers the given file-descriptor with readable event to the poller.
func (p *Poller) AddRead(fd int) error {
    return os.NewSyscallError("epoll_ctl add",
        unix.EpollCtl(p.fd, unix.EPOLL_CTL_ADD, fd, &unix.EpollEvent{Fd: int32(fd), Events: readEvents}))
}

// AddWrite registers the given file-descriptor with writable event to the poller.
func (p *Poller) AddWrite(fd int) error {
    return os.NewSyscallError("epoll_ctl add",
        unix.EpollCtl(p.fd, unix.EPOLL_CTL_ADD, fd, &unix.EpollEvent{Fd: int32(fd), Events: writeEvents}))
}

// ModRead renews the given file-descriptor with readable event in the poller.
func (p *Poller) ModRead(fd int) error {
    return os.NewSyscallError("epoll_ctl mod",
        unix.EpollCtl(p.fd, unix.EPOLL_CTL_MOD, fd, &unix.EpollEvent{Fd: int32(fd), Events: readEvents}))
}

// ModReadWrite renews the given file-descriptor with readable and writable events in the poller.
func (p *Poller) ModReadWrite(fd int) error {
    return os.NewSyscallError("epoll_ctl mod",
        unix.EpollCtl(p.fd, unix.EPOLL_CTL_MOD, fd, &unix.EpollEvent{Fd: int32(fd), Events: readWriteEvents}))
}

// Delete removes the given file-descriptor from the poller.
func (p *Poller) Delete(fd int) error {
    return os.NewSyscallError("epoll_ctl del", unix.EpollCtl(p.fd, unix.EPOLL_CTL_DEL, fd, nil))
}
```



### **3.4 loadBalancer分析**

**load_balancer.go**

```javascript
// loadBalancer is a interface which manipulates the event-loop set.
    loadBalancer interface {
        register(*eventloop)
        next(net.Addr) *eventloop
        iterate(func(int, *eventloop) bool)
        len() int
        calibrate(*eventloop, int32)
    }

    // roundRobinLoadBalancer with Round-Robin algorithm.
    // 轮询算法
    roundRobinLoadBalancer struct {
        nextLoopIndex int
        eventLoops    []*eventloop
        size          int
    }

    // leastConnectionsLoadBalancer with Least-Connections algorithm.
    // 最小连接算法
    leastConnectionsLoadBalancer struct {
        sync.RWMutex
        minHeap                 minEventLoopHeap
        cachedRoot              *eventloop
        threshold               int32
        calibrateConnsThreshold int32
    }

    // sourceAddrHashLoadBalancer with Hash algorithm.
    // 源地址hash
    sourceAddrHashLoadBalancer struct {
        eventLoops []*eventloop
        size       int
    }
```



## **4.gnet源码分析**

![](https://ask.qcloudimg.com/http-save/yehe-6570637/a280d79591ecf81f54ce40c8affe41de.png?imageView2/2/w/2560/h/7000)

# Reference
https://cloud.tencent.com/developer/article/1885432
https://www.infoq.cn/article/fea7chf9moohbxbtyres
