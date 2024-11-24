---
sr-due: 2024-10-15
sr-interval: 12
sr-ease: 230
---

#golang/channel   #note 

# 二、channel的存在定位

从内存的角度而言，并发模型只分两种：基于共享内存和基于消息通信（内存拷贝）。在Go中，两种并发模型的同步原语均有提供：sync.\*和atomic.\*代表的就是基于共享内存；channel代表的就是基于消息通信。而Go提倡后者，它包括三大元素：goroutine（执行体），channel（通信），select（协调）。

Do not communicate by sharing memory; instead, share memory by communicating.

在Go中通过goroutine+channel的方式，可以简单、高效地解决并发问题，channel就是goroutine之间的数据桥梁。

Concurrency is the key to designing high performance network services. Go's concurrency primitives (goroutines and channels) provide a simple and efficient means of expressing concurrent execution.
# 三、channel源码解析

channel源码位于src/go/runtime/chan.go。本章内容分为两部分：channel内部结构和channel操作。

## 3.1 channel内部结构

```go
package main
import "fmt"

func main() {
    c := make(chan int,2)

    go func() {
        c <- 1 // send to channel
    }()

    x := <-c // recv from channel

    fmt.Println(x)
}
```

对于以上channel的申明语句，我们可以在程序中加入断点，得到ch的信息如下。

![](https://pic4.zhimg.com/80/v2-1913a28388dbc472b692065ed4fe193f_720w.jpg)
通过查看汇编结果
```bash
go tool compile -N -l -S hello.go
-N表示禁用优化
-l禁用内联
-S打印结果
```
![[Pasted image 20220706153913.png]]
很好，看起来非常的清晰。但是，这些信息代表的是什么含义呢？接下来，我们先看几个重要的结构体。

### hchan
当我们通过make(chan Type, size)生成channel时，在runtime系统中，生成的是一个hchan结构体对象。源码位于src/runtime/chan.go

```go
type hchan struct {
    qcount   uint           // 循环队列中数据数
    dataqsiz uint           // 循环队列的大小
    buf      unsafe.Pointer // 指向大小为dataqsize的包含数据元素的数组指针
    elemsize uint16         // 数据元素的大小
    closed   uint32         // 代表channel是否关闭   
    elemtype *_type         // _type代表Go的类型系统，elemtype代表channel中的元素类型
    sendx    uint           // 发送索引号，初始值为0
    recvx    uint           // 接收索引号，初始值为0
	recvq    waitq          // 接收等待队列，存储试图从channel接收数据(<-ch)的阻塞goroutines
    sendq    waitq          // 发送等待队列，存储试图发送数据(ch<-)到channel的阻塞goroutines

    lock mutex              // 加锁能保护hchan的所有字段，包括waitq中sudoq对象
}
```
- qcount代表chan 中已经接收但还没被取走的元素的个数，函数 len 可以返回这个字段的值；
- dataqsiz和buf分别代表队列buffer的大小，cap函数可以返回这个字段的值以及队列buffer的指针，是一个定长的环形数组；
- elemtype 和 elemsiz表示chan 中元素的类型和 元素的大小；
- sendx：发送数据的指针在 buffer中的位置；
- recvx：接收请求时的指针在 buffer 中的位置；
- recvq和sendq分别表示等待接收数据的 goroutine 与等待发送数据的 goroutine；
- sendq和recvq的类型是waitq的结构体;

### waitq
waitq用于表达处于阻塞状态的goroutines链表信息，first指向链头goroutine，last指向链尾goroutine
```go
type waitq struct {
    first *sudog           
    last  *sudog
}
```
整个chan的图例大概是这样：
![[Pasted image 20220706154355.png]]
#### sudog
sudog代表的就是一个处于等待列表中的goroutine对象，源码位于src/runtime/runtime2.go
```go
type sudog struct {
    g *g
    next *sudog
    prev *sudog
    elem unsafe.Pointer // data element (may point to stack)
    c        *hchan // channel
  ...
}
```

为了更好理解hchan结构体，我们将通过以下代码来理解hchan中的字段含义。

```go
package main

import "time"

func goroutineA(ch chan int) {
    ch <- 100
}

func goroutineB(ch chan int) {
    ch <- 200
}

func goroutineC(ch chan int) {
    ch <- 300
}

func goroutineD(ch chan int) {
    ch <- 300
}

func main() {
    ch := make(chan int, 4)
    for i := 0; i < 4; i++ {
        ch <- i * 10
    }
    go goroutineA(ch)
    go goroutineB(ch)
    go goroutineC(ch)
    go goroutineD(ch)
    // 第一个sleep是为了给上足够的时间让所有goroutine都已启动
    time.Sleep(time.Millisecond * 500)
    time.Sleep(time.Second)
}
```

打开代码调试功能，将程序运行至断点time.Sleep(time.Second)处，此时得到的chan信息如下。

![](https://pic2.zhimg.com/80/v2-35fd1f36aeed997ab803b0db7e3bf275_720w.jpg)

在该channel中，通过make(chan int, 4)定义的channel大小为4，即dataqsiz的值为4。同时由于循环队列中已经添加了4个元素，所以qcount值也为4。此时，有4个goroutine（A-D）想发送数据给channel，但是由于存放数据的循环队列已满，所以只能进入发送等待列表，即sendq。同时要注意到，此时的发送和接收索引值均为0，即下一次接收数据的goroutine会从循环队列的第一个元素拿，发送数据的goroutine会发送到循环队列的第一个位置。

上述hchan结构可视化图解如下

![](https://pic1.zhimg.com/80/v2-7d63d1340a112e30c2094f0a14faacd4_720w.jpg)

## 3.2 channel操作
将channel操作分为四部分：创建、发送、接收和关闭。

### 创建

本文的参考Go版本为1.15.2。其channel的创建实现代码位于src/go/runtime/chan.go的makechan方法。

```go
const (
    maxAlign  = 8
    hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1)) 
)
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

  // 发送元素大小限制
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
  // 对齐检查
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }

  // 计算需要分配的buf空间,判断是否会内存溢出
    mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0 {
        panic(plainError("makechan: size out of range"))
    }

  // 为构造的hchan对象分配内存
    var c *hchan
    switch {
  // 无缓冲的channel或者元素大小为0的情况
    case mem == 0:
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        c.buf = c.raceaddr()
  // 元素不包含指针的情况  
    case elem.ptrdata == 0:
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
  // 元素包含指针  
    default:
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

  // 初始化相关参数
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    lockInit(&c.lock, lockRankHchan)

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
    }
    return c
}
```
`maxAlign是8，那么maxAlign-1的二进制就是111，然后和int(unsafe.Sizeof(hchan{}))取与就是取它的低三位，hchanSize就得到的是8的整数倍，做对齐使用。

这里switch有三种情况，
- 第一种情况是缓冲区所需大小为 0，那么在为 hchan 分配内存时，只需要分配 sizeof(hchan) 大小的内存；
- 第二种情况是缓冲区所需大小不为 0，而且数据类型不包含指针，那么就分配连续的内存。注意的是，我们在创建channel的时候可以指定类型为指针类型;
- 第三种情况是缓冲区所需大小不为 0，而且数据类型包含指针，那么就不使用add的方式让hchan和buf放在一起了，而是单独的为buf申请一块内存。

可以看到，makechan方法主要就是检查传送元素的合法性，并为hchan分配内存，初始化相关参数，包括对锁的初始化。

### 发送
在看发送数据的代码之前，我们先看一下什么是channel的阻塞和非阻塞。

一般情况下，传入的参数都是 `block=true`，即阻塞调用，一个往 channel 中插入数据的 goroutine 会阻塞到插入成功为止。

非阻塞是只这种情况：
```go
select {
case c <- v:
    ... foo
default:
    ... bar
}
```

编译器会将其改为：

```go
if selectnbsend(c, v) {
    ... foo
} else {
    ... bar
}
```

selectnbsend方法传入的block就是false：

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    return chansend(c, elem, false, getcallerpc())
}
```

#### chansend方法
channel的发送实现代码位于src/go/runtime/chan.go的chansend方法。发送过程，存在以下几种情况。
##### 1.  当发送的channel为nil
```go
if c == nil {
//对于非阻塞的发送，直接返回
    if !block {
        return false
    }
 // 对于阻塞的通道，将 goroutine 挂起
    gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
    throw("unreachable")
}
```
这里会对chan做一个判断，如果它是空的，那么对于非阻塞的发送，直接返回 false；对于阻塞的通道，将 goroutine 挂起，并且永远不会返回。

##### 2.  往已关闭的channel中发送数据

```go
if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
```

如果向已关闭的channel中发送数据，会引发panic。
##### 3. 非阻塞且通道没有关闭，但是无法发送数据
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    ...
    // 非阻塞的情况下，如果通道没有关闭，满足以下一条：
    // 1.没有缓冲区并且当前没有接收者   
    // 2.缓冲区不为0，并且已满
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }
    ...
}
```
需要注意的是这里是没有加锁的，go虽然在使用指针读取单个值的时候原子性的，但是读取多个值并不能保证，所以在判断完closed虽然是没有关闭的，那么在读取完之后依然可能在这一瞬间从未关闭状态转变成关闭状态。那么就有两种可能：

-   通道没有关闭，而且已经满了，那么需要返回false，没有问题；
-   通道关闭，而且已经满了，但是在非阻塞的发送中返回false，也没有问题；

有关go的一致性原语，可以看这篇：[The Go Memory Model](https://golang.org/ref/mem)。

上面的这些判断被称为 fast path，因为加锁的操作是一个很重的操作，所以能够在加锁之前返回的判断就在加锁之前做好是最好的。
下面看看加锁部分的代码
##### 4.  如果已经有阻塞的接收goroutines（即recvq中指向非空），那么数据将被直接发送给接收goroutine。
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    ...
    //加锁
    lock(&c.lock)
    // 是否关闭的判断
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    // 从 recvq 中取出一个接收者
    if sg := c.recvq.dequeue(); sg != nil { 
        // 如果接收者存在，直接向该接收者发送数据，绕过buffer
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    ...
}
```
进入了lock区域之后还需要再判断以下close的状态，然后从recvq 中取出一个接收者，如果已经有接收者，那么就向第一个接收者发送当前enqueue的消息。这里需要注意的是如果有接收者在队列中等待，则说明此时的缓冲区是空的。

该逻辑的实现代码在send方法和sendDirect中。

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
  ... // 省略了竞态代码
    if sg.elem != nil {
	    //直接把要发送的数据copy到reciever的栈空间
        sendDirect(c.elemtype, sg, ep)
        sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1)
}

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
    memmove(dst, src, t.size)
}
```
在send方法里，sg就是goroutine打包好的对象，ep是对应要发送数据的指针，sendDirect方法会调用memmove进行数据的内存拷贝
其中，memmove我们已经在源码系列中遇到多次了，它的目的是将内存中src的内容拷贝至dst中去。另外，注意到goready(gp, skip+1)这句代码，它会使得之前在接收等待队列中的第一个goroutine的状态变为runnable，这样go的调度器就可以重新让该goroutine得到执行。

###### 1.  对于有缓冲的channel来说，如果当前缓冲区hchan.buf有可用空间，那么会将数据拷贝至缓冲区

```go
if c.qcount < c.dataqsiz {
	//找到buf要填充数据的索引位置
    qp := chanbuf(c, c.sendx)
    if raceenabled {
        raceacquire(qp)
        racerelease(qp)
    }
    //将数据拷贝到buffer中
    typedmemmove(c.elemtype, qp, ep)
  // 发送索引号+1
    c.sendx++
  // 因为存储数据元素的结构是循环队列，所以当当前索引号已经到队末时，将索引号调整到队头
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
  // 当前循环队列中存储元素数+1
    c.qcount++
    unlock(&c.lock)
    return true
}
```

其中，chanbuf(c, c.sendx)是获取指向对应内存区域的指针。typememmove会调用memmove方法，完成数据的拷贝工作。另外注意到，当对hchan进行实际操作时，是需要调用lock(&c.lock)加锁，因此，在完成数据拷贝后，通过unlock(&c.lock)将锁释放。

###### 2.  有缓冲的channel，当hchan.buf已满；或者无缓冲的channel，当前没有接收的goroutine
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    ...
    // 缓冲区没有空间了，所以对于非阻塞调用直接返回
    if !block {
        unlock(&c.lock)
        return false
    }
    // 创建 sudog 对象
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    // 将sudog 对象入队
    c.sendq.enqueue(mysg)
    // 进入等待状态
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    ...
}
```

通过getg获取当前执行的goroutine。acquireSudog是先获得当前执行goroutine的线程M，再获取M对应的P，最后将P的sudugo缓存队列中的队头sudog取出（详见源码src/runtime/proc.go）。通过c.sendq.enqueue将sudug加入到channel的发送等待列表中，并调用gopark将当前goroutine转为waiting态。

#### 小结
-   发送操作会对hchan加锁。
-   当recvq中存在等待接收的goroutine时，数据元素将会被直接拷贝给接收goroutine。
-   当recvq等待队列为空时，会判断hchan.buf是否可用。如果可用，则会将发送的数据拷贝至hchan.buf中。
-   如果hchan.buf已满，那么将当前发送goroutine置于sendq中排队，并在运行时中挂起。
-   向已经关闭的channel发送数据，会引发panic。

对于无缓冲的channel来说，它天然就是hchan.buf已满的情况，因为它的hchan.buf的容量为0。

```go
package main

import "time"

func main() {
    ch := make(chan int)
    go func(ch chan int) {
        ch <- 100
    }(ch)
    time.Sleep(time.Millisecond * 500)
    time.Sleep(time.Second)
}
```

在上述示例中，发送goroutine向无缓冲的channel发送数据，但是没有接收goroutine。将断点置于time.Sleep(time.Second)，得到此时ch结构如下。

![](https://pic3.zhimg.com/80/v2-5765953bbdb3eff9ab30121b6c825a76_720w.jpg)

可以看到，在无缓冲的channel中，其hchan的buf长度为0，当没有接收groutine时，发送的goroutine将被置于sendq的发送队列中。

![Group100](https://img.luozhiyun.com/20210110110807.svg)

1.  检查 recvq 是否为空，如果不为空，则从 recvq 头部取一个 goroutine，将数据发送过去；
2.  如果 recvq 为空，，并且buf没有满，则将数据放入到 buf中；
3.  如果 buf已满，则将要发送的数据和当前 goroutine 打包成sudog，然后入队到sendq队列中，并将当前 goroutine 置为 waiting 状态进行阻塞。


### 接收
channel的接收实现分两种，v :=<-ch对应于chanrecv1，v, ok := <- ch对应于chanrecv2，但它们都依赖于位于src/go/runtime/chan.go的chanrecv方法。

```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}
```
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
    if c == nil {
        // 如果 c 为空且是非阻塞调用，那么直接返回 (false,false)
        if !block {
            return
        }
        // 阻塞调用直接等待
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    // 对于非阻塞的情况，并且没有关闭的情况
    // 如果是无缓冲chan或者是chan中没有数据，那么直接返回 (false,false)
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
        c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
        atomic.Load(&c.closed) == 0 {
        return
    }
    // 上锁
    lock(&c.lock)
    // 如果已经关闭，并且chan中没有数据，返回 (true,false)
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
    ...
}
```
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
    // 从发送者队列获取数据
    if sg := c.sendq.dequeue(); sg != nil { 
        // 发送者队列不为空，直接从发送者那里提取数据
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    } 
    ...
}

func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // 如果是无缓冲区chan
    if c.dataqsiz == 0 {
        ...
        if ep != nil {
            // 直接从发送者拷贝数据
            recvDirect(c.elemtype, sg, ep)
        }
    // 有缓冲区chan
    } else { 
        // 获取buf的存放数据指针
        qp := chanbuf(c, c.recvx) 
        ...
        // 直接从缓冲区拷贝数据给接收者
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        } 
        // 从发送者拷贝数据到缓冲区
        typedmemmove(c.elemtype, qp, sg.elem)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    // 将发送者唤醒
    goready(gp, skip+1)
}
```
在这里如果有发送者在队列等待，那么直接从发送者那里提取数据，并且唤醒这个发送者。需要注意的是由于有发送者在等待，**所以如果有缓冲区，那么缓冲区一定是满的**。

在唤醒发送者之前需要对缓冲区做判断，如果是无缓冲区，那么直接从发送者那里提取数据；如果有缓冲区首先会获取recvx的指针，然后将从缓冲区拷贝数据给接收者，再将发送者数据拷贝到缓冲区。

然后将recvx加1，相当于将新的数据移到了队尾，再将recvx的值赋值给sendx，最后调用goready将发送者唤醒，这里有些绕，我们通过图片来展示：

![Group 66](https://img.luozhiyun.com/20210110110811.svg)

这里展示的是在chansend中将数据拷贝到缓冲区中，当数据满的时候会将sendx的指针置为0，所以当buf环形队列是满的时候sendx等于recvx。

然后再来看看chanrecv中发送者队列有数据的时候移交缓冲区的数据是怎么做的：

![Group 85](https://img.luozhiyun.com/20210110110816.svg)

这里会将recvx为0处的数据直接从缓存区拷贝数据给接收者，然后将发送者拷贝数据到缓冲区recvx指针处，然后将recvx指针加1并将recvx赋值给sendx，由于是满的所以用recvx加1的效果实现了将新加入的数据入库到队尾的操作。

接着往下看：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
    // 如果缓冲区中有数据
    if c.qcount > 0 { 
        qp := chanbuf(c, c.recvx)
        ...
        // 从缓冲区复制数据到 ep
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        // 接收数据的指针前移
        c.recvx++
        // 环形队列，如果到了末尾，再从0开始
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        // 缓冲区中现存数据减一
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    ...
}
```

到了这里，说明缓冲区中有数据，但是发送者队列没有数据，那么将数据拷贝到接收数据的协程，然后将接收数据的指针前移，如果已经到了队尾，那么就从0开始，最后将缓冲区中现存数据减一并解锁。
面就是缓冲区中没有数据的情况：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
    // 非阻塞，直接返回
    if !block {
        unlock(&c.lock)
        return false, false
    } 
    // 创建sudog
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    } 
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    // 将sudog添加到接收队列中
    c.recvq.enqueue(mysg)
    // 阻塞住goroutine，等待被唤醒
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
    ...
}
```

如果是非阻塞调用，直接返回；阻塞调用会将当前goroutine 封装成sudog，然后将sudog添加到接收队列中，调用gopark阻塞住goroutine，等待被唤醒。

和chansend逻辑对应，具体处理准则如下。

-   接收操作会对hchan加锁。
-   当sendq中存在等待发送的goroutine时，意味着此时的hchan.buf已满（无缓存的天然已满），分两种情况（见代码src/go/runtime/chan.go的recv方法）：1. 如果是有缓存的hchan，那么先将缓冲区的数据拷贝给接收goroutine，再将sendq的队头sudog出队，将出队的sudog上的元素拷贝至hchan的缓存区。 2. 如果是无缓存的hchan，那么直接将出队的sudog上的元素拷贝给接收goroutine。两种情况的最后都会唤醒出队的sudog上的发送goroutine。
-   当sendq发送队列为空时，会判断hchan.buf是否可用。如果可用，则会将hchan.buf的数据拷贝给接收goroutine。
-   如果hchan.buf不可用，那么将当前接收goroutine置于recvq中排队，并在运行时中挂起。
-   与发送不同的是，当channel关闭时，goroutine还能从channel中获取数据。如果recvq等待列表中有goroutines，那么它们都会被唤醒接收数据。如果hchan.buf中还有未接收的数据，那么goroutine会接收缓冲区中的数据，否则goroutine会获取到元素的零值。

以下是channel关闭之后，接收goroutine的读取示例代码。

```go
func main() {
    ch := make(chan int, 1)
    ch <- 10
    close(ch)
    a, ok := <-ch
    fmt.Println(a, ok)
    b, ok := <-ch
    fmt.Println(b, ok)
    c := <-ch
    fmt.Println(c)
}

//输出如下
10 true
0 false
0
```

**注意：在channel中进行的所有元素转移都伴随着内存的拷贝。**

```go
func main() {
    type Instance struct {
        ID   int
        name string
    }

    var ins = Instance{ID: 1, name: "Golang"}

    ch := make(chan Instance, 3)
    ch <- ins

    fmt.Println("ins的原始值：", ins)

    ins.name = "Python"
    go func(ch chan Instance) {
        fmt.Println("channel接收值：", <-ch)
    }(ch)

    time.Sleep(time.Second)
    fmt.Println("ins的最终值：", ins)
}

// 输出结果
ins的原始值： {1 Golang}
channel接收值： {1 Golang}
ins的最终值： {1 Python}
```

前半段图解如下

![](https://pic1.zhimg.com/80/v2-2cc277f960a5e55af0648af82e61fcac_720w.jpg)

  

后半段图解如下

![](https://pic3.zhimg.com/80/v2-0f330a0d78a95497dc411d75c76fbc52_720w.jpg)

  

注意，如果把channel传递类型替换为Instance指针时，那么尽管channel存入到buf中的元素已经是拷贝对象了，从channel中取出又被拷贝了一次。但是由于它们的类型是Instance指针，拷贝对象与原始对象均会指向同一个内存地址，修改原有元素对象的数据时，会影响到取出数据。

```go
func main() {
    type Instance struct {
        ID   int
        name string
    }

    var ins = &Instance{ID: 1, name: "Golang"}

    ch := make(chan *Instance, 3)
    ch <- ins

    fmt.Println("ins的原始值：", ins)

    ins.name = "Python"
    go func(ch chan *Instance) {
        fmt.Println("channel接收值：", <-ch)
    }(ch)

    time.Sleep(time.Second)
    fmt.Println("ins的最终值：", ins)
}

// 输出结果
ins的原始值： &{1 Golang}
channel接收值： &{1 Python}
ins的最终值： &{1 Python}
```

因此，在使用channel时，尽量避免传递指针，如果传递指针，则需谨慎。

### 关闭

channel的关闭实现代码位于src/go/runtime/chan.go的chansend方法，详细执行逻辑已通过注释写明。

```go
func closechan(c *hchan) {
  // 如果hchan对象为nil，则会引发painc
    if c == nil {
        panic(plainError("close of nil channel"))
    }

  // 对hchan加锁
    lock(&c.lock)
  // 不同多次调用close(c chan<- Type)方法，否则会引发painc
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    if raceenabled {
        callerpc := getcallerpc()
        racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
        racerelease(c.raceaddr())
    }

  // close标志
    c.closed = 1

  // gList代表Go的GMP调度的G集合
    var glist gList

    // 该for循环是为了释放recvq上的所有等待接收sudog
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp)
    }

    // 该for循环会释放sendq上的所有等待发送sudog
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp)
    }
  // 释放sendq和recvq之后，hchan释放锁
    unlock(&c.lock)

  // 将上文中glist中的加入的goroutine取出，让它们均变为runnable（可执行）状态，等待调度器执行
    // 注意：我们上文中分析过，试图向一个已关闭的channel发送数据，会引发painc。
  // 所以，如果是释放sendq中的goroutine，它们一旦得到执行将会引发panic。
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```

关于关闭操作，有几个点需要注意一下。

-   如果关闭已关闭的channel会引发painc。
-   对channel关闭后，如果有阻塞的读取或发送goroutines将会被唤醒。读取goroutines会获取到hchan的已接收元素，如果没有，则获取到元素零值；发送goroutine的执行则会引发painc。

对于第二点，我们可以很好利用这一特性来实现对程序执行流的控制（类似于sync.WaitGroup的作用），以下是示例程序代码。

```go
func main() {
    ch := make(chan struct{})
    //
    go func() {
        // do something work...
        // when work has done, call close()
        close(ch)
    }()
    // waiting work done
    <- ch
    // other work continue...
}
```

## 四、总结

channel是Go中非常强大有用的机制，为了更有效地使用它，我们必须了解它的实现原理，这也是写作本文的目的。

-   hchan结构体有锁的保证，对于并发goroutine而言是安全的
-   channel接收、发送数据遵循FIFO（First In First Out）原语
-   channel的数据传递依赖于内存拷贝
-   channel能阻塞（gopark）、唤醒（goready）goroutine
-   所谓无缓存的channel，它的工作方式就是直接发送goroutine拷贝数据给接收goroutine，而不通过hchan.buf

另外，可以看到Go在channel的设计上权衡了简单与性能。为了简单性，hchan是有锁的结构，因为有锁的队列会更易理解和实现，但是这样会损失一些性能。考虑到整个 channel 操作带锁的成本较高，其实官方也曾考虑过使用无锁 channel 的设计，但是由于目前已有提案中（[https://github.com/golang/go/issues/8899](https://link.zhihu.com/?target=https%3A//github.com/golang/go/issues/8899)），无锁实现的channel可维护性差、且实际性能测试不具有说服力，而且也不符合Go的简单哲学，因此官方目前为止并没有采纳无锁设计。

在性能上，有一点，我们需要认识到：所谓channel中阻塞goroutine，只是在runtime系统中被blocked，它是用户层的阻塞。而实际的底层内核线程不受影响，它仍然是unblocked的。

## 参考链接
https://zhuanlan.zhihu.com/p/312041083
https://www.luozhiyun.com/archives/427
