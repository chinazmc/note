#golang 

## Netpoll的优化点

go的net库是BIO的，浪费了**更多的goroutine在阻塞，并且难以对连接池中的连接进行探活**。 netpoll采用了LT的触发方式，这种触发方式也就导致编程思路的不同

#### ET

![1642064579869-64deb77c-4845-4103-b735-94a0eb19937b.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b698848d3fa4ffebc571c1d4fc5ff88~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

#### LT

![1642064598829-68afa0a4-1e15-4598-aca0-a4b924e1645c.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83c21aabdd654178bd72531c26d75295~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

netpoll采用LT的编程思路 由于netpoll想在 系统调用 和 buffer 上做优化，所以采用LT的形式。

### 优化系统调用

syscall这个方法其实有三步：

0.  enter_runtime
1.  raw_syscall
2.  exit_runtime

由于系统调用是一个耗时的阻塞操作，容易造成goroutine阻塞，所以需要加入一些**runtime的调度流程**。 但是，epoll_wait触发的事件，保证不会被阻塞，所以netpoll直接采用RawSyscall方法做系统调用，跳过了runtime的一些逻辑。

### 优化调度

使用msec动态调参和runtime.Gosched主动让出P

##### msec动态调参

epoll_wait的系统调用有个参数是，等待时间，设置成-1是无限等待事件到来，0是不等待。

![1642066647357-64124af2-f093-4edd-bb41-d7117ce77b4d.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/518792312eb24661b2738edc6323c992~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这样就有事件到来的时候下次循环的epoll_wait采用立即返回，没有事件就一直阻塞，减少反复无用的调用。

##### runtime.Gosched主动让出P

如果msec为-1的话会立即进入下一次循环，开启新的epoll_wait调用，那么调用就阻塞在这里，goroutine阻塞时间长了之后会被runtime切换掉，只能等到下一次执行这个goroutine才行，导致时间浪费。 ne**tpoll调用runtime.Gosched方法主动将GMP中的P让出，减少runtime的调度过程**。

### 优化buffer

我们在读取和写入数据的时候需要使用到buffer。 多数框架使用环形buffer，可以做到流式读写。但是环形buffer容量是死的，需要扩容的话，需要重新copy数组，引入了很多的并发问题。

##### LinkBuffer

netpoll使用的buffer实现包括：

-   链表解决扩容copy问题
-   sync.Pool复用链表节点
-   atomic访问size，规避data race和锁竞争

还有一些nocopy方面的优化，减少了write和read的次数，从而提高了读取和发送的时候的编解码效率。
