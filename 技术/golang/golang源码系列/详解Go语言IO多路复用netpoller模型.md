---
sr-due: 2023-10-12
sr-interval: 11
sr-ease: 210
---

#golang  #note 

本文使用的go的源码15.7

可以从 Go 源码目录结构和对应代码文件了解 Go 在不同平台下的网络 I/O 模式的实现。比如，在 Linux 系统下基于 epoll，freeBSD 系统下基于 kqueue，以及 Windows 系统下基于 iocp。

因为我们的代码都是部署在Linux上的，所以本文以epoll封装实现为例子来讲解Go语言中I/O多路复用的源码实现。

# 介绍

## I/O多路复用

所谓 I/O 多路复用指的就是 select/epoll 这一系列的多路选择器：支持单一线程同时监听多个文件描述符（I/O 事件），阻塞等待，并在其中某个文件描述符可读写时收到通知。以防很多同学对select或epoll不那么熟悉，所以下面先来讲讲这两个选择器。

首先我们先说一下什么是文件描述符（File descriptor），根据它的英文首字母也简称FD，它是一个用于表述指向文件的引用的抽象化概念。它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。

### select
```go
int select(int nfds, 
		   fd_set *restrict readfds, 
		   fd_set *restrict writefds, 
		   fd_set *restrict errorfds, 
		   struct timeval *restrict timeout);
```

writefds、readfds、和errorfds是三个文件描述符集合。select会遍历每个集合的前nfds个描述符，分别找到可以读取、可以写入、发生错误的描述符，统称为就绪的描述符。

timeout参数表示调用select时的阻塞时长。如果所有文件描述符都未就绪，就阻塞调用进程，直到某个描述符就绪，或者阻塞超过设置的 timeout 后，返回。如果timeout参数设为 NULL，会无限阻塞直到某个描述符就绪；如果timeout参数设为 0，会立即返回，不阻塞。

当select函数返回后，可以通过遍历fdset，来找到就绪的描述符。

![multiplexing model](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210208204544304-720218418.png)

select的缺点也列举一下：

1.  select最大的缺陷就是单个进程所打开的FD是有一定限制的，它由FD_SETSIZE设置，默认值是1024;
2.  每次调用 select，都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大;
3.  每次 kernel 都需要线性扫描整个 fd_set，所以随着监控的描述符 fd 数量增长，其 I/O 性能会线性下降;

### epoll

epoll是select的增强版本，避免了“性能开销大”和“文件描述符数量少”两个缺点。

为方便理解后续的内容，先看一下epoll的用法：
```go
int listenfd = socket(AF_INET, SOCK_STREAM, 0); 
bind(listenfd, ...) 
listen(listenfd, ...) 

int epfd = epoll_create(...); 
epoll_ctl(epfd, ...); //将所有需要监听的fd添加到epfd中 
while(1){ 
	int n = epoll_wait(...) 
	for(接收到数据的socket){ 
	//处理 
	} 
}
```

先用epoll_create创建一个epoll对象实例epfd，同时返回一个引用该实例的文件描述符，返回的文件描述符仅仅指向对应的epoll实例，并不表示真实的磁盘文件节点。

epoll实例内部存储：

-   监听列表：所有要监听的文件描述符，使用红黑树；
-   就绪列表：所有就绪的文件描述符，使用链表；

再通过epoll_ctl将需要监视的fd添加到epfd中，同时为fd设置一个回调函数，并监听事件event，并添加到监听列表中。当有事件发生时，会调用回调函数，并将fd添加到epoll实例的就绪队列上。

最后调用epoll_wait阻塞监听 epoll 实例上所有的fd的 I/O 事件。当就绪列表中已有数据，那么epoll_wait直接返回，解决了select每次都需要轮询一遍的问题。

>epoll的优点：
>epoll的监听列表使用红黑树存储，epoll_ctl 函数添加进来的 fd 都会被放在红黑树的某个节点内，而红黑树本身插入和删除性能比较稳定，时间复杂度 O(logN)，并且可以存储大量的fd，避免了只能存储1024个fd的限制；
>epoll_ctl 中为每个文件描述符指定了回调函数，并在就绪时将其加入到就绪列表，因此不需要像select一样遍历检测每个文件描述符，只需要判断就绪列表是否为空即可；

epoll 是 Linux kernel 2.6 之后引入的新 I/O 事件驱动技术，I/O 多路复用的核心设计是 1 个线程处理所有连接的 `等待消息准备好` I/O 事件，这一点上 epoll 和 select&poll 是大同小异的。但 select&poll 错误预估了一件事，当数十万并发连接存在时，可能每一毫秒只有数百个活跃的连接，同时其余数十万连接在这一毫秒是非活跃的。select&poll 的使用方法是这样的： `返回的活跃连接 == select(全部待监控的连接)` 。

什么时候会调用 select&poll 呢？在你认为需要找出有报文到达的活跃连接时，就应该调用。所以，select&poll 在高并发时是会被频繁调用的。这样，这个频繁调用的方法就很有必要看看它是否有效率，因为，它的轻微效率损失都会被 `高频` 二字所放大。它有效率损失吗？显而易见，全部待监控连接是数以十万计的，返回的只是数百个活跃连接，这本身就是无效率的表现。被放大后就会发现，处理并发上万个连接时，select&poll 就完全力不从心了。这个时候就该 epoll 上场了，epoll 通过一些新的设计和优化，基本上解决了 select&poll 的问题。

epoll 的 API 非常简洁，涉及到的只有 3 个系统调用：

```go
#include <sys/epoll.h>  
int epoll_create(int size); // int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

其中，epoll_create 创建一个 epoll 实例并返回 epollfd；epoll_ctl 注册 file descriptor 等待的 I/O 事件(比如 EPOLLIN、EPOLLOUT 等) 到 epoll 实例上；epoll_wait 则是阻塞监听 epoll 实例上所有的 file descriptor 的 I/O 事件，它接收一个用户空间上的一块内存地址 (events 数组)，kernel 会在有 I/O 事件发生的时候把文件描述符列表复制到这块内存地址上，然后 epoll_wait 解除阻塞并返回，最后用户空间上的程序就可以对相应的 fd 进行读写了：

```go
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

epoll 的工作原理如下：

![](https://pic1.zhimg.com/80/v2-6697af18e4c3e16350529d15938adf6c_720w.webp)

与 select&poll 相比，epoll 分清了高频调用和低频调用。例如，epoll_ctl 相对来说就是非频繁调用的，而 epoll_wait 则是会被高频调用的。所以 epoll 利用 epoll_ctl 来插入或者删除一个 fd，实现用户态到内核态的数据拷贝，这确保了每一个 fd 在其生命周期只需要被拷贝一次，而不是每次调用 epoll_wait 的时候都拷贝一次。 epoll_wait 则被设计成几乎没有入参的调用，相比 select&poll 需要把全部监听的 fd 集合从用户态拷贝至内核态的做法，epoll 的效率就高出了一大截。

在实现上 epoll 采用红黑树来存储所有监听的 fd，而红黑树本身插入和删除性能比较稳定，时间复杂度 O(logN)。通过 epoll_ctl 函数添加进来的 fd 都会被放在红黑树的某个节点内，所以，重复添加是没有用的。当把 fd 添加进来的时候时候会完成关键的一步：该 fd 会与相应的设备（网卡）驱动程序建立回调关系，也就是在内核中断处理程序为它注册一个回调函数，在 fd 相应的事件触发（中断）之后（设备就绪了），内核就会调用这个回调函数，该回调函数在内核中被称为： `ep_poll_callback` ，**这个回调函数其实就是把这个 fd 添加到 rdllist 这个双向链表（就绪链表）中**。epoll_wait 实际上就是去检查 rdllist 双向链表中是否有就绪的 fd，当 rdllist 为空（无就绪 fd）时挂起当前进程，直到 rdllist 非空时进程才被唤醒并返回。

相比于 select&poll 调用时会将全部监听的 fd 从用户态空间拷贝至内核态空间并线性扫描一遍找出就绪的 fd 再返回到用户态，epoll_wait 则是直接返回已就绪 fd，因此 epoll 的 I/O 性能不会像 select&poll 那样随着监听的 fd 数量增加而出现线性衰减，是一个非常高效的 I/O 事件驱动技术。

**由于使用 epoll 的 I/O 多路复用需要用户进程自己负责 I/O 读写，从用户进程的角度看，读写过程是阻塞的，所以 select&poll&epoll 本质上都是同步 I/O 模型，而像 Windows 的 IOCP 这一类的异步 I/O，只需要在调用 WSARecv 或 WSASend 方法读写数据的时候把用户空间的内存 buffer 提交给 kernel，kernel 负责数据在用户空间和内核空间拷贝，完成之后就会通知用户进程，整个过程不需要用户进程参与，所以是真正的异步 I/O。**

### **延伸**

另外，我看到有些文章说 epoll 之所以性能高是因为利用了 Linux 的 mmap 内存映射让内核和用户进程共享了一片物理内存，用来存放就绪 fd 列表和它们的数据 buffer，所以用户进程在 `epoll_wait` 返回之后用户进程就可以直接从共享内存那里读取/写入数据了，这让我很疑惑，因为首先看 `epoll_wait` 的函数声明：

```go
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

第二个参数：就绪事件列表，是需要在用户空间分配内存然后再传给 `epoll_wait` 的，如果内核会用 mmap 设置共享内存，直接传递一个指针进去就行了，根本不需要在用户态分配内存，多此一举。其次，内核和用户进程通过 mmap 共享内存是一件极度危险的事情，内核无法确定这块共享内存什么时候会被回收，而且这样也会赋予用户进程直接操作内核数据的权限和入口，非常容易出现大的系统漏洞，因此一般极少会这么做。所以我很怀疑 epoll 是不是真的在 Linux kernel 里用了 mmap，我就去看了下最新版本（5.3.9）的 Linux kernel 源码：

```go
/*
 * Implement the event wait interface for the eventpoll file. It is the kernel
 * part of the user space epoll_wait(2).
 */
static int do_epoll_wait(int epfd, struct epoll_event __user *events,
    int maxevents, int timeout)
{
 ...
  
 /* Time to fish for events ... */
 error = ep_poll(ep, events, maxevents, timeout);
}

// 如果 epoll_wait 入参时设定 timeout == 0, 那么直接通过 ep_events_available 判断当前是否有用户感兴趣的事件发生，如果有则通过 ep_send_events 进行处理
// 如果设置 timeout > 0，并且当前没有用户关注的事件发生，则进行休眠，并添加到 ep->wq 等待队列的头部；对等待事件描述符设置 WQ_FLAG_EXCLUSIVE 标志
// ep_poll 被事件唤醒后会重新检查是否有关注事件，如果对应的事件已经被抢走，那么 ep_poll 会继续休眠等待
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout)
{
 ...
  
 send_events:
 /*
  * Try to transfer events to user space. In case we get 0 events and
  * there's still timeout left over, we go trying again in search of
  * more luck.
  */
  
 // 如果一切正常, 有 event 发生, 就开始准备数据 copy 给用户空间了
 // 如果有就绪的事件发生，那么就调用 ep_send_events 将就绪的事件 copy 到用户态内存中，
 // 然后返回到用户态，否则判断是否超时，如果没有超时就继续等待就绪事件发生，如果超时就返回用户态。
 // 从 ep_poll 函数的实现可以看到，如果有就绪事件发生，则调用 ep_send_events 函数做进一步处理
 if (!res && eavail &&
   !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
  goto fetch_events;
  
 ...
}

// ep_send_events 函数是用来向用户空间拷贝就绪 fd 列表的，它将用户传入的就绪 fd 列表内存简单封装到
// ep_send_events_data 结构中，然后调用 ep_scan_ready_list 将就绪队列中的事件写入用户空间的内存；
// 用户进程就可以访问到这些数据进行处理
static int ep_send_events(struct eventpoll *ep,
    struct epoll_event __user *events, int maxevents)
{
 struct ep_send_events_data esed;

 esed.maxevents = maxevents;
 esed.events = events;
 // 调用 ep_scan_ready_list 函数检查 epoll 实例 eventpoll 中的 rdllist 就绪链表，
 // 并注册一个回调函数 ep_send_events_proc，如果有就绪 fd，则调用 ep_send_events_proc 进行处理
 ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
 return esed.res;
}

// 调用 ep_scan_ready_list 的时候会传递指向 ep_send_events_proc 函数的函数指针作为回调函数，
// 一旦有就绪 fd，就会调用 ep_send_events_proc 函数
static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head, void *priv)
{
 ...
  
 /*
  * If the event mask intersect the caller-requested one,
  * deliver the event to userspace. Again, ep_scan_ready_list()
  * is holding ep->mtx, so no operations coming from userspace
  * can change the item.
  */
 revents = ep_item_poll(epi, &pt, 1);
 // 如果 revents 为 0，说明没有就绪的事件，跳过，否则就将就绪事件拷贝到用户态内存中
 if (!revents)
  continue;
 // 将当前就绪的事件和用户进程传入的数据都通过 __put_user 拷贝回用户空间,
 // 也就是调用 epoll_wait 之时用户进程传入的 fd 列表的内存
 if (__put_user(revents, &uevent->events) || __put_user(epi->event.data, &uevent->data)) {
  list_add(&epi->rdllink, head);
  ep_pm_stay_awake(epi);
  if (!esed->res)
   esed->res = -EFAULT;
  return 0;
 }
  
 ...
}
```

从 `do_epoll_wait` 开始层层跳转，我们可以很清楚地看到最后内核是通过 `__put_user` 函数把就绪 fd 列表和事件返回到用户空间，而 `__put_user` 正是内核用来拷贝数据到用户空间的标准函数。此外，我并没有在 Linux kernel 的源码中和 epoll 相关的代码里找到 mmap 系统调用做内存映射的逻辑，所以基本可以得出结论：epoll 在 Linux kernel 里并没有使用 mmap 来做用户空间和内核空间的内存共享，所以那些说 epoll 使用了 mmap 的文章都是误解。

# 解析
netpoll本质上是对 I/O 多路复用技术的封装，所以自然也是和epoll一样脱离不了下面几步：
- 1.  netpoll创建及其初始化；
- 2.  向netpoll中加入待监控的任务；
- 3.  从netpoll获取触发的事件；

在go中对epoll提供的三个函数进行了封装：
```go
func netpollinit() 
func netpollopen(fd uintptr, pd *pollDesc) int32 
func netpoll(delay int64) gList
```
netpollinit函数负责初始化netpoll；
netpollopen负责监听文件描述符上的事件；
netpoll会阻塞等待返回一组已经准备就绪的 Goroutine；

下面是Go语言中编写的一个TCP server：
```go
func main() { 
	listen, err := net.Listen("tcp", ":8888") 
	if err != nil { 
		fmt.Println("listen error: ", err) 
		return 
	} 
	for { 
		conn, err := listen.Accept() 
		if err != nil { 
			fmt.Println("accept error: ", err) 
			break 
		} 
		// 创建一个goroutine来负责处理读写任务 
		go HandleConn(conn) 
	} 
}
```

下面我们跟着这个TCP server的源码一起看看是在哪里使用了netpoll来完成epoll的调用。

## net.Listen

这个TCP server中会调用`net.Listen`创建一个socket同时返回与之对应的fd，该fd用来初始化listener的netFD(go层面封装的网络文件描述符)，接着调用 netFD的listenStream方法完成对 socket 的 bind&listen和netFD的初始化。

调用过程如下：

![listen](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210208204547398-858014858.png)

```go
func socket(ctx context.Context, 
			net string, 
			family, sotype, proto int, 
			ipv6only bool, 
			laddr, raddr sockaddr, 
			ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) { 
// 创建一个socket 
	s, err := sysSocket(family, sotype, proto) 
	if err != nil { 
		return nil, err 
	} 
	... 
	// 创建fd 
	if fd, err = newFD(s, family, sotype, net); err != nil { 
		poll.CloseFunc(s) 
		return nil, err 
	} 
	if laddr != nil && raddr == nil { 
		switch sotype { 
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET: 
			// 调用 netFD的listenStream方法完成对 socket 的 bind&listen和netFD的初始化 
			if err := fd.listenStream(laddr, listenerBacklog(), ctrlFn); err != nil { 
			fd.Close() 
			return nil, err 
	} 
			return fd, nil 
		case syscall.SOCK_DGRAM: 
			... 
		} 
	} 
		... 
	return fd, nil 
} 
func newFD(sysfd syscall.Handle, 
		   family, sotype int, 
		   net string) (*netFD, error) { 
ret := &netFD{ 
				pfd: poll.FD{ 
							Sysfd: sysfd, 
							IsStream: sotype == syscall.SOCK_STREAM, 
							ZeroReadIsEOF: sotype != syscall.SOCK_DGRAM && sotype != syscall.SOCK_RAW, 
							}, 
				family: family, 
				sotype: sotype, 
				net: net, 
			} 
return ret, nil 
}
```

sysSocket方法会发起一个系统调用创建一个socket，newFD会创建一个netFD，然后调用netFD的listenStream方法进行bind&listen操作，并对netFD进行init。

![netFD](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210208204547632-1827165543.png)

netFD是一个文件描述符的封装，netFD中包含一个FD数据结构，FD中包含了Sysfd 和pollDesc两个重要的数据结构，Sysfd是sysSocket返回的socket系统文件描述符，pollDesc用于监控文件描述符的可读或者可写。

我们继续看listenStream：
```go
func (fd *netFD) listenStream(laddr sockaddr, backlog int, 
							  ctrlFn func(string, string, syscall.RawConn) error) error { 	
	... 	
	// 完成绑定操作 	
	if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil { 		
		return os.NewSyscallError("bind", err) 	
	} 	
	// 进行监听操作 	
	if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil { 		
		return os.NewSyscallError("listen", err) 	
	} 	
	// 初始化fd 	
	if err = fd.init(); err != nil { 		
		return err 	
	} 	
	lsa, _ = syscall.Getsockname(fd.pfd.Sysfd) 	
	fd.setAddr(fd.addrFunc()(lsa), nil) 	
	return nil 
}
```

listenStream方法会调用Bind方法完成fd的绑定操作，然后调用listenFunc进行监听，接着调用fd的init方法，完成FD、pollDesc初始化。
```go
func (pd *pollDesc) init(fd *FD) error { 	
	// 调用到runtime.poll_runtime_pollServerInit
	serverInit.Do(runtime_pollServerInit) 	
	// 调用到runtime.poll_runtime_pollOpen 	
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd)) 	
	... 	
	return nil 
}
```

runtime_pollServerInit用Once封装保证只能被调用一次，这个函数在Linux平台上会创建一个epoll文件描述符实例；

poll_runtime_pollOpen调用了netpollopen会将fd注册到 epoll实例中，并返回一个pollDesc；

### netpollinit初始化
```go
func poll_runtime_pollServerInit() {
	netpollGenericInit() 
}  
func netpollGenericInit() { 	
	if atomic.Load(&netpollInited) == 0 { 		
		lock(&netpollInitLock) 		
		if netpollInited == 0 { 			
				netpollinit() 			
				atomic.Store(&netpollInited, 1) 		
		} 		
		unlock(&netpollInitLock) 	
	} 
}
```

netpollGenericInit会调用平台上特定实现的netpollinit，在Linux中会调用到netpoll_epoll.go的netpollinit方法：
```go
var ( 	
	epfd int32 = -1 // epoll descriptor  
)  
func netpollinit() { 	
	// 创建一个新的 epoll 文件描述符 	
	epfd = epollcreate1(_EPOLL_CLOEXEC) 	
	... 	
	// 创建一个用于通信的管道 	
	r, w, errno := nonblockingPipe() 	
	... 	
	ev := epollevent{ 		
		events: _EPOLLIN, 	
	} 	
	*(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd 	
	// 将读取数据的文件描述符加入监听 	
	errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev) 	
	... 	
	netpollBreakRd = uintptr(r) 	
	netpollBreakWr = uintptr(w) 
}
```

调用epollcreate1方法会创建一个epoll文件描述符实例，需要注意的是epfd是一个全局的属性。然后创建一个用于通信的管道，调用epollctl将读取数据的文件描述符加入监听。

### netpollopen加入事件监听

下面再看看poll_runtime_pollOpen方法：
```go
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) { 	
	pd := pollcache.alloc() 	
	lock(&pd.lock) 	
	if pd.wg != 0 && pd.wg != pdReady { 		
		throw("runtime: blocked write on free polldesc") 	
	} 	
	if pd.rg != 0 && pd.rg != pdReady { 		
		throw("runtime: blocked read on free polldesc") 	
	} 	
	pd.fd = fd 	
	pd.closing = false 	
	pd.everr = false 	
	pd.rseq++ 	
	pd.rg = 0 	
	pd.rd = 0 	
	pd.wseq++ 	
	pd.wg = 0 	
	pd.wd = 0 	
	pd.self = pd 	
	unlock(&pd.lock)  	
	var errno int32 	
	errno = netpollopen(fd, pd) 	
	return pd, int(errno) 
}  
func netpollopen(fd uintptr, pd *pollDesc) int32 { 	
	var ev epollevent 	
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET 	
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd 	
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev) 
}
```

poll_runtime_pollOpen方法会通过`pollcache.alloc`初始化总大小约为 4KB的pollDesc结构体。然后重置pd的属性，调用netpollopen向epoll实例epfd加入新的轮询事件监听文件描述符的可读和可写状态。

下面我们再看看pollCache是如何初始化pollDesc的。
```go
type pollCache struct { 	
	lock  mutex 	
	first *pollDesc  
}  
const pollBlockSize = 4 * 1024  
func (c *pollCache) alloc() *pollDesc { 	
		lock(&c.lock) 	
		// 初始化首节点 	
		if c.first == nil { 		
			const pdSize = unsafe.Sizeof(pollDesc{}) 		
			n := pollBlockSize / pdSize 		
			if n == 0 { 			
				n = 1 		
			}  		
			mem := persistentalloc(n*pdSize, 0, &memstats.other_sys)         
			// 初始化pollDesc链表 		
			for i := uintptr(0); i < n; i++ { 			
				pd := (*pollDesc)(add(mem, i*pdSize)) 			
				pd.link = c.first 			
				c.first = pd 		
			} 	
		} 	
		pd := c.first 	
		c.first = pd.link 	
		lockInit(&pd.lock, lockRankPollDesc) 	
		unlock(&c.lock) 	
		return pd 
}
```

pollCache的链表头如果为空，那么初始化首节点，首节点是一个pollDesc的链表头，每次调用该结构体都会返回链表头还没有被使用的pollDesc。

![pollCache](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210208204547962-1949357529.png)

到这里就完成了net.Listen的分析，下面我们看看listen.Accept。

## Listener.Accept

Listener.Accept方法最终会调用到netFD的accept方法中：

![Accept](https://img2020.cnblogs.com/blog/1204119/202102/1204119-20210208204548221-944194477.png)

```go
func (fd *netFD) accept() (netfd *netFD, err error) { 	
	// 调用netfd.FD的Accept接受新的 socket 连接，返回 socket 的 fd 	
	d, rsa, errcall, err := fd.pfd.Accept() 	
	... 	
	// 构造一个新的netfd 	
	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil { 		
		poll.CloseFunc(d) 		
		return nil, err 	
	} 	
	// 调用 netFD 的 init 方法完成初始化 	
	if err = netfd.init(); err != nil { 		
		netfd.Close() 		
		return nil, err 	
	} 	
	lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd) 	
	netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa)) 	
	return netfd, nil 
}
```

这个方法首先会调用到FD的Accept接受新的 socket 连接，并返回新的socket对应的fd，然后调用newFD构造一个新的netfd，并通过init 方法完成初始化。

init方法上面我们已经看过了，下面我们来看看Accept方法：
```go
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) { 	
	... 	
	for { 		
		// 使用 linux 系统调用 accept 接收新连接，创建对应的 socket 		
		s, rsa, errcall, err := accept(fd.Sysfd) 		
		if err == nil { 			
			return s, rsa, "", err 		
		} 		
		switch err { 		
		case syscall.EINTR: 			
			continue 		
		case syscall.EAGAIN: 			
			if fd.pd.pollable() { 				
				// 如果当前没有发生期待的 I/O 事件，那么 waitRead 会通过 park goroutine 让逻辑 block 在这里 				
				if err = fd.pd.waitRead(fd.isFile); err == nil {
				 		continue 				
				} 			
			} 		
		case syscall.ECONNABORTED:  			
			continue 		
		} 		
		return -1, nil, errcall, err 	
	} 
}
```

`FD.Accept`方法会使用 linux 系统调用 accept 接收新连接，创建对应的 socket，如果没有可读的消息，waitRead会被阻塞。这些被park住的goroutine会在goroutine的调度中调用`runtime.netpoll`被唤醒。

### pollWait事件等待

`pollDesc.waitRead`实际上是调用了`runtime.poll_runtime_pollWait`
```go
func poll_runtime_pollWait(pd *pollDesc, mode int) int { 	
	...     
	// 进入 netpollblock 并且判断是否有期待的 I/O 事件发生 	
	for !netpollblock(pd, int32(mode), false) { 		
		... 	
	} 	
	return 0 
}  
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool { 	
	gpp := &pd.rg 	
	if mode == 'w' { 		
		gpp = &pd.wg 	
	} 	
	// 这个 for 循环是为了等待 io ready 或者 io wait 	
	for { 		
		old := *gpp 		
		// gpp == pdReady 表示此时已有期待的 I/O 事件发生， 		
		// 可以直接返回 unblock 当前 goroutine 并执行响应的 I/O 操作 		
		if old == pdReady { 			
			*gpp = 0 			
			return true 		
		} 		
		if old != 0 { 			
			throw("runtime: double wait") 		
		} 		
		// 如果没有期待的 I/O 事件发生，则通过原子操作把 gpp 的值置为 pdWait 并退出 for 循环 		
		if atomic.Casuintptr(gpp, 0, pdWait) { 			
			break 		
		} 	
	} 	
	if waitio || netpollcheckerr(pd, mode) == 0 { 		
		// 让出当前线程，将 Goroutine 转换到休眠状态并等待运行时的唤醒 		
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5) 	
	} 	
	// be careful to not lose concurrent pdReady notification 	
	old := atomic.Xchguintptr(gpp, 0) 	
	if old > pdWait { 		
		throw("runtime: corrupted polldesc") 	
	} 	
	return old == pdReady 
}
```

poll_runtime_pollWait会用for循环调用netpollblock函数判断是否有期待的 I/O 事件发生，直到netpollblock返回true表示io ready才会走出循环。

netpollblock方法会判断当前的状态是不是处于pdReady，如果是那么直接返回true；如果不是，那么将gpp通过CAS设置为pdWait并退出 for 循环。通过gopark 把当前 goroutine 给 park 住，直到对应的 fd 上发生可读/可写或者其他I/O 事件为止。

这些被park住的goroutine会在goroutine的调度中调用`runtime.netpoll`被唤醒。

## netpoll轮询等待

`runtime.netpoll`的核心逻辑是： 根据入参 delay设置调用 epoll_wait 的 timeout 值，调用 epoll_wait 从 epoll 的 `eventpoll.rdllist`双向列表中获取IO就绪的fd列表，遍历epoll_wait 返回的fd列表， 根据调用`epoll_ctl`注册fd时封装的上下文信息组装可运行的 goroutine 并返回。

执行完 `netpoll` 之后，会返回一个就绪 fd 列表对应的 goroutine 列表，接下来将就绪的 goroutine 加入到调度队列中，等待调度运行。
```go
func netpoll(delay int64) gList { 
		if epfd == -1 { 
			return gList{} 
		} 
		var waitms int32 
		// 因为传入delay单位是纳秒，下面将纳秒转换成毫秒 
		if delay < 0 { 
			waitms = -1 
		} else if delay == 0 { 
			waitms = 0 
		} else if delay < 1e6 { 
			waitms = 1 
		} else if delay < 1e15 { 
			waitms = int32(delay / 1e6) 
		} else { 
		// An arbitrary cap on how long to wait for a timer. 
		// 1e9 ms == ~11.5 days. 
		waitms = 1e9 
		} 
		var events [128]epollevent 
	retry: 
		// 等待文件描述符转换成可读或者可写 
		n := epollwait(epfd, &events[0], int32(len(events)), waitms) 
		// 返回负值，那么重新调用epollwait进行等待 
		if n < 0 { 
			... 
			goto retry 
		} 
		var toRun gList 
		// 意味着被监控的文件描述符出现了待处理的事件 
		for i := int32(0); i < n; i++ { 
			ev := &events[i] 
			if ev.events == 0 { 
				continue 
			} 
			... 
			// 判断发生的事件类型，读类型或者写类型 
			var mode int32 
			if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 { 
				mode += 'r' 
			} 
			if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 { 
				mode += 'w' 
			} 
			if mode != 0 { 
				// 取出保存在 epollevent 里的 pollDesc 
				pd := *(**pollDesc)(unsafe.Pointer(&ev.data)) 
				pd.everr = false 
				if ev.events == _EPOLLERR { 
					pd.everr = true 
				} 
				// 调用 netpollready，传入就绪 fd 的 pollDesc
				netpollready(&toRun, pd, mode) 
			} 
		} 
		return toRun 
}
```

netpoll会调用epollwait获取就绪的 fd 列表，对应的epoll函数是epoll_wait。toRun是一个 g 的链表，存储要恢复的 goroutines，最后返回给调用方。如果epollwait返回的n大于零，那么表示被监控的文件描述符出现了待处理的事件，那么需要调用for循环进行处理。循环里面会根据时间类型设置mode，然后拿出对应的pollDesc，调用netpollready方法。

下面我们再看一下netpollready：
```go
func netpollready(toRun *gList, pd *pollDesc, mode int32) { 	
	var rg, wg *g 	
	// 获取对应的g的指针 	
	if mode == 'r' || mode == 'r'+'w' { 		
			rg = netpollunblock(pd, 'r', true) 	
	} 	
	if mode == 'w' || mode == 'r'+'w' { 		
			wg = netpollunblock(pd, 'w', true) 	
	} 	
	// 将对应的g加入到toRun列表中 	
	if rg != nil { 		
			toRun.push(rg) 	
	} 	
	if wg != nil { 		
			toRun.push(wg) 	
	} 
}  
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g { 	
		gpp := &pd.rg 	
		// 根据传入的mode判断事件类型 	
		if mode == 'w' { 		
			gpp = &pd.wg 	
		}  	
		for { 		
			// 取出 gpp 存储的 g 		
			old := *gpp 		
			if old == pdReady { 			
				return nil 		
			} 		
			if old == 0 && !ioready { 			
				return nil 		
			} 		
			var new uintptr 		
			if ioready { 			
				new = pdReady 		
			} 		
			// cas 将读或者写信号量转换成 pdReady 		
			if atomic.Casuintptr(gpp, old, new) { 			
				if old == pdWait { 				
						old = 0 			
				} 			
				// 返回对应的 g指针 			
				return (*g)(unsafe.Pointer(old)) 		
			} 	
		} 
}
```

讲完了`runtime.netpoll`的源码有个需要注意的地方，调用`runtime.netpoll`的地方有两处：

-   在调度器中执行`runtime.schedule()`，该方法中会执行`runtime.findrunable()`，在`runtime.findrunable()`中调用了`runtime.netpoll`获取待执行的goroutine；
-   Go runtime 在程序启动的时候会创建一个独立的sysmon监控线程，sysmon 每 20us~10ms 运行一次，每次运行会检查距离上一次执行netpoll是否超过10ms，如果是则会调用一次`runtime.netpoll`；

这些入口的调用感兴趣的可以自己去看看。

## 总结[#](https://www.cnblogs.com/luozhiyun/p/14390824.html#1229223999)

本文从I/O多路复用开始讲解select以及epoll，然后再回到go语言中去看它是如何实现多路复用这样的结构的。通过追踪源码可以发现，其实go也是根据epoll来封装自己的函数：
```go
func netpollinit() 
func netpollopen(fd uintptr, pd *pollDesc) int32 
func netpoll(block bool) gList
```

通过这三个函数来实现对epoll的创建实例、注册、事件等待操作。

对于I/O多路复用不是很了解的同学也可以借此机会多多的去学习一下网络编程方面的知识，扩充一下知识面。

# Reference
https://www.cnblogs.com/luozhiyun/p/14390824.html
https://www.luozhiyun.com