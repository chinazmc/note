Go一向以它的简单易学而著称，我也遇到过同学说只需半天就能掌握Go语言了，两三年的Go开发经验就称专家了。

想比较Rust等编程语言，Go语言的确容易上手，查看Go语言的规范你也会看到，它的语言规范相比较其它编程语言非常的简短，的确可以一个小时就可以读完。但是如果你深入了解这门语言(其它语言也是类似情况)，就会发现很多细节的地方，需要花谢心思和时间仔细琢磨琢磨。这不，我先来几道Go并发的源代码考考你，看看你能回答上来不？

## 几道Go并发的源代码问题

### 在另一个goroutine看到的write顺序

下面这段代码，是否可能输出`1, 0`?

```go
package main

import (

	"fmt"

	"sync"

)

func main() {

	var wg sync.WaitGroup

	wg.Add(2)

	var x, y int

	go func() {

		x = 1

		y = 1

		wg.Done()

	}()

	go func() {

		r1 := y

		r2 := x

        fmt.Println(r1, r2)  // ❶ r1 = 1, r2 = 0 可能吗?

		wg.Done()

	}()

	wg.Wait()

}
```

### 两个goroutine交叉read/write

下面这段代码是否可能输出 r1= 0, r2 =0这样的结果？

```go
package main

import (

	"fmt"

	"sync"

)

func main() {

	var wg sync.WaitGroup

	wg.Add(2)

	var x, y int

	go func() {

		x = 1

		r1 := y

		fmt.Println(r1) // ❶

		wg.Done()

	}()

	go func() {

		y = 1

		r2 := x

		fmt.Println(r2) // ❷

		wg.Done()

	}()

	wg.Wait()

}
```

### 两个goroutine看到的write顺序

下面这段代码, ❶和❷可能输出`1,0`和`0,1`这两种不同的结果吗？

```go
package main

import (

	"fmt"

	"sync"

)

func main() {

	var wg sync.WaitGroup

	wg.Add(4)

	var x, y int

	go func() {

		x = 1

		wg.Done()

	}()

	go func() {

		y = 1

		wg.Done()

	}()

	go func() {

		r1 := x

		r2 := y

		fmt.Println(r1, r2) // ❶ 1,0

		wg.Done()

	}()

	go func() {

		r3 := x

		r4 := y

		fmt.Println(r3, r4) // ❷ 0,1

		wg.Done()

	}()

	wg.Wait()

}
```

### sync.Once

下面是Go标准库中`sync.Once`的实现，虽然很简单的几行代码，你品一品，有没有什么疑问？

```go
func (o *Once) Do(f func()) {

	if atomic.LoadUint32(&o.done) == 0 { // ❶

		o.doSlow(f)

	}

}

func (o *Once) doSlow(f func()) {

	o.m.Lock() // ❷

	defer o.m.Unlock() // ❸

	if o.done == 0 { // ❹

		defer atomic.StoreUint32(&o.done, 1) // ❺

		f()

	}

}
```

为什么❹处不使用atomic?那么❶和❺处不使用atomic行不行？

### sync.WaitGroup

下面是一段`WaitGroup`的示例代码, ❶处需要使用atomic.Load吗？ 如果不使用，一定输出10吗？

```go
func main() {

	var count int64

    var wg sync.WaitGroup

	wg.Add(10)

    for i := 0; i < 10; i++ {

		go func() {

			atomic.AddInt64(&count, 1)

			wg.Done()

		}()

	}

    wg.Wait()

	fmt.Println(count) // ❶

}
```

### TryLock

下面这段代码是Mutex的TryLock方法的实现。❶处为什么不使用atomic,会不会导致锁已经释放了但是TryLock获取不到锁？

```go
func (m *Mutex) TryLock() bool {

	old := m.state // ❶

	if old&(mutexLocked|mutexStarving) != 0 {

		return false

	}

	if !atomic.CompareAndSwapInt32(&m.state, old, old|mutexLocked) {

		return false

	}

	return true

}
```

你可以先思考一下这几个问题，我们暂时不分析，接下来我们看看最新的Go内存模型，然后再分析这几个题目。

## Go 内存模型

新版的(Go 1.19)内存模型对原有的内存模型文字做了大幅度的[修改](https://go-review.googlesource.com/c/go/+/381315/):

-   文档化 Go 对内存模型整体的实现方法  
    data race指对同一个内存位置有并发的write-write或者write-read，除非使用了atomic进行原子操作，否则可能就有数据竞争，所以你看啊, read-read并没有数据竞争。没有数据竞争的程序的行为就像所有的goroutine运行在单处理器上，这个属性有时候也叫 `DRF-SC` (data-race-free programs execute in a sequentially consistent manner)。
    
-   文档化 multiword race 可能会导致崩溃
    
-   文档化 `runtime.SetFinalizer`的 happens-before 规则
-   文档化(或者增加[连接](https://go-review.googlesource.com/c/go/+/381316/)) 针对更多同步原语的 happens-before 规则
-   文档化 `sync/atomic`的 happens-before 规则， 对应 C++ 的 顺序一致性 atomic (以及其它语言Java，JavaScript, Rust, Swift, C, ...)
-   文档化 禁止编译器优化

内存操作可以分为三者：

-   read-like: 比如 read、atomic read、mutex lock 和channel receive
-   write-like: 比如 write、atomic write、mutex unlock、channel send 和 channe close
-   read-like和write-like并存: 比如 atomic compare-and-swap

内存模型有提出了两种关系: `sequenced before`和`synchronized before`。  
在一个goroutine中执行的语句和表达式遵循`sequenced before`,按照Go语言规范中定义的控制流程和表达式的求值顺序执行。  
如果一个同步执行的read-like的内存操作**r**观察到一个同步执行的write-like的内存操作**w**,那么w `synchronized before` r。  
而`happens before`关系就被定义为一组`sequenced before`和`synchronized before`构成的具有传递性的组合。

我的理解，Go内存模型相当于把`happens before`关系细化了。

一个非同步的数据读取r, 读取内存地址x,如果写w对r可见，也就是r能观察到w话，需要保证:

-   w happens before r
-   w does not happen before any other write w' (to x) that happens before r (翻译过来还不如看英文哈)

其实我不想讲太多的关于Go内存模型的理论了，毕竟是翻译官方的文档，而且我翻译几次发觉翻译的不好，不如直接看官方文档。另外个人觉得这个官方文档也写得不好，虽然前面有一些定义，但是一些术语或者论据并没有很好的解释或者严格的论证(一家之言)。比如当你说`some`的时候，你要严格定义some到底包含哪些场景。当你说特定的术语或者短语时，你要定义术语或或者增加引用。

不管怎样，重要的，或者说对于Gopher来说容易理解的是，它对各种同步场景给了严格的保证(定义)，比如`init`、goroutine的创建和执行、channel、Mutex、RWMutex、Once以及新增加`SetFinalizer`,但是最最重要的是，终于提供了atomic的同步定义。

### atomic的保证

`sync/atomic`包提供了一组原子操作。如果原子操作`A`能被原子操作`B`观察到，那么我们就说 A synchronized before B。  
所有的程序中的原子操作的执行就像按照顺序一致性顺序执行一样。

和C++中的sequentially consistent atomic， 以及Java的volatile变量一样。

因为不同的CPU指令不同，所以具体的atomic的实现也是针对不同架构各有不同。我们以AMD64架构为例。

对于读操作，比如atomic.Load其实没有做任何处理，因为是single word读取：

```go
func Load(ptr *uint32) uint32 {

	return *ptr

}

func Loadp(ptr unsafe.Pointer) unsafe.Pointer {

	return *(*unsafe.Pointer)(ptr)

}

func Load64(ptr *uint64) uint64 {

	return *ptr

}
```

对于写(读写)操作，使用`LOCK`前缀:

```
TEXT ·Cas64(SB), NOSPLIT, $0-25

	MOVQ	ptr+0(FP), BX

	MOVQ	old+8(FP), AX

	MOVQ	new+16(FP), CX

	LOCK

	CMPXCHGQ	CX, 0(BX)

	SETEQ	ret+24(FP)

	RET
```

LOCK前缀可以确保共享内存可以排他使用，并且对数据的修改会通过MESI对其它处理器可见。LOCK在老的Intel CPU中可能采用锁总线的方式，但是在现代的CPU中，通过锁cache line方式。一个更清楚的解释可以看这篇文章[x86 LOCK prefix](https://hackmd.io/@vesuppi/Syvoiw1f8)。

Store就使用了`XCHG`指令，不需要LOCK前缀，本身它就具有LOCK的功能。

```
TEXT ·Store64(SB), NOSPLIT, $0-16

	MOVQ	ptr+0(FP), BX

	MOVQ	val+8(FP), AX

	XCHGQ	AX, 0(BX)

	RET
```

虽然我们看的是AMD64架构的atomic实现，但是我们不妨看一个x86架构的Load64，这是一个很有意思的知识点。  
x86是32位的架构，word是32位，读取int64的值的时候就是multiword的场景，不使用atomic,就可能读到部分修改的值，所以32位的情况下，原子读取int64的值需要特殊处理一下，不想AMD64架构下直接读取:
```
TEXT ·Load64(SB), NOSPLIT, $0-12

	NO_LOCAL_POINTERS

	MOVL	ptr+0(FP), AX

	TESTL	$7, AX

	JZ	2(PC)

	CALL	·panicUnaligned(SB)

	MOVQ	(AX), M0

	MOVQ	M0, ret+4(FP)

	EMMS

	RET
```

首先它会检查是否是8byte对齐，要是不对齐就会panic。然后呢，使用MMX0寄存器(64bit)巧妙的实现原子读取。

### WaitGroup、Cond、sync.Pool、sync.Map

最新的Go内存模型还在注释(go doc)对各种并发原语的内存模型进行了定义，虽然我觉得在Go内存模型文档中记录更合理一些。

比如WaitGroup:

> In the terminology of the Go memory model, a call to Done “synchronizes before” the return of any Wait call that it unblocks.  
> 在 Go 内存模型的术语中，对 Done 的调用“在”它解除阻塞的任何 Wait 调用返回之前“同步”。

比如 Cond:

> In the terminology of the Go memory model, Cond arranges that a call to Broadcast or Signal “synchronizes before” any Wait call that it unblocks.  
> 在 Go 内存模型的术语中，Cond 安排对 Broadcast 或 Signal 的调用在它解除阻塞的任何 Wait 调用之前“同步”。

比如Pool:

> In the terminology of the Go memory model, a call to Put(x) “synchronizes before” a call to Get returning that same value x. Similarly, a call to New returning x “synchronizes before” a call to Get returning that same value x.  
> 在 Go 内存模型的术语中，对 Put(x) 的调用“先于”对返回相同值 x 的 Get 的调用“同步”。类似地，对 New 的调用返回 x 在对 Get 的调用返回相同值 x 之前“同步”。

比如sync.Map:

> In the terminology of the Go memory model, Map arranges that a write operation “synchronizes before” any read operation that observes the effect of the write.  
> 在 Go 内存模型的术语中，Map 安排写入操作“先于”任何观察写入效果的读取操作“同步”。

比如RWMutex:

> In the terminology of the Go memory model, the n'th call to Unlock “synchronizes before” the m'th call to Lock for any n < m, just as for Mutex. For any call to RLock, there exists an n such that the n'th call to Unlock “synchronizes before” that call to RLock, and the corresponding call to RUnlock “synchronizes before” the n+1'th call to Lock.  
> 在 Go 内存模型的术语中，对 Unlock 的第 n 次调用“同步于”对任何 n < m 的 Lock 的第 m 次调用，就像 Mutex 一样。对于任何对 RLock 的调用，都存在一个 n，使得对 Unlock 的第 n 次调用“先于”对 RLock 的调用“同步”，而对 RUnlock 的相应调用“先于”对 Lock 的第 n+1 次调用“同步”。

比如Mutex:

> In the terminology of the Go memory model, the n'th call to Unlock “synchronizes before” the m'th call to Lock for any n < m. A successful call to TryLock is equivalent to a call to Lock. A failed call to TryLock does not establish any “synchronizes before” relation at all.  
> 在 Go 内存模型的术语中，对 Unlock 的第 n 次调用“同步于”对任何 n < m 的 Lock 的第 m 次调用。成功调用 TryLock 等同于调用 Lock。对 TryLock 的失败调用根本不会建立任何“之前同步”的关系。

比如Once:

> In the terminology of the Go memory model, the n'th call to Unlock “synchronizes before” the m'th call to Lock for any n < m. A successful call to TryLock is equivalent to a call to Lock. A failed call to TryLock does not establish any “synchronizes before” relation at all.  
> 在 Go 内存模型的术语中，对 Unlock 的第 n 次调用“同步于”对任何 n < m 的 Lock 的第 m 次调用。成功调用 TryLock 等同于调用 Lock。对 TryLock 的失败调用根本不会建立任何“之前同步”的关系。

比如runtime.SetFinalizer:

> In the terminology of the Go memory model, a call SetFinalizer(x, f) “synchronizes before” the finalization call f(x). However, there is no guarantee that KeepAlive(x) or any other use of x “synchronizes before” f(x).  
> 在 Go 内存模型的术语中，调用 SetFinalizer(x, f) 在终结调用 f(x) 之前“同步”。但是，不能保证 KeepAlive(x) 或 x 的任何其他使用“在”f(x) 之前“同步”。

为什么要了解这些保证，或者说为什么要了解Go内存模型呢？就是为了能编写data-race-free、没有并发问题的程序。

比如下面一段并发代码：

```go
var a string

var done bool

func setup() {

	a = "hello, world"

	done = true

}

func main() {

	go setup()

	for !done { // 空转，等待done=true

	}

	print(a)

}
```

这里`done`的读写并没有使用atomic进行同步控制，所以在main goroutine中没有保证"a的赋值" happens before "done的赋值",所以这个程序有可能输出空的字符串，甚至极端main goroutine一直观察不到done的赋值导致程序永远没办法完成。

## x86的TSO

不同的CPU架构对memory order的处理是不一样的，而Go内存模型保证的是在所有支持的架构下的一致性。有些并发问题在x86是没问题的，但是在ARM架构下可能会有问题。

我们看看现代的x86的内存模型。下面是它们的一种架构:  
[![](https://colobu.com/2022/09/12/go-synchronization-is-hard/mem-tso.png)](https://colobu.com/2022/09/12/go-synchronization-is-hard/mem-tso.png)

每一个处理器都有自己的写队列，同时都连着一个共享主内存。处理器读的时候先查询自己的写队列，再查询主内存。处理器不会查询其它处理器的写队列。

虽然处理器会看到自己的写，暂时看不到其它处理器的写，但是到了主内存的时候，所有的写都是顺序的，所有的处理器看到的主内存的写都是有序的，所以x86的内存模型也叫做存储全序模型(TSO,total store order)。

对于x86内存模型，只要一个值写入了主内存，未来的读会看到它(直到一个新的写覆盖了它)。

ARM/POWER 使用一种更宽松的内存模型，每个处理器都读写它自己的内存副本，每个写都会独自传播到其它的处理器。在传播的时候允许重排序,并没有像x86 存储全序。  
[![](https://colobu.com/2022/09/12/go-synchronization-is-hard/mem-weak.png)](https://colobu.com/2022/09/12/go-synchronization-is-hard/mem-weak.png)

但是对于ARM/POWER， 不同线程对同一个内存地址的写是全序的，也就是一个线程观察到的同一个内存地址的写和另一个线程观察到的同一个内存地址的写顺序都是一样的，尽管对不同的地址的写观察到的顺序可能不一样，这被称之为**Coherence**。

## 分析上面的问题

啰嗦了这么多，其实还是讲一个memory model的背景,而且这些这些知识也在Russ Cox的硬件内存模型做了介绍，也有很多相关的文章专门介绍memory order等相关的知识。 Russ Cox为了修订Go内存模型，专门对此做了深入的研究，还写了[三篇文章](https://research.swtch.com/mm)对此进行介绍。

以上都是理论，但是Go内存模型对真实的并发程序有什么指导呢？那就逐个分析开头我们提出的几个问题吧。

### 在另一个goroutine看到的write顺序

第一道题，下面这段代码❶是否可能输出`1, 0`?

对于x86架构， 答案是: **否**。  
x86架构保证TSO,存储有序。 对于第一个写goroutine， x,y的写入顺序是保证有序的，写入到第一个线程的write queue保证 x sequenced before y, 又因为TSO存储有序， 其它的goroutine观察到的顺序也是x sequenced before y。

对于ARM架构， 答案是: **是**。 对于第一个写goroutine， x,y的写入顺序是有序的,但是由于arm架构不是TSO， 对x,y的写在传播到其它处理器时，可能会重排序，所以有可能有的处理器看到x,y write,有的看到y,x write。

```go
package main

import (

	"fmt"

	"sync"

)

func main() {

	var wg sync.WaitGroup

	wg.Add(2)

	var x, y int

	go func() {

		x = 1

		y = 1

		wg.Done()

	}()

	go func() {

		r1 := y

		r2 := x

        fmt.Println(r1, r2)  // ❶ r1 = 1, r2 = 0 可能吗?

		wg.Done()

	}()

	wg.Wait()

}
```

### 两个goroutine交叉read/write

第二道题，下面这段代码是否可能速出 r1= 0, r2 =0这样的结果？

正常的理解, `x=1`或者`y=1`先会发生，或者输出结果总会有一个1, 所以`r1= 0, r2 =0`不太可能发生。

**但是**，在x86架构下，两个goroutine(线程)把它们的write放入write buffer,之后接着读另一个变量，这个时候它们的写可能还没有同步到主内存中，就会导致它们的读到的还是零值。所以在x86下答案是： **是**, 有可能发生。

对于ARM/POWER, 也类似，goroutine各自的写有可能还未传播到对方的处理器，所以答案也是: **是**， 有可能发生。

```go
package main

import (

	"fmt"

	"sync"

)

func main() {

	var wg sync.WaitGroup

	wg.Add(2)

	var x, y int

	go func() {

		x = 1

		r1 := y

		fmt.Println(r1) // ❶

		wg.Done()

	}()

	go func() {

		y = 1

		r2 := x

		fmt.Println(r2) // ❷

		wg.Done()

	}()

	wg.Wait()

}
```

### 两个goroutine看到的write顺序

下面这段代码, ❶和❷可能输出`1,0`和`0,1`这两种不同的结果吗？

对于x86架构， 答案是: **否**, 不可能发生。既然第三个goroutine已经观察到了`1,0`,因为x86是TSO,存储有序，所以`x=1` sychronized before `y=1`, 第四个goroutine观察到的只能是`0,0`、`1,0`或者`1,1`,不可能是`0,1`。

对于ARM架构，答案是: **是**, 是有可能发生的，不同的线程观察到的对不同的内存地址的写的顺序可能不同。

```go
package main

import (

	"fmt"

	"sync"

)

func main() {

	var wg sync.WaitGroup

	wg.Add(4)

	var x, y int

	go func() {

		x = 1

		wg.Done()

	}()

	go func() {

		y = 1

		wg.Done()

	}()

	go func() {

		r1 := x

		r2 := y

		fmt.Println(r1, r2) // ❶ 1,0

		wg.Done()

	}()

	go func() {

		r3 := x

		r4 := y

		fmt.Println(r3, r4) // ❷ 0,1

		wg.Done()

	}()

	wg.Wait()

}
```

### sync.Once

上面都是一些测试例子，为了举例内存同步的复杂性，接下来我们看一下实际标准库中为了内存同步所做的特殊处理。

下面是Go标准库中`sync.Once`的实现，虽然很简单的几行代码，你品一品，有没有什么疑问？

这个简短的几行代码经常会对深入研究Go代码的同学造成困惑，为什么❹处不使用atomic? 那么❶和❺处不使用atomic行不行？

❹处执行一个double check, ❺处的对o.done执行了一个写，他俩不会data race么？

假定g1执行了❺, 一个并发的的g2也正好执行到了❷。

因为根据g1, ❺ sequnenced before ❸, 根据Mutex的保证， g2的❷ synchronized before ❹， 所以当g2获取了获取，执行到4的时候，他是能看到g1已经把o.done标记为1了。

那么 ❶和❺处不使用atomic行不行？不行！ 严谨的说，为了性能，不行!

❶和❺处为了实现同步，必须要使用同步原语，这里最简单的就是使用atomic。 ❺处的写一定会被❶处的读看到。

如果❶和❺处不使用atomic,那么极端情况❶处总是检查o.done==0,然后总是会进入doSlow进行double check,耗费性能。

对于x86架构， ❶处可以不使用atomic,但是❺处必须使用atomic,以便能把done=1写入到主内存中。

```go
func (o *Once) Do(f func()) {

	if atomic.LoadUint32(&o.done) == 0 { // ❶

		o.doSlow(f)

	}

}

func (o *Once) doSlow(f func()) {

	o.m.Lock() // ❷

	defer o.m.Unlock() // ❸

	if o.done == 0 { // ❹

		defer atomic.StoreUint32(&o.done, 1) // ❺

		f()

	}

}
```

### sync.WaitGroup

下面是一段`WaitGroup`的示例代码, ❶处需要使用atomic.Load吗？ 如果不使用，一定输出10吗？

答案是: **不需要**， 一定输出10。

❶ suquenced before ❷, ❷ synchronized before ❸, ❸ sequenced before ❹, 所以❹一定会看到各个goroutine对count的写， 所以❹处不需要atomic。

```go
func main() {

	var count int64

    var wg sync.WaitGroup

	wg.Add(10)

    for i := 0; i < 10; i++ {

		go func() {

			atomic.AddInt64(&count, 1) // ❶

			wg.Done() // ❷

		}()

	}

    wg.Wait() // ❸

	fmt.Println(count) // ❹

}
```

### TryLock

下面这段代码是Mutex的TryLock方法的实现。❶处为什么不使用atomic,会不会导致锁已经释放了但是TryLock获取不到锁？

答案是: **会**, 在某些CPU架构下，有可能锁已经释放了，但是后续的 goroutine会获取不到锁。

> As far as the memory model is concerned, `l.TryLock` (or `l.TryRLock`) may be considered to be able to return false even when the mutex l is unlocked.  
> 就内存模型而言， `l.TryLock` （或 `l.TryRLock` ）可能被认为即使在互斥体 l 解锁时也能够返回 false。  
> 就Go内存模型而言，即使 l 已经释放了锁, `l.TryLock` (或者 `l.TryRLock`) 也可能返回false。

对于我们普通开发者而言，这叫做bug或者data race, 但是对于Go开发组来说，他们称之为为性能优化所做的特殊设计。

> The memory model is defining when one event is synchronized before another. That sentence is saying that in a sequence Lock -> Unlock -> TryLock -> Lock there is no promise that the TryLock is synchronized before the Lock. The TryLock can fail to lock the mutex even though the mutex is unlocked.  
> 内存模型定义了一个事件何时在另一个事件之前同步。这句话是说在 Lock -> Unlock -> TryLock -> Lock 序列中，没有保证 TryLock 在 Lock 之前同步。即使互斥锁已解锁，TryLock 也可能无法锁定互斥锁。
> 
> From Ian Lance Taylor

但是对于x86来说，这不是一个问题。当一个goroutine释放了锁， ❶处是能及时观察到锁的释放的。但是对于Arm架构， 锁释放后， ❶处不一定能及时的获取到锁的释放，所以即使锁释放了，它依然可能返回false，这只是非常极端极端的情况。

```go
func (m *Mutex) TryLock() bool {

	old := m.state // ❶

	if old&(mutexLocked|mutexStarving) != 0 {

		return false

	}

	if !atomic.CompareAndSwapInt32(&m.state, old, old|mutexLocked) {

		return false

	}

	return true

}
```



# [译]硬件内存模型

Russ Cox关于内存模型的系列文章之一。这是第一篇 [Hardware Memory Models](https://research.swtch.com/hwmm)

## 简介: 童话之终局

很久以前，当每个人都写单线程程序的时候，让程序运行得更快最有效的方法之一是坐下来袖手旁观。下一代硬件和编译器的优化结果可以让程序像以前一样运行，只是速度会更快。在这个童话般的年代，有一个判断优化是否有效的简单测试方法:如果程序员不能区分合法程序的未优化执行结果和优化执行的结果之间的区别(除了速度的区别)，那么这个优化就是有效的。也就是说，有效的优化不会改变有效程序的行为。

几年前， 某个悲伤的日子，硬件工程师发现让单个处理器越来越快的魔法失效了。不过，他们发现了一个新的魔法，可以让他们创造出拥有越来越多处理器的计算机，操作系统使用线程抽象模型向程序员展示了这种硬件并行能力。这种新的魔法——多处理器以操作系统线程的形式提供并行能力——对硬件工程师来说效果更好，但它给编程语言设计者、编译器作者和程序员带来了严重的问题。

许多在单线程程序中不可见(因此有效)的硬件和编译器优化会在多线程程序中产生明显的结果变化。如果有效的优化没有改变有效程序的行为，那么这些优化应该被认为是无效的。或者现有程序必须被声明为无效的。到底是哪一个，怎么判断？

这里有一个类似C语言的简单示例程序。在这个程序和我们将要考虑的所有程序中，所有变量最初都设置为零。

```
// Thread 1           // Thread 2

x = 1;                while(done == 0) { /* loop */ }

done = 1;             print(x);
```

如果线程1和线程2都运行在自己专用处理器上，都运行到完成，这个程序能打印 0 吗？

看情况(`It depends`)。这取决于硬件，也取决于编译器。在x86多处理器上, 如果逐行翻译成汇编的程序执行的话总是会打印1。但是在ARM或POWER多处理器上，如果逐行翻译成汇编的程序可以打印0。此外，无论底层硬件是什么，标准编译器优化都可能使该程序打印0或进入无限循环。

“看情况”(`It depends`)并不是一个圆满的结局。程序员需要一个明确的答案来判断一个程序是否在新的硬件和新的编译器上能够正确运行。硬件设计人员和编译器开发人员也需要一个明确的答案，说明在执行给定的程序时，硬件和编译后的代码可以有多精确。因为这里的主要问题是对存储在内存中数据更改的可见性和一致性，所以这个契约被称为内存一致性模型（`memory consistency model`）或仅仅是内存模型(`memory model`)。

最初，内存模型的目标是定义程序员编写汇编代码时硬件提供的保证。在该定义下，是不包含编译器的内容的。25年前，人们开始尝试写内存模型 ，用来定义高级编程语言(如Java或C++)对用该语言编写代码的程序员提供的保证。在模型中包含编译器会使得定义一个合理模型的工作更加复杂。

这是关于硬件内存模型和编程语言内存模型的两篇文章中的第一篇。我写这些文章的目的是先介绍一下背景，以便讨论我们可能想要对Go的内存模型进行的改变。但是，要了解Go当前状况，我们可能想去哪里，首先我们必须了解其他硬件内存模型和语言内存模型的现状，以及他们采取的道路。

还是那句话，这篇文章讲的是硬件。假设我们正在为多处理器计算机编写汇编语言。程序员为了写出正确的程序，需要从计算机硬件上得到什么保证？四十多年来，计算机科学家一直在寻找这个问题的好答案。

## 顺序一致性

Leslie Lamport 1979年的论文[《How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs》](https://www.microsoft.com/en-us/research/publication/make-multiprocessor-computer-correctly-executes-multiprocess-programs/)引入了顺序一致性的概念:

> The customary approach to designing and proving the correctness of multiprocess algorithms for such a computer assumes that the following condition is satisfied: the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program. A multiprocessor satisfying this condition will be called sequentially consistent.

> 为这种计算机设计和证明多处理算法正确性的通常方法假定满足下列条件:任何执行的结果都是相同的，就好像所有处理器的操作都是按某种顺序执行的，每个处理器的操作都是按程序指定的顺序出现的。满足这一条件的多处理器系统将被称为顺序一致的。

今天，我们不仅讨论计算机硬件，还讨论保证顺序一致性的编程语言，当程序的唯一可能执行对应于某种线程操作交替成顺序执行时。顺序一致性通常被认为是理想的模型，是程序员最自然的工作模式。它允许您假设程序按照它们在页面上出现的顺序执行，并且单个线程的执行只是以某种顺序交替(`interleaving`)，而不是以其他方式排列。

人们可能会有理由质疑顺序一致性是否应该是理想的模型，但这超出了本文的范围。我只注意到，考虑到所有可能的线程交替(`interleaving`)依然存在，就像在1979年一样，即使过了四十几年，Leslie Lamport的“设计和证明多处理算法正确性的惯用方法”，依然没有什么能取代它。

之前我问这个程序能不能打印0:

```
// Thread 1           // Thread 2

x = 1;                while(done == 0) { /* loop */ }

done = 1;             print(x);
```

为了让程序更容易分析，让我们去掉循环和打印，并询问读取共享变量的可能结果:

```
Litmus Test: Message Passing

Can this program see r1 = 1, r2 = 0?

// Thread 1           // Thread 2

x = 1                 r1 = y

y = 1                 r2 = x
```

我们假设每个例子都是所有共享变量最初都被设置为零。因为我们试图确定硬件允许做什么，我们假设每个线程都在自己的专用处理器上执行，并且没有编译器来对线程中发生的事情进行重新排序:列表中的指令就是处理器执行的指令。rN这个名字表示一个线程本地寄存器，而不是一个共享变量，我们会问一个线程本地寄存器的值在执行结束时是否存在某种可能。

这种关于样本程序执行结果的问题被称为`litmus test`。因为它只有两个答案——这个结果可能还是不可能？——`litmus test`为我们提供了一种区分内存模型的清晰方法:如果一个模型支持特定的执行，而另一个不支持，那么这两个模型显然是不同的。不幸的是，正如我们将在后面看到的，一个特定的模型对一个特定的`litmus test`给出的答案往往令人惊讶。

If the execution of this litmus test is sequentially consistent, there are only six possible interleavings:

如果该`litmus test`的执行顺序一致，则只有六种可能的交替:

[![](https://colobu.com/2021/06/30/hwmm/mem-litmus.png)](https://colobu.com/2021/06/30/hwmm/mem-litmus.png)

因为没有交替执行的结果会产生`r1 = 1, r2 = 0`,所以这个结果是不允许的。也就是说，在顺序执行的硬件上，litmus test执行结果出现`r1 = 1, r2 = 0`是不可能的。

顺序一致性的一个很好的思维模型是想象所有处理器直接连接到同一个共享内存，它可以一次处理一个线程的读或写请求。 不涉及缓存，因此每次处理器需要读取或写入内存时，该请求都会转到共享内存。 一次使用一次的共享内存对所有内存访问的执行施加了顺序顺序：顺序一致性。

[![](https://colobu.com/2021/06/30/hwmm/mem-sc.png)](https://colobu.com/2021/06/30/hwmm/mem-sc.png)

(本文中三个内存模型图摘自 Maranget et al. [“A Tutorial Introduction to the ARM and POWER Relaxed Memory Models.”](https://colobu.com/2021/06/30/hwmm/A%20Tutorial%20Introduction%20to%20the%20ARM%20and%20POWER%20Relaxed%20Memory%20Models))

上图是顺序一致机器的模型，而不是构建机器的唯一方法。 实际上，可以使用多个共享内存模块和缓存来构建顺序一致的机器来帮助预测内存获取的结果，但顺序一致意味着机器的行为必须与该模型并无二致。 如果我们只是想了解顺序一致执行意味着什么，我们可以忽略所有这些可能的实现复杂性并只考虑这个模型。

不幸的是，对于我们程序员，放弃严格的顺序一致性可以让硬件更快地执行程序，所以所有现代硬件在各方面都会偏离了顺序一致性。准确定义具体的硬件偏离是相当困难的。本文以当今广泛使用的硬件中的两种内存模型为例:**x86**、**ARM和POWER处理器系列**。

## x86 Total Store Order (x86-TSO)

现代x86系统的内存模型对应于以下硬件图:  
[![](https://colobu.com/2021/06/30/hwmm/mem-tso.png)](https://colobu.com/2021/06/30/hwmm/mem-tso.png)

所有处理器仍然连接到一个共享内存，但是每个处理器都将对该内存的写入(`write`)放入到本地写入队列中。处理器继续执行新指令，同时写操作(`write`)会更新到这个共享内存。一个处理器上的内存读取在查询主内存之前会查询本地写队列，但它看不到其他处理器上的写队列。其效果就是当前处理器比其他处理器会先看到自己的写操作。但是——这一点非常重要——所有处理器都保证写入(存储`store`)到共享内存的(总)顺序，所以给这个模型起了个名字:总存储有序，或`TSO`。当一个写操作到达共享内存时，任何处理器上的任何未来读操作都将看到它并使用该值(直到它被以后的写操作覆盖，或者可能被另一个处理器的缓冲写操作覆盖)。

写队列是一个标准的先进先出队列:内存写操作以与处理器执行相同的顺序应用于共享内存。因为写入顺序由写入队列保留，并且由于其他处理器会立即看到对共享内存的写入，所以我们之前考虑的通过`litmus test`的消息与之前具有相同的结果:`r1 = 1，r2 = 0`仍然是不可能的。

```
Litmus Test: Message Passing

Can this program see r1 = 1, r2 = 0?

// Thread 1           // Thread 2

x = 1                 r1 = y

y = 1                 r2 = x

On sequentially consistent hardware: no. 

On x86 (or other TSO): no.
```

写队列保证线程1在y之前将x写入内存，关于内存写入顺序(总存储有序)的系统级协议保证线程2在读y的新值之前读x的新值。因此，`r1 = y`在`r2 = x`看不到新的x之前不可能看到新的y。存储顺序至关重要:线程1在写入y之前先写入x，因此线程2在看到x的写入之前不可能看到y的写入。

在这种情况下，顺序一致性和TSO模型是一致的，但是他们在其他litmus test的结果上并不一致。例如，这是区分两种型号的常用示例:

```
Litmus Test: Write Queue (also called Store Buffer)

Can this program see r1 = 0, r2 = 0?

// Thread 1           // Thread 2

x = 1                 y = 1

r1 = y                r2 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): yes!
```

在任何顺序一致的执行中，`x = 1`或`y = 1`必须首先发生，然后另一个线程中的读取必须能够观察到它(此赋值事件)，因此`r1 = 0，r2 = 0`是不可能的。但是在一个TSO系统中，线程1和线程2可能会将它们的写操作排队，然后在任何一个写操作进入内存之前从内存中读取，这样两个读操作都会看到零。

这个例子看起来可能是人为制造的，但是使用两个同步变量确实发生在众所周知的同步算法中，例如[德克尔算法](https://en.wikipedia.org/wiki/Dekker%27s_algorithm)或[彼得森算法](https://en.wikipedia.org/wiki/Dekker%27s_algorithm)，以及特定的方案。如果一个线程没有看到另一个线程的所有写操作，线程就可能会中断。

为了修复同步算法，我们需要依赖于更强的内存排序，非顺序一致的硬件提供了称为内存屏障(或栅栏)的显式指令，可用于控制排序。我们可以添加一个内存屏障，以确保每个线程在开始读取之前都会刷新其先前对内存的写入:

```
// Thread 1           // Thread 2

x = 1                 y = 1

barrier               barrier

r1 = y                r2 = x
```

加上正确的障碍，`r1 = 0，r2 = 0`也是不可能的，德克尔或彼得森的算法就可以正常工作了。内存屏障有很多种；具体细节因系统而异，不在本文讨论范围之内。关键是内存屏障的存在给了程序员或语言实现者一种在程序的关键时刻强制顺序一致行为的方法。

最后一个例子，说明为什么这种模式被称为总存储有序。在该模型中，读路径上有本地写队列，但没有缓存。一旦一个写操作到达主存储器，所有处理器不仅都认同该值存在，而且还认同它相对于来自其他处理器的写操作的先后顺序。考虑一下这个litmus test:

```
Litmus Test: Independent Reads of Independent Writes (IRIW)

Can this program see r1 = 1, r2 = 0, r3 = 1, r4 = 0?

(Can Threads 3 and 4 see x and y change in different orders?)

// Thread 1    // Thread 2    // Thread 3    // Thread 4

x = 1          y = 1          r1 = x         r3 = y

                              r2 = y         r4 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): no.
```

如果线程3看到x先于y变化，那么线程4能看到y先于x变化吗？对于x86和其他TSO机器，答案是否定的:对主内存的所有存储(写入)都有一个总顺序，所有处理器都认同这个顺序，只是每个处理器在到达主内存之前都先知道自己的写入而已。

## x86-TSO 之路

x86-TSO模型看起来相当整洁，但是这条道路充满了路障和错误的弯道。在20世纪90年代，第一批x86多处理器可用的手册几乎没有提到硬件提供的内存模型。

作为问题困扰的一个例子，Plan 9 是第一个在x86上运行的真正多处理器操作系统(没有全局内核锁)。1997年，在移植到多处理器 奔腾Pro的过程中，开发人员被写队列litmus test的不期望的行为所困扰。一小段同步代码假设`r1 = 0，r2 = 0`是不可能的，但它确实发生了。更糟糕的是，英特尔手册对内存模型的细节模糊不清。

针对邮件列表中提出的“使用锁最好保守一点，不要相信硬件设计师会做我们期望的事情”的建议，Plan 9的一名开发人员很好地[解释了这个问题](https://web.archive.org/web/20091124045026/http://9fans.net/archive/1997/04/76):

> 我当然同意。我们会在多处理器中遇到更宽松的顺序(relaxed ordering )。问题是，硬件设计者认为什么是保守的？在临界区的开头和结尾强制互锁对我来说似乎相当保守，但我显然不够富有想象力。奔腾Pro的手册在描述缓存和怎么使它们保持一致时非常详细，但似乎不在乎说任何关于执行或read顺序的细节。事实是，我们无法知道自己是否足够保守。

在讨论过程中，英特尔的一名架构师对内存模型做了非正式的解释，指出理论上，即使是多处理器486和奔腾系统也可能产生`r1 = 0，r2 = 0`的结果，并且奔腾Pro只是具有更大的流水线和写队列，所以会更频繁地暴露了这种行为。

这位英特尔架构师还写道:

> Loosely speaking, this means the ordering of events originating from any one processor in the system, as observed by other processors, is always the same. However, different observers are allowed to disagree on the interleaving of events from two or more processors.  
> Future Intel processors will implement the same memory ordering model.
> 
> 粗略地说，这意味着从系统中任何一个处理器产生的事件的顺序，正如其他处理器所观察到的，总是相同的。然而，允许不同的观察者对来自两个或更多处理器的事件的交替有不同的观察结果。  
> 未来的英特尔处理器将采用相同的内存顺序模式。

声称“允许不同的观察者对来自两个或更多处理器的事件的交替有不同的观察结果”是在说，IRIW litmus test的答案在x86上可以回答“是”，尽管在前面的部分我们看到x86回答“否”。这怎么可能呢？

答案似乎是，英特尔处理器实际上从未对这一litmus test做出“是”的回答，但当时英特尔架构人员不愿意为未来的处理器做出任何保证。体系结构手册中存在的少量文本几乎没有任何保证，使得很难针对它们进行编程。

Plan 9的讨论不是一个孤立的事件。从11月下旬开始，Linux内核开发人员在他们的邮件列表上[讨论了100多条消息](https://lkml.org/lkml/1999/11/20/76)。

在接下来的十年里，越来越多的人遇到了这些困难，为此，英特尔的一组架构师承担了为当前和未来的处理器写下有用的处理器行为保证的任务。第一个结果是2007年8月出版的[Intel 64 Architecture Memory Ordering White Paper](http://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf)，旨在为“软件作者提供对不同顺序的内存访问指令可能产生的结果的清晰理解”。同年晚些时候，AMD在[AMD64 Architecture Programmer's Manual revision 3.14](https://courses.cs.washington.edu/courses/cse351/12wi/supp-docs/AMD%20Vol%201.pdf)中发布了类似的描述。这些描述基于一个被称为“总锁序+因果一致性”(TLO+CC)的模型，故意弱于TSO。在公开访谈中，英特尔架构师表示，TLO+CC[“像要求的那样强大，但并不足够强大。”](http://web.archive.org/web/20080512021617/http://blogs.sun.com/dave/entry/java_memory_model_concerns_on)特别是，该模型保留了x86处理器在IRIW litmus test中回答“是”的权利。不幸的是，内存屏障的定义不够强大，不足以重建顺序一致的内存语义，即使每个指令之后都有一个屏障。更糟糕的是，研究人员观察到实际的英特尔x86硬件违反了TLO+CC模型。例如:

```
Litmus Test: n6 (Paul Loewenstein)

Can this program end with r1 = 1, r2 = 0, x = 1?

// Thread 1    // Thread 2

x = 1          y = 1

r1 = x         x = 2

r2 = y

On sequentially consistent hardware: no.

On x86 TLO+CC model (2007): no.

On actual x86 hardware: yes!

On x86 TSO model: yes! (Example from x86-TSO paper.)
```

2008年晚些时候对英特尔和AMD规范的修订保证了IRIW case的“不”，并加强了内存屏障，但仍允许不可预期的行为，这些行为似乎不会出现在任何合理的硬件上。例如:

```
Litmus Test: n5

Can this program end with r1 = 2, r2 = 1?

// Thread 1    // Thread 2

x = 1          x = 2

r1 = x         r2 = x

On sequentially consistent hardware: no.

On x86 specification (2008): yes!

On actual x86 hardware: no.

On x86 TSO model: no. (Example from x86-TSO paper.)
```

为了解决这些问题，欧文斯等人在[早期SPARCv8 TSO模型](https://research.swtch.com/sparcv8.pdf)的基础上提出了[x86-TSO模型提案](https://www.cl.cam.ac.uk/~pes20/weakmemory/x86tso-paper.tphols.pdf)。当时，他们声称“据我们所知，x86-TSO是可靠的，足够强大，可以在上面编程，并且大致符合供应商的意图。“几个月后，英特尔和AMD发布了广泛采用这一模式的的新手册。

似乎所有英特尔处理器从一开始就实现了x86-TSO，尽管英特尔花了十年时间才决定致力于此。回想起来，很明显，英特尔和AMD的设计师们正在努力解决如何编写一个能够为未来处理器优化留出空间的内存模型，同时仍然为编译器作者和汇编语言程序设计者提供有用的保证。“有多强就有多强，但没有多强”是一个艰难的平衡动作。

## ARM/POWER Relaxed Memory Model

现在让我们来看看一个更宽松的内存模型，在ARM和POWER处理器上找到的那个。在实现层面上，这两个系统在许多方面有所不同，但保证内存一致性的模型大致相似，比x86-TSO甚至x86-TLO+CC稍弱。

ARM和POWER系统的概念模型是，每个处理器从其自己的完整内存副本中读取和向其写入，每个写入独立地传播到其他处理器，随着写入的传播，允许重新排序。  
[![](https://colobu.com/2021/06/30/hwmm/mem-weak.png)](https://colobu.com/2021/06/30/hwmm/mem-weak.png)

这里没有总存储顺序。虽然没有描述，但是每个处理器都被允许推迟读取(`read`)，直到它等到它需要结果:读取(`read`)可以被延迟到稍后的写入(`write`)之后。在这个宽松的(`relaxed`)模型中，我们迄今为止所看到的每一个litmus test的答案都是“yes，这真的可能发生。”

对于通过litmus test的原始消息，单个处理器对写入的重新排序意味着线程1的写入可能不会被其他线程以相同的顺序观察到:

```
Litmus Test: Message Passing

Can this program see r1 = 1, r2 = 0?

// Thread 1           // Thread 2

x = 1                 r1 = y

y = 1                 r2 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): no.

On ARM/POWER: yes!
```

在ARM/POWER模型中，我们可以想象线程1和线程2都有各自独立的内存副本，写操作以任何顺序在内存之间传播。如果线程1的内存在发送x的更新(`update`)之前向线程2发送y的更新，并且如果线程2在这两次更新之间执行，它将确实看到结果`r1 = 1，r2 = 0`。

该结果表明，ARM/POWER内存模型比TSO更弱:对硬件的要求更低。ARM/POWER模型仍然承认TSO所做的各种重组:

```
Litmus Test: Store Buffering

Can this program see r1 = 0, r2 = 0?

// Thread 1           // Thread 2

x = 1                 y = 1

r1 = y                r2 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): yes!

On ARM/POWER: yes!
```

在ARM/POWER上，对x和y的写入(`write`)可能会写入本地存储器，但当读取发生在相反的线程上时，写入可能尚未传播开来。

下面是一个litmus test，它展示了x86拥有总存储顺序意味着什么:

```
Litmus Test: Independent Reads of Independent Writes (IRIW)

Can this program see r1 = 1, r2 = 0, r3 = 1, r4 = 0?

(Can Threads 3 and 4 see x and y change in different orders?)

// Thread 1    // Thread 2    // Thread 3    // Thread 4

x = 1          y = 1          r1 = x         r3 = y

                              r2 = y         r4 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): no.

On ARM/POWER: yes!
```

在ARM/POWER上，不同的线程可能以不同的顺序观察到不同的写操作。它们不能保证对到达主内存的总写入顺序达成一致的观察效果，因此线程3可以在y变化之前之前看到x的变化，而线程4可以在x变化之前看到y的变化。

作为另一个例子，ARM/POWER系统具有内存读取(负载 load)的可见缓冲或重新排序，如下面litmus test所示:

```
Litmus Test: Load Buffering

Can this program see r1 = 1, r2 = 1?

(Can each thread's read happen after the other thread's write?)

// Thread 1    // Thread 2

r1 = x         r2 = y

y = 1          x = 1

On sequentially consistent hardware: no.

On x86 (or other TSO): no.

On ARM/POWER: yes!
```

任何顺序一致的交替必须从线程1的`r1 = x`或线程2的`r2 = y`开始，该读取必须看到一个0，使得结果r1 = 1，r2 = 1不可能。然而，在ARM/POWER存储器模型中，处理器被允许延迟读取，直到指令流中稍后的写入之后，因此y = 1和x = 1在两次读取之前执行。

尽管ARM和POWER内存模型都允许这一结果，但Maranget等人(2012年)[报告说](https://www.cl.cam.ac.uk/~pes20/ppc-supplemental/test7.pdf)，只能在ARM系统上凭经验重现，而不能在POWER上复制。在这里，模型和真实度之间的差异开始发挥作用，就像我们在检查英特尔x86时一样:硬件实现比技术保证更强大的模型会鼓励对更强的行为的依赖，这意味着未来更弱的硬件将破坏程序，无论是否有效。

像TSO系统上一样，ARM和POWER也有内存屏障，我们可以在上面的例子中插入这些内存屏障，以强制顺序一致的行为。但显而易见的问题是，没有内存屏障的ARM/POWER是否完全排除了任何行为。任何litmus test的答案是否都是“no，那不可能发生？” 当我们专注于一个单一的内存位置时，它可以。

这里有一个litmus test，它可以测试即使在ARM和POWER上也不会发生的事情:

```
Litmus Test: Coherence

Can this program see r1 = 1, r2 = 2, r3 = 2, r4 = 1?

(Can Thread 3 see x = 1 before x = 2 while Thread 4 sees the reverse?)

// Thread 1    // Thread 2    // Thread 3    // Thread 4

x = 1          x = 2          r1 = x         r3 = x

                              r2 = x         r4 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): no.

On ARM/POWER: no.
```

这个litmus test与前一个测试类似，但是现在两个线程都在写入单个变量x，而不是两个不同的变量x和y。线程1和2将冲突的值1和2都写入x，而线程3和线程4都读取x两次。如果线程3看到x = 1被x = 2覆盖，那么线程4能看到相反的情况吗？

答案是**no**的，即使在ARM/POWER上也是如此:系统中的线程必须就写入单个内存位置的总顺序达成一致。也就是说，线程必须同意哪些写入会覆盖其他写入。这个性质叫做相干性。如果没有一致性属性，处理器要么不同意内存的最终结果，要么报告内存位置从一个值翻转到另一个值，然后又回到第一个值。编写这样一个系统是非常困难的。

我故意忽略了ARM和POWER弱内存模型中的许多微妙之处。更多详细信息，请参阅彼得·苏厄尔关于[该主题](https://www.cl.cam.ac.uk/~pes20/papers/topics.html#Power_and_ARM)的论文。有两个要点要记住。首先，这里有令人难以置信的微妙之处，这是由有非常持久力、非常聪明的人进行了十多年学术研究的主题。我自己并不声称完全理解。这不是我们应该希望向普通程序设计人员解释的事情，也不是我们在调试普通程序时希望能够坚持的事情。第二，允许和观察到的结果之间的差距造成了不幸的未来惊喜。如果当前的硬件没有展现出所有允许的行为——尤其是当首先很难推理出什么是允许的时候！—那么不可避免地会编写一些程序，这些程序会偶然地依赖于实际硬件的更受限制的行为。如果一个新的芯片在行为上受到的限制更少，那么硬件内存模型在技术上允许破坏程序的新行为——也就是说，这个错误在技术上是你的错——这一事实并不能给你带来什么安慰。这不是写程序的方法。

## 弱排序和无数据竞争的顺序一致性

到目前为止，我希望您确信硬件细节是复杂而微妙的，而不是您每次编写程序时都想解决的问题。 相反，它有助于识别“如果你遵循这些简单的规则，你的程序只会产生结果，就像通过一些顺序一致的执行的那样。” （我们仍在谈论硬件，所以我们仍在谈论交替独立的汇编指令。）

Sarita Adve and Mark Hill proposed exactly this approach in their 1990 paper “Weak Ordering – A New Definition”. They defined “weakly ordered” as follows.

Sarita Adve和Mark Hill在他们1990年的论文[“Weak Ordering – A New Definition”](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.42.5567)中正是提出了这种方法。他们把“弱有序”定义为如下。

> Let a synchronization model be a set of constraints on memory accesses that specify how and when synchronization needs to be done.
> 
> 同步模型是对内存访问的一组约束，这些约束指定了何时以及如何进行同步。

硬件相对于同步模型是弱有序的，当且仅当它在顺序上与遵守同步模型的所有软件一致时。

虽然他们的论文是关于捕捉当时的硬件设计(不是x86、ARM和POWER)，但将讨论提升到特定设计之上的想法使论文与今天的讨论依然相关。

我之前说过“有效的优化不会改变有效程序的行为。”这些规则定义了什么是有效的手段，然后任何硬件优化都必须让这些程序像在顺序一致的机器上一样工作。当然，有趣的细节是规则本身，定义程序有效的约束。

Adve和Hill提出了一种同步模型，他们称之为无数据竞争(data-race-free，DRF)。该模型假设硬件具有独立于普通内存读写的内存同步操作。普通的内存读写可以在同步操作之间重新排序，但不能在跨它们移动。(也就是说，同步操作也可用来做重新排序的内存屏障。)如果对于所有理想化的顺序一致的执行，从不同线程对同一位置的任何两个普通存储器访问要么都是读取，要么通过同步操作强制一个在另一个之前发生而分开执行，则程序被称为无数据竞争的。  
我们来看一些例子，摘自Adve和Hill的论文(为了演示而重新绘制)。这里有一个线程执行变量x的写操作，然后读取同一个变量。  
[![](https://colobu.com/2021/06/30/hwmm/mem-adve-1.png)](https://colobu.com/2021/06/30/hwmm/mem-adve-1.png)  
垂直箭头标记了单个线程内的执行顺序:先写后读。这个程序没有竞争，因为一切都在一个线程中。

相比之下，在这个双线程程序中有一个竞争:  
[![](https://colobu.com/2021/06/30/hwmm/mem-adve-2.png)](https://colobu.com/2021/06/30/hwmm/mem-adve-2.png)

这里线程2在不与线程1协调的情况下写入x。线程2的写入与线程1的写入和读取竞争。如果线程2读x而不是写x，程序在线程1写和线程2读之间只有一个竞争。每个竞争至少涉及一次写入:两次不协调的读取不会相互竞争。

为了避免竞争，我们必须添加同步操作，这将在共享一个同步变量的不同线程上的操作之间强制一个特定的顺序。如果同步S(a)(在变量a上同步，用虚线箭头标记)迫使线程2的写操作在线程1完成后发生，则竞争被消除-  
[![](https://colobu.com/2021/06/30/hwmm/mem-adve-3.png)](https://colobu.com/2021/06/30/hwmm/mem-adve-3.png)

现在线程2的写操作不能与线程1的操作同时发生。

如果线程2只是读取，我们只需要与线程1的写入同步。两次读取仍然可以同时进行:

[![](https://colobu.com/2021/06/30/hwmm/mem-adve-4.png)](https://colobu.com/2021/06/30/hwmm/mem-adve-4.png)

线程可以按同步顺序排序，甚至可以使用中间线程。这个程序没有竞争:

[![](https://colobu.com/2021/06/30/hwmm/mem-adve-5.png)](https://colobu.com/2021/06/30/hwmm/mem-adve-5.png)

另一方面，同步变量的使用本身并不能消除竞争:错误地使用它们是可能的。下面这个程序有一个竞争:  
[![](https://colobu.com/2021/06/30/hwmm/mem-adve-6.png)](https://colobu.com/2021/06/30/hwmm/mem-adve-6.png)

线程2的读取与其他线程中的写入完全同步——这肯定发生在两者之后——但是这两个写入本身并不同步。这个程序并不是data-race-free。

Adve和Hill将弱排序描述为“软件和硬件之间的契约”，具体来说，如果软件避免了数据竞争，那么硬件就好像是顺序一致的，这比我们在前面部分研究的模型更容易推理。但是硬件如何满足它的契约呢？

Adve和Hill给出了硬件“遵循DRF弱排序”的证明，这意味着它执行无数据竞争的程序，就好像是按照顺序一致的顺序一样，只要它满足一组特定的最低要求。我不打算详谈细节，但重点是在Adve和Hill的论文发表后，硬件设计师们有了一份由理论支持的手册:做这些事情，你就可以断言你的硬件将与data-race-free程序顺序一致。事实上，假设同步操作的适当实现，大多数宽松的硬件确实是这样做的，并且一直在继续这样做。Adve和Hill最初关注的是VAX，但x86、ARM和POWER肯定也能满足这些限制。这种系统保证无数据竞争程序的顺序一致性的观点通常被缩写为DRF-SC。

DRF-SC标志着硬件内存模型的一个转折点，为硬件设计者和软件作者提供了一个清晰的策略，至少是那些用汇编语言编写软件的人。正如我们将在下一篇文章中看到的，高级编程语言的内存模型问题没有一个整洁的答案。


# [译]编程语言内存模型

这是Russ Cox的第二篇[Programming Language Memory Models](https://research.swtch.com/plmm)。

如果你已经阅读了前一篇[硬件内存模型](https://colobu.com/2021/06/30/hwmm/#%E5%BC%B1%E6%8E%92%E5%BA%8F%E5%92%8C%E6%97%A0%E6%95%B0%E6%8D%AE%E7%AB%9E%E4%BA%89%E7%9A%84%E9%A1%BA%E5%BA%8F%E4%B8%80%E8%87%B4%E6%80%A7),以及如果有Java内存模型或者C++内存模型的经验，本文还好理解，如果你没有相关经验，可能阅读起来比较费劲，建议先阅读一下相关的材料。论文有些词句比较难以理解，本人才学疏浅，有翻译不当之处欢迎批评指正。

编程语言内存模型回答了并行程序可以依靠什么行为以便它们的线程之间可以共享内存的问题。例如，考虑下面这个类似C语言的程序，其中x和done都从零开始：

```
// Thread 1           // Thread 2

x = 1;                while(done == 0) { /* loop */ }

done = 1;             print(x);
```

程序试图通过变量x从线程1向线程2发送一条消息(x)，使用done作为信号，通知线程2消息已经准备好被接收。如果线程1和线程2都运行在自己的专用处理器上，并且都运行完成，那么这个程序是否保证能够按照预期完成并打印1？编程语言内存模型回答了这个问题，以及其它类似问题。

Although each programming language differs in the details, a few general answers are true of essentially all modern multithreaded languages, including C, C++, Go, Java, JavaScript, Rust, and Swift:

尽管每种编程语言在细节上有所不同，但是一些通用答案基本上适用于所有现代多线程语言，包括C、C++、Go、Java、JavaScript、Rust和Swift:

-   首先，如果x和done是普通变量，那么线程2的循环可能永远不会停止。一种常见的编译器优化是在变量首次使用时将其加载到寄存器中，然后尽可能长时间地重用该寄存器，以便将来访问该变量。如果线程2在线程1执行之前将done复制到一个寄存器中，它可能会在整个循环中一直使用该寄存器，永远不会注意到线程1后来修改了done。
-   其次，即使线程2的循环会停止，也就是观察到done == 1，它仍然可能打印x的值为0。编译器通常会根据优化试探法甚至是生成代码时使用哈希表或其他中间数据结构的方式，对程序读写进行重新排序。线程1的编译代码可能在done赋值之后而不是之前写入x，或者线程2的编译代码也可能在循环前读取x。

既然这个程序有并发问题，那么问题是如何修复它。

现代语言以原子变量(atomic variable)或原子操作(atomic operation)的形式提供特殊能力，允许程序同步其线程。如果我们使用一个原子变量实现done(或者用原子操作来操作它)，那么我们的程序保证会执行完成并打印1。使用原子变量或者原子操作会产生很多效果：

-   线程1的编译代码必须确保对x的写入完成，并且在对done的写入可见之前对x的写入对其他线程可见。
-   线程2的编译代码必须在循环的每次迭代中(重新)读取done。
-   线程2的编译代码必须在读取done之后才读取x。
-   编译后的代码必须做任何必要的事情来禁用可能会重新引入这些问题的硬件优化

使done原子化的最终结果是程序按照我们想要的方式运行，成功地将x的值从线程1传递到线程2。

在最初始的程序中，在编译器的代码重新排序之后，线程1可能会在线程2读取x的同时写x。这是data race问题。在修改后的程序中，原子变量done用于同步对x的访问:线程1现在不可能在线程2读取x的同时写入x。这个程序没有数据竞争。一般来说，现代语言保证了无数据竞争的程序总是以顺序一致（sequentially consistent）的方式执行，就好像来自不同线程的操作被随意地但没有重新排序地转移到单个处理器上一样。这是硬件内存模型的[DRF-SC属性](https://colobu.com/2021/06/30/hwmm/#%E5%BC%B1%E6%8E%92%E5%BA%8F%E5%92%8C%E6%97%A0%E6%95%B0%E6%8D%AE%E7%AB%9E%E4%BA%89%E7%9A%84%E9%A1%BA%E5%BA%8F%E4%B8%80%E8%87%B4%E6%80%A7)，在编程语言环境中采用。

另外，这些原子变量或原子操作更恰当应该称之为“同步原子”(synchronizing atomic)，在数据库的意义上，操作是原子的，允许同时进行读和写，就像以某种顺序按顺序运行一样:当使用原子时，普通变量上的竞争不是竞争。但更重要的是，atomic同步了程序的其余部分，提供了一种消除非原子数据竞争的方法。标准术语就是“atomic”，也就是这篇文章使用的属于。除非另有说明，请记住将“原子”理解为“同步原子”。

编程语言内存模型规定了程序员和编译器所需的额外细节，作为他们之间的约定。上面谈到的通用特征基本上适用于所有现代语言，但直到最近，事情才收敛到一点:在21世纪初，有明显更多的变种。即使在今天，各种语言在更多的排序问题上也有显著的差异，包括:

-   原子变量们本身的排序保证是什么？
-   变量是否既可以原子访问，有可以非原子访问？
-   除了原子之外是否还有其它同步机制？
-   是否存在不同步的原子操作？
-   有数据竞争的程序有什么保证？

在做了一些准备之后，这篇文章的剩余部分将探讨不同的语言如何回答这些相关的问题，以及它们解决这些问题之道。这篇文章介绍探索路上的许多错误初始设计，强调我们仍然在学习啥是有效的方案，啥是无效的方案

## 硬件、Litmus Tests、Happens Before 和 DRF-SC

在我们了解任何特定语言的细节之前，我们需要记住[硬件内存模型](https://colobu.com/2021/06/30/hwmm/)的简要经验总结。

> 不同的CPU体系架构允许不同数量的指令重新排序，因此在多个处理器上并行运行的代码可以根据体系架构的不同有不同的结果。黄金标准是[顺序一致性](https://colobu.com/2021/06/30/hwmm/#%E9%A1%BA%E5%BA%8F%E4%B8%80%E8%87%B4%E6%80%A7)，即任何执行都必须表现得好像在不同处理器上执行的程序只是以某种顺序交替在单个处理器上执行。对于开发人员来说，这种模型更容易推理，但是今天没有重要的架构能够提供这种模型，因为提供较弱的并发保证能够足够的性能。

很难对不同的内存模型做出完全通用的比较。反过来我们可以关注特定的测试用例，称为Litmus Test。如果两个内存模型通过Litmus Test有不同的行为，那么可以证明它们是不同的，并且通常可以帮助我们判断，至少对于那个测试用例，一个模型比另一个模型是弱还是强。例如，这是我们之前检查的程序的Litmus Test:

```
Litmus Test: Message Passing

Can this program see r1 = 1, r2 = 0?

// Thread 1           // Thread 2

x = 1                 r1 = y

y = 1                 r2 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): no.

On ARM/POWER: yes!

In any modern compiled language using ordinary variables: yes!
```

和前一篇文章一样，我们假设每个例子一开始所有共享变量都为零。rN这个名字表示私有存储，如寄存器或函数变量；像x和y这样的名称是共享(全局)变量。我们问在执行结束时，寄存器的特定设置是否存在可能。在回答硬件的litmus test时，我们假设没有编译器对线程中发生的事情进行重新排序:清单中的指令被直接翻译成汇编指令，交给处理器执行。

结果r1 = 1，r2 = 0代表原始程序的线程2完成了循环(这里简化了循环，而是简单的使用y进行赋值)，但随后打印0。这个结果在程序操作的任何顺序一致的交替执行中是不可能的。对于汇编语言版本，在x86上打印0是不可能的，尽管由于处理器本身的重新排序优化，在ARM和POWER等更宽松的架构上打印0是可能的。在现代语言中，编译期间可能发生的重新排序使得这种结果成为可能，不管底层硬件是什么。

正如我们前面提到的，今天的处理器不保证顺序一致性，而是保证一种称为[“无数据竞争的顺序一致性”或DRF-DRF(有时也写成SC-DRF)](https://colobu.com/2021/06/30/hwmm/#%E5%BC%B1%E6%8E%92%E5%BA%8F%E5%92%8C%E6%97%A0%E6%95%B0%E6%8D%AE%E7%AB%9E%E4%BA%89%E7%9A%84%E9%A1%BA%E5%BA%8F%E4%B8%80%E8%87%B4%E6%80%A7)的属性。一个保证DRF-SC的系统必须提供被称为同步指令的特定指令，它提供了一种协调不同处理器(相当于线程)的方法。程序使用这些指令在一个处理器上运行的代码和另一个处理器上运行的代码之间创建一种“happens before”的关系。

例如，这里描述了一个程序在两个线程上的短暂执行；像以前一样，假设每个处理器都有自己的专用处理器:

[![](https://colobu.com/2021/07/11/Programming-Language-Memory-Models/mem-adve-4.png)](https://colobu.com/2021/07/11/Programming-Language-Memory-Models/mem-adve-4.png)

我们在之前的帖子里也看到了这个程序。线程1和线程2执行同步指令。在这个特定执行中，两条S(a)指令建立了从线程1到线程2的happens-before关系，因此线程1中的W(x)发生在线程2中的R(x)之前。

不同处理器上的两个事件，如果不是按照happens-before的顺序排序，可能会同时发生:确切的顺序搞不清楚。我们说它们同时执行。数据竞争（data race）是指对一个变量的写操作与对同一变量的读操作或写操作同时执行。提供DRF-SC的处理器保证没有数据竞争的程序行为就像它们在一个顺序一致的架构上运行一样。这是在现代处理器上编写正确的多线程汇编程序的基本保证。

正如我们前面所看到的，DRF-SC也是现代语言所采用的基本保证，使得用更高级别语言编写正确的多线程程序成为可能。

## 编译器和优化

我们已经提到过几次，编译器可能会在生成最终可执行代码的过程中对输入程序中的操作重新排序。让我们仔细看看这个声明和其他可能导致问题的优化。

人们普遍认为，编译器几乎可以任意地对普通的内存读写进行重新排序，前提是重新排序不能改变观察到的单线程代码执行的效果。例如，考虑这个程序:

```
w = 1

x = 2

r1 = y

r2 = z
```

由于w、x、y和z都是不同的变量，这四个语句可以以编译器认为最好的任何顺序执行。

如上所述，如此自由地重新排序读写的能力使得普通编译程序的保证至少和ARM/POWER宽松内存模型一样弱，因为编译程序无法通过消息传递的litmus test。事实上，编译程序的保证更弱。

在上一篇硬件内存模型的文章中，我们将一致性（coherence）看作是ARM/POWER架构所能保证的一个例子:

```
Litmus Test: Coherence

Can this program see r1 = 1, r2 = 2, r3 = 2, r4 = 1?

(Can Thread 3 see x = 1 before x = 2 while Thread 4 sees the reverse?)

// Thread 1    // Thread 2    // Thread 3    // Thread 4

x = 1          x = 2          r1 = x         r3 = x

                              r2 = x         r4 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): no.

On ARM/POWER: no.

In any modern compiled language using ordinary variables: yes!
```

所有现代硬件都保证一致性，这也可以看作是对单个存储单元的操作的顺序一致性。在这个程序中，一个写操作必须覆盖另一个，并且整个系统必须就哪个是哪个达成一致。事实证明，由于编译过程中程序的重新排序，现代语言甚至不能提供一致性。

假设编译器对线程4中的两次读取进行了重新排序，然后指令按照以下顺序交替运行:

```
// Thread 1    // Thread 2    // Thread 3    // Thread 4

                                             // (reordered)

(1) x = 1                     (2) r1 = x     (3) r4 = x

               (4) x = 2      (5) r2 = x     (6) r3 = x
```

结果r1 = 1，r2 = 2，r3 = 2，r4 = 1 在汇编程序中是不可能的，但在高级语言中是可能的。从这个意义上说，编程语言内存模型都比最宽松的硬件内存模型都弱。

但是有一些保证。每个人都同意需要提供DRF-SC，它不允许引入新的读或写的优化，即使这些优化在单线程代码中是有效的。

例如，考虑下面的代码:

```
if(c) {

	x++;

} else {

	... lots of code ...

}
```

有一个if语句，在else中有很多代码，在if主体中只有一个x++。拥有更少的分支并彻底消除if体可能更快。如果我们编写代码有问题，我们可以在if之前运行了x++,然后在else中用x--进行调整。也就是说，编译器可能会考虑将该代码重写为:

```
x++;

if(!c) {

	x--;

	... lots of code ...

}
```

这是安全的编译器优化吗？在单线程程序中，是的。在一个多线程程序中，当c为false时，x与另一个线程共享，答案是否: 优化会在x上引入一个原始程序中没有的数据

这个例子来源于Hans Boehm 2004年的论文[Threads Cannot Be Implemented As a Library](https://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf)中的一个例子，它说明了语言不能对多线程执行的语义保持沉默。

编程语言内存模型试图精确回答这些问题，即哪些优化是允许的，哪些是不允许的。通过研究过去几十年来尝试编写这些模型的历史，我们可以了解哪些可行，哪些不可行，并了解事情的发展方向。

## 原始的Java内存模型 (1996)

Java是第一个试图写下多线程程序保证的主流语言。它包括互斥体(mutex)，并定义了它们隐含的内存排序要求。它还包括“volatile”原子变量: volatile变量的所有读和写都需要直接在主内存中按程序顺序执行，使得对volatile变量的操作以顺序一致的方式进行。最后，Java制定了(或者至少试图制定)具有数据竞争的程序的行为。其中的一部分是为普通变量规定一种一致性的形式，我们将在下面详细讨论。不幸的是，在Java语言规范(1996)的第一版中，这种尝试至少有两个严重的缺陷。凭借后见之明和我们已经做好的准备，它们很容易解释。当时，它们远没有那么明显被发现。

### Atomic 需要同步

第一个缺陷是volatile原子变量是不同步的，所以它们无助于消除程序其余部分的竞争。我们在上面看到的消息传递程序的Java版本是:

```
int x;

volatile int done;

// Thread 1           // Thread 2

x = 1;                while(done == 0) { /* loop */ }

done = 1;             print(x);
```

因为done被声明为volatile，所以循环肯定会结束: 编译器不能将它缓存在寄存器中并导致无限循环。但是，程序不能保证打印1。编译器没有被禁止重新排序对x和done的访问，也没有被要求禁止硬件做同样的事情。

因为Java volatile是非同步原子，所以您不能使用它们来构建新的同步原语。从这个意义上说，最初的Java内存模型太弱了。

### 一致性与编译器优化不兼容

初的Java内存模型也太强了: 强制一致性 —— 一旦线程读取了内存位置的新值，它就不能再读取旧值——不允许基本的编译器优化。前面我们讨论了重新排序读操作会如何破坏一致性，但是你可能会想，好吧，不要重新排序读操作。这里有一个更微妙的方法，可以通过另一个优化来打破一致性:公共子表达式消除。考虑一下这个Java程序:

考虑一下这个Java程序:

```
// p and q may or may not point at the same object.

int i = p.x;

// ... maybe another thread writes p.x at this point ...

int j = q.x;

int k = p.x;
```

在这个程序中，公共子表达式消除(common subexpression elimination)会注意到p.x被计算了两次，并将最后一行优化为k = i。但是，如果p和q指向同一个对象，并且另一个线程在读入I和j之间向p.x写入，那么为k重用旧值i违反了一致性:读入i看到了旧值，读入j看到了新值，但是读入k重用i会再次看到旧值。不能优化掉冗余读取会阻碍大多数编译器，使生成的代码变慢。

硬件比编译器更容易提供一致性，因为硬件可以应用动态优化:它可以根据给定内存读写序列中涉及的确切地址来调整优化路径。相比之下，编译器只能应用静态优化:无论涉及什么地址和值，它们都必须提前写出正确的指令序列。在这个例子中，编译器不能根据p和q是否碰巧指向同一个对象来轻易改变发生的事情，至少在没有为这两种可能性写出代码的情况下不能，这导致了大量的时间和空间开销。编译器对内存位置之间可能存在的别名不完全了解意味着：实际上要实现一致性就需要放弃基本的优化。

Bill Pugh在他1999年的论文[修复Java内存模型](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.17.7914&rep=rep1&type=pdf)中指出了这个问题和其他问题.

## 新的Java内存模型 (2004)

由于这些问题，并且因为最初的Java内存模型甚至对于专家来说都很难理解，Pugh和其他人开始努力为Java提供一个新的内存模型。该模型后来成为JSR-133，并在2004年发布的Java 5.0中被采用。规范参考是Jeremy Manson, Bill Pugh和Sarita Adve的[Java内存模型(2005)](http://rsim.cs.uiuc.edu/Pubs/popl05.pdf)，Jeremy Manson的博士论文中有更多细节。新模型遵循DRF-SC方法:保证无数据竞争的Java程序以顺序一致的方式执行。

### 同步原子和其它操作

正如我们前面看到的，要编写一个无数据竞争的程序，程序员需要同步操作，这些同步操作可以建立happens-before关系，以确保一个线程不会在另一个线程读取或写入非原子变量的同时写入该变量。在Java中，主要的同步操作有:

-   线程的创建发生在(happen before)它的第一个动作之前
-   mutex m的unlock发生在m的后续lock之前
-   写volatile变量v发生在后续读取v之前

“subsequent”(“后续”)`是什么意思？Java定义了所有锁、解锁和volatile变量访问的行为，就好像它们发生在一些顺序一致的中断中，给出了整个程序中所有这些操作的总顺序。“后续”是指总顺序中的较晚执行。也就是说:锁、解锁和volatile变量访问的总顺序定义了后续的含义，然后后续定义了特定执行创建了happen before关系，然后happend before关系定义了该特定执行是否有数据竞争。如果没有数据竞争，那么执行就会以顺序一致的方式进行。

事实上， volatile访问必须表现得好像在某种总排序中一样，这意味着在[存储缓冲区litmus test](https://research.swtch.com/hwmm#x86)中，不能出现r1 = 0和r2 = 0的结果:

```
Litmus Test: Store Buffering

Can this program see r1 = 0, r2 = 0?

// Thread 1           // Thread 2

x = 1                 y = 1

r1 = y                r2 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): yes!

On ARM/POWER: yes!

On Java using volatiles: no.
```

在Java中，对于volatile变量x和y，读取和写入不能被重新排序:一个写入必须排在第二位，第二个写入之后的读取必须看到第一个写入。如果我们没有顺序一致的要求——比如说，如果只要求volatile是一致的——两次读取可能会错过写入。

这里有一个重要但微妙的点:所有同步操作的总顺序与happen-before关系是分开的。在程序中的每个锁、解锁或volatile变量访问之间，在一个方向或另一个方向上不存在happen-before关系:从写入到观察写入的读取，您只获得了happen-before的关系。例如，不同互斥体的锁定和解锁之间没有happen-before关系。

### 有数据竞争的程序的语义

DRF-SC只保证没有数据竞争的程序的顺序一致行为。新的Java内存模型和最初的一样，定义了有数据竞争的程序的行为，原因有很多:

-   支持Java的一般安全（security）和安全保障（safety guarantee）。
-   让程序员更容易发现错误。
-   使攻击者更难利用问题，因为由于数据竞争的原因可能造成的损失更有限。
-   让程序员更清楚他们的程序是做什么的。

新模型不再依赖于一致性，而是重新使用了happens-before关系(已经用来决定程序是否有竞争)来决定竞争读写的结果。

Java的具体规则是，对于word大小或更小的变量，对变量(或字段)x的读取必须看到对x的某一次写入所存储的值。如果读取r观察到对x的写入w，那么r不发生在w之前。这意味着r可以观察发生在r之前的所有写入，并且它可以观察与r竞争的写入。

使用happens-before，结合同步原子(volatile)就可以建立新的happen before关系，是对原始Java内存模型的重大改进。它为程序员提供了更多有用的保证，并使大量重要的编译器优化得到了明确的允许。这个模型至今仍然是Java的内存模型。也就是说，这仍然是不完全正确的:在试图定义竞争程序的语义时，使用before-before是有问题的。

### happen-before不排除语无伦次(incoherence)

定义程序语义的“happen-before”关系的第一个问题与一致性有关(有一次!).(以下例子摘自Jaroslav Ševčík 和 David Aspinall的论文[《论Java内存模型中程序转换的有效性》(2007年)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.112.1790&rep=rep1&type=pdf))。)

这里有一个三线程的程序。让我们假设线程1和线程2已知在线程3开始之前完成。

```
// Thread 1           // Thread 2           // Thread 3

lock(m1)              lock(m2)

x = 1                 x = 2

unlock(m1)            unlock(m2)

                                            lock(m1)

                                            lock(m2)

                                            r1 = x

                                            r2 = x

                                            unlock(m2)

                                            unlock(m1)
```

线程1在持有mutex m1时写入x = 1。线程2在持有 mutex m2时写入x = 2。这些是不同的mutex，所以两个写操作是竞争的。然而，只有线程3读取x，并且它是在获取两个mutex后读取的。对r1的读取可以是读也可以是写:两者都发生在它之前，并且都不会完全覆盖另一个。通过相同的参数，读入r2可以读或写。但是严格来说，Java内存模型中没有任何东西说两次读取必须一致：从技术上讲，r1和r2可以读取不同的x值。也就是说，这个程序可以以r1和r2持有不同的值结束。当然，没有真正的实现会产生不同的r1和r2。互斥意味着这两次读取之间没有写操作发生。他们必须得到相同的值。但是内存模型允许不同读取值的事实表明，从某种技术角度来说，它并没有精确地描述真实的Java实现。

情况变得更糟。如果我们在两个读数之间再加一个指令，x = r1，会怎么样:

```
// Thread 1           // Thread 2           // Thread 3

lock(m1)              lock(m2)

x = 1                 x = 2

unlock(m1)            unlock(m2)

                                            lock(m1)

                                            lock(m2)

                                            r1 = x

                                            x = r1   // !?

                                            r2 = x

                                            unlock(m2)

                                            unlock(m1)
```

很明显，r2 = x读数必须使用x = r1写的值，因此程序必须在r1和r2中获得相同的值。两个值r1和r2现在保证相等。

这两个程序之间的差异意味着我们在编译器方面有问题。看到r1 = x后跟着x = r1时编译器很可能想要删除第二个赋值，这“显然”是多余的。但这种“优化”将第二个程序(r1和r2的值必须相同)变成了第一个程序(从技术上讲，r1可能不同于r2)。因此，根据Java内存模型，这种优化在技术上是无效的:它改变了程序的含义。明确地说，这种优化不会改变在任何你能想象的真实JVM上执行的Java程序的意义。但不知何故，Java内存模型不允许这样做，这表明还有更多需要说的。

有关这个例子和其他例子的更多信息，请参见evík和Aspinall的论文。

## 以前发生的事不排除无用性（acausality）

最后一个例子证明是个简单的问题。这里有一个更难的问题。考虑这个litmus test，使用普通的(非volatile)Java变量:

```
Litmus Test: Racy Out Of Thin Air Values

Can this program see r1 = 42, r2 = 42?

// Thread 1           // Thread 2

r1 = x                r2 = y

y = r1                x = r2

(Obviously not!)
```

这个程序中的所有变量都像往常一样从零开始，然后这个程序在一个线程中有效地运行y = x，在另一个线程中运行x = y。x和y最终能变成42吗？在现实生活中，显然不能。但为什么不呢？内存模型并没有否定这个结果。

假设“r1 = x”的读数是42。那么“y = r1”会将42写入y，然后竞争“r2 = y”会读取42，导致“x = r2”写入42到x，且write与原始“r1 = x”竞争(因此可被原始“r1 = x”观察到)，看起来证明原始假设是正确的。在这个例子中，42被称为无中生有的值，因为它看起来没有任何理由，但随后用循环逻辑证明了自己。如果内存在当前的0之前曾经持有42，而硬件错误地推测它仍然是42，会怎么样？这种猜测可能会成为一个自我实现的预言。(在Spectre和相关攻击显示出硬件是如何不断进步的之前，这个论点似乎更加牵强。即便如此，没有一种硬件是这样凭空创造值的。)

很明显，这个程序不能以r1和r2设置为42结束，但是happens-before本身并不能解释为什么不能这样做。这再次表明存在某种不完整性。新的Java内存模型花费了大量时间来解决这种不完整性，稍后将对此进行更详细的描述。

这个程序有一个竞争——x和y的读取与其他线程中的写入竞争——所以我们可能会继续认为它是一个不正确的程序。但是这里有一个没有数据竞争的版本:

```
Litmus Test: Non-Racy Out Of Thin Air Values

Can this program see r1 = 42, r2 = 42?

// Thread 1           // Thread 2

r1 = x                r2 = y

if (r1 == 42)         if (r2 == 42)

    y = r1                x = r2

(Obviously not!)
```

由于x和y从零开始，任何顺序一致的执行都不执行写操作，所以这个程序没有写操作，所以没有竞争。不过，同样，仅happen-before并不排除这样的可能性，假设r1 = x看到竞争不是write，然后根据这个假设，两个条件最终都为真，x和y最终都是42。这是另一种无中生有的价值，但这一次是在没有竞争的程序中。任何保证DRF-SC的模型都必须保证这个程序只在末尾看到全零，然而happens-before并没有解释为什么。

Java内存模型花了很多我不想赘述的话来试图排除这些类型的假设。不幸的是，五年后，Sarita Adve and Hans Boehm对这个内存模型有这样的评价:

> Prohibiting such causality violations in a way that does not also prohibit other desired optimizations turned out to be surprisingly difficult. … After many proposals and five years of spirited debate, the current model was approved as the best compromise. … Unfortunately, this model is very complex, was known to have some surprising behaviors, and has recently been shown to have a bug.
> 
> 以一种不妨碍其他期望的优化的方式来禁止这种因果关系冲突，结果令人惊讶地难以实现。……经过许多提议和五年的激烈辩论，目前的模式被认为是最好的折衷方案。……不幸的是，这个模型非常复杂，已知有一些令人惊讶的缺点，最近被证明有一个错误。

(Adve 和 Boehm, [“Memory Models: A Case For Rethinking Parallel Languages and Hardware,”](https://cacm.acm.org/magazines/2010/8/96610-memory-models-a-case-for-rethinking-parallel-languages-and-hardware/fulltext) August 2010)

## C++11 内存模型 (2011)

让我们把Java放在一边，研究C++。受Java新内存模型明显成功的启发，许多同样的人开始为C++定义一个类似的内存模型，最终在C++11中采用。与Java相比，C++在两个重要方面有所不同。首先，C++对具有数据竞争的程序不做任何保证，这似乎消除了对Java模型复杂性的需求。其次，C++提供了三种原子性:强同步(“顺序一致”)、弱同步(“acquire/release”,、coherence-only)和无同步(“relaxed”，用于隐藏竞争)。“relaxed”的原子性重新引入了Java关于定义什么是竞争程序的所有复杂性。结果是C++模型比Java更复杂，但对程序员的帮助更小。

C++11还定义了原子栅栏作为原子变量的替代，但是它们并不常用，我不打算讨论它们。

### DRF-SC 还是 着火(Catch Fire）

与Java不同，C++没有给有竞争的程序任何保证。任何有竞争的程序都属于[“未定义的行为”](https://blog.regehr.org/archives/213)。允许在程序执行的最初几微秒内进行竞争访问，从而在几小时或几天后导致任意的错误行为。这通常被称为“DRF-SC或着火”:如果程序没有数据竞争，它以顺序一致的方式运行，如果有数据竞争，它可以做任何事情，包括着火。

关于DRF-SC或Catch Fire的论点的更详细的介绍，参见Boehm，[“内存模型原理”(2007)](http://open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2176.html#undefined) 和Boehm和Adve，[“C++并发内存模型的基础”(2008)](https://www.hpl.hp.com/techreports/2008/HPL-2008-56.pdf)。

简而言之，这中情况有四个正当理由:

Briefly, there are four common justifications for this position:

-   C和C++已经充斥着未定义的行为，这是编译器优化横行的语言小角落，用户最好不要迟疑。多一个未定义行为又有多大的坏处？
-   现有的编译器和库编写时没有考虑线程，以任意方式破坏了有竞争的程序。找到并修复所有的问题太难了，或者这个争论没有了，尽管还不清楚那些不固定的编译器和库是如何应对宽松的原子的。
-   真正知道自己在做什么并希望避免未定义行为的程序员可以使用relaxed的原子。
-   不定义竞争语义允许实现检测和诊断竞争并停止执行。

就我个人而言，最后一个理由是我认为唯一有说服力的，尽管我认为这个意思是说“允许使用竞争检测器”，而不是说“一个整数的竞争会使整个程序无效。”

这里有一个来自“内存模型原理”的例子，我认为它抓住了C++方法的本质以及它的问题。考虑这个程序，它引用了一个全局变量x。

```
unsigned i = x;

if (i < 2) {

	foo: ...

	switch (i) {

	case 0:

		...;

		break;

	case 1:

		...;

		break;

	}

}
```

据称，C++编译器可能会将i保存在寄存器中，但如果标签foo处的代码很复杂，则需要重用这些寄存器。而不是转移i当前的值到栈上， 编译器可能会决定在到达switch语句时，再次从全局x加载i。结果是，在if体中，i < 2可能不再为真。如果编译器使用由i索引的表将开关编译成计算跳转，那么代码将从表的末尾索引并跳转到一个意外的地址，这可能非常糟糕。

从这个例子和其他类似的例子中，C++内存模型的作者得出结论，任何有竞争的访问都必须被允许对程序的未来执行造成无限的损害。我个人的结论是，在多线程程序中，编译器不应该认为它们可以通过重新执行初始化局部变量的内存读取来重新加载像i这样的局部变量。指望为单线程世界编写的现有C++编译器找到并修复像这样的代码生成问题可能是不切实际的，但是在新的语言中，我认为我们应该有更高的目标。

### 题外话, C/C++的未定义行为

另外，C和C++坚持编译器对程序中的错误行为进行任意的行为的能力导致了真正荒谬的结果。例如，考虑这个程序，这是2017在推特上讨论的话题：

```c++
#include <cstdlib>

typedef int (*Function)();

static Function Do;

static int EraseAll() {

	return system("rm -rf slash");

}

void NeverCalled() {

	Do = EraseAll;

}

int main() {

	return Do();

}
```

如果你是一个像Clang这样的现代C++编译器，你可能会想到这个程序如下：

-   很明显，main函数中，Do要么为空，要么为EraseAll。
-   如果Do是Erasell，那么Do()与Erasell()相同。
-   如果Do 是null, 那么Do()是未定义行为。我可以随意实现，包括作为EraseAll()无条件实现。
-   因此，我可以将间接调用Do()优化为直接调用EraseAll()。
-   当我处理这里的时候，我可能直接内联EraseAll。

最终结果是，Clang将程序优化为:

```
int main() {

	return system("rm -rf slash");

}
```

你必须承认:前一个例子，局部变量i可能在if (i < 2)体的中途突然停止小于2的可能性似乎并不合适。

本质上，现代C和C++编译器假设没有程序员敢尝试未定义的行为。一个程序员写一个有bug的程序？不可思议！

就像我说的，在新的语言中，我认为我们应该有更高的目标。

### Acquire/release atomic

C++采用了顺序一致的原子变量，很像(新的)Java的volatile变量(与C++ volatile没有关系)。在我们的消息传递示例中，我们可以将done声明为

```
atomic<int> done;
```

然后像使用普通变量一样使用done，就像在Java中一样。或者我们可以把一个普通的整型变量去掉。然后使用

```
atomic_store(&done, 1);
```

和：

```
while(atomic_load(&done) == 0) { /* loop */ }
```

去访问它。

无论哪种方式，完成的操作都参与原子操作的顺序一致的总顺序，并同步程序的其余部分。

C++还添加了较弱的原子，可以使用atomic_store_explicit和atomic_load_explicit以及附加的memory排序参数来访问这些原子。使用memory_order_seq_cst使显式调用等效于上面较短的调用。

较弱的原子称为acquire/release原子，一个release如果被后来的acquire观察到，那么就创建了一个happen-before的关系(从release到acquire)。这个术语意在唤起mutex:release就像unlock mutex，acquire就像lock同一个mutex。release之前执行的写入必须对后续acquire之后执行的读取可见，就像unlock mutex之前执行的写入必须对后来unlock mutex之后执行的读取可见一样。

The terminology is meant to evoke mutexes: release is like unlocking a mutex, and acquire is like locking that same mutex. The writes executed before the release must be visible to reads executed after the subsequent acquire, just as writes executed before unlocking a mutex must be visible to reads executed after later locking that same mutex.

为了使用较弱的原子，我们可以将消息传递示例改为:

```
atomic_store(&done, 1, memory_order_release);
```

和
```
while(atomic_load(&done, memory_order_acquire) == 0) { /* loop */ }
```

它仍然是正确的。但不是所有的程序都会这样

回想一下，顺序一致的原子要求程序中所有原子的行为与执行的一些全局交替执行(全局顺序)一致。acquire/release原子不会。它们只需要对单个内存位置的操作进行顺序一致的交替执行。也就是说，它们只需要一致性。结果是，一个使用具有多个存储位置的acquire/release原子的程序可能会观察到无法用程序中所有acquire/release原子的顺序一致的交替来解释的执行，这可以说是违反了DRF-SC！

为了说明不同之处，这里再举一个存储缓冲区的例子:

```
Litmus Test: Store Buffering

Can this program see r1 = 0, r2 = 0?

// Thread 1           // Thread 2

x = 1                 y = 1

r1 = y                r2 = x

On sequentially consistent hardware: no.

On x86 (or other TSO): yes!

On ARM/POWER: yes!

On Java (using volatiles): no.

On C++11 (sequentially consistent atomics): no.

On C++11 (acquire/release atomics): yes!
```

C++顺序一致的原子与Java的volatile相匹配。但是acquire-release原子在x的顺序和y的顺序之间没有强加任何关系。特别地，允许程序表现得好像r1 = y发生在y = 1之前，而同时r2 = x发生在x = 1之前，使得r1 = 0，r2 = 0与整个程序的顺序一致性相矛盾。为什么要引入这些较弱的获取/发布原子？可能是因为它们是x86上的普通内存操作。

请注意，对于观察特定写入的一组给定的特定读取，C++顺序一致原子和C++ acquire/release原子创建相同的happen-before关系。它们之间的区别在于，顺序一致的原子不允许观察特定写入的某些特定读取集，但acuqire/release原子允许这些特定读取集。一个这样的例子是导致存储缓冲测试出现r1 = 0，r2 = 0的结果。

acquire/release原子在实践中不如提供顺序一致性的原子有用。这里有一个例子。假设我们有一个新的同步原语，一个具有通知和等待两种方法的一次性条件变量。为了简单起见，只有一个线程会调用Notify，只有一个线程会调用Wait。我们想安排Notify在另一个线程还没有等待的时候是无锁的。我们可以用一对原子整数来实现:

```
class Cond {

	atomic<int> done;

	atomic<int> waiting;

	...

};

void Cond::notify() {

	done = 1;

	if (!waiting)

		return;

	// ... wake up waiter ...

}

void Cond::wait() {

	waiting = 1;

	if(done)

		return;

	// ... sleep ...

}
```

这段代码的重要部分是在检查waiting之前notify设置done为1, 为wait在检查done之前设置waiting为1,因此并发调用notify和wait不会导致notify立即返回并等待休眠。但是使用C++ acquire/release原子，它们可以。而且它们可能只需要很少的几率发生，使得这种错误很难重现和诊断。(更糟糕的是，在像64位ARM这样的一些架构上，实现acquire/release原子的最佳方式是顺序一致的原子，因此您可能会编写在64位ARM上运行良好的代码，但在移植到其他系统时才发现它是不正确的。)

基于这种理解，“acquire/release”对于这些原子来说是一个不幸的名字，因为顺序一致的原子做同样的acquire和release。不同之处在于顺序一致性的丧失。称这些为“一致性”原子可能更好。太迟了。

### Relaxed atomic

C++并没有仅仅停留在连贯的获取/发布原子上。它还引入了非同步原子，称为relaxed原子。这些原子根本没有同步效果——它们没有创建先发生的边——并且它们根本没有排序保证。事实上，宽松原子读/写和普通读/写没有区别，除了宽松原子上的竞争不被认为是竞争，不能着火。

C++没有停止与仅仅提供一致性的acquire/release原子。它还引入了非同步原子，称为relaxed原子(memory_order_relaxed)。这些原子根本没有同步效果——它们没有创建happens-before关系——并且它们根本没有排序保证。事实上，relaxed原子读/写和普通读/写没有区别，除了relaxed原子上的竞争不被认为是竞争，不能着火。

修改后的Java内存模型的大部分复杂性来自于定义具有数据竞争的程序的行为。如果C++采用DRF-SC或Catch Fire——实际上不允许有数据竞争的程序——意味着我们可以扔掉前面看到的所有奇怪的例子，那么C++语言规范将比Java语言规范更简单，那就太好了。不幸运的是，包括releaxed的原子最终保留了所有这些关注，这意味着C++11规范最终并不比Java简单。

像Java的内存模型一样，C++11的内存模型最终也是不正确的。考虑之前的无数据竞争计划:

```
Litmus Test: Non-Racy Out Of Thin Air Values

Can this program see r1 = 42, r2 = 42?

// Thread 1           // Thread 2

r1 = x                r2 = y

if (r1 == 42)         if (r2 == 42)

    y = r1                x = r2

(Obviously not!)

C++11 (ordinary variables): no.

C++11 (relaxed atomics): yes!
```

Viktor Vafeiadis和其他人在他们的论文[“Common Compiler Optimisations are Invalid in the C11 Memory Model and what we can do about it” (2015)](https://fzn.fr/readings/c11comp.pdf)中表明，C++11规范保证当x和y是普通变量时，该程序必须以x和y设置为零结束。但是如果x和y是relaxed的原子，那么，严格来说，C++11规范不排除r1和r2最终都可能达到42。(惊喜！)

详情见论文，但在较高的层次上，C++11规范有一些正式规则，试图禁止无中生有的值，并结合了一些模糊的词语来阻止其他类型的有问题的值。这些正式的规则就是问题所在，因此C++14放弃了它们，只留下了模糊的词语。引用删除它们的基本原理，C++11公式证明是“既不充分的，因为它使人们基本上无法对内存顺序放松的程序进行推理，也严重有害，因为它可以说不允许在ARM和POWER等体系结构上对memory_order_relaxed的所有合理实现。”

综上所述，Java试图正式排除所有不合法的执行，但失败了。然后，借助Java的后知后觉，C++11试图正式地只排除一些不合法的执行，也失败了。C++14然后说什么都不正式。这不是正确的方向。

事实上，Mark Batty和其他人在2015年发表的一篇题为[“编程语言并发语义的问题”](https://www.cl.cam.ac.uk/~jp622/the_problem_of_programming_language_concurrency_semantics.pdf)的论文给出了这一发人深省的评估:

> Disturbingly, 40+ years after the first relaxed-memory hardware was introduced (the IBM 370/158MP), the field still does not have a credible proposal for the concurrency semantics of any general-purpose high-level language that includes high-performance shared-memory concurrency primitives.
> 
> 令人不安的是，在引入第一个relaxed内存硬件(IBM 370/158MP)40多年后，该领域仍然没有一个可信的提案来描述任何包含高性能共享内存并发原语的通用高级语言的并发语义。

甚至定义弱有序硬件的语义(忽略软件和编译器优化的复杂性)也不太顺利。张思卓等人在2018年发表的一篇名为[《构建弱记忆模型》](https://arxiv.org/abs/1805.07886)的论文讲述了最近发生的一些事情:

> Sarkar et al. published an operational model for POWER in 2011, and Mador-Haim et al. published an axiomatic model that was proven to match the operational model in 2012. However, in 2014, Alglave et al. showed that the original operational model, as well as the corresponding axiomatic model, ruled out a newly observed behavior on POWER machines. For another instance, in 2016, Flur et al. gave an operational model for ARM, with no corresponding axiomatic model. One year later, ARM released a revision in their ISA manual explicitly forbidding behaviors allowed by Flur's model, and this resulted in another proposed ARM memory model. Clearly, formalizing weak memory models empirically is error-prone and challenging.
> 
> Sarkar等人在2011年公布了POWER的运行模型，Mador-Haim等人在2012年公布了一个公理化模型，该模型被证明与运行模型相匹配。然而，在2014年，Alglave等人表明，最初的操作模型以及相应的公理模型排除了在POWER机器上新观察到的行为。再比如，2016年，Flur等人给出了一个ARM的操作模型，没有对应的公理模型。一年后，ARM在他们的ISA手册中发布了一个修订版，明确规定了Flur模型允许的行为，这导致了另一个提出的ARM内存模型。显然，根据经验形式化弱记忆模型是容易出错且具有挑战性的。

在过去的十年里，致力于定义和形式化所有这些的研究人员非常聪明、有才华和坚持不懈，我并不想通过指出结果中的不足来贬低他们的努力和成就。我从这些简单的结论中得出结论，这个指定线程程序的确切行为的问题，即使没有竞争，也是难以置信的微妙和困难。如今，即使是最优秀、最聪明的研究人员似乎也无法理解这一点。即使不是，编程语言定义在日常开发人员可以理解的情况下效果最好，而不需要花费十年时间研究并发程序的语义。

## C, Rust 和 Swift 的内存模型

C11也采用了C++11内存模型，使其成为C/C++11内存模型。

2015年的[Rust 1.0.0](https://doc.rust-lang.org/std/sync/atomic/)和2020年的[Swift 5.3](https://github.com/apple/swift-evolution/blob/master/proposals/0282-atomics.md)都整体采用了C/C++内存模型，拥有DRF-SC或Catch Fire以及所有的原子类型和原子栅栏。

毫不奇怪，这两种语言都采用了C/C++ 模型，因为它们建立在C/C++编译器工具链(LLVM)上，并强调与C/C++代码的紧密集成。

### 硬件题外话： 有效的顺序一致性atomic

早期的多处理器体系结构有多种同步机制和内存模型，具有不同程度的可用性。在这种多样性中，不同同步抽象的效率取决于它们如何映射到架构所提供的内容。为了构造顺序一致的原子变量的抽象，有时唯一的选择是使用比严格必要的要多得多、贵得多的内存栅栏barriers，特别是在ARM和POWER上。

随着C、C++和Java都提供了这种顺序一致性同步原子的抽象，硬件设计者就应该让这种抽象变得高效。ARMv8架构(32位和64位)引入了ldar和stlr load和store指令，提供了直接的实现。在2017年的一次谈话中，赫伯·萨特声称，IBM已经批准了他的说法，他们希望未来的POWER实现对顺序一致的原子也有某种更有效的支持，这让程序员“更没有理由使用relaxed的原子。”我不知道是否发生了这种情况，尽管在2021年，POWER的相关性远不如ARMv8。

这种融合的效果是顺序一致的原子现在被很好地理解，并且可以在所有主要的硬件平台上有效地实现，这使得它们成为编程语言内存模型的一个很好的目标。

## JavaScript 内存模型 (2017)

你可能会认为JavaScript，一种众所周知的单线程语言，不需要担心内存模型，当代码在多个处理器上并行运行时会发生什么。我当然有。但是你和我都错了。

JavaScript有web workers，它允许在另一个线程中运行代码。按照最初的设想，工作人员只通过显式的消息复制与主JavaScript线程进行通信。没有共享的可写内存，就没有必要考虑像数据竞争这样的问题。然而，ECMAScript 2017 (ES2017)增加了SharedArrayBuffer对象，它让主线程和工作线程共享一块可写内存。为什么要这样做？在提案的[早期草稿](https://github.com/tc39/ecmascript_sharedmem/blob/master/historical/Spec_JavaScriptSharedMemoryAtomicsandLocks.pdf)中，列出的第一个原因是将多线程C++代码编译成JavaScript。

当然，共享可写内存还需要定义同步的原子操作和内存模型。JavaScript在三个重要方面偏离了C++:

-   首先，它将原子操作限制在顺序一致的原子上。其他原子可以被编译成顺序一致的原子，可能会损失效率，但不会损失正确性，只有一种原子可以简化系统的其余部分。
-   第二，JavaScript不采用“DRF-SC或着火。”相反，像Java一样，它仔细定义了竞争访问的可能结果。其原理与Java非常相似，尤其是安全性。允许竞争read返回任何值允许(可以说是鼓励)实现返回不相关的数据，这可能会导致运行时[私有数据的泄漏](https://github.com/tc39/ecmascript_sharedmem/blob/master/DISCUSSION.md#races-leaking-private-data-at-run-time)。
-   第三，部分是因为JavaScript为竞争程序提供了语义，它定义了当原子和非原子操作在同一个内存位置使用时，以及当使用不同大小的访问访问同一个内存位置时会发生什么。

精确定义racy程序的行为会导致relaxed内存语义的复杂性，以及如何禁止无中生有的读取和类似情况。除了这些与其他地方基本相同的挑战之外，ES2017定义还有两个有趣的错误，它们是由于与新的ARMv8原子指令的语义不匹配而引起的。这些例子改编自康拉德·瓦特等人2020年的论文[“Repairing and Mechanising the JavaScript Relaxed Memory Model.”](https://www.cl.cam.ac.uk/~jp622/repairing_javascript.pdf)

正如我们在上一节中提到的，ARMv8增加了ldar和stlr指令，提供顺序一致的原子加载和存储。这些是针对C++的，它没有定义任何具有数据竞争的程序的行为。因此，毫不奇怪，这些指令在竞争程序中的行为与ES2017作者的期望不符，尤其是它不符合ES2017对竞争程序行为的要求。

```
Litmus Test: ES2017 racy reads on ARMv8

Can this program (using atomics) see r1 = 0, r2 = 1?

// Thread 1           // Thread 2

x = 1                 y = 1

r1 = y                x = 2 (non-atomic)

                      r2 = x

C++: yes (data race, can do anything at all).

Java: the program cannot be written.

ARMv8 using ldar/stlr: yes.

ES2017: no! (contradicting ARMv8)
```

在这个程序中，所有的读和写都是顺序一致的原子，除了x = 2:线程1使用原子存储写x = 1，但是线程2使用非原子存储写x = 2。在C++中，这是一场数据竞争，所以所有的赌注都取消了。在Java中，这个程序是不能写的:x必须要么声明为volatile，要么不声明；它有时不能被原子访问。在ES2017中，内存模型不允许r1 = 0，r2 = 1。如果r1 = y读取0，线程1必须在线程2开始之前完成，在这种情况下，非原子x = 2似乎发生在x = 1之后并覆盖x = 1，导致原子r2 = x读取2。这个解释似乎完全合理，但这不是ARMv8处理器的工作方式。

事实证明，对于ARMv8指令的等效序列，对x的非原子写可以在对y的原子写之前重新排序，因此该程序实际上产生r1 = 0，r2 = 1。这在C++中不是问题，因为竞争意味着程序可以做任何事情，但对于ES2017来说，这是一个问题，它将竞争行为限制在一组不包括r1 = 0、r2 = 1的结果上

由于ES2017的明确目标是使用ARMv8指令来实现顺序一致的原子操作，Watt等人报告说，他们建议的修复(计划包含在标准的下一个修订版中)将削弱竞争行为约束，足以允许这种结果。(当时我不清楚“下一次修订”是指ES2020还是ES2021。)

Watt等人提出的修改还包括对第二个bug的修复，第一个bug是由Watt、Andreas Rossberg和Jean Pichon-pharabad提出的，其中一个无数据竞争的程序没有按照ES2017规范给出顺序一致的语义。该程序由下式给出:

```
Litmus Test: ES2017 data-race-free program

Can this program (using atomics) see r1 = 1, r2 = 2?

// Thread 1           // Thread 2

x = 1                 x = 2

                      r1 = x

                      if (r1 == 1) {

                          r2 = x // non-atomic

                      }

On sequentially consistent hardware: no.

C++: I'm not enough of a C++ expert to say for sure.

Java: the program cannot be written.

ES2017: yes! (violating DRF-SC).
```

在这个程序中，所有的读和写都是顺序一致的原子，除了r2 = x，标记为。这个程序是无数据竞争的:非原子读取必须参与任何数据竞争，只有当r1 = 1时才执行，这证明线程1的x = 1发生在r1 = x之前，因此也发生在r2 = x之前。DRF-SC意味着程序必须以顺序一致的方式执行，因此r1 = 1，r2 = 2是不可能的，但ES2017规范允许这样做。

因此，ES2017程序行为规范同时太强(它不允许racy程序的真实ARMv8行为)和太弱(它允许无竞争程序的非顺序一致行为)。如前所述，这些错误已经改正。即便如此，这也再次提醒我们，精确地使用以前发生的事情来指定无数据竞争程序和活泼程序的语义是多么微妙，以及将语言内存模型与底层硬件内存模型相匹配是多么微妙。

令人鼓舞的是，至少到目前为止，除了顺序一致的原子之外，JavaScript避免了添加任何其他原子，并抵制了“DRF-SC或着火”结果是内存模型作为C/C++编译目标是有效的，但更接近于Java。

## 结论

看看C、C++、Java、JavaScript、Rust和Swift，我们可以得出以下结论:

-   它们都提供顺序一致的同步原子，用于协调并行程序的非原子部分。
-   它们的目的都是确保程序使用适当的同步来避免数据竞争，就像以顺序一致的方式执行一样。
-   Java和JavaScript避免了引入弱(acquire/release)同步原子，这似乎是为x86量身定制的。
-   它们都为程序提供了一种方式来执行“有意的”数据竞争，而不会使程序的其余部分无效。在C、C++、Rust和Swift中，这种机制是relaxed，非同步原子，一种特殊的内存访问形式。在Java和JavaScript中，这种机制就是普通的内存访问。
-   没有一种语言找到了正式禁止悖论的方法，比如无中生有的值，但是所有语言都非正式地禁止它们。

与此同时，处理器制造商似乎已经接受了顺序一致同步原子的抽象对于高效实现非常重要，并开始这样做：ARMv8和RISC-V都提供了直接支持。

最后，真正大量的验证和形式分析工作已经进入了理解这些系统和精确陈述它们的行为。特别令人鼓舞的是，瓦特等人在2020年能够给出一个JavaScript重要子集的正式模型，并使用定理证明器来证明编译对ARM、POWER、RISC-V和x86-TSO的正确性。

在第一个Java内存模型问世25年后，经过许多人世纪的研究努力，我们可能开始能够形式化整个内存模型。也许，有一天，我们也会完全理解他们。


# [译]更新Go内存模型

这是Russ Cox的系列论文的第三篇，也是最后一篇: [Updating the Go Memory Model](https://research.swtch.com/gomm)。

文章对 官方的Go内存模型做了一些补充和思考。

[![](https://colobu.com/2021/07/13/Updating-the-Go-Memory-Model/gophers-racing.jpeg)](https://colobu.com/2021/07/13/Updating-the-Go-Memory-Model/gophers-racing.jpeg)

当前的Go语言内存模型是在2009年编写的，从那以后略有更新。很明显，至少有一些细节我们应该添加到当前的内存这个内存模型中，其中包括对竞态检测器的明确认可，以及关于sync/atomic中的API是如何同步程序的清晰声明。

这篇文章重申了Go的总体哲学和当前的内存模型，然后概述了我认为我们应该对Go内存模型进行的相对较小的调整。假定你已经了解了前两篇文章[“硬件内存模型”](https://colobu.com/2021/06/30/hwmm/)和[“编程语言内存模型”](https://colobu.com/2021/07/11/Programming-Language-Memory-Models/)中的背景知识。

我已经开启了一个[GitHub讨论项目](https://golang.org/s/mm-discuss)来收集对反馈。根据这些反馈，我打算在本月晚些时候准备一份正式的Go提案。使用GitHub讨论本身就是一个实验，我还会继续尝试[找到一个合理的方法来扩大这些重要变化的讨论](https://research.swtch.com/proposals-discuss)。

## Go 设计哲学

Go旨在成为构建实用、高效系统的编程环境。它的目标是为小型项目的轻量级开发语言，但也可以优雅地扩展到大型项目和大型工程团队。

Go鼓励在高层次上处理并发，特别是通过通信。第一句Go箴言([Go proverb](https://go-proverbs.github.io/))就是“不要通过共享内存来通信，而是通过通信共享内存。”另一个流行的谚语是“清晰胜于聪明。”换句话说，Go鼓励通过避免使用巧妙的代码来避免狡猾的bug。

Go的目标不仅仅是可以理解的程序，还包括可以理解的语言和可以理解的package API。复杂或巧妙的语言特征或API与这一目标相矛盾。正如Tony Hoare在1980年[图灵奖演讲](https://www.cs.fsu.edu/~engelen/courses/COP4610/hoare.pdf)中所说:

> I conclude that there are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies and the other way is to make it so complicated that there are no obvious deficiencies.
> 
> 我的结论是，构建软件设计有两种方法:一种方法是简单实现，以至于明显没有缺陷；另一种方法是异常复杂，以至于没有明显缺陷。

第一种方法要困难得多。它需要同样的技巧、奉献、洞察力，甚至是灵感，就像发现构成复杂自然现象基础的简单物理定律一样。它还要求愿意接受受物理、逻辑和技术限制的目标，并在冲突的目标无法实现时接受妥协。

这与Go的设计API的理念非常吻合。我们通常在设计过程中花很长时间来确保一个应用编程接口是正确的，并努力将其简化为最基本、最有用的精华。

Go作为一个有用的编程环境的另一方面是为最常见编程错误有定义明确的语义，这有助于理解和调试。这个想法并不新鲜。再次引用Tony Hoare的话，这是来自他1972年的“软件质量”检查单:

> As well as being very simple to use, a software program must be very difficult to misuse; it must be kind to programming errors, giving clear indication of their occurrence, and never becoming unpredictable in its effects.
> 
> 一个软件程序不仅使用起来非常简单，而且很难被误用；它必须友好对待编程错误，给出它们发生的明确指示，并且其影响永远不会变得不可预测。

为有问题的程序定义良好的语义，这种常识并不像人们预期的那样普遍。在C/C++中，未定义的行为已经演变成一种编译器作者的全权委托，以越来越有趣的方式将有轻微问题的程序转换成有大问题的程序。Go不采用这种方法:不存在“未定义的行为”。特别是，像空指针取消引用、整数溢出和无意的无限循环这样的错误都在Go中定义了语义。

## 当前的 Go内存模型

Go的内存模型始于以下建议，符合Go的总体哲学:

-   修改由多个goroutines同时访问的数据的程序必须串行化这些访问。
-   为了实现串行访问, 需要使用channel操作或其他同步原语(如sync和sync/atomic包中的原语)来保护数据。
-   如果你必须阅读本文的其余部分才能理解你的程序的行为，那你太聪明了。
-   别自作聪明。

这仍然是个好建议。该建议也与其他语言对DRF-SC的鼓励使用一致:同步以消除数据竞争，然后程序将表现得好像顺序一致，不需要理解内存模型的其余部分。

根据这个建议，Go内存模型定义了一个传统的基于happens-before对读写竞争的定义。像在Java和JavaScript中一样，在Go中的读操作可以观察到任何更早但尚未被覆盖的写操作，或者任何竞争的写操作；仅安排一个这样的写入会强制产生指定的结果。

然后，内存模型继续定义同步操作，这些操作建立交替执行的goroutine的happen-before关系。操作尽管稀松平常，但是还是带有一些Go特有的风格:

-   如果package **p**引入了package **q**,那么**q**的init函数的执行完成一定happen-before **p**的所有init函数(之前)
-   **main.main** 函数一定 happen after 所有的init函数完成(之后)
-   go语句创建一个goroutine一定happen before goroutine执行(之前)
-   往一个channel中send happen before 从这个channel receive这个数据完成(之前)
-   一个channel的close一定happen before 从这个channel receive到零值数据(这里指因为close而返回的零值数据)
-   从一个unbuffered channel的receive一定happen before 往这个channel send完成(之前)
-   从容量为C的channel receive第k个数据一定happen before第k+C次send完成(之前)
-   对于任意的**sync.Mutex**或者**sync.RWMutex**类型的变量l以及n < m, 调用第n次l.UnLock()一定happen before 第m次的l.Lock()返回(之前)
-   once.Do(f)中的对f的单次调用一定happen before 任意次的对once.Do(f)调用返回(之前)

值得注意的是，这个列表忽略了package sync中新加的API以及sync/atomic的API。

Go内存模型规范以一些不正确同步的例子结束。它没有包含错误编译的例子。

## 对Go内存模型做的改变

2009年，当我们着手编写Go的内存模型时，Java内存模型进行了新的修订，C/C++11内存模型正在定稿。一些人强烈鼓励我们采用C/C++11模型，并充分利用了其已经完成的所有工作。对我们来说这似乎很冒险。相反，我们决定采用一种更保守的方法来保证我们要做的，这一决定得到了随后十年详细描述Java/C/C++内存模型中非常狡猾问题的论文的证实。是的，定义足够充分的内存模型来指导程序员和编译器作者是很重要的，但是完全正式地定义一个正确的内存模型似乎仍然超出了最有才华的研究人员的能力范围。Go定义一个最小的需求就足够了。

下面这一部分列出了我认为我们应该做的调整。如前所述，我已经开启了一个[GitHub讨论项](https://golang.org/s/mm-discuss)来收集反馈。根据这些反馈，我计划在本月晚些时候准备一份正式的Go提案。

### 文档化Go的整体方法

“不要聪明”的建议很重要，应该坚持下去，但我们也需要在深入研究happen before细节之前，对Go的整体方法更多的谈一谈。我看到过很多关于Go方法的不正确总结，比如宣称Go的模型是C/C++的“DRF-SC或Catch Fire”。 这种误会是可以理解的: Go内存模型规范没有说它的方法是什么，而且它是如此之短(材料又如此微妙)，以至于人们看到了他们期望看到的东西，而不是那里有什么或没有什么。

拟在Go内存模型规范中增加的文档大致如下:

> ## 概观
> 
> Go以与本语言其余部分几乎相同的方式处理其内存模型，旨在保持语义简单、可理解和有用。
> 
> 数据竞争被定义为对存储器位置的写入与对同一位置的另一次读取或写入同时发生，除非所有访问都是由sync/atomic package提供的原子数据访问提供。如前所述，强烈建议程序员使用适当的同步来避免数据竞争。在没有数据竞争的情况下，Go程序表现得好像所有的gorouitine都被多路复用到一个处理器上。这个属性有时被称为DRF-SC:无数据竞争的程序以顺序一致的方式执行。
> 
> 其他编程语言通常采用两种方法之一来处理包含数据竞争的程序。第一，以C和C++为例，带有数据竞争的程序是无效的:编译器可能会以任意令人惊讶的方式中断它们。第二，以Java和JavaScript为例，具有数据竞争的程序定义了语义，通过限制竞争的可能影响，使程序更加可靠和易于调试。Go的方法介于这两者之间。具有数据竞争的程序是无效的，因为语言实现可能会报告竞争并终止程序。但另一方面，具有数据竞争的程序定义了具有有限数量结果的语义，使得错误的程序更可靠，更容易调试。

这些文字应该阐明Go和其他语言有什么不同，纠正读者先前的任何期望。

在“happen before”一节的最后，我们还应该澄清某些竞争仍然会导致内存损坏。当前它以下面的句子结束：

> Reads and writes of values larger than a single machine word behave as multiple machine-word-sized operations in an unspecified order.

我们应该加上一点:

> 请注意，这意味着多word数据结构上的竞争可能导致单次写入产生不一致值。当值依赖于内部(指针、长度)或(指针、类型)pair的一致性时，就像大多数Go实现中的接口、map、切片和字符串的情况一样，这种竞争又会导致内存损坏。

这将更清楚地说明保证对具有数据竞争的程序的限制。

### 文档化 sync库的happen before

自从Go内存模型发布以来，一些新的API已经被添加到sync包中。我们需要将它们添加到内存模型中([issue#7948](https://golang.org/issue/7948))。谢天谢地谢广坤，增加的内容看起来很简单。我相信它们应该如下：

-   对于sync.Cond, **Broadcast** 和 **Signal** 一定happen before 它解锁的Wait方法调用完成(之前)
-   对于sync.Map, Load, LoadAndDelete 和 LoadOrStore 都是读操作， Delete、LoadAndDelete和 Store都是写操作。LoadOrStore当它的loaded返回false时是写操作。一个写操作happen before 能观察到这个写操作的读操作(之前)
-   对于sync.Pool,对Put(x)的调用一定happen before Get方法返回这个x(之前)。同样的，返回x的New方法一定happen before Get方法返回这个x(之前)
-   对于sync.EWaitGroup, Done方法的调用一定happen before 它解锁的Wait方法调用返回(之前)

这些API的用户需要知道保证，以便有效地使用它们。因此，虽然我们应该将这些文字保留在内存模型中以供介绍，但我们也应该将其包含在package sync的文档注释中。这也将有助于为第三方同步原语树立一个榜样，说明记录由API建立的顺序保证的重要性。

### 文档话 sync/atomic的happen before

Atomic operations are missing from the memory model. We need to add them (issue #5045). I believe we should say:

内存模型中缺少原子操作的保证。我们需要添加它们([issue #5045](https://golang.org/issue/5045))。我认为我们应该说:

> sync/atomic package中的API统称为“原子操作”，可用于同步各种goroutine执行。如果原子操作A的效果被原子操作B观察到，那么A发生在B之前(happen before)。在一个程序中执行的所有原子操作表现得好像是以某种顺序一致的顺序执行的。

这是Dmitri Vyukov在2013年提出的建议，也是我在2016年非正式承诺的。它还与Java的volatiles和C++的默认原子具有相同的语义。

就C/C++而言，同步原子只有两种选择:顺序一致或acquire/release(Relaxed原子不会创建happen before，因此没有同步效果). 对这两者的决策归结为，第一，能够推理出多个位置上原子操作的相对顺序有多重要，第二，顺序一致的原子与acquire/release原子相比要多昂贵(慢)。

首先要考虑的是，关于多个位置上原子操作的相对顺序的推理非常重要。在之前的一篇文章中，我举了一个使用两个原子变量实现的无锁快速路径的条件变量的[例子](https://research.swtch.com/plmm#cond)，这两个原子变量被使用acuqire/release原子打破了。这种模式反复出现。例如，sync.WaitGroup曾经的实现使用了一对[原子uint32值](https://go.googlesource.com/go/+/ee6e1a3ff77a41eff5a606a5aa8c46bf8b571a13/src/pkg/sync/waitgroup.go#54)：wg.counter和wg.waiters。[Go运行时中的信号量的实现](https://go.googlesource.com/go/+/cf148f3d468f4d0648e7fc6d2858d2afdc37f70d/src/runtime/sema.go#134)也依赖于两个独立的原子word，即信号量值*addr和相应的waiter count root.nwait。还有更多。在缺乏顺序一致的语义的情况下(也就是说，如果我们改为采用acquire/release语义)，人们仍然会像这样错误地编写代码；它会神秘地失败，而且只在特定的情况下。

根本的问题是，使用acuqire/release原子使无数据竞争的程序不会导致程序以顺序一致的方式运行，因为原子本身不会提供保证。也就是说，这样的程序不提供DRF-SC。这使得这种程序很难推理，因此很难正确编写。

关于第二个考虑，正如在之前的文章中提到的，硬件设计人员开始为顺序一致的原子提供直接支持。例如，ARMv8添加了ldar和stlr指令来实现顺序一致的原子，它们也是acquire/release原子的推荐实现。如果我们为sync/atomic采用acquire/release语义，那么写在ARMv8上的程序无论如何都会获得顺序一致性。这无疑会导致依赖更强顺序的程序意外地在更弱的平台上崩溃。，如果由于竞争窗口很小, acquire/release和结果一致的原子之间的差异在实践中很难观察到，这甚至可能发生在单个架构上。

这两种考虑都强烈建议我们应该采用顺序一致的原子而不是acquire/release原子:顺序一致的原子更有用，一些芯片已经完全缩小了这两个级别之间的差距。如果差距很大，想必其他人也会这么做。

同样的考虑，以及Go拥有小型、易于理解的API的总体哲学，所有这一切都反对将acuqire/release作为一套额外的并行API来提供。似乎最好只提供最容易理解的，最有用的，很难被误用的原子操作。

另一种可能性是提供原始屏障，而不是原子操作(当然，C++两者都提供)。屏障的缺点是使期望变得不那么清晰，并且在某种程度上更加局限于特定的体系结构。Hans Boehm文章[“Why atomics have integrated ordering constraints”](http://www.hboehm.info/c++mm/ordering_integrated.html)给出了提供原子而不是屏障的论点(他使用术语栅栏fence)。一般来说，原子比栅栏更容易理解，而且由于我们现在已经提供了原子操作，所以我们不能轻易移除它们。一个机制要比提供两个好。

### 可能的改变： 为sync/atomic提供类型化的API

上面的定义说，当一个特定的内存块必须由多个线程同时访问而没有其他同步时，消除争用的唯一方法是让所有的访问都使用原子。仅仅让一些访问使用原子是不够的。例如，与原子读或写并发的非原子写仍然是s数据竞争，与非原子读或写并发的原子写也是数据竞争。

因此，一个特定的值是否应该用atomic访问是该值的属性，而不是特定的访问。正因为如此，大多数语言将这些信息放在类型系统中，比如Java的volatile int和C++的atomic。Go当前的API没有，这意味着正确的使用需要仔细标注结构或全局变量的哪些字段预计只能使用原子API来访问。

> 译者按: uber提供了类似的库[uber-go/atomic](https://github.com/uber-go/atomic)。

为了提高程序的正确性，我开始认为Go应该定义一组类型化的原子值，类似于当前的原子值。值:Bool、Int、Uint、Int32、Uint32、Int64、Uint64和Uintptr。像Value一样，它们也有CompareAndSwap、Load、Store和Swap方法。例如:

```go
type Int32 struct { v int32 }

func (i *Int32) Add(delta int32) int32 {

	return AddInt32(&i.v, delta)

}

func (i *Int32) CompareAndSwap(old, new int32) (swapped bool) {

	return CompareAndSwapInt32(&i.v, old, new)

}

func (i *Int32) Load() int32 {

	return LoadInt32(&i.v)

}

func (i *Int32) Store(v int32) {

	return StoreInt32(&i.v, v)

}

func (i *Int32) Swap(new int32) (old int32) {

	return SwapInt32(&i.v, new)

}
```

我将Bool包括在列表中，因为我们在Go标准库中多次用原子整数构造了原子Bool(在未暴露的API中)。显然是有需要的。

我们还可以利用即将到来的泛型支持，并为原子指针定义一个API，该API是类型化的，并且在其API中没有包不安全:

```go
type Pointer[T any] struct { v *T }

func (p *Pointer[T]) CompareAndSwap(old, new *T) (swapped bool) {

	return CompareAndSwapPointer(... lots of unsafe ...)

}
```

(以此类推),你可能会想到不能使用泛型定义一个类型吗？我没有看到一个干净的方法使用泛型来实现atomic.Atomic[T]，避免我们引入Bool、Int等作为单独的类型。走走看吧。

### 可能的改变: 增加非同步的atomic

所有其他现代编程语言都提供了一种方法来进行并发内存读写，这种方法不会使程序同步，但也不会使程序无效(不会算作数据竞争)。C、C++、Rust和Swift都有relaxed原子。Java有VarHandle的“普通”模式。JavaScript对共享内存缓冲区(唯一的共享内存)有非原子的访问权限。Go没有办法做到这一点。或许应该有，我不知道。

如果我们想添加非同步的原子读写，我们可以向类型化的原子添加UnsyncAdd、UnsyncCompareAndSwap、UnsyncLoad、UnsyncStore和 UnsyncSwap方法。将它们命名为“unsync”避免了一些“relaxed”名称的问题。首先，有些人用relaxed作为相对的比较，如“acquire/release是比顺序一致性更宽松的内存顺序。”你可以说这不是这个术语的恰当用法，但它确实发生了。其次，也是更重要的，这些操作的关键细节不是操作本身的内存排序，而是它们对程序其余部分的同步没有影响。对于不是内存模型专家的人来说，看到UnsyncLoad应该清楚没有同步，而RelaxedLoad可能不会。在人群中喵一眼Unsync也知道它是不安全的。

有了API，真正的问题是到底要不要添加这些。对提供非同步原子的争论是，它确实对某些数据结构中快速路径的性能有影响。我的总体印象是，它在非x86架构上最重要，尽管我没有数据来支持这一点。不提供不同步的原子可以被认为是对那些架构的惩罚。

反对提供非同步原子的一个可能的争论是，在x86上，忽略了潜在的编译器重组的影响，非同步原子与acquire/release原子是无法区分的。因此，他们可能会被滥用来编写只适用于x86的代码。反驳的理由是，这样的花招不会通过race检测器，它实现的是实际的内存模型，而不是x86内存模型。

由于缺乏证据，我们没有理由添加这个API。如果有人强烈认为我们应该添加它，那么证明这一点的方法是收集两方面的证据:(1)程序员需要编写的代码的普遍适用性，以及(2)使用非同步原子对广泛使用的系统产生的显著性能改进。(使用Go以外的语言的程序来显示这一点是很好的。)

### 文档化对编译器优化的禁止项

当前的内存模型最后给出了无效程序的例子。由于内存模型是程序员和编译器作者之间的契约，我们应该添加无效编译器优化的例子。例如，我们可以添加:

#### 不正确的编译

Go内存模型和Go程序一样限制编译器优化。一些在单线程程序中有效的编译器优化在Go程序中是无效。特别是，编译器不能在无竞争程序中引入数据竞争。它不能允许单次读取观察到多个值。并且它不能允许一个写操作写入多个值。

Not introducing data races into race-free programs means not moving reads or writes out of conditional statements in which they appear. For example, a compiler must not invert the conditional in this program:

不在无竞争程序中引入数据竞争意味着不移动出现条件语句的读或写。例如，编译器不得反转该程序中的条件:

```go
i := 0

if cond {

	i = *p

}
```

也就是说，编译器不能将程序重写为这个:

```go
i := *p

if !cond {

	i = 0

}
```

如果cond为false，另一个goroutine正在写*p，那么原始程序是无竞争的，但是重写的程序包含竞争。

不引入数据竞争也意味着不假设循环终止。例如，在这个程序中，编译器不能将对_p或_q访问移动到循环前面:

```go
n := 0

for e := list; e != nil; e = e.next {

	n++

}

i := *p

*q = 1
```

如果列表指向循环列表，那么原始程序永远不会访问_p或_q，但是重写的程序会。

不引入数据竞争也意味着不假设被调用的函数总是返回或者没有同步操作。例如，在这个程序中，编译器不能移动对_p或_q访问到函数调用之前:
```
f()

i := *p

*q = 1
```

如果调用从未返回，那么原始程序将不会再访问_p或_q，但是重写的程序会。如果调用包含同步操作，那么原始程序可以建立f和_p/_q的happen before关系，但是重写的程序就破坏了这个关系。

不允许单次读取观察多个值,意味着不从共享内存中重新加载局部变量。例如，在这个程序中，编译器不能扔掉(spill)i,并重新加载它:

```
i := *p

if i < 0 || i >= len(funcs) {

	panic("invalid function index")

}

... complex code ...

// compiler must NOT reload i = *p here

funcs[i]()
```

如果复杂的代码需要许多寄存器，单线程程序的编译器可以在不保存副本的情况下丢弃i，然后在funcs[i](https://colobu.com/2021/07/13/Updating-the-Go-Memory-Model/)之前重新加载i = \_p。Go编译器不能，因为_p的值可能已经更改。(相反，编译器可能会将i移动到栈上)。

不允许一次写操作写入多个值也意味着不使用在写入之前将本地变量作为临时存储写入的内存。例如，编译器不得在此程序中使用\*p作为临时存储:
```
*p = i + *p/2
```

也就是说，它绝不能把程序改写成这样:

```
*p /= 2

*p += i
```

如果i和_p开始等于2，则原始代码最终_p = 3，但是一个竞争线程只能从_p读取2或3。重写后的代码最终_p = 1，然后\*p = 3，这也允许竞争线程读取1。

请注意，所有这些优化在C/C++编译器中都是允许的:与C/C++编译器共享后端的Go编译器必须注意禁用对Go无效的优化。

这些分类和示例涵盖了最常见的C/C++编译器优化，这些优化与为竞争数据访问定义的语义不兼容。他们明确规定Go和C/C++有不同的要求。

## 结论

Go在其内存模型中保守的方法很好地服务了我们，应该继续下去。然而，有一些早该做的更改，包括定义sync和sync/package package中新API的同步行为。特别是atomic的内存模型应该被文档化，其以提供顺序一致的行为，这种行为创建了与它们左右的非原子代码同步的happen before关系。这与所有其他现代系统语言提供的默认原子相匹配。

也许更新中最独特的部分是清楚地声明具有数据竞争的程序可能会被停止以报告竞争，但是在其他方面具有明确定义的语义。这约束了程序员和编译器，它优先考虑并发程序的可调试性和正确性，而不是编译器编写者的便利性。


# Reference
https://colobu.com/2021/06/30/hwmm/#%E5%BC%B1%E6%8E%92%E5%BA%8F%E5%92%8C%E6%97%A0%E6%95%B0%E6%8D%AE%E7%AB%9E%E4%BA%89%E7%9A%84%E9%A1%BA%E5%BA%8F%E4%B8%80%E8%87%B4%E6%80%A7

https://colobu.com/2022/09/12/go-synchronization-is-hard/

https://colobu.com/2021/07/11/Programming-Language-Memory-Models/

https://colobu.com/2021/07/13/Updating-the-Go-Memory-Model/