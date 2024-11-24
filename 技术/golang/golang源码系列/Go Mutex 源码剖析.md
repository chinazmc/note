#golang 


上一篇文章[《一文完全掌握 Go math/rand》](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FdbdP_Jz9OHCuLO1ffOmCNg "https://mp.weixin.qq.com/s/dbdP_Jz9OHCuLO1ffOmCNg")，我们知道 math/rand 的 global rand 有一个全局锁，我的文章里面有一句话：“修复方案: 就是把 rrRand 换成了 globalRand, 在线上高并发场景下, 发现全局锁影响并不大.”， 有同学私聊我“他们遇到线上服务的锁竞争特别激烈”。确实我这句话说的并不严谨。但是也让我有了一个思考：到底多高的 QPS 才能让 Mutex 产生强烈的锁竞争 ？

到底加锁的代码会不会产生线上问题？ 到底该不该使用锁来实现这个功能？线上的问题是不是由于使用了锁造成的？针对这些问题，本文就从源码角度剖析 Go Mutex, 揭开 Mutex 的迷雾。

## 源码分析

Go mutex 源码只有短短的 228 行，但是却包含了很多的状态转变在里面，很不容易看懂，具体可以参见下面的流程图。Mutex 的实现主要借助了 CAS 指令 + 自旋 + 信号量来实现，具体代码我就不再每一行做分析了，有兴趣的可以根据下面流程图配合源码阅读一番。

### Lock

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3415d8d24334dddae482cf5d986f2f6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### Unlock

![Unlock](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f95dbc06af2a456095b7053cbb4c406d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## 一些例子

#### 1. 一个 goroutine 加锁解锁过程

![加锁加锁](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00f9606fb7c54584b9f1ba5d0195b47f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

#### 2. 没有加锁，直接解锁问题

![没有加锁直接解锁](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c8b952622354103af1d8578a892dedc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

#### 3. 两个 Goroutine，互相加锁解锁

![互相加锁解锁](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02d63030eb6444328aa8fa09584823d6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

#### 4. 三个 Goroutine 等待加锁过程

![三个 goroutine 等待加锁](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab2b3696e89e4fd9b8a70a9d43afa1fd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

整篇源码其实涉及比较难以理解的就是 Mutex 状态（mutexLocked，mutexWoken，mutexStarving，mutexWaiterShift） 与 Goroutine 之间的状态（starving，awoke）改变， 我们下面将逐一说明。

### 什么是 Goroutine 排队?

![排队](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8daefccd01704582ab7d99ba098b05b0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

如果 Mutex 已经被一个 Goroutine 获取了锁, 其它等待中的 Goroutine 们只能一直等待。那么等这个锁释放后，等待中的 Goroutine 中哪一个会优先获取 Mutex 呢?

正常情况下, 当一个 Goroutine 获取到锁后, 其他的 Goroutine 开始进入自旋转(为了持有CPU) 或者进入沉睡阻塞状态(等待信号量唤醒). 但是这里存在一个问题, 新请求的 Goroutine 进入自旋时是仍然拥有 CPU 的, 所以比等待信号量唤醒的 Goroutine 更容易获取锁. 用官方话说就是，新请求锁的 Goroutine具有优势，它正在CPU上执行，而且可能有好几个，所以刚刚唤醒的 Goroutine 有很大可能在锁竞争中失败.

于是如果一个 Goroutine 被唤醒过后, 仍然没有拿到锁, 那么该 Goroutine 会放在等待队列的最前面. 并且那些等待超过 1 ms 的 Goroutine 还没有获取到锁，该 Goroutine 就会进入饥饿状态。该 Goroutine 是饥饿状态并且 Mutex 是 Locked 状态时，才有可能给 Mutex 设置成饥饿状态.

获取到锁的 Goroutine Unlock, 将 Mutex 的 Locked 状态解除, 发出来解锁信号, 等待的 Goroutine 开始竞争该信号. 如果发现当前 Mutex 是饥饿状态, 直接将唤醒信号发给第一个等待的 Goroutine

这就是所谓的 Goroutine 排队

### 排队功能是如何实现的

我们知道在正常状态下，所有等待锁的 Goroutine 按照 FIFO 顺序等待，在 Mutex 饥饿状态下，会直接把释放锁信号发给等待队列中的第一个Goroutine。排队功能主要是通过 runtime_SemacquireMutex, runtime_Semrelease 来实现的.

#### 1. runtime_SemacquireMutex -- 入队

当 Mutex 被其他 Goroutine 持有时，新来的 Goroutine 将会被 runtime_SemacquireMutex 阻塞。阻塞会分为2种情况:

**Goroutine 第一次被阻塞：**

当 Goroutine 第一次尝试获取锁时，由于当前锁可能不能被锁定，于是有可能进入下面逻辑

```go
queueLifo := waitStartTime != 0
if waitStartTime == 0 {
    waitStartTime = runtime_nanotime()
}
runtime_SemacquireMutex(&m.sema, queueLifo, 1)

```

由于 waitStartTime 等于 0，runtime_SemacquireMutex 的 queueLifo 等于 false, 于是该 Goroutine 放入到队列的尾部。

**Goroutine 被唤醒过，但是没加锁成功，再次被阻塞** 由于 Goroutine 被唤醒过，waitStartTime 不等于 0，runtime_SemacquireMutex 的 queueLifo 等于 true, 于是该 Goroutine 放入到队列的头部。

#### 2. runtime_Semrelease -- 出队

当某个 Goroutine 释放锁时，调用 Unlock，这里同样存在两种情况：

**当前 mutex 不是饥饿状态**

```go
if new&mutexStarving == 0 {
    old := new
    for {
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
            return
        }
        // Grab the right to wake someone.
        new = (old - 1<<mutexWaiterShift) | mutexWoken
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime_Semrelease(&m.sema, false, 1)
            return
        }
        old = m.state
    }
}

```

Unlock 时 Mutex 的 Locked 状态被去掉。当发现当前 Mutex 不是饥饿状态，设置 runtime_Semrelease 的 handoff 参数是 false, 于是唤醒其中一个 Goroutine。

**当前 mutex 已经是饥饿状态**

```go
} else {
    // Starving mode: handoff mutex ownership to the next waiter, and yield
    // our time slice so that the next waiter can start to run immediately.
    // Note: mutexLocked is not set, the waiter will set it after wakeup.
    // But mutex is still considered locked if mutexStarving is set,
    // so new coming goroutines won't acquire it.
    runtime_Semrelease(&m.sema, true, 1)
}

```

同样 Unlock 时 Mutex 的 Locked 状态被去掉。由于当前 Mutex 是饥饿状态，于是设置 runtime_Semrelease 的 handoff 参数是 true, 于是让等待队列头部的第一个 Goroutine 获得锁。

### Goroutine 的排队 与 mutex 中记录的 Waiters 之间的关系?

通过上面的分析，我们知道 Goroutine 的排队是通过 runtime_SemacquireMutex 来实现的。Mutex.state 记录了目前通过 runtime_SemacquireMutex 排队的 Goroutine 的数量

### Goroutine 的饥饿与 Mutex 饥饿之间的关系？

Goroutine 的状态跟 Mutex 的是息息相关的。只有在 Goroutine 是饥饿状态下，才有可能给 Mutex 设置成饥饿状态。在 Mutex 是饥饿状态时，才有可能让饥饿的 Goroutine 优先获取到锁。不过需要注意的是，触发 Mutex 饥饿的 Goroutine 并不一定获取锁，有可能被其他的饥饿的 Goroutine 截胡。

### Goroutine 能够加锁成功的情况

Mutex 没有被 Goroutine 占用 Mutex.state = 0, 这种情况下一定能获取到锁. 例如: 第一个 Goroutine 获取到锁 还有一种情况 Goroutine有可能加锁成功:

1.  当前 Mutex 不是饥饿状态, 也不是 Locked 状态, 尝试 CAS 加锁时, Mutex 的值还没有被其他 Goroutine 改变, 当前 Goroutine 才能加锁成功.
2.  某个 Goroutine 刚好被唤醒后, 重新获取 Mutex, 这个时候 Mutex 处于饥饿状态. 因为这个时候只唤醒了饥饿的 Goroutine, 其他的 Goroutine 都在排队中, 没有其他 Goroutine 来竞争 Mutex, 所以能直接加锁成功

## Mutex 锁竞争的相关问题

### 探测锁竞争

日常开发中锁竞争的问题还是能经常遇到的，我们如何去发现锁竞争呢？其实还是需要靠 pprof 来人肉来分析。

《一次错误使用 go-cache 导致出现的线上问题》就是我真是遇到的一次线上问题，表象就是接口大量超时，打开pprof 发现大量 Goroutine 都集中 Lock 上。这个真实场景的具体的分析过程，有兴趣的可以阅读一下。 ![mutex 竞争](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/275e9bd9ed124885813ec12719693898~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp) 简单总结一下： 压测或者流量高的时候发现系统不正常，打开 pprof 发现 goroutine 指标在飙升，并且大量 Goroutine 都阻塞在 Mutex 的 Lock 上，这个基本就可以确定是锁竞争。

pprof 里面是有个 pprof/mutex 指标，不过该指标默认是关闭的，而且并没有太多资料有介绍这个指标如何来分析 Mutex。有知道这个指标怎么用的大佬，欢迎留言。

### mutex 锁的瓶颈

现在模拟业务开发中的某接口，平均耗时 10 ms, 在 32C 物理机上压测。CentOS Linux release 7.3.1611 (Core), go1.15.8  压测代码如下：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"

	_ "net/http/pprof"
)

var mux sync.Mutex

func testMutex(w http.ResponseWriter, r *http.Request) {
	mux.Lock()
	time.Sleep(10 * time.Millisecond)
	mux.Unlock()
}

func main() {
	go func() {
		log.Println(http.ListenAndServe(":6060", nil))
	}()

	http.HandleFunc("/test/mutex", testMutex)
	if err := http.ListenAndServe(":8000", nil); err != nil {
		fmt.Println("start http server fail:", err)
	}
}

```

![moni_mutex.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc33b994b9e741559c200eca50a54479~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![yaceresult.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90f4cee534664df88b25819f551d0322~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

这个例子写的比较极端了，全局共享一个 Mutex。经过压测发现在 100 qps 时，Mutex 没啥竞争，在 150 QPS 时竞争就开始变的激烈了。

当然我们写业务代码并不会这么写，但是可以通过这个例子发现 Mutex 在 QPS 很低的时候，锁竞争就会很激烈。需要说明的一点：这个压测是数值没啥具体的意义，不同的机器上表现肯定还会不一样。

这个例子告诉我们几点：

1.  写业务时不能全局使用同一个 Mutex
2.  尽量避免使用 Mutex，如果非使用不可，尽量多声明一些 Mutex，采用取模分片的方式去使用其中一个 Mutex

## 日常使用注意点

#### 1. Lock/Unlock 成对出现

我们日常开发中使用 Mutex 一定要记得：先 Lock 再 Unlock。

特别要注意的是：没有 Lock 就去 Unlock。当然这个 case 一般情况下我们都不会这么写。不过有些变种的写法我们要尤其注意，例如

```go
var mu sync.Mutex

func release() {
	mu.Lock()
    fmt.Println("lock1 success")
	time.Sleep(10 * time.Second)

	mu.Lock()
	fmt.Println("lock2 success")
}

func main() {
	go release()

	time.Sleep(time.Second)
	mu.Unlock()
	fmt.Println("unlock success")
	for {}
}

```

输出结果：

```go
release lock1 success
main unlock success
release lock2 success

```

我们看到 release goroutine 的锁竟然被 main goroutine 给释放了，同时 release goroutine 又能重新获取到锁。

这段代码可能你想不到有啥问题，其实这个问题蛮严重的，想象一下你的代码中，本来是要加锁给用户加积分的，但是竟然被别的 goroutine 给解锁了，导致积分没有增加成功，同时解锁的时候还别的 Goroutine 的锁给 Unlock 了，互相加锁解锁，导致莫名其妙的问题。

所以一般情况下，要在本 Goroutine 中完成 Mutex 的 Lock&Unlock，千万不要将要加锁和解锁分到两个 Goroutine 中进行。如果你确实需要这么做，请抽支烟冷静一下，你真的是否需要这么做。

#### 2. Mutex 千万不能被复制

我之前发过的[《当 Go struct 遇上 Mutex》](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FOYGVR0d-fq1hgOvrdsUnYA "https://mp.weixin.qq.com/s/OYGVR0d-fq1hgOvrdsUnYA")里面详细分析了不能被复制的原因，以及如何 Mutex 的最佳使用方式，建议没看过的同学去看一遍。我们还是举个例子说下为啥不能被复制，以及如何用源码进行分析

```go
type Person struct {
	mux sync.Mutex
}

func Reduce(p1 Person) {
	fmt.Println("step...", )
	p1.mux.Lock()
	fmt.Println(p1)
	defer p1.mux.Unlock()
	fmt.Println("over...")
}

func main() {
	var p Person
	p.mux.Lock()
	go Reduce(p)
	p.mux.Unlock()
	fmt.Println(111)
	for {}
}

```

问题分析：

1.  main Goroutine 已经给 p.mux 加了锁 , 这个时候 p.mux 的 state 的值是 mutexLocked。
2.  然后将 p.mux 复制给了 Reduce Goroutine。这个时候被复制的 p1.mux 的 state 的值也是 mutexLocked。
3.  main Goroutine 虽然已经解锁了, 但是 Reduce Goroutine 跟 main Goroutine 的 mutex 已经不是同一个 mutex 了, 所以 Reduce Goroutine 就会加锁失败, 产生死锁，关键是编译器还发现不了这个 Deadlock.

关于为什么编译器不能发现这个死锁，可以看我的博客[《一次 Golang Deadlock 的讨论》](https://link.juejin.cn/?target=https%3A%2F%2Fwww.haohongfan.com%2Fpost%2F2019-07-11-deadlock-detector%2F "https://www.haohongfan.com/post/2019-07-11-deadlock-detector/")

至此 Go Mutex 的源码剖析全部完毕了，有什么想跟我交流的可以再评论区留言。


## 一、什么是sync.Mutex

> sync.Mutex即为互斥锁，是传统的并发程序对共享资源进行访问控制的主要手段。只有两个公开方法 Lock 和 Unlock，顾名思义，前者被用于锁定当前的互斥量，而后者则用来对当前互斥量进行解锁。互斥锁的零值是未被锁定的互斥锁。接下来分析它的实现。

## Mutex结构体以及常量说明

```go
type Mutex struct {
	state int32
	sema  uint32
}
const (
	mutexLocked = 1 << iota  // 1 << 0 = 1
	mutexWoken  //  1 << 1 = 2
	mutexStarving // 1 << 2 = 4
	mutexWaiterShift = iota //3

	starvationThresholdNs = 1e6 
) 
```

### state

> 定义为int32 类型，允许为其调用原子方法来实现原子化的设定锁的状态 。一共32位，右起第一位表示锁定位，右起第二位表示唤醒位，右起第三位表示饥饿位。其余29位标识等待唤醒的goroutine的数量，下图为state结构图。

![](https://pic2.zhimg.com/80/v2-301027da84c3f21589c8d5f183bd1f21_720w.jpg)

### sema

> 信号量，用于唤醒或者挂起goroutine，具体用在四种信号处理函数那里。

### mutexLocked

> 值为 1，用mutex.state&mutexLocked的结果来判断mutex锁的状态，1表示已加锁，0表示未加锁

### mutexWoken

> 值为 2，用mutex.state&mutexWoken的结果判断mutex是否被唤醒，1被唤醒，0未被唤醒。

### mutexStarving

> 值为 4 ，用mutex.state&mutexStarving的结果判断mutex是否处于饥饿模式：1表示处于饥饿模式，0表示正常模式。

### mutexWaiterShift

> 值为 3，表示mutex.state >>3 即为阻塞的goroutine 的数量。

### starvationThresholdNs

> 值为 1e6。切换为饥饿模式的等待时间。

## 二、Mutex两种操作模式

### 正常模式

> 等待的goroutines按照FIFO（先进先出）顺序排队，但是goroutine被唤醒之后并不能立即得到mutex锁，它需要与新到达的goroutine争夺mutex锁。因为新到达的goroutine已经在CPU上运行了，所以被唤醒的goroutine很大概率是争夺mutex锁是失败的。出现这种情况的时候，被唤醒goroutine需要排队在队列的前面。如果被唤醒的goroutine有超过1ms没有获取到mutex锁，那么它就会变为饥饿模式。

### 饥饿模式

> 锁的所有权将从解锁的gorutine直接交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，而是放在等待队列的尾部。如果有一个等待的goroutine获取到mutex锁了，并且是等待队列中的最后一个goroutine或者它的等待时间不超过1ms。则会将mutex的模式切换回正常模式。  
> 正常模式：有更好的性能，因为goroutine可以获取到锁，即使有阻塞的goroutine，也要连续数次执行互斥锁。  
> 饥饿模式：是对正常模式的补充，防止等待队列中的goroutine永远没有机会获取锁。

## 三、四种信号处理函数

### runtime_canSpin

> 在src/runtime/proc.go中被实现sync_runtime_canSpin；表示比较保守的自旋，golang中自旋锁并不会一直自旋下去，在runtime包中runtime_canSpin方法做了一些限制,传递过来的iter大等于4或者cpu核数小等于1，最大逻辑处理器大于1，至少有个本地的P队列，并且本地的P队列可运行G队列为空。

### runtime_doSpin

> 在src/runtime/proc.go中被实现sync_runtime_doSpin；表示会调用procyield函数，该函数也是汇编语言实现。函数内部循环调用PAUSE指令。PAUSE指令什么都不做，但是会消耗CPU时间，在执行PAUSE指令时，CPU不会对它做不必要的优化。

### runtime_SemacquireMutex

> 在src/runtime/sema.go中被实现sync_runtime_SemacquireMutex，表示通过信号量阻塞当前协程，函数首先检查信号量是否为0：如果大于0，让信号量减1，返回；如果等于0，就调用goparkunlock函数，把当前Goroutine放入该sema的等待队列，并设为等待状态。

### runtime_Semrelease

> 在src/runtime/sema.go 中被实现 sync_runtime_Semrelease。首先让信号量加一，然后检查是否有正在等待的Goroutine，如果没有，直接返回；如果有，调用goready函数唤醒一个Goroutine。

## 四、源码实现

### 加锁源码分析

```go
func (m *Mutex) Lock() {
	// 快速路径，加锁
        // CAS比较state和0是否相等，如果相等则说明mutex锁未获得锁。
        // 此时将state赋值为1，表明加锁成功。
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
                // 竞态检测
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// 表明已经锁在使用中，则调用goroutine直到互斥锁可用为止
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64   // 标记当前goroutine的开始等待时间戳
	starving := false // 标记当前goroutine的饥饿模式，true:饥饿模式，false:正常模式
	awoke := false // 标记当前goroutine是否已唤醒，true:被唤醒，flase:未被唤醒
	iter := 0 // 自旋次数
	old := m.state // 保存当前对象锁状态赋值给old变量，做对比用 
	for {
                // mutexLocked:      0001   锁定状态
                // mutexWoken:       0010   唤醒状态
                // mutexStarving:    0100   饥饿模式
                // mutexWaiterShift: 3      等待上锁的goroutine数量

                // old&(mutexLocked|mutexStarving) == mutexLocked,于是old只能有以下两种状态：
                // 1) old = 0011时，说明mutex锁被锁定并且被唤醒，此时 0011&(0001|0100) = 0001
                // 2) old = 0001时，说明mutex锁被锁定，此时0001&(0001|0100) = 0001 
                // 间接说明了mutex不能处于饥饿模式下。
                
                // runtime_canSpin 见文末尾

                // 判断：被锁定状态；正常模式；可以自旋。（不要在饥饿模式下自旋）
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
                        // !awoke: 当前goroutine未被唤醒
                        // old&mutexWoken == 0：当前mutex锁是否被唤醒。
                        // 只有当old=0001时，才有0001&0010=0，说明当前mutex没有被唤醒
                        // old>>mutexWaiterShift != 0：查看当前mutex锁排队的goroution数量
                        // atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken)：state追加唤醒状态
			// 为mutex锁和当前goroutine追加唤醒状态
                        if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true //此时当前goroutine被唤醒
			}
			runtime_doSpin() //runtime_canSpin 见文末尾
			iter++ // 自旋次数+1
			old = m.state // 将old重新赋值
			continue //跳出本次循环，开始下次循环
		}
                // 不使用自旋锁或者自旋失败还是没有获得锁。
		new := old
                // old&mutexStarving == 0即 old&(0100)=0,满足条件的old只有以下两种
                // 1） 当old = 0011，0011&0100=0，即old处于被锁定，被唤醒状态。
                // 2） 当old = 0001，0011&0100=0，即old处于被锁定状态。
                // 也说明了只有在正常模式下，才能标记new为被锁定状态，准备抢占锁
		if old&mutexStarving == 0 {
			new |= mutexLocked 
		}
                // 当前mutex锁处于饥饿模式并且加锁状态
                // 新加入进来的goroutine放到队列后面，所以等待者数目+1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift 
		}
                // 当前goroutine处于饥饿模式，并且当前锁被占用,标记new变量为饥饿状态
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving 
		}
                // 当前goroutine被唤醒
		if awoke {
                        // 一定要将mutex标识为唤醒状态，不然panic
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
                // 将state与old比较，如果相同就将state赋值为new
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
                        // 只有mutex处于未加锁状态，正常模式下，才加锁成功。
			if old&(mutexLocked|mutexStarving) == 0 {
				break //使用cas方法成功抢占到锁。
			}
			// waitStartTime != 0 说明当前goroutine是等待状态唤醒的 ，
                        // 此时queueLifo为true，反之为false。
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
                                // 记录当前goroutine等待时间
				waitStartTime = runtime_nanotime()
			}
                        // 
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
                        // 当goroutine等待时间超过starvationThresholdNs，mutex进入饥饿模式
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state //得到mutex锁的状态
			if old&mutexStarving != 0 { //当前mutex处于饥饿模式
                // 如果当前的state已加锁，已唤醒，或者等待的队列中为空,那么state是一个非法状态，panic
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
                                // 等待状态的goroutine - 1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 如果本goroutine并不处于饥饿状态（等待时间小于1ms），或者它是最后一个等待者
                                if !starving || old>>mutexWaiterShift == 1 {
                                        // 退出饥饿模式
					delta -= mutexStarving
				}
                                // 设置新state, 因为已经获得了锁，退出、返回
				atomic.AddInt32(&m.state, delta)
				break
			}
                        // 修改为本goroutine为唤醒状态，并且自旋次数清0
			awoke = true
			iter = 0
		} else {
                        //如果CAS不成功，重新获取锁的state, 从for循环开始处重新开始 继续上述动作
			old = m.state
		}
	}
	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

![](https://pic3.zhimg.com/80/v2-5d859389391fdb85d7e36776ec237902_720w.jpg)

### 解锁源码分析

```go
func (m *Mutex) Unlock() {
        // 竞态检测
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}
	//  快速路径解锁
        //  原子操作移除state锁定标识。
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 { //存在其他goroutine等待
                // 慢路径解锁
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
        // 将state追加锁定标志，判断是否处于锁定状态
        // 未加锁状态下解锁会panic
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 { // 正常模式下
		old := new
		for {
                        // 没有等待唤醒的goroutine
                        // 或者加锁状态、被唤醒、饥饿状态下不需要唤醒等待的goroutine
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// 等待goroutine个数-1，并添加唤醒标识
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
                                // 唤醒一个阻塞的goroutine 但不是第一个等待者
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
                // 饥饿模式下
                // 将锁交给队列第一个等待的goroutine
                // 即使期间有新来的goroutine到来，只要处于饥饿模式 
                // 锁就不会被新来的goroutine抢占
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

![](https://pic2.zhimg.com/80/v2-5133b5a4c7a978e63508ec81c65e5371_720w.jpg)
---

## Go互斥锁设计

互斥锁是实现互斥功能的常见实现，Go中的互斥锁即`sync.Mutex`。本文将基于Go 1.15.2版本，对互斥锁的实现深入研究。

```go
type Mutex struct {
    state int32
    sema  uint32
}

const (
    mutexLocked = 1 << iota
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota   // mutexWaiterShift值为3，通过右移3位的位运算，可计算waiter个数
    starvationThresholdNs = 1e6 // 1ms，进入饥饿状态的等待时间
)
```

`state`字段表示当前互斥锁的状态信息，它是`int32`类型，其低三位的二进制位均有相应的状态含义。

-   `mutexLocked`是`state`中的低1位，用二进制表示为**0001**（为了方便，这里只描述后4位），它代表该互斥锁是否被加锁。
-   `mutexWoken`是低2位，用二进制表示为**0010**，它代表互斥锁上是否有被唤醒的goroutine。
-   `mutexStarving`是低3位，用二进制表示为**0100**，它代表当前互斥锁是否处于饥饿模式。
-   `state`剩下的29位用于统计在互斥锁上的等待队列中goroutine数目（`waiter`）。

默认的`state`字段（无锁状态）如下图所示。

![](https://pic4.zhimg.com/80/v2-b7336eba90ef35af6fb8dc4ecc5158b3_720w.jpg)

  

`sema`字段是信号量，用于控制goroutine的阻塞与唤醒，下文中会有介绍到。

### 两种模式

Go实现的互斥锁有两种模式，分别是**正常模式**和**饥饿模式**。

在正常模式下，waiter按照先进先出（**FIFO**）的方式获取锁，但是一个刚被唤醒的waiter与新到达的goroutine竞争锁时，大概率是干不过的。新来的goroutine有一个优势：它已经在CPU上运行，并且有可能不止一个新来的，因此waiter极有可能失败。在这种情况下，waiter还需要在等待队列中排队。为了避免waiter长时间抢不到锁，当waiter超过 1ms 没有获取到锁，它就会将当前互斥锁切换到饥饿模式，防止等待队列中的waiter被饿死。

在饥饿模式下，锁的所有权直接从解锁（unlocking）的goroutine转移到等待队列中的队头waiter。新来的goroutine不会尝试去获取锁，也不会自旋。它们将在等待队列的队尾排队。

如果某waiter获取到了锁，并且满足以下两个条件之一，它就会将锁从饥饿模式切换回正常模式。

-   它是等待队列的最后一个goroutine
-   它等待获取锁的时间小于1ms

饥饿模式是在 Go 1.9版本引入的，它防止了队列尾部waiter一直无法获取锁的问题。与饥饿模式相比，正常模式下的互斥锁性能更好。因为相较于将锁的所有权明确赋予给唤醒的waiter，直接竞争锁能降低整体goroutine获取锁的延时开销。

### 加锁

既然被称作锁，那就存在加锁和解锁的操作。`sync.Mutex`的加锁`Lock()`代码如下

```go
func (m *Mutex) Lock() {
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    m.lockSlow()
}
```

代码非常简洁，首先通过CAS判断当前锁的状态（CAS的原理和实现可以参照小菜刀写的《同步原语的基石》一文）。如果锁是完全空闲的，即`m.state`为0，则对其加锁，将`m.state`的值赋为1，此时加锁后的`state`如下

![](https://pic2.zhimg.com/80/v2-9aeafe40f05e99381d95101ab6f18c51_720w.jpg)

  

如果，当前锁已经被其他goroutine加锁，则进入`m.lockSlow()`逻辑。`lockSlow`函数比较长，这里我们分段阐述。

1.  **初始化**

```go
func (m *Mutex) lockSlow() {
    var waitStartTime int64  // 用于计算waiter的等待时间
    starving := false        // 饥饿模式标志
    awoke := false           // 唤醒标志
    iter := 0                // 统计当前goroutine的自旋次数
    old := m.state           // 保存当前锁的状态
    ...
}
```

第一段程序是做一些初始化状态、标志的动作。

1.  **自旋**

`lockSlow`函数余下的代码，就是一个大的`for`循环，首先看自旋部分。

```go
for { 
    // 判断是否能进入自旋
    if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
        // !awoke 判断当前goroutine是不是在唤醒状态
        // old&mutexWoken == 0 表示没有其他正在唤醒的goroutine
        // old>>mutexWaiterShift != 0 表示等待队列中有正在等待的goroutine
        if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
            // 尝试将当前锁的低2位的Woken状态位设置为1，表示已被唤醒
            // 这是为了通知在解锁Unlock()中不要再唤醒其他的waiter了
            atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
            awoke = true
        }
        // 自旋
        runtime_doSpin()
        iter++
        old = m.state
        continue
    }
    ...
}
```

关于自旋，这里需要简单阐述一下。自旋是自旋锁的行为，它通过忙等待，让线程在某段时间内一直保持执行，从而避免线程上下文的调度开销。**自旋锁对于线程只会阻塞很短时间的场景是非常合适的**。很显然，单核CPU是不适合使用自旋锁的，因为，在同一时间只有一个线程是处于运行状态，假设运行线程A发现无法获取锁，只能等待解锁，但因为A自身不挂起，所以那个持有锁的线程B没有办法进入运行状态，只能等到操作系统分给A的时间片用完，才能有机会被调度。这种情况下使用自旋锁的代价很高。

在本场景中，之所以想让当前goroutine进入自旋行为的依据是，我们乐观地认为：**当前正在持有锁的goroutine能在较短的时间内归还锁**。

`runtime_canSpin()`函数的实现如下

```go
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
func sync_runtime_canSpin(i int) bool {
  // active_spin = 4
    if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
        return false
    }
    if p := getg().m.p.ptr(); !runqempty(p) {
        return false
    }
    return true
}
```

由于自旋本身是空转CPU的，所以如果使用不当，反倒会降低程序运行性能。结合函数中的判断逻辑，这里总结出来goroutine能进入自旋的条件如下

-   当前互斥锁处于正常模式
-   当前运行的机器是多核CPU，且GOMAXPROCS>1
-   至少存在一个其他正在运行的处理器P，并且它的本地运行队列（local runq）为空
-   当前goroutine进行自旋的次数小于4

前面说到，自旋行为就是让当前goroutine并不挂起，占用cpu资源。我们看一下`runtime_doSpin()`的实现。

```go
//go:linkname sync_runtime_doSpin sync.runtime_doSpin
func sync_runtime_doSpin() {
    procyield(active_spin_cnt)  // active_spin_cnt = 30
}
```

`runtime_doSpin`调用了`procyield`，其实现如下（以amd64为例）

```text
TEXT runtime·procyield(SB),NOSPLIT,$0-0
    MOVL    cycles+0(FP), AX
again:
    PAUSE
    SUBL    $1, AX
    JNZ again
    RET
```

很明显，所谓的忙等待就是执行 30 次 `PAUSE` 指令，通过该指令占用 CPU 并消耗 CPU 时间。

1.  **计算期望状态**

前面说过，当前goroutine进入自旋是需要满足相应条件的。如果不满足自旋条件，则进入以下逻辑。

```go
// old是锁当前的状态，new是期望的状态，以期于在后面的CAS操作中更改锁的状态
    new := old
        if old&mutexStarving == 0 {
      // 如果当前锁不是饥饿模式，则将new的低1位的Locked状态位设置为1，表示加锁
            new |= mutexLocked
        }
        if old&(mutexLocked|mutexStarving) != 0 {
      // 如果当前锁已被加锁或者处于饥饿模式，则将waiter数加1，表示当前goroutine将被作为waiter置于等待队列队尾
            new += 1 << mutexWaiterShift
        }
        if starving && old&mutexLocked != 0 {
      // 如果当前锁处于饥饿模式，并且已被加锁，则将低3位的Starving状态位设置为1，表示饥饿
            new |= mutexStarving
        }
    // 当awoke为true，则表明当前goroutine在自旋逻辑中，成功修改锁的Woken状态位为1
        if awoke {
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
      // 将唤醒标志位Woken置回为0
      // 因为在后续的逻辑中，当前goroutine要么是拿到锁了，要么是被挂起。
      // 如果是挂起状态，那就需要等待其他释放锁的goroutine来唤醒。
      // 假如其他goroutine在unlock的时候发现Woken的位置不是0，则就不会去唤醒，那该goroutine就无法再醒来加锁。
            new &^= mutexWoken
        }
```

这里需要重点理解一下位操作**A |= B**，它的含义就是在B的二进制位为1的位，将A对应的二进制位设为1，如下图所示。因此，`new |= mutexLocked`的作用就是将`new`的最低一位设置为1。

![](https://pic4.zhimg.com/80/v2-23c8745f26242f1f10b5e9c7294df72f_720w.jpg)

  

1.  **更新期望状态**

在上一步，我们得到了锁的期望状态，接下来通过CAS将锁的状态进行更新。

```go
// 尝试将锁的状态更新为期望状态
    if atomic.CompareAndSwapInt32(&m.state, old, new) {
      // 如果锁的原状态既不是被获取状态，也不是处于饥饿模式
      // 那就直接返回，表示当前goroutine已获取到锁
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
      // 如果走到这里，那就证明当前goroutine没有获取到锁
      // 这里判断waitStartTime != 0就证明当前goroutine之前已经等待过了，则需要将其放置在等待队列队头
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
        // 如果之前没有等待过，就以现在的时间来初始化设置
                waitStartTime = runtime_nanotime()
            }
      // 阻塞等待
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
      // 被信号量唤醒之后检查当前goroutine是否应该表示为饥饿
      // （这里表示为饥饿之后，会在下一轮循环中尝试将锁的状态更改为饥饿模式）
      // 1. 如果当前goroutine已经饥饿（在上一次循环中更改了starving为true）
      // 2. 如果当前goroutine已经等待了1ms以上
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            // 再次获取锁状态
      old = m.state
      // 走到这里，如果此时锁仍然是饥饿模式
      // 因为在饥饿模式下，锁是直接交给唤醒的goroutine
      // 所以，即把锁交给当前goroutine
            if old&mutexStarving != 0 {
        // 如果当前锁既不是被获取也不是被唤醒状态，或者等待队列为空
        // 这代表锁状态产生了不一致的问题
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
        // 因为当前goroutine已经获取了锁，delta用于将等待队列-1
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
        // 如果当前goroutine中的starving标志不是饥饿
        // 或者当前goroutine已经是等待队列中的最后一个了
        // 就通过delta -= mutexStarving和atomic.AddInt32操作将锁的饥饿状态位设置为0，表示为正常模式
                if !starving || old>>mutexWaiterShift == 1 {
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta)
        // 拿到锁退出，业务逻辑处理完之后，需要调用Mutex.Unlock()方法释放锁
                break
            }
      // 如果锁不是饥饿状态
      // 因为当前goroutine已经被信号量唤醒了
      // 那就将表示当前goroutine状态的awoke设置为true
      // 并且将自旋次数的计数iter重置为0，如果能满足自旋条件，重新自旋等待
            awoke = true
            iter = 0
        } else {
      // 如果CAS未成功,更新锁状态，重新一个大循环
            old = m.state
        }
```

这里需要理解一下`runtime_SemacquireMutex(s *uint32, lifo bool, skipframes int)` 函数，**它是用于同步库的sleep原语**，它的实现是位于`src/runtime/sema.go`中的`semacquire1`函数，与它类似的还有`runtime_Semacquire(s *uint32)` 函数。两个睡眠原语需要等到 `*s>0` （本场景中 `m.sema>0` ），然后原子递减 `*s`。`SemacquireMutex`用于分析竞争的互斥对象，如果`lifo`（本场景中`queueLifo`）为true，则将等待者排在等待队列的队头。`skipframes`是从`SemacquireMutex`的调用方开始计数，表示在跟踪期间要忽略的帧数。

所以，运行到 `SemacquireMutex` 就证明当前goroutine在前面的过程中获取锁失败了，就需要sleep原语来阻塞当前goroutine，并通过信号量来排队获取锁：如果是新来的goroutine，就需要放在队尾；如果是被唤醒的等待锁的goroutine，就放在队头。

### 解锁

前面说过，有加锁就必然有解锁。我们来看解锁的过程

```go
func (m *Mutex) Unlock() {
    if race.Enabled {
        _ = m.state
        race.Release(unsafe.Pointer(m))
    }

  // new是解锁的期望状态
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        m.unlockSlow(new)
    }
}
```

通过原子操作`AddInt32`想将锁的低1位`Locked`状态位置为0。然后判断新的`m.state`值，如果值为0，则代表当前锁已经完全空闲了，结束解锁，否则进入`unlockSlow()`逻辑。

这里需要注意的是，锁空闲有两种情况，第一种是完全空闲，它的状态就是锁的初始状态。

![](https://pic4.zhimg.com/80/v2-b7336eba90ef35af6fb8dc4ecc5158b3_720w.jpg)

  

第二种空闲，是指的当前锁没被占有，但是会有等待拿锁的goroutine，只是还未被唤醒，例如以下状态的锁也是空闲的，它有两个等待拿锁的goroutine（未唤醒状态）。

![](https://pic3.zhimg.com/80/v2-c13fe387eaad723d74b63a884c955bda_720w.jpg)

  

以下是`unlockSlow`函数实现。

```go
func (m *Mutex) unlockSlow(new int32) {
  // 1. 如果Unlock了一个没有上锁的锁，则会发生panic。
   if (new+mutexLocked)&mutexLocked == 0 {
      throw("sync: unlock of unlocked mutex")
   }
  // 2. 正常模式
   if new&mutexStarving == 0 {
      old := new
      for {
        // 如果锁没有waiter,或者锁有其他以下已发生的情况之一，则后面的工作就不用做了，直接返回
        // 1. 锁处于锁定状态，表示锁已经被其他goroutine获取了
        // 2. 锁处于被唤醒状态，这表明有等待goroutine被唤醒，不用再尝试唤醒其他goroutine
        // 3. 锁处于饥饿模式，那么锁之后会被直接交给等待队列队头goroutine
         if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
            return
         }
        // 如果能走到这，那就是上面的if判断没通过
        // 说明当前锁是空闲状态，但是等待队列中有waiter，且没有goroutine被唤醒
        // 所以，这里我们想要把锁的状态设置为被唤醒，等待队列waiter数-1
         new = (old - 1<<mutexWaiterShift) | mutexWoken
        // 通过CAS操作尝试更改锁状态
         if atomic.CompareAndSwapInt32(&m.state, old, new) {
           // 通过信号量唤醒goroutine，然后退出
            runtime_Semrelease(&m.sema, false, 1)
            return
         }
        // 这里是CAS失败的逻辑
        // 因为在for循环中，锁的状态有可能已经被改变了，所以这里需要及时更新一下状态信息
        // 以便下个循环里作判断处理
         old = m.state
      }
   // 3. 饥饿模式
   } else {
     // 因为是饥饿模式，所以非常简单
     // 直接唤醒等待队列队头goroutine即可
      runtime_Semrelease(&m.sema, true, 1)
   }
}
```

在这里，需要理解一下`runtime_Semrelease(s *uint32, handoff bool, skipframes int)`函数。**它是用于同步库的wakeup原语**，`Semrelease`原子增加`*s`值（本场景中`m.sema`），并通知阻塞在`Semacquire`中正在等待的goroutine。如果`handoff`为真，则跳过计数，直接唤醒队头waiter。`skipframes`是从`Semrelease`的调用方开始计数，表示在跟踪期间要忽略的帧数。

## 总结

从代码量而言，go中互斥锁的代码非常轻量简洁，通过巧妙的位运算，仅仅采用state一个字段就实现了四个字段的效果，非常之精彩。

但是，代码量少并不代表逻辑简单，相反，它很复杂。互斥锁的设计中包含了大量的位运算，并包括了两种不同锁模式、信号量、自旋以及调度等内容，读者要真正理解加解锁的过程并不容易，这里再做一个简单回顾总结。

在正常模式下，waiter按照先进先出的方式获取锁；在饥饿模式下，锁的所有权直接从解锁的goroutine转移到等待队列中的队头waiter。

**模式切换**

如果当前 goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式。

如果当前 goroutine 是互斥锁最后一个waiter，或者等待的时间小于 1ms，互斥锁切换回正常模式。

**加锁**

1.  如果锁是完全空闲状态，则通过CAS直接加锁。  
    
2.  如果锁处于正常模式，则会尝试自旋，通过持有CPU等待锁的释放。  
    
3.  如果当前goroutine不再满足自旋条件，则会计算锁的期望状态，并尝试更新锁状态。  
    
4.  在更新锁状态成功后，会判断当前goroutine是否能获取到锁，能获取锁则直接退出。  
    
5.  当前goroutine不能获取到锁时，则会由sleep原语`SemacquireMutex`陷入睡眠，等待解锁的goroutine发出信号进行唤醒。
6.  唤醒之后的goroutine发现锁处于饥饿模式，则能直接拿到锁，否则重置自旋迭代次数并标记唤醒位，重新进入步骤2中。

**解锁**

1.  如果通过原子操作`AddInt32`后，锁变为完全空闲状态，则直接解锁。
2.  如果解锁一个没有上锁的锁，则直接抛出异常。
3.  如果锁处于正常模式，且没有goroutine等待锁释放，或者锁被其他goroutine设置为了锁定状态、唤醒状态、饥饿模式中的任一种（非空闲状态），则会直接退出；否则，会通过wakeup原语`Semrelease`唤醒waiter。
4.  如果锁处于饥饿模式，会直接将锁的所有权交给等待队列队头waiter，唤醒的waiter会负责设置`Locked`标志位。

另外，从Go的互斥锁带有自旋的设计而言，如果我们通过`sync.Mutex`只锁定执行耗时很低的关键代码，例如锁定某个变量的赋值，性能是非常不错的（因为等待锁的goroutine不用被挂起，持有锁的goroutine会很快释放锁）。**所以，我们在使用互斥锁时，应该只锁定真正的临界区**。


# Reference
https://juejin.cn/post/6943418870937944078
https://zhuanlan.zhihu.com/p/365552668
https://zhuanlan.zhihu.com/p/343563412

