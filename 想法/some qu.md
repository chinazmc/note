## 2.选择三个常见 golang 组件

channel, goroutine, [], map, sync.Map等，

列举它们常见的严重伤害性能的 anti-pattern?

select 很多 channel 的时候，并发较高时会有性能问题。因为 select 本质是按 chan 地址排序，顺序加锁。lock1->lock2->lock3->lock4 活跃 goroutine 数量较多时，会导致全局的延迟不可控。比如 99 分位惨不忍睹。

slice append 的时候可能触发扩容，初始 cap 不合适会有大量 growSlice。
map hash collision，会导致 overflow bucket 很长，但这种基本不太可能，hash seed 每次启动都是随机的。此外，map 中 key value 超过 128 字节时，会被转为 indirectkey 和 indirectvalue，会对 GC 的扫描阶段造成压力，如果 k v 多，则扫描的 stw 就会很长。
sync.Map 有啥性能问题？写多读少的时候？

## 3.noCopy

noCopy原理：

```
type noCopy struct {}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

## 4.sync.Pool源码分析

-   Get 的整个过程就非常清晰了：

首先，调用 p.pin() 函数将当前的 goroutine 和 P 绑定，禁止被抢占，返回当前 P 对应的 poolLocal，以及 pid。然后直接取 l.private，赋值给 x，并置 l.private 为 nil。判断 x 是否为空，若为空，则尝试从 l.shared 的头部 pop 一个对象出来，同时赋值给 x。如果 x 仍然为空，则调用 getSlow 尝试从其他 P 的 shared 双端队列尾部“偷”一个对象出来,偷不到就去victim的private中拿，拿不到就是victim的shared中拿。Pool 的相关操作做完了，调用 runtime_procUnpin() 解除非抢占。最后如果还是没有取到缓存的对象，那就直接调用预先设置好的 New 函数，创建一个出来。

-   Put 的逻辑也很清晰：

先绑定 g 和 P，然后尝试将 x 赋值给 private 字段。如果失败，就调用 pushHead 方法尝试将其放入 shared 字段所维护的双端队列中

## 5.请列举两种不同的观察 GC 占用 CPU 程度的方法，观察方法无需绝对精确，但需要可实际运用于 profiling 和 optimization。

-   pprof 的 cpuprofile 火焰图 启动时传入环境变量 Debug=gctrace
-   go tool trace，可以看到 p 绑定的 g 实际的 GC 动作和相应时长，以及阻塞时间

## 6.JSON 标准库对 nil slice 和 空 slice 的处理是一致的吗

```
package main
import (    "encoding/json"    "log")
type S struct {    
	A []string
}
func main() {    
data := &S{}    
data2 := &S{A: []string{}
}    
	buf, err := json.Marshal(&data)    
	log.Println(string(buf), err)    
	buf2, err2 := json.Marshal(&data2)    
	log.Println(string(buf2), err2)
}
```

输出：

```
2009/11/10 23:00:00 {"A":null} <nil>
2009/11/10 23:00:00 {"A":[]} <nil>
```

## 7.string与[]byte转换

结论：string与[]byte转换过程中的Data都会发生拷贝，例如：

像如下代码会进行优化。

```go
func Equal(a, b []byte) bool { 
	// Neither cmd/compile nor gccgo allocates for these string conversions. 
	return string(a) == string(b)
}
```

-   []byte转string没问题 提供的String方法就是将[]byte转换为string类型，这里为了避免内存拷贝的问题，使用了强制转换来避免内存拷贝：
```
func (b *Builder) String() string { return *(*string)(unsafe.Pointer(&b.buf))}
```

-   string转[]byte有问题 这种直接转换有风险，string只有raw ptr和length，没有capacity的概念，当reslice或者append等操作时，需要capacity，此时会panic。
```
func NoAllocBytes(buf string) []byte { return *(*[]byte)(unsafe.Pointer(&buf))}
```

正确做法：

```
func NoAllocBytes(buf string) []byte { 
		x := (*reflect.StringHeader）(unsafe.Pointer(&buf)) 
		h := reflect.SliceHeader{x.Data, x.Len, x.Len} 
		return *(*[]byte)(unsafe.Pointer(&h))
}
```

## 11.字符串拼接
-   使用+操作符进行拼接时，会对字符串进行遍历，计算并开辟一个新的空间来存储原来的两个字符串。
-   fmt 由于采用了接口参数，必须要用反射获取值，因此有性能损耗。
-   strings.Builder 用WriteString()进行拼接，内部实现是指针+切片，同时String()返回拼接后的字符串，它是直接把[]byte转换为string，从而避免变量拷贝。strings.Builder使用问题，内部有addr(是否指向自身，copyCheck检查)，buf([]byte数组，存放内容，写入时append方式)。
注意点：两个strings.Builder禁止在写入之后拷贝，写入之前可以拷贝，如下例子,主要原因是写入会触发copyCheck检查。

```
var b1 strings.Builder
b1.WriteString("ABC")
b2 := b1// b2.WriteString("DEF") 失败 在写操作会进行copyCheck -> 如果内存空 会操作addr，不空则会判断地址是否一致
fmt.Println(b1, b2)
var b3, b4 strings.Builder
b4 = b3 // 一开始都为空 所以可以进行copy
b3.WriteString("123")
b4.WriteString("456")
fmt.Println(b3.String(), b4.String())
```

-   bytes.Buffer bytes.Buffer是一个一个缓冲byte类型的缓冲器，这个缓冲器里存放着都是byte，包含一个offset变量，控制尾部写入数据，从头部读取数据，
-   strings.Join 基于strings.Builder构建，支持分隔符拼接，内部调用时会先Grow, 以提前进行容量分配可以减少内存分配，很高效。strings.Join ≈ strings.Builder > bytes.Buffer > "+" > fmt.Sprintf

## 12.类型比较

Go 结构体有时候并不能直接比较，当其基本类型包含：slice、map、function 时，是不能比较的。若强行比较，就会导致出现例子中的直接报错的情况。reflect.DeepEqual来判断不可比较类型，如:map、slice、func等

```
type Value struct {    Name   string    Gender *string}
func main() {    
v1 := Value{Name: "1", Gender: new(string)}    
v2 := Value{Name: "1", Gender: new(string)}    
if v1 == v2 {        
fmt.Println("yes")        
return    
}    
fmt.Println("no")}
```

输出：no，因为地址不一样

## 14.Go垃圾回收机制

垃圾回收机制是Go一大特(nan)色(dian)。Go1.3采用标记清除法， Go1.5采用三色标记法，Go1.8采用三色标记法+混合写屏障。

**标记清除法(mark and sweep):**

分为两个阶段：标记和清除 第一步，暂停程序业务逻辑, 分类出可达和不可达的对象 第二步, 开始标记，程序找出它所有可达的对象，并做上标记 第三步,  标记完了之后，然后开始清除未标记的对象. 第四步, 停止暂停，让程序继续跑。然后循环重复这个过程，直到process程序生命周期结束

缺点：
-   STW，stop the world；让程序暂停，程序出现卡顿
-   标记需要扫描整个heap；
-   清除数据会产生heap碎片。早期优化，缩短STW时间，原来STW包括：启动STW、标记、清除、停止STW，优化为启动STW、标记、停止STW，清除操作与程序并行，而不被包含在STW过程中。

**三色标记法:**

第一步 , 每次新创建的对象，默认的颜色都是标记为“白色”， 第二步, 每次GC回收开始, 会从根节点开始遍历所有对象，把遍历到的对象从白色集合放入"灰色"集合。第三步, 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合。第四步, 重复第三步, 直到灰色中无任何对象。第五步: 回收所有的白色标记表的对象. 也就是回收垃圾。我们将全部的白色对象进行删除回收，剩下的就是全部依赖的黑色对象。

不加STW,会遇到对象丢失问题：
-   条件1: 一个白色对象被黑色对象引用(白色被挂在黑色下)
-   条件2: 灰色对象与它之间的可达关系的白色对象遭到破坏(灰色同时丢了该白色) 如果当以上两个条件同时满足时，就会出现对象丢失现象!

**屏障机制:**

```
强弱三色不变式：1.强三色不变式强制不允许黑色对象引用白色对象，目的在于破坏条件1。2.弱三色不变式黑色对象允许引用白色对象，白色对象存在其他灰色对象引用，或者可达它的链路上存在灰色对象，目的在于破坏条件2。
```

因此，在三色标级中满足强三色不变式或弱三色不变式之一，即可保证对象不丢失。

```
1.插入屏障 (为了保证栈的速度，不在栈上使用) 具体操作: 在A对象引用B对象的时候，B对象被标记为灰色。(将B挂在A下游，B必须被标记为灰色)满足: 强三色不变式. (不存在黑色对象引用白色对象的情况了， 因为白色会强制变成灰色)插入屏障不会在栈上操作，堆上处理没问题，但是如果栈不添加,当全部三色标记扫描之后,栈上有可能依然存在白色对象被引用的情况(黑色引用白色对象).  所以要对栈重新进行三色标记扫描, 但这次为了对象不丢失, 要对本次标记扫描启动STW暂停. 直到栈空间的三色标记结束.所以插入屏障，最后会对栈上的所有对象进行一次三色标记法+STW保护。
2. 删除屏障具体操作: 被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。满足: 弱三色不变式. (保护灰色对象到白色对象的路径不会断)这种方式的回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉。
```

**Go V1.8的混合写屏障(hybrid write barrier)机制**

插入写屏障和删除写屏障的短板：

-   插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；
-  删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。

Go V1.8版本引入了混合写屏障机制（hybrid write barrier），避免了对栈re-scan的过程，极大的减少了STW的时间。结合了两者的优点。1）混合写屏障规则 
1、GC开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)， 
2、GC期间，任何在栈上创建的新对象，均为黑色。
3、被删除的对象标记为灰色。
4、被添加的对象标记为灰色。满足: 变形的弱三色不变式. 这里我们注意， 屏障技术是不在栈上应用的，因为要保证栈的运行效率。

## 15.切片扩容

-   如果期望容量大于当前容量的两倍就会使用期望容量；
-   如果当前切片的长度小于 1024 就会将容量翻倍；
-   如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；


## 17.协程泄漏

-   缺少接收器，导致发送阻塞
-   缺少发送器，导致接收阻塞
-   死锁。多个协程由于竞争资源导致死锁。
-   WaitGroup Add()和Done()不相等，前者更大。

## 18.Go 可以限制运行时操作系统线程的数量吗？常见的goroutine操作函数有哪些？

runtime三大函数：runtime.Gosched()、runtime.Goexit()、runtime.GOMAXPROCS() 可以，
使用runtime.GOMAXPROCS(num int)可以设置线程数目。该值默认为CPU逻辑核数，如果设的太大，会引起频繁的线程切换，降低性能。
runtime.Gosched()，用于让出CPU时间片，让出当前goroutine的执行权限，调度器安排其它等待的任务运行，并在下次某个时候从该位置恢复执行。
runtime.Goexit()，调用此函数会立即使当前的goroutine的运行终止（终止协程），而其它的goroutine并不会受此影响。runtime.Goexit在终止当前goroutine前会先执行此goroutine的还未执行的defer语句。
请注意千万别在主函数调用runtime.Goexit，因为会引发panic。

## 19.defer底层原理

-   后调用的 defer 函数会先执行：
-   后调用的 defer 函数会被追加到 Goroutine \_defer 链表的最前面；
-   运行 runtime.\_defer 时是从前到后依次执行；defer nil -> panic defer运行时就会立刻将传递给延迟函数的参数保存起来 for loop defer need a func wrap channel
-   同步 Channel — 不需要缓冲区，发送方会直接将数据交给（Handoff）接收方；
-   异步 Channel — 基于环形缓存的传统生产者消费者模型；
-   chan struct{} 类型的异步 Channel — struct{} 类型不占用内存空间，不需要实现缓冲区和直接发送（Handoff）的语义；runtime.hchan包含缓冲区指针、元素个数、循环队列长度、发送操作处理到的位置、 接收操作处理到的位置、收发元素类型/大小、sendq 和 recvq 存储了当前 Channel 由于缓冲区空间不足而阻塞的 Goroutine 列 makechan
-   如果当前 Channel 中不存在缓冲区，那么就只会为 runtime.hchan 分配一段内存空间；
-   如果当前 Channel 中存储的类型不是指针类型，会为当前的 Channel 和底层的数组分配一块连续的内存空间；
-   在默认情况下会单独为 runtime.hchan 和缓冲区分配内存；发送数据的过程中包含几个会触发 Goroutine 调度的时机：
-   发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 runnext 属性，但是并不会立刻触发调度；
-   发送数据时并没有找到接收方并且缓冲区已经满了，这时会将自己加入 Channel 的 sendq 队列并调用 runtime.goparkunlock 触发 Goroutine 的调度让出处理器的使用权；从 Channel 接收数据时，会触发 Goroutine 调度的两个时机：
-   当 Channel 为空时；
-   当缓冲区中不存在数据并且也不存在数据的发送者时；

chan panic触发时机：
-   向已经关闭的channel写。
-   关闭已经关闭的channel。

chan 阻塞触发时机：

无缓存channel：
-   通道中无数据，但执行读通道。
-   通道中无数据，向通道写数据，但无协程读取。

有缓存channel：
-   通道的缓存无数据，但执行读通道。
-   通道的缓存已经占满，向通道写数据，但无协程读。

## 20.make和new

make 和 new 关键字的实现原理，make 关键字的作用是创建切片、哈希表和 Channel 等内置的数据结构，而 new 的作用是为类型申请一片内存空间，并返回指向这片内存的指针。

## 21.panic和recover

panic 函数可以被连续多次调用，它们之间通过 link 可以组成链表，内部包含：argp 是指向 defer 调用时参数的指针；arg 是调用 panic 时传入的参数；link 指向了更早调用的 runtime.\_panic 结构；recovered 表示当前 runtime.\_panic 是否被 recover 恢复；aborted 表示当前的 panic 是否被强行终止；

-   编译器会负责做转换关键字的工作；将 panic 和 recover 分别转换成 runtime.gopanic 和 runtime.gorecover；○ 将 defer 转换成 runtime.deferproc 函数；○ 在调用 defer 的函数末尾调用 runtime.deferreturn 函数；
    
-   在运行过程中遇到 runtime.gopanic 方法时，会从 Goroutine 的链表依次取出 runtime.\_defer 结构体并执行；
    
-   如果调用延迟执行函数时遇到了 runtime.gorecover 就会将 \_panic.recovered 标记成 true 并返回 panic 的参数；在这次调用结束之后，runtime.gopanic 会从 runtime.\_defer 结构体中取出程序计数器 pc 和栈指针 sp 并调用 runtime.recovery 函数进行恢复程序；○ runtime.recovery 会根据传入的 pc 和 sp 跳转回 runtime.deferproc；○ 编译器自动生成的代码会发现 runtime.deferproc 的返回值不为 0，这时会跳回 runtime.deferreturn 并恢复到正常的执行流程；
    
-   如果没有遇到 runtime.gorecover 就会依次遍历所有的 runtime.\_defer，并在最后调用 runtime.fatalpanic 中止程序、打印 panic 的参数并返回错误码 2；

## 22.map

-   开放地址法 线性探测，装载因子=元素数量/数组长度

index := hash("author") % array.len

-   拉链法 装载因子:=元素数量/桶数量 实现拉链法一般会使用数组加上链表，不过一些编程语言会在拉链法的哈希中引入红黑树以优化性能，拉链法会使用链表数组作为哈希底层的数据结构，我们可以将它看成可以扩展的二维数组 在一般情况下使用拉链法的哈希表装载因子都不会超过 1，当哈希表的装载因子较大时会触发哈希的扩容，创建更多的桶来存储哈希中的元素，保证性能不会出现严重的下降。如果有 1000 个桶的哈希表存储了 10000 个键值对，它的性能是保存 1000 个键值对的 1/10，但是仍然比在链表中直接读写好 1000 倍。
1.创建map
    
-   计算哈希占用的内存是否溢出或者超出能分配的最大值；
    
-   调用 runtime.fastrand 获取一个随机的哈希种子；
    
-   根据传入的 hint 计算出需要的最小需要的桶的数量；
    
-   使用 runtime.makeBucketArray 创建用于保存桶的数组；
2.扩容
    
-   装载因子已经超过 6.5；
    
-   哈希使用了太多溢出桶；根据触发的条件不同扩容的方式分成两种，如果这次扩容是溢出的桶太多导致的，那么这次扩容就是等量扩容 sameSizeGrow，sameSizeGrow 是一种特殊情况下发生的扩容，当我们持续向哈希中插入数据并将它们全部删除时，如果哈希表中的数据量没有超过阈值，就会不断积累溢出桶造成缓慢的内存泄漏

- 4。runtime: limit the number of map overflow buckets 引入了 sameSizeGrow 通过复用已有的哈希扩容机制解决该问题，一旦哈希中出现了过多的溢出桶，它会创建新桶保存数据，垃圾回收会清理老的溢出桶并释放内存    

哈希在存储元素过多时会触发扩容操作，每次都会将桶的数量翻倍，扩容过程不是原子的，而是通过 runtime.growWork 增量触发的，在扩容期间访问哈希表时会使用旧桶，向哈希表写入数据时会触发旧桶元素的分流。除了这种正常的扩容之外，为了解决大量写入、删除造成的内存泄漏问题，哈希引入了 sameSizeGrow 这一机制，在出现较多溢出桶时会整理哈希的内存减少空间的占用。

哈希表的每个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会存储到哈希的溢出桶中。随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。

## 23.context

1.Context接口

```
Deadline()
Done()
Err()
Value()
```

-   context.Background 是上下文的默认值，所有其他的上下文都应该从它衍生出来；
    
-   context.TODO 应该仅在不确定应该使用哪种上下文时使用；context.Background与context.TODO实现都一致,都是emptyCtx(int列名类型) 2.WithCancel从context.Context衍生出一个新的子上下文并返回用于取消该上下文的函数 WithCancel返回一个bool,CancelFunc。CancelFunc为匿名函数，内部调用cancelCtx的cancel方法。cancelCtx结构里面包含了：

```go
type cancelCtx struct { 
	Context // 传入的context 
	mu       sync.Mutex            // 保护接下来的三个字段 
	done     chan struct{}         // created lazily, closed by first cancel call 
	children map[canceler]struct{} // set to nil by the first cancel call 
	err      error                 // set to non-nil by the first cancel call
}
```

一旦我们执行返回的取消函数，当前上下文以及它的子上下文都会被取消，所有的 Goroutine 都会同步收到这一取消信号 3.WithValue WithValue从父上下文中创建一个子上下文，返回valueCtx

```go
type valueCtx struct { 
	Context key, 
	val interface{}
}
```

重写Value方法，从父上下文中获取 val 4.WithDeadline WithDeadline创建可以被取消的计时器上下文timerCtx

```go
type timerCtx struct { 
	cancelCtx timer *time.Timer // Under cancelCtx.mu. 
	deadline time.Time
}
```

context.WithDeadline 在创建 context.timerCtx 的过程中判断了父上下文的截止日期与当前日期，并通过 time.AfterFunc 创建定时器，当时间超过了截止日期后会调用 context.timerCtx.cancel 同步取消信号。context.timerCtx 内部不仅通过嵌入 context.cancelCtx 结构体继承了相关的变量和方法，还通过持有的定时器 timer 和截止时间 deadline 实现了定时取消的功能 5.WithDeadline WithTimeout会调用WithDeadline

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) { 
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

24.数组与切片的区别 数组是值类型，定长数组, StringHeader

```
type StringHeader struct { Data uintptr Len  int}
```

切片是引用类型，动态指向数组的指针，不定长指向数组array, SliceHeader

```
type SliceHeader struct { Data uintptr Len  int Cap  int}
```

## 25.接口

Go 语言根据接口类型是否包含一组方法将接口类型分成了两类：

-   使用 runtime.iface 结构体表示包含方法的接口
    

```
type iface struct { // 16 字节 tab  *itab data unsafe.Pointer}
```

-   使用 runtime.eface 结构体表示不包含任何方法的 interface{} 类型；
    

```
type eface struct { // 16 字节 _type *_type data  unsafe.Pointer}
```

接口类型interface{}

```
type emptyInterface struct { typ  *rtype word unsafe.Pointer}
```

## 26.reflect反射

-   reflect.TypeOf能获取类型信息 reflect.TypeOf 的实现原理其实并不复杂，它只是将一个 interface{} 变量转换成了内部的 reflect.emptyInterface 表示，然后从中获取相应的类型信息。
    
-   reflect.ValueOf能获取到数据的运行时信息 reflect.Type为接口,reflect.Value为结构体 私有成员，对外暴露公共方法。我们通过 reflect.TypeOf、reflect.ValueOf 可以将一个普通的变量转换成反射包中提供的 reflect.Type 和 reflect.Value，随后就可以使用反射包中的方法对它们进行复杂的操作。用于获取接口值 reflect.Value 的函数 reflect.ValueOf 实现也非常简单，在该函数中我们先调用了 reflect.escapes 保证当前值逃逸到堆上，然后通过 reflect.unpackEface 从接口中获取 reflect.Value 结构体。
    
原则1: 接口到任意对象，入参均是interface{},出参为任意对象。
原则2: 反射对象获取interface{}变量。
可以通过reflect.Value.Interface完成这项工作。
```
v := reflect.ValueOf(1)
v.Interface().(int)
```

上述两个原则转换：

-   从接口值到反射对象：○ 从基本类型到接口类型的类型转换；○ 从接口类型到反射对象的转换；
-   从反射对象到接口值：○ 反射对象转换成接口类型；○ 通过显式类型转换变成原始类型；
-   调用 reflect.ValueOf 获取变量指针；
-   调用 reflect.Value.Elem 获取指针指向的变量；
-   调用 reflect.Value.SetInt 更新变量的值：
```go
func main() { 
	i := 1 
	v := reflect.ValueOf(&i) 
	v.Elem().SetInt(10) 
	fmt.Println(i)
}
```

## 27.http

按照HTTP/1.1的规范，Go http包的http server和client的实现默认将所有连接视为长连接，无论这些连接上的初始请求是否带有Connection: keep-alive。1.http.Client 如果要关闭keep-alive, 需要通过http.Transport设置：

```go
tr := &http.Transport{  DisableKeepAlives: true, } 
c := &http.Client{  Transport: tr, }

// 发送请求 
rsp, err:= c.Do(req) 
defer rsp.Body.Close() // 读取数据 
ioutil.ReadAll(rsp.Body)
```

2.http.Server server端完全不支持keep-alive连接方式：

```go
s := http.Server{  Addr: ":8080",  Handler: xxx, } 
s.SetKeepAliveEnabled(false) 
s.ListenAndServe()
```

闲置连接超时控制：

```go
s := http.Server{     Addr: ":8080",     Handler: xxx,  IdleTimeout: 5 * time.Second, } 
s.ListenAndServe()
```

客户端和服务端响应的次数

-   长连接：可以多次。
-   短链接：一次。传输数据的方式
-   长连接：连接--数据传输--保持连接
-   短连接：连接--数据传输--关闭连接 长连接和短链接的优缺点
-   长连接 优点 省去较多的TCP建立和关闭的操作，从而节约时间。■ 性能比较好。（因为客户端一直和服务端保持联系） ○ 缺点 当客户端越来越多的时候，会将服务器压垮。■ 连接管理难。■ 安全性差。（因为会一直保持着连接，可能会有些无良的客户端，随意发送数据等）
-   短链接 优点 服务管理简单。存在的连接都是有效连接 ○ 缺点 请求频繁，在TCP的建立和关闭操作上浪费时间

## 28.主协程如何优雅等待子协程

-   channel进行同步
-   sync.WaitGroup同步

## 29.Go中map如何顺序读取

由于map底层实现与 slice不同, map底层使用hash表实现,插入数据位置是随机的, 所以遍历过程中新插入的数据不能保证遍历。主要是对 key 排序，那么我们便可将 map 的 key 全部拿出来，放到一个数组中，然后对这个数组排序后对有序数组遍历，再间接取 map 里的值就行了。

## 30.变量地址

```go
package main
const cl  = 100
var bl    = 123
func main()  {    
	println(&bl,bl)    
	println(&cl,cl) // 不可取cl地址
}
```

常量 常量不同于变量的在运行期分配内存，常量通常会被编译器在预处理阶段直接展开，作为指令数据使用，

## 31.goto

goto不可以跳转到其他函数或内层代码

```
package mainfunc main()  {    
	for i:=0;i<10 ;i++  {    
		loop:        
			println(i)    
	}    
	goto loop
}
```
---
## 项目难点 

  > 老生常谈的项目亮点和遇到的难点是什么，这个需要说到什么粒度？比如我优化了一个本地缓存，本来是全局一个前缀树，解析字符串，根据 xxxx 的业务场景，改成了二级 map ，减小了锁的粒度。这么说是不是过于简单了？  
  
是的简单了，我的思路是：  
1. 实际过程中，在线上业务或者 perf 测试时候发现了问题，如何发现的？这里涉及到了可观测性工具，以及 perf 工具，以及 debug 思路  
2. 找到了问题，那么就要考虑原有的结构， 如何调优，如何测试，有没有更好地思路，是否去做了探查，业界有哪些方法来解决这个问题（方法论问题）  
3. 解决完成后，看收益，要具体到数字指标  
4. 如何持续性优化思路

1. 项目亮点和遇到的难点  
项目亮点主要是说你项目的特殊性 跟普通的 curd 有哪些不同 你可以说业务上有哪些特殊需求 然后技术方案上怎么处理的，难点部分主要针对你自己解决的问题，需求开发或项目运营周期内有哪些棘手比较难解决的问题例如 bug 、接口告警、性能问题等等，最后通过什么方案（不一定是技术方案）解决了，xx官想通过你解决问题的方式了解你的能力水平。  
  
很多人写在简历上的项目就是白开水，就算真的是你也可以有一些自己的思考，可以是你自己没有真的实现但技术能力范围可以实现的，可以介绍一下给他听，比如说你的想法需要协调很多资源最终没能实现，但你自己做过了什么调研可以确认如果按照这个方案是可以解决某些问题的，这也相当于解决了。  
  
2.系统设计相关的题目，当你给出一个可以实际解决问题的回答时如果xx官追问有没有别的方案，你可以说以往的工作中是这样处理的，是不是有哪里不太完善希望xx官给一些提示。  
  
这里有一部分问题是他自己经历过的问题最终解决了，可能你说的没有他想听到的答案或者细节，最好是反问下他到底想听什么，大部分的人都会给一些提示引导一下方向毕竟太开放了也没办法回答，实在答不出也可以邀请对方聊一下，从对方后续的回答中也可以理解下对方想问的方向如果你能继续聊两句 你也可以顺着他的思路再补充一下说刚才没完全理解题目之类的，尽量不要在这种题目把话题聊死说不会答不出等等 尽量沟通上平滑过度 不然很容易快速结束xx。。。 这种题展示的是你的思路和思考的过程，简答题知道吧。。。别空着瞎聊也聊几句  

## 我做过的性能优化
es优化写入

### 本地缓存优化
同一个查询条件，可以将入参什么的作为key，返回值作为value放到redis缓存，如果没有的话，就查询redis和mysql的原始数据然后将回参放到本地缓存中，并加上一个极短的过期时间，目前设置是五分钟，没错就是五分钟，然后设置的过程需要加redis nx分布式锁，用来保证不会重复插入缓存

当然我们完全可以把这个动作放到本地缓存中，可以以此来作为一个渐进式的缓存填充动作，保证准确性和速度的一个平衡，毕竟少点缓存，就少点数据不一致的风险，并且每个缓存可以不在同一个时间失效，而且就算出现了数据不一致的情况，也不会说完全等到一个完整的过期时间窗口才开始修正，毕竟每个pod的缓存时间不是同时开始的，总有会个pod比其他pod更早设置缓存和更早失去缓存。

上面这种情况是可以用于布局的数据，查询商品的数据这种情况。

暂时没有做本地缓存强制失效的情况，因为我们的运营商修改数据之后，我们都是建议会在几分钟内同步，让运营商有反悔的机会

理由是，之前用的是团队比较早期封装的cache(就是下面这个)，我稍微按照freeCache改了一下，让其他人不需要改动协议
并且对freeCache进行了一些特定的优化。
优化方案：freecache 因为其高效的性能获得很多人的喜爱，我也不例外。却因为里面使用了go Mutex导致了在非常高的并发情况下性能无法进一步提高，主要原因就是使用 get 方法中的Mutex.Lock()。 通过跟原作者的沟通后，发现因为 freecache get方法并不是只读方法，所以才动了念头修改一版能够满足自己高并发http平台的cache，使用的是rwmutex

之前的freecache使用mutex来读数据，是因为需要修改accessTime，这个accessTime是用来计算lru淘汰规则的，所以作者就不用读写锁，而且另外一个方面，在并发程度比较低的情况下，rwmutex的性能可能比mutex差，因为rwmutex比mutex做了更多的工作。

但是我公司的场景都是用于比较频繁请求的情况才会去弄本地缓存，而且还有自己的热点检测策略，所以进行了rwmutex的修改，并且将lru规则从
```go
leastRecentUsed := int64(oldHdr.accessTime)*atomic.LoadInt64(&seg.totalCount) <= atomic.LoadInt64(&seg.totalTime)
```
改成
```go
randomKeyDelete := !isExistHot && (oldHdr.expireAt & 3) == 3
```
不存在于热点数据且命中四分之一的话就去干掉

热点数据的话就是使用heavyKeeper+最小堆来实现的，heavyKeeper使用二维数组来实现，通过key和index获取murmur3的hash来为每一维的数组的对应cow添加指纹和count，如果cunt=0就会直接加入，如果count不为0，就要根据count来获取衰减概率来进行衰减，然后获取最小堆中的最小count来判断当前元素是否可以加入最小堆。本地缓存的lru就要考虑hot key了，所以随机删除只要不排除hot key就行。

热点检测是一个单独的协程从chan中线性处理的，而且freeCache和热点检测是采用了同一个minHeap，所以可以看到同一个值。


(真实)binlog去除refresh=true就会每秒新增六个es的写入。

### binlog处理速度要快，而且不能引起rebalance，而且还需要保证事务的完整性，还需要尽量使用批处理？ 
大数据量binlog的处理，内存调优，sync.pool,json序列化

之所以需要用redis来做队列，是因为分片太多了导致事务分散了，所以需要将同一个商品或者sku的事务聚合起来

放入redis的消息只需要变更的记录和变更类型crud，不需要有那么多数据,算是一个预处理，将数据量减少
es和redis项目还不能放到一起，因为不能互相影响！！！

放入redis的数据是需要做一下过滤的，去掉那些不需要关心的数据

至于key的数量，应该是提前固定好key的数量来简化问题的复杂度，初步定位十个，这个数量应该来说是比较大的，然后每个pod的话要根据配置文件来确定自己需要的key的上限，key的列表和使用上限都写在配置文件中，然后pod会定时去抢锁，当然如果抢到了上限的key，就不需要抢锁了，如果没抢到，每隔几秒就需要去判断下哪些key可以抢来抢下来。
为了预防有些key一直没有消费，或者消费太慢导致大key，发送者应该发送几次后观察下key的内容是否太大了，如果太大就需要报警，并停止发送。

redis中通过zset中的es执行时间作为分数，并且为了避免大key问题，所以需要为商品id和skuiD分别创建多个zset，然后add zset的时候需要取hash，然后每个pod定时去拉取自己的key，至于怎么取到自己的key，就是通过分布式锁来设置redis 过期key,然后不断续租就行，注意临界条件是刚好要拉取数据的时候就是一个事务的第一条binlog过来，所以不要取最近几秒的日志，只取几秒之前的binlog，然后再将binlog通过offset排序然后再去重。

那么怎么保证消费者成功消费消息呢？那么就是消费成功之后再去删除zset中的数据！

防止消费太慢，因为poll超过某个时间没有执行，就会rebalance，应该怎么做？
按道理来说，kafka消费是分片的，然后直接塞入redis的，所以速度应该是没什么问题，塞入的方式使用pipeline
主要是防止redis出现大key问题：
应该是需要按照商品id或者skuid来各自分几个key,商品id或者skuid通过hash来获取到可以使用的key，然后pod就根据key的数量来抢各自的key，每个pod只会抢一个key，然后每个key的商品id或者skuid都是唯一的，需要集合去重。

如果是redis的话，速度很快
如果是es的话，需要bulk，并且进行写入优化，而且要确定需要多少es分片，对es的写入的话就需要做好文档的字段管理。
然后如果发现从redis中获取的数据，时间超过某个限度的话，说明消费太慢了，就进行消息丢弃然后来补偿

没有优化前，business-data-clean最低是每分钟处理2762个binlog，平均大概2286个binlog，每秒最低大概处理46个binlog
business-product-sync最低是每分钟处理1377个binlog，每秒最低大概处理23个binlog
gc停顿的话大概是clean是十秒停一次，sync是八秒一次

使用了一次拉取多个消息然后逐个提交，使用sync.pool和去除json，还有字符串拼接使用stringbuilder
business-data-clean后平均处理4886，每秒钟82个binlog，分别是4104，4339，6215

business-product-sync去除refresh后，加大translog和refresh间隔后，每分钟处理2306，每秒处理39个binlog。但是gc尖刺严重，需要sync.pool

cpu的话
通过profile判断cpu在哪里，然后就看到一个for循环没有做好退出条件。这个问题不一定复现，因为这个分支不是每次都进入的，所以要根据日志判断那些表的变动会出现这个问题，或者那些行那些字段会出现这个问题，然后根据这个问题来重试来后去pprof和火焰图。

内存泄漏：
内存泄漏的时候，在一个不会退出的函数中的for循环使用了defer，导致defer函数一直无法使用。
使用kubectl proxy去映射端口，然后通过多个heap文件来判断增长的代码在哪里，然后list那个地方，就看到defer了

## redis 高性能原理，主从同步原理，项目中使用的场景  
redis的数据结构非常精简，会为每个数据结构的不同数据量考虑不同的数据结构，比如sds，还有以前的list的压缩列表和链表，现在的list的快速列表

主从同步的gossip，每秒发好几次请求给其他节点，每次请求都会发一下自己的信息和部分其他节点的信息，如果有些节点很久没有通信，就会立即发送出去

使用场景的话会有 分布式锁，缓存，简单的延迟队列

## 系统设计，小红书 ugc 的评论列表，如何设计以保持其查询的高性能。可以大量使用  
内存空间，支持按时间戳、热度排序，评论数量 10w+  

　　评论记录表 - likes : 每一次的点赞记录（用户Mid、被评论的实体ID（messageID）、点赞来源、评论内容、时间）等信息，并且在Mid、messageID两个维度上建立了满足业务求的联合索引。

　　评论数表 - counts : 以业务ID（BusinessID）+实体ID(messageID)为主键，聚合了该实体的评论数等信息。并且按照messageID维度建立满足业务查询的索引。

redis存储的内容：
评论数缓存： 
key:           xxxx前缀：{businessId}:{messageId}
value:       {nums}

key = user:likes:patten:{business_id} :{messageid}
value =  点赞来源 -评论内容       -score(likeTimestamp)* 
用mid与业务ID作为key，value则是一个ZSet,member为被点赞的实体ID，score为点赞的时间。`当改业务下某用户有新的点赞操作的时候，被点赞的实体则会通过 zadd的方式把最新的点赞记录加入到该ZSet里面来为了维持用户点赞列表的长度（不至于无限扩张），需要在每一次加入新的点赞记录的时候，按照固定长度裁剪用户的点赞记录缓存。该设计也就代表用户的点赞记录在缓存中是有限制长度的，超过该长度的数据请求需要回源DB查询`

　　③、第三层存储

　　LocalCache - 本地缓存

　　本地缓存的建立，目的是为了应对缓存热点问题。

　　利用最小堆算法，在可配置的时间窗口范围内，统计出访问最频繁的缓存Key,并将热Key（Value）按照业务可接受的TTL存储在本地内存中。

点赞的话可以通过binlog来做，如果介意速度的话就同步，但是一般不会太慢的

## 9.操作系统的用户态和核心态切换条件以及为什么要切换  
为什么要区分用户态和内核
如此设计的本质意义是进行权限保护。 限定用户的程序不能乱搞操作系统，如果人人都可以任意读写任意地址空间软件管理便会乱套.

我们总说用户态和内核态切换的开销大，那么切换的开销具体大在哪里呢？

具体来说有以下几点

- 保留用户态现场（上下文、寄存器、用户栈等）
- 复制用户态参数，用户栈切到内核栈，进入内核态
- 额外的检查（因为内核代码对用户不信任）
- 执行内核态代码
- 复制内核态代码执行结果，回到用户态
- 恢复用户态现场（上下文、寄存器、用户栈等）

用户态到内核态切换的场景

- 系统调用：用户态进程主动切换到内核态的方式，用户态进程通过系统调用向操作系统申请资源完成工作，例如 fork（）就是一个创建新进程的系统调用，系统调用的机制核心使用了操作系统为用户特别开放的一个中断来实现，如Linux 的 int 80h 中断，也可以称为软中断
    
- 异常：当 C P U 在执行用户态的进程时，发生了一些没有预知的异常，这时当前运行进程会切换到处理此异常的内核相关进程中，也就是切换到了内核态，如缺页异常
    
- 中断：当 C P U 在执行用户态的进程时，外围设备完成用户请求的操作后，会向 C P U 发出相应的中断信号，这时 C P U 会暂停执行下一条即将要执行的指令，转到与中断信号对应的处理程序去执行，也就是切换到了内核态。如硬盘读写操作完成，系统会切换到硬盘读写的中断处理程序中执行后边的操作等。


## 10.线程间的通信方式
![[Pasted image 20230529144551.png]]

## MySQL 的主从复制怎么做
- **MySQL 主从复制原理**

MySQL主从复制涉及到三个线程，一个运行在主节点（log dump thread），其余两个(I/O thread, SQL thread)运行在从节点，如下图所示:  

![](https://pic1.zhimg.com/80/v2-1b0c3f31bd398c39b9e0930059b0ca24_720w.webp)

  
**l 主节点 binary log dump 线程**  
当从节点连接主节点时，主节点会创建一个log dump 线程，用于发送bin-log的内容。在读取bin-log中的操作时，此线程会对主节点上的bin-log加锁，当读取完成，甚至在发动给从节点之前，锁会被释放。

**l 从节点I/O线程**  
当从节点上执行`start slave`命令之后，从节点会创建一个I/O线程用来连接主节点，请求主库中更新的bin-log。I/O线程接收到主节点binlog dump 进程发来的更新之后，保存在本地relay-log中。

**l 从节点SQL线程**  
SQL线程负责读取relay log中的内容，解析成具体的操作并执行，最终保证主从数据的一致性。  

对于每一个主从连接，都需要三个进程来完成。当主节点有多个从节点时，主节点会为每一个当前连接的从节点建一个binary log dump 进程，而每个从节点都有自己的I/O进程，SQL进程。从节点用两个线程将从主库拉取更新和执行分成独立的任务，这样在执行同步数据任务的时候，不会降低读操作的性能。比如，如果从节点没有运行，此时I/O进程可以很快从主节点获取更新，尽管SQL进程还没有执行。如果在SQL进程执行之前从节点服务停止，至少I/O进程已经从主节点拉取到了最新的变更并且保存在本地relay日志中，当服务再次起来之后，就可以完成数据的同步。  

要实施复制，首先必须打开Master 端的binary log（bin-log）功能，否则无法实现。  
因为整个复制过程实际上就是Slave 从Master 端获取该日志然后再在自己身上完全顺序的执行日志中所记录的各种操作。如下图所示：

![](https://pic4.zhimg.com/80/v2-17a1d089c3266a59b5d00d7bd055bed7_720w.webp)

复制的基本过程如下：

- 从节点上的I/O 进程连接主节点，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容；
- 主节点接收到来自从节点的I/O请求后，通过负责复制的I/O进程根据请求信息读取指定日志指定位置之后的日志信息，返回给从节点。返回信息中除了日志所包含的信息之外，还包括本次返回的信息的bin-log file 的以及bin-log position；从节点的I/O进程接收到内容后，将接收到的日志内容更新到本机的relay log中，并将读取到的binary log文件名和位置保存到master-info 文件中，以便在下一次读取的时候能够清楚的告诉Master“我需要从某个bin-log 的哪个位置开始往后的日志内容，请发给我”；
- Slave 的 SQL线程检测到relay-log 中新增加了内容后，会将relay-log的内容解析成在祝节点上实际执行过的操作，并在本数据库中执行。

- **MySQL 主从复制模式**

MySQL 主从复制默认是异步的模式。MySQL增删改操作会全部记录在binary log中，当slave节点连接master时，会主动从master处获取最新的bin log文件。并把bin log中的sql relay。  
**l 异步模式（mysql async-mode）**  
异步模式如下图所示，这种模式下，主节点不会主动push bin log到从节点，这样有可能导致failover的情况下，也许从节点没有即时地将最新的bin log同步到本地。

![](https://pic1.zhimg.com/80/v2-c15bfffe3e398eafc7e0ffdaeebfcaac_720w.webp)

  
**l 半同步模式(mysql semi-sync)**  
这种模式下主节点只需要接收到其中一台从节点的返回信息，就会commit；否则需要等待直到超时时间然后切换成异步模式再提交；这样做的目的可以使主从数据库的数据延迟缩小，可以提高数据安全性，确保了事务提交后，binlog至少传输到了一个从节点上，不能保证从节点将此事务更新到db中。性能上会有一定的降低，响应时间会变长。如下图所示：

![](https://pic2.zhimg.com/80/v2-d9ac9c5493d1d772f5bf57ede089f0d5_720w.webp)

  
半同步模式不是mysql内置的，从mysql 5.5开始集成，需要master 和slave 安装插件开启半同步模式。

  
**l 全同步模式**  
全同步模式是指主节点和从节点全部执行了commit并确认才会向客户端返回成功。

- **binlog记录格式**

MySQL 主从复制有三种方式：基于SQL语句的复制（statement-based replication，SBR），基于行的复制（row-based replication，RBR)，混合模式复制（mixed-based replication,MBR)。对应的binlog文件的格式也有三种：STATEMENT,ROW,MIXED。  

l Statement-base Replication (SBR)就是记录sql语句在bin log中，Mysql 5.1.4 及之前的版本都是使用的这种复制格式。优点是只需要记录会修改数据的sql语句到binlog中，减少了binlog日质量，节约I/O，提高性能。缺点是在某些情况下，会导致主从节点中数据不一致（比如sleep(),now()等）。

  
l Row-based Relication(RBR)是mysql master将SQL语句分解为基于Row更改的语句并记录在bin log中，也就是只记录哪条数据被修改了，修改成什么样。优点是不会出现某些特定情况下的存储过程、或者函数、或者trigger的调用或者触发无法被正确复制的问题。缺点是会产生大量的日志，尤其是修改table的时候会让日志暴增,同时增加bin log同步时间。也不能通过bin log解析获取执行过的sql语句，只能看到发生的data变更。

  
l Mixed-format Replication(MBR)，MySQL NDB cluster 7.3 和7.4 使用的MBR。是以上两种模式的混合，对于一般的复制使用STATEMENT模式保存到binlog，对于STATEMENT模式无法复制的操作则使用ROW模式来保存，MySQL会根据执行的SQL语句选择日志保存方式。

- **GTID复制模式**

@ 在传统的复制里面，当发生故障，需要主从切换，需要找到binlog和pos点，然后将主节点指向新的主节点，相对来说比较麻烦，也容易出错。在MySQL 5.6里面，不用再找binlog和pos点，我们只需要知道主节点的ip，端口，以及账号密码就行，因为复制是自动的，MySQL会通过内部机制GTID自动找点同步。  
@ 多线程复制（基于库），在MySQL 5.6以前的版本，slave的复制是单线程的。一个事件一个事件的读取应用。而master是并发写入的，所以延时是避免不了的。唯一有效的方法是把多个库放在多台slave，这样又有点浪费服务器。在MySQL 5.6里面，我们可以把多个表放在多个库，这样就可以使用多线程复制。  

- **基于GTID复制实现的工作原理**
- 主节点更新数据时，会在事务前产生GTID，一起记录到binlog日志中。
- 从节点的I/O线程将变更的bin log，写入到本地的relay log中。
- SQL线程从relay log中获取GTID，然后对比本地binlog是否有记录（所以MySQL从节点必须要开启binary log）。
- 如果有记录，说明该GTID的事务已经执行，从节点会忽略。
- 如果没有记录，从节点就会从relay log中执行该GTID的事务，并记录到bin log。
- 在解析过程中会判断是否有主键，如果没有就用二级索引，如果有就用全部扫描。

## 2.MySQL 的索引，使用 B+树索引的好处  
可以遍历呀，数据层级小，查询方便
在实际场景应用当中，MySQL表数据，一般情况下都是比较庞大、海量的。如果使用红黑树，树的高度会特别高，红黑树虽说查询效率很高。但是在海量数据的情况下，树的高度并不可控。如果我们要查询的数据，正好在树的叶子节点。那查询会非常慢。故而MySQL并没有采用红黑树来组织索引。

mysql进行过磁盘读取时，是以页为单位进行读取，每个节点表示一页。而平衡二叉树每个节点存储一个关键词，导致存储空间被浪费。

内存中为什么要使用btree，不用红黑树？
首先btree有现成的框架，而且比较简单，红黑树比较复杂。其次在内存中，性能差不多，插入非常频繁，数据量又大，确实红黑树比较好。而且两者时间复杂度差不多。

## 3.MySQL 性能查看以及如何优化
![[Pasted image 20230529145542.png]]
服务端配置
1. 增加可用连接数，修改环境变量`max_connections`，默认情况下服务端的最大连接数为`151`个
2. 及时释放不活动的连接，系统默认的客户端超时时间是28800秒（8小时），我们可以把这个值调小一点

客户端尽量用连接池
  
## 5.Redis 的持久化操作
aof和rdb
aof是可以选择每秒一次还是每次操作都写磁盘的
rdb是可以选择多大的时间间隔内操作多少数据才生成快照的
最新版本是混合持久化，就是aof的开头就是rdb文件

## 6.如何利用 redis 处理热点数据  
面对热点数据，如果缓存过期了的话，要通过分布式锁去更新缓存，不能所有的请求都到db去。而且非必要，不需要去设置过期时间。、
还可以通过读写分离的方案，比如：上面我们提到从redis只不负责读和写请求的，**但redis官方提供了一个方法，在操作读请求时，可以先加上readonly命令**，这样从redis就可以提供读请求服务了，不需要转发到主redis。

> 根据这个特性，我们可以对**客户端工具进行改造**，读请求方法时，加上**readonly这个命令**，从而实现读写分离，提高了从redis的利用率。

即达到了多台从redis去扛大量请求了，减少了主redis压力。**这个方案需要对客户端进行改造，而且redis官方推荐没有必要使用读写分离。**

还有就是可以做好本地缓存。对于热点发现和加强热点数据的命中，可以使用redis的lfu策略以及hotkeys查询

## 7.TCP 三次握手的过程，如果没有第三次握手有什么问题。  
服务端不会认为链接已经建立，会一直发ack请求直到超时

TCP 建立连接时通过三次握手可以有效地避免历史错误连接的建立，减少通信双方不必要的资源消耗，三次握手能够帮助通信双方获取初始化序列号，它们能够保证数据包传输的不重不丢，还能保证它们的传输顺序，不会因为网络传输的问题发生混乱，到这里不使用『两次握手』和『四次握手』的原因已经非常清楚了：

- 『两次握手』：无法避免历史错误连接的初始化，浪费接收方的资源；
- 『四次握手』：TCP 协议的设计可以让我们同时传递 `ACK` 和 `SYN` 两个控制信息，减少了通信次数，所以不需要使用更多的通信次数传输相同的信息；

## 1.cap 和base了解么，分别指什么  
CAP原则又称CAP定理，指的是在一个分布式系统中Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性）`这三个基本需求，最多只能同时满足其中的2个。`

|选项|描述|
|---|---|
|Consistency（一致性）|指数据在多个副本之间能够保持一致的特性（严格的一致性）|
|Availability（可用性）|指系统提供的服务必须一直处于可用的状态，每次请求都能获取到非错的响应（不保证获取的数据为最新数据）|
|Partition tolerance（分区容错性）|分布式系统在遇到任何网络分区故障的时候，仍然能够对外提供满足一致性和可用性的服务，除非整个网络环境都发生了故障|

BASE理论是Basically Available(基本可用)，Soft State（软状态）和Eventually Consistent（最终一致性）三个短语的缩写。

BASE是对CAP中一致性和可用性权衡的结果，是基于CAP定律逐步演化而来。其核心思想是即使无法做到强一致性，但每个应用都可以根据自身业务特点，才用适当的方式来使系统达到最终一致性。
  
软状态是允许系统中的数据存在中间状态，并认为该状态不影响系统的整体可用性，即允许系统在多个不同节点的数据副本存在数据延时。

**举例**
比如redis主从节点数据复制。在某一瞬间，主从节点数据同步，存在数据不一致的情况。

## 3.Redis 是单线程还是多线程？ Redis 的分布式集群怎么做?  
redis6.0之前使用单线程的优点：
- Redis是基于内存的操作，磁盘IO以及CPU不是Redis的瓶颈， Redis的瓶颈最有可能是机器内存的大小和网络带宽。
- 采用单线程，避免了不必要的上下文切换和竞争条件， 也不存在多进程或者多线程导致的切换而消耗 CPU。
- 不用去考虑各种锁的问题，不存在加锁释放锁操作， 没有因为可能出现死锁而导致的性能消耗

多线程引入的原因：
多线程是 Redis6.0 推出的一个新特性。Redis 是核心线程负责网络 IO ， 命令处理以及写数据到缓冲， 而随着网络硬件的性能提升， 单个主线程处理⽹络请求的速度跟不上底层⽹络硬件的速度， 导致网络 IO 的处理成为了 Redis 的性能瓶颈。

而 Redis6.0 就是从单线程处理网络请求到多线程处理， 通过多个 IO 线程并⾏处理网络操作提升实例的整体处理性能。 需要注意的是对于读写命令，Redis 仍然使⽤单线程来处理， 这是因为继续使⽤单线程执行命令操作，就不⽤为了保证 Lua 脚本、事务的原⼦性， 额外开发多线程互斥机制了。

需要注意的是在 Redis6.0 中，多线程机制默认是关闭的，需要在 redis.conf 中 完成以下两个设置才能启用多线程。

设置 io-thread-do-reads 配置项为 yes，表示启用多线程。 `io-threads-do-reads yes` 设置线程个数。⼀般来说，线程个数要小于 Redis 实例所在机器的 CPU 核数， 例如，对于⼀个 8 核的机器来说，Redis 官⽅建议配置 6 个 IO 线程。 `io-threads 6`

![[Pasted image 20230529155721.png]]
全部流程主要分为 4 个阶段：

**阶段一：服务端和客⼾端建立 Socket 连接，并分配处理线程**

当有客⼾端请求和实例建立 Socket 连接时，主线程会创建和客户端的连接， 并把 Socket 放入全局等待队列中。然后主线程通过轮询方法把 Socket 连接 分配给 IO 线程。

**阶段二：IO 线程读取并解析请求**

主线程把 Socket 分配给 IO 线程后，会进⼊阻塞状态等待 IO 线程 完成客户端请求读取和解析。

**阶段三：主线程执⾏请求操作**

IO 线程解析完请求后，主线程以单线程的⽅式执⾏这些命令操作。

**阶段四：IO 线程回写 Socket 和主线程清空全局队**

主线程执行完请求操作后，会把需要返回的结果写入缓冲区。 然后，主线程会阻塞等待 IO 线程把这些结果回写到 Socket 中， 并返回给客户端。等到 IO 线程回写 Socket 完毕，主线程会清空全局队列， 等待客户端的后续请求。

  Redis 之所以如此设计它的多线程网络模型，我认为主要的原因是为了保持兼容性，因为以前 Redis 是单线程的，所有的客户端命令都是在单线程的事件循环里执行的，也因此 Redis 里所有的数据结构都是非线程安全的，现在引入多线程，如果按照标准的 Multi-Reactors/Master-Workers 模式来实现，则所有内置的数据结构都必须重构成线程安全的，这个工作量无疑是巨大且麻烦的。

  
## 5.负载均衡怎么做的呢，为什么这么做，了解过集群雪崩么。 
使用k8s的svc的轮询机制，并且会根据cpu来做hpa，还会通过konga来限制ip

一些节点不可用的时候，会加大其他节点的负载，导致所有的负载起来一个死一个，要先做好限流和缓存数据，至于熔断的话基于业务的复杂性，我们还没有考虑。

## 6.谈谈高并发场景下削峰，限流的实现？
主要通过nsq和kafka来发送消息让下游慢慢消费，限流的话我们主要是只在关键节点做单机限流，并且会做k8s的hpa扩容，konga的ip限流

限流的话我们是通过redis来做的，比如使用zset保存每一次的请求，分数使用时间戳，然后判断下zset中当前的数量是否等于限流值就行，因为用的是滑动窗口的方式，所以我们会将窗口外的数据删除，并且这个zset key也是会超时删除的，避免不活跃的数据一直占用内存

用redis做分布式限流的缺点可能就是计算不准确，同一个毫秒内都有可能多个请求一起来，但是因为是同一时间，所以都算成一个请求了
常见的使用redis实现。引入了中心化存储，就需要解决以下问题：

- 数据一致性  
    在限流模式中理想的模式为时间点一致性。时间点一致性的定义中要求所有数据组件的数据在任意时刻都是完全一致的，但是一般来说信息传播的速度最大是光速，其实并不能达到任意时刻一致，总有一定的时间不一致，对于我们CAP中的一致性来说只要达到读取到最新数据即可，达到这种情况并不需要严格的任意时间一致。这只能是理论当中的一致性模型，可以在限流中达到线性一致性即可。
- 时间一致性  
    这里的时间一致性与上述的时间点一致性不一样，这里就是指各个服务节点的时间一致性。一个集群有3台机器，但是在某一个A/B机器的时间为`Tue Dec 3 16:29:28 CST 2019`，C为`Tue Dec 3 16:29:28 CST 2019`，那么它们的时间就不一致。那么使用ntpdate进行同步也会存在一定的误差，对于时间窗口敏感的算法就是误差点。
- 超时  
    在分布式系统中就需要网络进行通信，会存在网络抖动问题，或者分布式限流中间件压力过大导致响应变慢，甚至是超时时间阈值设置不合理，导致应用服务节点超时了，此时是放行流量还是拒绝流量？
- 性能与可靠性  
    分布式限流中间件的资源总是有限的，甚至可能是单点的（写入单点），性能存在上限。如果分布式限流中间件不可用时候如何退化为单机限流模式也是一个很好的降级方案。

所以针对这些问题，lua脚本虽然获取不到系统资源，但是可以通过time这个关键字来获取时间，让每一个请求的时间保持一致，然后为了优化，还可以让每一次的请求多拿一些资源回来，比如每一次就拿十个，然后放在本地让人获取，这样就不会让redis成为热点key，也不会频繁交互浪费时间，拿不到目标值就拿最大值，在本地获取的话就是使用 CompareAndSwapT 并且获取资源的时候要通过chan去拉取，拉取前先判断限制是否有资源，没有再去拉取

## es写入速度优化
综合来说,提升写入速度从以下几方面入手:

-   加大 translog flush ,目的是降低 iops,writeblock  不用每次请求都flush，可以设置为每隔多少秒或者多少mb来flush
-   加大 index refresh间隔, 目的除了降低 io, 更重要的降低了 segment merge 频率，默认情况下索引的refresh_interval为1秒,这意味着数据写1秒后就可以被搜索到,每次索引的 refresh 会产生一个新的 lucene 段,这会导致频繁的 segment merge 行为,如果你不需要这么高的搜索实时性,应该降低索引refresh 周期,还可以在每次发送请求的时候，使用默认的行为，也就是？refresh=false，如果是true的话就是立即刷新，如果是wait_for的话就是查询要等待刷新

-   调整 bulk 线程池和队列
-   优化磁盘间的任务均匀情况,将 shard 尽量均匀分布到物理主机的各磁盘
-   优化节点间的任务分布,将任务尽量均匀的发到各节点，当`client.transport.sniff`为 true,(默认为 false),列表为所有数据节点 。否则,列表为初始化客户端对象时添加进去的节点.但是使用阿里云服务的话，就是只有一个url而已。

-   优化 lucene 层建立索引的过程,目的是降低 CPU 占用率及 IO
1、调整字段为不分词，不索引
2、对不需要评分的字段使用norms禁用

## es搜索速度优化
优化过程里我突然想起了官网对于`Filter or Query`的优化意见 (详见延伸阅读链接 1)，文中说:

> 过滤查询（Filtering queries）只是简单的检查包含或者排除，这就使得计算起来非常快。考虑到至少有一个过滤查询（filtering query）的结果是 “稀少的”（很少匹配的文档），并且经常使用不评分查询（non-scoring queries），结果会被缓存到内存中以便快速读取，所以有各种各样的手段来优化查询结果。
> 
> 相反，评分查询（scoring queries）不仅仅要找出 匹配的文档，还要计算每个匹配文档的相关性，计算相关性使得它们比不评分查询费力的多。同时，查询结果并不缓存。
> 
> 多亏倒排索引（inverted index），一个简单的评分查询在匹配少量文档时可能与一个涵盖百万文档的 filter 表现的一样好，甚至会更好。但是在一般情况下，一个 filter 会比一个评分的 query 性能更优异，并且每次都表现的很稳定。
> 
> 过滤（filtering）的目标是减少那些需要通过评分查询（scoring queries）进行检查的文档。

---

「在每一层 terms aggregation 内部加一个 {"execution_hint":"map"}」。在评论区 @kennywu76 也给了详细的解释，我认为算是全网最好的解释了，转发一下:

> Terms aggregation 默认的计算方式并非直观感觉上的先查询，然后在查询结果上直接做聚合。
> 
> ES 假定用户需要聚合的数据集是海量的，如果将查询结果全部读取回来放到内存里计算，内存消耗会非常大。因此 ES 利用了一种叫做 global ordinals 的数据结构来对聚合的字段来做 bucket 分配，这个 ordinals 用有序的数值来代表字段里唯一的一个字符串，因此为每个 ordinals 值分配一个 bucket 就等同于为每个唯一的 term 分配了 bucket。 之后遍历查询结果的时候，可以将结果映射到各个 bucket 里，就可以很快的统计出每个 bucket 里的文档数了。
> 
> 这种计算方式主要开销在构建 global ordinals 和分配 bucket 上，如果索引包含的原始文档非常多，查询结果包含的文档也很多，那么默认的这种计算方式是内存消耗最小，速度最快的。
> 
> 如果指定 execution_hint:map 则会更改聚合执行的方式，这种方式不需要构造 global ordinals，而是直接将查询结果拿回来在内存里构造一个 map 来计算，因此在查询结果集很小的情况下会显著的比 global ordinals 快。
> 
> 要注意的是这中间有一个平衡点，当结果集大到一定程度的时候，map 的内存开销带来的代价可能就抵消了构造 global ordinals 的开销，从而比 global ordinals 更慢，所以需要根据实际情况测试对比一下才能找好平衡点。

---
正是上面的这段对于`Filter or Query`的优化意见，让我也注意到一个词:「不评分查询」，前面的查询结果其实会返回传入的 IDS 列表的对应文档结果，但是我们根本不需要，所以可以不返回 source:

---
通过分片，ES 把数据放在不同节点上，这样可以存储超过单节点容量的数据。而副本分片数的增加可以提高搜索的吞吐量 (主分片与副本都能处理查询请求，ES 会自动对搜索请求进行负载均衡)。在《Mastering Elasticsearch》里面还提到了路由 (routing) 功能 (延伸阅读链接 6)，ES 在写入文档时，文档会通过一个公式路由到一个索引中的一个分片上。默认的选择分片公式如下：

shard_num = hash(\_routing) % num_primary_shards

`_routing`字段的取值默`_id`字段。如果不指定路由，在查询 / 聚合时就需要由 ES 协调节点搜集到每个分片 (每个主分片或者其副本上查询结果，再将查询的结果进行排序然后返回：在我们这个场景，每次要从 5 个分片上聚合。
> 在搜索性能方面，哪种设置会表现得最好？通常情况下，每个节点拥有较少分片的设置会表现得更好。原因是它给每个分片提供了更多的可用文件系统缓存份额，而文件系统缓存可能是Elasticsearch的头号性能因素。
> 
> 那么，什么是正确的复制数量？如果你的集群有num_nodes节点，num_primaries主分片，如果你希望能够一次最多应付max_failures节点故障，那么对你来说正确的复制数量是max(max_failures, ceil(num_nodes / num_primaries) - 1) 。

## es写入的流程
-   协调节点默认使用文档ID参与计算（也支持通过routing），以便为路由提供合适的分片。

```
shard = hash(document_id) % (num_of_primary_shards)
```
-   当分片所在的节点接收到来自协调节点的请求后，会将请求写入到Memory Buffer，然后定时（默认是每隔1秒）写入到Filesystem Cache，这个从Momery Buffer到Filesystem Cache的过程就叫做refresh；
-   当然在某些情况下，存在Momery Buffer和Filesystem Cache的数据可能会丢失，ES是通过translog的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到translog中，当Filesystem cache中的数据写入到磁盘中时，才会清除掉，这个过程叫做flush。
-   在flush过程中，内存中的缓冲将被清除，内容被写入一个新段，段的fsync将创建一个新的提交点，并将内容刷新到磁盘，旧的translog将被删除并开始一个新的translog。 flush触发的时机是定时触发（默认30分钟）或者translog变得太大（默认为512M）时。
-   **merge 过程**

由于自动刷新流程每秒会创建一个新的段 ，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦。 每一个段都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个段；所以段越多，搜索也就越慢。

Elasticsearch通过在后台进行Merge Segment来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段。

当索引的时候，刷新（refresh）操作会创建新的段并将段打开以供搜索使用。合并进程选择一小部分大小相似的段，并且在后台将它们合并到更大的段中。这并不会中断索引和搜索。
一旦合并结束，老的段被删除：

1.  新的段被刷新（flush）到了磁盘。 ** 写入一个包含新段且排除旧的和较小的段的新提交点。
2.  新的段被打开用来搜索。
3.  老的段被删除。

translog相当于wal模型了，先写内存，再写log，因为es很容易写入失败。

设置路由条件，如果Request中指定了路由条件，则直接使用Request中的Routing，否则使用Mapping中配置的，如果Mapping中无配置，则使用默认的_id字段值。

上面详细介绍了Elasticsearch的写入流程及其各个流程的工作机制，我们在这里再次总结下之前提出的分布式系统中的六大特性：

-   可靠性：由于Lucene的设计中不考虑可靠性，在Elasticsearch中通过Replica和TransLog两套机制保证数据的可靠性。
-   一致性：Lucene中的Flush锁只保证Update接口里面Delete和Add中间不会Flush，但是Add完成后仍然有可能立即发生Flush，导致Segment可读。这样就没法保证Primary和所有其他Replica可以同一时间Flush，就会出现查询不稳定的情况，这里只能实现最终一致性。
-   原子性：Add和Delete都是直接调用Lucene的接口，是原子的。当部分更新时，使用Version和锁保证更新是原子的。
-   隔离性：仍然采用Version和局部锁来保证更新的是特定版本的数据。
-   实时性：使用定期Refresh Segment到内存，并且Reopen Segment方式保证搜索可以在较短时间（比如1秒）内被搜索到。通过将未刷新到磁盘数据记入TransLog，保证对未提交数据可以通过ID实时访问到。
-   性能：性能是一个系统性工程，所有环节都要考虑对性能的影响，在Elasticsearch中，在很多地方的设计都考虑到了性能，一是不需要所有Replica都返回后才能返回给用户，只需要返回特定数目的就行；二是生成的Segment现在内存中提供服务，等一段时间后才刷新到磁盘，Segment在内存这段时间的可靠性由TransLog保证；三是TransLog可以配置为周期性的Flush，但这个会给可靠性带来伤害；四是每个线程持有一个Segment，多线程时相互不影响，相互独立，性能更好；五是系统的写入流程对版本依赖较重，读取频率较高，因此采用了versionMap，减少热点数据的多次磁盘IO开销。Lucene中针对性能做了大量的优化。
## es读取的流程
搜索和读取文档都属于读操作，可以从主分片或副分片中读取数据。读取单个文档的流程（图片来自官网）如下图所示。
这个例子中的索引有一个主分片和两个副分片。以下是从主分片或副分片中读取时的步骤：
（1）客户端向NODE1发送读请求。
（2）NODE1使用文档ID来确定文档属于分片0，通过集群状态中的内容路由表信息获知分片0有三个副本数据，位于所有的三个节点中，此时它可以将请求发送到任意节点，这里它将请求转发到NODE2。
（3）NODE2将文档返回给 NODE1,NODE1将文档返回给客户端。

NODE1作为协调节点，会将客户端请求轮询发送到集群的所有副本来实现负载均衡。
在读取时，文档可能已经存在于主分片上，但还没有复制到副分片。在这种情况下，读请求命中副分片时可能会报告文档不存在，但是命中主分片可能成功返回文档。一旦写请求成功返回给客户端，则意味着文档在主分片和副分片都是可用的。

我们需要警惕实时读取特性，GET API默认是实时的，实时的意思是写完了可以立刻读取，但仅限于GET、MGET操作，不包括搜索。在5.x版本之前，GET/MGET的实时读取依赖于从translog中读取实现，5.x版本之后的版本改为refresh，因此系统对实时读取的支持会对写入速度有负面影响。由此引出另一个较深层次的问题是，update操作需要先GET再写，为了保证一致性，update调用GET时将realtime选项设置为true，并且不可配置。因此update操作可能会导致refresh生成新的Lucene分段。

## 如何应对mysql和es的深度分页
如果是mysql的话，应该禁止随机跳页，应该是使用小范围的跳页，每次只能跳十页以内，然后sql就可以改写成：
原分页SQL：

``# 第一页 SELECT * FROM `year_score` where `year` = 2017 ORDER BY id limit 0, 20;  # 第N页 SELECT * FROM `year_score` where `year` = 2017 ORDER BY id limit (N - 1) * 20, 20;  复制代码``

通过上下文关系，改写为：

``# XXXX 代表已知的数据 SELECT * FROM `year_score` where `year` = 2017 and id > XXXX ORDER BY id limit 20; 复制代码``

如果非要随机跳页，那么就需要先根据聚簇索引来过滤出id，再去查询

es可以跟mysql解决方案一样使用from-to

![[Pasted image 20230605142543.png]]

## 当初为什么使用dapeng rpc
dapeng的最早设计是在2013年，当时thrift和protobuf都已经成熟，最终我们选择thrift作为我们的底层数据协议（2者差异不大，选A或B都没有太大差别）。虽然是基于thrift的协议模型，但是dapeng还是做了很多的改进：

- 重新设计 Java POJO 对象的映射模型。官方生成的POJO完全不是面向开发者的，一大堆杂乱无章的序列化相关的代码。（protobuf也是同样的风格，阅读protobuf生成的代码对人来说完全是一种折磨），考虑到 SOA 的接口模型中，数据对象是API最为核心的部分，我们希望生成的代码是符合开发者阅读体验的，所以我们将 POJO 和 序列化代码进行了隔离。
- 更好的 optional 映射。 null是Java中一个尴尬的存在，NPE则是服务开发者中的一个小噩梦。我们希望使用 Optional[T] 来强语意的表达 null。
- 扩展数据类型支持。最为常见的，是对 Timestamp 和 BigDecimal 的支持。在商务类服务中，不支持这两个数据类型，是不完备的。
### dapeng 应用层/会话层协议设计

#### 请求/返回结构

```
i32  frameLength
byte[frameLength-4]  frameData
    byte stx = 0x02
    byte version = 0x01
    byte protocol
    i32  seqId
    byte[?]  header-thrift-struct
    byte[?]  body-thrift-struct
    byte  etx = 0x03
```

这里的i32/byte等都是采用非压缩的thrift二进制格式，其中 header-thrift-struct 由dapeng core定义，对应于 `com.github.dapeng.core.SoaHeader`, 包含了本次请求/回应的重要信息：包括serviceName, version, method，也包括了请求和返回的更多信息，这些信息是服务路由、服务限流、调用跟踪等诸多服务治理所需要的，详情在下面有介绍。

整个传输包采用带有包长度界定的方式，首先传输整个包的长度，然后是包的内容，包的内容中由以0x02开头，0x03结束，这样设计的优点：

- 可以高效的进行拆包、粘包的处理。在最外层可以不需要关系协议内部的具体内容，而简单的进行断包。
- 有简单的容错机制，如果包不是0x02开头，0x03结尾，则当前的会话出现了通信错乱，当前会话需要废弃。

version 字段目前值为 1, 这个字段是保留给后续进行版本升级的，在设计上，新版本的dapeng应该有能力兼容旧版本的协议。如果version升级，version后的内容将不需要沿用目前的版本，可以灵活扩展。

seqId 是当前请求/返回的序列号，在用一个会话（TCP连接）上，可以同时发送多个请求，而无需等待前面的请求完成，才能发送后面的请求，这一般称之为多路复用。采用多路复用技术后，可以大大提高客户端是多线程环境下的请求处理能力，不再需要每一个线程创建一个TCP连接，而是大家可以共享一个连接。这种模式在异步编程中也是非常有必要的。在多路复用时，seqId用来标注当前的返回所对应的是哪一个请求。

采用上述的包结构，也设定了dapeng不是面向流的，而是面向 request - response的。所以，目前来说，dapeng 不适合流处理，比如说一次请求，后续会逐步的收到多个数据返回的情况。
### SoaHeader

SoaHeader是dapeng服务调用的一个核心数据结构，用于传输在应用层（方法参数）之外的参数，传统的thrift调用是没有这一块的开销的，dapeng引入了这个新的结构体，带来了额外的开销，也带来了丰富的额外功能，这些构成了服务化治理的各种收益。

```
1 optional string serviceName
2 optional string methodName
3 optional string version
4 optional string callerMid
5 optional string callerIp
6 optional i32 callerPort
7 optional string sessionTid
8 optional string userIp
9 optional string callerTid
10 optional i32 timeout
11 optional string respCode
12 optional string respMessage
13 optional string calleeTid
14 optional string calleeIp
15 optional i64 operatorId
16 optional i32 calleePort
17 optional i64 userId
18 optional string calleeMid
19 optional i32 transactionId
20 optional i32 transactionSequence
21 optional i32 calleeTime1
22 optional i32 calleeTime2
23 map<string,string> cookie
```

- serviceName, version 对请求包来说是必须的，对返回包来说是不必须的。dapeng使用service:version来定位服务，同一个服务可以有多个版本(major.minor.patch):
    - major 大版本。当大版本号升级时，表示新服务版本在API上出现重大变更，并对当前版本不兼容。即3.y.z的服务不兼容2.y.z的服务。在同一个大版本下，服务需要提供向下兼容，即1.1.10应该兼容1.1.9，也应该兼容1.0.12。
    - minor 小版本。当API升级但同时兼容当前的版本时，可以增加minor版本号，如：
        - 新增新的方法
        - 在请求结构体中新增可选的字段
        - 在返回结构体中新增字段
        - 不改变现有结构体的字段tag值，及数据类型(未来版本可能会支持从byte,i16,i32到i16,i32,i64的升级) 开发人员可以使用dapeng的命令行工具来检查服务的版本兼容性，防止在破坏了兼容性的情况下，没有正确同步升级服务版本。
    - patch 一般的，在API不发生变化的情况下，但是对服务的实现进行了升级，或者进行了bug修复。这时，可以升级patch版本号。
- method 一个服务可以包含一组method，method是最小的可调用的服务单元。
- 客户端信息相关信息
    - callerMid: 调用者ModuleId。mid用来标识一个分布式应用中的一个参与者，包括：
        - web前端
        - 服务。
        - 定时任务
        - 脚本或工具
        - 消息消费者 通过为每一个参与者分配一个mid，可以在调用链跟踪系统中统计，监控这些参与者之间的调用情况，包括调用次数，服务时间，成功率等指标
    - callerIp：服务调用者IP
    - callerPort：服务调用者Port
    - userIp：用于跟踪最原始的userIp，这个值是在服务调用链传递的，可用于反馈最初发起服务的user所在的浏览器IP。
    - operatorId：内部操作员ID
    - userId：外部用户ID
- 服务端相关信息
    - calleeMid
    - calleeIp
    - calleePort
    - respCode
    - respMessage
    - calleeTime1
    - calleeTime2
- 线索相关信息：这一块在服务调用跟踪中详细介绍
    - sessionTid
    - callerTid
    - calleeTid
- 分布式事务：在目前的dapeng 2.0版本中暂不支持。
    - transactionId
    - transactionSequence
- cookie: 在整个服务调用链中是传输的。后续版本还会支持服务端可以返回新的cookie给到客户端。

### 常用算法

#### Random（随机）

最简单的算法，通过随机来选定最终的服务器。

#### Round Robbin（轮询）

顾名思义，依次使用不同的服务器来处理业务。保证每台服务器都能分配到业务。

#### Least Load（最小负载）

该算法可以根据服务的负载情况来选择，有一定的智能性。衡量服务器负载的度量有很多。在dapneg中，选择的最小连接数（leastActive）来表示。

#### ConsistentHash （一致性哈希）

根据参数进行hash来选定服务器。在dapeng中目前还未实现。

### 权重设计

上述算法都是默认所有的服务器的权重是一致的。但是在实际中，根据服务器的性能和环境不同，需要对其配以不同的权重才能满足生产需求，因此在dapeng的负载均衡算法中对其进行了权重设计。因为最小连接数算法并未涉及到权重，因此dapeng仅对随机和轮询算法进行了权重设计。

#### 带权重的随机算法

在服务实例的权重进行更改后或者删减和增加服务实例后，对所有实例进行遍历，在遍历的过程中，判断所有实例的权重是否相同并计算总权重。如果所有实例的权重相同则随机选取一个实例（与权重无关），如果权重不相同，则以总权重为上限，随机产生一个offset，然后遍历实例一次减去当前实例的权重，当offset<0时，则选到目标实例。 如此权重越高的实例，被选中的可能性越高。

#### 带权重的轮询算法

该算法同样是在服务实例的权重进行更改后或者删减和增加服务实例后遍历所有的实例，遍历过程中判断权重是否相同，如果相同则使用无权重的轮询。如果不相同，则要计算所有实例的权重的最大公约数。然后轮询时，使用lastIndex和currentWeight记录上次轮询的状态，每次轮询时使用currentWeight减去权重最大公约数，只有当当前实例的权重大于currentWeight时才选择该实例。例如有3个实例权重比为1:10:1，会选择第二个实例 9次，再依次轮询3个实例。

### 负载均衡算法及权重配置

在dapeng带权重的负载均衡中支持对全局的配置和服务的配置。

在ZK的节点路径：soa/config/services下配置，即代表全局配置，如下：

```
   loadBlance/random
   weight/127.0.0.1/9095/700
   weight/127.0.0.1/500
```

每行表示一个功能配置。该全局配置表示：

- 对所有服务设置的负载均衡算法为random算法；
- 对在127.0.0.1:9095下的所有服务设置权重为600；
- 对在127.0.0.1下的所有服务设置权重为500.

当上述配置中的第二条和第三条同时配置时，前者的优先级高。

在ZK的节点路径：soa/config/services/xxx.service下对相应的服务进行权重设置，根据实际情况动态的改变该服务下的实例的权重值，从而改变负载均衡算法选择的服务实例。该配置即代表对具体服务的配置，配置格式如下：

```
   loadBalance/random,method1:leastActive,method2:roundRobin
   weight/127.0.0.1/9095/700
   weight/127.0.0.1/500
```

每行表示一个功能配置，在配置负载均衡算法时，按”/”分割服务级别和方法级别的配置，包含”/”表示服务级别，不包含则表示方法级别。方法名与算法名使用”:”分隔，多个设置使用”,”分隔。

该对服务的配置表示：

- 对该服务设置的负载均衡算法为random算法，对该服务下的两个方法method1和method2分别设置leastActive和roundRobin算法；
- 对该服务下的127.0.0.1:9095服务实例设置权重为700;
- 对该服务下IP为127.0.0.1的所有服务实例设置权重为500;

当出现如上设置情况时：指定设置的特定实例127.0.0.1:9095与指定设定的IP：127.0.0.1相同时，则以配置weight/127.0.0.1/9095/700为准，即前者的优先级高。



## 2PC与3PC是什么，两者的流程，以及优缺点  

从名字我们可以理解到，2PC就是有两个阶段，一个是准备阶段，一个是提交阶段
![[Pasted image 20230607105102.png]]
Prepare阶段：协调者发起提议，问大家是否接受，这个阶段做了除提交事务外的所有事情。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbc00d762f54455baa33036eba991eef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

Commit阶段：根据参与者的反馈，通知大家提交或者终止事务，如果参与者全部同意就提交，只要有一个参与者不同意就终止。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6720adeb56eb4a9fa8f92442c357662d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2PC能协调参与者统一提交或者终止，按道理是能实现我们需要的分布式事务的，但是我们需要考虑网络波动以及协调者挂掉的情况，在这些情况下2PC是会存在一些问题的。

单点故障：在上图可以看得出协调者是一个非常重要的角色，一旦协调者发生故障，那么参与者会一直阻塞下去，无法完成事务操作。

性能问题：无论是上面的第一个阶段还是第二个阶段，所有的参与者和协调者的资源都是被锁定的，要等所有参与者准备完毕，协调者才会通知进行全局提交。

数据不一致：在第二阶段的时候，有的参与者接受到commit请求后，发生了网络波动，导致部分参与者接收到commit请求但是部分没有接受到，导致当接受到命令的参与者执行命令后导致数据不一致的问题。
### **协调者故障分析**

**协调者是一个单点，存在单点故障问题**。

假设协调者在**发送准备命令之前**挂了，还行等于事务还没开始。

假设协调者在**发送准备命令之后**挂了，这就不太行了，有些参与者等于都执行了处于事务资源锁定的状态。不仅事务执行不下去，还会因为锁定了一些公共资源而阻塞系统其它操作。

假设协调者在**发送回滚事务命令之前**挂了，那么事务也是执行不下去，且在第一阶段那些准备成功参与者都阻塞着。

假设协调者在**发送回滚事务命令之后**挂了，这个还行，至少命令发出去了，很大的概率都会回滚成功，资源都会释放。但是如果出现网络分区问题，某些参与者将因为收不到命令而阻塞着。

假设协调者在**发送提交事务命令之前**挂了，这个不行，傻了！这下是所有资源都阻塞着。

假设协调者在**发送提交事务命令之后**挂了，这个还行，也是至少命令发出去了，很大概率都会提交成功，然后释放资源，但是如果出现网络分区问题某些参与者将因为收不到命令而阻塞着。
### **协调者故障，通过选举得到新协调者**

因为协调者单点问题，因此我们可以通过选举等操作选出一个新协调者来顶替。

如果处于第一阶段，其实影响不大都回滚好了，在第一阶段事务肯定还没提交。

如果处于第二阶段，假设参与者都没挂，此时新协调者可以向所有参与者确认它们自身情况来推断下一步的操作。

假设有个别参与者挂了！这就有点僵硬了，比如协调者发送了回滚命令，此时第一个参与者收到了并执行，然后协调者和第一个参与者都挂了。

此时其他参与者都没收到请求，然后新协调者来了，它询问其他参与者都说OK，但它不知道挂了的那个参与者到底O不OK，所以它傻了。

问题其实就出在**每个参与者自身的状态只有自己和协调者知道**，因此新协调者无法通过在场的参与者的状态推断出挂了的参与者是什么情况。
2PC 是一种**尽量保证强一致性**的分布式事务，因此它是**同步阻塞**的，而同步阻塞就导致长久的资源锁定问题，**总体而言效率低**，并且存在**单点故障**问题，在极端条件下存在**数据不一致**的风险。

当然具体的实现可以变形，而且 2PC 也有变种，例如 Tree 2PC、Dynamic 2PC。

还有一点不知道你们看出来没，2PC 适用于**数据库层面的分布式事务场景**，而我们业务需求有时候不仅仅关乎数据库，也有可能是上传一张图片或者发送一条短信。

3PC 的出现是为了解决 2PC 的一些问题，相比于 2PC 它在**参与者中也引入了超时机制**，并且**新增了一个阶段**使得参与者可以利用这一个阶段统一各自的状态。

让我们来详细看一下。

3PC 包含了三个阶段，分别是**准备阶段、预提交阶段和提交阶段**，对应的英文就是：`CanCommit、PreCommit 和 DoCommit`。

看起来是**把 2PC 的提交阶段变成了预提交阶段和提交阶段**，但是 3PC 的准备阶段协调者只是询问参与者的自身状况，比如你现在还好吗？负载重不重？这类的。

而预提交阶段就是和 2PC 的准备阶段一样，除了事务的提交该做的都做了。

提交阶段和 2PC 的一样，让我们来看一下图。
![[Pasted image 20230607105325.png]]
三阶段提交是二阶段提交的升级版，改动点如下

1.引入超时机制

2.在第一阶段和第二阶段中间插入了一个准备阶段，保证了在最后提交阶段前各节点的状态一致。

阶段一：canCommit: 协调者向所有的参与者询问是否可以提交事务，参与者接到请求后根据自己的情况返回yes或者是no。

阶段二：preCommit: 协调者根据第一阶段参与者反馈的情况判断是否可以执行基于事务的preCommit操作（如果阶段一全部返回yes则向所有参与者发出preCommit请求，进入准备阶段，参与者接受到请求后，执行事务操作，将undo和redo信息记录到事务日志中，但是不提交事务，然后参与者向协调者反馈ack响应或者no响应），这里做了除提交事务外的所有事情。

阶段三：do Commit: 真正提交事务的阶段，如果阶段二中所有的参与者返回的都是ack，那么协调者会向参与者发出do commit请求，参与者收到请求后会执行事务提交，完成后向发起者反馈ack消息；如果阶段二中只要有一个参与者反馈no或者是超时后协调者无法收到所有参与者的反馈，即中断事务，中断是由协调者向参与者发出abort请求，参与者使用阶段一的undo信息执行回滚操作，操作完成后向协调者返回ack。

当进入到阶段三，无论是协调者出现问题，或者是协调者和参与者网络出现了问题，都可能会导致参与者无法接收到协调者发出的do Commit请求或者abort请求，此时参与者都会在超时后自动提交事务。

总结：虽然相对于2PC进行了一些优化，不会在准备阶段就直接锁住资源；但因为3PC毕竟多引入了一个阶段，所以相对于2PC性能会更差，而且依然会存在数据不一致的问题。
## **TCC**

**2PC 和 3PC 都是数据库层面的，而 TCC 是业务层面的分布式事务**，就像我前面说的分布式事务不仅仅包括数据库的操作，还包括发送短信等，这时候 TCC 就派上用场了！

TCC 指的是`Try - Confirm - Cancel`。

- Try 指的是预留，即资源的预留和锁定，**注意是预留**。
- Confirm 指的是确认操作，这一步其实就是真正的执行了。
- Cancel 指的是撤销操作，可以理解为把预留阶段的动作撤销了。

其实从思想上看和 2PC 差不多，都是先试探性的执行，如果都可以那就真正的执行，如果不行就回滚。

比如说一个事务要执行A、B、C三个操作，那么先对三个操作执行预留动作。如果都预留成功了那么就执行确认操作，如果有一个预留失败那就都执行撤销动作。

我们来看下流程，TCC模型还有个事务管理者的角色，用来记录TCC全局事务状态并提交或者回滚事务。

![](https://pic4.zhimg.com/80/v2-90179fa933c0a389ffa6ac04e244a58f_720w.webp)

可以看到流程还是很简单的，难点在于业务上的定义，对于每一个操作你都需要定义三个动作分别对应`Try - Confirm - Cancel`。

因此 **TCC 对业务的侵入较大和业务紧耦合**，需要根据特定的场景和业务逻辑来设计相应的操作。

还有一点要注意，撤销和确认操作的执行可能需要重试，因此还需要保证**操作的幂等**。

相对于 2PC、3PC ，TCC 适用的范围更大，但是开发量也更大，毕竟都在业务上实现，而且有时候你会发现这三个方法还真不好写。不过也因为是在业务上实现的，所以**TCC可以跨数据库、跨不同的业务系统来实现事务**。

## **本地消息表**

本地消息表其实就是利用了 **各系统本地的事务**来实现分布式事务。

本地消息表顾名思义就是会有一张存放本地消息的表，一般都是放在数据库中，然后在执行业务的时候 **将业务的执行和将消息放入消息表中的操作放在同一个事务中**，这样就能保证消息放入本地表中业务肯定是执行成功的。

然后再去调用下一个操作，如果下一个操作调用成功了好说，消息表的消息状态可以直接改成已成功。

如果调用失败也没事，会有 **后台任务定时去读取本地消息表**，筛选出还未成功的消息再调用对应的服务，服务更新成功了再变更消息的状态。

这时候有可能消息对应的操作不成功，因此也需要重试，重试就得保证对应服务的方法是幂等的，而且一般重试会有最大次数，超过最大次数可以记录下报警让人工处理。

可以看到本地消息表其实实现的是**最终一致性**，容忍了数据暂时不一致的情况。

## **消息事务**

RocketMQ 就很好的支持了消息事务，让我们来看一下如何通过消息实现事务。

第一步先给 Broker 发送事务消息即半消息，**半消息不是说一半消息，而是这个消息对消费者来说不可见**，然后**发送成功后发送方再执行本地事务**。

再根据**本地事务的结果向 Broker 发送 Commit 或者 RollBack 命令**。

并且 RocketMQ 的发送方会提供一个**反查事务状态接口**，如果一段时间内半消息没有收到任何操作请求，那么 Broker 会通过反查接口得知发送方事务是否执行成功，然后执行 Commit 或者 RollBack 命令。

如果是 Commit 那么订阅方就能收到这条消息，然后再做对应的操作，做完了之后再消费这条消息即可。

如果是 RollBack 那么订阅方收不到这条消息，等于事务就没执行过。

可以看到通过 RocketMQ 还是比较容易实现的，RocketMQ 提供了事务消息的功能，我们只需要定义好事务反查接口即可。

![](https://pic2.zhimg.com/80/v2-72ba7bed684e855606c44ddda185987d_720w.webp)

可以看到消息事务实现的也是最终一致性。

## **最大努力通知**

其实我觉得本地消息表也可以算最大努力，事务消息也可以算最大努力。

就本地消息表来说会有后台任务定时去查看未完成的消息，然后去调用对应的服务，当一个消息多次调用都失败的时候可以记录下然后引入人工，或者直接舍弃。这其实算是最大努力了。

事务消息也是一样，当半消息被commit了之后确实就是普通消息了，如果订阅者一直不消费或者消费不了则会一直重试，到最后进入死信队列。其实这也算最大努力。

所以**最大努力通知其实只是表明了一种柔性事务的思想**：我已经尽力我最大的努力想达成事务的最终一致了。

适用于对时间不敏感的业务，例如短信通知。

## **总结**

可以看出 2PC 和 3PC 是一种强一致性事务，不过还是有数据不一致，阻塞等风险，而且只能用在数据库层面。

而 TCC 是一种补偿性事务思想，适用的范围更广，在业务层面实现，因此对业务的侵入性较大，每一个操作都需要实现对应的三个方法。

本地消息、事务消息和最大努力通知其实都是最终一致性事务，因此适用于一些对时间不敏感的业务。

## 基于本地消息表常用的分布式事务解决方案

上面提到了分布式系统中要实现强一致性比较困难，往往很多业务场景不要求强一致性，允许有个临时的业务中间状态。因此就可以采用最终一致性的分布式事务方案。

常用的分布式解决方案有实现XA事务的Atomikos，本地消息表方案，基于消息中间件的最终一致性方案，TCC方案，阿里的SEATA,SAGA方案和最大努力通知。下面主要对基于本地消息表实现最终一致性的分布式事务方案进行介绍。

本地消息表方案最初是ebay提出的，其实也是BASE理论的应用，属于可靠消息最终一致性的范畴。这里以支付服务和会计服务为例展开介绍本地消息表方案，大概流程是这样子：用户在支付服务完成了支付订单支付成功后，此时会调用会计服务的接口生成一条原始的会计凭证到数据库中，如图1所示。这里必须明确：支付服务处理完订单支付等逻辑后，此时若直接调用会计服务生成会计凭证数据的接口肯定会遇到分布式事务的问题。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/9/16f8968129a8fd22~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

图1

因为用户完成支付后，此时得立马给用户一个支付的反馈，要做的就是提醒用户支付成功。因为会计服务生成的会计凭证保存到数据库的过程中可以对用户透明，用户也无需知道有这么一个流程，为了提高响应速度和解耦，因此可以引入mq来做到异步生成会计凭证，即用户完成支付订单支付后，此时可以将消息投递到mq中，然后会计服务再去监听mq消息去处理消费逻辑。此时如图2：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/9/16f8968977db1f16~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

图2

在支付服务和会计服务之间引入mq后，此时又引入了新的问题。大概列举如下：

1，若支付服务完成支付逻辑后，在投递消息到mq中间件的过程中由于网络抖动等原因，没有投递到mq中导致消息丢失了怎么办？

2，mq接收到消息后，由于内部原因导致消息丢失了怎么办？

3，会计服务在监听消息的过程中，由于网络原因没有接收到消息或消费过程中遇到异常，此时也会导致消息丢失，测试怎么办？

经过以上分析，mq可能会丢失消息，传统的mq没有实现分布式事务（注意rocketmq的某些版本有实现分布式事务功能），因此这里可以引入本地消息表结合mq的方式来解决分布式事务的问题，保证消息的可靠投递。

图3是由图2细化后的图，其中红框处引入了一个本地消息表。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/9/16f89694bba58c9c~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

图3

根据图3，正向流程步骤大概如下：

1）在支付库中引入一张消息表来记录支付消息，即用户支付成功后同时往这张消息表插入一条支付成功的消息，状态为“发送中”。注意支付逻辑和插入消息表的代码要包裹在一个事务里面，这里保证了本地事务的强一致性。即支付逻辑和插入消息表的消息组成了一个强一致性的事务，要么同时成功，要么同时失败。

2）完成 1）步的逻辑后，此时再向mq的PAY_QUEUE队列中投递一条支付消息，这条支付消息的内容跟保存在支付库消息表的消息内容一致。

3）mq接收到消息后，此时会计服务也监听到这条消息了，此时会计服务处理消费逻辑即开始生成会计凭证。

4）会计凭证生成后，再反向向mq投递一条消费成功的消息到ACC_QUEUE队列

5）同时支付服务又来监听这个会计服务消费成功的消息，当支付服务监听到这个消费成功的消息后，此时再将本地消息表的消息状态改为“已发送”。

6）经过前面5步后，整个业务就已经完成了。

以上是引入本地消息表后的正常的业务流程，前文分析过生产者，mq和消费者三个环节中都可能弄丢消息，即图4中的红框处可能会造成消息丢失。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/9/16f8969cb9537e50~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

图4

此时可能你会有个疑问：用户支付成功后，若消息在投递过程中丢失了就丢失了，会计服务那边也消费不到了，此时同样也会造成支付服务（生产者）和会计服务（消费者）之间的数据不一致。

之前增加的本地消息表好像也没起作用啊？

那此时怎么办呢？如何来解决消息丢失的问题，做到消息的可靠投递呢？

其实解决方案就是消息重复投递，但消费者的消费接口要实现幂等性。

怎么来让消息重复投递呢？此时本地消息表就派上用场了，刚才我们在支付库中新增加了一张本地消息表，即支付等逻辑处理成功，这张本地消息表也会记录一条记录，此时的消息状态是“发送中”。若第一次生产者投递的消息丢失后，此时我们只要将这张本地消息表状态为“发送中”的消息重新投递即可，直到消费者消费成功为止，消费者消费成功后将这条消息的状态改为“已发送”即可。

因此为了能将丢失后的消息重发，此时我们引入一个定时任务好了，暂且叫它“消息恢复系统”吧，如下图所示。这个消息恢复系统就是每隔一段时间去本地消息表中捞取状态为“发送中”的消息，然后重新投递到mq中间件中，然后消费者就会重新消费了。若消费者已经消费过了，此时就不再处理消费业务逻辑，直接反向投递一条消费成功的消息到mq中，此时原来的生产者此时也会监听这条消费成功的消息，将本地消息表的消息状态改为“已发送”，此时消息恢复系统就不会再去捞取这条状态为“已发送”的消息，然后进行重新投递了。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/9/16f896a251355018~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

图5

此时若消息丢失后且消息恢复系统在重新投递过程中，也可能会再次投递失败。此时我们一般会指定最大重试次数，重试间隔时间根据重试次数而线性增长。若达到最大重试次数后，同时记录日志，我们可以根据记录的日志来通过邮件或短信来发送告警通知，接收到告警通知后及时介入人工处理即可。

基于本地消息表的分布式事务方案就介绍到这里了，本地消息表的方案的优点是建设成本比较低，其虽然实现了可靠消息的传递确保了分布式事务的最终一致性，其实它也有一些缺陷：

1）本地消息表与业务耦合在一起，难于做成通用性，不可独立伸缩。

2）本地消息表是基于数据库来做的，而数据库是要读写磁盘IO的，因此在高并发下是有性能瓶颈的

## TCC
> 柔性事务：分布式理论的AP，遵循BASE，允许一定时间内不同节点的数据不一致，但要求最终一致。

### [#](#补偿事务-tcc) 补偿事务 (TCC)

> TCC（Try-Confirm-Cancel）又被称补偿事务，TCC与2PC的思想很相似，事务处理流程也很相似，但**2PC是应用于在DB层面，TCC则可以理解为在应用层面的2PC，是需要我们编写业务逻辑来实现**。

TCC它的核心思想是："针对每个操作都要注册一个与其对应的确认（Try）和补偿（Cancel）"。

还拿下单扣库存解释下它的三个操作：

- **Try阶段**：下单时通过Try操作去扣除库存预留资源。
- **Confirm阶段**：确认执行业务操作，在只预留的资源基础上，发起购买请求。
- **Cancel阶段**：只要涉及到的相关业务中，有一个业务方预留资源未成功，则取消所有业务资源的预留请求。
![[Pasted image 20230607115729.png]]
**TCC的缺点**：

1.空回滚

当一个分支事务所在的服务发生宕机或者网络异常导致调用失败，并未执行try方法，当恢复后事务执行回滚操作就会调用此分支事务的cancel方法,如果cancel方法不能处理此种情况就会出现空回滚。

是否出现空回滚，我们需要需要判断是否执行了try方法，如果执行了就没有空回滚。解决方法就是当主业务发起事务时，生成一个全局事务记录，并生成一个全局唯一ID，贯穿整个事务，再创建一张分支事务记录表，用于记录分支事务，try执行时将全局事务ID和分支事务ID存入分支事务表中，表示执行了try阶段，当cancel执行时，先判断表中是否有该全局事务ID的数据，如果有则回滚，否则不做任何操作。比如seata的AT模式中就有分支事务表。

2.幂等问题

由于服务宕机或者网络问题，方法的调用可能出现超时，为了保证事务正常执行我们往往会加入重试的机制，因此就需要保证confirm和cancel阶段操作的幂等性。

我们可以在分支事务记录表中增加事务执行状态，每次执行confirm和cancel方法时都查询该事务的执行状态，以此判断事务的幂等性。

3.悬挂问题

TCC中，在调用try之前会先注册分支事务，注册分支事务之后，调用出现超时，此时try请求还未到达对应的服务，因为调用超时了，所以会执行cancel调用，此时cancel已经执行完了，然而这个时候try请求到达了，这个时候执行了try之后就没有后续的操作了，就会导致资源挂起，无法释放。

执行try方法时我们可以判断confirm或者cancel方法是否执行，如果执行了那么就不执行try阶段。同样借助分支事务表中事务的执行状态。如果已经执行了confirm或者cancel那么try就执行。



## 简述ZAB协议  
## Zab 一致性协议

ZooKeeper 是通过 Zab 协议来保证分布式事务的最终一致性。Zab（ZooKeeper Atomic Broadcast，ZooKeeper 原子广播协议）支持崩溃恢复，基于该协议，ZooKeeper 实现了一种主备模式的系统架构来保持集群中各个副本之间数据一致性。

系统架构可以参考下面这张图：

![img](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%9845%E8%AE%B2-%E5%AE%8C/assets/Ciqah16O5QyAB4rJAAEiJ-4T3bE046.png)

在 ZooKeeper 集群中，所有客户端的请求都是写入到 Leader 进程中的，然后，由 Leader 同步到其他节点，称为 Follower。在集群数据同步的过程中，如果出现 Follower 节点崩溃或者 Leader 进程崩溃时，都会通过 Zab 协议来保证数据一致性。

Zab 协议的具体实现可以分为以下两部分：

-   消息广播阶段

Leader 节点接受事务提交，并且将新的 Proposal 请求广播给 Follower 节点，收集各个节点的反馈，决定是否进行 Commit，在这个过程中，也会使用上一课时提到的 Quorum 选举机制。

-   崩溃恢复阶段

如果在同步过程中出现 Leader 节点宕机，会进入崩溃恢复阶段，重新进行 Leader 选举，崩溃恢复阶段还包含数据同步操作，同步集群中最新的数据，保持集群的数据一致性。

整个 ZooKeeper 集群的一致性保证就是在上面两个状态之前切换，当 Leader 服务正常时，就是正常的消息广播模式；当 Leader 不可用时，则进入崩溃恢复模式，崩溃恢复阶段会进行数据同步，完成以后，重新进入消息广播阶段。

## Zab 协议中的 Zxid

Zxid 在 ZooKeeper 的一致性流程中非常重要，在详细分析 Zab 协议之前，先来看下 Zxid 的概念。

Zxid 是 Zab 协议的一个事务编号，Zxid 是一个 64 位的数字，其中低 32 位是一个简单的单调递增计数器，针对客户端每一个事务请求，计数器加 1；而高 32 位则代表 Leader 周期年代的编号。

这里 Leader 周期的英文是 epoch，可以理解为当前集群所处的年代或者周期，对比另外一个一致性算法 Raft 中的 Term 概念。在 Raft 中，每一个任期的开始都是一次选举，Raft 算法保证在给定的一个任期最多只有一个领导人。

![img](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%9845%E8%AE%B2-%E5%AE%8C/assets/Cgq2xl6O5QyAeZqMAAB5C-BWbeI425.png)

Zab 协议的实现也类似，每当有一个新的 Leader 选举出现时，就会从这个 Leader 服务器上取出其本地日志中最大事务的 Zxid，并从中读取 epoch 值，然后加 1，以此作为新的周期 ID。总结一下，高 32 位代表了每代 Leader 的唯一性，低 32 位则代表了每代 Leader 中事务的唯一性。

## Zab 流程分析

Zab 的具体流程可以拆分为消息广播、崩溃恢复和数据同步三个过程，下面我们分别进行分析。

![img](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%9845%E8%AE%B2-%E5%AE%8C/assets/Ciqah16O5QyADv0LAAA84x9hlQc078.png)

### 消息广播

在 ZooKeeper 中所有的事务请求都由 Leader 节点来处理，其他服务器为 Follower，Leader 将客户端的事务请求转换为事务 Proposal，并且将 Proposal 分发给集群中其他所有的 Follower。

完成广播之后，Leader 等待 Follwer 反馈，当有过半数的 Follower 反馈信息后，Leader 将再次向集群内 Follower 广播 Commit 信息，Commit 信息就是确认将之前的 Proposal 提交。

这里的 Commit 可以对比 SQL 中的 COMMIT 操作来理解，MySQL 默认操作模式是 autocommit 自动提交模式，如果你显式地开始一个事务，在每次变更之后都要通过 COMMIT 语句来确认，将更改提交到数据库中。

Leader 节点的写入也是一个两步操作，第一步是广播事务操作，第二步是广播提交操作，其中过半数指的是反馈的节点数 >=N/2+1，N 是全部的 Follower 节点数量。

消息广播的过程描述可以参考下图：

![img](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%9845%E8%AE%B2-%E5%AE%8C/assets/Cgq2xl6O5Q2ASjMpAAHdAdF67vE736.png)

-   客户端的写请求进来之后，Leader 会将写请求包装成 Proposal 事务，并添加一个递增事务 ID，也就是 Zxid，Zxid 是单调递增的，以保证每个消息的先后顺序；
-   广播这个 Proposal 事务，Leader 节点和 Follower 节点是解耦的，通信都会经过一个先进先出的消息队列，Leader 会为每一个 Follower 服务器分配一个单独的 FIFO 队列，然后把 Proposal 放到队列中；
-   Follower 节点收到对应的 Proposal 之后会把它持久到磁盘上，当完全写入之后，发一个 ACK 给 Leader；
-   当 Leader 收到超过半数 Follower 机器的 ack 之后，会提交本地机器上的事务，同时开始广播 commit， Follower 收到 commit 之后，完成各自的事务提交。

分析完消息广播，我们再来看一下崩溃恢复。

### 崩溃恢复

消息广播通过 Quorum 机制，解决了 Follower 节点宕机的情况，但是如果在广播过程中 Leader 节点崩溃呢？

这就需要 Zab 协议支持的崩溃恢复，崩溃恢复可以保证在 Leader 进程崩溃的时候可以重新选出 Leader，并且保证数据的完整性。

崩溃恢复和集群启动时的选举过程是一致的，也就是说，下面的几种情况都会进入崩溃恢复阶段：

-   初始化集群，刚刚启动的时候
-   Leader 崩溃，因为故障宕机
-   Leader 失去了半数的机器支持，与集群中超过一半的节点断连

崩溃恢复模式将会开启新的一轮选举，选举产生的 Leader 会与过半的 Follower 进行同步，使数据一致，当与过半的机器同步完成后，就退出恢复模式， 然后进入消息广播模式。

Zab 中的节点有三种状态，伴随着的 Zab 不同阶段的转换，节点状态也在变化：![img](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%9845%E8%AE%B2-%E5%AE%8C/assets/Cgq2xl6O5-SASu1cAABBzFJh3s0114.png)我们通过一个模拟的例子，来了解崩溃恢复阶段，也就是选举的流程。

假设正在运行的集群有五台 Follower 服务器，编号分别是 Server1、Server2、Server3、Server4、Server5，当前 Leader 是 Server2，若某一时刻 Leader 挂了，此时便开始 Leader 选举。

选举过程如下：

**1****.各个节点****变更状态****，变更为** **Looking**

ZooKeeper 中除了 Leader 和 Follower，还有 Observer 节点，Observer 不参与选举，Leader 挂后，余下的 Follower 节点都会将自己的状态变更为 Looking，然后开始进入 Leader 选举过程。

**2****.各个** **Server** **节点都****会发出一个投票****，参与选举**

在第一次投票中，所有的 Server 都会投自己，然后各自将投票发送给集群中所有机器，在运行期间，每个服务器上的 Zxid 大概率不同。

**3****.****集群接收来自各个服务器的投票，开始处理投票和选举**

处理投票的过程就是对比 Zxid 的过程，假定 Server3 的 Zxid 最大，Server1 判断 Server3 可以成为 Leader，那么 Server1 就投票给 Server3，判断的依据如下：

> 首先选举 epoch 最大的

> 如果 epoch 相等，则选 zxid 最大的

> 若 epoch 和 zxid 都相等，则选择 server id 最大的，就是配置 zoo.cfg 中的 myid

在选举过程中，如果有节点获得超过半数的投票数，则会成为 Leader 节点，反之则重新投票选举。

![img](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%9845%E8%AE%B2-%E5%AE%8C/assets/Ciqah16O5Q2AL03bAADZjhkw3ys031.png)

**4****.****选举成功，改变服务器的状态****，参考上面这张图的状态变更**

### 数据同步

崩溃恢复完成选举以后，接下来的工作就是数据同步，在选举过程中，通过投票已经确认 Leader 服务器是最大Zxid 的节点，同步阶段就是利用 Leader 前一阶段获得的最新Proposal历史，同步集群中所有的副本。

上面分析了 Zab 协议的具体流程，接下来我们对比一下 Zab 协议和 Paxos 算法。

## Zab 与 Paxos 算法的联系与区别

Paxos 的思想在很多分布式组件中都可以看到，Zab 协议可以认为是基于 Paxos 算法实现的，先来看下两者之间的联系：

-   都存在一个 Leader 进程的角色，负责协调多个 Follower 进程的运行
-   都应用 Quorum 机制，Leader 进程都会等待超过半数的 Follower 做出正确的反馈后，才会将一个提案进行提交
-   在 Zab 协议中，Zxid 中通过 epoch 来代表当前 Leader 周期，在 Paxos 算法中，同样存在这样一个标识，叫做 Ballot Number

两者之间的区别是，Paxos 是理论，Zab 是实践，Paxos 是论文性质的，目的是设计一种通用的分布式一致性算法，而 Zab 协议应用在 ZooKeeper 中，是一个特别设计的崩溃可恢复的原子消息广播算法。

Zab 协议增加了崩溃恢复的功能，当 Leader 服务器不可用，或者已经半数以上节点失去联系时，ZooKeeper 会进入恢复模式选举新的 Leader 服务器，使集群达到一个一致的状态。

## 总结

这一课时的内容分享了 ZooKeeper 一致性实现，包括 Zab 协议中的 Zxid 结构，Zab 协议具体的流程实现，以及 Zab 和原生 Paxos 算法的区别和联系。  

问题：强⼀致性和最终⼀致性的区别，ZooKeeper的⼀致性是怎样的？  
强⼀致性：只要写⼊⼀条数据，⽴⻢⽆论从zk哪台机器上都可以⽴⻢读到这条数据，强⼀致性，你的写⼊  
操作卡住，直到leader和全部follower都进⾏了commit之后，才能让写⼊操作返回，认为写⼊成功了  
此时只要写⼊成功，⽆论你从哪个zk机器查询，都是能查到的，强⼀致性  
ZAB协议机制，zk⼀定不是强⼀致性  
最终⼀致性：写⼊⼀条数据，⽅法返回，告诉你写⼊成功了，此时有可能你⽴⻢去其他zk机器上查是查不  
到的，短暂时间是不⼀致的，但是过⼀会⼉，最终⼀定会让其他机器同步这条数据，最终⼀定是可以查到  
的 研  
究了ZooKeeper的ZAB协议之后，你会发现，其实过半follower对事务proposal返回ack，就会发送  
commit给所有follower了，只要follower或者leader进⾏了commit，这个数据就会被客户端读取到了

18  
那么有没有可能，此时有的follower已经commit了，但是有的follower还没有commit？绝对会的，所以  
有可能其实某个客户端连接到follower01，可以读取到刚commit的数据，但是有的客户端连接到  
follower02在这个时间还没法读取到  
所以zk不是强⼀致的，不是说leader必须保证⼀条数据被全部follower都commit了才会让你读取到数据，  
⽽是过程中可能你会在不同的follower上读取到不⼀致的数据，但是最终⼀定会全部commit后⼀致，让你  
读到⼀致的数据的  
zk官⽅给⾃⼰的定义：顺序⼀致性  
因此zk是最终⼀致性的，但是其实他⽐最终⼀致性更好⼀点，出去要说是顺序⼀致性的，因为leader⼀定  
会保证所有的proposal同步到follower上都是按照顺序来⾛的，起码顺序不会乱  
但是全部follower的数据⼀致确实是最终才能实现⼀致的  
如果要求强⼀致性，可以⼿动调⽤zk的sync()操作  
问题：⽺群效应是什么，如何解决的？  
zk在共享锁的获取和释放流程图：[http://note.youdao.com/s/aKrNn9K9](http://note.youdao.com/s/aKrNn9K9)  
在整个分布式锁的竞争过程中，⼤量的“Watcher通知”和“⼦节点列表获取”两个操作重复运⾏，  
并且绝⼤多数的运⾏结果都是判断出⾃⼰并⾮是序号最⼩的节点，从⽽继续等待下⼀次通知，  
这看起来显然不怎么科学。客户端⽆端地接收到过多和⾃⼰不相关的事件通知，如果在集群规模⽐较⼤的  
情况下，不仅会对ZooKeeper服务器造成巨⼤的性能影响和⽹络冲击，  
更为严重的是，如果同⼀时间有多个节点对应的客户端完成事务或是事务中断引起节点消失，ZooKeeper  
服务器就会在短时间内向其余客户端发送⼤量的事件通知，这就是⽺群效应。  
这个ZooKeeper分布式锁实现中出现⽺群效应的根源在于，没有找准客户端真正的关注点。我们再来回顾  
⼀下上⾯分布式锁的竞争过程。
 
它的核⼼逻辑在于：判断⾃⼰是否是所有⼦节点中序号最⼩的。于是很容易可以联想到，每个节点对应的  
客户端只需要关注⽐⾃⼰序号⼩的那个相关节点的变更情况就可以了，⽽不需要关注全局的⼦列表变更情  
况。  
改进过的zk在共享锁的获取和释放流程图：[http://note.youdao.com/s/YofrjZGB](http://note.youdao.com/s/YofrjZGB)  

## 6.给你个sql,你说一下怎么创建索引？索引结构？联合索引为什么用要存储主键id，存主键id的内存地址不行吗？索引计划有哪些指标，你关注哪些指标？你了解mysql哪些锁？都是干嘛的？什么操作会有锁？需要注意什么？

根据具体的sql来创建索引,联合索引最左原则(字段不超过5个),索引个数5个以内 ; 
索引结构:b+树叶子节点存储数据页 ; 如果存储地址值需要二次寻址 ;
执行计划指标:select_type type 可能使用的索引 优化器实际使用索引等,主要关注连接类型type,索引使用情况和extra中内容 ; 
mysql锁:全局锁 表锁(显示锁和元数据锁MDL) 临界锁(间隙锁+行锁),锁注意事项主要还是死锁
学会通过show engine innodb status来分析死锁案例

## 7.redis你了解哪些？你工作什么场景用？底层数据结构有哪些？分布式锁怎么用？有哪些问题？你们怎么解决的？数据一致性你们怎么做的？分布式事务了解不？你们用过没？你们怎么用的？用了之后有啥问题？你们怎么解决的？redis如何可以动态扩容和缩容？如果你来设计，你怎么设计？

redis 数据结构类型 
内存淘汰策略 过期策略 
持久化rdb和aof  
主从集群(哨兵高可用) 复制和redisCluster这些建议大家看一下《redis设计与实现》 ;
分布式锁:原生set nx ex+lua(get+del)或redisson hash结构支持可重入 或redLock或优化版本加上栅栏验证token ;数据库和缓存一致性推荐大家看一下老师早期出的亿级流量;分布式事务这里就直接引用老师gitee上的文章了

我现在用的居然是单节点。
## 10.xx官问，2020年6月18号，晚上18:00点，浏览器输入了jd.com，你把整个过程给我画个图，越详细越好

首先是cdn解析域名获取ip地址，然后通过ip地址是不是同一个网段来转发，因为是外部的网段，所以发到默认网关，然后默认网关就会把这个请求层层转发出去，在这个过程中，mac地址一直通过arp协议在变化，一直到了目标机器，就会分析tcp协议，接受到syn请求和序列号，发ack按照相同路径回去，后面就是三次握手了，然后接受从类似nginx的地方接受静态资源，以及发起http接口请求来获取数据，数据会通过redis,本地缓存，db来获取。

## 12.设计一个通用限流器,10、100、1000、10w、1000w的qps可以随意配置，误差率在99%
使用golang的rate模块

## 15.MQ延迟队列你知道多少？你能设计一下吗？你能设计一个支持事务的MQ的功能吗？
kafka的延迟的话就自己做几个主题，比如5s，10s,15s,20s的主题，每个消息向上取整去发送到对应的延时主题，然后启动一个项目定时拉取主题消息发到真实主题。
事务：
生产者必须提供唯一的transactionalId，启动后请求事务协调器获取一个PID，transactionalId与PID一一对应。

每次发送数据给<Topic, Partition>前，需要先向事务协调器发送AddPartitionsToTxnRequest，事务协调器会将该<Transaction, Topic, Partition>存于__transaction_state内，并将其状态置为BEGIN。

在处理完 AddOffsetsToTxnRequest 之后，生产者还会发送 TxnOffsetCommitRequest 请求给 GroupCoordinator，从而将本次事务中包含的消费位移信息 offsets 存储到主题 __consumer_offsets 中

一旦上述数据写入操作完成，应用程序必须调用KafkaProducer的commitTransaction方法或者abortTransaction方法以结束当前事务。无论调用 commitTransaction() 方法还是 abortTransaction() 方法，生产者都会向 TransactionCoordinator 发送 EndTxnRequest 请求。  
TransactionCoordinator 在收到 EndTxnRequest 请求后会执行如下操作：

1. 将 PREPARE_COMMIT 或 PREPARE_ABORT 消息写入主题 __transaction_state
2. 通过 WriteTxnMarkersRequest 请求将 COMMIT 或 ABORT 信息写入用户所使用的普通主题和 __consumer_offsets
3. 将 COMPLETE_COMMIT 或 COMPLETE_ABORT 信息写入内部主题 __transaction_state标明该事务结束

在消费端有一个参数isolation.level，设置为“read_committed”，表示消费端应用不可以看到尚未提交的事务内的消息。如果生产者开启事务并向某个分区值发送3条消息 msg1、msg2 和 msg3，在执行 commitTransaction() 或 abortTransaction() 方法前，设置为“read_committed”的消费端应用是消费不到这些消息的，不过在 KafkaConsumer 内部会缓存这些消息，直到生产者执行 commitTransaction() 方法之后它才能将这些消息推送给消费端应用。反之，如果生产者执行了 abortTransaction() 方法，那么 KafkaConsumer 会将这些缓存的消息丢弃而不推送给消费端应用。

## 2.mysql binlog的用途，binlog的结构，聚集索引与非聚集索引的区别，聚集索引叶子节点的数据结构，mysql主从是推模型还是拉模型，为什么。
Binlog是**用来记录Mysql内部对数据库的改动（只记录对数据的修改操作），主要用于数据库的主从复制以及增量恢复**。
### Binlog结构和内容

日志由一组二进制日志文件（Binlog）,加上一个索引文件（index）；Binlog是一个二进制文件集合，每个Binlog以一个4字节的魔数开头，接着是一组Events；

1.魔数：0xfe62696e对应的是0xfebin；

2.Event：每个Event包含header和data两个部分；header提供了Event的创建时间，哪个服务器等信息，data部分提供的是针对该Event的具体信息，如具体数据的修改；

3.第一个Event用于描述binlog文件的格式版本，这个格式就是event写入binlog文件的格式；

4.其余的Event按照第一个Event的格式版本写入；

5.最后一个Event用于说明下一个binlog文件；

6.Binlog的索引文件是一个文本文件，其中内容为当前的binlog文件列表


InnoDB 存储引擎支持以下几种常见的索引： B+树索引、全文索引、哈希索引 而 B+树索引最为常见，可以分为聚集索引和非聚集索引。非聚集索引也可以叫做辅助索引，二级索引。

两种索引相同点： 内部都是 B+ 树，高度平衡，叶子节点存放着所有的数据。

不同点： 聚集索引的叶子节点存放是一整行的信息。 聚集索引一个表只能有一个，而非聚集索引一个表可以存在多个。 聚集索引存储记录是物理上连续存在，而非聚集索引是逻辑上的连续，物理存储并不连续。 聚集索引查询数据速度快，插入数据速度慢;非聚集索引反之。 聚集索引范围查询快。

聚集索引： InnoDB 存储引擎表是索引组织表，表种数据按照主键顺序存放，而聚集索引就是按照每张表的主键构造一颗 B+ 数，同时叶子节点中存放的就是整张表的行记录数据，也将聚集索引的叶子节点称为数据页。 每张表只能拥有一个聚集索引。 查询优化器倾向于采用聚集索引。

非聚集索引： 叶子节点不包含记录的全部数据。 叶子节点中索引行中还包含了一个书签，用来告诉 InnoDB 存储引擎在哪里可以找到与索引相应的行数据。 这个书签就是相应的行数据的聚集索引键。 可以有多个非聚集索引。 使用非聚集索引来寻找数据时，通过叶级别的指针获得指向主键索引的主键，再通过主键索引找到一个完整的行记录。

### **MySQL支持三种复制方式:**

MySQL 5.6、5.7、8.0 版本支持三种复制方式：异步、半同步、强同步；5.5 版本支持异步方式

#### 1) **异步复制(MySQL默认) : AP模型**

Master不等待Slave同步，直接返回client => **性能最高，数据可能出现不一致；可用性优先: 适合对性能要求，能够容忍计算场景少量数据丢失场景**

应用发起数据更新（含 操作）请求，insert、update、deleteMaster 在执行完更新操作后立即向应用程序返回响应，然后 Master 再向 Slave 复制数据。

数据更新过程中 Master 不需要等待 Slave 的响应，因此异步复制的数据库实例通常具有较高的性能，且 Slave 不可用并不影响 Master 对外提供服务。但因数据并非实时同步到 Slave，而 Master 在 Slave 有延迟的情况下发生故障则有较小概率会引起数据不一致

#### **2)半同步复制: AP模型**

Master等待Slave写入relaylog返回client & Slave宕机或网络中断，Master暂停10s 降级 异步复制，Slave恢复后 恢复半同步复制 => **性能居中，可用性优先，极端场景少量不一致；**

应用发起数据更新（含 insert、update、delete 操作）请求，Master 在执行完更新操作后立即向 Slave 复制数据，Slave 接收到数据并写到 relay log 中（无需回放执行）后才向 Master 返回成功信息，Master 必须在接受到 Slave 的成功信息后再向应用程序返回响应。

仅在数据复制发生异常（Slave 节点不可用或者数据复制所用网络发生异常）的情况下，Master 会暂停（MySQL 默认10秒左右）对应用的响应，将复制方式降为异步复制。当数据复制恢复正常，将恢复为半同步复制。

#### **3) 强同步复制: CP模型**

Master等待Slave写入relaylog返回client; Slave宕机或网络中断，Master不会降级为 异步复制 => 保证强一致性，暂停对应用响应，直到Slave恢复正常 => **性能最差，强一致性 =>强一致性: 牺牲可用性，适合对一致性要求高，能够接收服务停机或暂时不可用，保证数据强一致性业务场景。**

应用发起数据更新（含 insert、update、delete 操作）请求，Master 在执行完更新操作后立即向 Slave 复制数据，Slave 接收到数据并写到 relay log 中（无需执行） 后才向 Master 返回成功信息，Master 必须在接受到 Slave 的成功信息后再向应用程序返回响应。

在数据复制发生异常（Slave 节点不可用或者数据复制所用网络发生异常）的情况下，复制方式均不会发生降级，为保障数据一致性，此时 Master 会暂停对应用的响应，直至异常结束。

### **4.MySQL主从复制原理：**

1. Master主库将数据变更DataChanges记录 binlog日志中。
2. Slave起一个I/O线程连接到Master，dump读取Master的binlog日志并写入到Slave的中继日志Relaylog中。
3. Slave中的SQL线程读取中继日志Relaylog进行SQL回放执行操作，完成主从复制，保证主从最终一致性。
所以应该是拉模型


## 3.mysql 组合索引的使用规则
组合索引就是先排第一个key，再排第二个key，所以应该先用上第一个key，才能用上第二个key

## golang的内存模型
为了避免内存重排序和编译重排序，加入了写屏障，并且符合了不严格的顺序一致性模型，允许一定范围内的指令重排序

## 4.redis跳跃表结构，跳跃表中层级的生成规则
![[Pasted image 20230606134207.png]]
Redis 的 zset 是一个复合结构，一方面它需要一个 hash 结构来存储 value 和 score 的对应关系，另一方面需要提供按照 score 来排序的功能，还需要能够指定 score 的范围来获取 value 列表的功能，这就需要另外一个结构「跳跃列表」。zset 的内部实现是一个 hash 字典加一个跳跃列表 (skiplist)。hash 结构在讲字典结构时已经详细分析过了，它很类似于 Java 语言中的 HashMap 结构。

对于每一个新插入的节点，都需要调用一个随机算法给它分配一个合理的层数。直观上期望的目标是 50% 的 Level1，25% 的 Level2，12.5% 的 Level3，一直到最顶层`2^-63`，因为这里每一层的晋升概率是 50%。
不过 Redis 标准源码中的晋升概率只有 25%，也就是代码中的 ZSKIPLIST_P 的值。所以官方的跳跃列表更加的扁平化，层高相对较低，在单个层上需要遍历的节点数量会稍多一点。

也正是因为层数一般不高，所以遍历的时候从顶层开始往下遍历会非常浪费。跳跃列表会记录一下当前的最高层数`maxLevel`，遍历时从这个 maxLevel 开始遍历性能就会提高很多。

## interface 最著名的坑的，应该就是下面这个了。

```go
func main() {
    var a interface{} = nil
    var b *int = nil
    
    isNil(a)
    isNil(b)
}

func isNil(x interface{}) {
    if x == nil {
      fmt.Println("empty interface")
      return
    }
    fmt.Println("non-empty interface")
}
复制代码
```

output:

```
empty interface
non-empty interface
复制代码
```

为什么会这样呢？这就涉及到 interface == nil 的判断方式了。一般情况只有 eface 的 type 和 data 都为 nil 时，interface == nil 才是 true。

当我们把 b 复制给 interface 时，x.\_type.Kind = kindPtr。虽说 x.data = nil，但是不符合 interface == nil 的判断条件了。

## mq如何保证消息不丢失
### Producer端解决方案

在剖析Producer端丢失场景的时候， 我们得出其是通过「**异步**」方式进行发送的，所以如果此时是使用「**发后即焚**」的方式发送，即调用 Producer.send(msg) 会立即返回，由于没有回调，可能因网络原因导致 Broker 并没有收到消息，此时就丢失了。

因此我们可以从以下几方面进行解决 Producer 端消息丢失问题：

### **更换调用方式：**

不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。这样一旦发现发送失败， 就可以做针对性处理。 解决 （1）网络抖动导致消息丢失，Producer 端可以进行重试。 （2）消息大小不合格，可以进行适当调整，符合 Broker 承受范围再发送。

最大限度的保证消息发送成功

### **ack确认机制**

该参数代表了对 **"已提交"** 消息的定义。

需要将 **request.required.acks设置为 -1/ all**，-1/all 表示有多少个副本 Broker 全部收到消息，才认为是消息提交成功的标识。

分两种情况

**1）数据发送到 Leader Partition， 且所有的 ISR 成员全部同步完数据， 此时，LeaderPartition异常 Crash 掉，那么会选举新的 Leader Partition，数据不会丢失**

**2）数据发送到Leader Partition**，部分 ISR 成员同步完成，此时 Leader Partition 异常 Crash，剩下的 Follower Partition 都可能被选举成新的 Leader Partition，会给 Producer 端发送失败标识， 后续会重新发送数据，数据可能会重复。

但是在此操作中需要保证 **replication.factor >= 2** **min.insync.replicas> 1**

#### ****重试次数 retries：****

该参数表示 Producer 端发送消息的重试次数。

需要将 retries设置为大于0的数， 在 Kafka 2.4 版本中默认设置为Integer.MAX_VALUE。另外如果需要保证发送消息的顺序性，配置如下：

`retries = Integer.MAX_VALUE max.in.flight.requests.per.connection = 1`

这样 Producer 端就会一直进行重试直到 Broker 端返回 ACK 标识，同时只有一个连接向 Broker 发送数据保证了消息的顺序性。

#### ******重试时间 retry.backoff.ms：******

该参数表示消息发送超时后**两次重试之间的间隔时间**，避免无效的频繁重试，默认值为100ms, **推荐设置为300ms**。

### Broker 端解决方案

### **unclean.leader.election.enable** **：**

该参数表示**有哪些 Follower 可以有资格被选举为 Leader** , 如果一个 Follower 的数据落后 Leader 太多，那么一旦它被选举为新的 Leader， 数据就会丢失，因此我们要将其设置为false，防止此类情况发生。

### ****replication.factor**** ****：****

该参数表示分区副本的个数。建议设置****replication.factor >=3****, 这样如果 Leader 副本异常 Crash 掉，Follower 副本会被选举为新的 Leader 副本继续提供服务。

### ******min.insync.replicas****** ******：******

该参数表示消息至少要被写入成功到 ISR 多少个副本才算 **"已提交"，** 建议设置**min.insync.replicas > 1,** 这样才可以提升消息持久性，保证数据不丢失。

另外我们还需要确保一下**replication.factor > min.insync.replicas**, 如果相等，只要有一个副本异常 Crash 掉，整个分区就无法正常工作了，因此推荐设置成：**replication.factor =min.insync.replicas +1**, 最大限度保证系统可用性。

### Consumer 端解决方案

在剖析 Consumer 端丢失场景的时候，我们得出其拉取完消息后是需要提交 Offset 位移信息的，因此为了不丢数据，正确的做法是：**拉取数据、** **业务逻辑处理、** **提交消费 Offset 位移信息。**

我们还需要设置参数**enable.auto.commit = false, 采用手动提交位移的方式。**

另外对于消费消息重复的情况，业务自己保证幂等性, **保证只成功消费一次即可**。

## 总结

针对上面分析，我们做一下kafka无丢失配置。

1.不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。

2.设置 acks = all。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。

3.设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。

4.设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。

5.设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。

6.设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。

7.确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。

8.确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。

## 解答

1. min.insync.replicas只有在ack=-1时才生效

2.关于 第二条的ack=all与第六条的min.insync.replicas的理解，acks=all表示消息要写入所有ISR副本，但没要求ISR副本有多少个。但是min.insync.replicas要求的是最少写入的副本数，也就是写入消息的下限。

举例：min.insync.replicas=2， replication.factor =3，正常情况下，如果3个副本都在ISR中，那么他们必须都同步才行。如果挂掉一个，此时min.insync.replicas = replication.factor，此时说明消息要全部写入副本才算成功，如果在挂掉一个，就不能工作了，因为，副本数就一个，producer的消息必须阻塞等broker的isr数量达到min.insync.replicas才提交成功，显然成功不了，最终producer请求会超时。

简单点来说，Producer端认为消息已经成功提交的条件是：ISR中所有副本都已经保存了该消息，但producer并没有指定ISR中需要几个副本。这就是min.insync.replicas参数的作用。


## 4. MQ保证消息不被重复消费  
消费者消费完没有提交offset，宕机后再消费，或者生产者发后ack没有到全部就broker down了然后又发了一次。
需要根据不同场景来做，尽量做到幂等无所谓的那种。

## 6. MQ如何保证消息的顺序性  
使用一个分片
或者指定分区发送
或者指定消息键，这样就会去hash然后%分片了
kafka应该可以指定消息键要发送路由的规则，可以实现某个接口
## 8. MQ如何处理消息丢失的问题
#### 2.1 生产者导致消息丢失

对于 Kafka 来说，生产者基本不会弄丢消息，因为生产者发送消息会等待 Kafka 响应成功，如果响应失败，生产者会自动不断地重试。

#### 2.2 Kafka 弄丢了数据

Kafka 通常会一台 leader + 两台 follower，当生产者消息刚写入 leader 成功，但是还没同步到 follower 时，leader 宕机了，此时会重新选举 leader，新的 leader 由于还未同步到这条数据，导致该条消息丢失。

  

解决办法是做一些配置，当有其他 follower 同步到了消息后才通知生产者消息接收成功了。配置如下：

- 给 topic 设置 `replication.factor` 参数：这个值必须大于 1，要求每个 partition 必须有至少 2 个副本。
    
- 在 Kafka 服务端设置 `min.insync.replicas` 参数：这个值必须大于 1，这个是要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队，这样才能确保 leader 挂了还有一个 follower。
    
- 在 producer 端设置 `acks=all` ：这个是要求每条数据，必须是**写入所有 replica 之后，才能认为是写成功了**。
    

按上面的配置配置后，就可以保证在 Kafka Broker 端就算 leader 故障了，进行新 leader 选举切换时，也不会丢失数据。

#### 2.3 消费者导致消息丢失

Kafka 消费端弄丢数据原因跟 RabbitMQ 类似，Kafka 消费者会在接收到消息的时候，会自动提交一个 offset 给 Kafka，告诉 Kafka 消息已经处理了。处理方法也跟 RabbitMQ 类似，关闭 offset 的自动提交即可。

## 10. MQ消息持续积压⼏⼩时怎么处理  
要先判断是发送快了，还是消费慢了
发送快的话就减少发送
消费忙了的话，就需要平时做好重load逻辑，快速将消息丢弃
或者加大pod的数量多去消费

或者将现有consumer停掉，然后新建一个topic，分片是原来的十倍，将推挤的数据发到新topic上，然后启动十倍的pod去消费。

## 1. Redis线程模型  
 io多路复用程序，事件队列，文件时间分配器，命令请求处理器，连接应答处理器，命令回复处理器
 其他线程的话就是用于边路逻辑，比如aof，rdb，删除数据，或者tcp数据的解析
 
## 6. ⾼并发场景下的缓存⼀致性问题
binlog处理，缓存双删，缓存缺失的时候就加分布式锁去load，其他人等待一会再看看数据是否存在

## 7. CPU多级缓存模型 ,总线加锁机制和MESI缓存⼀致性协议  
![](https://pic4.zhimg.com/80/v2-d7d831f9b4fb63707c456fe72b4f7f33_720w.webp)
![](https://pic1.zhimg.com/80/v2-e69746e86cbc9b37f2ffe3988c680730_720w.webp)


##  https

非对称加密 - ca证书验证 - 对称加密传输

 ## 为什么不直接使用非对称加密传输

非对称加密慢,消耗资源

## TCP四次挥手,各个阶段的状态

重点说了time_wait和last_ack , 以及各个阶段没收到数据包会执行什么操作
![](https://pic3.zhimg.com/80/v2-d33f16171fe5ba49c8004fcd59d47e7e_720w.webp)

**1．为什么建立连接协议是三次握手，而关闭连接却是四次握手呢？**

> 这是因为服务端的LISTEN状态下的SOCKET当收到SYN报文的建连请求后，它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。但关闭连接时，当收到对方的FIN报文通知时，它仅仅表示对方没有数据发送给你了；但未必你所有的数据都全部发送给对方了，所以你可以未必会马上会关闭SOCKET,也即你可能还需要发送一些数据给对方之后，再发送FIN报文给对方来表示你同意现在可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

![](https://pic2.zhimg.com/80/v2-f42a8b3b88e2f4b769519cc4884dd985_720w.webp)

2、什么是2MSL

> MSL是Maximum Segment Lifetime英文的缩写，中文可以译为“报文最大生存时间”，他是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为tcp报文（segment）是ip数据报（datagram）的数据部分，具体称谓请参见《数据在网络各层中的称呼》一文，而ip头中有一个TTL域，TTL是time to live的缩写，中文可以译为“生存时间”，这个生存时间是由源主机设置初始值但不是存的具体时间，而是存储了一个ip数据报可以经过的最大路由数，每经过一个处理他的路由器此值就减1，当此值为0则数据报将被丢弃，同时发送ICMP报文通知源主机。RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。  
> 2MSL即两倍的MSL，TCP的TIME_WAIT状态也称为2MSL等待状态，当TCP的一端发起主动关闭，在发出最后一个ACK包后，即第3次握手完成后发送了第四次握手的ACK包后就进入了TIME_WAIT状态，必须在此状态上停留两倍的MSL时间，等待2MSL时间主要目的是怕最后一个ACK包对方没收到，那么对方在超时后将重发第三次握手的FIN包，主动关闭端接到重发的FIN包后可以再发一个ACK应答包。在TIME_WAIT状态时两端的端口不能使用，要等到2MSL时间结束才可继续使用。当连接处于2MSL等待阶段时任何迟到的报文段都将被丢弃。不过在实际应用中可以通过设置SO_REUSEADDR选项达到不必等待2MSL时间结束再使用此端口。  
> 这是因为虽然双方都同意关闭连接了，而且握手的4个报文也都协调和发送完毕，按理可以直接回到CLOSED状态（就好比从SYN_SEND状态到ESTABLISH状态那样）：  
> 一方面是可靠的实现TCP全双工连接的终止，也就是当最后的ACK丢失后，被动关闭端会重发FIN，因此主动关闭端需要维持状态信息，以允许它重新发送最终的ACK。  
> 另一方面，但是因为我们必须要假想网络是不可靠的，你无法保证你最后发送的ACK报文会一定被对方收到，因此对方处于LAST_ACK状态下的SOCKET可能会因为超时未收到ACK报文，而重发FIN报文，所以这个TIME_WAIT状态的作用就是用来重发可能丢失的ACK报文。  
> TCP在2MSL等待期间，定义这个连接(4元组)不能再使用，任何迟到的报文都会丢弃。设想如果没有2MSL的限制，恰好新到的连接正好满足原先的4元组，这时候连接就可能接收到网络上的延迟报文就可能干扰最新建立的连接。

**3．为什么TIME_WAIT状态还需要等2MSL后才能返回到CLOSED状态？**

> **第一，**保证可靠的实现TCP全双工链接的终止：为了保证主动端A发送的最后一个ACK报文能够到达被动段B，确保被动端能进入CLOSED状态。  
> 对照上图，在四次挥手协议中，当B向A发送Fin+Ack后，A就需要向B发送ACK+Seq报文，A这时候就处于TIME_WAIT 状态，但是这个报文ACK有可能会发送失败而丢失，因而使处在LAST-ACK状态的B收不到对已发送的FIN+ACK报文段的确认。B会超时重传这个FIN+ACK报文段，而A就能在2MSL时间内收到这个重传的FIN+ACK报文段。如果A在TIME-WAIT状态不等待一段时间，而是在发送完ACK报文段后就立即释放连接，就无法收到B重传的FIN+ACK报文段，因而也不会再发送一次确认报文段。这样B就无法按照正常的步骤进入CLOSED状态。  
>   
> **第二，**防止已失效的连接请求报文段出现在本连接中。**A在发送完ACK报文段后，再经过2MSL时间**，就可以使本连接持续的时间所产生的所有报文段都从网络中消失。这样就可以使下一个新的连接中不会出现这种旧的连接请求的报文段。  
> 假设在A的XXX1端口和B的80端口之间有一个TCP连接。我们关闭这个连接，过一段时间后在相同的IP地址和端口建立另一个连接。后一个链接成为前一个的化身。因为它们的IP地址和端口号都相同。TCP必须防止来自某一个连接的老的重复分组在连 接已经终止后再现，从而被误解成属于同一链接的某一个某一个新的化身。为做到这一点，TCP将不给处于TIME_WAIT状态的链接发起新的化身。既然 TIME_WAIT状态的持续时间是MSL的2倍，这就足以让某个方向上的分组最多存活msl秒即被丢弃，另一个方向上的应答最多存活msl秒也被丢弃。 通过实施这个规则，我们就能保证每成功建立一个TCP连接时。来自该链接先前化身的重复分组都已经在网络中消逝了。  

4、解决linux发现系统存在大量TIME_WAIT状态

> 如果linux发现系统存在大量TIME_WAIT状态的连接，可以通过调整内核参数解决：vi /etc/sysctl.conf 加入以下内容：  
> net.ipv4.tcp_max_tw_buckets=5000 \#TIME-WAIT Socket 最大数量  
> \#注意：不建议开启該设置，NAT 模式下可能引起连接 RST  
> net.ipv4.tcp_tw_reuse = 1 \#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；  
> net.ipv4.tcp_tw_recycle = 1 \#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。  
> 然后执行 /sbin/sysctl -p 让参数生效。

## fmt print的实现：用到了sync pool
![[7a09915ac0c9297da53e9048d97879a.png]]

## 什么时候goalng会stw
![[ab95e1e954f20798f9933e36c6e23e7.png]]

## socket建立后，客户端网线被拔了，服务端会发生什么？
**拔掉网线这个动作并不会影响 TCP 连接的状态。**  
  
不过，这个答案还是有点笼统。实际上，我们应该在更具体的场景中来看待这个问题，答案才更准确一些。  
  
**这个具体场景就是：**  
- _**1）**_当拔掉网线后，有数据传输时；
- _**2）**_当拔掉网线后，没有数据传输时。

### 4、具体场景1：拔掉网线后，有数据传输时
#### 4.1数据传输过程中，恰好又把网线插回去了
如果是客户端被拔掉网线后，服务端向客户端发送的数据报文会得不到任何的响应，在等待一定时长后，服务端就会触发TCP协议的超时重传机制（详见：《[TCP/IP详解](http://www.52im.net/topic-tcpipvol1.html) - [第21章·TCP的超时与重传](http://docs.52im.net/extend/docs/book/tcpip/vol1/21/)》），然而此时重传并不能得到响应的数据报文。  
  
如果在服务端重传报文的过程中，客户端恰好把网线插回去了，由于拔掉网线并不会改变客户端的 TCP 连接状态，并且还是处于 ESTABLISHED 状态，所以这时客户端是可以正常接收服务端发来的数据报文的，然后客户端就会回 ACK 响应报文。  
  
**此时：**客户端和服务端的 TCP 连接将依然存在且工作状态不会受到影响，给应用层的感觉就像什么事情都没有发生。。。  
#### 4.2数据传输过程中，网线一直没有插回去  
上面这种情况下，如果在服务端TCP协议重传报文的过程中，客户端一直没有将网线插回去，那么服务端超时重传报文的次数达到一定阈值后，内核就会判定出该 TCP 有问题。然后就会通过 Socket 接口告诉应用程序该 TCP 连接出问题了，于是服务端的 TCP 连接就会断开。  
  
接下来，如果客户端再插回网线，如果客户端向服务端发送了数据，由于服务端已经没有与客户端匹配的 TCP 连接信息了，因此服务端内核就会回复 RST 报文，客户端收到后就会释放该 TCP 连接。  
  
**此时：**客户端和服务端的 TCP 连接已经明确被断开，原本的这个连接也就不存在了。

本着知其然更应知其所以然的精神，我们来刨根问底一下：**TCP 的数据报文到底有重传几次呢？**  
  
在 Linux 系统中，提供了一个叫 _tcp_retries2_ 配置项，默认值是 15（如下图所示）。  
  
![不为人知的网络编程(十四)：拔掉网线再插上，TCP连接还在吗？一文即懂！_2.png](http://www.52im.net/data/attachment/forum/202203/07/094733b7nr7hy1hgmj0ytx.png "不为人知的网络编程(十四)：拔掉网线再插上，TCP连接还在吗？一文即懂！_2.png")  
  
**如上图所示：**这个内核参数是控制 TCP 连接建立的情况下，超时重传的最大次数。  
  
不过 _tcp_retries2_ 设置了 15 次，并不代表 TCP 超时重传了 15 次才会通知应用程序终止该 TCP 连接，内核还会基于“最大超时时间”来判定。  
  
每一轮的超时时间都是倍数增长的，比如第一次触发超时重传是在 2s 后，第二次则是在 4s 后，第三次则是 8s 后，以此类推。

### 5、具体场景2：拔掉网线后，没有数据传输时
#### 5.1场景分析

  
针对拔掉网线后，没有数据传输的场景，还得具体看看是否开启了 TCP KeepAlive 机制 （详见《[彻底搞懂TCP协议层的KeepAlive保活机制](http://www.52im.net/thread-3506-1-1.html)》）。  
  
**_1）如果没有开启 TCP KeepAlive 机制：_**  
  
在客户端拔掉网线后，并且双方都没有进行数据传输，那么客户端和服务端的 TCP 连接将会一直保持存在。  
  
**_2）如果开启了 TCP KeepAlive 机制：_**  
  
在客户端拔掉网线后，即使双方都没有进行数据传输，在持续一段时间后，TCP 就会发送KeepAlive探测报文。  
  
**根据KeepAlive探测报文响应情况，会有以下两种可能：**  
  

- _**1）**_如果对端正常工作：当探测报文被对端收到并正常响应， TCP 保活时间将被重置，等待下一个 TCP 保活时间的到来；
- _**2）**_如果对端主机崩溃或对端由于其他原因导致报文不可达：当探测报文发送给对端后，石沉大海、没有响应，连续几次，达到保活探测次数后，TCP 会报告该连接已经死亡。  
    
**所以：**TCP 保活机制可以在双方没有数据交互的情况，通过TCP KeepAlive 机制的探测报文，来确定对方的 TCP 连接是否存活。

#### 5.2刨根问底：TCP KeepAlive 机制具体是什么样的？

  
**TCP KeepAlive 机制的原理是这样的：**  

定义一个时间段，在这个时间段内，如果没有任何连接相关的活动，TCP 保活机制会开始作用，每隔一个时间间隔，发送一个探测报文。该探测报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的 TCP 连接已经死亡，系统内核将错误信息通知给上层应用程序。

  
在 Linux 内核可以有对应的参数可以设置保活时间、保活探测的次数、保活探测的时间间隔。  
  
**以下是 Linux 中的默认值：**  

|   |   |
|---|---|
|1<br><br>2<br><br>3|`net.ipv4.tcp_keepalive_time=7200`<br><br>`net.ipv4.tcp_keepalive_intvl=75` <br><br>`net.ipv4.tcp_keepalive_probes=9`|

  
**解释一下：**  
  
_**1）**_tcp_keepalive_time=7200：表示保活时间是 7200 秒（2小时），也就 2 小时内如果没有任何连接相关的活动，则会启动保活机制；  
_**2）**_tcp_keepalive_intvl=75：表示每次检测间隔 75 秒；  
_**3）**_tcp_keepalive_probes=9：表示检测 9 次无响应，认为对方是不可达的，从而中断本次的连接。  
也就是说在 Linux 系统中，最少需要经过 **2 小时 11 分 15 秒**才可以发现一个“死亡”连接。

### 6、本文小结

  
下面简单总结一下文中的内容，本文开头的问题并不是简单一句话能够准确说清楚的，需要分情况对待。  
  
**也就是：**客户端拔掉网线后，并不会直接影响 TCP 的连接状态。所以拔掉网线后，TCP 连接是否还会存在，关键要看拔掉网线之后，有没有进行数据传输。  
  
**_1）有数据传输的情况：_**  
  

- _**a.**_ 在客户端拔掉网线后：如果服务端发送了数据报文，那么在服务端重传次数没有达到最大值之前，客户端恰好插回网线的话，那么双方原本的 TCP 连接还是能存在并正常工作，就好像什么事情都没有发生;
- _**a.**_ 在客户端拔掉网线后：如果服务端发送了数据报文，在客户端插回网线之前，服务端重传次数达到了最大值时，服务端就会断开 TCP 连接。等到客户端插回网线后，向服务端发送了数据，因为服务端已经断开了与客户端相同四元组的 TCP 连接，所以就会回 RST 报文，客户端收到后就会断开 TCP 连接。至此， 双方的 TCP 连接都断开了。  
    

  
_**2）没有数据传输的情况：**_  
  

- _**a.**_ 如果双方都没有开启 TCP keepalive 机制：那么在客户端拔掉网线后，如果客户端一直不插回网线，那么客户端和服务端的 TCP 连接状态将会一直保持存在；
- _**b.**_ 如果双方都开启了 TCP keepalive 机制：那么在客户端拔掉网线后，如果客户端一直不插回网线，TCP keepalive 机制会探测到对方的 TCP 连接没有存活，于是就会断开 TCP 连接。而如果在 TCP 探测期间，客户端插回了网线，那么双方原本的 TCP 连接还是能正常存在。  
    

  
**除了客户端拔掉网线的场景，还有客户端“宕机和杀死进程”的两种场景。**  
  
**第一个场景：**客户端宕机这件事跟拔掉网线是一样无法被服务端的感知的，所以如果在没有数据传输，并且没有开启 TCP keepalive 机制时，，服务端的 TCP 连接将会一直处于 _ESTABLISHED_ 连接状态，直到服务端重启进程。  
  
所以：我们可以得知一个点——即在没有使用 TCP 保活机制且双方不传输数据的情况下，一方的 TCP 连接处在 _ESTABLISHED_ 状态时，并不代表另一方的 TCP 连接还一定是正常的。  
  
**第二个场景：**杀死客户端的进程后，客户端的内核就会向服务端发送 FIN 报文，与客户端进行四次挥手（见《[跟着动画来学TCP三次握手和四次挥手](http://www.52im.net/thread-1729-1-1.html)》）。  
  
所以：即使没有开启 TCP KeepAlive，此时双方也没有数据交互的情况下，如果其中一方的进程发生了崩溃，那么操作系统是可以感知到这个过程的，于是就会发送 FIN 报文给对方，然后与对方正常进行 TCP 四次挥手。

#### 4.2数据传输过程中，网线一直没有插回去

  
上面这种情况下，如果在服务端TCP协议重传报文的过程中，客户端一直没有将网线插回去，那么服务端超时重传报文的次数达到一定阈值后，内核就会判定出该 TCP 有问题。然后就会通过 Socket 接口告诉应用程序该 TCP 连接出问题了，于是服务端的 TCP 连接就会断开。  
  
接下来，如果客户端再插回网线，如果客户端向服务端发送了数据，由于服务端已经没有与客户端匹配的 TCP 连接信息了，因此服务端内核就会回复 RST 报文，客户端收到后就会释放该 TCP 连接。  
  
**此时：**客户端和服务端的 TCP 连接已经明确被断开，原本的这个连接也就不存在了。

##   text多少长度，文本最大长度多少
TINYTEXT，TEXT,MEDIUMTEXT,LONGTEXT
0-255,0-65535,0-16777215,XXXX bytes

char,varchar
0-255,0-65535 bytes

- TEXT文本类型，可以存比较大的文本段，搜索速度稍慢，因此如果不是特别大的内容，建议使用CHAR，VARCHAR来代替。还有TEXT类型不用加默认值，加了也没用。而且text和blob类型的数据删除后容易导致“空洞”，使得文件碎片比较多，所以频繁使用的表不建议包含TEXT类型字段，建议单独分出去，单独用一个表。

## -   索引什么情况会失效
like%,字符集不对，类型不对，使用不等于（!=、<>）使索引失效，使用了函数操作，在查询语句中使用or连接之后，会使索引失效。哪怕是左右两个都是有索引的字段也会使索引失效。当数据库统计这个字段的值不够分散的时候，这个是模糊统计的，有可能真的不够分散，或者说统计错误，就会选择不走索引

-   介绍下MVCC
-   聚簇索引非聚簇索引区别

2.  为什么mysql 500w条 内比较合适
事实上，这个数值和实际记录的条数无关，而与 MySQL 的配置以及机器的硬件有关。因为，MySQL 为了提高性能，会将表的索引装载到内存中。InnoDB buffer size 足够的情况下，其能完成全加载进内存，查询不会有问题。但是，当单表数据库到达某个量级的上限时，导致内存无法存储其索引，使得之后的 SQL 查询会产生磁盘 IO，从而导致性能下降。当然，这个还有具体的表结构的设计有关，最终导致的问题都是内存限制。这里，增加硬件配置，可能会带来立竿见影的性能提升哈
　　1）如果单表容量大（大于2G），但是索引少（只通过主键ID查），性能也不会慢

　　2）如果数据量大（大于500W），但是索引容量小（都是小字节字段），性能也不会慢

　　3）所以，单表查询的性能取决于索引的大小（因为会放内存里），而索引的查询速度又受硬件的影响。

　　4）建议：大表（数据量大、容量大）。先拆成主表（字段多）、detail表（容量高）。主表严格控制索引的质量，detail表只能用主键ID查（后期可以通过主键ID分表）。

## -   每天100w的日志表设计
动态表名，表名+日期


## -   倒排索引是什么 （这是ES里的啊）
![](https://pica.zhimg.com/80/v2-70574e6602e492fcf5bd4c2ae62a67eb_720w.webp?source=1940ef5c)

创建[倒排索引](https://www.zhihu.com/search?q=%E5%80%92%E6%8E%92%E7%B4%A2%E5%BC%95&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A254503794%7D)，分为以下几步：

1）创建文档列表：

l lucene首先对原始文档数据进行编号（DocID），形成列表，就是一个文档列表

![](https://picx.zhimg.com/80/v2-d50ffd4b3bc38e25e281fea9e07e14e6_720w.webp?source=1940ef5c)

2）创建[倒排索引列表](https://www.zhihu.com/search?q=%E5%80%92%E6%8E%92%E7%B4%A2%E5%BC%95%E5%88%97%E8%A1%A8&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A254503794%7D)

l 然后对文档中数据进行分词，得到[词条](https://www.zhihu.com/search?q=%E8%AF%8D%E6%9D%A1&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A254503794%7D)。对词条进行编号，以词条创建索引。然后记录下包含该词条的所有文档编号（及其它信息）。

![](https://picx.zhimg.com/80/v2-0e77a230cbce4cd3c8b8e121bb211518_720w.webp?source=1940ef5c)

谷歌之父--> 谷歌、之父
倒排索引创建索引的流程：
1） 首先把所有的[原始数据](https://www.zhihu.com/search?q=%E5%8E%9F%E5%A7%8B%E6%95%B0%E6%8D%AE&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A254503794%7D)进行[编号](https://www.zhihu.com/search?q=%E7%BC%96%E5%8F%B7&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A254503794%7D)，形成文档列表
2） 把文档数据进行分词，得到很多的词条，以词条为索引。保存包含这些词条的文档的编号信息。

搜索的过程：

当用户输入任意的词条时，首先对用户输入的数据进行分词，得到用户要搜索的所有词条，然后拿着这些词条去倒排索引列表中进行匹配。找到这些词条就能找到包含这些词条的所有文档的编号。

然后根据这些编号去文档列表中找到文档

## 怎么查看redis 大key
### 1、使用命令 --bigkeys

--bigkeys 是 redis 自带的命令，对整个 Key 进行扫描，统计 string，list，set，zset，hash 这几个常见数据类型中每种类型里的最大的 key。
### 2、使用 memory 命令查看 key 的大小（仅支持 Redis 4.0 以后的版本）


## 了解索引下推吗？什么情况下会下推到引擎去处理？
- 索引下推（index condition pushdown ）简称ICP，在Mysql5.6的版本上推出，用于优化查询。
- 在不使用ICP的情况下，在使用非主键索引（又叫普通索引或者二级索引）进行查询时，存储引擎通过索引检索到数据，然后返回给MySQL服务器，服务器然后判断数据是否符合条件 。
- 在使用ICP的情况下，如果存在某些被索引的列的判断条件时，MySQL服务器将这一部分判断条件传递给存储引擎，然后由存储引擎通过判断索引是否符合MySQL服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器 。
- 索引条件下推优化可以减少存储引擎查询基础表的次数，也可以减少MySQL服务器从存储引擎接收数据的次数。

### **开撸**
- 在开始之前先先准备一张用户表(user)，其中主要几个字段有：id、name、age、address。建立联合索引（name，age）。
- 假设有一个需求，要求匹配姓名第一个为陈的所有用户，sql语句如下：

```sql
SELECT * from user where  name like '陈%'
```
- 根据 "最佳左前缀" 的原则，这里使用了联合索引（name，age）进行了查询，性能要比全表扫描肯定要高。
- 问题来了，如果有其他的条件呢？假设又有一个需求，要求匹配姓名第一个字为陈，年龄为20岁的用户，此时的sql语句如下：
```sql
SELECT * from user where  name like '陈%' and age=20
```
- 这条sql语句应该如何执行呢？下面对Mysql5.6之前版本和之后版本进行分析。
### **Mysql5.6之前的版本**

- 5.6之前的版本是没有索引下推这个优化的，因此执行的过程如下图：
![](https://pic3.zhimg.com/80/v2-04b4a496ab53eccc5feba150bf9fb7ea_720w.webp)
- 会忽略age这个字段，直接通过name进行查询，在(name,age)这课树上查找到了两个结果，id分别为2,1，然后拿着取到的id值一次次的回表查询，因此这个过程需要**回表两次**。
### **Mysql5.6及之后版本**

- 5.6版本添加了索引下推这个优化，执行的过程如下图：
![](https://pic1.zhimg.com/80/v2-211aaba883221c81d5d7578783a80764_720w.webp)
- InnoDB并没有忽略age这个字段，而是在索引内部就判断了age是否等于20，对于不等于20的记录直接跳过，因此在(name,age)这棵索引树中只匹配到了一个记录，此时拿着这个id去主键索引树中回表查询全部数据，这个过程只需要回表一次。

### **实践**

- 当然上述的分析只是原理上的，我们可以实战分析一下，因此陈某装了Mysql5.6版本的Mysql，解析了上述的语句，如下图：
![](https://pic4.zhimg.com/80/v2-e92ce90a6c810a238e709760ed67daaf_720w.webp)
- 根据explain解析结果可以看出Extra的值为**Using index condition**，表示已经使用了索引下推。

### **总结**
- 索引下推在**非主键索引**上的优化，可以有效减少回表的次数，大大提升了查询的效率。
- 关闭索引下推可以使用如下命令，配置文件的修改不再讲述了，毕竟这么优秀的功能干嘛关闭呢：

```sql
set optimizer_switch='index_condition_pushdown=off';
```

# WHERE id NOT IN (?, ?, ?) 会走索引吗？
-   还是要看成本；举例：id 字段只包含 3 个值，1、2、3，3 只有几行，而 1、2 各有 100w 行，如果查询条件是 NOT IN (1, 2) 会走索引，如果查询条件是 NOT IN (3) 不会走索引。


## struct的字节对齐
假设一个 struct 包含三个字段，`a int8`、`b int16`、`c int64`，顺序会对 struct 的大小产生影响吗？我们来做一个实验：

```go
type demo1 struct {  
	a int8  
	b int16  
	c int32  
}  
  
type demo2 struct {  
	a int8  
	c int32  
	b int16  
}  
  
func main() {  
	fmt.Println(unsafe.Sizeof(demo1{})) // 8  
	fmt.Println(unsafe.Sizeof(demo2{})) // 12  
}  
```

答案是会产生影响。每个字段按照自身的对齐倍数来确定在内存中的偏移量，字段排列顺序不同，上一个字段因偏移而浪费的大小也不同。

接下来逐个分析，首先是 demo1：

-   a 是第一个字段，默认是已经对齐的，从第 0 个位置开始占据 1 字节。
-   b 是第二个字段，对齐倍数为 2，因此，必须空出 1 个字节，偏移量才是 2 的倍数，从第 2 个位置开始占据 2 字节。
-   c 是第三个字段，对齐倍数为 4，此时，内存已经是对齐的，从第 4 个位置开始占据 4 字节即可。

因此 demo1 的内存占用为 8 字节。

其实是 demo2：

-   a 是第一个字段，默认是已经对齐的，从第 0 个位置开始占据 1 字节。
-   c 是第二个字段，对齐倍数为 4，因此，必须空出 3 个字节，偏移量才是 4 的倍数，从第 4 个位置开始占据 4 字节。
-   b 是第三个字段，对齐倍数为 2，从第 8 个位置开始占据 2 字节。

demo2 的对齐倍数由 c 的对齐倍数决定，也是 4，因此，demo2 的内存占用为 12 字节。

![memory alignment](https://geektutu.com/post/hpg-struct-alignment/memory_alignment_order.png)

因此，在对内存特别敏感的结构体的设计上，我们可以通过调整字段的顺序，减少内存的占用。

空 `struct{}` 大小为 0，作为其他 struct 的字段时，一般不需要内存对齐。但是有一种情况除外：即当 `struct{}` 作为结构体最后一个字段时，需要内存对齐。因为如果有指针指向该字段, 返回的地址将在结构体之外，如果此指针一直存活不释放对应的内存，就会有内存泄露的问题（该内存不因结构体释放而释放）。


## -   Map 的 panic 能被 recover 掉吗？了解 panic 和 recover 的机制吗？
map的并发读写是runtime的panic,`runtime` 中调用 `throw` 函数抛出的异常是无法在业务代码中通过 `recover` 捕获的，这点最为致命。所以，对于并发读写 map 的地方，应该对 map 加锁。

类型断言出错也会panic，但是是普通的panic

## -   Map 怎么知道自己处于竞争状态？是 Go 编码实现的还是底层硬件实现的？
 通过结构体中的标记位实现的，可能是通过 CAS 操作的；
```
if h.flags&hashWriting != 0 {  throw("concurrent map read and map write")}
```

## -   CAS 具体是怎么实现的呢？
现在几乎所有的CPU指令都支持CAS的原子操作，`X86`下对应的是 `CMPXCHG` 汇编指令。有了这个原子操作，我们就可以用其来实现各种[无锁（lock free）](https://blog.haohtml.com/archives/30670)的数据结构。

CAS 在Golang中是以共享内存的方式来实现的一种同步机制，它是一个原子操作，一般格式如下

```go
fun addValue(delta int32){
	for{
		oldValue := atomic.LoadInt32(&addr)		
		if atomic.CompareAndSwapInt32(&addr, oldValue, oldValue+delta){
			break;
		}
	}
}
```

先从一个内存地址 `&addr` 读取出来当前存储的值，假如读取完以后，没有其它线程对此变量 进行修改的话，则下面的 `atomic.CompareAndSwapInt32` 语句会在执行时先再判断一次它的值是否与上次的相等，这时必须是相等的，则直接更新它的值；如果在读取值后，有其它线程对变量值进行了修改，发现值不相等，这时就再重新开始下一轮的判断，直到修改成功为止。

对于 `atomic.CompareAndSwapIntxx()` 之类函数的实现是在 `[src/sync/atomic.asm.s](https://github.com/golang/go/blob/go1.15.6/src/sync/atomic/asm.s#L24-L37)` 文件里声明的，真正对应的汇编文件位于 `[src/runtime/internal/atomic/*.s](https://github.com/golang/go/blob/go1.15.6/src/runtime/internal/atomic)`，如64架构对应的文件为 `[asm_amd64.s](https://github.com/golang/go/blob/go1.15.6/src/runtime/internal/atomic/asm_amd64.s)`。

`atomic.CompareAndSwapInt32` 和 `atomic.CompareAndSwapUint32` 对应的汇编是 `runtime∕internal∕atomic·Cas`;

// bool Cas(int32 \*val, int32 old, int32 new)

// Atomically:

// if(*val == old){

// *val = new;

// return 1;

// } else

// return 0;

TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17

MOVQ ptr+0(FP), BX

MOVL old+8(FP), AX

MOVL new+12(FP), CX

LOCK

CMPXCHGL CX, 0(BX)

SETEQ ret+16(FP)

RET

`atomic.CompareAndSwapInt64` 和 `atomic.CompareAndSwapUint64` 对应的汇编是 `runtime∕internal∕atomic·Cas64`；

// bool runtime∕internal∕atomic·Cas64(uint64 *val, uint64 old, uint64 new)

// Atomically:

// if(*val == *old){

// *val = new;

// return 1;

// } else {

// return 0;

// }

TEXT runtime∕internal∕atomic·Cas64(SB), NOSPLIT, $0-25

MOVQ ptr+0(FP), BX

MOVQ old+8(FP), AX

MOVQ new+16(FP), CX

LOCK

CMPXCHGQ CX, 0(BX)

SETEQ ret+24(FP)

RET

可以看到汇编相关的指令有 `LOCK` 和 `CMPXCHGQ` 及其它几个指令。

这里的LOCK 称为 LOCK指令前缀，是用来设置CPU的 `LOCK#` 信号的（[见这里](https://blog.csdn.net/imred/article/details/51994189)）。（译注：这个信号会使总线锁定，阻止其他处理器接管总线访问内存），直到使用LOCK前缀的指令执行结束，这会使这条指令的执行变为原子操作。在多处理器环境下，设置 `LOCK#` 信号能保证某个处理器对共享内存的独占使用。

## -   有对比过 sync.Map 和加锁的区别吗？
sync.map是读读不互斥的，应该都是读readonly的map，写的话就会控制写到dirty还是readmap要加锁。



## 问一段代码输出结果？

```go
func main() {
	fmt.Println(test1())
	fmt.Println(test2())
	fmt.Println(test3())
	fmt.Println(test4())

	return
}

func test1() (v int) {
	defer fmt.Println(v)
	return v
}

func test2() (v int) {
	defer func() {
		fmt.Println(v)
	}()
	return 3
}

func test3() (v int) {
	defer fmt.Println(v)
	v = 3
	return 4
}

func test4() (v int) {
	defer func(n int) {
		fmt.Println(n)
	}(v)
	return 5
}
0
0
3
3
0
4
0
5

```

-   Map 的查询时间复杂度如何分析？
数据量比较小的情况下是O（1），多的情况下是O（n）
-   极端情况下有很多哈希冲突，Golang 标准库如何去避免最坏的查询时间复杂度？
增量rehash和等量rehash。增量hash让链表长度变短，等量hash让数据均匀。

- 当哈希表中的元素数量过少，而哈希表的容量过大时，会触发收缩操作。此时，哈希表的容量会减半，并且哈希表中所有的元素都会重新哈希到新的槽位中。


-   Golang map Rehash 的策略是怎样的？什么时机会发生 Rehash？
数据大于等于桶的6.5倍，溢出桶大于等于桶 这两者分别触发增量rehash和等量rehash。写入和删除的时候触发渐进式hash。缩容的时候也会rehash
## -   Rehash 具体会影响什么？哈希结果会受到什么影响？
再来看一下扩容具体是怎么做的。由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 Go map 的扩容采取了一种称为“渐进式”地方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。

上面说的 `hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。
有一个特殊情况是：有一种 key，每次对它计算 hash，得到的结果都不一样。这个 key 就是 `math.NaN()` 的结果，它的含义是 `not a number`，类型是 float64。当它作为 map 的 key，在搬迁的时候，会遇到一个问题：再次计算它的哈希值和它当初插入 map 时的计算出来的哈希值不一样！

你可能想到了，这样带来的一个后果是，这个 key 是永远不会被 Get 操作获取的！当我使用 `m[math.NaN()]` 语句的时候，是查不出来结果的。这个 key 只有在遍历整个 map 的时候，才有机会现身。所以，可以向一个 map 插入任意数量的 `math.NaN()` 作为 key。

当搬迁碰到 `math.NaN()` 的 key 时，只通过 tophash 的最低位决定分配到 X part 还是 Y part（如果扩容后是原来 buckets 数量的 2 倍）。如果 tophash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。
## # float 类型可以作为 map 的 key 吗
当用 float64 作为 key 的时候，先要将其转成 unit64 类型，再插入 key 中。
float 型可以作为 key，但是由于精度的问题，会导致一些诡异的问题，慎用之。

## # 可以边遍历边删除吗
map 并不是一个线程安全的数据结构。同时读写一个 map 是未定义的行为，如果被检测到，会直接 panic。

上面说的是发生在多个协程同时读写同一个 map 的情况下。 如果在同一个协程内边遍历边删除，并不会检测到同时读写，理论上是可以这样做的。但是，遍历的结果就可能不会是相同的了，有可能结果遍历结果集中包含了删除的 key，也有可能不包含，这取决于删除 key 的时间：是在遍历到 key 所在的 bucket 时刻前或者后。

一般而言，这可以通过读写锁来解决：`sync.RWMutex`。

读之前调用 `RLock()` 函数，读完之后调用 `RUnlock()` 函数解锁；写之前调用 `Lock()` 函数，写完之后，调用 `Unlock()` 解锁。

另外，`sync.Map` 是线程安全的 map，也可以使用。


## 如何比较两个 map 相等

map 深度相等的条件：

1、都为 nil

2、非空、长度相等，指向同一个 map 实体对象

3、相应的 key 指向的 value “深度”相等

直接将使用 map1 == map2 是错误的。这种写法只能比较 map 是否为 nil。

```go
package main

import "fmt"
func main() {

var m map[string]int

var n map[string]int

fmt.Println(m == nil)

fmt.Println(n == nil)

// 不能通过编译

//fmt.Println(m == n)

}
```

输出结果：

true

true

因此只能是遍历map 的每个元素，比较元素是否都是深度相等。

## 读取的过程
oldbuckets 不为 nil，说明发生了扩容，// 如果不是同 size 扩容（看后面扩容的内容）
// 对应条件 1 的解决方案，// 求出 key 在老的 map 中的 bucket 位置// 如果 oldb 没有搬迁到新的 bucket
// 那就在老的 bucket 中寻找

## map 的删除过程是怎样的

写操作底层的执行函数是 `mapdelete`：

func mapdelete(t \*maptype, h \*hmap, key unsafe.Pointer)

根据 key 类型的不同，删除操作会被优化成更具体的函数：

|   |   |
|---|---|
|key 类型|删除|
|uint32|mapdelete\_fast32(t \_maptype, h_ hmap, key uint32)|
|uint64|mapdelete\_fast64(t \_maptype, h_ hmap, key uint64)|
|string|mapdelete\_faststr(t \_maptype, h_ hmap, ky string)|

当然，我们只关心 `mapdelete` 函数。它首先会检查 h.flags 标志，如果发现写标位是 1，直接 panic，因为这表明有其他协程同时在进行写操作。

计算 key 的哈希，找到落入的 bucket。检查此 map 如果正在扩容的过程中，直接触发一次搬迁操作。

删除操作同样是两层循环，核心还是找到 key 的具体位置。寻找过程都是类似的，在 bucket 中挨个 cell 寻找。

找到对应位置后，对 key 或者 value 进行“清零”操作：

```go
// 对 key 清零
if t.indirectkey {
	*(*unsafe.Pointer)(k) = nil
} else {
	typedmemclr(t.key, k)
}

// 对 value 清零
if t.indirectvalue {
	*(*unsafe.Pointer)(v) = nil
} else {
	typedmemclr(t.elem, v)
}
```

最后，将 count 值减 1，将对应位置的 tophash 值置成 `Empty`。

## 可以对 map 的元素取地址吗

无法对 map 的 key 或 value 进行取址。以下代码不能通过编译：

```go
package main

import "fmt"
func main() {
	m := make(map[string]int)
	fmt.Println(&m["qcrao"])
}
```

编译报错：
```go
./main.go:8:14: cannot take the address of m["qcrao"]
```
如果通过其他 hack 的方式，例如 unsafe.Pointer 等获取到了 key 或 value 的地址，也不能长期持有，因为一旦发生扩容，key 和 value 的位置就会改变，之前保存的地址也就失效了。

## map 的遍历过程是怎样的

本来 map 的遍历过程比较简单：遍历所有的 bucket 以及它后面挂的 overflow bucket，然后挨个遍历 bucket 中的所有 cell。每个 bucket 中包含 8 个 cell，从有 key 的 cell 中取出 key 和 value，这个过程就完成了。

但是，现实并没有这么简单。还记得前面讲过的扩容过程吗？扩容过程不是一个原子的操作，它每次最多只搬运 2 个 bucket，所以如果触发了扩容操作，那么在很长时间里，map 的状态都是处于一个中间态：有些 bucket 已经搬迁到新家，而有些 bucket 还待在老地方。

因此，遍历如果发生在扩容的过程中，就会涉及到遍历新老 bucket 的过程，这是难点所在。

我先写一个简单的代码样例，假装不知道遍历过程具体调用的是什么函数：

```go
package main

import "fmt"
func main() {

ageMp := make(map[string]int)

ageMp["qcrao"] = 18
for name, age := range ageMp {

fmt.Println(name, age)

}

}
```

执行命令：

go tool compile -S main.go

得到汇编命令。这里就不逐行讲解了，可以去看之前的几篇文章，说得很详细。

这样，关于 map 迭代，底层的函数调用关系一目了然。先是调用 `mapiterinit` 函数初始化迭代器，然后循环调用 `mapiternext` 函数进行 map 迭代。

![](https://user-images.githubusercontent.com/7698088/57976471-ad2ebf00-7a13-11e9-8dd8-d7be54f96440.png)

map iter loop

迭代器的结构体定义：

```go
type hiter struct {

// key 指针

key unsafe.Pointer

// value 指针

value unsafe.Pointer

// map 类型，包含如 key size 大小等

t *maptype

// map header

h *hmap

// 初始化时指向的 bucket

buckets unsafe.Pointer

// 当前遍历到的 bmap

bptr *bmap

overflow [2]*[]*bmap

// 起始遍历的 bucet 编号

startBucket uintptr

// 遍历开始时 cell 的编号（每个 bucket 中有 8 个 cell）

offset uint8

// 是否从头遍历了

wrapped bool

// B 的大小

B uint8

// 指示当前 cell 序号

i uint8

// 指向当前的 bucket

bucket uintptr

// 因为扩容，需要检查的 bucket

checkBucket uintptr

}
```

`mapiterinit` 就是对 hiter 结构体里的字段进行初始化赋值操作。

前面已经提到过，即使是对一个写死的 map 进行遍历，每次出来的结果也是无序的。下面我们就可以近距离地观察他们的实现了。

```go
// 生成随机数 r
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
	r += uintptr(fastrand()) << 31
}

// 从哪个 bucket 开始遍历
it.startBucket = r & (uintptr(1)<<h.B - 1)
// 从 bucket 的哪个 cell 开始遍历
it.offset = uint8(r >> h.B & (bucketCnt - 1))
```

例如，B = 2，那 `uintptr(1)<<h.B - 1` 结果就是 3，低 8 位为 `0000 0011`，将 r 与之相与，就可以得到一个 `0~3` 的 bucket 序号；bucketCnt - 1 等于 7，低 8 位为 `0000 0111`，将 r 右移 2 位后，与 7 相与，就可以得到一个 `0~7` 号的 cell。

于是，在 `mapiternext` 函数中就会从 it.startBucket 的 it.offset 号的 cell 开始遍历，取出其中的 key 和 value，直到又回到起点 bucket，完成遍历过程。

源码部分比较好看懂，尤其是理解了前面注释的几段代码后，再看这部分代码就没什么压力了。所以，接下来，我将通过图形化的方式讲解整个遍历过程，希望能够清晰易懂。

假设我们有下图所示的一个 map，起始时 B = 1，有两个 bucket，后来触发了扩容（这里不要深究扩容条件，只是一个设定），B 变成 2。并且， 1 号 bucket 中的内容搬迁到了新的 bucket，`1 号`裂变成 `1 号`和 `3 号`；`0 号` bucket 暂未搬迁。老的 bucket 挂在在 `*oldbuckets` 指针上面，新的 bucket 则挂在 `*buckets` 指针上面。

![](https://user-images.githubusercontent.com/7698088/57978113-f8a79400-7a38-11e9-8e27-3f3ba4fa557f.png)

map origin

这时，我们对此 map 进行遍历。假设经过初始化后，startBucket = 3，offset = 2。于是，遍历的起点将是 3 号 bucket 的 2 号 cell，下面这张图就是开始遍历时的状态：

![](https://user-images.githubusercontent.com/7698088/57980268-a4fa7200-7a5b-11e9-9ad1-fb2b64fe3159.png)

map init

标红的表示起始位置，bucket 遍历顺序为：3 -> 0 -> 1 -> 2。

因为 3 号 bucket 对应老的 1 号 bucket，因此先检查老 1 号 bucket 是否已经被搬迁过。判断方法就是：

```go
func evacuated(b *bmap) bool {

h := b.tophash[0]

return h > empty && h < minTopHash

}
```

如果 b.tophash[0] 的值在标志值范围内，即在 (0,4) 区间里，说明已经被搬迁过了。

```go
empty = 0

evacuatedEmpty = 1

evacuatedX = 2

evacuatedY = 3

minTopHash = 4
```

在本例中，老 1 号 bucket 已经被搬迁过了。所以它的 tophash[0] 值在 (0,4) 范围内，因此只用遍历新的 3 号 bucket。

依次遍历 3 号 bucket 的 cell，这时候会找到第一个非空的 key：元素 e。到这里，mapiternext 函数返回，这时我们的遍历结果仅有一个元素：

![](https://user-images.githubusercontent.com/7698088/57980302-56010c80-7a5c-11e9-8263-c11ddcec2ecc.png)

iter res

由于返回的 key 不为空，所以会继续调用 mapiternext 函数。

继续从上次遍历到的地方往后遍历，从新 3 号 overflow bucket 中找到了元素 f 和 元素 g。

遍历结果集也因此壮大：

![](https://user-images.githubusercontent.com/7698088/57980349-2d2d4700-7a5d-11e9-819a-a59964f70a7c.png)

iter res

新 3 号 bucket 遍历完之后，回到了新 0 号 bucket。0 号 bucket 对应老的 0 号 bucket，经检查，老 0 号 bucket 并未搬迁，因此对新 0 号 bucket 的遍历就改为遍历老 0 号 bucket。那是不是把老 0 号 bucket 中的所有 key 都取出来呢？

并没有这么简单，回忆一下，老 0 号 bucket 在搬迁后将裂变成 2 个 bucket：新 0 号、新 2 号。而我们此时正在遍历的只是新 0 号 bucket（注意，遍历都是遍历的 `*bucket` 指针，也就是所谓的新 buckets）。所以，我们只会取出老 0 号 bucket 中那些在裂变之后，分配到新 0 号 bucket 中的那些 key。

因此，`lowbits == 00` 的将进入遍历结果集：

![](https://user-images.githubusercontent.com/7698088/57980449-6fa35380-7a5e-11e9-9dbf-86332ea0e215.png)

iter res

和之前的流程一样，继续遍历新 1 号 bucket，发现老 1 号 bucket 已经搬迁，只用遍历新 1 号 bucket 中现有的元素就可以了。结果集变成：

![](https://user-images.githubusercontent.com/7698088/57980487-e8a2ab00-7a5e-11e9-8e47-050437a099fc.png)

iter res

继续遍历新 2 号 bucket，它来自老 0 号 bucket，因此需要在老 0 号 bucket 中那些会裂变到新 2 号 bucket 中的 key，也就是 `lowbit == 10` 的那些 key。

这样，遍历结果集变成：

![](https://user-images.githubusercontent.com/7698088/57980574-ae85d900-7a5f-11e9-8050-ae314a90ee05.png)

iter res

最后，继续遍历到新 3 号 bucket 时，发现所有的 bucket 都已经遍历完毕，整个迭代过程执行完毕。

顺便说一下，如果碰到 key 是 `math.NaN()` 这种的，处理方式类似。核心还是要看它被分裂后具体落入哪个 bucket。只不过只用看它 top hash 的最低位。如果 top hash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。据此决定是否取出 key，放到遍历结果集里。

map 遍历的核心在于理解 2 倍扩容时，老 bucket 会分裂到 2 个新 bucket 中去。而遍历操作，会按照新 bucket 的序号顺序进行，碰到老 bucket 未搬迁的情况时，要在老 bucket 中找到将来要搬迁到新 bucket 来的 key。

## # map 的赋值过程是怎样的
函数首先会检查 map 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。这也说明了 map 对协程是不安全的。

通过前文我们知道扩容是渐进式的，如果 map 处在扩容的过程中，那么当 key 定位到了某个 bucket 后，需要确保这个 bucket 对应的老 bucket 完成了迁移过程。即老 bucket 里的 key 都要迁移到新的 bucket 中来（分裂到 2 个新 bucket），才能在新的 bucket 中进行插入或者更新的操作。

上面说的操作是在函数靠前的位置进行的，只有进行完了这个搬迁操作后，我们才能放心地在新 bucket 里定位 key 要安置的地址，再进行之后的操作。

现在到了定位 key 应该放置的位置了，所谓找准自己的位置很重要。准备两个指针，一个（`inserti`）指向 key 的 hash 值在 tophash 数组所处的位置，另一个(`insertk`)指向 cell 的位置（也就是 key 最终放置的地址），当然，对应 value 的位置就很容易定位出来了。这三者实际上都是关联的，在 tophash 数组中的索引位置决定了 key 在整个 bucket 中的位置（共 8 个 key），而 value 的位置需要“跨过” 8 个 key 的长度。

在循环的过程中，inserti 和 insertk 分别指向第一个找到的空闲的 cell。如果之后在 map 没有找到 key 的存在，也就是说原来 map 中没有此 key，这意味着插入新 key。那最终 key 的安置地址就是第一次发现的“空位”（tophash 是 empty）。

如果这个 bucket 的 8 个 key 都已经放置满了，那在跳出循环后，发现 inserti 和 insertk 都是空，这时候需要在 bucket 后面挂上 overflow bucket。当然，也有可能是在 overflow bucket 后面再挂上一个 overflow bucket。这就说明，太多 key hash 到了此 bucket。

在正式安置 key 之前，还要检查 map 的状态，看它是否需要进行扩容。如果满足扩容的条件，就主动触发一次扩容操作。

这之后，整个之前的查找定位 key 的过程，还得再重新走一次。因为扩容之后，key 的分布都发生了变化。

最后，会更新 map 相关的值，如果是插入新 key，map 的元素数量字段 count 值会加 1；在函数之初设置的 `hashWriting` 写标志出会清零。

另外，有一个重要的点要说一下。前面说的找到 key 的位置，进行赋值操作，实际上并不准确。我们看 `mapassign` 函数的原型就知道，函数并没有传入 value 值，所以赋值操作是什么时候执行的呢？

func mapassign(t \*maptype, h \*hmap, key unsafe.Pointer) unsafe.Pointer

答案还得从汇编语言中寻找。我直接揭晓答案，有兴趣可以私下去研究一下。`mapassign` 函数返回的指针就是指向的 key 所对应的 value 值位置，有了地址，就很好操作赋值了。

-   Rehash 过程中存放在旧桶的元素如何迁移？
渐进式hash中每次最少搬运一个桶，最多两个桶，将取最低位的数量增加，就会hash到不同的位置，然后来hash.
-   如果并发环境想要用这种哈希容器有什么方案？
-   sync.Mutex / sync.RWMutex
-   sync.Map
-   加锁存在什么问题呢？
-   sync.Map 比加锁的方案好在哪里，它的底层数据结构是怎样的？
-   缓存 + map 组成的结构底层 map 的操作依然是加锁的，但是读的时候使用上缓存可以增加并发性能

-   sync.Map 的 Load() 方法流程？
从readonlymap中读，加读锁，读不到就加锁从dirty中读，然后增加miss，miss到了某个值就会把dirty复制到read

-   sync.Map Store() 如何保持缓存层和底层 Map 数据是相同的? 是不是每次执行修改都需要去加锁？
dirty的数据的readmap的超集，当dirty迁移到readonly的时候，先置为nil，然后再下一次写入的时候，将readonly放入ditry。替换过程会加锁。为什么要先nil再写入，因为防止dirty留着一些readonly没有的数据。

##  channel 被 close 操作之后进行读和写会有什么问题？
先创建了一个有缓冲的 channel，向其发送一个元素，然后关闭此 channel。之后两次尝试从 channel 中读取数据，第一次仍然能正常读出值。第二次返回的 ok 为 false，说明 channel 已关闭，且通道里没有数据。

写的话就panic
## # 如何优雅地关闭 channel
不要从一个 receiver 侧关闭 channel，也不要在有多个 sender 时，关闭 channel。

根据 sender 和 receiver 的个数，分下面几种情况：

- 1. 一个 sender，一个 receiver
- 2.  一个 sender， M 个 receiver
- 3.  N 个 sender，一个 reciver
- 4.N 个 sender， M 个 receiver

对于 1，2，只有一个 sender 的情况就不用说了，直接从 sender 端关闭就好了，没有问题。重点关注第 3，4 种情况。

第 3 种情形下，优雅关闭 channel 的方法是：the only receiver says "please stop sending more" by closing an additional signal channel。

解决方案就是增加一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据 channel 指令。senders 监听到关闭信号后，停止接收数据。
这里的 stopCh 就是信号 channel，它本身只有一个 sender，因此可以直接关闭它。senders 收到了关闭信号后，select 分支 “case <- stopCh” 被选中，退出函数，`不再发送数据。``

需要说明的是，上面的代码并没有明确关闭 dataCh。在 Go 语言中，对于一个 channel，`如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。`所以，在这种情形下，所谓的优雅地关闭 channel 就是不关闭 channel，让 gc 代劳。

最后一种情况，优雅关闭 channel 的方法是：any one of them says "let's end the game" by notifying a moderator to close an additional signal channel。

和第 3 种情况不同，这里有 M 个 receiver，如果直接还是采取第 3 种解决方案，由 receiver 直接关闭 stopCh 的话，就会重复关闭一个 channel，导致 panic。因此需要增加一个中间人，M 个 receiver 都向它发送关闭 dataCh 的“请求”，中间人收到第一个请求后，就会直接下达关闭 dataCh 的指令（通过关闭 stopCh，这时就不会发生重复关闭的情况，因为 stopCh 的发送方只有中间人一个）。另外，这里的 N 个 sender 也可以向中间人发送关闭 dataCh 的请求。

```go
func main() {
	rand.Seed(time.Now().UnixNano())
	const Max = 100000
	const NumReceivers = 10
	const NumSenders = 1000
	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})
	// It must be a buffered channel.
	toStop := make(chan string, 1)
	var stoppedBy string
	// moderato
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()
	// senders
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}
				select {
					case <- stopCh:
					return
					case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}
// receivers
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			for {
				select {
					case <- stopCh:
						return
					case value := <-dataCh:
						if value == Max-1 {
						select {
						case toStop <- "receiver#" + id:
					default:
				}
					return
			}
			fmt.Println(value)
		}
	}
	}(strconv.Itoa(i))
}
select {
case <- time.After(time.Hour):
}
}
```

代码里 toStop 就是中间人的角色，使用它来接收 senders 和 receivers 发送过来的关闭 dataCh 请求。

这里将 toStop 声明成了一个 缓冲型的 channel。假设 toStop 声明的是一个非缓冲型的 channel，那么第一个发送的关闭 dataCh 请求可能会丢失。因为无论是 sender 还是 receiver 都是通过 select 语句来发送请求，如果中间人所在的 goroutine 没有准备好，那 select 语句就不会选中，直接走 default 选项，什么也不做。这样，第一个关闭 dataCh 的请求就会丢失。

如果，我们把 toStop 的容量声明成 Num(senders) + Num(receivers)，那发送 dataCh 请求的部分可以改成更简洁的形式：

```go
...

toStop := make(chan string, NumReceivers + NumSenders)

...

value := rand.Intn(Max)

if value == 0 {

toStop <- "sender#" + id

return

}

...

if value == Max-1 {

toStop <- "receiver#" + id

return

}

...
```

直接向 toStop 发送请求，因为 toStop 容量足够大，所以不用担心阻塞，自然也就不用 select 语句再加一个 default case 来避免阻塞。

可以看到，这里同样没有真正关闭 dataCh，原样同第 3 种情况。

以上，就是最基本的一些情形，但已经能覆盖几乎所有的情况及其变种了。只要记住：

> don't close a channel from the receiver side and don't close a channel if the channel has multiple concurrent senders.

以及更本质的原则：

> don't close (or send values to) closed channels.

## # 10 - channel 在什么情况下会引起资源泄漏

Channel 可能会引发 goroutine 泄漏。

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，不见天日。

另外，程序运行过程中，对于一个 channel，如果没有任何 goroutine 引用了，gc 会对其进行回收操作，不会引起内存泄漏。

## 什么是csp
不要通过共享内存来通信，而要通过通信来实现内存共享。

这就是 Go 的并发哲学，它依赖 CSP 模型，基于 channel 实现。

## # 09 - channel 发送和接收元素的本质是什么

Channel 发送和接收元素的本质是什么？

> All transfer of value on the go channels happens with the copy of value.

就是说 channel 的发送和接收操作本质上都是 “值的拷贝”，无论是从 sender goroutine 的栈到 chan buf，还是从 chan buf 到 receiver goroutine，或者是直接从 sender goroutine 到 receiver goroutine。

## 关闭chan的过程
close 逻辑比较简单，对于一个 channel，recvq 和 sendq 中分别保存了阻塞的发送者和接收者。关闭 channel 后，对于等待接收者而言，会收到一个相应类型的零值。对于等待发送者，会直接 panic。所以，在不了解 channel 还有没有接收者的情况下，不能贸然关闭 channel。

close 函数先上一把大锁，接着把所有挂在这个 channel 上的 sender 和 receiver 全都连成一个 sudog 链表，再解锁。最后，再将所有的 sudog 全都唤醒。

唤醒之后，该干嘛干嘛。sender 会继续执行 chansend 函数里 goparkunlock 函数之后的代码，很不幸，检测到 channel 已经关闭了，panic。receiver 则比较幸运，进行一些扫尾工作后，返回。这里，selected 返回 true，而返回值 received 则要根据 channel 是否关闭，返回不同的值。如果 channel 关闭，received 为 false，否则为 true。这我们分析的这种情况下，received 返回 false。

##   未被初始化的 channel 进行读写会有什么问题？
**为什么对未初始化的 `chan` 就会阻塞呢？**

**1. 对于写的情况**
```go
//在 src/runtime/chan.go中 
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {  
	if c == nil {       
		// 不能阻塞，直接返回 false，表示未发送成功       
		if !block {         return false       }       
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)       
		throw("unreachable")  
	}   // 省略其他逻辑 }
```

- 未初始化的 `chan` 此时是等于 `nil`，当它不能阻塞的情况下，直接返回 `false`，表示写 `chan` 失败
- 当 `chan` 能阻塞的情况下，则直接阻塞 `gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)`, 然后调用 `throw(s string)` 抛出错误，其中 `waitReasonChanSendNilChan` 就是刚刚提到的报错 `"chan send (nil chan)"`

**2. 对于读的情况**
```go
//在 src/runtime/chan.go中 
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {     
	//省略逻辑...     
	if c == nil {         
		if !block {           
			return         
		}         
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")     
	}     
	//省略逻辑... }
```

- 未初始化的 `chan` 此时是等于 `nil`，当它不能阻塞的情况下，直接返回 `false`，表示读 `chan` 失败
- 当 `chan` 能阻塞的情况下，则直接阻塞 `gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)`, 然后调用 `throw(s string)` 抛出错误，其中 `waitReasonChanReceiveNilChan` 就是刚刚提到的报错 `"chan receive (nil chan)"`

## channel的happend-before
根据晃岳攀老师在 Gopher China 2019 上的并发编程分享，关于 channel 的发送（send）、发送完成（send finished）、接收（receive）、接收完成（receive finished）的 happened-before 关系如下：

- 1.    第 n 个 `send` 一定 `happened before` 第 n 个 `receive finished`，无论是缓冲型还是非缓冲型的 channel。

- 2.    对于容量为 m 的缓冲型 channel，第 n 个 `receive` 一定 `happened before` 第 n+m 个 `send finished`。

- 3.    对于非缓冲型的 channel，第 n 个 `receive` 一定 `happened before` 第 n 个 `send finished`。

- 4.    channel close 一定 `happened before` receiver 得到通知。

##    channel 底层数据结构是怎样的，尝试用结构体来表述一下？
```go
type hchan struct {
	// chan 里元素数量
	qcount uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx uint // send index
	// 已接收元素在循环数组中的索引
	recvx uint // receive index
	// 等待接收的 goroutine 队列
	recvq waitq // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq waitq // list of send waiters
	// 保护 hchan 中所有字段
	lock mutex   //!!!!!!!会加锁的
}
```
怎么讲检测热key？
heavyKeeper

## 大量的 TIME_WAIT 状态 TCP 连接存在，其本质原因是什么?

- 大量的短连接存在
- 特别是 HTTP 请求中，如果 connection 头部取值被设置为 close 时，基本都由「服务端」发起主动关闭连接
- 而，TCP 四次挥手关闭连接机制中，为了保证 ACK 重发和丢弃延迟数据，设置 time_wait 为 2 倍的 MSL(报文最大存活时间)

TIME_WAIT 状态：

- TCP 连接中，主动关闭连接的一方出现的状态;(收到 FIN 命令，进入 TIME_WAIT 状态，并返回 ACK 命令)
- 保持 2 个 MSL 时间，即，4 分钟;(MSL 为 2 分钟)

-   查看 TCP 连接状态需要用什么命令？
```
// 统计：各种连接的数量 
$ netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}' 
ESTABLISHED 1154 
TIME_WAIT 1645 
```

## -   发现存在大量 TIME_WAIT 状态会存在什么问题？
- 每一个 time_wait 状态，都会占用一个「本地端口」，上限为 65535(16 bit，2 Byte);
- 当大量的连接处于 time_wait 时，新建立 TCP 连接会出错，address already in use : connect 异常

## 关于 HTTP 请求中，设置的主动关闭 TCP 连接的机制：TIME_WAIT的是主动断开方才会出现的，所以主动断开方是服务端?

- 答案是是的。在HTTP1.1协议中，有个 Connection 头，Connection有两个值，close和keep-alive，这个头就相当于客户端告诉服务端，服务端你执行完成请求之后，是关闭连接还是保持连接，保持连接就意味着在保持连接期间，只能由客户端主动断开连接。还有一个keep-alive的头，设置的值就代表了服务端保持连接保持多久。
- HTTP默认的Connection值为close，那么就意味着关闭请求的一方几乎都会是由服务端这边发起的。那么这个服务端产生TIME_WAIT过多的情况就很正常了。
- 虽然HTTP默认Connection值为close，但是，现在的浏览器发送请求的时候一般都会设置Connection为keep-alive了。所以，也有人说，现在没有必要通过调整参数来使TIME_WAIT降低了。

## 关于 time_wait：

(1) TCP 连接建立后，「主动关闭连接」的一端，收到对方的 FIN 请求后，发送 ACK 响应，会处于 time_wait 状态;

(2) time_wait 状态，存在的必要性：

- 可靠的实现 TCP 全双工连接的终止：四次挥手关闭 TCP 连接过程中，最后的 ACK 是由「主动关闭连接」的一端发出的，如果这个 ACK 丢失，则，对方会重发 FIN 请求，因此，在「主动关闭连接」的一段，需要维护一个 time_wait 状态，处理对方重发的 FIN 请求;
- 处理延迟到达的报文：由于路由器可能抖动，TCP 报文会延迟到达，为了避免「延迟到达的 TCP 报文」被误认为是「新 TCP 连接」的数据，则，需要在允许新创建 TCP 连接之前，保持一个不可用的状态，等待所有延迟报文的消失，一般设置为 2 倍的 MSL(报文的最大生存时间)，解决「延迟达到的 TCP 报文」问题;

## Redis 的哈希数据类型，想插入元素，底层数据结构是怎样变化？
桶上加链表，数据等于桶数量就扩容，小于某个数量就缩容。渐进式hash，每次搬几个桶，还有就是定时hash。

数据小的情况下就用压缩链表，结构为
```c
struct ziplist<T> {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}
```
```go
struct entry {
    int<var> prevlen; // 前一个 entry 的字节长度
    int<var> encoding; // 元素类型编码
    optional byte[] content; // 元素内容
}
```

zset的数据结构是
hash+跳表

set的数据结构是
hash，小数据的时候就用inset

##  AOF 在做压缩的时候 Redis 对外是否正常提供服务？
可以的，重写交给子进程。


## **1、golang 中 make 和 new 的区别？

**共同点：**给变量分配内存

**不同点：**

1）作用变量类型不同，new给string,int和数组分配内存，make给切片，map，channel分配内存；

2）返回类型不一样，new返回指向变量的指针，make返回变量本身；

3）new 分配的空间被清零。make 分配空间后，会进行初始化；

4) 字节的xx官还说了另外一个区别，就是分配的位置，在堆上还是在栈上？make放在堆上，因为类型不确定，大小不确定，new的话就考虑内存逃逸的情况。

### **2、数组和切片的区别 （基本必问）**

**相同点：**

1)只能存储一组相同类型的数据结构

2)都是通过下标来访问，并且有容量长度，长度通过 len 获取，容量通过 cap 获取

**区别：**

1）数组是定长，访问和复制不能超过数组定义的长度，否则就会下标越界，切片长度和容量可以自动扩容

2）数组是值类型，切片是引用类型，每个切片都引用了一个底层数组，切片本身不能存储任何数据，都是这底层数组存储数据，所以修改切片的时候修改的是底层数组中的数据。切片一旦扩容，指向一个新的底层数组，内存地址也就随之改变

**简洁的回答：**

1）定义方式不一样 2）初始化方式不一样，数组需要指定大小，大小不改变 3）在函数传递中，数组切片都是值传递。

### **3、for range 的时候它的地址会发生变化么？**

答：在 for a,b := range c 遍历中， a 和 b 在内存中只会存在一份，即之后每次循环时遍历到的数据都是以值覆盖的方式赋给 a 和 b，a，b 的内存地址始终不变。由于有这个特性，for 循环里面如果开协程，不要直接把 a 或者 b 的地址传给协程。解决办法：在每次循环时，创建一个临时变量。

### **4、go defer，多个 defer 的顺序，defer 在什么时机会修改返回值？**

作用：defer延迟函数，释放资源，收尾工作；如释放锁，关闭文件，关闭链接；捕获panic;

避坑指南：defer函数紧跟在资源打开后面，否则defer可能得不到执行，导致内存泄露。

多个 defer 调用顺序是 LIFO（后入先出），defer后的操作可以理解为压入栈中

defer，return，return value（函数返回值） 执行顺序：首先return，其次return value，最后defer。defer可以修改函数最终返回值，修改时机：**有名返回值或者函数返回指针** 参考：

**有名返回值**

```go
func b() (i int) {
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return i //或者直接写成return
}
func main() {
	fmt.Println("return:", b())
}
```

////////////////////////////////////////////////////////////////////////////////

**函数返回指针**

```go
func c() *int {
	var i int
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return &i
}
func main() {
	fmt.Println("return:", *(c()))
}
```

## **5、uint 类型溢出问题**

超过最大存储值如uint8最大是255

var a uint8 =255

var b uint8 =1

a+b = 0总之类型溢出会出现难以意料的事

### **6、能介绍下 rune 类型吗？**

相当int32

golang中的字符串底层实现是通过byte数组的，中文字符在unicode下占2个字节，在utf-8编码下占3个字节，而golang默认编码正好是utf-8

byte 等同于int8，常用来处理ascii字符

rune 等同于int32,常用来处理unicode或utf-8字符

### **7、 golang 中解析 tag 是怎么实现的？反射原理是什么？(中高级肯定会问，比较难，需要自己多去总结)**

```go
type User struct {
	name string `json:name-field`
	age  int
}
func main() {
	user := &User{"John Doe The Fourth", 20}

	field, ok := reflect.TypeOf(user).Elem().FieldByName("name")
	if !ok {
		panic("Field not found")
	}
	fmt.Println(getStructTag(field))
}

func getStructTag(f reflect.StructField) string {
	return string(f.Tag)
}
```

Go 中解析的 tag 是通过反射实现的，反射是指计算机程序在运行时（Run time）可以访问、检测和修改它本身状态或行为的一种能力或动态知道给定数据对象的类型和结构，并有机会修改它。反射将接口变量转换成反射对象 Type 和 Value；反射可以通过反射对象 Value 还原成原先的接口变量；反射可以用来修改一个变量的值，前提是这个值可以被修改；tag是啥:结构体支持标记，name string `json:name-field` 就是 `json:name-field` 这部分

**gorm json yaml gRPC protobuf gin.Bind()都是通过反射来实现的**

### **8、调用函数传入结构体时，应该传值还是指针？ （Golang 都是传值）**

Go 的函数参数传递都是值传递。所谓值传递：指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。参数传递还有引用传递，所谓引用传递是指在调用函数时将实际参数的地址传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数

因为 Go 里面的 map，slice，chan 是引用类型。变量区分值类型和引用类型。所谓值类型：变量和变量的值存在同一个位置。所谓引用类型：变量和变量的值是不同的位置，变量的值存储的是对值的引用。但并不是 map，slice，chan 的所有的变量在函数内都能被修改，不同数据类型的底层存储结构和实现可能不太一样，情况也就不一样。

### **9、讲讲 Go 的 slice 底层数据结构和一些特性？**

答：Go 的 slice 底层数据结构是由一个 array 指针指向底层数组，len 表示切片长度，cap 表示切片容量。slice 的主要实现是扩容。对于 append 向 slice 添加元素时，假如 slice 容量够用，则追加新元素进去，slice.len++，返回原来的 slice。当原容量不够，则 slice 先扩容，扩容之后 slice 得到新的 slice，将元素追加进新的 slice，slice.len++，返回新的 slice。对于切片的扩容规则：当切片比较小时（容量小于 1024），则采用较大的扩容倍速进行扩容（新的扩容会是原来的 2 倍），避免频繁扩容，从而减少内存分配的次数和数据拷贝的代价。当切片较大的时（原来的 slice 的容量大于或者等于 1024），采用较小的扩容倍速（新的扩容将扩大大于或者等于原来 1.25 倍），主要避免空间浪费，网上其实很多总结的是 1.25 倍，那是在不考虑内存对齐的情况下，实际上还要考虑内存对齐，扩容是大于或者等于 1.25 倍。

（关于刚才问的 slice 为什么传到函数内可能被修改，如果 slice 在函数内没有出现扩容，函数外和函数内 slice 变量指向是同一个数组，则函数内复制的 slice 变量值出现更改，函数外这个 slice 变量值也会被修改。如果 slice 在函数内出现扩容，则函数内变量的值会新生成一个数组（也就是新的 slice，而函数外的 slice 指向的还是原来的 slice，则函数内的修改不会影响函数外的 slice。）

### **10、讲讲 Go 的 select 底层数据结构和一些特性？

答：go 的 select 为 golang 提供了多路 IO 复用机制，和其他 IO 复用一样，用于检测是否有读写事件是否 ready。linux 的系统 IO 模型有 select，poll，epoll，go 的 select 和 linux 系统 select 非常相似。

select 结构组成主要是由 case 语句和执行的函数组成 select 实现的多路复用是：每个线程或者进程都先到注册和接受的 channel（装置）注册，然后阻塞，然后只有一个线程在运输，当注册的线程和进程准备好数据后，装置会根据注册的信息得到相应的数据。

**select 的特性**

1）select 操作至少要有一个 case 语句，出现读写 nil 的 channel 该分支会忽略，在 nil 的 channel 上操作则会报错。

2）select 仅支持管道，而且是单协程操作。

3）每个 case 语句仅能处理一个管道，要么读要么写。

4）多个 case 语句的执行顺序是随机的。

5）存在 default 语句，select 将不会阻塞，但是存在 default 会影响性能。

### **11、讲讲 Go 的 defer 底层数据结构和一些特性？**

答：每个 defer 语句都对应一个_defer 实例，多个实例使用指针连接起来形成一个单连表，保存在 gotoutine 数据结构中，每次插入_defer 实例，均插入到链表的头部，函数结束再一次从头部取出，从而形成后进先出的效果。

**defer 的规则总结**：

延迟函数的参数是 defer 语句出现的时候就已经确定了的。

延迟函数执行按照后进先出的顺序执行，即先出现的 defer 最后执行。

延迟函数可能操作主函数的返回值。

申请资源后立即使用 defer 关闭资源是个好习惯。

### **12、单引号，双引号，反引号的区别？**

单引号，表示byte类型或rune类型，对应 uint8和int32类型，默认是 rune 类型。byte用来强调数据是raw data，而不是数字；而rune用来表示Unicode的code point。

双引号，才是字符串，实际上是字符数组。可以用索引号访问某字节，也可以用len()函数来获取字符串所占的字节长度。

反引号，表示字符串字面量，但不支持任何转义序列。字面量 raw literal string 的意思是，你定义时写的啥样，它就啥样，你有换行，它就换行。你写转义字符，它也就展示转义字符。


### 3、 map 中删除一个 key，它的内存会释放么？
将key和value置为nil，然后减少map count，topHash置为空标志。但是空位是不会减少的，位置是一定留着的。

如果删除的元素是值类型，如int，float，bool，string以及数组和struct，map的内存不会自动释放.因为B的数量是不会变的，所以一定只能增长或者维持现状，不会减少桶的数量的（溢出桶除外）

如果删除的元素是引用类型，如指针，slice，map，chan等，map的内存会自动释放，但释放的内存是子元素应用类型的内存占用。假如value是数组，那么就会将数组释放掉，但是留下的空位是不会释放的。

将map设置为nil后，内存被回收。比如**创建一个新的 map 并从旧的 map 中复制元素**。

**这个问题还需要大家去搜索下答案，我记得有不一样的说法，谨慎采用本题答案。**


### 5、 nil map 和空 map 有何不同？

1）可以对未初始化的map进行取值，但取出来的东西是空：

var m1 map[string]string

fmt.Println(m1["1"])

////////////////////////////////////////////////////////////////////////////////

2）不能对未初始化的map进行赋值，这样将会抛出一个异常：

var m1 map[string]string

m1["1"] = "1"

panic: assignment to entry in nil map

////////////////////////////////////////////////////////////////////////////////

3) 通过fmt打印map时，空map和nil map结果是一样的，都为map[]。所以，这个时候别断定map是空还是nil，而应该通过map == nil来判断。

**nil map 未初始化，空map是长度为空**

### 6、map 的数据结构是什么？是怎么实现扩容？

答：golang 中 map 是一个 kv 对集合。底层使用 hash table，用链表来解决冲突 ，出现冲突时，不是每一个 key 都申请一个结构通过链表串起来，而是以 bmap 为最小粒度挂载，一个 bmap 可以放 8 个 kv。在哈希函数的选择上，会在程序启动时，检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。每个 map 的底层结构是 hmap，是有若干个结构为 bmap 的 bucket 组成的数组。每个 bucket 底层都采用链表结构。

**hmap 的结构如下：**

```text
type hmap struct {
    count     int                  // 元素个数
    flags     uint8
    B         uint8                // 扩容常量相关字段B是buckets数组的长度的对数 2^B
    noverflow uint16               // 溢出的bucket个数
    hash0     uint32               // hash seed
    buckets    unsafe.Pointer      // buckets 数组指针
    oldbuckets unsafe.Pointer      // 结构扩容的时候用于赋值的buckets数组
    nevacuate  uintptr             // 搬迁进度
    extra *mapextra                // 用于扩容的指针
}
```

**map 的容量大小**
  
底层调用 makemap 函数，计算得到合适的 B，map 容量最多可容纳 6.5\*2^B 个元素，6.5 为装载因子阈值常量。装载因子的计算公式是：装载因子=填入表中的元素个数/散列表的长度，装载因子越大，说明空闲位置越少，冲突越多，散列表的性能会下降。底层调用 makemap 函数，计算得到合适的 B，map 容量最多可容纳 6.52^B 个元素，6.5 为装载因子阈值常量。装载因子的计算公式是：装载因子=填入表中的元素个数/散列表的长度，装载因子越大，说明空闲位置越少，冲突越多，散列表的性能会下降。

**触发 map 扩容的条件**

1）装载因子超过阈值，源码里定义的阈值是 6.5。

2）overflow 的 bucket 数量过多 map 的 bucket 定位和 key 的定位高八位用于定位 bucket，低八位用于定位 key，快速试错后再进行完整对比

### 7、slices能作为map类型的key吗？

当时被问的一脸懵逼，其实是这个问题的变种：golang 哪些类型可以作为map key？

答案是：**在golang规范中，可比较的类型都可以作为map key；**这个问题又延伸到在：golang规范中，哪些数据类型可以比较？

**不能作为map key 的类型包括：**

-   slices
-   maps
-   functions

## 三**、context相关**

### **1、context 结构是什么样的？context 使用场景和用途？**

答：Go 的 Context 的数据结构包含 Deadline，Done，Err，Value，Deadline 方法返回一个 time.Time，表示当前 Context 应该结束的时间，ok 则表示有结束时间，Done 方法当 Context 被取消或者超时时候返回的一个 close 的 channel，告诉给 context 相关的函数要停止当前工作然后返回了，Err 表示 context 被取消的原因，Value 方法表示 context 实现共享数据存储的地方，是协程安全的。context 在业务中是经常被使用的，

**其主要的应用 ：**

1：上下文控制，2：多个 goroutine 之间的数据交互等，3：超时控制：到某个时间点超时，过多久超时。

### **5、讲讲 Go 的 chan 底层数据结构和主要使用场景**

答：channel 的数据结构包含 qccount 当前队列中剩余元素个数，dataqsiz 环形队列长度，即可以存放的元素个数，buf 环形队列指针，elemsize 每个元素的大小，closed 标识关闭状态，elemtype 元素类型，sendx 队列下表，指示元素写入时存放到队列中的位置，recv 队列下表，指示元素从队列的该位置读出。recvq 等待读消息的 goroutine 队列，sendq 等待写消息的 goroutine 队列，lock 互斥锁，chan 不允许并发读写。

**无缓冲和有缓冲区别：** 管道没有缓冲区，从管道读数据会阻塞，直到有协程向管道中写入数据。同样，向管道写入数据也会阻塞，直到有协程从管道读取数据。管道有缓冲区但缓冲区没有数据，从管道读取数据也会阻塞，直到协程写入数据，如果管道满了，写数据也会阻塞，直到协程从缓冲区读取数据。

**channel 的一些特点** 1）、读写值 nil 管道会永久阻塞 2）、关闭的管道读数据仍然可以读数据 3）、往关闭的管道写数据会 panic 4）、关闭为 nil 的管道 panic 5）、关闭已经关闭的管道 panic

**向 channel 写数据的流程：** 如果等待接收队列 recvq 不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从 recvq 取出 G,并把数据写入，最后把该 G 唤醒，结束发送过程； 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程； 如果缓冲区中没有空余位置，将待发送数据写入 G，将当前 G 加入 sendq，进入睡眠，等待被读 goroutine 唤醒；

**向 channel 读数据的流程：** 如果等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G，把 G 中数据读出，最后把 G 唤醒，结束读取过程； 如果等待发送队列 sendq 不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把 G 中数据写入缓冲区尾部，把 G 唤醒，结束读取过程； 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；将当前 goroutine 加入 recvq，进入睡眠，等待被写 goroutine 唤醒；

**使用场景：** 消息传递、消息过滤，信号广播，事件订阅与广播，请求、响应转发，任务分发，结果汇总，并发控制，限流，同步与异步

## **五、GMP相关**

### 1、什么是 GMP？（必问）

答：G 代表着 goroutine，P 代表着上下文处理器，M 代表 thread 线程，在 GPM 模型，有一个全局队列（Global Queue）：存放等待运行的 G，还有一个 P 的本地队列：也是存放等待运行的 G，但数量有限，不超过 256 个。GPM 的调度流程从 go func()开始创建一个 goroutine，新建的 goroutine 优先保存在 P 的本地队列中，如果 P 的本地队列已经满了，则会保存到全局队列中。M 会从 P 的队列中取一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会从其他的 MP 组合偷取一个可执行的 G 来执行，当 M 执行某一个 G 时候发生系统调用或者阻塞，M 阻塞，如果这个时候 G 在执行，runtime 会把这个线程 M 从 P 中摘除，然后创建一个新的操作系统线程来服务于这个 P，当 M 系统调用结束时，这个 G 会尝试获取一个空闲的 P 来执行，并放入到这个 P 的本地队列，如果这个线程 M 变成休眠状态，加入到空闲线程中，然后整个 G 就会被放入到全局队列中。

关于 G,P,M 的个数问题，G 的个数理论上是无限制的，但是受内存限制，P 的数量一般建议是逻辑 CPU 数量的 2 倍，M 的数据默认启动的时候是 10000，内核很难支持这么多线程数，所以整个限制客户忽略，M 一般不做设置，设置好 P，M 一般都是要大于 P。

### 2、进程、线程、协程有什么区别？（必问）

进程：是应用程序的启动实例，每个进程都有独立的内存空间，不同的进程通过进程间的通信方式来通信。

线程：从属于进程，每个进程至少包含一个线程，线程是 CPU 调度的基本单位，多个线程之间可以共享进程的资源并通过共享内存等线程间的通信方式来通信。

协程：为轻量级线程，与线程相比，协程不受操作系统的调度，协程的调度器由用户应用程序提供，协程调度器按照调度策略把协程调度到线程中运行

## **3、抢占式调度是如何抢占的？**

**基于协作式抢占**

**基于信号量抢占**

就像操作系统要负责线程的调度一样，Go的runtime要负责goroutine的调度。现代操作系统调度线程都是抢占式的，我们不能依赖用户代码主动让出CPU，或者因为IO、锁等待而让出，这样会造成调度的不公平。基于经典的时间片算法，当线程的时间片用完之后，会被时钟中断给打断，调度器会将当前线程的执行上下文进行保存，然后恢复下一个线程的上下文，分配新的时间片令其开始执行。这种抢占对于线程本身是无感知的，系统底层支持，不需要开发人员特殊处理。

基于时间片的抢占式调度有个明显的优点，能够避免CPU资源持续被少数线程占用，从而使其他线程长时间处于饥饿状态。goroutine的调度器也用到了时间片算法，但是和操作系统的线程调度还是有些区别的，因为整个Go程序都是运行在用户态的，所以不能像操作系统那样利用时钟中断来打断运行中的goroutine。也得益于完全在用户态实现，goroutine的调度切换更加轻量。

**上面这两段文字只是对调度的一个概括，具体的协作式调度、信号量调度大家还需要去详细了解，这偏底层了，大厂或者中高级开发会问。（字节就问了）**

### 4、M 和 P 的数量问题？

p默认cpu内核数

M与P的数量没有绝对关系，一个M阻塞，P就会去创建或者切换另一个M，所以，即使P的默认数量是1，也有可能会创建很多个M出来

## 六、锁相关

### 1、除了 mutex 以外还有那些方式安全读写共享变量？

* 将共享变量的读写放到一个 goroutine 中，其它 goroutine 通过 channel 进行读写操作。

* 可以用个数为 1 的信号量（semaphore）实现互斥

* 通过 Mutex 锁实现

### 2、Go 如何实现原子操作？

答：原子操作就是不可中断的操作，外界是看不到原子操作的中间状态，要么看到原子操作已经完成，要么看到原子操作已经结束。在某个值的原子操作执行的过程中，CPU 绝对不会再去执行其他针对该值的操作，那么其他操作也是原子操作。

Go 语言的标准库代码包 sync/atomic 提供了原子的读取（Load 为前缀的函数）或写入（Store 为前缀的函数）某个值（这里细节还要多去查查资料）。

**原子操作与互斥锁的区别**

1）、互斥锁是一种数据结构，用来让一个线程执行程序的关键部分，完成互斥的多个操作。

2）、原子操作是针对某个值的单个互斥操作。

### 3、Mutex 是悲观锁还是乐观锁？悲观锁、乐观锁是什么？

**悲观锁**

悲观锁：当要对数据库中的一条数据进行修改的时候，为了避免同时被其他人修改，最好的办法就是直接对该数据进行加锁以防止并发。这种借助数据库锁机制，在修改数据之前先锁定，再修改的方式被称之为悲观并发控制【Pessimistic Concurrency Control，缩写“PCC”，又名“悲观锁”】。

**乐观锁**

乐观锁是相对悲观锁而言的，乐观锁假设数据一般情况不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果冲突，则返回给用户异常信息，让用户决定如何去做。乐观锁适用于读多写少的场景，这样可以提高程序的吞吐量

### 4、Mutex 有几种模式？

**1）正常模式**

1.  当前的mutex只有一个goruntine来获取，那么没有竞争，直接返回。
2.  新的goruntine进来，如果当前mutex已经被获取了，则该goruntine进入一个先入先出的waiter队列，在mutex被释放后，waiter按照先进先出的方式获取锁。该goruntine会处于自旋状态(不挂起，继续占有cpu)。
3.  新的goruntine进来，mutex处于空闲状态，将参与竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。

**2）饥饿模式**

在饥饿模式下，Mutex 的拥有者将直接把锁交给队列最前面的 waiter。新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin（自旋），它会乖乖地加入到等待队列的尾部。 如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正常模式:

1.  此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；
2.  此 waiter 的等待时间小于 1 毫秒。

### 5、goroutine 的自旋占用资源如何解决

自旋锁是指当一个线程在获取锁的时候，如果锁已经被其他线程获取，那么该线程将循环等待，然后不断地判断是否能够被成功获取，直到获取到锁才会退出循环。

**自旋的条件如下：**

1）还没自旋超过 4 次,

2）多核处理器，

3）GOMAXPROCS > 1，

4）p 上本地 goroutine 队列为空。

mutex 会让当前的 goroutine 去空转 CPU，在空转完后再次调用 CAS 方法去尝试性的占有锁资源，直到不满足自旋条件，则最终会加入到等待队列里。

## **七、并发相关**

### 1、怎么控制并发数？

**第一，有缓冲通道**

根据通道中没有数据时读取操作陷入阻塞和通道已满时继续写入操作陷入阻塞的特性，正好实现控制并发数量。

```text
func main() {
	count := 10 // 最大支持并发
	sum := 100 // 任务总数
	wg := sync.WaitGroup{} //控制主协程等待所有子协程执行完之后再退出。

	c := make(chan struct{}, count) // 控制任务并发的chan
	defer close(c)

	for i:=0; i<sum;i++{
		wg.Add(1)
		c <- struct{}{} // 作用类似于waitgroup.Add(1)
		go func(j int) {
			defer wg.Done()
			fmt.Println(j)
			<- c // 执行完毕，释放资源
		}(i)
	}
	wg.Wait()
}
```

**第二，三方库实现的协程池**

panjf2000/ants（比较火）

Jeffail/tunny

```text
import (
	"log"
	"time"

	"github.com/Jeffail/tunny"
)
func main() {
	pool := tunny.NewFunc(10, func(i interface{}) interface{} {
		log.Println(i)
		time.Sleep(time.Second)
		return nil
	})
	defer pool.Close()

	for i := 0; i < 500; i++ {
		go pool.Process(i)
	}
	time.Sleep(time.Second * 4)
}
```

### 2、多个 goroutine 对同一个 map 写会 panic，异常是否可以用 defer 捕获？

可以捕获异常，但是只能捕获一次，Go语言，可以使用多值返回来返回错误。不要用异常代替错误，更不要用来控制流程。在极个别的情况下，才使用Go中引入的Exception处理：defer, panic, recover Go中，对异常处理的原则是：多用error包，少用panic

```text
defer func() {
		if err := recover(); err != nil {
			// 打印异常，关闭资源，退出此函数
			fmt.Println(err)
		}
	}()
```

### 3、如何优雅的实现一个 goroutine 池

（百度、手写代码，本人面传音控股被问道：请求数大于消费能力怎么设计协程池）

这一块能啃下来，offer满天飞，这应该是保证高并发系统稳定性、高可用的核心部分之一。

**建议参考：**

[](https://link.zhihu.com/?target=https%3A//blog.csdn.net/finghting321/article/details/106492915/)

**这篇文章的目录是：**

1. 为什么需要协程池？

2. 简单的协程池

3. go-playground/pool

4. ants（推荐）

**所以直接研究ants底层吧，省的造轮子。**

## **八、GC相关**

### 1、go gc 是怎么实现的？（必问）

答：

**细分常见的三个问题：1、GC机制随着golang版本变化如何变化的？2、三色标记法的流程？3、插入屏障、删除屏障，混合写屏障（具体的实现比较难描述，但你要知道屏障的作用：避免程序运行过程中，变量被误回收；减少STW的时间）4、虾皮还问了个开放性的题目：你觉得以后GC机制会怎么优化？**

Go 的 GC 回收有三次演进过程，Go V1.3 之前普通标记清除（mark and sweep）方法，整体过程需要启动 STW，效率极低。GoV1.5 三色标记法，堆空间启动写屏障，栈空间不启动，全部扫描之后，需要重新扫描一次栈(需要 STW)，效率普通。GoV1.8 三色标记法，混合写屏障机制：栈空间不启动（全部标记成黑色），堆空间启用写屏障，整个过程不要 STW，效率高。

Go1.3 之前的版本所谓标记清除是先启动 STW 暂停，然后执行标记，再执行数据回收，最后停止 STW。Go1.3 版本标记清除做了点优化，流程是：先启动 STW 暂停，然后执行标记，停止 STW，最后再执行数据回收。

Go1.5 三色标记主要是插入屏障和删除屏障，写入屏障的流程：程序开始，全部标记为白色，1）所有的对象放到白色集合，2）遍历一次根节点，得到灰色节点，3）遍历灰色节点，将可达的对象，从白色标记灰色，遍历之后的灰色标记成黑色，4）由于并发特性，此刻外界向在堆中的对象发生添加对象，以及在栈中的对象添加对象，在堆中的对象会触发插入屏障机制，栈中的对象不触发，5）由于堆中对象插入屏障，则会把堆中黑色对象添加的白色对象改成灰色，栈中的黑色对象添加的白色对象依然是白色，6）循环第 5 步，直到没有灰色节点，7）在准备回收白色前，重新遍历扫描一次栈空间，加上 STW 暂停保护栈，防止外界干扰（有新的白色会被添加成黑色）在 STW 中，将栈中的对象一次三色标记，直到没有灰色，8）停止 STW，清除白色。至于删除写屏障，则是遍历灰色节点的时候出现可达的节点被删除，这个时候触发删除写屏障，这个可达的被删除的节点也是灰色，等循环三色标记之后，直到没有灰色节点，然后清理白色，删除写屏障会造成一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮 GC 中被清理掉。

GoV1.8 混合写屏障规则是：

1）GC 开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需 STW)，2）GC 期间，任何在栈上创建的新对象，均为黑色。3）被删除的对象标记为灰色。4）被添加的对象标记为灰色。

### 2、go 是 gc 算法是怎么实现的？ （得物，出现频率低）

```text
func GC() {
	n := atomic.Load(&amp;work.cycles)
	gcWaitOnMark(n)

	gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})
	gcWaitOnMark(n + 1)

	for atomic.Load(&amp;work.cycles) == n+1 &amp;&amp; sweepone() != ^uintptr(0) {
		sweep.nbgsweep++
		Gosched()
	}
	for atomic.Load(&amp;work.cycles) == n+1 &amp;&amp; atomic.Load(&amp;mheap_.sweepers) != 0 {
		Gosched()
	}
	mp := acquirem()
	cycle := atomic.Load(&amp;work.cycles)
	if cycle == n+1 || (gcphase == _GCmark &amp;&amp; cycle == n+2) {
		mProf_PostSweep()
	}
	releasem(mp)
}
```



### 3、GC 中 stw 时机，各个阶段是如何解决的？ （百度）

1）在开始新的一轮 GC 周期前，需要调用 gcWaitOnMark 方法上一轮 GC 的标记结束（含扫描终止、标记、或标记终止等）。

2）开始新的一轮 GC 周期，调用 gcStart 方法触发 GC 行为，开始扫描标记阶段。

3）需要调用 gcWaitOnMark 方法等待，直到当前 GC 周期的扫描、标记、标记终止完成。

4）需要调用 sweepone 方法，扫描未扫除的堆跨度，并持续扫除，保证清理完成。在等待扫除完毕前的阻塞时间，会调用 Gosched 让出。

5）在本轮 GC 已经基本完成后，会调用 mProf_PostSweep 方法。以此记录最后一次标记终止时的堆配置文件快照。

6）结束，释放 M。

### 4、GC 的触发时机？

初级必问，分为系统触发和主动触发。

1）gcTriggerHeap：当所分配的堆大小达到阈值（由控制器计算的触发堆的大小）时，将会触发。

2）gcTriggerTime：当距离上一个 GC 周期的时间超过一定时间时，将会触发。时间周期以runtime.forcegcperiod 变量为准，默认 2 分钟。

3）gcTriggerCycle：如果没有开启 GC，则启动 GC。

4）手动触发的 runtime.GC 方法。

## 你觉得以后GC机制会怎么优化
go1.19的setMemoryLimit
1.20的arena手动管理内存池，现在很多人都用mmap自己搞内存来维护，1.20后就不用了

## **九、内存相关**

### 1、谈谈内存泄露，什么情况下内存会泄露？怎么定位排查内存泄漏问题？

答：go 中的内存泄漏一般都是 goroutine 泄漏，就是 goroutine 没有被关闭，或者没有添加超时控制，让 goroutine 一只处于阻塞状态，不能被 GC。

**内存泄露有下面一些情况**

1）如果 goroutine 在执行时被阻塞而无法退出，就会导致 goroutine 的内存泄漏，一个 goroutine 的最低栈大小为 2KB，在高并发的场景下，对内存的消耗也是非常恐怖的。

2）互斥锁未释放或者造成死锁会造成内存泄漏

3）time.Ticker 是每隔指定的时间就会向通道内写数据。作为循环触发器，必须调用 stop 方法才会停止，从而被 GC 掉，否则会一直占用内存空间。

4）字符串的截取引发临时性的内存泄漏

```text
func main() {
	var str0 = "12345678901234567890"
	str1 := str0[:10]
}
```

5）切片截取引起子切片内存泄漏

```text
func main() {
	var s0 = []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
	s1 := s0[:3]
}
```

6）函数数组传参引发内存泄漏【如果我们在函数传参的时候用到了数组传参，且这个数组够大（我们假设数组大小为 100 万，64 位机上消耗的内存约为 800w 字节，即 8MB 内存），或者该函数短时间内被调用 N 次，那么可想而知，会消耗大量内存，对性能产生极大的影响，如果短时间内分配大量内存，而又来不及 GC，那么就会产生临时性的内存泄漏，对于高并发场景相当可怕。】

**排查方式：**

一般通过 pprof 是 Go 的性能分析工具，在程序运行过程中，可以记录程序的运行信息，可以是 CPU 使用情况、内存使用情况、goroutine 运行情况等，当需要性能调优或者定位 Bug 时候，这些记录的信息是相当重要。

### 2、知道 golang 的内存逃逸吗？什么情况下会发生内存逃逸？（必问）

答：1)本该分配到栈上的变量，跑到了堆上，这就导致了内存逃逸。2)栈是高地址到低地址，栈上的变量，函数结束后变量会跟着回收掉，不会有额外性能的开销。3)变量从栈逃逸到堆上，如果要回收掉，需要进行 gc，那么 gc 一定会带来额外的性能开销。编程语言不断优化 gc 算法，主要目的都是为了减少 gc 带来的额外性能开销，变量一旦逃逸会导致性能开销变大。

**内存逃逸的情况如下：**

1）方法内返回局部变量指针。

2）向 channel 发送指针数据。

3）在闭包中引用包外的值。

4）在 slice 或 map 中存储指针。

5）切片（扩容后）长度太大。

6）在 interface 类型上调用方法。

### 3、请简述 Go 是如何分配内存的？

mcache mcentral mheap mspan

Go 程序启动的时候申请一大块内存，并且划分 spans，bitmap，areana 区域；arena 区域按照页划分成一个个小块，span 管理一个或者多个页，mcentral 管理多个 span 供现场申请使用；mcache 作为线程私有资源，来源于 mcentral。

**这里描述的比较简单，你可以自己再去搜索下更简洁完整的答案。**

### 4、Channel 分配在栈上还是堆上？哪些对象分配在堆上，哪些对象分配在栈上？

Channel 被设计用来实现协程间通信的组件，其作用域和生命周期不可能仅限于某个函数内部，所以 golang 直接将其分配在堆上

准确地说，你并不需要知道。Golang 中的变量只要被引用就一直会存活，存储在堆上还是栈上由内部实现决定而和具体的语法没有关系。

知道变量的存储位置确实和效率编程有关系。如果可能，Golang 编译器会将函数的局部变量分配到函数栈帧（stack frame）上。然而，如果编译器不能确保变量在函数 return 之后不再被引用，编译器就会将变量分配到堆上。而且，如果一个局部变量非常大，那么它也应该被分配到堆上而不是栈上。

当前情况下，如果一个变量被取地址，那么它就有可能被分配到堆上,然而，还要对这些变量做逃逸分析，如果函数 return 之后，变量不再被引用，则将其分配到栈上。

### 5、介绍一下大对象小对象，为什么小对象多了会造成 gc 压力？

小于等于 32k 的对象就是小对象，其它都是大对象。一般小对象通过 mspan 分配内存；大对象则直接由 mheap 分配内存。通常小对象过多会导致 GC 三色法消耗过多的 CPU。优化思路是，减少对象分配。

小对象：如果申请小对象时，发现当前内存空间不存在空闲跨度时，将会需要调用 nextFree 方法获取新的可用的对象，可能会触发 GC 行为。

大对象：如果申请大于 32k 以上的大对象时，可能会触发 GC 行为。

## 十、其他问题

### 1、Go 多返回值怎么实现的？

答：Go 传参和返回值是通过 FP+offset 实现，并且存储在调用函数的栈帧中。FP 栈底寄存器，指向一个函数栈的顶部;PC 程序计数器，指向下一条执行指令;SB 指向静态数据的基指针，全局符号;SP 栈顶寄存器。

### 2、讲讲 Go 中主协程如何等待其余协程退出?

答：Go 的 sync.WaitGroup 是等待一组协程结束，sync.WaitGroup 只有 3 个方法，Add()是添加计数，Done()减去一个计数，Wait()阻塞直到所有的任务完成。Go 里面还能通过有缓冲的 channel 实现其阻塞等待一组协程结束，这个不能保证一组 goroutine 按照顺序执行，可以并发执行协程。Go 里面能通过无缓冲的 channel 实现其阻塞等待一组协程结束，这个能保证一组 goroutine 按照顺序执行，但是不能并发执行。

### 3、Go 语言中不同的类型如何比较是否相等？

答：像 string，int，float interface 等可以通过 reflect.DeepEqual 和等于号进行比较，像 slice，struct，map 则一般使用 reflect.DeepEqual 来检测是否相等。

### 4、Go 中 init 函数的特征?

答：一个包下可以有多个 init 函数，每个文件也可以有多个 init 函数。多个 init 函数按照它们的文件名顺序逐个初始化。应用初始化时初始化工作的顺序是，从被导入的最深层包开始进行初始化，层层递出最后到 main 包。不管包被导入多少次，包内的 init 函数只会执行一次。应用初始化时初始化工作的顺序是，从被导入的最深层包开始进行初始化，层层递出最后到 main 包。但包级别变量的初始化先于包内 init 函数的执行。

### 5、Go 中 uintptr 和 unsafe.Pointer 的区别？

答：unsafe.Pointer 是通用指针类型，它不能参与计算，任何类型的指针都可以转化成 unsafe.Pointer，unsafe.Pointer 可以转化成任何类型的指针，uintptr 可以转换为 unsafe.Pointer，unsafe.Pointer 可以转换为 uintptr。uintptr 是指针运算的工具，但是它不能持有指针对象（意思就是它跟指针对象不能互相转换），unsafe.Pointer 是指针对象进行运算（也就是 uintptr）的桥梁。

### 7、从数组中取一个相同大小的slice有成本吗？
从使用上，没有成本，只是对底层数组的指针进行偏移而已，而且重新设置len和cap
但是从gc上，小数组就引用了大数组，这样大数组就gc不了了


## 连 nil 切片和空切片一不一样都不清楚？
nil切片和空切片的len和cap都为0，但是数组指针的话，nil切片是nil，空切片是一个真实地址。

## 字符串转成 byte 数组，会发生内存拷贝吗？
字符串转成切片，会产生拷贝。严格来说，只要是发生类型强转都会发生内存拷贝。那么问题来了。
**有没有什么办法可以在字符串转成切片的时候不用发生拷贝呢？**

### **代码实现**

```go
package main

import (
 "fmt"
 "reflect"
 "unsafe"
)

func main() {
 a :="aaa"
 ssh := *(*reflect.StringHeader)(unsafe.Pointer(&a))
 b := *(*[]byte)(unsafe.Pointer(&ssh))  
 fmt.Printf("%v",b)
}

```

## **解释**

- `StringHeader` 是`字符串`在go的底层结构。

```go
type StringHeader struct {
 Data uintptr
 Len  int
}
```

- `SliceHeader` 是`切片`在go的底层结构。

```go
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```

- 那么如果想要在底层转换二者，只需要把 `StringHeader` 的地址强转成 `SliceHeader` 就行。那么go有个很强的包叫 `unsafe` 。

- 1.`unsafe.Pointer(&a)`方法可以得到变量`a`的地址。  
    
- 2.`(*reflect.StringHeader)(unsafe.Pointer(&a))` 可以把字符串a转成底层结构的形式。  
    
- 3.`(*[]byte)(unsafe.Pointer(&ssh))` 可以把ssh底层结构体转成byte的切片的指针。  
    
- 4.再通过 `*`转为指针指向的实际内容。


## 昨天那个在 for 循环里 append 元素的同事，今天还在么？
```go
package main

import "fmt"

func main() {
 s := []int{1,2,3,4,5}
 for _, v:=range s {
  s =append(s, v)
  fmt.Printf("len(s)=%v\n",len(s))
 }
}
```

**这个代码会造成死循环吗？**

- **不会死循环**，`for range`其实是`golang`的`语法糖`，在循环开始前会获取切片的长度 `len(切片)`，然后再执行`len(切片)`次数的循环。

## for select 时，如果通道已经关闭会怎么样？如果只有一个 case 呢？
for循环 select 时，如果其中一个case通道已经关闭，则每次都会执行到这个case。 如果select里边只有一个case，而这个case被关闭了，则**会出现死循环**。
怎么样才能不读关闭后通道
使用default


string 相关的 goroutine 泄露的问题
string[0:1]什么的，只要用了切片然后用了一个字母，父string就不会释放了

## 能说说 uintptr 和 unsafe.Pointer 的区别吗？
- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；
- 而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；
- unsafe.Pointer 可以和 普通指针 进行相互转换；
- unsafe.Pointer 可以和 uintptr 进行相互转换。

- 通过一个例子加深理解，接下来尝试用指针的方式给结构体赋值。

```go
package main

import (
 "fmt"
 "unsafe"
)

type W struct {
 b int32
 c int64
}

func main() {
 var w *W = new(W)
 //这时w的变量打印出来都是默认值0，0
 fmt.Println(w.b,w.c)

 //现在我们通过指针运算给b变量赋值为10
 b := unsafe.Pointer(uintptr(unsafe.Pointer(w)) + unsafe.Offsetof(w.b))
 *((*int)(b)) = 10
 //此时结果就变成了10，0
 fmt.Println(w.b,w.c)
}
```

- `uintptr(unsafe.Pointer(w))` 获取了 `w` 的指针`起始值`
- `unsafe.Offsetof(w.b)` 获取 `b` 变量的`偏移量`
- 两个`相加`就得到了 `b` 的`地址值`，将通用指针 `Pointer` 转换成具体指针 `((*int)(b))`，通过 `*` 符号取值，然后赋值。`*((*int)(b))` 相当于把 `(*int)(b)` 转换成 `int` 了，最后对变量重新赋值成 `10`，这样指针运算就完成了。

## reflect（反射包）如何获取字段 tag？为什么 json 包不能导出私有变量的 tag？
- `reflect.TypeOf(stru).Elem()`获取指针指向的值对应的结构体内容。
- `NumField()`可以获得该结构体含有几个字段。
- 遍历结构体内的字段，通过`t.Field(i).Tag.Get("json")`可以获取到`tag`为`json`的字段。

`json包`不能导出`私有变量的tag`是因为**取不到**`反射信息`的说法，但是直接取`t.Field(i).Tag.Get("json")`**却可以**获取到私有变量的`json字段`，是**为什么**呢？

其实**准确的**说法是，`json`包里不能导出私有变量的`tag`是因为`json`包里认为私有变量为不可导出的`Unexported`，所以**跳过获取**名为`json`的`tag`的内容。 具体可以看`/src/encoding/json/encode.go:1070`的代码。

## 协程和线程的差别
进程、线程、协程的对比如下：

- 协程既不是进程也不是线程，协程仅是一个特殊的函数。协程、进程和线程不是一个维度的。
- 一个进程可以包含多个线程，一个线程可以包含多个协程。虽然一个线程内的多个协程可以切换但是这多个协程是串行执行的，某个时刻只能有一个线程在运行，没法利用CPU的多核能力。
- 协程与进程一样，也存在上下文切换问题。
- 进程的切换者是操作系统，切换时机是根据操作系统自己的切换策略来决定的，用户是无感的。进程的切换内容包括页全局目录、内核栈和硬件上下文，切换内容被保存在内存中。 进程切换过程采用的是“从用户态到内核态再到用户态”的方式，切换效率低。
- 线程的切换者是操作系统，切换时机是根据操作系统自己的切换策略来决定的，用户是无感的。线程的切换内容包括内核栈和硬件上下文。线程切换内容被保存在内核栈中。线程切换过程采用的是“从用户态到内核态再到用户态”的方式，切换效率中等。
- 协程的切换者是用户(编程者或应用程序),切换时机是用户自己的程序来决定的。协程的切换内容是硬件上下文，切换内存被保存在用自己的变量(用户栈或堆)中。协程的切换过程只有用户态(即没有陷入内核态),因此切换效率高。


开源库里会有一些类似下面这种奇怪的用法：var _ io.Writer = (\*myWriter)(nil)，是为什么？
编译器会由此检查 `*myWriter` 类型是否实现了 `io.Writer` 接口。


开多个线程和开多个协程会有什么区别
栈小，而且遇到系统调用，p可以挪开用于其他。线程切换消耗大，协程只是应用层面切换，简单方便。

## 两个 interface {} 能不能比较
**[Go 语言](https://haicoder.net/golang/golang-tutorial.html)** 中的 **[空接口](https://haicoder.net/golang/golang-interface-nil.html)** 在保存不同的值后，可以和其他 **[变量](https://haicoder.net/golang/golang-variable.html)** 一样使用 == 进行比较操作。

### 空接口的比较特性

- **[类型](https://haicoder.net/golang/golang-datatype.html)** 不同的空接口间的比较结果不相同
- 不能比较空接口中的动态值

### 空接口的比较说明

|类 型|说 明|
|---|---|
|**[map](https://haicoder.net/golang/golang-map.html)**|不可比较，如果比较，程序会报错|
|**[切片（[]T）](https://haicoder.net/golang/golang-slice.html)**|不可比较，如果比较，程序会报错|
|**[通道（channel）](https://haicoder.net/golang/golang-channel.html)**|可比较，必须由同一个 make 生成，也就是同一个通道才会是 **[true](https://haicoder.net/golang/golang-bool.html)**，否则为 false|
|**[数组（[容量]T）](https://haicoder.net/golang/golang-array.html)**|可比较，编译期知道两个数组是否一致|
|**[结构体](https://haicoder.net/golang/golang-struct.html)**|可比较，可以逐个比较结构体的值|
|**[函数](https://haicoder.net/golang/golang-func.html)**|可比较|


### 比较两个类型相同值相同的接口

两个类型相同值相同的接口比较结果为 true

```go
package main 
import ( 	"fmt" ) 
func main() { 	
	fmt.Println("嗨客网(www.haicoder.net)") 	// 两个类型相同值相同的接口 	
	var intvalue1 interface{} = 1024 	
	var intvalue2 interface{} = 1024 	
	fmt.Println(intvalue1 == intvalue2) 
}
```

程序运行后，控制台输出如下：

![19_golang空接口比较.png](https://haicoder.net/uploads/pic/server/golang/golang-interface/19_golang%E7%A9%BA%E6%8E%A5%E5%8F%A3%E6%AF%94%E8%BE%83.png)

首先，我们定义了一个空接口 intvalue1 并赋值为 1024，同时定义了一个空接口 intvalue2 并赋值为 1024，接着，我们使用 == 比较这两个接口。

最后，我们发现输出的结果为 true，即，这两个类型相同值也相同的接口是相等的。

### 比较两个类型相同值不相同的接口

比较两个类型相同值不相同的接口结果为 false
```go
package main import ( 	"fmt" ) 
func main() { 	
	fmt.Println("嗨客网(www.haicoder.net)") 	// 比较两个类型相同值不相同的接口结果为 false 	
	var intvalue1 interface{} = 1024 	
	var intvalue2 interface{} = 10 	
	fmt.Println(intvalue1 == intvalue2) 
}
```
程序运行后，控制台输出如下：

![20_golang空接口比较.png](https://haicoder.net/uploads/pic/server/golang/golang-interface/20_golang%E7%A9%BA%E6%8E%A5%E5%8F%A3%E6%AF%94%E8%BE%83.png)

我们定义了两个类型相同，值不同的空接口，并进行比较，结果返回了 false。

### 比较两个类型不同的接口

比较两个类型不同的接口结果为 false
```go
package main 
import ( 	"fmt" ) 
func main() { 	
	fmt.Println("嗨客网(www.haicoder.net)") 	// 比较两个类型不同的接口结果为 false 	
	var intvalue1 interface{} = 1024 	
	var intvalue2 interface{} = 1024.0 	
	fmt.Println(intvalue1 == intvalue2) 
}
```

程序运行后，控制台输出如下：

![21_golang空接口比较.png](https://haicoder.net/uploads/pic/server/golang/golang-interface/21_golang%E7%A9%BA%E6%8E%A5%E5%8F%A3%E6%AF%94%E8%BE%83.png)

我们定义了两个空接口，一个保存了 **[int 类型](https://haicoder.net/golang/golang-int.html)** 的值，另一个保存了 **[float64 类型](https://haicoder.net/golang/golang-float.html)** 的值，最后，使用 == 比较这两个接口，返回 false。

### 空接口中的动态值不能比较

空接口中的动态值不能比较
```go
package main import ( 	"fmt" ) 
func main() { 	
	fmt.Println("嗨客网(www.haicoder.net)") 	// 空接口中的动态值不能比较 	
	var sliceValue1 interface{} = []uint64{1024} 	
	var sliceValue2 interface{} = []uint64{1024} 	
	fmt.Println(sliceValue1 == sliceValue2) 
}
```

程序运行后，控制台输出如下：

![22_golang空接口比较.png](https://haicoder.net/uploads/pic/server/golang/golang-interface/22_golang%E7%A9%BA%E6%8E%A5%E5%8F%A3%E6%AF%94%E8%BE%83.png)

我们定义了两个空接口，都保存了切片类型的值，并且切片中的内容完全相等，最后，我们使用 == 比较这两个切片，程序报错。

即如果空接口中保存的是动态值，那么这两个空接口是不能比较的。

### Golang空接口比较总结

类型不同的空接口间的比较结果不相同，不能比较空接口中的动态值。

## 必须要手动对齐内存的情况
有些同学可能不知道，struct 中的字段顺序不同，内存占用也有可能会相差很大。比如：

```
type T1 struct {
	a int8
	b int64
	c int16
}

type T2 struct {
	a int8
	c int16
	b int64
}
```

在 64 bit 平台上，T1 占用 24 bytes，T2 占用 16 bytes 大小；而在 32 bit 平台上，T1 占用 16 bytes，T2 占用 12 bytes 大小。**可见不同的字段顺序，最终决定 struct 的内存大小，所以有时候合理的字段顺序可以减少内存的开销**。

这是为什么呢？因为有内存对齐的存在，编译器使用了内存对齐，那么最后的大小结果就会不一样。至于为什么要做对齐，主要考虑下面两个原因：

- 平台（移植性）
    
    不是所有的硬件平台都能够访问任意地址上的任意数据。例如：特定的硬件平台只允许在特定地址获取特定类型的数据，否则会导致异常情况
    
- 性能
    
    若访问未对齐的内存，将会导致 CPU 进行两次内存访问，并且要花费额外的时钟周期来处理对齐及运算。而本身就对齐的内存仅需要一次访问就可以完成读取动作，这显然高效很多，是标准的空间换时间做法
    

> 有的小伙伴可能会认为内存读取，就是一个简单的字节数组摆放。但实际上 CPU 并不会以一个一个字节去读取和写入内存，相反 CPU 读取内存是一块一块读取的，块的大小可以为 2、4、6、8、16 字节等大小，块大小我们称其为内存访问粒度。假设访问粒度为 4，那么 CPU 就会以每 4 个字节大小的访问粒度去读取和写入内存。

在不同平台上的编译器都有自己默认的 “对齐系数”。一般来讲，我们常用的 x86 平台的系数为 4；x86_64 平台系数为 8。需要注意的是，除了这个默认的对齐系数外，还有不同数据类型的对齐系数。数据类型的对齐系数在不同平台上可能会不一致。例如，在 x86_64 平台上，int64 的对齐系数为 8，而在 x86 平台上其对齐系数就是 4。

还是拿上面的 T1、T2 来说，在 x86_64 平台上，T1 的内存布局为：

![](https://ms2008.github.io/img/in-post/memory-alignment/T1.png)

T2 的内存布局为（int16 的对齐系数为 2）：

![](https://ms2008.github.io/img/in-post/memory-alignment/T2.png)

仔细看，T1 存在许多 padding，显然它占据了不少空间。那么也就不难理解，为什么调整结构体内成员变量的字段顺序就能达到缩小结构体占用大小的疑问了，是因为巧妙地减少了 padding 的存在。让它们更 “紧凑” 了。

其实内存对齐除了可以降低内存占用之外，还有一种情况是必须要手动对齐的：**在 x86 平台上原子操作 64bit 指针。之所以要强制对齐，是因为在 32bit 平台下进行 64bit 原子操作要求必须 8 字节对齐，否则程序会 panic**。详情可以参考 [atomic](https://godoc.org/sync/atomic#pkg-note-bug) 官方文档（这么重要的信息竟然放在页面的最底部！！！😱）：

> **Bugs**
> 
> On x86-32, the 64-bit functions use instructions unavailable before the Pentium MMX. On non-Linux ARM, the 64-bit functions use instructions unavailable before the ARMv6k core. On ARM, x86-32, and 32-bit MIPS, it is the caller’s responsibility to arrange for 64-bit alignment of 64-bit words accessed atomically. The first word in a variable or in an allocated struct, array, or slice can be relied upon to be 64-bit aligned.

比如，下面这段代码：

```
package main

import (
	"sync/atomic"
)

type T3 struct {
	b int64
	c int32
	d int64
}

func main() {
	a := T3{}
	atomic.AddInt64(&a.d, 1)
}
```

编译为 64bit 可执行文件，运行没有任何问题；但是当编译为 32bit 可执行文件，运行就会 panic:

```
$ GOARCH=386 go build aligned.go
$
$ ./aligned
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x8049f2c]

goroutine 1 [running]:
runtime/internal/atomic.Xadd64(0x941218c, 0x1, 0x0, 0x809a4c0, 0x944e070)
	/usr/local/go/src/runtime/internal/atomic/asm_386.s:105 +0xc
main.main()
	/root/gofourge/src/lab/archive/aligned.go:18 +0x42
```

原因就是 T3 在 32bit 平台上是 4 字节对齐，而在 64bit 平台上是 8 字节对齐。在 64bit 平台上其内存布局为：

![](https://ms2008.github.io/img/in-post/memory-alignment/T3-x86_64.png)

可以看到编译器为了让 d 8 字节对齐，在 c 后面 padding 了 4 个字节。而在 32bit 平台上其内存布局为：

![](https://ms2008.github.io/img/in-post/memory-alignment/T3-x86.png)

编译器用的是 4 字节对齐，所以 c 后面 4 个字节并没有 padding，而是直接排列 d 的高低位字节。

为了解决这种情况，我们必须手动 padding T3，让其 “看起来” 像是 8 字节对齐的：

```
type T3 struct {
	b int64
	c int32
	_ int32
	d int64
}
```

这样 T3 的内存布局就变成了：

![](https://ms2008.github.io/img/in-post/memory-alignment/T3-x86-8.png)

看起来就像 8 字节对齐了一样，这样就能完美兼容 32bit 平台了。其实很多知名的项目，都是这么处理的，比如 [groupcache](https://github.com/golang/groupcache/blob/869f871628b6baa9cfbc11732cdf6546b17c1298/groupcache.go#L169-L172)：

```
type Group struct {
	_ int32 // force Stats to be 8-byte aligned on 32-bit platforms

	// Stats are statistics on the group.
	Stats Stats
}
```

说了这么多，但是在我们实际编码的时候，多数情况都不会考虑到最优的内存对齐。那有没有什么办法能自动检测当前的内存布局是最优呢？答案是：有的。

[golang-sizeof.tips](http://golang-sizeof.tips/) 这个网站就可以可视化 struct 的内存布局，但是只支持 8 字节对齐，是个缺点。还有一种方法，就是用 golangci-lint 做静态检测，比如在我的一个项目中检测结果是这样的：

```
$ golangci-lint run --disable-all -E maligned
config/config.go:79:11: struct of size 48 bytes could be of size 40 bytes (maligned)
type SASL struct {
          ^
```

提示有一处 struct 可以优化，来看一下这个 struct 的定义：

```
type SASL struct {
	Enable    bool
	Username  string
	Password  string
	Handshake bool
}
```

通过 [golang-sizeof.tips](http://golang-sizeof.tips/) 对比，显然字段按照下面这样排序更为合理：

```
type SASL struct {
	Username  string
	Password  string
	Handshake bool
	Enable    bool
}
```
## go 栈扩容和栈缩容，连续栈的缺点
在1.4版本之前go的协程栈管理使用[分段栈](https://link.segmentfault.com/?enc=q08Gx5GoWPuq89uHnn6qWQ%3D%3D.dGkqPV7%2BIm73qxS59o76sMJ2M7S7fuqPfqdZKDFNweotcsB%2Fi30I8q27denm6bzR)机制实现。实现方式：当检测到函数需要更多栈时，分配一块新栈，旧栈和新栈使用指针连接起来，函数返回就释放。 这样的机制存在2个问题：

- 多次循环调用同一个函数会出现“hot split”问题，例子：[stacksplit.go](https://link.segmentfault.com/?enc=6pGUh42%2B%2F6BtBSQvKds%2Bew%3D%3D.WFlbvSBgkvpo1VZj2g3k%2BR%2FS6QRksEEW50g0qQtv1HZAPSD1e4LebUoj2axhn5LocIyEUadmP0L80K9ze4FLEGFG8%2BRmMk7kQvB31NBXXoOwTEzCGfsP1z4KvT1%2FKs5RIsqmaF%2B6i%2BTPT90w6fmZPg%3D%3D)
- 每次分配和释放都要额外消耗
- 缺点：函数返回时会将新分配的内存释放回堆中，当有大量函数调用，比如递归，循环执行函数时会造成频繁创建，释放栈存储，这就是所谓的热分裂问题。

为了解决这2个问题，官方使用：**连续栈**。连续栈的实现方式：当检测到需要更多栈时，分配一块比原来大一倍的栈，把旧栈数据copy到新栈，释放旧栈。
栈的缩容主要是发生在GC期间。一个协程变成常驻状态，繁忙时需要占用很大的内存，但空闲时占用很少，这样会浪费很多内存，为了避免浪费Go在GC时对协程的栈进行了缩容，缩容也是分配一块新的内存替换原来的，大小只有原来的1/2。

连续栈虽然解决了分段栈的2个问题，但这种实现方式也会带来其他问题：

- 更多的虚拟内存碎片。尤其是你需要更大的栈时，分配一块连续的内存空间会变得更困难
- 指针会被限制放入栈。在go里面不允许二个协程的指针相互指向。这会增加实现的复杂性。


## golang 隐藏技能：怎么访问私有成员
对于一个结构体，通过 offset 函数可以获取结构体成员的偏移量，进而获取成员的地址，读写该地址的内存，就可以达到改变成员值的目的。

这里有一个内存分配相关的事实：结构体会被分配一块连续的内存，结构体的地址也代表了第一个成员的地址。

我们来看一个例子：
```golang
package main

import (
	"fmt"
	"unsafe"
)

type Programmer struct {
	name string
	language string
}

func main() {
	p := Programmer{"stefno", "go"}
	fmt.Println(p)
	
	name := (*string)(unsafe.Pointer(&p))
	*name = "qcrao"

	lang := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Offsetof(p.language)))
	*lang = "Golang"

	fmt.Println(p)
}
```
运行代码，输出：
```shell
{stefno go}
{qcrao Golang}
```
name 是结构体的第一个成员，因此可以直接将 &p 解析成 \*string。这一点，在前面获取 map 的 count 成员时，用的是同样的原理。

对于结构体的私有成员，现在有办法可以通过 unsafe.Pointer 改变它的值了。

我把 Programmer 结构体升级，多加一个字段：
```golang
type Programmer struct {
	name string
	age int
	language string
}
```
并且放在其他包，这样在 main 函数中，它的三个字段都是私有成员变量，不能直接修改。但我通过 unsafe.Sizeof() 函数可以获取成员大小，进而计算出成员的地址，直接修改内存。
```golang
func main() {
	p := Programmer{"stefno", 18, "go"}
	fmt.Println(p)

	lang := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Sizeof(int(0)) + unsafe.Sizeof(string(""))))
	*lang = "Golang"

	fmt.Println(p)
}
```
输出：
```shell
{stefno 18 go}
{stefno 18 Golang}
```


-   Redis slowlog 原理
redis的"慢查询"与redis定义慢查询的时间阈值有关，Redis提供了**slowlog-log-slower-than**和**slowlog-max-len**两个配置，**slowlog-log-slower-than**指当redis命令的执行时间超过该值时，redis会将其记录在redis的慢查询日志中，slowlog-max-len表示记录的条数（超过时会只存储最新的slowlog-max-len条），slowlog-log-slower-than的默认值为10000us，也就是一般来说，redis命令执行时间超过10ms时，我们认为产生了慢查询,redis慢查询日志记录的是执行时间，没有慢查询，并不表示客户端没有超时问题，有可能网络传输有延迟，也有可能排队的命令比较多，导致redis查询也很慢。

## 产生redis慢查询有哪些原因？

在业务使用过程中，我们发现redis使用上出现慢查询的原因有主要以下几种：

- **使用复杂度过高的命令**
- **大key问题**
- **集中过期**
所谓大key问题其实并不是值key过大，实则值key对应的value的值过大，此类问题在SET / DEL这类命令中也常出现慢查询。 首先我们要了解下redis写入与删除数据做了什么：

- 写入数据:为该数据分配内存。
- 删除数据：释放该数据对应的内存空间。 很明显，当数据值比较大时，redis分配数据内存或释放空间内存都会比较耗时，针对大key问题，有以下建议：

1. 尽量避免写入大key(不要写入无关的数据，数据实在过大可以进行拆分，通过多key存储)
2. 如果你使用的 Redis 是 4.0 以上版本，用 UNLINK 命令替代 DEL，此命令可以把释放 key 内存的操作，放到后台线程中去执行，从而降低对 Redis 的影响
3. 如果你使用的 Redis 是 6.0 以上版本，可以开启 lazy-free 机制（lazyfree-lazy-user-del = yes），在执行 DEL 命令时，释放内存也会放到后台线程中执行。

排查大key先用bigkeys扫描每个结构的最大key，然后再用命令查看占用的内存


## 10.  innodb二次写是什么
## 概念点

如图，还是先来说几个基础的概念：

- 数据库表空间由段（segment）、区(extent)、页(page)组成
    
- 默认情况下有一个共享表空间ibdata1,如使用了innodb_file_per_table则每张表独立表空间（指存放数据、索引、插入缓冲bitmap页）
    
- 段包括了数据段（B+树的叶子结点）、索引段、回滚段
    
- 区，由连续的页组成，任何情况下每个区都为1M，一个区中有64个连续页（16k）
    
- 页，数据页（B-tree Node）默认大小为16KB
    
- 文件系统一页 默认大小为4KB
    
- 盘片被分为许多扇形的区域，每个区域叫一个扇区，硬盘中每个扇区的大小固定为512字节
    
- 脏页，当数据从磁盘加载到缓冲池的数据页后，数据页内容被修改后，此数据页称为脏页
    

### 出现的问题

通过上次讲的 [重要，知识点：InnoDB的插入缓冲](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F6t0_XByG8-yuyB0YaLuuBA "https://mp.weixin.qq.com/s/6t0_XByG8-yuyB0YaLuuBA") 我们知道，脏页会在某些场景下进行刷盘，将缓冲池内的脏页数据落地到磁盘。

因为存储引擎缓冲池内的数据页大小默认为16KB，而文件系统一页大小为4KB，所以在进行刷盘操作时，就有可能发生如下场景：
![[Pasted image 20230609142232.png]]
如图所示，数据库准备刷新脏页时，需要四次IO才能将16KB的数据页刷入磁盘。

但当执行完第二次IO时，数据库发生意外宕机，导致此时才刷了2个文件系统里的页，这种情况被称为写失效（partial page write）。

此时重启后，磁盘上就是不完整的数据页，就算使用redo log也是无法进行恢复的。

**注意**：

- redo log无法恢复数据页损坏的问题，恢复必须是数据页正常并且redo log正常。
    
- 这里要知道一点，redo log中记录的是对页的物理操作，如偏移量600，写'xxxx'记录。
    
- 如果这个页本身已经发生了损坏，再对其进行重做是没有意义的
    

**该怎么解决这个问题**

那应该怎么来解决这个问题呢？其实大家想一下就会有个大概的答案，就是给它搞个备份呗。

如果写脏页的时候发生宕机，在重启后使用下备份先恢复下数据页在写磁盘就可以了，其实这就是`Double Write` 。

### Double Write 出现

千呼万唤始出来，为了防止我们可怜的数据被破坏，InnoDB存储引擎提供了重要的Double Write 特性，避免了数据丢失的惨剧发生。

下面我们来慢慢的来看看Double Write 到底是怎么提高可靠性的

#### Double Write 解决的问题

在数据库进行脏页刷新时，如果此时宕机，有可能会导致磁盘数据页损坏，丢失我们重要的数据。此时就算重做日志也是无法进行恢复的，因为重做日志记录的是对页的物理修改。

其实就是在重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本来还原该页，再进行重做，这就是double write。

#### Double Write 架构
![[Pasted image 20230609142251.png]]
如图，其实Double Write 分为了两个组成部分：

- 内存中的double write buffer
- 物理磁盘上共享表空间中连续的128个页，即2个区（extent），大小同样为2MB

可以看出，有了Double write后的脏页刷新流程就是多了几步操作：

1. 在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是会通过memcpy函数将脏页先复制到内存中的Double write buffer
    
2. 通过Double write buffer再分两次，每次1MB顺序地写入共享表空间的物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题
    

#### Double write崩溃恢复
![[Pasted image 20230609142302.png]]
如图，如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，InnoDB存储引擎可以从共享表空间中的Double write中找到该页的一个副本，将其复制到表空间文件，再应用重做日志。

下面显示了一个由Double write进行恢复的情况：

#### Double Write 的问题

Double write buffer 它是在物理文件上的一个buffer, 其实也就是file，所以它会导致系统有更多的fsync操作，而因为硬盘的fsync性能问题，所以也会影响到数据库的整体性能。

Double write页是连续的，因此这个过程是顺序写的，开销并不是很大。

在完成Double write页的写入后，再将Double write buffer中的页写入各个数据文件中，此时的写入则是离散的

## 总结

1. 当commit 一个修改语句时，如果redo log有空闲区域，直接写redo log，如果redo log没有空闲区域，那么需要把被覆盖的redo log对应的数据页刷新到data file 中，最后改pool buffer中的记录
    
2. innodb的redo log 不会记录完整的一页数据，因为这样日志太大，它只会记录那次（sequence）如何操作了（update,insert）哪页(page)的哪行(row)
    
3. 因为数据库使用的页（page，默认16KB）大小和操作系统对磁盘的操作页（page，默认4KB）不一样，当提交了一个页需要刷新到磁盘，会有多次IO， 此时刷了前面的8k时异常发生宕机。在系统恢复正常后，如果没有double write机制，此时数据库磁盘内的数据页已损坏，无法使用redo log进行恢复。
    
4. 如果有double write buffer，会检查double writer的数据的完整性，如果不完整直接丢弃double write buffer内容，重新执行那条redo log，如果double write buffer的数据是完整的，用double writer buffer的数据更新该数据页，跳过该redo log。
    


## 5.  innodb commit之前，redo 的prepare然后binlog commit，然后redo再commit有什么缺点？5.6之后是怎么优化的？
二进制日志就是binlog，事务日志就是redo日志

为了提高性能，通常会将有关联性的多个数据修改操作放在一个事务中，这样可以避免对每个修改操作都执行完整的持久化操作。这种方式，可以看作是人为的组提交(group commit)。

除了将多个操作组合在一个事务中，记录binlog的操作也可以按组的思想进行优化：将多个事务涉及到的binlog一次性flush，而不是每次flush一个binlog。

事务在提交的时候不仅会记录事务日志，还会记录二进制日志，但是它们谁先记录呢？二进制日志是MySQL的上层日志，先于存储引擎的事务日志被写入。

在MySQL5.6以前，当事务提交(即发出commit指令)后，MySQL接收到该信号进入commit prepare阶段；进入prepare阶段后，立即写内存中的二进制日志，写完内存中的二进制日志后就相当于确定了commit操作；然后开始写内存中的事务日志；最后将二进制日志和事务日志刷盘，它们如何刷盘，分别由变量 sync_binlog 和 innodb_flush_log_at_trx_commit 控制。

但因为要保证二进制日志和事务日志的一致性，在提交后的prepare阶段会启用一个**prepare_commit_mutex**锁来保证它们的顺序性和一致性。但这样会导致开启二进制日志后group commmit失效，特别是在主从复制结构中，几乎都会开启二进制日志。

在MySQL5.6中进行了改进。提交事务时，在存储引擎层的上一层结构中会将事务按序放入一个队列，队列中的第一个事务称为leader，其他事务称为follower，leader控制着follower的行为。虽然顺序还是一样先刷二进制，再刷事务日志，但是机制完全改变了：删除了原来的prepare_commit_mutex行为，也能保证即使开启了二进制日志，group commit也是有效的。

MySQL5.6中分为3个步骤：**flush阶段、sync阶段、commit阶段。**

![](https://images2018.cnblogs.com/blog/733013/201805/733013-20180508203426454-427168291.png)

- flush阶段：向内存中写入每个事务的二进制日志。
- sync阶段：将内存中的二进制日志刷盘。若队列中有多个事务，那么仅一次fsync操作就完成了二进制日志的刷盘操作。这在MySQL5.6中称为BLGC(binary log group commit)。
- commit阶段：leader根据顺序调用存储引擎层事务的提交，由于innodb本就支持group commit，所以解决了因为锁 prepare_commit_mutex 而导致的group commit失效问题。

在flush阶段写入二进制日志到内存中，但是不是写完就进入sync阶段的，而是要等待一定的时间，多积累几个事务的binlog一起进入sync阶段，等待时间由变量 binlog_max_flush_queue_time 决定，默认值为0表示不等待直接进入sync，设置该变量为一个大于0的值的好处是group中的事务多了，性能会好一些，但是这样会导致事务的响应时间变慢，所以建议不要修改该变量的值，除非事务量非常多并且不断的在写入和更新。

进入到sync阶段，会将binlog从内存中刷入到磁盘，刷入的数量和单独的二进制日志刷盘一样，由变量 sync_binlog 控制。

当有一组事务在进行commit阶段时，其他新事务可以进行flush阶段，它们本就不会相互阻塞，所以group commit会不断生效。当然，group commit的性能和队列中的事务数量有关，如果每次队列中只有1个事务，那么group commit和单独的commit没什么区别，当队列中事务越来越多时，即提交事务越多越快时，group commit的效果越明显。

**事务崩溃恢复过程如下：**

1.崩溃恢复时，扫描最后一个Binlog文件，提取其中的xid；

2.InnoDB维持了状态为Prepare的事务链表（commit两阶段提交中的第一阶段，为Prepare阶段，会把事务设置为Prepare状态）将这些事务的xid和Binlog中记录的xid做比较，如果在Binlog中存在，则提交，否则回滚事务。

通过这种方式，可以让InnoDB redo 和Binlog中的事务状态保持一致。



## 6.  go的网络IO为什么快？还有优化空间么
fasthttp 提升性能的点：

1. 控制异步 Goroutine 的同时处理数量，最大默认是 `256 * 1024`个；（这个协程池的做法提升不大）
2. 使用 sync.Pool 来大量的复用对象和切片，减少内存分配；
3. 尽量的避免`[]byte`到string转换时带来的内存分配和拷贝带来的消耗 ；p9o090u0

byte切片转string，使用string([]byte)这个方式是会分配新内存然后拷贝的。要零拷贝的话就使用unsafe包

##  对比 mysql redis http 连接池的实现 
http的实现：
成功获取 connection 分为如下几步：

1.  根据当前的请求的地址去**空闲 connection 字典**中查看存不存在空闲的 connection 列表；
2.  如果能获取到空闲的 connection 列表，那么获取到列表的最后一个 connection；
3.  返回；
当获取不到空闲 connection 时：

1.  根据当前的请求的地址去**空闲 connection 字典**中查看存不存在空闲的 connection 列表；
2.  不存在该请求的 connection 列表，那么将该 wantConn 加入到 **等待获取空闲 connection 字典**中；
在获取不到空闲连接之后，会尝试去建立连接，从上面的图大致可以看到，总共分为以下几个步骤：

1.  在调用 queueForDial 方法的时候会校验 MaxConnsPerHost 是否未设置或已达上限；
    1.  检验不通过则将当前的请求放入到 connsPerHostWait 等待字典中；
2.  如果校验通过那么会异步的调用 dialConnFor 方法创建连接；
3.  dialConnFor 方法首先会调用 dialConn 方法创建 TCP 连接，然后启动两个异步线程来处理读写数据，然后调用 tryDeliver 将连接绑定到 wantConn 上面。


sql 的排队意味着我对连接池申请连接后，把自己的编号告诉连接池。连接那边一看到有空闲了，就叫我的号。我答应了一声，然后连接池就直接给个连接给我。我如果不归还，连接池就一直不叫下一个号。
redis 这边的意思是，我去和连接池申请的不是连接而是令牌。我就一直排队等着，连接池给我令牌了，我才去仓库里面找空闲连接或者自己新建一个连接。用完了连接除了归还连接外，还得归还令牌。当然了，如果我自己新建连接出错了，我哪怕拿不到连接回家，我也得把令牌给回连接池，不然连接池的令牌数少了，最大连接数也会变小。
使用工厂函数创建全局 `Pool` 对象，首先会初始化一个 `ConnPool` 实例，初始化 `PoolSize` 大小的连接队列和轮转队列；然后在 `checkMinIdleConns` 方法中，会根据 `MinIdleConns` 参数维持一个最小连接数，以保证连接池中有这么多数量的连接处于活跃状态（按照配置的最小空闲连接数，往池中添加`MinIdleConns`个连接）

`IdleTimeout` 和 `IdleCheckFrequency` 参数用来在每过一段时间内会对连接池中不活跃的连接做清理操作，如果同时设置了，则异步开启回收连接的 goroutine，即 `reaper` 方法；再看 `ConnPool` 结构体的定义，`conns` 和 `idleConns` 这两个队列，`conns` 就是连接队列，而 `idleConns` 就是轮转队列。两者作用如下：

- `conns`：
- `idleConns`：

注意到`ConnPool.queue`这个参数，该参数的作用是当client获取连接之前，会向这个channel写入数据，如果能够写入说明不用等待就可以获取连接，否则需要等待其他goroutine从这个channel取走数据才可以获取连接，假如获取连接成功的话，在用完的时候，要向这个channel取出一个`struct{}`；不然的话就会让别人一直阻塞，go-redis用这种方法保证池中的连接数量不会超过`poolsize`；这种实现也算是最简单的令牌桶算法了


## 20.  说说 为什么用跳表 而不是红黑树 二叉树
- skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个key的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。
- 在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
- 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。
- 从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势。
- 查找单个key，skiplist和平衡树的时间复杂度都为O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的。
- 从算法实现难度上来比较，skiplist比平衡树要简单得多。
## 常量与变量

1.  下列代码的输出是：

```go
func main() {  
	const (  
		a, b = "golang", 100  
		d, e  
		f bool = true  
		g  
	)  
	fmt.Println(d, e, g)  
}  
```
  

答案

golang 100 true

在同一个 const group 中，如果常量定义与前一行的定义一致，则可以省略类型和值。编译时，会按照前一行的定义自动补全。即等价于

```
func main() {  
	const (  
		a, b = "golang", 100  
		d, e = "golang", 100  
		f bool = true  
		g bool = true  
	)  
	fmt.Println(d, e, g)  
}  
```

2.  下列代码的输出是：

```
func main() {  
	const N = 100  
	var x int = N  
  
	const M int32 = 100  
	var y int = M  
	fmt.Println(x, y)  
}  
```

答案

编译失败：cannot use M (type int32) as type int in assignment

Go 语言中，常量分为无类型常量和有类型常量两种，`const N = 100`，属于无类型常量，赋值给其他变量时，如果字面量能够转换为对应类型的变量，则赋值成功，例如，`var x int = N`。但是对于有类型的常量 `const M int32 = 100`，赋值给其他变量时，需要类型匹配才能成功，所以显示地类型转换：

```
var y int = int(M)  
```

3.  下列代码的输出是：

```
func main() {  
	var a int8 = -1  
	var b int8 = -128 / a  
	fmt.Println(b)  
}  
```

答案

-128

int8 能表示的数字的范围是 [-2^7, 2^7-1]，即 [-128, 127]。-128 是无类型常量，转换为 int8，再除以变量 -1，结果为 128，常量除以变量，结果是一个变量。变量转换时允许溢出，符号位变为1，转为补码后恰好等于 -128。

对于有符号整型，最高位是是符号位，计算机用补码表示负数。补码 = 原码取反加一。

例如：

-1 :  11111111  
00000001(原码)    11111110(取反)    11111111(加一)  
-128：      
10000000(原码)    01111111(取反)    10000000(加一)  
  
-1 + 1 = 0  
11111111 + 00000001 = 00000000(最高位溢出省略)  
-128 + 127 = -1  
10000000 + 01111111 = 11111111  

4.  下列代码的输出是：

```
func main() {  
	const a int8 = -1  
	var b int8 = -128 / a  
	fmt.Println(b)  
}  
```

答案

编译失败：constant 128 overflows int8

-128 和 a 都是常量，在编译时求值，-128 / a = 128，两个常量相除，结果也是一个常量，常量类型转换时不允许溢出，因而编译失败。

##  作用域

1.  下列代码的输出是：

```
func main() {  
	var err error  
	if err == nil {  
		err := fmt.Errorf("err")  
		fmt.Println(1, err)  
	}  
	if err != nil {  
		fmt.Println(2, err)  
	}  
}  
```

答案

1 err

`:=` 表示声明并赋值，`=` 表示仅赋值。

变量的作用域是大括号，因此在第一个 if 语句 `if err == nil` 内部重新声明且赋值了与外部变量同名的局部变量 err。对该局部变量的赋值不会影响到外部的 err。因此第二个 if 语句 `if err != nil` 不成立。所以只打印了 `1 err`。

## 1.  raft 如何实现从 follower 读取？
Raft对只读操作的处理办法是

1. 只读请求最终也必须依靠Leader来执行，如果是Follower接收请求的，那么必须转发
2. 记录下当前日志的commitIndex => readIndex
3. 执行读操作前要向集群广播一次心跳，并得到majority的反馈
4. 等待状态机的applyIndex移动过readIndex
5. 通过查询状态机来执行读操作并返回客户端最终结果。

在步骤1到步骤2之间，如果Leader刚刚当选，还必须等待no-op操作同步完成。

上面的步骤看起来很复杂，其中最重要的就是心跳广播，这是为了确认当前集群没有被网络分区。

**只读操作的进一步优化**

标准的强一致只读操作是完全是在Leader端进行的。这里可以做一步改进让只读操作主要在Follower端进行。

1. Follower接收到只读指令后，向leader索要当前的readIndex值。
2. Follower端等待自身的状态机的applyIndex移动过readIndex。
3. Follower端通过查询自身的状态机来执行读操作并返回客户端最终结果。

leader向follower返回readIndex也不是简单的直接返回，而是需要重复前面的标准步骤1~步骤3，来确认网络没有分区。这还是一次RPC操作，无法省略。但是毫无疑问，它分担了leader的压力，可以让leader有更多的资源来处理自身的读写操作。


# Go 语言与鸭子类型的关系
先直接来看维基百科里的定义：

> If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.

翻译过来就是：如果某个东西长得像鸭子，像鸭子一样游泳，像鸭子一样嘎嘎叫，那它就可以被看成是一只鸭子。

`Duck Typing`，鸭子类型，是动态编程语言的一种对象推断策略，它更关注对象能如何被使用，而不是对象的类型本身。Go 语言作为一门静态语言，它通过通过接口的方式完美支持鸭子类型。

例如，在动态语言 python 中，定义一个这样的函数：

1.  `def hello_world(coder):`
2.      `coder.say_hello()`

当调用此函数的时候，可以传入任意类型，只要它实现了 `say_hello()` 函数就可以。如果没有实现，运行过程中会出现错误。

而在静态语言如 Java, C++ 中，必须要显示地声明实现了某个接口，之后，才能用在任何需要这个接口的地方。如果你在程序中调用 `hello_world` 函数，却传入了一个根本就没有实现 `say_hello()` 的类型，那在编译阶段就不会通过。这也是静态语言比动态语言更安全的原因。

动态语言和静态语言的差别在此就有所体现。静态语言在编译期间就能发现类型不匹配的错误，不像动态语言，必须要运行到那一行代码才会报错。插一句，这也是我不喜欢用 `python` 的一个原因。当然，静态语言要求程序员在编码阶段就要按照规定来编写程序，为每个变量规定数据类型，这在某种程度上，加大了工作量，也加长了代码量。动态语言则没有这些要求，可以让人更专注在业务上，代码也更短，写起来更快，这一点，写 python 的同学比较清楚。

Go 语言作为一门现代静态语言，是有后发优势的。它引入了动态语言的便利，同时又会进行静态语言的类型检查，写起来是非常 Happy 的。Go 采用了折中的做法：不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。

来看个例子：

先定义一个接口，和使用此接口作为参数的函数：

1.  `type IGreeting interface {`
2.      `sayHello()`
3.  `}`

5.  `func sayHello(i IGreeting) {`
6.      `i.sayHello()`
7.  `}`

再来定义两个结构体：

1.  `type Go struct {}`
2.  `func (g Go) sayHello() {`
3.      `fmt.Println("Hi, I am GO!")`
4.  `}`

6.  `type PHP struct {}`
7.  `func (p PHP) sayHello() {`
8.      `fmt.Println("Hi, I am PHP!")`
9.  `}`

最后，在 main 函数里调用 sayHello() 函数：

1.  `func main() {`
2.      `golang := Go{}`
3.      `php := PHP{}`

5.      `sayHello(golang)`
6.      `sayHello(php)`
7.  `}`

程序输出：

1.  `Hi, I am GO!`
2.  `Hi, I am PHP!`

在 main 函数中，调用调用 sayHello() 函数时，传入了 `golang, php` 对象，它们并没有显式地声明实现了 IGreeting 类型，只是实现了接口所规定的 sayHello() 函数。实际上，编译器在调用 sayHello() 函数时，会隐式地将 `golang, php` 对象转换成 IGreeting 类型，这也是静态语言的类型检查功能。

顺带再提一下动态语言的特点：

> 变量绑定的类型是不确定的，在运行期间才能确定函数和方法可以接收任何类型的参数，且调用时不检查参数类型不需要实现接口

总结一下，鸭子类型是一种动态语言的风格，在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由它”当前方法和属性的集合”决定。Go 作为一种静态语言，通过接口实现了 `鸭子类型`，实际上是 Go 的编译器在其中作了隐匿的转换工作。

# ## 7. 有了 GC，为什么还会发生内存泄露？
在一个具有 GC 的语言中，我们常说的内存泄漏，用严谨的话来说应该是：预期的能很快被释放的内存由于附着在了长期存活的内存上、或生命期意外地被延长，导致预计能够立即回收的内存而长时间得不到回收。

在 Go 中，由于 goroutine 的存在，所谓的内存泄漏除了附着在长期对象上之外，还存在多种不同的形式。

### 形式1：预期能被快速释放的内存因被根对象引用而没有得到迅速释放

### 形式2：goroutine 泄漏

## 11. 触发 GC 的时机是什么？

Go 语言中对 GC 的触发时机存在两种形式：

1.  **主动触发**，通过调用 runtime.GC 来触发 GC，此调用阻塞式地等待当前 GC 运行完毕。
    
2.  **被动触发**，分为两种方式：
    
    -   使用系统监控，当超过两分钟没有产生任何 GC 时，强制触发 GC。
        
    -   使用步调（Pacing）算法，其核心思想是控制内存增长的比例。
        

通过 `GOGC` 或者 `debug.SetGCPercent` 进行控制（他们控制的是同一个变量，即堆的增长率 $\rho$）。整个算法的设计考虑的是优化问题：如果设上一次 GC 完成时，内存的数量为 $H_m$（heap marked），估计需要触发 GC 时的堆大小 $H_T$（heap trigger），使得完成 GC 时候的目标堆大小 $H_g$（heap goal） 与实际完成时候的堆大小 $H_a$（heap actual）最为接近，即： $\min |H_g - H_a| = \min|(1+\rho)H_m - H_a|$。

![gc-pacing](https://static.sitestack.cn/projects/qcrao-Go-Questions/GC/assets/gc-pacing.png)

除此之外，步调算法还需要考虑 CPU 利用率的问题，显然我们不应该让垃圾回收器占用过多的 CPU，即不应该让每个负责执行用户 goroutine 的线程都在执行标记过程。理想情况下，在用户代码满载的时候，GC 的 CPU 使用率不应该超过 25%，即另一个优化问题：如果设 $u_g$为目标 CPU 使用率（goal utilization），而 $u_a$为实际 CPU 使用率（actual utilization），则 $\min|u_g - u_a|$。

## 13. GC 关注的指标有哪些？

Go 的 GC 被设计为成比例触发、大部分工作与赋值器并发、不分代、无内存移动且会主动向操作系统归还申请的内存。因此最主要关注的、能够影响赋值器的性能指标有：

-   CPU 利用率：回收算法会在多大程度上拖慢程序？有时候，这个是通过回收占用的 CPU 时间与其它 CPU 时间的百分比来描述的。
-   GC 停顿时间：回收器会造成多长时间的停顿？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿。
-   GC 停顿频率：回收器造成的停顿频率是怎样的？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿。
-   GC 可扩展性：当堆内存变大时，垃圾回收器的性能如何？但大部分的程序可能并不一定关心这个问题。

## 14. Go 的 GC 如何调优？

Go 的 GC 被设计为极致简洁，与较为成熟的 Java GC 的数十个可控参数相比，严格意义上来讲，Go 可供用户调整的参数只有 GOGC 环境变量。当我们谈论 GC 调优时，通常是指减少用户代码对 GC 产生的压力，这一方面包含了减少用户代码分配内存的数量（即对程序的代码行为进行调优），另一方面包含了最小化 Go 的 GC 对 CPU 的使用率（即调整 GOGC）。

GC 的调优是在特定场景下产生的，并非所有程序都需要针对 GC 进行调优。只有那些对执行延迟非常敏感、当 GC 的开销成为程序性能瓶颈的程序，才需要针对 GC 进行性能调优，几乎不存在于实际开发中 99% 的情况。除此之外，Go 的 GC 也仍然有一定的可改进的空间，也有部分 GC 造成的问题，目前仍属于 Open Problem。

总的来说，我们可以在现在的开发中处理的有以下几种情况：

1.  对停顿敏感：GC 过程中产生的长时间停顿、或由于需要执行 GC 而没有执行用户代码，导致需要立即执行的用户代码执行滞后。
2.  对资源消耗敏感：对于频繁分配内存的应用而言，频繁分配内存增加 GC 的工作量，原本可以充分利用 CPU 的应用不得不频繁地执行垃圾回收，影响用户代码对 CPU 的利用率，进而影响用户代码的执行效率。

从这两点来看，所谓 GC 调优的核心思想也就是充分的围绕上面的两点来展开：优化内存的申请速度，尽可能的少申请内存，复用已申请的内存。或者简单来说，不外乎这三个关键字：**控制、减少、复用**。

我们将通过三个实际例子介绍如何定位 GC 的存在的问题，并一步一步进行性能调优。当然，在实际情况中问题远比这些例子要复杂，这里也只是讨论调优的核心思想，更多的时候也只能具体问题具体分析。

### 例1：合理化内存分配的速度、提高赋值器的 CPU 利用率
### 例2：降低并复用已经申请的内存

### 例3：调整 GOGC

我们已经知道了 GC 的触发原则是由步调算法来控制的，其关键在于估计下一次需要触发 GC 时，堆的大小。可想而知，如果我们在遇到海量请求的时，为了避免 GC 频繁触发，是否可以通过将 GOGC 的值设置得更大，让 GC 触发的时间变得更晚，从而减少其触发频率，进而增加用户代码对机器的使用率呢？答案是肯定的。

我们可以非常简单粗暴的将 GOGC 调整为 1000，来执行上一个例子中未复用对象之前的程序：
### 例3：调整setmemoryLimit和禁用GogC

## 15. Go 的垃圾回收器有哪些相关的 API？其作用分别是什么？

在 Go 中存在数量极少的与 GC 相关的 API，它们是

-   runtime.GC：手动触发 GC
-   runtime.ReadMemStats：读取内存相关的统计信息，其中包含部分 GC 相关的统计信息
-   debug.FreeOSMemory：手动将内存归还给操作系统
-   debug.ReadGCStats：读取关于 GC 的相关统计信息
-   debug.SetGCPercent：设置 GOGC 调步变量
-   debug.SetMaxHeap（尚未发布[10]）：设置 Go 程序堆的上限值

## epoll、select、poll 区别

select 机制刚开始的时候，需要把 fd_set 从`用户空间拷贝到内核空间`，并且检测的 fd 数是有限制的，由 `FD_SETSIZE` 设置，一般是1024。`数组实现`。

poll 的实现和 select 非常相似，只是描述 fd集合 的方式不同，poll使用 `pollfd结构`而不是 select的 fd_set 结构，其他的都差不多。`链表实现`。

epol l引入了 `epoll_ctl系统调用`，将高频调用的 epoll_wait 和低频的 epoll_ctl 隔离开。epoll_ctl 通过(EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL)三个操作来分散对需要监控的fds集合的修改，做到了有变化才变更，**将select或poll高频、大块内存拷贝(集中处理)变成epoll_ctl的低频、小块内存的拷贝(分散处理)，避免了大量的内存拷贝。** epoll使用 `红黑树` 来组织监控的fds集合

## epoll 的水平触发和边缘触发的区别

Edge Triggered (ET) 边沿触发：

1.  **socket 的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件。**
2.  **socket 的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件。**
3.  **仅在缓冲区状态变化时触发事件。**

Level Triggered (LT) 水平触发：

1.  **socket 接收缓冲区`不为空`，有数据可读，则读事件一直触发。**
2.  **socket 发送缓冲区`不满可以继续写入数据`，则写事件一直触发。**

## TCP 的流量控制

因为我们总希望数据传输的更快一些。但如果`发送方把数据发得过快`，接收方就可能`来不及接收`，这就会造成数据的丢失。流量控制（flow control）就是**让发送方的发送速率不要太快，要让接收方来得及接收。**

利用`滑动窗口机制`可以很方便地在 TCP连接上实现**发送方流量控制**。通过接收方的**确认报文中的窗口字段**，发送方能够`准确地控制发送字节数。`

## 为什么有了流量控制还要有拥塞控制?

流量控制是避免发送方的数据填满接收方的缓存，但并不知道网络中发生了什么。

计算机网络是处在一个**共享**的环境中。因此也有可能会发生网络的拥堵。在网络出现拥堵时，如果继续发送大量的数据包，可能会导致数据包`时延`、`丢失`，这时 TCP 就会`重传数据`，但是⼀重传就会导致网络的负担更重，于是会导致`更大的延迟以及更多的丢包`。

控制的目的就是**避免发送方的数据填满整个网络**。为了在发送方调节所要发送数据的数据量，定义了⼀个叫做「拥塞窗口」的概念。**拥塞窗口 cwnd 是发送方维护的⼀个的状态变量，它会根据网络的拥塞程度动态变化的。**

## TCP 不是可靠传输吗？为什么会丢包呢？

TCP的可靠传输是会在丢包的时候进行重传，来形成可靠的传输，丢包这是网络的问题，而不是TCP机制的问题。

## 那你介绍一下拥塞控制的算法？

拥塞控制一共有四个算法：

1.  慢启动：TCP 在刚建立连接完成后，首先是有个慢启动的过程，这个慢启动的意思就是⼀点⼀点的提高发送数据包的数量。慢启动的算法规则：当`发送方每收到⼀个 ACK，拥塞窗⼝ cwnd 的大小就会加 1`。
2.  拥塞避免：当拥塞窗口 cwnd「超过」`慢启动门限 ssthresh`就会进⼊拥塞避免算法。那么进⼊拥塞避免算法后，它的规则是：**每当收到⼀个 ACK 时，cwnd 增加 1/cwnd**。
3.  快重传：当接收方发现丢了⼀个中间包的时候，`发送三次前⼀个包的ACK`，**于是发送端就会快速地重传，不必等待超时再重传。**
4.  快恢复：快重传和快恢复⼀般同时s使用，快速恢复算法是认为，你还能`收到 3 个重复的 ACK`说明网络也不那么糟糕，所以没有必要像 RTO 超时那么强烈。**进⼊快速恢复之前， cwnd 和 ssthresh 已被更新了：cwnd = cwnd/2 ，也就是设置为原来的⼀半; ssthresh = cwnd 。**

## 进程、线程的区别

进程

线程

系统中正在运行的一个应用程序

系统分配处理器时间资源的基本单元

程序一旦运行就是进程

进程之内独立执行的一个单元执行流

资源分配的最小单位

程序执行的最小单位

进程有`独立的地址空间`，一个进程崩溃后，在`保护模式下不会对其它进程产生影响`，而线程只是一个进程中的不同执行路径。

线程有自己的**堆栈和局部变量**，但线程`没有单独的地址空间`，一个线程死掉就等于整个进程死掉，所以多进程的程序要比多线程的程序健壮，但在`进程切换`时，耗费资源较大，效率要差一些。但对于一些 **`要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程`**

## Go里面GMP模型是怎么样的？

G：表示goroutine，存储了goroutine的执行stack信息、goroutine状态以及goroutine的任务函数等；另外G对象是`可以重用`的。  
P：表示逻辑processor，P 的数量决定了系统内最大可并行的 G 的数量（前提：系统的物理cpu核数 >= P的数量）；P的最大作用还是其拥有的各种G对象队列、链表、一些cache和状态。  
M：M 代表着真正的执行计算资源，物理 Processor。

> G 如果想运行起来必须依赖 P，因为 P 是它的`逻辑处理单元`，但是 P 要想真正的运行，他也需要与 M 绑定，这样才能真正的运行起来，`P 和 M 的这种关系就相当于 Linux 系统中的用户层面的线程和内核的线程是一样的`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/ced7c30900e94e9db88417483971df45.png)

## 你用的是 mysql 是吧，那 B树 和 B+树 的区别是？
1.  B+ 树内节点不存储数据，所有 data 存储在叶节点导致查询时间复杂度固定为`log (n)`。而 B-树 查询时间复杂度不固定，与 key 在树中的位置有关，最好为 `O(1)`。
2.  B+ 树叶节点两两相连可大大增加区间访问性，可使用在范围查询等，而 B- 树 每个节点 key 和 data 在一起，则无法区间查找。同时我们访问某数据后，可以将`相近的数据预读进内存`，这是利用了`空间局部性`。
3.  B+ 树更适合`外部存储`。由于内节点无 data 域，每个节点能索引的范围更大更精确。

## 介绍一下死锁产生的必要条件

互斥条件，请求和保持条件，不剥夺条件，环路等待条件。

## 如何实现互斥锁？

一个互斥锁需要`有阻塞和唤醒功能`，所以需要一下几个情况。

1.  需要有一个标记锁状态的 state 变量。
2.  需要记录哪个线程持有了锁。
3.  需要有一个队列维护所有的线程。

## 如何实现自旋锁？

> 当时真不会，只磕磕绊绊说了一些自旋锁的东西。。

大家可以看这篇博客 [自旋锁C++实现方式](https://blog.csdn.net/weixin_44228698/article/details/108561517)

## kafka 和 其他消息队列，比如 rocketmq，rabbitmq ，有什么优势？

kafka 具备非常高可靠的分布式架构，功能简单，只支持一些主要的MQ功能，像一些消息查询，消息回溯等功能没有提供。时效是在ms级别以内的。

## kafka如何保证消息不丢失？

那我们可以从两个方面也就是生产者，消费者以及 broker ，来说说

1.  生产者：生产者(Producer) 调用`send 方法`发送消息之后，消息可能因为`网络问题并没有发送过去`。Kafka提供了`同步发送消息方法`，会返回一个`Future对象`，调用`get()`方法进行阻塞等待，就可以知道消息是否发送成功。如果消息发送失败的话，可以通过`Producer 的 retries（重试次数）`参数进行设置，当出现网络问题之后能够`自动重试消息发送，避免消息丢失`。另外，建议还要设置`重试间隔`，因为间隔太小的话重试的效果就不明显。
2.  消费者：消息在被追加到 Partition(分区) 的时候都会分配一个特定的偏移量（offset）。**偏移量（offset)表示 Consumer 当前消费到的 Partition(分区)的所在的位置。** Kafka 通过偏移量（offset）可以保证消息在分区内的顺序性。`当消费者拉取到了分区的某个消息之后，消费者会自动提交了 offset`。自动提交的话会有一个问题，试想一下，**当消费者刚拿到这个消息准备进行真正消费的时候，突然挂掉了，消息实际上并没有被消费，但是 offset 却被自动提交了。** 可以通过`enable.auto.commit设置为false`，关闭`自动提交 offset`，每次在真正消费完消息之后之后再自己`手动提交 offset`。
3.  broker：当消息发送到了分区的leader副本，leader 副本所在的 broker 突然挂掉，那么就要从 follower 副本重新选出一个 leader ，但是 leader 的数据还有一些没有被 follower 副本的同步的话，就会造成消息丢失。解决办法就是 `将producer设置 acks = all`。

## https为什么是安全的？

因为 https 是基于 ssl 和 tls 加密而成的，https 的 s 就代表 ssl 和 tls。

## ssl/tls 是怎么保证安全的？经过几次握手？

经过四次握手，客户端向服务器端索要并验证公钥，双方协商生成"对话密钥"，双方采用"对话密钥"进行加密通信。  
四次握手主要是交换以下信息：

1.  数字证书：该证书包含了公钥等信息，一般是由服务器发给客户端，接收方通过验证这个证书是不是由信赖的CA签发，或者与本地的证书相对比，来判断证书是否可信；假如需要双向验证，则服务器和客户端都需要发送数字证书给对方验证；
2.  三个随机数：这三个随机数构成了后续通信过程中用来对数据进行对称加密解密的“对话密钥”。

> 首先客户端先发第一个随机数N1，然后服务器回了第二个随机数N2（这个过程同时把之前`提到的证书发给客户端`），这两个随机数都是明文的；而第三个随机数N3，客户端用`数字证书的公钥进行非对称加密`，发给服务器；  
> 而服务器用只有自己知道的私钥来解密，获取第三个随机数。  
> 这样，服务端和客户端都有了三个随机数N1+N2+N3，然后两端就使用这三个随机数来生成“对话密钥”，`在此之后的通信都是使用这个“对话密钥”来进行对称加密解密`。  
> 因为这个过程中，服务端的私钥只用来解密第三个随机数，从来没有在网络中传输过，这样的话，只要`私钥没有被泄露`，那么数据就是安全的。

3.  加密通信协议：就是双方商量使用哪一种加密方式，假如两者支持的加密方式不匹配，则无法进行通信；

![在这里插入图片描述](https://img-blog.csdnimg.cn/155fe83484794257898c79e18d31b583.png)

## 事务的四大特性？

原子性、隔离性、一致性、持久性

sync.map context select的源码
https://qcrao.com/post/dive-into-go-sync-map/
https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/
https://mp.weixin.qq.com/s/9-eLJqYZrOpNLoiTGpBrKA

# Reference
https://mp.weixin.qq.com/s/yA-O2258GtSV5I8ol9LfSg
https://zhuanlan.zhihu.com/p/519979757
https://learnku.com/articles/56078
https://geektutu.com/post/qa-golang-c1.html
https://www.bookstack.cn/read/qcrao-Go-Questions/spilt.13.GC-GC.md