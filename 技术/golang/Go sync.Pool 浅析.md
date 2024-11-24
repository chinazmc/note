#golang 

sync.Pool 应该是 Go 里面明星级别的数据结构，有很多优秀的文章都在介绍这个结构，本篇文章简单剖析下 sync.Pool。不过说实话 sync.Pool 并不是我们日常开发中使用频率很高的的并发原语。

尽管用的频率很低，但是不可否认的是 sync.Pool 确实是 Go 的杀手锏，合理使用 sync.Pool 会让我们的程序性能飙升。本篇文章会从使用方式，源码剖析，运用场景等方面，让你对 sync.Pool 有一个清晰的认知。

## 使用方式

sync.Pool 使用很简单，但是想用对却很麻烦，因为你有可能看到网上一堆错误的示例，各位同学在搜索 sync.Pool 的使用例子时，要特别注意。

sync.Pool 是一个内存池。通常内存池是用来防止内存泄露的（例如C/C++)。sync.Pool 这个内存池却不是干这个的，带 GC 功能的语言都存在垃圾回收 STW 问题，需要回收的内存块越多，STW 持续时间就越长。如果能让 new 出来的变量，一直不被回收，得到重复利用，是不是就减轻了 GC 的压力。

正确的使用示例（下面的demo选自gin）

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := engine.pool.Get().(*Context)
    c.writermem.reset(w)
    c.Request = req
    c.reset()

    engine.handleHTTPRequest(c)

    engine.pool.Put(c)
}

```

一定要注意的是：是先 Get 获取内存空间，基于这个内存做相关的处理，然后再将这个内存还回（Put）到 sync.Pool。

### Pool 结构

![sync.Pool 全景图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/569778a65faf42f2a48b2da2d454430a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## 源码图解

![Pool.Get](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb91ec6a3db04a2da7f6236a1647fc1a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![Pool.Put](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7e321ae5b3846e6980cb125ebb8b479~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

简单点可以总结成下面的流程：

![Pool.Get 流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/414c269b376f41279cd9cdcb1bae9760~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![Pool.Put流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/534ea3bb82ed4b51afc67f165d5c09da~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![Pool GC 流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bbc3455e45f498cb8755e7af577ac64~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## Sync.Pool 梳理

### Pool 的内容会清理？清理会造成数据丢失吗？

sync.Pool 会在每个 GC 周期内定期清理 sync.Pool 内的数据。定时清理数据并不会造成数据丢失。

要分几个方面来说这个问题。

1.  已经从 sync.Pool Get 的值，在 poolClean 时虽说将 pool.local 置成了nil，Get 到的值依然是有效的，是被 GC 标记为黑色的，不会被 GC回收，当 Put 后又重新加入到 sync.Pool 中
2.  在第一个 GC 周期内 Put 到 sync.Pool 的数值，在第二个 GC 周期没有被 Get 使用，就会被放在 local.victim 中。如果在 第三个 GC 周期仍然没有被使用就会被 GC 回收。

### runtime.GOMAXPROCS 与 pool 之间的关系？

```go
s := p.localSize
l := p.local
if uintptr(pid) < s {
    return indexLocal(l, pid), pid
}

if p.local == nil {
    allPools = append(allPools, p)
}
// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
size := runtime.GOMAXPROCS(0)
local := make([]poolLocal, size)
atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
runtime_StoreReluintptr(&p.localSize, uintptr(size))     // store-release

```

runtime.GOMAXPROCS(0) 是获取当前最大的 p 的数量。sync.Pool 的 poolLocal 数量受 p 的数量影响，会开辟 runtime.GOMAXPROCS(0) 个 poolLocal。某些场景下我们会使用 runtime.GOMAXPROCS（N) 来改变 p 的数量，会使 sync.Pool 的 pool.poolLocal 释放重新开辟新的空间。

为什么要开辟 runtime.GOMAXPROCS 个 local？

pool.local 是个 poolLocal 结构，这个结构体是 private + shared链表组成，在多 goroutine 的 Get/Put 下是有数据竞争的，如果只有一个 local 就需要加锁来操作。每个 p 的 local 就能减少加锁造成的数据竞争问题。

### New() 的作用？假如没有 New 会出现什么情况？

从上面的 pool.Get 流程图可以看出来，从 sync.Pool 获取一个内存会尝试从当前 private，shared，其他的 p 的 shared 获取或者 victim 获取，如果实在获取不到时，才会调用 New 函数来获取。也就是 New() 函数才是真正开辟内存空间的。New() 开辟出来的的内存空间使用完毕后，调用 pool.Put 函数放入到 sync.Pool 中被重复利用。

如果 New 函数没有被初始化会怎样呢？很明显，sync.Pool 就废掉了，因为没有了初始化内存的地方了。

### 先 Put，再 Get 会出现什么情况？

**一定要注意，下面这个例子的用法是错误的**

```go
func main(){
    pool:= sync.Pool{
        New: func() interface{} {
            return item{}
        },
    }
    pool.Put(item{value:1})
    data := pool.Get()
    fmt.Println(data)
}

```

如果你直接跑这个例子，能得到你想像的结果，但是在某些情况下就不是这个结果了。

在 Pool.Get 注释里面有这么一句话：“Callers should not assume any relation between values passed to Put and the values returned by Get.”，告诉我们不能把值 Pool.Put 到 sync.Pool 中，再使用 Pool.Get 取出来，因为 sync.Pool 不是 map 或者 slice，放入的值是有可能拿不到的，sync.Pool 的数据结构就不支持做这个事情。

前面说使用 sync.Pool 容易被错误示例误导，就是上面这个写法。为什么 Put 的值 再 Get 会出现问题？

-   情况1：sync.Pool 的 poolCleanup 函数在系统 GC 时会被调用，Put 到 sync.Pool 的值，由于有可能一直得不到利用，被在某个 GC 周期内就有可能被释放掉了。
-   情况2：不同的 goroutine 绑定的 p 有可能是不一样的，当前 p 对应的 goroutine 放入到 sync.Pool 的值有可能被其他的 p 对应的 goroutine 取到，导致当前 goroutine 再也取不到这个值。
-   情况3：使用 runtime.GOMAXPROCS（N) 来改变 p 的数量，会使 sync.Pool 的 pool.poolLocal 释放重新开辟新的空间，导致 sync.Pool 被释放掉。
-   情况4：还有很多情况

### 只 Get 不 Put 会内存泄露吗？

使用其他的池，如连接池，如果取连接使用后不放回连接池，就会出现连接池泄露，**是不是 sync.Pool 也有这个问题呢？**

通过上面的流程图，可以看出来 Pool.Get 的时候会尝试从当前 private，shared，其他的 p 的 shared 获取或者 victim 获取，如果实在获取不到时，才会调用 New 函数来获取，New 出来的内容本身还是受系统 GC 来控制的。所以如果我们提供的 New 实现不存在内存泄露的话，那么 sync.Pool 是不会内存泄露的。当 New 出来的变量如果不再被使用，就会被系统 GC 给回收掉。

如果不 Put 回 sync.Pool，会造成 Get 的时候每次都调用的 New 来从堆栈申请空间，达不到减轻 GC 压力。

## 使用场景

上面说到 sync.Pool 业务开发中不是一个常用结构，我们业务开发中没必要假想某块代码会有强烈的性能问题，一上来就用 sync.Pool 硬怼。 sync.Pool 主要是为了解决 Go GC 压力过大问题的，所以一般情况下，当线上高并发业务出现 GC 问题需要被优化时，才需要用 sync.Pool 出场。

## 使用注意点

1.  sync.Pool 同样不能被复制。
2.  好的使用习惯，从 pool.Get 出来的值进行数据的清空（reset），防止垃圾数据污染。

> 本文基于的 Go 源码版本：1.16.2



最近在工作中碰到了 GC 的问题：项目中大量重复地创建许多对象，造成 GC 的工作量巨大，CPU 频繁掉底。准备使用 `sync.Pool` 来缓存对象，减轻 GC 的消耗。为了用起来更顺畅，我特地研究了一番，形成此文。本文从使用到源码解析，循序渐进，一一道来。

> 本文基于 Go 1.14

# 是什么

`sync.Pool` 是 sync 包下的一个组件，可以作为保存临时取还对象的一个“池子”。个人觉得它的名字有一定的误导性，因为 Pool 里装的对象可以被无通知地被回收，可能 `sync.Cache` 是一个更合适的名字。

# 有什么用

对于很多需要重复分配、回收内存的地方，`sync.Pool` 是一个很好的选择。频繁地分配、回收内存会给 GC 带来一定的负担，严重的时候会引起 CPU 的毛刺，而 `sync.Pool` 可以将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力，提升系统的性能。

# 怎么用

首先，`sync.Pool` 是协程安全的，这对于使用者来说是极其方便的。使用前，设置好对象的 `New` 函数，用于在 `Pool` 里没有缓存的对象时，创建一个。之后，在程序的任何地方、任何时候仅通过 `Get()`、`Put()` 方法就可以取、还对象了。

下面是 2018 年的时候，《Go 夜读》上关于 `sync.Pool` 的分享，关于适用场景：

> 当多个 goroutine 都需要创建同⼀个对象的时候，如果 goroutine 数过多，导致对象的创建数⽬剧增，进⽽导致 GC 压⼒增大。形成 “并发⼤－占⽤内存⼤－GC 缓慢－处理并发能⼒降低－并发更⼤”这样的恶性循环。

> 在这个时候，需要有⼀个对象池，每个 goroutine 不再⾃⼰单独创建对象，⽽是从对象池中获取出⼀个对象（如果池中已经有的话）。

因此关键思想就是对象的复用，避免重复创建、销毁，下面我们来看看如何使用。

## 简单的例子

首先来看一个简单的例子：

```golang
package main
import (
	"fmt"
	"sync"
)

var pool *sync.Pool

type Person struct {
	Name string
}

func initPool() {
	pool = &sync.Pool {
		New: func()interface{} {
			fmt.Println("Creating a new Person")
			return new(Person)
		},
	}
}

func main() {
	initPool()

	p := pool.Get().(*Person)
	fmt.Println("首次从 pool 里获取：", p)

	p.Name = "first"
	fmt.Printf("设置 p.Name = %s\n", p.Name)

	pool.Put(p)

	fmt.Println("Pool 里已有一个对象：&{first}，调用 Get: ", pool.Get().(*Person))
	fmt.Println("Pool 没有对象了，调用 Get: ", pool.Get().(*Person))
}
```

运行结果：

```golang
Creating a new Person
首次从 pool 里获取： &{}
设置 p.Name = first
Pool 里已有一个对象：&{first}，Get:  &{first}
Creating a new Person
Pool 没有对象了，Get:  &{}
```

首先，需要初始化 `Pool`，唯一需要的就是设置好 `New` 函数。当调用 Get 方法时，如果池子里缓存了对象，就直接返回缓存的对象。如果没有存货，则调用 New 函数创建一个新的对象。

另外，我们发现 Get 方法取出来的对象和上次 Put 进去的对象实际上是同一个，Pool 没有做任何“清空”的处理。但我们不应当对此有任何假设，因为在实际的并发使用场景中，无法保证这种顺序，最好的做法是在 Put 前，将对象清空。

## fmt 包如何用

这部分主要看 `fmt.Printf` 如何使用：

```golang
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

继续看 `Fprintf`：

```golang
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

`Fprintf` 函数的参数是一个 `io.Writer`，`Printf` 传的是 `os.Stdout`，相当于直接输出到标准输出。这里的 `newPrinter` 用的就是 Pool：

```golang
// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
	p := ppFree.Get().(*pp)
	p.panicking = false
	p.erroring = false
	p.wrapErrs = false
	p.fmt.init(&p.buf)
	return p
}

var ppFree = sync.Pool{
	New: func() interface{} { return new(pp) },
}
```

回到 `Fprintf` 函数，拿到 pp 指针后，会做一些 format 的操作，并且将 p.buf 里面的内容写入 w。最后，调用 free 函数，将 pp 指针归还到 Pool 中：

```golang
// free saves used pp structs in ppFree; avoids an allocation per invocation.
func (p *pp) free() {
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}
```

归还到 Pool 前将对象的一些字段清零，这样，通过 Get 拿到缓存的对象时，就可以安全地使用了。

## pool_test

通过 test 文件学习源码是一个很好的途径，因为它代表了“官方”的用法。更重要的是，测试用例会故意测试一些“坑”，学习这些坑，也会让自己在使用的时候就能学会避免。

`pool_test` [文件](https://github.com/golang/go/blob/release-branch.go1.14/src/sync/pool_test.go)里共有 7 个测试，4 个 BechMark。

`TestPool` 和 `TestPoolNew` 比较简单，主要是测试 Get/Put 的功能。我们来看下 `TestPoolNew`：

```golang
func TestPoolNew(t *testing.T) {
	// disable GC so we can control when it happens.
	defer debug.SetGCPercent(debug.SetGCPercent(-1))

	i := 0
	p := Pool{
		New: func() interface{} {
			i++
			return i
		},
	}
	if v := p.Get(); v != 1 {
		t.Fatalf("got %v; want 1", v)
	}
	if v := p.Get(); v != 2 {
		t.Fatalf("got %v; want 2", v)
	}

	// Make sure that the goroutine doesn't migrate to another P
	// between Put and Get calls.
	Runtime_procPin()
	p.Put(42)
	if v := p.Get(); v != 42 {
		t.Fatalf("got %v; want 42", v)
	}
	Runtime_procUnpin()

	if v := p.Get(); v != 3 {
		t.Fatalf("got %v; want 3", v)
	}
}
```

首先设置了 `GC=-1`，作用就是停止 GC。那为啥要用 defer？函数都跑完了，还要 defer 干啥。注意到，`debug.SetGCPercent` 这个函数被调用了两次，而且这个函数返回的是上一次 GC 的值。因此，defer 在这里的用途是还原到调用此函数之前的 GC 设置，也就是恢复现场。

接着，调置了 Pool 的 New 函数：直接返回一个 int，变且每次调用 New，都会自增 1。然后，连续调用了两次 Get 函数，因为这个时候 Pool 里没有缓存的对象，因此每次都会调用 New 创建一个，所以第一次返回 1，第二次返回 2。

然后，调用 `Runtime_procPin()` 防止 goroutine 被强占，目的是保护接下来的一次 Put 和 Get 操作，使得它们操作的对象都是同一个 P 的“池子”。并且，这次调用 Get 的时候并没有调用 New，因为之前有一次 Put 的操作。

最后，再次调用 Get 操作，因为没有“存货”，因此还是会再次调用 New 创建一个对象。

`TestPoolGC` 和 `TestPoolRelease` 则主要测试 GC 对 Pool 里对象的影响。这里用了一个函数，用于计数有多少对象会被 GC 回收：

```golang
runtime.SetFinalizer(v, func(vv *string) {
	atomic.AddUint32(&fin, 1)
})
```

当垃圾回收检测到 `v` 是一个不可达的对象时，并且 `v` 又有一个关联的 `Finalizer`，就会另起一个 goroutine 调用设置的 finalizer 函数，也就是上面代码里的参数 func。这样，就会让对象 v 重新可达，从而在这次 GC 过程中不被回收。之后，解绑对象 v 和它所关联的 `Finalizer`，当下次 GC 再次检测到对象 v 不可达时，才会被回收。

`TestPoolStress` 从名字看，主要是想测一下“压力”，具体操作就是起了 10 个 goroutine 不断地向 Pool 里 Put 对象，然后又 Get 对象，看是否会出错。

`TestPoolDequeue` 和 `TestPoolChain`，都调用了 `testPoolDequeue`，这是具体干活的。它需要传入一个 `PoolDequeue` 接口：

```golang
// poolDequeue testing.
type PoolDequeue interface {
	PushHead(val interface{}) bool
	PopHead() (interface{}, bool)
	PopTail() (interface{}, bool)
}
```

`PoolDequeue` 是一个双端队列，可以从头部入队元素，从头部和尾部出队元素。调用函数时，前者传入 `NewPoolDequeue(16)`，后者传入 `NewPoolChain()`，底层其实都是 `poolDequeue` 这个结构体。具体来看 `testPoolDequeue` 做了什么：

![双端队列](https://cdn.jsdelivr.net/gh/qcrao/images/blog/20200410125923.png)

总共起了 10 个 goroutine：1 个生产者，9 个消费者。生产者不断地从队列头 pushHead 元素到双端队列里去，并且每 push 10 次，就 popHead 一次；消费者则一直从队列尾取元素。不论是从队列头还是从队列尾取元素，都会在 map 里做标记，最后检验每个元素是不是只被取出过一次。

剩下的就是 Benchmark 测试了。第一个 `BenchmarkPool` 比较简单，就是不停地 Put/Get，测试性能。

`BenchmarkPoolSTW` 函数会先关掉 GC，再向 pool 里 put 10 个对象，然后强制触发 GC，记录 GC 的停顿时间，并且做一个排序，计算 P50 和 P95 的 STW 时间。这个函数可以加入个人的代码库了：

```golang
func BenchmarkPoolSTW(b *testing.B) {
	// Take control of GC.
	defer debug.SetGCPercent(debug.SetGCPercent(-1))

	var mstats runtime.MemStats
	var pauses []uint64

	var p Pool
	for i := 0; i < b.N; i++ {
		// Put a large number of items into a pool.
		const N = 100000
		var item interface{} = 42
		for i := 0; i < N; i++ {
			p.Put(item)
		}
		// Do a GC.
		runtime.GC()
		// Record pause time.
		runtime.ReadMemStats(&mstats)
		pauses = append(pauses, mstats.PauseNs[(mstats.NumGC+255)%256])
	}

	// Get pause time stats.
	sort.Slice(pauses, func(i, j int) bool { return pauses[i] < pauses[j] })
	var total uint64
	for _, ns := range pauses {
		total += ns
	}
	// ns/op for this benchmark is average STW time.
	b.ReportMetric(float64(total)/float64(b.N), "ns/op")
	b.ReportMetric(float64(pauses[len(pauses)*95/100]), "p95-ns/STW")
	b.ReportMetric(float64(pauses[len(pauses)*50/100]), "p50-ns/STW")
}
```

我在 mac 上跑了一下：

```shell
go test -v -run=none -bench=BenchmarkPoolSTW
```

得到输出：

```shell
goos: darwin
goarch: amd64
pkg: sync
BenchmarkPoolSTW-12    361    3708 ns/op    3583 p50-ns/STW    5008 p95-ns/STW
PASS
ok      sync    1.481s
```

最后一个 `BenchmarkPoolExpensiveNew` 测试当 New 的代价很高时，Pool 的表现。也可以加入个人的代码库。

## 其他

标准库中 `encoding/json` 也用到了 sync.Pool 来提升性能。著名的 `gin` 框架，对 context 取用也到了 `sync.Pool`。

来看下 `gin` 如何使用 sync.Pool。设置 New 函数：

```golang
engine.pool.New = func() interface{} {
	return engine.allocateContext()
}

func (engine *Engine) allocateContext() *Context {
	return &Context{engine: engine, KeysMutex: &sync.RWMutex{}}
}
```

使用：

```golang
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```

先调用 Get 取出来缓存的对象，然后会做一些 reset 操作，再执行 `handleHTTPRequest`，最后再 Put 回 Pool。

另外，Echo 框架也使⽤了 `sync.Pool` 来管理 `context`，并且⼏乎达到了零堆内存分配：

> It leverages sync pool to reuse memory and achieve zero dynamic memory allocation with no GC overhead.

# 源码分析

## Pool 结构体

首先来看 Pool 的结构体：

```golang
type Pool struct {
	noCopy noCopy

    // 每个 P 的本地队列，实际类型为 [P]poolLocal
	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	// [P]poolLocal的大小
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// 自定义的对象创建回调函数，当 pool 中无可用对象时会调用此函数
	New func() interface{}
}
```

因为 Pool 不希望被复制，所以结构体里有一个 noCopy 的字段，使用 `go vet` 工具可以检测到用户代码是否复制了 Pool。

> `noCopy` 是 go1.7 开始引入的一个静态检查机制。它不仅仅工作在运行时或标准库，同时也对用户代码有效。

> 用户只需实现这样的不消耗内存、仅用于静态分析的结构，来保证一个对象在第一次使用后不会发生复制。

实现非常简单：

```golang
// noCopy 用于嵌入一个结构体中来保证其第一次使用后不会被复制
//
// 见 https://golang.org/issues/8005#issuecomment-190753527
type noCopy struct{}

// Lock 是一个空操作用来给 `go ve` 的 -copylocks 静态分析
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

`local` 字段存储指向 `[P]poolLocal` 数组（严格来说，它是一个切片）的指针，`localSize` 则表示 local 数组的大小。访问时，P 的 id 对应 `[P]poolLocal` 下标索引。通过这样的设计，多个 goroutine 使用同一个 Pool 时，减少了竞争，提升了性能。

在一轮 GC 到来时，victim 和 victimSize 会分别“接管” local 和 localSize。`victim` 的机制用于减少 GC 后冷启动导致的性能抖动，让分配对象更平滑。

> Victim Cache 本来是计算机架构里面的一个概念，是 CPU 硬件处理缓存的一种技术，`sync.Pool` 引入的意图在于降低 GC 压力的同时提高命中率。

当 Pool 没有缓存的对象时，调用 `New` 方法生成一个新的对象。

```golang
type poolLocal struct {
	poolLocalInternal

	// 将 poolLocal 补齐至两个缓存行的倍数，防止 false sharing,
	// 每个缓存行具有 64 bytes，即 512 bit
	// 目前我们的处理器一般拥有 32 * 1024 / 64 = 512 条缓存行
	// 伪共享，仅占位用，防止在 cache line 上分配多个 poolLocalInternal
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
    // P 的私有缓存区，使用时无需要加锁
	private interface{}
	// 公共缓存区。本地 P 可以 pushHead/popHead；其他 P 则只能 popTail
	shared  poolChain
}
```

字段 `pad` 主要是防止 `false sharing`，董大的[《什么是 cpu cache》](https://www.jianshu.com/p/dc4b5562aad2)里讲得比较好：

> 现代 cpu 中，cache 都划分成以 cache line (cache block) 为单位，在 x86_64 体系下一般都是 64 字节，cache line 是操作的最小单元。

> 程序即使只想读内存中的 1 个字节数据，也要同时把附近 63 节字加载到 cache 中，如果读取超个 64 字节，那么就要加载到多个 cache line 中。

简单来说，如果没有 pad 字段，那么当需要访问 0 号索引的 poolLocal 时，CPU 同时会把 0 号和 1 号索引同时加载到 cpu cache。在只修改 0 号索引的情况下，会让 1 号索引的 poolLocal 失效。这样，当其他线程想要读取 1 号索引时，发生 cache miss，还得重新再加载，对性能有损。增加一个 `pad`，补齐缓存行，让相关的字段能独立地加载到缓存行就不会出现 `false sharding` 了。

`poolChain` 是一个双端队列的实现：

```golang
type poolChain struct {
	// 只有生产者会 push to，不用加锁
	head *poolChainElt

	// 读写需要原子控制。 pop from
	tail *poolChainElt
}

type poolChainElt struct {
	poolDequeue

	// next 被 producer 写，consumer 读。所以只会从 nil 变成 non-nil
	// prev 被 consumer 写，producer 读。所以只会从 non-nil 变成 nil
	next, prev *poolChainElt
}

type poolDequeue struct {
	// The head index is stored in the most-significant bits so
	// that we can atomically add to it and the overflow is
	// harmless.
	// headTail 包含一个 32 位的 head 和一个 32 位的 tail 指针。这两个值都和 len(vals)-1 取模过。
	// tail 是队列中最老的数据，head 指向下一个将要填充的 slot
    // slots 的有效范围是 [tail, head)，由 consumers 持有。
	headTail uint64

	// vals 是一个存储 interface{} 的环形队列，它的 size 必须是 2 的幂
	// 如果 slot 为空，则 vals[i].typ 为空；否则，非空。
	// 一个 slot 在这时宣告无效：tail 不指向它了，vals[i].typ 为 nil
	// 由 consumer 设置成 nil，由 producer 读
	vals []eface
}
```

> `poolDequeue` 被实现为单生产者、多消费者的固定大小的无锁（atomic 实现） Ring 式队列（底层存储使用数组，使用两个指针标记 head、tail）。生产者可以从 head 插入、head 删除，而消费者仅可从 tail 删除。

> `headTail` 指向队列的头和尾，通过位运算将 head 和 tail 存入 headTail 变量中。

我们用一幅图来完整地描述 Pool 结构体：

![Pool 结构体](https://cdn.jsdelivr.net/gh/qcrao/images/blog/20200416125200.png)

结合木白的技术私厨的[《请问sync.Pool有什么缺点?》](https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487149&idx=1&sn=f38f2d72fd7112e19e97d5a2cd304430&source=41#wechat_redirect)里的一张图，对于双端队列的理解会更容易一些：

![Pool 结构体](https://cdn.jsdelivr.net/gh/qcrao/images/blog/image-20190805225842592.png)

我们看到 Pool 并没有直接使用 poolDequeue，原因是它的大小是固定的，而 Pool 的大小是没有限制的。因此，在 poolDequeue 之上包装了一下，变成了一个 `poolChainElt` 的双向链表，可以动态增长。

## Get

直接上源码：

```golang
func (p *Pool) Get() interface{} {
    // ......
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		x, _ = l.shared.popHead()
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
    // ......
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

省略号的内容是 `race` 相关的，属于阅读源码过程中的一些噪音，暂时注释掉。这样，Get 的整个过程就非常清晰了：

1.  首先，调用 `p.pin()` 函数将当前的 goroutine 和 P 绑定，禁止被抢占，返回当前 P 对应的 poolLocal，以及 pid。
    
2.  然后直接取 l.private，赋值给 x，并置 l.private 为 nil。
    
3.  判断 x 是否为空，若为空，则尝试从 l.shared 的头部 pop 一个对象出来，同时赋值给 x。
    
4.  如果 x 仍然为空，则调用 getSlow 尝试从其他 P 的 shared 双端队列尾部“偷”一个对象出来。
    
5.  Pool 的相关操作做完了，调用 `runtime_procUnpin()` 解除非抢占。
    
6.  最后如果还是没有取到缓存的对象，那就直接调用预先设置好的 New 函数，创建一个出来。
    

我用一张流程图来展示整个过程：

![Get 流程图](https://cdn.jsdelivr.net/gh/qcrao/images/blog/20200418091935.png)

整体流程梳理完了，我们再来看一下其中的一些关键函数。

### pin

先来看 `Pool.pin()`：

```golang
// src/sync/pool.go

// 调用方必须在完成取值后调用 runtime_procUnpin() 来取消抢占。
func (p *Pool) pin() (*poolLocal, int) {
	pid := runtime_procPin()
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
	// 因为可能存在动态的 P（运行时调整 P 的个数）
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}
```

`pin` 的作用就是将当前 groutine 和 P 绑定在一起，禁止抢占。并且返回对应的 poolLocal 以及 P 的 id。

如果 G 被抢占，则 G 的状态从 running 变成 runnable，会被放回 P 的 localq 或 globaq，等待下一次调度。下次再执行时，就不一定是和现在的 P 相结合了。因为之后会用到 pid，如果被抢占了，有可能接下来使用的 pid 与所绑定的 P 并非同一个。

“绑定”的任务最终交给了 `procPin`：

```golang
// src/runtime/proc.go

func procPin() int {
	_g_ := getg()
	mp := _g_.m

	mp.locks++
	return int(mp.p.ptr().id)
}
```

实现的代码很简洁：将当前 goroutine 绑定的 m 上的一个锁字段 locks 值加 1，即完成了“绑定”。关于 pin 的原理，可以参考[《golang的对象池sync.pool源码解读》](https://zhuanlan.zhihu.com/p/99710992)，文章详细分析了为什么执行 `procPin` 之后，不可抢占，且 GC 不会清扫 Pool 里的对象。

我们再回到 `p.pin()`，原子操作取出 `p.localSize` 和 `p.local`，如果当前 `pid` 小于 `p.localSize`，则直接取 poolLocal 数组中的 pid 索引处的元素。否则，说明 Pool 还没有创建 poolLocal，调用 `p.pinSlow()` 完成创建工作。

```golang
func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	// 没有使用原子操作，因为已经加了全局锁了
	s := p.localSize
	l := p.local
	// 因为 pinSlow 中途可能已经被其他的线程调用，因此这时候需要再次对 pid 进行检查。 如果 pid 在 p.local 大小范围内，则不用创建 poolLocal 切片，直接返回。
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	// 当前 P 的数量
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	// 旧的 local 会被回收
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
	return &local[pid], pid
}
```

因为要上一把大锁 `allPoolsMu`，所以函数名带有 `slow`。我们知道，锁粒度越大，竞争越多，自然就越“slow”。不过要想上锁的话，得先解除“绑定”，锁上之后，再执行“绑定”。原因是锁越大，被阻塞的概率就越大，如果还占着 P，那就浪费资源。

在解除绑定后，pinSlow 可能被其他的线程调用过了，p.local 可能会发生变化。因此这时候需要再次对 pid 进行检查。如果 pid 在 p.localSize 大小范围内，则不用再创建 poolLocal 切片，直接返回。

之后，根据 P 的个数，使用 make 创建切片，包含 `runtime.GOMAXPROCS(0)` 个 poolLocal，并且使用原子操作设置 p.local 和 p.localSize。

最后，返回 p.local 对应 pid 索引处的元素。

关于这把大锁 `allPoolsMu`，曹大在[《几个 Go 系统可能遇到的锁问题》](https://xargin.com/lock-contention-in-go/)里讲了一个例子。第三方库用了 `sync.Pool`，内部有一个结构体 `fasttemplate.Template`，包含 `sync.Pool` 字段。而 rd 在使用时，每个请求都会新建这样一个结构体。于是，处理每个请求时，都会尝试从一个空的 Pool 里取缓存的对象，最后 goroutine 都阻塞在了这把大锁上，因为都在尝试执行：`allPools = append(allPools, p)`，从而造成性能问题。

### popHead

回到 Get 函数，再来看另一个关键的函数：`poolChain.popHead()`：

```golang
func (c *poolChain) popHead() (interface{}, bool) {
	d := c.head
	for d != nil {
		if val, ok := d.popHead(); ok {
			return val, ok
		}
		// There may still be unconsumed elements in the
		// previous dequeue, so try backing up.
		d = loadPoolChainElt(&d.prev)
	}
	return nil, false
}
```

`popHead` 函数只会被 producer 调用。首先拿到头节点：c.head，如果头节点不为空的话，尝试调用头节点的 popHead 方法。注意这两个 popHead 方法实际上并不相同，一个是 `poolChain` 的，一个是 `poolDequeue` 的，有疑惑的，不妨回头再看一下 Pool 结构体的图。我们来看 `poolDequeue.popHead()`：

```golang
// /usr/local/go/src/sync/poolqueue.go

func (d *poolDequeue) popHead() (interface{}, bool) {
	var slot *eface
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		head, tail := d.unpack(ptrs)
		// 判断队列是否为空
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// head 位置是队头的前一个位置，所以此处要先退一位。
		// 在读出 slot 的 value 之前就把 head 值减 1，取消对这个 slot 的控制
		head--
		ptrs2 := d.pack(head, tail)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			// We successfully took back slot.
			slot = &d.vals[head&uint32(len(d.vals)-1)]
			break
		}
	}

    // 取出 val
	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}
	
	// 重置 slot，typ 和 val 均为 nil
	// 这里清空的方式与 popTail 不同，与 pushHead 没有竞争关系，所以不用太小心
	*slot = eface{}
	return val, true
}
```

此函数会删掉并且返回 `queue` 的头节点。但如果 `queue` 为空的话，返回 false。这里的 `queue` 存储的实际上就是 Pool 里缓存的对象。

整个函数的核心是一个无限循环，这是 Go 中常用的无锁化编程形式。

首先调用 `unpack` 函数分离出 head 和 tail 指针，如果 head 和 tail 相等，即首尾相等，那么这个队列就是空的，直接就返回 `nil，false`。

否则，将 head 指针后移一位，即 head 值减 1，然后调用 `pack` 打包 head 和 tail 指针。使用 `atomic.CompareAndSwapUint64` 比较 headTail 在这之间是否有变化，如果没变化，相当于获取到了这把锁，那就更新 headTail 的值。并且把 vals 相应索引处的元素赋值给 slot。

因为 `vals` 长度实际是只能是 2 的 n 次幂，因此 `len(d.vals)-1` 实际上得到的值的低 n 位是全 1，它再与 head 相与，实际就是取 head 低 n 位的值。

得到相应 slot 的元素后，经过类型转换并判断是否是 `dequeueNil`，如果是，说明没取到缓存的对象，返回 nil。

```golang
// /usr/local/go/src/sync/poolqueue.go
// 因为使用 nil 代表空的 slots，因此使用 dequeueNil 表示 interface{}(nil)
type dequeueNil *struct{}
```

最后，返回 val 之前，将 slot “归零”：`*slot = eface{}`。

回到 `poolChain.popHead()`，调用 `poolDequeue.popHead()` 拿到缓存的对象后，直接返回。否则，将 `d` 重新指向 `d.prev`，继续尝试获取缓存的对象。

### getSlow

如果在 shared 里没有获取到缓存对象，则继续调用 `Pool.getSlow()`，尝试从其他 P 的 poolLocal 偷取：

```golang
func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
	// 从其他 P 中窃取对象
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// 尝试从victim cache中取对象。这发生在尝试从其他 P 的 poolLocal 偷去失败后，
	// 因为这样可以使 victim 中的对象更容易被回收。
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// 清空 victim cache。下次就不用再从这里找了
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

从索引为 pid+1 的 poolLocal 处开始，尝试调用 `shared.popTail()` 获取缓存对象。如果没有拿到，则从 victim 里找，和 poolLocal 的逻辑类似。

最后，实在没找到，就把 victimSize 置 0，防止后来的“人”再到 victim 里找。

在 Get 函数的最后，经过这一番操作还是没找到缓存的对象，就调用 New 函数创建一个新的对象。

### popTail

最后，还剩一个 popTail 函数：

```golang
func (c *poolChain) popTail() (interface{}, bool) {
	d := loadPoolChainElt(&c.tail)
	if d == nil {
		return nil, false
	}

	for {
		d2 := loadPoolChainElt(&d.next)

		if val, ok := d.popTail(); ok {
			return val, ok
		}

		if d2 == nil {
			// 双向链表只有一个尾节点，现在为空
			return nil, false
		}

		// 双向链表的尾节点里的双端队列被“掏空”，所以继续看下一个节点。
		// 并且由于尾节点已经被“掏空”，所以要甩掉它。这样，下次 popHead 就不会再看它有没有缓存对象了。
		if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {
			// 甩掉尾节点
			storePoolChainElt(&d2.prev, nil)
		}
		d = d2
	}
}
```

在 `for` 循环的一开始，就把 d.next 加载到了 d2。因为 d 可能会短暂为空，但如果 d2 在 pop 或者 pop fails 之前就不为空的话，说明 d 就会永久为空了。在这种情况下，可以安全地将 d 这个结点“甩掉”。

最后，将 `c.tail` 更新为 `d2`，可以防止下次 `popTail` 的时候查看一个空的 `dequeue`；而将 `d2.prev` 设置为 `nil`，可以防止下次 `popHead` 时查看一个空的 `dequeue`。

我们再看一下核心的 `poolDequeue.popTail`：

```golang
// src/sync/poolqueue.go:147

func (d *poolDequeue) popTail() (interface{}, bool) {
	var slot *eface
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		head, tail := d.unpack(ptrs)
		// 判断队列是否空
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// 先搞定 head 和 tail 指针位置。如果搞定，那么这个 slot 就归属我们了
		ptrs2 := d.pack(head, tail+1)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			// Success.
			slot = &d.vals[tail&uint32(len(d.vals)-1)]
			break
		}
	}

	// We now own slot.
	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}

	slot.val = nil
	atomic.StorePointer(&slot.typ, nil)
	// At this point pushHead owns the slot.

	return val, true
}
```

`popTail` 从队列尾部移除一个元素，如果队列为空，返回 false。此函数可能同时被多个`消费者`调用。

函数的核心是一个无限循环，又是一个无锁编程。先解出 head，tail 指针值，如果两者相等，说明队列为空。

因为要从尾部移除一个元素，所以 tail 指针前进 1，然后使用原子操作设置 headTail。

最后，将要移除的 slot 的 val 和 typ “归零”：

```golang
slot.val = nil
atomic.StorePointer(&slot.typ, nil)
```

## Put

```golang
// src/sync/pool.go

// Put 将对象添加到 Pool 
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	// ……
	l, _ := p.pin()
	if l.private == nil {
		l.private = x
		x = nil
	}
	if x != nil {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
    //…… 
}
```

同样删掉了 race 相关的函数，看起来清爽多了。整个 Put 的逻辑也很清晰：

1.  先绑定 g 和 P，然后尝试将 x 赋值给 private 字段。
    
2.  如果失败，就调用 `pushHead` 方法尝试将其放入 shared 字段所维护的双端队列中。
    

同样用流程图来展示整个过程：

![Put 流程图](https://cdn.jsdelivr.net/gh/qcrao/images/blog/20200418092028.png)

### pushHead

我们来看 `pushHead` 的源码，比较清晰：

```golang
// src/sync/poolqueue.go

func (c *poolChain) pushHead(val interface{}) {
	d := c.head
	if d == nil {
		// poolDequeue 初始长度为8
		const initSize = 8 // Must be a power of 2
		d = new(poolChainElt)
		d.vals = make([]eface, initSize)
		c.head = d
		storePoolChainElt(&c.tail, d)
	}

	if d.pushHead(val) {
		return
	}

    // 前一个 poolDequeue 长度的 2 倍
	newSize := len(d.vals) * 2
	if newSize >= dequeueLimit {
		// Can't make it any bigger.
		newSize = dequeueLimit
	}

    // 首尾相连，构成链表
	d2 := &poolChainElt{prev: d}
	d2.vals = make([]eface, newSize)
	c.head = d2
	storePoolChainElt(&d.next, d2)
	d2.pushHead(val)
}
```

如果 `c.head` 为空，就要创建一个 poolChainElt，作为首结点，当然也是尾节点。它管理的双端队列的长度，初始为 8，放满之后，再创建一个 poolChainElt 节点时，双端队列的长度就要翻倍。当然，有一个最大长度限制（2^30）：

```golang
const dequeueBits = 32
const dequeueLimit = (1 << dequeueBits) / 4
```

调用 `poolDequeue.pushHead` 尝试将对象放到 poolDeque 里去：

```golang
// src/sync/poolqueue.go

// 将 val 添加到双端队列头部。如果队列已满，则返回 false。此函数只能被一个生产者调用
func (d *poolDequeue) pushHead(val interface{}) bool {
	ptrs := atomic.LoadUint64(&d.headTail)
	head, tail := d.unpack(ptrs)
	if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
		// 队列满了
		return false
	}
	slot := &d.vals[head&uint32(len(d.vals)-1)]

	// 检测这个 slot 是否被 popTail 释放
	typ := atomic.LoadPointer(&slot.typ)
	if typ != nil {
		// 另一个 groutine 正在 popTail 这个 slot，说明队列仍然是满的
		return false
	}

	// The head slot is free, so we own it.
	if val == nil {
		val = dequeueNil(nil)
	}
	
	// slot占位，将val存入vals中
	*(*interface{})(unsafe.Pointer(slot)) = val

	// head 增加 1
	atomic.AddUint64(&d.headTail, 1<<dequeueBits)
	return true
}
```

首先判断队列是否已满：

```golang
if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
	// Queue is full.
	return false
}
```

也就是将尾部指针加上 `d.vals` 的长度，再取低 31 位，看它是否和 head 相等。我们知道，`d.vals` 的长度实际上是固定的，因此如果队列已满，那么 if 语句的两边就是相等的。如果队列满了，直接返回 false。

否则，队列没满，通过 head 指针找到即将填充的 slot 位置：取 head 指针的低 31 位。

```golang
// Check if the head slot has been released by popTail.
typ := atomic.LoadPointer(&slot.typ)
if typ != nil {
	// Another goroutine is still cleaning up the tail, so
	// the queue is actually still full.
	// popTail 是先设置 val，再将 typ 设置为 nil。设置完 typ 之后，popHead 才可以操作这个 slot
	return false
}
```

上面这一段用来判断是否和 popTail 有冲突发生，如果有，则直接返回 false。

最后，将 val 赋值到 slot，并将 head 指针值加 1。

```golang
// slot占位，将val存入vals中
*(*interface{})(unsafe.Pointer(slot)) = val
```

> 这里的实现比较巧妙，slot 是 eface 类型，将 slot 转为 interface{} 类型，这样 val 能以 interface{} 赋值给 slot 让 slot.typ 和 slot.val 指向其内存块，于是 slot.typ 和 slot.val 均不为空。

## pack/unpack

最后我们再来看一下 pack 和 unpack 函数，它们实际上是一组绑定、解绑 head 和 tail 指针的两个函数。

```golang
// src/sync/poolqueue.go

const dequeueBits = 32

func (d *poolDequeue) pack(head, tail uint32) uint64 {
	const mask = 1<<dequeueBits - 1
	return (uint64(head) << dequeueBits) |
		uint64(tail&mask)
}
```

`mask` 的低 31 位为全 1，其他位为 0，它和 tail 相与，就是只看 tail 的低 31 位。而 head 向左移 32 位之后，低 32 位为全 0。最后把两部分“或”起来，head 和 tail 就“绑定”在一起了。

相应的解绑函数：

```golang
func (d *poolDequeue) unpack(ptrs uint64) (head, tail uint32) {
	const mask = 1<<dequeueBits - 1
	head = uint32((ptrs >> dequeueBits) & mask)
	tail = uint32(ptrs & mask)
	return
}
```

取出 head 指针的方法就是将 ptrs 右移 32 位，再与 mask 相与，同样只看 head 的低 31 位。而 tail 实际上更简单，直接将 ptrs 与 mask 相与就可以了。

## GC

对于 Pool 而言，并不能无限扩展，否则对象占用内存太多了，会引起内存溢出。

> 几乎所有的池技术中，都会在某个时刻清空或清除部分缓存对象，那么在 Go 中何时清理未使用的对象呢？

答案是 GC 发生时。

在 pool.go 文件的 init 函数里，注册了 GC 发生时，如何清理 Pool 的函数：

```golang
// src/sync/pool.go

func init() {
	runtime_registerPoolCleanup(poolCleanup)
}
```

编译器在背后做了一些动作：

```golang
// src/runtime/mgc.go

// Hooks for other packages

var poolcleanup func()

// 利用编译器标志将 sync 包中的清理注册到运行时
//go:linkname sync_runtime_registerPoolCleanup sync.runtime_registerPoolCleanup
func sync_runtime_registerPoolCleanup(f func()) {
	poolcleanup = f
}
```

具体来看下：

```golang
func poolCleanup() {
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	oldPools, allPools = allPools, nil
}
```

`poolCleanup` 会在 STW 阶段被调用。整体看起来，比较简洁。主要是将 local 和 victim 作交换，这样也就不致于让 GC 把所有的 Pool 都清空了，有 victim 在“兜底”。

> 如果 `sync.Pool` 的获取、释放速度稳定，那么就不会有新的池对象进行分配。如果获取的速度下降了，那么对象可能会在两个 `GC` 周期内被释放，而不是以前的一个 `GC` 周期。

> 鸟窝的[【Go 1.13中 sync.Pool 是如何优化的?】](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/)讲了 1.13 中的优化。

参考资料[【理解 Go 1.13 中 sync.Pool 的设计与实现】](https://zhuanlan.zhihu.com/p/110140126) 手动模拟了一下调用 `poolCleanup` 函数前后 oldPools，allPools，p.vitcim 的变化过程，很精彩：

> 1.  初始状态下，oldPools 和 allPools 均为 nil。

2.  第 1 次调用 Get，由于 p.local 为 nil，将会在 pinSlow 中创建 p.local，然后将 p 放入 allPools，此时 allPools 长度为 1，oldPools 为 nil。
3.  对象使用完毕，第 1 次调用 Put 放回对象。
4.  第 1 次GC STW 阶段，allPools 中所有 p.local 将值赋值给 victim 并置为 nil。allPools 赋值给 oldPools，最后 allPools 为 nil，oldPools 长度为 1。
5.  第 2 次调用 Get，由于 p.local 为 nil，此时会从 p.victim 里面尝试取对象。
6.  对象使用完毕，第 2 次调用 Put 放回对象，但由于 p.local 为 nil，重新创建 p.local，并将对象放回，此时 allPools 长度为 1，oldPools 长度为 1。
7.  第 2 次 GC STW 阶段，oldPools 中所有 p.victim 置 nil，前一次的 cache 在本次 GC 时被回收，allPools 所有 p.local 将值赋值给 victim 并置为nil，最后 allPools 为 nil，oldPools 长度为 1。

我根据这个流程画了一张图，可以理解地更清晰一些：

![poolCleanup 过程](https://cdn.jsdelivr.net/gh/qcrao/images/blog/20200416125040.png)

需要指出的是，`allPools` 和 `oldPools` 都是切片，切片的元素是指向 Pool 的指针，Get/Put 操作不需要通过它们。在第 6 步，如果还有其他 Pool 执行了 Put 操作，`allPools` 这时就会有多个元素。

在 Go 1.13 之前的实现中，`poolCleanup` 比较“简单粗暴”：

```golang
func poolCleanup() {
    for i, p := range allPools {
        allPools[i] = nil
        for i := 0; i < int(p.localSize); i++ {
            l := indexLocal(p.local, i)
            l.private = nil
            for j := range l.shared {
                l.shared[j] = nil
            }
            l.shared = nil
        }
        p.local = nil
        p.localSize = 0
    }
    allPools = []*Pool{}
}
```

直接清空了所有 Pool 的 `p.local` 和 `poolLocal.shared`。

> 通过两者的对比发现，新版的实现相比 Go 1.13 之前，GC 的粒度拉大了，由于实际回收的时间线拉长，单位时间内 GC 的开销减小。

> 由此基本明白 p.victim 的作用。它的定位是次级缓存，GC 时将对象放入其中，下一次 GC 来临之前如果有 Get 调用则会从 p.victim 中取，直到再一次 GC 来临时回收。

> 同时由于从 p.victim 中取出对象使用完毕之后并未放回 p.victim 中，在一定程度也减小了下一次 GC 的开销。原来 1 次 GC 的开销被拉长到 2 次且会有一定程度的开销减小，这就是 p.victim 引入的意图。

[【理解 Go 1.13 中 sync.Pool 的设计与实现】](https://zhuanlan.zhihu.com/p/110140126) 这篇文章最后还总结了 `sync.Pool` 的设计理念，包括：无锁、操作对象隔离、原子操作代替锁、行为隔离——链表、Victim Cache 降低 GC 开销。写得非常不错，推荐阅读。

另外，关于 `sync.Pool` 中锁竞争优化的文章，推荐阅读芮大神的[【优化锁竞争】](http://xiaorui.cc/archives/5878)。

# 总结

本文先是介绍了 Pool 是什么，有什么作用，接着给出了 Pool 的用法以及在标准库、一些第三方库中的用法，还介绍了 pool_test 中的一些测试用例。最后，详细解读了 `sync.Pool` 的源码。

本文的结尾部分，再来详细地总结一下关于 `sync.Pool` 的要点：

1.  关键思想是对象的复用，避免重复创建、销毁。将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力。
    
2.  `sync.Pool` 是协程安全的，使用起来非常方便。设置好 New 函数后，调用 Get 获取，调用 Put 归还对象。
    
3.  Go 语言内置的 fmt 包，encoding/json 包都可以看到 sync.Pool 的身影；`gin`，`Echo` 等框架也都使用了 sync.Pool。
    
4.  不要对 Get 得到的对象有任何假设，更好的做法是归还对象时，将对象“清空”。
    
5.  Pool 里对象的生命周期受 GC 影响，不适合于做连接池，因为连接池需要自己管理对象的生命周期。
    
6.  Pool 不可以指定⼤⼩，⼤⼩只受制于 GC 临界值。
    
7.  `procPin` 将 G 和 P 绑定，防止 G 被抢占。在绑定期间，GC 无法清理缓存的对象。
    
8.  在加入 `victim` 机制前，sync.Pool 里对象的最⼤缓存时间是一个 GC 周期，当 GC 开始时，没有被引⽤的对象都会被清理掉；加入 `victim` 机制后，最大缓存时间为两个 GC 周期。
    
9.  Victim Cache 本来是计算机架构里面的一个概念，是 CPU 硬件处理缓存的一种技术，`sync.Pool` 引入的意图在于降低 GC 压力的同时提高命中率。
    
10.  `sync.Pool` 的最底层使用切片加链表来实现双端队列，并将缓存的对象存储在切片中。
    

## 二、结构体说明

![](https://pic3.zhimg.com/80/v2-188ef300177a16a313d0d28f68797b72_720w.jpg)

### Pool

> `**_noCopy_**`：可以嵌入到结构中，在第一次使用后不可复制,使用go vet作为检测使用。  
> `**_local_**`：P大小的poolLocal数组的指针([P]poolLocal),P一般是cpu的核数。  
> `**_localSize_**`：本地poolLocal数组大小  
> `**_victim_**`：victim也是一个poolLocal数组的指针,每次垃圾回收的时候，会被移除。然后把local数据给victim。  
> `**_victimSize_**`：本地victim数组大小  
> `**_New_**`：当池中没有对象时，会调用new函数构造一个对象

### poolLocal

> `**_poolLocalInternal_**`：本地pool池。  
> `**_pad_**`：防止在缓存行上分配多个poolLocalInternal从而造成false sharing（伪共享）

### poolLocalInternal

> `**_private_**`：本地私有池，只能被对应的P(goroutine执行占用的处理器)使用。不会有并发问题，不用加锁。  
> `**_share_**`：本地共享池，本地能通过pushHead和popHead，其他任何P通过popTail使用。因为可能有多个goroutine同时操作，所以需要加锁。类型为poolChain,是被实现为poolDequeue的双向队列。具体见下图。

![](https://pic4.zhimg.com/80/v2-497e4fec92d469234070e38e7ffe8c67_720w.jpg)

### poolChain

> `_**head**_`：队列头部。  
> `_**tail**_`：队列尾部。

### poolChainElt

> `_**poolDequeue**_`：存放数据；是一个无锁的固定大小的单生产者，多消费者的环形队列。  
> `_**next,prev**_`：在poolChain中，上一个和下一个连接到的poolChainElt节点。

### poolDequeue

> `**_headTail_**`：是环状队列的首位位置的指针，可以通过位运算解析出首尾的位置。生产者可以从 head 插入、head 删除，而消费者仅可从 tail 删除  
> `**_vals_**` ：是存储 interface{} 值,这个的大小必须是2的幂。

### eface

> `_**typ,val**_`：存储eface值，是一个无特殊意义的指针，可以包含任意类型的地址。

## 三、runtime*函数处理

### runtime_procUnpin

> 在src/runtime/proc.go中被实现sync_runtime_procUnpin；表示获取当前goroutine，并解除禁止抢占。

### runtime_procPin

> 在src/runtime/proc.go中被实现sync_runtime_procPin；表示获取当前goroutine绑定的M，获取M绑定的P的id，并且禁止抢占。

### runtime_registerPoolCleanup

> 在src/runtime/proc.go中被实现sync_runtime_registerPoolCleanup；表示在GC之前调用该函数。

## 四、源码实现

### Put

```go
func (p *Pool) Put(x interface{}) {
        // 对象x 不能为nil
	if x == nil {
		return
	}
        // 竞态检测
	if race.Enabled {
		if fastrand()%4 == 0 {
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
        // 获取当前P对应的poolLocal对象池
	l, _ := p.pin() 
        // 本地私有池为空
	if l.private == nil {
                // 赋值
		l.private = x
		x = nil
	}
        // x不为空
	if x != nil {
                // 添加至本地共享池
		l.shared.pushHead(x)
	}
        // 调用方必须在完成取值后调用 runtime_procUnpin() 来取消禁止抢占
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}

func (c *poolChain) pushHead(val interface{}) {
	d := c.head //获取头节点
	if d == nil {  // 如果头节点为空
                // 初始化链表
		const initSize = 8 //必须是2的幂
		d = new(poolChainElt) //初始化poolChainElt
		d.vals = make([]eface, initSize)
		c.head = d
		storePoolChainElt(&c.tail, d)
	}
        // 调用pushHead放入到环形队列中
	if d.pushHead(val) {
		return
	}
        // 当前出队已满，分配一个队列的长度翻倍。不能超过dequeueLimit即为2的30次方
	newSize := len(d.vals) * 2
	if newSize >= dequeueLimit {
		newSize = dequeueLimit
	}
        // 将新的节点d2和d相互绑定，并设置d2位头节点，将传入的对象push到d2中
	d2 := &poolChainElt{prev: d}
	d2.vals = make([]eface, newSize)
	c.head = d2
	storePoolChainElt(&d.next, d2)
	d2.pushHead(val)
}

func (d *poolDequeue) pushHead(val interface{}) bool {
	ptrs := atomic.LoadUint64(&d.headTail)
	head, tail := d.unpack(ptrs) // 解包headTail
        // 判断队列是否已满
	if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
		return false
	}
        // 找到head的槽位
	slot := &d.vals[head&uint32(len(d.vals)-1)]
	// 检查slot是否和popTail有冲突
	typ := atomic.LoadPointer(&slot.typ)
	if typ != nil { //另一个goroutine仍在清理tail,所以队列实际上仍然已满
		return false
	}
	if val == nil {
		val = dequeueNil(nil)
	}
        // 将 val 赋值到 slot，并将 head 指针值加 1
	*(*interface{})(unsafe.Pointer(slot)) = val
	atomic.AddUint64(&d.headTail, 1<<dequeueBits)
	return true
}



func (p *Pool) pin() (*poolLocal, int) {
        // 获取当前goroutine绑定到对应的M上，返回M绑定P对应的id，id为当前cpu的id。
        // 禁止抢占
	pid := runtime_procPin()
        // 原子获取本地poolLocal数组的大小
	s := atomic.LoadUintptr(&p.localSize) 
	l := p.local     
        // 如果当前pid>s， 说明当前goroutine对应的pid的poolLocal不存在。
        // 反之，即为存在，返回该pid对应的poolLocal对象                    
	if uintptr(pid) < s { 
		return indexLocal(l, pid), pid
	}
        // pid对应的poolLocal对象不存在时调用
	return p.pinSlow()
}
// 获取当前Pid对应的本地poolLocal对象
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}

func (p *Pool) pinSlow() (*poolLocal, int) {
	runtime_procUnpin() //解除抢占
	allPoolsMu.Lock() // 加上全局锁
	defer allPoolsMu.Unlock() //全局解除锁定
	pid := runtime_procPin() // 将当前goroutine订在P上，禁止抢占，返回当前P对应的id
	// 当再次固定P时poolCleanup不会被调用
	s := p.localSize
	l := p.local
        // 如果当前pid>s， 说明当前goroutine对应的pid的poolLocal不存在。
        // 反之，即为存在，返回该pid对应的poolLocal对象     
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
        // 初始化local前会将pool放入到allPools数组中
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// 如果GOMAXPROCS在GC之间更改，我们将初始化重新分配该数组，并丢失旧的数组
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) 
	atomic.StoreUintptr(&p.localSize, uintptr(size))        
        // 返回P的id对应的本地poolLocal对象，和 pid
        // 所以pool的最大个数为runtime.GOMAXPROCS(0)
	return &local[pid], pid
}
```

![](https://pic2.zhimg.com/80/v2-7247497f82ffae8b90d6b6c563a507e9_720w.jpg)

### Get

```go
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
        // 获取当前goroutine对应的P的id对应的poolLocal对象
	l, pid := p.pin()
	x := l.private //赋值x为本地私有池，优先获取本地私有池对象
	l.private = nil
	if x == nil { 
                // 如果private没有，那么从share本地共享池的头部获取
		x, _ = l.shared.popHead()
		if x == nil {
                        // 如果还是没有，那么调用getSlow从别地poolLocal去偷取
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin() //解除抢占
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
        // 如果还是没有获取到，尝试使用new函数生成一个新的
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
func (p *Pool) getSlow(pid int) interface{} {
	size := atomic.LoadUintptr(&p.localSize) // 获取本地poolLocal数量
	locals := p.local                        // 获取本地poolLocal
	// 尝试从其他P中偷取
	for i := 0; i < int(size); i++ { 
                // 遍历是从pid+1开始的
		l := indexLocal(locals, (pid+i+1)%int(size))
                // 从其他P对应的本地共享池列表尾部获取对象
		if x, _ := l.shared.popTail(); x != nil {
                        // 成功直接返回
			return x
		}
	}
 
        // 尝试在下次GC前偷取到上次GC初始化时设置的本地对象池
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
        // victim的private不为空则返回
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
        //  遍历victim对应的locals列表，从其他的local的shared列表尾部获取对象
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

        // 获取不到，将victimSize置为0
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}

func (c *poolChain) popHead() (interface{}, bool) {
	d := c.head // 这里头节点是一个poolChainElt
	for d != nil {
                // 从poolChainElt的环状列表中获取值
		if val, ok := d.popHead(); ok {
			return val, ok
		}
                // load poolChain下一个对象，重新开始for循环 
		d = loadPoolChainElt(&d.prev)
	}
	return nil, false
}

func (d *poolDequeue) popHead() (interface{}, bool) {
	var slot *eface
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
                // 解包headTail，高32位为head，低32位为tail
		head, tail := d.unpack(ptrs) 
		if tail == head { //头节点与尾节点相等，证明队列为空，返回
			return nil, fals
		}
                // 这里需要head--之后再获取slot
		head--
		ptrs2 := d.pack(head, tail) //打包
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			slot = &d.vals[head&uint32(len(d.vals)-1)]
			break
		}
	}

	val := *(*interface{})(unsafe.Pointer(slot))
        // // 说明没取到缓存的对象，返回 nil
	if val == dequeueNil(nil) {
		val = nil
	}
        // 重置slot 
	*slot = eface{}
	return val, true
}
```

![](https://pic3.zhimg.com/80/v2-0c7996a9af447300cc37456715cac842_720w.jpg)


# sync.pool的优化过程
1.12及之前版本的sync.Pool有三个问题：

1.  每次GC都回收所有对象，如果缓存对象数量太大，会导致STW1阶段的耗时增加。
    
2.  每次GC都回收所有对象，导致缓存对象命中率下降，New方法的执行造成额外的内存分配消耗。
    
3.  Pool.Get方法底层有锁，极端情况下，要尝试最多P次抢锁，也获取不到缓存对象，最后得执行New方法返回对象。
    

  

这些问题就对sync.Pool的室使用提出了要求，不满足时，性能并不会有大幅提升：

1.  最好是高并发场景。（对应问题3）
    
2.  最好两次GC之间的间隔足够长。（对应问题1，2）
    

  

**先简单介绍下原理，看哪块源码造成这三个问题。**

> 如果对sync.Pool的基本原理一点都不了解，可以移步先阅读《golang标准库sync.Pool原理及源码简析》或点击阅读原文

  

sync.Pool对象内部为每个P都分配了一个**private区**和**shared区**。

private区只能存放一个可复用对象，因为每个P在任意时刻只运行一个G，所以在private区上写入和取出对象是不用加锁的。

shared区可以放多个可复用对象，它本身是slice。进shared区就append，出shared区就slice[:last-1]。但shared区上写入和取出对象要加锁，因为别的G可能过来偷对象。
![[Pasted image 20220513111818.png]]
  

问题3 就是由于shared区是一个带锁的后进先出队列造成的。每次Pool.Get方法在调用时，执行顺序是：

1.  先看当前P的private区是否为空。
    
2.  加锁，看当前P的shared区是否为空。
    
3.  加锁，循环遍历看其他P的shared区是否为空。
    
4.  只要上面三步任意一步就不为空，就可以把缓存对象返回了。但若都为空，最后就得调用New方法返回对象。
    
![[Pasted image 20220513111831.png]]

  

这一顿的加锁操作和Mutex锁自带的阻塞唤醒开销，Get方法在极端情况下就会有性能问题。

> Mutex锁分析参考《[一份详细注释的go Mutex源码](http://mp.weixin.qq.com/s?__biz=MzIyMTY3NDI3Ng==&mid=2247483656&idx=1&sn=49f1499054001e1a30b22d9ea1db98ac&chksm=e8386f43df4fe6551d3baea53f4c4241bdfd36775fbe660a2fc67bf7d61f34c7c590c23a208b&scene=21#wechat_redirect)》

  

问题2和3 都是由于每次GC时，遍历清空所有缓存对象造成的。

sync.Pool在init()中向runtime注册了一个cleanup方法，它在STW1阶段被调用的。如果它执行过久，就会硬生生延长STW1阶段耗时。

```go
func init() {
  runtime_registerPoolCleanup(poolCleanup)
}
```

这个cleanup方法干的事情是遍历所有的sync.Pool对象，再遍历每个sync.Pool对象中的每个P的shared区，把shared区每个缓存对象设置为nil。代码中就是三层for循环，简单粗暴时间复杂度高。  

![[Pasted image 20220513111920.png]]

  

好消息是1.13beta1已经解决了这三个问题。注意是beta版本，而不是stable版本。

接下来主要看1.13通过什么思路解决这些问题的。

> 不排除未来的stable版本或1.13的小版本会对这块实现做小改动。

  

### 取消每次GC默认对全部对象进行回收

解决问题2和3的思路就是不能每次全部回收。但该回收多少呢？

@aclements 提出了一种思路。这轮在sync.Pool中的对象，最快也在下轮GC才被回收。

> https://github.com/golang/go/issues/22950#issuecomment-352935997

  

还记得上面说过每个P都有private区和shared区吗？现在每个P里两个区合在一起构成数组，给个名字叫**local**（其实也是源码中的实现）。1.13版本的实现中再引入一个**victim**，它结构与local一致。
![[Pasted image 20220513111939.png]]

  

有了victim，Get和Put方法的步骤就有所变化：

1.  Get时，先从local里尝试取出缓存对象（包括所有的P）。如果失败，就尝试从victim里取。
    
2.  victim里也取对象失败，就调用New方法。
    
3.  Put时，只放local里。
    

  

新的数据结构下，cleanup方法策略也有所变化，改为每次只把victim里的对象回收掉。然后victim再指向当前的local。

![[Pasted image 20220513111950.png]]

  

显然这样好处就是这轮的缓存对象在GC时不会立马回收，而是存放起来，滞后一轮。这样下一轮能得到复用机会，提高了缓存对象的命中率。并且回收对象时，由对shared区O(n)的遍历操作，变成O(1)。

从benchmark感受这个优化带来的性能提升：
![[Pasted image 20220513112000.png]]

1.9.7版本的STW1阶段耗时TP96线是285485ns，而1.13beta1是7720ns。

> Benchmark代码参考1.13beat1源码src/sync/pool_test.go.BenchmarkPoolSTW方法

  

### 使用无锁队列替换shared区

问题3 是因为在shared的访问加了一把Mutex锁造成的。如果不消除这把锁，引入victim区也是徒劳。因为此时victim的访问也得加锁。

  

旧实现中shared区是单纯的带锁后进先出队列，1.13beta版本改成了单生产者，多消费者的双端无锁环形队列。

  

单生产者是指，每个P上运行的G，执行Put方法时，就往队列里存放缓存对象（别的P上运行的G不能往里放），并且只能放在队列头部。由于每个P任意时刻只有一个G被运行，所以存放缓存对象不需要加锁。

多消费者分两种角色，一是在P上运行的G，执行Get方法时，从队列头部取出缓存对象。同上，取对象不用加锁；二是在其他P上运行的G，执行Get方法时，本地没有缓存对象，就到别的P上偷。此时盗窃者G只能从队列尾部取出对象，因为盗窃者可能有多个，所以尾部取数据用CAS来实现无锁。

> 注意，每个P都持有自己无锁队列，下图只画出了P0的。
> 
> 并且队列也可能有多个，下图只画出单队列的情况。

![图片](https://mmbiz.qpic.cn/mmbiz_png/WlfJ9xvT2b2CchdAMk2PqrRrbWDicy4hEIBf9Vx0nM77aeSiarTnOpsv2GM5RnvVqDHnHnEXT3VBGguj3jotLiaTQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> 如何正确地实现无锁队列超出本文意图，不展开介绍。感兴趣可以自行找资料学习或看源码。

  

每个P持有的循环队列初始化多大呢？增长和收缩策略呢？

下图用一张图做宏观介绍。

![图片](https://mmbiz.qpic.cn/mmbiz_png/WlfJ9xvT2b2CchdAMk2PqrRrbWDicy4hEDiahv2vibrAXnTiauBI73eoFphhZdPnhzFUlmVweN4FZtvOYWblAI893A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  

陈述要点：

-   shared区改用双向链表，每个链表节点指向一个无锁环形队列。
    
-   链表节点必须在头部插入。
    
-   当前P上的G取缓存对象时，只从头部链表节点指向的无锁队列里取。取不到，沿着prev指针到下一个无锁队列上重复操作，也没有的话。就到别的P上偷。
    
-   盗窃者G在偷缓存对象时，只从尾部链表节点指向的无锁队列里取。取不到，沿着next指针到下一个无锁队列上重复操作，也没有的话。就到别的P上继续偷，直到都偷不着，就调用New方法。
    
-   链表首次插入节点时，指向无锁队列初始化大小为8，增长策略为在头部插入新节点，指向的无锁队列大小为旧头部节点指向无锁队列大小的两倍，始终保持2的n次方大小。
    
-   假如在链表长度为3的情况下。尾部节点指向的无锁队列里缓存对象被偷光了。那么尾部节点会沿着next指针前移，把旧的无锁队列内存释放掉。此时链表长度变为2，这就是链表的收缩策略。最小时剩下一个节点，不会收缩成空链表。
    
-   无锁队列的自身最大的大小是2\* 30。达到上限时，再执行Put操作就放不进去，也不报错。


Go 1.13持续对 `sync.Pool`进行了改进，这里我们有两个简单的灵魂拷问：

1、做了哪些改进？  
2、如何做的改进？

首先回答第一个问题：

-   对STW暂停时间做了优化, 避免大的sync.Pool严重影响STW时间
-   第二个优化是GC时入股对sync.Pool进行回收，不会一次将池化对象全部回收，这就避免了sync.Pool释放对象和重建对象导致的性能尖刺，造福于sync.Pool重度用户。
-   第三个就是对性能的优化。

对以上的改进主要是两次提交：: [sync: use lock-free structure for Pool stealing](https://github.com/golang/go/commit/d5fd2dd6a17a816b7dfd99d4df70a85f1bf0de31#diff-491b0013c82345bf6cfa937bd78b690d)和[sync: use lock-free structure for Pool stealing](https://github.com/golang/go/commit/2dcbf8b3691e72d1b04e9376488cef3b6f93b286#diff-491b0013c82345bf6cfa937bd78b690d)。

两次提交都不同程度的对性能有所提升，依据不同的场景，提升0.7% ～ 92%不同。 Go开发者有一个很好的习惯，或者叫做约定，或者是他们的开发规范，对于标准库的修改都会执行性能的比较，代码的修改不应该带来性能的降低。这两次提交的注释文档都详细的介绍了性能的比较结果。

知道了第一个问题的答案可以让我们对sync.Pool有信心，在一些场景下可以考虑使用sync.Pool，以便减少对象的创建和回收对GC的影响。

了解第二个问题可以让我们学到Go开发者的优化手段，或许在我们自己的项目中也使用这些优化手段来优化我们自己的代码。

## sync: use lock-free structure for Pool stealing

第一次提交提升用来提高sync.Pool的性能，减少STW时间。

Go 1.13之前，Pool使用一个Mutex保护的slice来存储每个shard的overflow对象。(sync.Pool使用shard方式存储池化对象，减少竞争。 每个`P`对应一个shard。如果需要创建多于一个池化对象，这些对象就叫做overflow)。

```go
type poolLocalInternal struct {

 	private interface{}   // Can be used only by the respective P.	by the respective P.

 	shared  []interface{} // Can be used by any P.

 	Mutex                 // Protects shared.		

 }
```
那么在Go 1.13中，使用是以 lock-free的数据结构代替`slice` + `Mutex`的方式：

```go
type poolLocalInternal struct {

	private interface{} // Can be used only by the respective P.

	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.

}
```

这个lock-free的数据结构的实现很特别。

我们知道，实现lock-free的数据结构一般采用`atomic`的方式实现，通过`CAS`避免操作`block`住。sync.Pool也是采用这种方式，它定义了一个lock-free的双向链表:
```go
type poolDequeue struct {

	headTail uint64

	vals []eface

}
```

`poolDequeue`是一个[特别的队列](https://github.com/golang/go/blob/master/src/sync/poolqueue.go)，有以下特点：

-   lock-free
-   固定大小，ring形结构(底层存储使用数组,使用两个指针标记ehead、tail)
-   单生产者
-   多消费者
-   生产者可以从head进行`pushHead`、`popHead`
-   消费者可以从tail进行`popTail`

它的head和tail是采用一个uint64数值来表示的，好处就是我们可以通过`atomic`对这两个值整体进行CAS。它提供了`unpack`、`pack`可以从`headTail`中解析初独立的head和tail, 以及执行相反的操作。

数组存储在`vals`数组中，它采用interface的结构进行存储。

如果你看它的`pushHead`、`popHead`和`popTail`代码，可以看到它主要使用`atomic`来实现lock-free。lock-free代码比较简单，本文就不进行详细解读了，阅读的时候注意head和tail的边界问题。因为它是使用一个数组(准确的说是slice)实现一个ringbuffer的数据结构，这样能充分利用分配的空间。

sync.Pool还不是直接使用`poolDequeue`这样一个数据结构，原因在于`poolDequeue`是一个固定大小的队列，这个大小取什么值才合理呢？取的太小，以后可能不得不grow, 取的太大，又可能浪费。

解决这个问题就是采用动态增长的方式。它定义了一个队列链表池,可以实现动态的上述队列的增减:
```go
type poolChain struct {

	head *poolChainElt //只会被生产者使用

	tail *poolChainElt //只会被消费者使用

}
```

一开始，它会使用长度为8的`poolDequeue`做存储，一旦这个队列满了，就会再创建一个长度为16的队列，以此类推，只要当前的队列满了，就会新创建一 2*n的`poolDequeue`做存储。如果当前的`poolDequeue`消费完，就会丢弃。

这样一个动态可变的lock-free队列正是sync.Pool所要的,当然为了CPU缓存优化还进行了缓存行的对齐：
```go
type poolLocalInternal struct {

	private interface{} // Can be used only by the respective P.

	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.

}

type poolLocal struct {

	poolLocalInternal

	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte

}

type Pool struct {

	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal

	localSize uintptr        // size of the local array

	New func() interface{}

}
```

`Pool`使用shard的方式实现池(local实际上是[P]poolLocal, 这里采用指针的方式)，它的`Get`、`Put`移除了Mutex的加锁控制，而是采用lock-free数据结构`poolChain`实现。

当然有人可能提出质疑: `lock-free`真的比`Mutex`性能好吗？在一定的竞争条件下，确实`lock-free`的性能要好于Mutex, 如果你举极端的例子，比如竞争非常激烈的情况，或许会有不同的结果，但是绝大部分情况下，`lock-free`性能还是要好一些。

[![](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/lock-free-performance.png)](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/lock-free-performance.png)

注意Pool的实现中使用了`runtime_procPin()`方法，它可以将一个goroutine死死占用当前使用的`P`(P-M-G中的processor)，不允许其它goroutine/M抢占,这样它就可以自由的使用shard中和这个P相关的local，不必担心竞争的问题。释放pin的方法是`runtime_procUnpin`。

此时的`poolCleanup` (GC的时候对池化对象的释放)还是全部清空，进一步的优化在下一个提交中。

## sync: smooth out Pool behavior over GC with a victim cache

上一节提到每次Pool清理的时候都是把所有的池化对象都释放掉，这会带来两个问题：

1.  浪费: 池化对象全部释放后等需要的时候又不得不重新创建
2.  GC尖峰:突然释放大量的池化对象会导致GC耗时增加

所以这次提交引入了`victim cache`的机制。`victim cache`原是CPU硬件处理缓存的一种[技术](https://en.wikipedia.org/wiki/Victim_cache),

> 所谓受害者缓存（Victim Cache），是一个与直接匹配或低相联缓存并用的、容量很小的全相联缓存。当一个数据块被逐出缓存时，并不直接丢弃，而是暂先进入受害者缓存。如果受害者缓存已满，就替换掉其中一项。当进行缓存标签匹配时，在与索引指向标签匹配的同时，并行查看受害者缓存，如果在受害者缓存发现匹配，就将其此数据块与缓存中的不匹配数据块做交换，同时返回给处理器。
> 
> from wikipedia

相比较先前的直接清除Pool, 这次修改后是清除`victim cache`，然后将`primary cache`转移给`victim cache`。如果sync.Pool的获取释放速度稳定，那么就不会又新的池对象进行分配。如果获取的速度下降了，那么对象可能会在两个GC周期内被释放，而不是以前的一个GC周期。

同时，`victim cache`的设计也间接的提升GC的性能，因为稳定的sync.Pool使用导致池化的对象都是long-live的对象，而GC的主要对象是short-live的对象，所以会减少GC的执行。

相对于以前的实现，现在的sync.Pool的实现增加了victim相关的两个字段:

```go
type Pool struct {

	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal

	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle

	victimSize uintptr        // size of victims array

	New func() interface{}

}
```

它主要影响两个方法的实现: `getSlow`和`poolCleanup`。

[![](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/getSlow.png)](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/getSlow.png)

当前goroutine从自己的`P`对应的本地获取不到free的池化对象的话，就会调用`getSlow`, 尝试从其它shard中"偷取"。

如果不幸的是其它shard也没有free的池化对象的话，那么就就尝试从`victim`中找一个，寻找的方法和从本地中寻找是一样一样的。

找到的话就返回，找不到的话如果定义`New`创建函数，就创建一个，如果没定义`New`返回空。

[![](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/poolCleanup.png)](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/poolCleanup.png)

清理的时候就是把每一个sync.Pool的`victim`都清空,然后再把本地local的池化对象赋值给`victim`， 本地对象都清空。

## sync.Pool 整体 Get/Put 逻辑

Vincent Blanchon曾在他的[Go: Understand the Design of Sync.Pool](https://medium.com/a-journey-with-go/go-understand-the-design-of-sync-pool-2dde3024e277)一文中给出了sync.Pool go 1.12版本的Get/Put的流程图。这里我画了一个go 1.13版本的流程图，可以很好的理解sync.Pool处理的过程。

[![](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/Get.png)](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/Get.png)

[![](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/Put.png)](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/Put.png)

  


### 要解决的问题

一个新技术亦或是一个新名词，总是为了解决一些问题才出现的，所以先搞明白解决什么问题是第一步。 核心来说，我们的代码中会各种创建对象，比如new一个结构体、创建一个连接、甚至创建一个int都属于对象。那么假设在某些场景下，你的代码会频繁的创建某一种对象，那么这种操作可能会影响你程序的性能，原因是什么呢？

1.我们知道创建对象肯定是需要申请内存的

2.频繁的创建对象，会对GC造成较大的压力，其实主要是GC压力较大，golang的官方库sync.pool就是为了解决它，看名字就是池的方法。

### 池和缓存

sync.pool的思想很简单就是对象池，由于最近一直在做相关的事情，这里我们说个题外话，关于池和缓存说下我的一些看法。

1.工作中遇到过很多池：连接池，线程池，协程池，内存池等，会发现这些所谓池，都是解决同一个类型的问题，创建连接、线程等比较消耗资源，所以用池化的思想来解决这些问题，直接复用已经创建好的。

2.其实缓存也是，用到缓存的地方比如说，本地缓存、容灾缓存，性能缓存等名词，这些缓存的思想无非就是把计算好的存起来，真正的流量过来的时候，直接使用缓存好的内容，能提服务响应高速度。

总结下来：

1.复用之前的内容，不用每次新建

2.提前准备好，不用临时创建

3.采用性能高的存储做缓存，更加提高响应速度 其实看下来跟我们的对象池，没什么区别，我们对象池也就是复用之前创建好的对象。

最后发散下思想，影响我们的程序性能的有以下几个，存储、计算、网络等，其实都可以做缓存，或者提前准备好，亦或者复用之前的结果。我们程序中很多init的东西不就是提前准备好的存储吗，我们很多做的local cache其实就是减少网络传输时间等，以后优化服务性能可以从这个角度考虑。

### go1.12 原理

我们先看下如何使用如下结构体

```text
package main

import (
    "sync"
    "fmt"
    )

type item struct {
    value int
}

func main(){
    pool:= sync.Pool{
        New: func() interface{} {
            return item{}
        },
    }
    pool.Put(item{value:1})
    data := pool.Get()
    fmt.Println(data)
}
```

看起来使用方式很简单，创建一个对象池的方式传进去一个New对象的函数，然后就是两个函数获取对象Get和放入对象Put。

想彻底搞明白原理，莫过于直接去读源码。 先瞅一眼结构体

```text
type Pool struct {
    noCopy noCopy
    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
    localSize uintptr // size of the local array
    New func() interface{}
}
```

New函数不必细说，创建新对象使用。这里主要是local和localSize，看注释知道，这实际是指向数组的地址，该数组实际的类型[P]poolLocal，至于P的含义实际上是goroutine调度里面的一个概念，每个goroutine都会必须要绑定一个P才能得以执行，每个P都有一个待执行的goroutine队列，P的个数一般设置的跟CPU核数相等，详情可以看 [一个EOF引发的探索之路之五（goroutine的调度原理尝试解释篇）](https://zhuanlan.zhihu.com/p/53688556) ，下面的localSize就是该数组的长度。再来看看poolLocal结构体

```text
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
type poolLocalInternal struct {
    private interface{}   // Can be used only by the respective P.
    shared  []interface{} // Can be used by any P.
    Mutex                 // Protects shared.
}
```

poolLocal里面有包了一层结构体poolLocalInternal，里面三个字段private私有字段，shared共享数组，Mutex是的改结构体可以加锁。 这些结构体中的字段具体含义、使用方式时怎样的，我们还是根据Get和Put函数来看比较好

```text
func (p *Pool) Get() interface{} {
    if race.Enabled {
        race.Disable()
    }
    l := p.pin()
    x := l.private
    l.private = nil
    runtime_procUnpin()
    if x == nil {
        l.Lock()
        last := len(l.shared) - 1
        if last >= 0 {
            x = l.shared[last]
            l.shared = l.shared[:last]
        }
        l.Unlock()
        if x == nil {
            x = p.getSlow()
        }
    }
    if race.Enabled {
        race.Enable()
        if x != nil {
            race.Acquire(poolRaceAddr(x))
        }
    } 
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

-   首先是race的设置，看代码是不允许检测，本文不做细说。
-   调用pin函数获取一个poolLocal，它就是本地的，怎么理解呢，我们之前说过每个goroutine都会绑定一个P才能执行，这里的本地其实指的就是，当前goroutine所在的P所属的，获取的详细过程我们稍后看。
-   把l.private赋予x，然后清空该字段，很好理解，我们不是要Get一个对象吗，首先从本地取出一个对象，并且是本地的私有对象，私有的含义，顾名思义只能本地的goroutine能这么用。
-   这里有一个runtime_procUnpin()操作，先暂时不管，稍后我们细说。
-   如果如果x为nil无非就是我们的本地对象为空，如果不了解goroutine调度的可能会有些疑惑，这里解释下我们之前说过每个P会有一个待执行goroutine队列，所以可能g1把私有对象获取后置空，g2再去获取必然拿到是nil。ok，我们继续看如果发现本地私有对象为空，怎么做：
-   首先给该pool加锁，为啥呢，等会我们看看就知道了。
-   拿到pool的shared数组的最后一个元素index，看到这里大概也明白了为啥刚才会对pool加锁，shared相对于刚才的private，即为共享给其他的P的数组，所以可能会有并发情况，加锁也就是必要的了。
-   如果shared数组非空，则取出最后一个元素赋给x，并且从数组中删除。
-   解锁后，再次判断如果刚才shared数组中也没有元素，则调用getSlow()函数获取，看着函数名字就知道获取对象不容易，比较慢，该函数我们稍后再说，大概其实就是从其他的P的poolLocal中盗取一个，感觉跟goroutine调度的思想有点相似。
-   如果最后搞了半天还是没找到可复用的对象，并且New函数非空，则直接New一个新的返回。

其实看到这里基本明白之前的几个字段，Pool中的local就是指向一个数组，该数组的元素就是poolLocal，并且每个P一个，而每个P的poolLocal有两个字段，private，就是所属本地P的goroutine拿对象的时候，首先从该字段获取，如果没有则从另外一个共享的对象数组shared获取，获取的时候别忘记加锁。

ok，我们接下来把刚才看代码遗留的几个问题一一描述 我们来看看pin函数是如何获取本地的poolLocal的：

```text
func (p *Pool) pin() *poolLocal {
    pid := runtime_procPin()
    s := atomic.LoadUintptr(&p.localSize) // load-acquire
    l := p.local                          // load-consume
    if uintptr(pid) < s {
        return indexLocal(l, pid)
    }
    return p.pinSlow()
}
```

-   其实我们刚才已经看到一个runtime_procPin()其实就是跟这里的runtime_procUnpin()配成一对，我们来详细说下这个函数到底在干啥。
-   pid := runtime_procPin()，看代码就知道它返回的是P的id，但只是这样吗？我们还是去runtime里面瞅瞅源码吧。

```text
//go:linkname sync_runtime_procPin sync.runtime_procPin
             //go:nosplit
         func sync_runtime_procPin() int {
                return procPin()
        }
              //go:linkname sync_runtime_procUnpin sync.runtime_procUnpin
             //go:nosplit
        func sync_runtime_procUnpin() {
                procUnpin()
        }
            //go:nosplit
        func procPin() int {
            _g_ := getg()
            mp := _g_.m

            mp.locks++
            return int(mp.p.ptr().id)
        }
              //go:nosplit
        func procUnpin() {
            _g_ := getg()
            _g_.m.locks--
        }
```

如果你了解goroutine的调度原理，就容易理解些，这里procPin函数实际上就是先获取当前goroutine，然后对当前协程绑定的线程（即为m）加锁，即mp.locks++，然后返回m目前绑定的p的id。这个所谓的加锁有什么用呢？这个理就涉及到goroutine的调度了，系统线程在对协程调度的时候，有时候会抢占当前正在执行的协程的所属p，原因是不能让某个协程一直占用计算资源，那么在进行抢占的时候会判断m是否适合抢占，其中有一个条件就是判断m.locks==0，ok，看起来这个procPin的含义就是禁止当前P被抢占。相应的，procUnpin就是解锁了呗，取消禁止抢占。

-   那么我们来看下，为何要对m设置禁止抢占呢？其实所谓抢占，就是把m绑定的P给剥夺了，其实我们后面获取本地的poolLocal就是根据P获取的，如果这个过程中P突然被抢走了，后面就乱套了，我们继续看是如何获取本地的poolLocal的。
-   获取pool的localSize大小，这里加了一个原子操作atomic.LoadUintptr来获取，为什么呢？核心来说其实就是这个localSize有可能会存在并发读写的情况，而且我们的赋值语句并非一个原子操作，有可能会读取到中间状态的值，这是不可接受的。
-   pool的poolLocal数组地址赋给了l
-   然后比较pid与s的大小，如果小于s则直接indexLocal访问即可，否则直接调p.pinSlow函数。我们知道，s代表poolLocal数组大小，并且每个P拥有其中一个元素，看着代码我们知道pid的取值范围就是0~N
-   我们先来看看indexLocal是如何获取本地的poolLocal的

```text
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
            lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
            return (*poolLocal)(lp)
        }
```

很简单做根据首地址，再根据index做下地址转换，然后转成poolLocal地址。返回即可

-   我们继续看下，如果数组size不够大时候的pinSlow函数

```text
func (p *Pool) pinSlow() *poolLocal {
            // Retry under the mutex.
            // Can not lock the mutex while pinned.
            runtime_procUnpin()
            allPoolsMu.Lock()
            defer allPoolsMu.Unlock()
            pid := runtime_procPin()
            // poolCleanup won't be called while we are pinned.
            s := p.localSize
            l := p.local
            if uintptr(pid) < s {
                return indexLocal(l, pid)
            }
            if p.local == nil {
                allPools = append(allPools, p)
            }
            // If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
            size := runtime.GOMAXPROCS(0)
            local := make([]poolLocal, size)
            atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
            atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
            return &local[pid]
        }

        var (
        allPoolsMu Mutex
        allPools   []*Pool
        )
```

这里比较有意思，我们看首先把m的锁给解了，表示可以抢占了，为什么呢，继续看allPoolsMu.Lock()，allPoolsMu又是什么鬼，看如上代码中最后几行所示，我们会维护一个全局的allPools，所有的对象池都会在这个数组里，我们对它加锁看代码应该是要修改这个数组，对这样一个全局对象加锁可能会等待一阵子，等待的这段时间里如果占着P不放，有点浪费资源的意思了。

-   _如果对全局pool加锁成功后，接下来又获取一遍pid并设置非抢占，很简单，刚才已经放开了禁止抢占的标示，这段时间里，p可能会被抢，当前的p跟之前的不一样了。_
-   然后又来了一遍读取localSize和local地址，这里该问了为啥不跟上面一样搞个原子操作呢？其实没必要了，我们已经对全局allPoolsMu加锁了，已经不会存在并发读写的可能了。
-   _继续看又判断了一遍 uintptr(pid) < s，如果小于的话，直接调用indexLocal返回，这里会有疑问，为啥还要判断呢？刚才不就是已经判断过了吗？实际上还是跟刚才解除了禁止抢占，等现在再拿到pid的时候可能不会之前的pid了。_
-   接下来如果local这个指针是空的，放入allPools中，说明是pool的第一次Get。 _接下来获取P的数量赋予size_
-   创建一个size大小的数组，类型就是poolLocal
-   _把local数组的首地址放入我们的pool的local中，看起来这是重新创建的啊，看注释说是两次GC之前需要重新创建，看起来GC会把pool的local清空，稍后我们细说_
-   设置size
-   返回根据pid的poolLocal

从其他的P所属的poolLocal中盗取一个对象

```text
func (p *Pool) getSlow() (x interface{}) {
    // See the comment in pin regarding ordering of the loads.
    size := atomic.LoadUintptr(&p.localSize) // load-acquire
    local := p.local                         // load-consume
    // Try to steal one element from other procs.
    pid := runtime_procPin()
    runtime_procUnpin()
    for i := 0; i < int(size); i++ {
        l := indexLocal(local, (pid+i+1)%int(size))
        l.Lock()
        last := len(l.shared) - 1
        if last >= 0 {
            x = l.shared[last]
            l.shared = l.shared[:last]
            l.Unlock()
            break
        }
        l.Unlock()
    }
    return x
}
```

-   首先重新获取pool的localSize大小并且是原子操作，原因之前描述。
-   获取pool的poolLocal数组
-   为了获取pid，调了runtime_procPin，但又赶紧runtime_procUnpin把locks减一恢复，因为它就是单纯为了获取pid。
-   遍历size大小，找到别的P所属的poolLocal，然后对l加锁，原因是我们在使用shared数组，这个数组会有并发问题
-   然后获取最后一个元素，如果成功，则直接返回，否则继续
-   如果到最后还没拿到则直接返回一个nil 注意：之前看到这里会有一个小疑问，在循环遍历每个P所属的poolLocal的时候，会有可能遇到GC，因为我们已经runtime_procUnpin把禁止抢占标识释放了，这是很有可能的，我们知道GC会把pool清空，那么我们在循环里这样直接访问l := indexLocal(local, (pid+i+1)%int(size))，会访问到非法地址的，这岂不是大bug！！！我以为我找到了重大bug，仔细想想，我想多了，我们知道golang的GC采用三色标记法，第一步就是mark阶段，mark的大体思路就是看看每个对象是否有人引用，我们仔细看看代码

```text
local := p.local
```

这句话实际上把那块内存地址赋给了local，GC的时候即使是清空pool里面的所有地址为nil，但是实际那块地址被local指向了，GC还是不会把它清空的，白兴奋一场。 ok，终于把Get看完了，接下来继续Put

```text
func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }
    if race.Enabled {
        if fastrand()%4 == 0 {
            // Randomly drop x on floor.
            return
        }
        race.ReleaseMerge(poolRaceAddr(x))
        race.Disable()
    }
    l := p.pin()
    if l.private == nil {
        l.private = x
        x = nil
    }
    runtime_procUnpin()
    if x != nil {
        l.Lock()
        l.shared = append(l.shared, x)
        l.Unlock()
    }
    if race.Enabled {
        race.Enable()
    }
}
```

看过Get再看Put，就简单的不能再简单了。

-   _如果放进来的对象为nil，直接返回_
-   获取本地的poolLocal，我们知道如果最初没有，会直接创建新的
-   _如果本地poolLocal的private对象是空的，则优先放入这里，然后把x置空，后面会用到_
-   前面调用的pin函数会设置禁止抢占，这里会取消。
-   如果x为非nil，代表刚才存入private失败，则直接放入本地poolLocal的shared数组中，这里要加锁，因为有并发的可能。

对象池的清除

在刚才代码中，我们多次描述到pool的清空操作，那么什么会清除pool呢？首先，不清除行吗？答案很显然不行，如果一直不清楚，内存会一直增加，有内存溢出的风险。那么什么时候清除呢？

```text
func poolCleanup() {
    for i, p := range allPools {
        allPools[i] = nil
        for i := 0; i < int(p.localSize); i++ {
            l := indexLocal(p.local, i)
            l.private = nil
            for j := range l.shared {
                l.shared[j] = nil
            }
            l.shared = nil
        }
        p.local = nil
        p.localSize = 0
    }
    allPools = []*Pool{}
}
```

首先pool.go中有这个清空pool的函数，很粗暴，直接所有指针清空。 那么什么时候调用呢？

```text
// Implemented in runtime.
func runtime_registerPoolCleanup(cleanup func())
```

有看到了这行代码以及注释，看起来这个是注册清空函数的，链接到了runtime中去了，去runtime/mgc.go中找了找，果然有收获

```text
func sync_runtime_registerPoolCleanup(f func()) {
    poolcleanup = f
}

func clearpools() {
    // clear sync.Pools
    if poolcleanup != nil {
        poolcleanup()
    }
    //下面代码不在本文关注范围（其实是没时间看了）
    ....
}
```

注册函数把函数赋给了poolcleanup了，clearpools调用的它，继续追下代码

```text
func gcStart(trigger gcTrigger) {
    mp := acquirem()
    if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
        releasem(mp)
        return
    }
    releasem(mp)
    ...//太长省略
    gcBgMarkStartWorkers()

    gcResetMarkState()
    ...//太长省略
    systemstack(stopTheWorldWithSema)
    systemstack(func() {
        finishsweep_m()
    })
        // clearpools before we start the GC. If we wait they memory will not be
    // reclaimed until the next GC cycle.
    clearpools()
    ...//太长省略
}
```

简单扫一眼就知道，很明显是GC之前先把pool清空。ok，这次明白网上很多文章的每次GC都会把pool清空的出处了

### 遇到的问题

1.如何禁止抢占P 我们前面一直在说禁止抢占P，其中有个方法就是runtime_procPin函数，到底是怎么搞的呢？ 熟悉goroutine的调度的知道，goroutine的调度有抢占式的，和陷入阻塞然后被调度的，其中抢占式的，就是防止某些协程一直占用P，使得其他的协程陷入饥饿。 那么这种抢占式调度是如何进行呢，直接看下我们熟悉的retake函数的一小段代码

```text
} else if s == _Prunning {
            // Preempt G if it's running for too long.
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
                continue
            }
            if pd.schedwhen+forcePreemptNS > now {
                continue
            }
            preemptone(_p_)
        }
```

什么时候抢占，就是你占用的时间超过我给你的时间片，就直接调用preemptone，给你抢占了。 看下这个函数

```text
func preemptone(_p_ *p) bool {
    mp := _p_.m.ptr()
    if mp == nil || mp == getg().m {
        return false
    }
    gp := mp.curg
    if gp == nil || gp == mp.g0 {
        return false
    }
    gp.preempt = true

    // Every call in a go routine checks for stack overflow by
    // comparing the current stack pointer to gp->stackguard0.
    // Setting gp->stackguard0 to StackPreempt folds
    // preemption into the normal stack overflow check.
    gp.stackguard0 = stackPreempt
    return true
}
```

其实核心就是把当前goroutine的preempt设置为true，然后设置goroutine的stackguard0为stackPreempt，看注释，这样会使得协程做栈溢出检测时，检测出有益处，会做栈扩容嘛？我们去看看栈扩容函数，在stack.go中

```text
func newstack() {
    ...
    if preempt {
        if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
            // Let the goroutine keep running for now.
            // gp->preempt is set, so it will be preempted next time.
            gp.stackguard0 = gp.stack.lo + _StackGuard
            gogo(&gp.sched) // never return
        }
    }
    ....
}
```

找到了如上一段代码，看起来终于对上了，如果locks !=0的话，会继续让该协程执行，其实newstack函数中还有一大段代码，是真正的抢占P的代码，在上面这段代码的下面执行。这里也明白了，发生抢占的方式，其实是给协程设置栈溢出，然后新建栈的时候，再去处理是否要发生抢占。

2.为何禁止抢占P就能防止GC，进而防止清空pool 我们再回头去看下gcStart函数，看下stopTheWorldWithSema，其实这个就是GC的STW阶段，进去看下该函数

```text
func stopTheWorldWithSema() {
    ....
    preemptall()
    // stop current P
    _g_.m.p.ptr().status = _Pgcstop // Pgcstop is only diagnostic.
    sched.stopwait--
    // try to retake all P's in Psyscall status
    for _, p := range allp {
        s := p.status
        if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) {
            if trace.enabled {
                traceGoSysBlock(p)
                traceProcStop(p)
            }
            p.syscalltick++
            sched.stopwait--
        }
    }
    // stop idle P's
    for {
        p := pidleget()
        if p == nil {
            break
        }
        p.status = _Pgcstop
        sched.stopwait--
    }
    wait := sched.stopwait > 0
    unlock(&sched.lock)

    // wait for remaining P's to stop voluntarily
    if wait {
        for {
            // wait for 100us, then try to re-preempt in case of any races
            if notetsleep(&sched.stopnote, 100*1000) {
                noteclear(&sched.stopnote)
                break
            }
            preemptall()
        }
    }

    // sanity checks
    bad := ""
    if sched.stopwait != 0 {
        bad = "stopTheWorld: not stopped (stopwait != 0)"
    } else {
        for _, p := range allp {
            if p.status != _Pgcstop {
                bad = "stopTheWorld: not stopped (status != _Pgcstop)"
            }
        }
    }
    if atomic.Load(&freezing) != 0 {
        lock(&deadlock)
        lock(&deadlock)
    }
    if bad != "" {
        throw(bad)
    }
}
```

这段代码看起来比较清晰，首先会调用preemptall，看名字就能理解，把所有的P都抢过来，但注意，抢占的方式就是设置栈溢出标志，不一定能抢成功喔

```text
func preemptall() bool {
    res := false
    for _, _p_ := range allp {
        if _p_.status != _Prunning {
            continue
        }
        if preemptone(_p_) {
            res = true
        }
    }
    return res
}
```

接下来把本地的P状态设置为stop，然后遍历所有P，如果陷入syscall即为系统调用的，则直接设置为stop，然后循环获取闲置的P，置为stop。最后再看看还有没有需要等待stop，如果有则循环等待，并循环调用preemptall尝试抢占。 看到这里，我们也大概明白了，如果设置禁止抢占标记，STW就会一直等待，GC无法进行下去，自然也无法clearpool了。

3.noCopy是什么

我们注意到，pool这个结构体中有一个字段叫noCopy，类型名字也是noCopy，看代码发现这是个空结构体

```text
type noCopy struct{}
```

看这个名字，我们能猜到这是不让拷贝的，那么怎么做到的呢？然后去晚上查了查，如下解释： Go中没有原生的禁止拷贝的方式，所以如果有的结构体，你希望使用者无法拷贝，只能指针传递保证全局唯一的话，可以这么干，定义 一个结构体叫 noCopy，实现如下的接口，然后嵌入到你想要禁止拷贝的结构体中，这样go vet就能检测出来。

```text
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

注意喔，go vet能检测出来，就代表如果你不用go vet检测，代码也能跑的。 本着提问题原则，先问是不是，再问为什么。 写个例子试试。

```text
package main

import (
    "fmt"
)

type noCopy struct{}

func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

type item struct {
    noCopy noCopy
    value  int
}

func A(a item) {
    fmt.Println(a)
}

func main() {
    var a item
    A(a)
}
```

go vet main.g果然是这样

```text
command-line-arguments
./main.go:17:10: A passes lock by value: command-line-arguments.item contains command-line-arguments.noCopy
./main.go:18:14: call of fmt.Println copies lock value: command-line-arguments.item contains command-line-arguments.noCopy
./main.go:23:4: call of A copies lock value: command-line-arguments.item contains command-line-arguments.noCopy
```

改为传递指针就不会报错了。 ok，那么我们的对象池为啥要禁止拷贝呢，其实仔细回想下也能明白，pool的使用是基于各个协程之间的，相互偷对象又加锁啥的。最重要的，我们GC要保证pool的字段情况，如果你突然来个copy，这个pool清空了，拷贝的pool却没有被清掉，这样的话pool里面指针所指向的对象岂不是不会被GC掉，这可是个大问题。

4.伪共享 仔细看我们的poolLocal结构体，会发现有一个pad字段 类型为

```text
[128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
```

为啥要这么搞呢，其实就是为了防止伪共享。网上有关伪共享的解释有很多，我简单总结下， 我们知道从处理访问到内存，中间有好几级缓存，而这些缓存的存储单位是cacheline，也就是说每次从内存中加载数据，是以cacheline为单位的，这样就会存在一个问题，如果代码中的变量A和B被分配在了一个cacheline，但是处理器a要修改变量A，处理器b要修改B变量。此时，这个cacheline会被分别加载到a处理器的cache和b处理器的cache，当a修改A时，缓存系统会强制使b处理器的cacheline置为无效，同样当b要修改B时，会强制使得a处理器的cacheline失效，这样会导致cacheline来回无效，来回从低级的缓存加载数据，影响性能。

看了如上的原理，我们知道只有在并发比较严重的时候才可能会发生如上的情况，很显然我们的poolLocal满足这样的条件。 那么我们该怎么做呢？方法就是让这个变量不要跟其他变量分配在一个cacheline，让它占满一个cacheline，不够的话补上即可。 想看伪共享，推荐一篇文章 [伪共享(False Sharing)](https://link.zhihu.com/?target=http%3A//ifeve.com/falsesharing/) 我看了下我的64位开发机的cacheline大小为64

```text
jinzhongwei@XXX~ cat /sys/devices/system/cpu/cpu1/cache/index0/coherency_line_size
jinzhongwei@XXX~ 64
```

我看代码中补齐了128个字节的整数倍，看起来足够了。

### go1.13 原理

1.12的时候，会有一些问题，其中最大的问题就是经常清空我们的pool，使得我们经常new新对象，以及复用别的P的shared对象的时候，会经常加锁，性能都会有一些损耗，go1.13对这些问题都有了新的解决办法。 同样的我们先看下结构体

```text
type Pool struct {
    noCopy noCopy

    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
    localSize uintptr        // size of the local array

    victim     unsafe.Pointer // local from previous cycle
    victimSize uintptr        // size of victims array
    New func() interface{}
}
```

看pool的结构体，比之前多了两个字段victim和victimSize，具体作用，我们等会看代码 先来看下Get代码

```text
func (p *Pool) Get() interface{} {
    if race.Enabled {
        race.Disable()
    }
    l, pid := p.pin()
    x := l.private
    l.private = nil
    if x == nil {
        x, _ = l.shared.popHead()
        if x == nil {
            x = p.getSlow(pid)
        }
    }
    runtime_procUnpin()
    if race.Enabled {
        race.Enable()
        if x != nil {
            race.Acquire(poolRaceAddr(x))
        }
    }
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

看起来跟原理的流程没太大变化

-   _第一步，获取poolLocal，我看了下pin函数，跟原先的代码无任何变化。_
-   获取本地的private对象，然后清空本地对象
-   _如果本地对象是空的，则直接获取本地poolLocal的shared中的元素，这里会有变化了，我们稍后细看_
-   如果还是空的，则调用getSow，注意这里直接把pid传进去了，之前的逻辑是在getSlow里面重新判断当前P的，原因是什么呢？仔细看上面的代码，比之前少了一步 runtime_procUnpin()，那么在调用pin()的时候会把当前P锁住，所以可以直接使用，至于原因，我们稍后细说
-   _解锁P_
-   如果拿到的数据还是nil，则直接创建新的

看起来整体思路没太大，我们看下一些不同，首先看下刚才的popHead函数

```text
type poolLocalInternal struct {
    private interface{} 
    shared  poolChain   
}

type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

我们来看下这个shared数组的类型是poolChain

```text
type poolChain struct {
    head *poolChainElt
    tail *poolChainElt
}
type poolChainElt struct {
    poolDequeue
    next, prev *poolChainElt
}
type poolDequeue struct {
    headTail uint64
    vals []eface
}
```

之前的shared是一个数组，现在改成了一个双向链表，继续来看代码 我们来看下popHead函数

```text
func (c *poolChain) popHead() (interface{}, bool) {
    d := c.head
    for d != nil {
        if val, ok := d.popHead(); ok {
            return val, ok
        }
        // There may still be unconsumed elements in the
        // previous dequeue, so try backing up.
        d = loadPoolChainElt(&d.prev)
    }
    return nil, false
}
```

看如上代码，是从head获取，调用poolDequeue的popHead函数获取对象，如果没有获取，则根据双线链表找到pre节点，继续获取，直到遍历到最后一个节点则直接返回 我们来看下poolDequeue的popHead

```text
func (d *poolDequeue) popHead() (interface{}, bool) {
    var slot *eface
    for {
        ptrs := atomic.LoadUint64(&d.headTail)
        head, tail := d.unpack(ptrs)
        if tail == head {
            return nil, false
        }
        head--
        ptrs2 := d.pack(head, tail)
        if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
            slot = &d.vals[head&uint32(len(d.vals)-1)]
            break
        }
    }

    val := *(*interface{})(unsafe.Pointer(slot))
    if val == dequeueNil(nil) {
        val = nil
    }
    *slot = eface{}
    return val, true
}
```

-   这里是循环获取
-   首先拿到poolDequeue的headTail，我们来说下这个结构体，它是一个ring数组，底层采用slice，headTail指的就是ring的首尾位置，放在了一个uint64字段里面，主要是为了方便整体进行原子操作，为了方便，还专门搞了两个函数分别是解析出两个值，以及合并在一起

```text
func (d *poolDequeue) unpack(ptrs uint64) (head, tail uint32) {
    const mask = 1<<dequeueBits - 1
    head = uint32((ptrs >> dequeueBits) & mask)
    tail = uint32(ptrs & mask)
    return
}

func (d *poolDequeue) pack(head, tail uint32) uint64 {
    const mask = 1<<dequeueBits - 1
    return (uint64(head) << dequeueBits) |
        uint64(tail&mask)
}
```

看起来就是通过位操作从headTail中拆分出head和tail

-   _如果head和tail指向同一个位置，则代表当前ring是空的，直接返回false_
-   对head减一，然后pack出新的headTail
-   _接下来调用原子操作CompareAndSwapUint64来更新headTail，注意这个函数的含义是，如果headTail还是我们最初读取的ptrs的话，我们把他更新为ptrs2，什么意思呢？其实就是说我们从读取headTail到此时这段时间内，没有其他协程对headTail操作的话，我们才会真实的更新headTail，否则继续循环，尝试操作，如果成功，则赋值头部对象为slot，break循环。_
-   把slot转成interface{}类型的value
-   _注意到最后一步，_slot = eface{}，是把ring数组的head位置置空，但是置空的方式时为空eface，源码注释中写道：这里清空的方式与popTail不同，这里与pushHead没有竞争关系，所以不用太小心。我们稍后再解释原因。

我们在回过头看最初的Get函数中，如果从自己的shared中还是无法获取到对象，调用的getSlow，go1.12的时候，getSlow是从其他的P的shared数组中获取对象，那么我们来看下go1.13有什么不同

```text
func (p *Pool) getSlow(pid int) interface{} {
    size := atomic.LoadUintptr(&p.localSize) 
    locals := p.local                        
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i+1)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }
    size = atomic.LoadUintptr(&p.victimSize)
    if uintptr(pid) >= size {
        return nil
    }
    locals = p.victim
    l := indexLocal(locals, pid)
    if x := l.private; x != nil {
        l.private = nil
        return x
    }
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }

    atomic.StoreUintptr(&p.victimSize, 0)

    return nil
}
```

-   获取p.localSize大小和p.local地址，跟之前没区别
-   开启一个循环，不断的从其他p的local中获取对象，跟之前唯一的区别是调用shared.popTail()，我们稍后再看
-   注意，接着发现如果从别的P的shared还是无法获取的话，会有其他操作，涉及到pool中的我们之前看到的两个字段，victimSize和victim，看他们的类型与local和localSize一样，这其实是引用一种算法，来减少我们的GC，我们稍后细说，我们先看看使用的方式
-   获取victimSize大小，victim赋予locals，pid小于size，直接返回
-   获取本地poolLocal，看本地private字段，如有有则直接获取，没有继续
-   从shared中获取，注意这里直接pid+i，而不是pid+i+1开始，也就是说不区分本地shared了，获取方式同样是shared.popTail()，看到这里你会发现这个从victim获取对象的方式，跟之前local没什么区别，原因是什么，我们还是后面说。
-   注意最后一行代码，你会发现当从victim也无法获取对象的时候，会直接设置victimSize为0，这是为什么呢？我们也是后面细说

接下来我们看下popTail函数

```text
func (c *poolChain) popTail() (interface{}, bool) {
    d := loadPoolChainElt(&c.tail)
    if d == nil {
        return nil, false
    }

    for {
        d2 := loadPoolChainElt(&d.next)

        if val, ok := d.popTail(); ok {
            return val, ok
        }

        if d2 == nil {
            return nil, false
        }

        if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {

            storePoolChainElt(&d2.prev, nil)
        }
        d = d2
    }
}
```

-   获取到链表的尾部指针d
-   如果尾部指针为空，则直接返回
-   接着for循环，首先遍历d的next对象，调用d.popTail()获取对象，如果获取成功，则直接返回，否则判断next对象是否为空，如果为空，则直接返回false，如果next对象不为空，则直接把刚才的d从链表中删除，继续从刚才的next对象获取对象，以此类推，具体的d.popTail()我们稍后在看。

第一次看到上面的代码，我是一脸懵逼的，我根据自己的疑问来说明上面代码的含义，为啥这么写。

-   _既然d已经是链表的尾部指针了，为啥还要访问它的next？ 我以为有什么骚操作，实际上不是的，链表的head到tail方向是prev指向的方向，这跟我理解的正常的链表指向正好相反，看完之后笑哭，回过头看popHead函数，发现它的遍历方向是prev，正好对应上。_
-   为啥无法从d中获取到对象，并且对d做删除时还要原子操作？ 这个理解就比较简单，为啥从尾部的节点无法获取到对象，因为可能尾部节点是空的啊，那空的为啥没被删除呢？如果上一次get从这个尾部节点get成功，根据代码看并没有直接删除节点，直接返回了，等着下次删除呢，至于为啥会原子操作，因为此时可能有多个协程从这个链表的尾部节点获取对象，因为是steal别的P的嘛，尾部节点为空的情况下，大家都去删除，要保证只有一个删除成功的。
-   _d2位空为啥直接返回呢？_ _我们遍历节点是从尾部节点向头部节点遍历，当遍历到nil节点肯定无法再，因为到头了。_
-   看起来这里是收缩链表的地方，那么popHead为啥没有收缩链表的操作呢？ 回过头看看popHead的操作，很简单就是不断的遍历链表节点，不断的获取对象，直到获取成功，或者是遍历到nil节点。其实也没必要，尾部删除节点就够了，最主要的是我们还要从头部push节点，避免出现多协程操作链表，比如你把头部节点删除了，我此时正好要往里面插入数据怎么办，会更麻烦。

ok，我们再来看看那个ring的popTail函数，它其实是lock-free的核心所在

```text
func (d *poolDequeue) popTail() (interface{}, bool) {
    var slot *eface
    for {
        ptrs := atomic.LoadUint64(&d.headTail)
        head, tail := d.unpack(ptrs)
        if tail == head {
            return nil, false
        }

        ptrs2 := d.pack(head, tail+1)
        if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
            slot = &d.vals[tail&uint32(len(d.vals)-1)]
            break
        }
    }
    val := *(*interface{})(unsafe.Pointer(slot))
    if val == dequeueNil(nil) {
        val = nil
    }
    slot.val = nil
    atomic.StorePointer(&slot.typ, nil)
    return val, true
}
```

-   popTail和popHead很相似，不同处就在于一个是从ring头部获取对象，一个是尾部。
-   看下代码，首先看到一个for循环，不断尝试获取对象，我们来看下for循环内部
-   首先获取headTail指针，解析成head和tail
-   如果tail跟head相等，ring就是空的，直接返回
-   给tail+1然后pack成新的headTail，采用原子操作atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) 来更新，这个函数的含义是如果当前的headTail是ptrs，那么就更新成ptrs2，如果成功，则赋值当前数据给slot，否则继续循环。
-   跳出循环后，代表我们拿到数据了，则转化成interface{},然后清空slot，清空的方式时给eface的value和type都置为nil，注意它分了两步第一步，先清空value，然后再用原子操作把type置为nil，这跟popHead的直接使用如下方式 go *slot = eface{} 不太一样，同样的，留着这个问题，我们看完pool的Put函数再说这个问题。

接下来，我们看之前遗留的一个问题，victim和victimSize到底是用来干嘛的，此处声明下，我引用了网上查询的一些资料，整理成自己理解的文字。 其实这是叫victim cache的机制。victim cache原是CPU硬件处理缓存的一种技术

> 所谓受害者缓存（Victim Cache），是一个与直接匹配或低相联缓存并用的、容量很小的全相联缓存。当一个数据块被逐出缓存时，并不直接丢弃，而是暂先进入受害者缓存。如果受害者缓存已满，就替换掉其中一项。当进行缓存标签匹配时，在与索引指向标签匹配的同时，并行查看受害者缓存，如果在受害者缓存发现匹配，就将其此数据块与缓存中的不匹配数据块做交换，同时返回给处理器。 -- 维基百科

所以这已经是成熟的解决算法了，go1.13只是引用进来了，我们来看下是怎么做的，以及这么做的好处是什么呢？

-   _首先我们之前是把pool全部清除，这就导致了每次都要大量重新创建对象会造成短暂的GC压力，而此次我们只会清空victime cache，清空的量就会变少，清空victime cache后，我们再把原先的cache的内容作为新的victime cache。_
-   回顾我们之前GetSlow，如果从别的P的shared中还拿不到对象，则直接去victime cache拿对象，用完之后肯定再Put到我们的原先的cache中去 了，这样有什么好处呢？如果我们的Get速度和Put速度比较相似，及时每次GC的时候，都会把老的cache置为victime cache，但是很快Get就会从victime cache那出来了（敢在下次GC之前），及时Get的速度降下来，此时对象会有一部分在victime cache，一部分在localCache，会变成分两次GC，第一次直接清空victime cache，然后localCache置为victime cache，第二次GC再把刚才的victime cache清空。
-   最后我们想下，这种思想有点像分代垃圾回收的思想，分代垃圾回收其实就是把生命周期短的对象回收，尽量保留生命周期长的对象。详情可以参考[从垃圾回收解开Golang内存管理的面纱之三垃圾回收](https://zhuanlan.zhihu.com/p/53928921)

到此，应该把Get函数看完了，我们再看下Put函数

```text
func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }
    if race.Enabled {
        if fastrand()%4 == 0 {
            // Randomly drop x on floor.
            return
        }
        race.ReleaseMerge(poolRaceAddr(x))
        race.Disable()
    }
    l, _ := p.pin()
    if l.private == nil {
        l.private = x
        x = nil
    }
    if x != nil {
        l.shared.pushHead(x)
    }
    runtime_procUnpin()
    if race.Enabled {
        race.Enable()
    }
}
```

看起来跟之前的逻辑差不太多，具体看下 _对象为空直接返回_ pin获取localcache _如果private为nil，直接放入即可_ 否则push到shared中，调用pushHead(x)

我们来看下这个函数

```text
func (c *poolChain) pushHead(val interface{}) {
    d := c.head
    if d == nil {
        const initSize = 8
        d = new(poolChainElt)
        d.vals = make([]eface, initSize)
        c.head = d
        storePoolChainElt(&c.tail, d)
    }

    if d.pushHead(val) {
        return
    }

    newSize := len(d.vals) * 2
    if newSize >= dequeueLimit {
        newSize = dequeueLimit
    }

    d2 := &poolChainElt{prev: d}
    d2.vals = make([]eface, newSize)
    c.head = d2
    storePoolChainElt(&d.next, d2)
    d2.pushHead(val)
}
```

-   首先我们看到，获取head节点赋予d
-   如果head节点都是空的，则直接创建一个新节点，改节点的ring大小为8，把head指向该节点，并且把tail指针指向该节点，但是这里用了原子操作，为啥呢？很简单我们之前看到poptail的时候，会收缩链表，清除节点的操作，会有竞争关系，所以需要原子操作，那么head指针赋值的为啥不做原子操作，因为之前收缩不会修改head指针啊，没有竞争关系的。
-   接下来往head节点push对象，如果成功，则直接返回
-   如果没成功，代表该节点的ring满了，那么该怎么做，肯定新建节点了，而且这里新建节点的ring长度是原先长度的2倍，并且有个最大长度限制，如下

```text
const dequeueBits = 32
    const dequeueLimit = (1 << dequeueBits) / 4
```

为啥是这个值，目前我还没明白。 _最后把这个节点插入到链表中，采用原子操作原因，还是跟popTail的链表收缩的竞争关系_ 最后再往新建的节点push对象。 我们继续看这个ring的pushHead函数

```text
func (d *poolDequeue) pushHead(val interface{}) bool {
        ptrs := atomic.LoadUint64(&d.headTail)
        head, tail := d.unpack(ptrs)
        if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
            return false
        }
        slot := &d.vals[head&uint32(len(d.vals)-1)]
        typ := atomic.LoadPointer(&slot.typ)
        if typ != nil {
            return false
        }

        if val == nil {
            val = dequeueNil(nil)
        }
        *(*interface{})(unsafe.Pointer(slot)) = val
        atomic.AddUint64(&d.headTail, 1<<dequeueBits)
        return true
    }
```

-   首先获取headTail，然后解析出head和tail
-   判定是ring是否满了，满了直接返回false
-   然后直接获取最ring的head对象，然后注意下，这里的判断typ != nil 正好对应上我们之前的问题，为啥popTail的时候情况对象的方式和popHead不一样，popTail会先把value置为nil，然后原子操作把typ置为nil，上面的代码就是原因，因为popTail和pushHead有竞争关系，而popHead和pushHead没有，稍后我们解释原因。
-   如果type为非nil，说明还未被popTail置空，不能放入，直接返回即可
-   如果type为nil，就说明已经被释放，直接放入即可。

最后我们来解释下遗留的一个问题，为啥pushHead和popTail有竞争的可能，而pushHead和popHead没有竞争可能。 _对于前者好解释，我们pushHead是对本地P的localCache操作，而popTail则是抢其他P的localCache的操作，所以会存在竞争的可能_ 而popHead，思考下，它只会在本地的localCache中才会popHead，只有本地的拿不出来，才会去popTail拿别的P的，同一个P同时跑的goroutine只能是一个，所以肯定不会跟pushHead冲突的。

最后，我们再看下它的clean

```text
func poolCleanup() {
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }
    oldPools, allPools = allPools, nil
}
```

跟我们之前说的一样，清空victim cache，然后把现有的local cache置为victim cache。

到此整个对象池代码看完了，收货颇丰，以下几点

-   _cacheline 伪共享问题_
-   禁止拷贝
-   _lock-free_
-   Victim Cache 机制
-   GC的细节


## 参考链接
https://juejin.cn/post/6963450780166651912
https://www.cnblogs.com/qcrao-2018/p/12736031.html
https://zhuanlan.zhihu.com/p/366399469
https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487149&idx=1&sn=f38f2d72fd7112e19e97d5a2cd304430&source=41#wechat_redirect
https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/
https://zhuanlan.zhihu.com/p/99710992


【欧神 源码分析】[https://changkun.us/archives/2018/09/256/](https://changkun.us/archives/2018/09/256/)

【Go 夜读】[https://reading.hidevops.io/reading/20180817/2018-08-17-sync-pool-reading.pdf](https://reading.hidevops.io/reading/20180817/2018-08-17-sync-pool-reading.pdf)

【夜读第 14 期视频】[https://www.youtube.com/watch?v=jaepwn2PWPk&list=PLe5svQwVF1L5bNxB0smO8gNfAZQYWdIpI](https://www.youtube.com/watch?v=jaepwn2PWPk&list=PLe5svQwVF1L5bNxB0smO8gNfAZQYWdIpI)

【源码分析，伪共享】[https://juejin.im/post/5d4087276fb9a06adb7fbe4a](https://juejin.im/post/5d4087276fb9a06adb7fbe4a)

【golang的对象池sync.pool源码解读】[https://zhuanlan.zhihu.com/p/99710992](https://zhuanlan.zhihu.com/p/99710992)

【理解 Go 1.13 中 sync.Pool 的设计与实现】[https://zhuanlan.zhihu.com/p/110140126](https://zhuanlan.zhihu.com/p/110140126)

【优缺点，图】[http://cbsheng.github.io/posts/golang标准库sync.pool原理及源码简析/](http://cbsheng.github.io/posts/golang%E6%A0%87%E5%87%86%E5%BA%93sync.pool%E5%8E%9F%E7%90%86%E5%8F%8A%E6%BA%90%E7%A0%81%E7%AE%80%E6%9E%90/)

【xiaorui 优化锁竞争】[http://xiaorui.cc/archives/5878](http://xiaorui.cc/archives/5878)

【性能优化之路，自定义多种规格的缓存】[https://blog.cyeam.com/golang/2017/02/08/go-optimize-slice-pool](https://blog.cyeam.com/golang/2017/02/08/go-optimize-slice-pool)

【sync.Pool 有什么缺点】[https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487149&idx=1&sn=f38f2d72fd7112e19e97d5a2cd304430&source=41#wechat_redirect](https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487149&idx=1&sn=f38f2d72fd7112e19e97d5a2cd304430&source=41#wechat_redirect)

【1.12 和 1.13 的演变】[https://github.com/watermelo/dailyTrans/blob/master/golang/sync_pool_understand.md](https://github.com/watermelo/dailyTrans/blob/master/golang/sync_pool_understand.md)

【董泽润 演进】[https://www.jianshu.com/p/2e08332481c5](https://www.jianshu.com/p/2e08332481c5)

【noCopy】[https://github.com/golang/go/issues/8005#issuecomment-190753527](https://github.com/golang/go/issues/8005#issuecomment-190753527)

【董泽润 cpu cache】[https://www.jianshu.com/p/dc4b5562aad2](https://www.jianshu.com/p/dc4b5562aad2)

【gomemcache 例子】[https://docs.kilvn.com/The-Golang-Standard-Library-by-Example/chapter16/16.01.html](https://docs.kilvn.com/The-Golang-Standard-Library-by-Example/chapter16/16.01.html)

【鸟窝 1.13 优化】[https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/)

【A journey with go】[https://medium.com/a-journey-with-go/go-understand-the-design-of-sync-pool-2dde3024e277](https://medium.com/a-journey-with-go/go-understand-the-design-of-sync-pool-2dde3024e277)

【封装了一个计数组件】[https://www.akshaydeo.com/blog/2017/12/23/How-did-I-improve-latency-by-700-percent-using-syncPool/](https://www.akshaydeo.com/blog/2017/12/23/How-did-I-improve-latency-by-700-percent-using-syncPool/)

【伪共享】[http://ifeve.com/falsesharing/](http://ifeve.com/falsesharing/)