其实已经有很多文章介绍 chan 的原理了，我就不详细描述了，从影响性能的角度来看，简单来说有两点：

- 如果 recv 协程空闲，则 send 协程会优先给 recv 协程，以提高效率；
- recv 和 send 协议都使用了`hchan.lock`对象，该对象是一个锁，也就是说它们之间会有竞争；

对于前者，给我们提供了一个思路，那就是由多个 recv 协程读取，这样可以提高放入的性能，但是具体设置多少呢？实际上业务并不好判断，因此并不是一种好的选择。

对于后者，我们先来说下这个 lock，这个 lock 不是`sync.Mutex`，`sync.Mutex`是 go 给我们开发者提供一个锁，这个锁的实现其实比较复杂，也有很多文章来说明，我这里就不详细说明了，但是这个锁有一个好处，它其实是一个**轻量级**的锁，这个轻量主要说的是这个锁是由 go 自身提供的，当出现锁切换时，它并不需要调用操作系统的 lock 来让渡出 cpu 资源，而仅仅是通过 gopark()的方式，将目前正在执行的 g 放回到 p 中，等待其他 m 来调度它，它的切换相对来说没有那么耗费资源。

而 hchan 中的 lock 是一个`runtime.mutex`。这个锁是一个互斥锁，在 linux 系统中它的实现是**futex**，在没有竞争的情况下，会退化成为一个自旋操作，速度非常快，但是**当竞争比较大时，它就会在内核中休眠**。注意，此处是在内核中休眠，而与`runtime.Mutex`是不同的，这也是为什么 chan 会这么慢的原因。

# Sync.Pool无锁ringbuffer队列+双向链表构建高性能缓存池

# Sync.Pool核心原理剖析

上篇文章主要是聊了下Pool的使用相关，这篇文章主要从源码角度剖析Pool如何表现的这么优秀，它背后的设计理念有哪些值得我们学习，那么这篇文章就相对很干了，言归正传开始正题。

干货：

1. ringbuffer-无锁竞争
2. 双向链表-动态扩容和P相互之间易窃取
3. victim cache-两轮GC保证和提升GC性能
4. 伪共享-cacheline内存优化
5. 缓存池与P绑定-尽量减少竞争和利用多核特性
6. 数据竞争-cas原子操作比Mutex性能好
7. 位操作-性能优化极致(pack,unpack,cas)，想想epoll事件位操作

## 1. Pool结构
```go
type Pool struct {
  //noCopy标明该pool不应该被复制，结合go vet可实现代码检测
 noCopy noCopy 

  //local+localSize决定poolLocal
 local     unsafe.Pointer 
 localSize uintptr        

  //victim+victimSize接管poolLocal负责GC相关
 victim     unsafe.Pointer 
 victimSize uintptr        

  //构造新对象方法
 New func() interface{}
}
```

poolLocal结构

```go
type poolLocal struct {
 poolLocalInternal
  //pad对齐 防止伪共享(false sharing) 后面单独介绍
 pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

poolLocalInternal结构
```go
type poolLocalInternal struct {
 private interface{}  //只能被owner P所使用.
 shared  poolChain //owner P 可以 pushHead/popHead操作; 其他P只允许popTail.
}
```

poolChain结构
```go
// poolChain的实现是一个双向队列,其中每个元素是前一个的双倍 
type poolChain struct {
  // head节点只能被producer访问，所以不需要同步机制 
  head *poolChainElt
  // tail节点被consumers访问，所以必须读写保证原子操作 
  tail *poolChainElt
}
```

poolChainElt结构
```go
type poolChainElt struct {
  poolDequeue
  next, prev *poolChainElt
}
```

poolDequeue结构
```go
//poolDequeue是一个无锁的固定大小单生产者，多消费者队列。 
type poolDequeue struct {
  headTail uint64 //存储head tail索引，通过mask和cas实现
  vals []eface //存储真正对象的数组
}
```

上面的pad属性介绍：

`pad 属性`  
，从这个属性的定义方式来看，明显是为了凑齐了 128 Byte 的整数倍。为什么会这么做呢？

这里是为了避免 CPU Cache 的 false sharing 问题：CPU Cache Line 通常是以 64 byte 或 128 byte 为单位。在我们的场景中，各个 P 的 poolLocal 是以数组形式存在一起。假设 CPU Cache Line 为 128 byte，而 poolLocal 不足 128 byte 时，那 cacheline 将会带上其他 P 的 poolLocal 的内存数据，以凑齐一整个 Cache Line。如果这时，两个相邻的 P 同时在两个不同的 CPU 核上运行，将会同时去覆盖刷新 CacheLine，造成 Cacheline 的反复失效，那 CPU Cache 将失去了作用。

CPU Cache 是距离 CPU 最近的 cache，如果能将其运用好，会极大提升程序性能。Golang 这里为了防止出现 false sharing 问题，主动使用 pad 的方式凑齐 128 个 byte 的整数倍，这样就不会和其他 P 的 poolLocal 共享一套 CacheLine。

因此pad属性是防止cpu 伪共享出现,提升程序性能的。

## 2. 结构图

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211207_7816cf9c-5701-11ec-8cf6-fa163eb4f6be.png)

图片左边部分介绍：
- Pool管理poolLocal，而poolLocal是长度为localSize(近似P的数量，通过runtime.GOMAXPROCS获取)的数组local的类型，这里面存储着各个P对应的本地对象池，可以看成是[P]poolLocal这种形式。
- poolLocal有两个属性，一个是private，一个是share。private比较简单，因为他是owner P特持有的，不和其他P共享，重点介绍share。
图片中间部分介绍：
- share是拥有poolChain这样结构的双向链表，有头尾指针，每个节点是一个poolDequeue结构。

图片最后一部分介绍：
- poolDequeue就是实现了ringbuffer结构的无锁队列，它是一个固定大小的单生产者，多消费者队列。
- 队列有头尾指针，不过因为底层是`vals []eface`  
    数组，因此消费和添加都是通过headTail索引来实现的，操作需要cas保证。
- head和tail之间的是可用对象，用绿色标识的部分。

为什么 poolChain 是这么一个双向链表 + ring buffer 的复杂结构呢？简单的每个链表项为单一元素不行吗？

1. 使用 ring buffer 是因为它有以下优点：
    `ring buffer的下面两个特性，非常适合于sync.Pool的应用场景`  
- `1. 预先分配好内存，且分配的内存项可不断复用` 。
- `2. 由于ring buffer 本质上是个数组，是连续内存结构，非常利于 CPU Cache。在访问poolDequeue 某一项时，其附近的数据项都有可能加载到统一 Cache Line 中，访问速度更快`  。
- 我们再注意看一个细节，poolDequeue 作为一个 ring buffer，自然需要记录下其 head 和 tail 的值。但在 poolDequeue 的定义中，head 和 tail 并不是独立的两个变量，只有一个 uint64 的 headTail 变量。
- 这是因为 headTail 变量将 head 和 tail 打包在了一起：其中高 32 位是 head 变量，低 32 位是 tail 变量。如下图所示：![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211207_782feb76-5701-11ec-8cf6-fa163eb4f6be.png)
    
    为什么会有这个复杂的打包操作呢？这个其实是个非常常见的`lock free(无锁)`  
    优化手段。
    
    对于一个 poolDequeue 来说，可能会被多个 P 同时访问（具体原因见下文 Get 函数中的对象窃取逻辑），这个时候就会带来并发问题。
    
    例如：当 ring buffer 空间仅剩一个的时候，即 head - tail = 1。如果多个 P 同时访问 ring buffer，在没有任何并发措施的情况下，两个 P 都可能会拿到对象，这肯定是不符合预期的。
    
    在不引入 Mutex 锁的前提下，sync.Pool 是怎么实现的呢？sync.Pool 利用了 atomic 包中的 CAS 操作。两个 P 都可能会拿到对象，但在最终设置 headTail 的时候，只会有一个 P 调用 CAS 成功，另外一个 CAS 失败。
```go
    ptrs2 := d.pack(head, tail+1)
    if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
       // Success.
       slot = &d.vals[tail&uint32(len(d.vals)-1)]
       break
  }




```


    在更新 head 和 tail 的时候，也是通过原子变量 + 位运算进行操作的。例如，当实现 head-- 的时候，需要通过以下代码实现：
```go
  head--
  ptrs2 := d.pack(head, tail)
  if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
     // We successfully took back slot.
     slot = &d.vals[head&uint32(len(d.vals)-1)]
     break
  }
```

- 使用双向链表是因为它有以下优点：
- 当前P的缓存池为空的时候，那么将尝试从其他P的缓存池中窃取对象。窃取的时候是从别的P的缓存池中尾部开始取的。
- 当前P的缓存池不为空就从自己的缓存池中头部开始取的。
- 所以一个是从后向前遍历，另一个是从前向后遍历，因此只有双向链表适合这种形式
- 如果长度为8的poolDequeue装满了，可以申请2倍大小的poolDequeue，让head指向它，这种动态扩容的方式链表很合适啊。
- 还有就是P如果是自己pushHead/popHead它自己的缓存池是不用加锁的，这样效率更高。

## 3. Put/Get源码

Get
```go
func (p *Pool) Get() interface{} {
  // 获取当前P的对象池和Pid
 l, pid := p.pin()
 x := l.private
 l.private = nil
 if x == nil {
  //尝试从本地share队列头部pop
    x, _ = l.shared.popHead()
    if x == nil {
        //如果空,尝试从其他P的share队列尾部pop 窃取
       x = p.getSlow(pid)
     }
   }
    
  // 解绑groutine和P
 runtime_procUnpin()
  
  //如果还没有获取到对象，那么尝试构造新对象
 if x == nil && p.New != nil {
   x = p.New()
 }
  
 return x
}


func (p *Pool) getSlow(pid int) interface{} {
 size := runtime_LoadAcquintptr(&p.localSize)
 locals := p.local    
  
  // Try to steal one element from other procs.
  //尝试从其他P偷取一个对象
 for i := 0; i < int(size); i++ {
   l := indexLocal(locals, (pid+i+1)%int(size))
   if x, _ := l.shared.popTail(); x != nil {
      return x
  }
 }

 //尝试从victim cache中获取。因为我们尽可能想淘汰victim中的对象
  // 所以在尝试从所有的primary cache中获取失败后操作
 size = atomic.LoadUintptr(&p.victimSize)
 if uintptr(pid) >= size {
   return nil
 }
  
 locals = p.victim
 l := indexLocal(locals, pid)
 if x := l.private; x != nil {
   l.private = nil
   return x
 }
  
  //从victim cache中窃取一个对象
 for i := 0; i < int(size); i++ {
   l := indexLocal(locals, (pid+i)%int(size))
    if x, _ := l.shared.popTail(); x != nil {
      return x
   }
 }
  
 //Mark the victim cache as empty
 //标记victim cache为空 即清理victim cache
 atomic.StoreUintptr(&p.victimSize, 0)

 return nil
}
```

Put
```go
func (p *Pool) Put(x interface{}) {
 if x == nil {
  return
 }

 //返回当前groutine所绑定的p,并且固定当前P和groutine的绑定
 l, _ := p.pin()
 //首先给private 
 if l.private == nil {
  l.private = x
  x = nil
 }
  
 //private不为空，就插入到当前head指向的双向链表节点的ringbuffer的head位置，并更新head索引。
 if x != nil {
  l.shared.pushHead(x)
 }
  
 //解绑P和当前groutine的固定关系
 runtime_procUnpin()
}
```

## 4. GC清理
```go
func poolCleanup() {
 // 函数将在gc开始前被调用.
 // 在gc stop word阶段,pool将不会产生外部函数调用分配对象的行为. 
 // 从oldPools中删除所有的victim cache.
 for _, p := range oldPools {
   p.victim = nil
   p.victimSize = 0
 }

 // 移动local cache到victim cache.
 //victim接管local
 for _, p := range allPools {
   p.victim = p.local
   p.victimSize = p.localSize
   p.local = nil
   p.localSize = 0
 }

 // 现在所有的local cache被移动到了victim cache中，local cache为空
 oldPools, allPools = allPools, nil
}
```

经历两轮GC，第一轮GC让victim cache接管local cache；第二轮GC就是全部清除victim cache，为什么这么设计？因为假如第一轮就全部清除，那么之后再获取对象的时候性能就会下降，因为很有可能在大量创建对象了，所以它GC第一轮先放到victim cache中，等到第二轮在处理。当然如果经历了第一轮GC之后又从victim cache中获取了一些对象，那么在第二轮的时候就不会删除，想想LRU的机制。。。victim的设计也间接提升了GC的性能，因为相对来说Sync.Pool池化的对象是long-live的，而GC的主要对象是short-live的，所以会减少GC的执行执行。

## 5. 小结

此篇文章主要是介绍了Sync.Pool核心原理，因为涉及到的源码比较多，需要大家好好消化。还有就是从源码中学到了很多NB的技术和思想，这种探索非常值得，内卷不是说我看了很多源码就不了了之，更重要的是从其中真正学到了什么，如果学不到我还研究它有何意义？只是单纯为了内卷？

# Reference
https://www.modb.pro/db/190715