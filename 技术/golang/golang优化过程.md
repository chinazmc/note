# holmes

基于规则的自动Golang Profile Dumper.

作为一名"懒惰"的程序员，如何避免在线上Golang系统半夜宕机 （一般是OOM导致的）时起床保存现场呢？又或者如何dump压测时性能尖刺时刻的profile文件呢？

holmes 或许能帮助您解决以上问题。

## 设计

holmes 每隔一段时间收集一次以下应用指标：

- 协程数，通过`runtime.NumGoroutine`。
- 当前应用所占用的RSS，通过[gopsutil](https://github.com/shirou/gopsutil)第三方库。
- CPU使用率，比如8C的机器，如果使用了4C，则使用率为50%，通过[gopsutil](https://github.com/shirou/gopsutil)第三方库。

除此之外，holmes还会根据Gc周期收集RSS指标，如果您开启了`GCheap dump`的话。

在预热阶段（应用启动后，holmes会收集十次指标）结束后，holmes会比较当前指标是否满足用户所设置的阈值/规则，如果满足的话，则dump profile， 以日志或者二进制文件的格式保留现场。

## 如何使用


```shell
    go get mosn.io/holmes
```

在应用初始化逻辑加上对应的holmes配置。

```go
func main() {
	
  h := initHolmes()
  
  // start the metrics collect and dump loop
  h.Start()
  ......
  
  // quit the application and stop the dumper
  h.Stop()
}
func initHolmes() *Holmes{
    h, _ := holmes.New(
    holmes.WithCollectInterval("5s"),
    holmes.WithDumpPath("/tmp"),
    holmes.WithCPUDump(20, 25, 80, time.Minute),
    holmes.WithCPUMax(90),
    )
    h.EnableCPUDump()
    return h
}
```

holmes 支持对以下几种应用指标进行监控:

```
* mem: 内存分配     
* cpu: cpu使用率      
* thread: 线程数    
* goroutine: 协程数
* gcHeap: 基于GC周期的内存分配
```

### Dump Goroutine profile

```go
h, _ := holmes.New(
    holmes.WithCollectInterval("5s"),
    holmes.WithDumpPath("/tmp"),
    holmes.WithTextDump(),
    holmes.WithDumpToLogger(true),
    holmes.WithGoroutineDump(10, 25, 2000, 10*1000, time.Minute),
)
h.EnableGoroutineDump()

// start the metrics collect and dump loop
h.Start()

// stop the dumper
h.Stop()
```

- WithCollectInterval("5s") 每5s采集一次当前应用的各项指标，该值建议设置为大于1s。
- WithDumpPath("/tmp") profile文件保存路径。
- WithTextDump() 以文本格式保存profile内容。
- WithDumpToLogger() profile内容将会输出到日志。
- WithGoroutineDump(10, 25, 2000, 100\*1000, time.Minute) 当goroutine指标满足以下条件时，将会触发dump操作。 current_goroutine_num > `10` && current_goroutine_num < `100*1000` && current_goroutine_num > `125`% * previous_average_goroutine_num or current_goroutine_num > `2000`. `time.Minute` 是两次dump操作之间最小时间间隔，避免频繁profiling对性能产生的影响。

> WithGoroutineDump(min int, diff int, abs int, max int, coolDown time.Duration) 当应用所启动的goroutine number大于`Max` 时，holmes会跳过dump操作，因为当goroutine number很大时， dump goroutine profile操作成本很高（STW && dump），有可能拖垮应用。当`Max`=0 时代表没有限制。

### Dump cpu profile

```go
h, _ := holmes.New(
holmes.WithCollectInterval("5s"),
holmes.WithDumpPath("/tmp"),
holmes.WithCPUDump(20, 25, 80, time.Minute),
holmes.WithCPUMax(90),
)
h.EnableCPUDump()

// start the metrics collect and dump loop
h.Start()

// stop the dumper
h.Stop()
```

- WithCollectInterval("5s") 每5s采集一次当前应用的各项指标，该值建议设置为大于1s。
- WithDumpPath("/tmp") profile文件保存路径。
- cpu profile支持保存文件，不支持输出到日志中，所以WithBinaryDump()和 WithTextDump()在这场景会失效。
- WithCPUDump(10, 25, 80, time.Minute) 会在满足以下条件时dump profile cpu usage > `10%` && cpu usage > `125%` * previous cpu usage recorded or cpu usage > `80%`. `time.Minute` 是两次dump操作之间最小时间间隔，避免频繁profiling对性能产生的影响。
- WithCPUMax 当cpu使用率大于`Max`, holmes会跳过dump操作，以防拖垮系统。

### Dump Heap Memory Profile

```go
h, _ := holmes.New(
    holmes.WithCollectInterval("5s"),
    holmes.WithDumpPath("/tmp"),
    holmes.WithTextDump(),
    holmes.WithMemDump(30, 25, 80, time.Minute),
)

h.EnableMemDump()

// start the metrics collect and dump loop
h.Start()

// stop the dumper
h.Stop()
```

- WithCollectInterval("5s") 每5s采集一次当前应用的各项指标，该值建议设置为大于1s。
- WithDumpPath("/tmp") profile文件保存路径。
- WithTextDump() profile的内容将会输出到日志中。
- WithMemDump(30, 25, 80, time.Minute) 会在满足以下条件时抓取heap profile memory usage > `10%` && memory usage > `125%` * previous memory usage or memory usage > `80%`， `time.Minute` 是两次dump操作之间最小时间间隔，避免频繁profiling对性能产生的影响。

### 基于Gc周期的Heap Memory Dump

在一些场景下，我们无法通过定时的`Memory Dump`保留到现场, 比如应用在一个`CollectInterval`周期内分配了大量内存， 又快速回收了它们，此时`holmes`在周期前后的采集到内存使用率没有产生过大波动，与实际情况不符。为了解决这种情况，`holmes`开发了基于GC周期的 `Profile`类型，它会在内存使用率飙高的前后两个GC周期内各dump一次profile，然后开发人员可以使用`pprof --base`命令去对比 两个时刻堆内存之间的差异。 [具体实现介绍](https://uncledou.site/2022/go-pprof-heap/)。

```go
	h, _ := holmes.New(
		holmes.WithDumpPath("/tmp"),
		holmes.WithLogger(holmes.NewFileLog("/tmp/holmes.log", mlog.INFO)),
		holmes.WithBinaryDump(),
		holmes.WithMemoryLimit(100*1024*1024), // 100MB
		holmes.WithGCHeapDump(10, 20, 40, time.Minute),
		// holmes.WithProfileReporter(reporter),
	)
	h.EnableGCHeapDump().Start()
	time.Sleep(time.Hour)
```

### 动态设置holmes配置

您可以通过`Set`在系统运行时更新holmes的配置。它的使用十分简单，和初始化时的`New`方法一样。

```go
    h.Set(
        WithCollectInterval("2s"),
        WithGoroutineDump(10, 10, 50, 90, time.Minute))
```

### Dump事件上报

您可以通过实现`Reporter` 来实现以下功能：

- 发送包含现场的告警信息，当`holmes`触发`Dump`操作时。
- 将`Profiles`上传到其他地方，以防实例被销毁，从而导致profile丢失，或进行分析。

```go
        type ReporterImpl struct{}
        func (r *ReporterImpl) 	Report(pType string, filename string, reason ReasonType, eventID string, sampleTime time.Time, pprofBytes []byte, scene Scene) error{ 
			// do something	
        }
        ......
        r := &ReporterImpl{} // a implement of holmes.ProfileReporter Interface.
    	h, _ := holmes.New(
            holmes.WithProfileReporter(reporter),
            holmes.WithDumpPath("/tmp"),
            holmes.WithLogger(holmes.NewFileLog("/tmp/holmes.log", mlog.INFO)),
            holmes.WithBinaryDump(),
            holmes.WithMemoryLimit(100*1024*1024), // 100MB
            holmes.WithGCHeapDump(10, 20, 40, time.Minute),
)
  
```

### 开启全部


holmes当然不是只支持一个类型的dump啦，您可以按需选择您需要的dump类型。

```go
h, _ := holmes.New(
    holmes.WithCollectInterval("5s"),
    holmes.WithDumpPath("/tmp"),
    holmes.WithTextDump(),

    holmes.WithCPUDump(10, 25, 80, time.Minute),
    holmes.WithMemDump(30, 25, 80, time.Minute),
    holmes.WithGCHeapDump(10, 20, 40, time.Minute),
    holmes.WithGoroutineDump(500, 25, 20000, 0, time.Minute),
)

    h.EnableCPUDump().
    EnableGoroutineDump().
	EnableMemDump().
	EnableGCHeapDump().Start()
```

### 在docker 或者cgroup环境下运行 holmes

```go
h, _ := holmes.New(
    holmes.WithCollectInterval("5s"),
    holmes.WithDumpPath("/tmp"),
    holmes.WithTextDump(),

    holmes.WithCPUDump(10, 25, 80, time.Minute),
    holmes.WithCGroup(true), // set cgroup to true
)
```

## 已知风险

Gorountine dump 会导致 STW，[从而导致时延](https://github.com/golang/go/issues/33250)。

> 目前Go官方已经有一个[CL](https://go-review.googlesource.com/c/go/+/387415/)在优化这个问题了。


# 计算密集型服务 性能优化实战始末

## 背景

### 项目背景

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9J4NAbEHy096EYh9xr2FGpb7icWgeIGia56F49kT8wuVZFicUY1SHaJV1xcJeTWib3t922t8H2ufyFN1A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

worker 服务数据链路图  

worker 服务消费上游数据（工作日高峰期产出速度达近 **200 MB/s**，节假日高峰期可达 **300MB/s** 以上），进行中间处理后，写入多个下游。在实践中结合业务场景，基于**快慢隔离**的思想，以三个不同的 consumer group 消费同一 Topic，隔离三种数据处理链路。

### 面对问题

- worker 服务在高峰期时 CPU Idle 会降至 60%，因其属于数据处理类计算密集型服务，CPU Idle 过低会使服务吞吐降低，在数据处理上产生较大延时，且受限于 Kafka 分区数，无法进行横向扩容；
- 对上游数据的采样率达 **30%**，业务方对数据的完整性有较大诉求，但系统 CPU 存在瓶颈，无法满足；

## 性能优化

针对以上问题，开始着手对服务 CPU Idle 进行优化；抓取服务 pprof profile 图如下：go tool pprof -http=:6061 http://「ip:port」/debug/pprof/profile
![[Pasted image 20240806131249.png]]
优化前的 pprof profile 图  

### 服务与存储之间置换压力

#### 背景

worker 服务消费到上游数据后，会先写全部写入 Apollo 的数据库，之后每分钟定时捞取处理，但消息体大小 P99 分位达近 **1.5M**，对于 Apollo 有较严重的大 Key 问题，再结合 RocksDB 特有的写放大问题，会进一步加剧存储压力。在这个背景下，我们采用 zlib 压缩算法，对消息体先进行压缩后再写入 Apollo，减缓读写大 Key 对 Apollo 的压力。

#### 优化

在 CPU 的优化过程中，我们发现服务在压缩操作上占用了较多的 CPU，于是对压缩等级进行调整，以减小压缩率、增大下游存储压力为代价，减少压缩操作对服务 CPU 的占用，提升服务 CPU 。这一操作本质上是在服务与存储之间进行压力置换，是一种**空间换时间**的做法。

#### 关于压缩等级

这里需要特别注意的是，在压缩等级的设置上可能存在较为严重的**边际效用递减**问题。在进行基准测试时发现，将压缩等级由 BestCompression 调整为 DefaultCompression 后，压缩率只有近 **1‱** 的下降，但压缩方面的 CPU 占用却相对提高近 **50%**。此结论不能适用于所有场景，需从实际情况出发，但在使用时应注意这个问题，选择相对最优的压缩方式。
![[Pasted image 20240806131341.png]]
zlib 可设置的压缩等级  

### 使用更高效的序列化库

#### 背景

worker 服务在设计之初基于快慢隔离的思想，使用三个不同的 consumer group 进行分开消费，导致对同一份数据会重复消费三次，而上游产出的数据是在 PB 序列化之后写入 Kafka，消费侧亦需要进行 PB 反序列化方能使用，因此导致了 PB 反序列化操作在 CPU 上的较大开销。

#### 优化

经过探讨和调研后发现，gogo/protobuf 三方库相较于原生的 golang/protobuf 库性能更好，在 CPU 上占用更低，速度更快，因此采用 gogo/protobuf 库替换掉原生的 golang/protobuf 库。

#### gogo/protobuf 为什么快

- 通过对每一个字段都生成代码的方式，**取消了对反射的使用**；
- 采用预计算方式，在序列化时能够减少内存分配次数，进而**减少了内存分配**带来的系统调用、锁和 GC 等代价。

> 用过去或未来换现在的时间：页面静态化、池化技术、预编译、代码生成等都是提前做一些事情，用过去的时间，来降低用户在线服务的响应时间；另外对于一些在线服务非必须的计算、存储的耗时操作，也可以异步化延后进行处理，这就是用未来的时间换现在的时间。出处：https://mp.weixin.qq.com/s/S8KVnG0NZDrylenIwSCq8g

#### 关于序列化库

这里只列举的了 PB 序列化库的优化 Case，但在 JSON 序列化方面也存在一样的优化手段，如 json-iterator、sonic、gjson 等等，我们在 Feature 服务中先后采用了 json-iterator 与 gjson 库替换原有的标准库 JSON 序列化方式，均取得了显著效果。JSON 库调研报告：https://segmentfault.com/a/1190000041591284
![[Pasted image 20240806131438.png]]
调整压缩等级与更换 PB 序列化库之后  

### 数据攒批 减少调用

#### 背景

在观察 pprof 图后发现写 hbase 占用了近 50% 的相对 CPU，经过进一步分析后，发现每次在序列化一个字段时 Thrift 都会调用一次 socket->syscall，带来频繁的上下文切换开销。

#### 优化

阅读代码后发现， 原代码中使用了 Thrift 的 TTransport 实现，其功能是包装 TSocket，裸调 Syscall，每次 Write 时都会调用 socket 写入进而调用 Syscall。这与通常我们的编码习惯不符，认为应该有一个 buffer 充当中间层进行数据攒批，当 buffer 写完或者写满后再向下层写入。于是进一步阅读 Thrift 源码，发现其中有多种 Transport 实现，而 TTBufferedTransport 是符合我们编码习惯的。
![[Pasted image 20240806131525.png]]
对 Thrift 调用进行优化，使用带 buffer 的 transport，大大减少对 Syscall的调用
![[Pasted image 20240806131547.png]]
更换 transport 之后，对 HBase 的调用消耗只剩最右侧的一条了  
![[Pasted image 20240806131606.png]]
对 HBase 的访问耗时大幅下降  

#### Thrift Client 部分源码分析

Transport 使用了装饰器模式

|Transport 实现|作用|
|---|---|
|TTransport|包装 TSocket，裸调 Syscall，每次 Write 都会调用 syscall；|
|TTBufferedTransport|需要提前声明 buffer 的大小，在调用 Socket 之前加了一层 buffer，写满或者写完之后再调用 Syscall；|
|TFramedTransport|与 TTBufferedTransport 类似，但只会在全部写入 buffer 后，再调用 Syscall。数据格式为：size+content，客户端与服务端必须都使用该实现，否则会因为格式不兼容报错；|
|streamTransport|传入自己实现的 IO 接口；|
|TMemoryBufferTransport|纯内存交换，不与网络交互；|

Protocol

  

|Protocol 实现|作用|
|---|---|
|TBinaryProtocol|直接的二进制格式；|
|TCompactProtocol|紧凑型、高效和压缩的二进制格式；|
|TJSONProtocol|JSON 格式；|
|TSimpleJSONProtocol|SimpleJSON 产生的输出适用于 AJAX 或脚本语言，它不保留Thrift的字段标签，不能被 Thrift 读回，它不应该与全功能的 TJSONProtocol 相混淆；https://cwiki.apache.org/confluence/display/THRIFT/ThriftUsageJava|

#### 关于数据攒批

数据攒批：将数据先写入用户态内存中，而后统一调用 syscall 进行写入，常用在数据落盘、网络传输中，可降低系统调用次数、利用磁盘顺序写特性等，是一种空间换时间的做法。有时也会牺牲一定的数据实时性，如 kafka producer 侧。相似优化可见：https://mp.weixin.qq.com/s/ntNGz6mjlWE7gb_ZBc5YeA

### 语法调整

除在对库的使用上进行优化外，在 GO 语言本身的使用上也存在一些优化方式；

- slice、map 预初始化，减少频繁扩容导致的内存拷贝与分配开销；
- 字符串连接使用 strings.builder(预初始化) 代替 fmt.Sprintf()；
```go
func ConcatString(sl ...string) string {
   n := 0
   for i := 0; i < len(sl); i++ {
      n += len(sl[i])
   }

   var b strings.Builder
   b.Grow(n)
   for _, v := range sl {
      b.WriteString(v)
   }
   return b.String()
}

```

- buffer 修改返回 string([]byte) 操作为 []byte，减少内存 []byte -> string 的内存拷贝开销；
![[Pasted image 20240806131649.png]]

- string <-> []byte 的另一种优化，需确保 []byte 内容后续不会被修改，否则会发生 panic；
```go
func String2Bytes(s string) []byte {
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh := reflect.SliceHeader{
        Data: sh.Data,
        Len:  sh.Len,
        Cap:  sh.Len,
    }
    return *(*[]byte)(unsafe.Pointer(&bh))
}

func Bytes2String(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}


```

#### 关于语法调整

更多语法调整，见以下文章

- https://www.bacancytechnology.com/blog/golang-performance
    
- https://mp.weixin.qq.com/s/Lv2XTD-SPnxT2vnPNeREbg
    

### GC 调优

#### 背景

在上次优化完成之后，系统已经基本稳定，CPU Idle 高峰期也可以维持在 80% 左右，但后续因业务诉求对上游数据采样率调整至 100%，CPU.Idle 高峰期指标再次下降至近 70%，且由于定时任务的问题，存在 CPU.Idle 掉 0 风险；

#### 优化

经过对 pprof 的再次分析，发现 runtime.gcMarkWorker 占用不合常理，达到近 30%，于是开始着手对 GC 进行优化；
![[Pasted image 20240806131856.png]]
GC 优化前 pprof 图  

##### 方法一：使用 sync.pool()

通常来说使用 sync.pool() 缓存对象，减少对象分配数，是优化 GC 的最佳方式，因此我们在项目中使用其对 bytes.buffer 对象进行缓存复用，意图减少 GC 开销，但实际上线后 **CPU Idle 却略微下降，且 GC 问题并无缓解**。原因有二：

1. sync.pool 是全局对象，读写存在竞争问题，因此在这方面会消耗一定的 CPU，但之所以通常用它优化后 CPU 会有提升，是因为它的对象复用功能对 GC 带来的优化，因此 sync.pool 的优化效果取决于锁竞争增加的 CPU 消耗与优化 GC 减少的 CPU 消耗这两者的差值；
    
2. GC 压力的大小通常取决于 inuse_objects，与 inuse_heap 无关，也就是说与正在使用的对象数有关，与正在使用的堆大小无关；

本次优化时选择对 bytes.buffer 进行复用，是想做到减少堆大小的分配，出发点错了，对 GC 问题的理解有误，对 GC 的优化因从 pprof heap 图 inuse_objects 与 alloc_objects 两个指标出发。
![[Pasted image 20240806131945.png]]
甚至没有依赖经验， 只是单纯的想当然了🤦‍♂️  

##### 方法二：设置 GOGC

**原理**：GOGC 默认值是 100，也就是下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍，通过调大 GOGC 值（gcpercent）的方式，达到减少 GC 次数的目的；

> 公式：gc_trigger = heap_marked * (1+gcpercent/100) gcpercent：通过 GOGC 来设置，默认是 100，也就是当前内存分配到达上次存活堆内存 2 倍时，触发 GC；heap_marked：上一个 GC 中被标记的(存活的)字节数；

**问题**：GOGC 参数不易控制，设置较小提升有限，设置较大容易有 OOM 风险，因为堆大小本身是在实时变化的，在任何流量下都设置一个固定值，是一件有风险的事情。这个问题目前已经有解决方案，Uber 发表的文章中提到了一种自动调整 GOGC 参数的方案，用于在这种方式下优化 GO 的 GC CPU 占用，不过业界还没有开源相关实现
![[Pasted image 20240806132020.png]]
设置 GOGC 至 1000% 后，GC 占用大幅缩小
![[Pasted image 20240806132039.png]]
内存利用率接近 100%
![[Pasted image 20240806132050.png]]
##### 方法三：GO ballast 内存控制

> ballast 压舱物--航海。为提供所需的吃水和稳定性而临时或永久携带在船上的重型材料。来源：Dictionary.com

**原理**：仍然是从利用了下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍这一原理，初始化一个生命周期贯穿整个 Go 应用生命周期的超大 slice，用于内存占位，增大 heap_marked 值降低 GC 频率；实际操作有以下两种方式

> 公式：gc_trigger = heap_marked * (1+gcpercent/100) gcpercent：通过 GOGC 来设置，默认是 100，也就是当前内存分配到达上次存活堆内存 2 倍时，触发 GC；heap_marked：上一个 GC 中被标记的(存活的)字节数；
![[Pasted image 20240806132114.png]]
方式一
![[Pasted image 20240806132126.png]]
方式二  

两种方式都可以达到同样的效果，但是方式一会实际占用物理内存，在可观测性上会更舒服一点，方式二并不会实际占用物理内存。

> 原因：Memory in ‘nix (and even Windows) systems is virtually addressed and mapped through page tables by the OS. When the above code runs, the array the ballast slice points to will be allocated in the program’s virtual address space. Only if we attempt to read or write to the slice, will the page fault occur that causes the physical RAM backing the virtual addresses to be allocated. 引用自：https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/
![[Pasted image 20240806132206.png]]
优化后 GC 频率由每秒 5 次降低到了每秒 0.1 次
![[Pasted image 20240806132243.png]]
使用 ballast 内存控制后，GC 占用缩小至红框大小
![[Pasted image 20240806132232.png]]
使用方式一后，内存始终稳定在 25% -30%，即 3G 大小  

**相比于设置 GOGC 的优势**

- 安全性更高，OOM 风险小；
    
- 效果更好，可以从 pprof 图看出，后者的优化效果更大；
    

**负面考量**问：虽然通过大切片占位的方式可以有效降低 GC 频率，但是每次 GC 需要扫描和回收的对象数量变多了，是否会导致进行 GC 的那一段时间产生耗时毛刺？答：不会，GC 有两个阶段 mark 与 sweep，unused_objects 只与 sweep 阶段有关，但这个过程是非常快速的；mark 阶段是 GC 时间占用最主要的部分，但其只与当前的 inuse_objects 有关，与 unused_objects 无太大关系；因此，综上所述，降低频率确实会让每次 GC 时的 unused_objects 有所增长，但并不会对 GC 增加太多负担；

> 关于 ballast 内存控制更详细的内容请看：https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/

#### 关于 GC 调优

- GC 优化手段的优先级：设置 GOGC、GO ballast 内存控制等操作是一种治标不治本略显 trick 的方式，在做 GC 优化时还应先从对象复用、减少对象分配角度着手，在确无优化空间或优化成本较大时，再选择此种方式；
    
- 设置 GOGC、GO ballast 内存控制等操作本质上也是一种空间换时间的做法，在内存与 CPU 之间进行压力置换；
    
- 在 GC 调优方面，还有很多其他优化方式，如 bigcache 在堆内定义大数组切片自行管理、fastcache 直接调用 syscall.mmap 申请堆外内存使用、offheap 使用 cgo 管理堆外内存等等。
    

## 优化效果
![[Pasted image 20240806132356.png]]
黄色线：调整压缩等级与更换 PB 序列化库；绿色线：Thrift 序列化更换带 buffer 的 transport；
![[Pasted image 20240806132413.png]]
蓝色曲线抖动是因为上游业务放量，后又做垂直伸缩将 CPU 由 8 核提至 16 核 GC 优化（红框部分）  

- 先后将 CPU 提升 **25%、10%**（假设不做伸缩）；
    
- 支持上游数据 **100%** 放量；
    
- 通过对 CPU 瓶颈的解决，顺利合并服务，下掉 **70 台**容器。
    

## 总结

**经验分享**

- 做性能优化经验很重要，其次在优化之前掌握一部分前置知识更好；
    
- 平时多看一些资料学习，有优化机会就抓住实践，避免书到用时方恨少；
    
- 仔细观察 pprof 图，分析大块部分；
    
- 观察问题点的 api 使用，可能具有更高效的使用方式
    
- 记录优化过程和优化效果，以后分享、吹逼用的上；
    
- 最好可以构建稳定的基准环境，验证效果；
    
- 空间换时间是万能钥匙，工程问题不要只 case by case 的看，很多解决方案都是同一种思想的不同落地，尝试去总结和掌握这种思想，最后达到迁移复用的效果；
    
- 多和大佬讨论，非常重要，以上多项优化都出自与大佬（特别鸣谢 @李小宇@曹春晖）讨论后的实践；
    
# IO 密集型服务 性能优化实战记录

## 背景

### 项目背景

Feature 服务作为特征服务，产出特征数据供上游业务使用。服务压力：高峰期 API 模块 10wQPS，计算模块 20wQPS。服务本地缓存机制：

- 计算模块有本地缓存，且命中率较高，最高可达 50% 左右；
- 计算模块本地缓存在每分钟第 0 秒会全部失效，而在此时流量会全部击穿至下游 Codis；
- Codis 中 Key 名 = 特征名 + 地理格子 Id + 分钟级时间串；

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAlqkibcA9FDIxSAnGDZCfKWVOsj6VmHmmho3ibxKibSXC8Jic6eEsmq1d7Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

                             Feature 服务模块图

### 面对问题

服务 API 侧存在较严重的 P99 耗时毛刺问题（固定出现在每分钟第 0-10s），导致上游服务的访问错误率达到 1‰ 以上，影响到业务指标；目标：解决耗时毛刺问题，将 P99 耗时整体优化至 15ms 以下；![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAEvFFyicbBh0hKmIcym63NMJZybCIeIhP8lK9cMFHZBibDBwoR0SfFuCA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)                               API 模块返回上游 P99 耗时图

## 解决方案

### 服务 CPU 优化

#### 背景

偶然的一次上线变动中，发现对 Feature 服务来说 CPU 的使用率的高低会较大程度上影响到服务耗时，因此从**提高服务 CPU Idle** 角度入手，对服务耗时毛刺问题展开优化。

#### 优化

通过对 Pprof profile 图的观察发现 JSON 反序列化操作占用了较大比例（50% 以上），因此通过减少反序列化操作、更换 JSON 序列化库（json-iterator）两种方式进行了优化。
#### 效果

收益：CPU idle 提升 5%，P99 耗时毛刺从 30ms 降低至 **20 ms 以下**。
![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAxDWTcia2v3En2kvibucqRQtj2W9Q56IC0nicwsPOUXK9kg8bmgd6eqSdw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)优化后的耗时曲线（红色与绿色线）

#### 关于 CPU 与耗时

**为什么 CPU Idle 提升耗时会下降**

- 反序列化时的开销减少，使单个请求中的计算时间得到了减少；
- 单个请求的处理时间减少，使同时并发处理的请求数得到了减少，减轻了调度切换、协程/线程排队、资源竞争的开销；

#### 关于 json-iterator 库

**json-iterator 库为什么快**标准库 json 库使用 reflect.Value 进行取值与赋值，但 reflect.Value 不是一个可复用的反射对象，每次都需要按照变量生成 reflect.Value 结构体，因此性能很差。json-iterator 实现原理是用 reflect.Type 得出的类型信息通过「对象指针地址+字段偏移」的方式直接进行取值与赋值，而不依赖于 reflect.Value，reflect.Type 是一个可复用的对象，同一类型的 reflect.Type 是相等的，因此可按照类型对 reflect.Type 进行 cache 复用。总的来说其作用是**减少内存分配**和**反射调用次数**，进而减少了内存分配带来的系统调用、锁和 GC 等代价，以及使用反射带来的开销。

> 详情可见：https://cloud.tencent.com/developer/article/1064753

### 调用方式优化 - 对冲请求

#### 背景

1. Feature 服务 API 模块访问计算模块 P99 显著高于 P95；![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyArzDQiaiacfI9G4TJlrD8Tcvk2kRwnBxcH6oNVrkiaGUiczDuzP9gmkjgsw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)API 模块访问计算模块 P99 与 P95 耗时曲线
    
2. 经观察计算模块不同机器之间毛刺出现时间点不同，单机毛刺呈**偶发现象**，所有机器聚合看**呈规律性**毛刺；![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAHALNSicOLKUkYOz7CvE8ictoBlcMnG0eO2fGeA05jXrqF8gwpmrwAYHw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)计算模块返回 API P99 耗时曲线（未聚合）![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAricDY8PzgEeZNFgibvS2r8YcpsRIbe8A4ecfhye5XZTzLicKC0icBVceYw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)计算模块返回 API P99 耗时曲线（均值聚合）

#### 优化

1. 针对 P99 高于 P95 现象，提出对冲请求方案，对毛刺问题进行优化；

> 对冲请求：把对下游的一次请求拆成两个，先发第一个，n毫秒超时后，发出第二个，两个请求哪个先返回用哪个；Hedged requests. A simple way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever replica responds first. We term such requests “hedged requests” because a client first sends one request to the replica be- lieved to be the most appropriate, but then falls back on sending a secondary request after some brief delay. The cli- ent cancels remaining outstanding re- quests once the first result is received. Although naive implementations of this technique typically add unaccept- able additional load, many variations exist that give most of the latency-re- duction effects while increasing load only modestly. One such approach is to defer send- ing a secondary request until the first request has been outstanding for more than the 95th-percentile expected la- tency for this class of requests. This approach limits the additional load to approximately 5% while substantially shortening the latency tail. The tech- nique works because the source of la- tency is often not inherent in the par- ticular request but rather due to other forms of interference. 摘自：论文《The Tail at Scale》

2. 调研

- 阅读论文 Google《The Tail at Scale》；
- 开源实现：BRPC、RPCX；
- 工业实践：百度默认开启、Grab LBS 服务（下游纯内存型数据库）效果非常明显、谷歌论文中也有相关的实践效果阐述；

4. 落地实现：修改自 RPCX 开源实现
```go
package backuprequest


import (
 "sync/atomic"
 "time"

 "golang.org/x/net/context"
)


var inflight int64

// call represents an active RPC.
type call struct {
 Name  string
 Reply interface{} // The reply from the function (*struct).
 Error error       // After completion, the error status.
 Done  chan *call  // Strobes when call is complete.
}


func (call *call) done() {
 select {
 case call.Done <- call:
 default:
  logger.Debug("rpc: discarding Call reply due to insufficient Done chan capacity")
 }
}


func BackupRequest(backupTimeout time.Duration, fn func() (interface{}, error)) (interface{}, error) {
 ctx, cancelFn := context.WithCancel(context.Background())
 defer cancelFn()
 callCh := make(chan *call, 2)
 call1 := &call{Done: callCh, Name: "first"}
 call2 := &call{Done: callCh, Name: "second"}


 go func(c *call) {
  defer helpers.PanicRecover()
  c.Reply, c.Error = fn()
  c.done()
 }(call1)


 t := time.NewTimer(backupTimeout)
 select {
 case <-ctx.Done(): // cancel by context
  return nil, ctx.Err()
 case c := <-callCh:
  t.Stop()
  return c.Reply, c.Error
 case <-t.C:
  go func(c *call) {
   defer helpers.PanicRecover()
   defer atomic.AddInt64(&inflight, -1)
   if atomic.AddInt64(&inflight, 1) > BackupLimit {
    metric.Counter("backup", map[string]string{"mark": "limited"})
    return
   }

   metric.Counter("backup", map[string]string{"mark": "trigger"})
   c.Reply, c.Error = fn()
   c.done()
  }(call2)
 }


 select {
 case <-ctx.Done(): // cancel by context
  return nil, ctx.Err()
 case c := <-callCh:
  metric.Counter("backup_back", map[string]string{"call": c.Name})
  return c.Reply, c.Error
 }
}

```

#### 效果

收益：P99 耗时整体从 20-60ms 降低至 6ms，毛刺全部干掉；（backupTimeout=5ms）
![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAhXEHUwAUao8ZhdhHq9zxMr3OTwwWh0jagJB1Vy4Jh0aVxlqLBPDekQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)API 模块返回上游服务耗时统计图

#### 《The Tail at Scale》论文节选及解读

括号中内容为个人解读**为什么存在变异性？（高尾部延迟的响应时间）**

- 导致服务的个别部分出现高尾部延迟的响应时间的变异性（耗时长尾的原因）可能由于许多原因而产生，包括：
- 共享的资源。机器可能被不同的应用程序共享，争夺共享资源（如CPU核心、处理器缓存、内存带宽和网络带宽）（在云上环境中这个问题更甚，如不同容器资源争抢、Sidecar 进程影响）；在同一个应用程序中，不同的请求可能争夺资源。
- 守护程序。后台守护程序可能平均只使用有限的资源，但在安排时可能产生几毫秒的中断。
- 全局资源共享。在不同机器上运行的应用程序可能会争夺全球资源（如网络交换机和共享文件系统（数据库））。
- 维护活动。后台活动（如分布式文件系统中的数据重建，BigTable等存储系统中的定期日志压缩（此处指 LSM Compaction 机制，基于 RocksDB 的数据库皆有此问题），以及垃圾收集语言中的定期垃圾收集（自身和上下游都会有 GC 问题 1. Codis proxy 为 GO 语言所写，也会有 GC 问题；2. 此次 Feature 服务耗时毛刺即时因为服务本身 GC 问题，详情见下文）会导致周期性的延迟高峰；以及排队。中间服务器和网络交换机的多层排队放大了这种变化性。
    

**减少组件的可变性**

- 后台任务可以产生巨大的CPU、磁盘或网络负载；例子是面向日志的存储系统的日志压缩和垃圾收集语言的垃圾收集器活动。
    
- 通过节流、将重量级的操作分解成较小的操作（例如 GO、Redis rehash 时渐进式搬迁），并在整体负载较低的时候触发这些操作（例如某数据库将 RocksDB Compaction 操作放在凌晨定时执行），通常能够减少后台活动对交互式请求延迟的影响。
    

**关于消除变异源**

- 消除大规模系统中所有的延迟变异源是不现实的，特别是在共享环境中。
    
- 使用一种类似于容错计算的方法（此处指对冲请求），容尾软件技术从不太可预测的部分中形成一个可预测的整体（对下游耗时曲线进行建模，从概率的角度进行优化）。
    
- 一个真实的谷歌服务的测量结果，该服务在逻辑上与这个理想化的场景相似；根服务器通过中间服务器将一个请求分发到大量的叶子服务器。该表显示了大扇出对延迟分布的影响。在根服务器上测量的单个随机请求完成的第99个百分点的延迟是10ms。然而，所有请求完成的第99百分位数延迟是140ms，95%的请求完成的第99百分位数延迟是70ms，这意味着等待最慢的5%的请求完成的时间占总的99%百分位数延迟的一半。专注于这些慢速异常值的技术可以使整体服务性能大幅降低。
    
- 同样，由于消除所有的变异性来源也是不可行的，因此正在为大规模服务开发尾部容忍技术。尽管解决特定的延迟变异来源的方法是有用的，但最强大的尾部容错技术可以重新解决延迟问题，而不考虑根本原因。这些尾部容忍技术允许设计者继续为普通情况进行优化，同时提供对非普通情况的恢复能力。
    

#### 对冲请求原理

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAz6YzBhUlGLOicXwiaeo8TLZddM4HHYl1OtN9lDtiasthSRroxhNic5VBfQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)对冲请求典型场景

- 其原理是从概率角度出发，利用下游服务的耗时模型，在这条耗时曲线上任意取两个点，其中一个小于x的概率，这个概率远远大于任意取一个点小于x的概率，所以可以极大程度降低耗时；
- 但如果多发的请求太多了，比如说1倍，会导致下游压力剧增，耗时曲线模型产生恶化，达不到预期的效果，如果控制比如说在5%以内，下游耗时曲线既不会恶化，也可以利用他95分位之前的那个平滑曲线，因此对冲请求超时时间的选择也是一个需要关注的点；
- 当超过95分位耗时的时候，再多发一个请求，这时候这整个请求剩余的耗时就取决于在这整个线上任取一点，和在95分位之后的那个线上任取一点，耗时是这两点中小的那个，从概率的角度看，这样95分位之后的耗时曲线，会比之前平滑相当多；
- 这个取舍相当巧妙，只多发5%的请求，就能基本上直接干掉长尾情况；
- 局限性

- 如同一个 mget 接口查 100 个 key 与查 10000 个 key 耗时一定差异很大，这种情况下对冲请求时无能为力的，因此需要保证同一个接口请求之间质量是相似的情况下，这样下游的耗时因素就不取决于请求内容本身；
    
- 如 Feature 服务计算模块访问 Codis 缓存击穿导致的耗时毛刺问题，在这种情况下对冲请求也无能为力，甚至一定情况下会恶化耗时；
    
- 请求需要幂等，否则会造成数据不一致；
    
- 总得来说对冲请求是从**概率的角度消除偶发因素的影响**，从而解决长尾问题，因此需要考量耗时是否为业务侧自身固定因素导致，举例如下：
    
- 对冲请求超时时间并非动态调整而是人为设定，因此极端情况下会有雪崩风险，解决方案见一下小节；
    

> 名称来源 backup request 好像是 BRPC 落地时候起的名字，论文原文里叫 Hedged requests，简单翻译过来是对冲请求，GRPC 也使用的这个名字。

#### 关于雪崩风险

对冲请求超时时间并非动态调整，而是人为设定，因此极端情况下会有雪崩风险；

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyADW2KvvW05q926sb4GSz4H84EK6DZKCRhia9RharVkBXkOfib2OqG4LXQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

摘自《Google SRE》 如果不加限制确实会有雪崩风险，有如下解法

- BRPC 实践：对冲请求会消耗一次对下游的重试次数；![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAAgIbadHUOUSbkNicFz5JobIRKu2OU0fF20QTZpFOzlBZ58GefBIbcOg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
- bilibili 实践：
    
- 对 retry 请求下游会阻断级联；
- 本身要做熔断；
- 在 middleware 层实现窗口统计，限制重试总请求占比，比如 1.1 倍；
- 服务自身对下游实现熔断机制，下游服务对上游流量有限流机制，保证不被打垮。从两方面出发保证服务的稳定性；
- Feature 服务实践：对每个对冲请求在发出和返回时增加 atmoic 自增自减操作，如果大于某个值（请求耗时 ✖️ QPS ✖️ 5%），则不发出对冲请求，从控制并发请求数的角度进行流量限制；
    

### 语言 GC 优化

#### 背景

在引入对冲请求机制进行优化后，在耗时方面取得了突破性的进展，但为从根本上解决耗时毛刺，优化服务内部问题，达到标本兼治的目的，着手对服务的耗时毛刺问题进行最后的优化；

#### 优化

**第一步：观察现象，初步定位原因**对 Feature 服务早高峰毛刺时的 Trace 图进行耗时分析后发现，在毛刺期间程序 GC pause 时间（GC 周期与任务生命周期重叠的总和）长达近 50+ms（见左图），绝大多数 goroutine 在 GC 时进行了长时间的辅助标记（mark assist，见右图中浅绿色部分），GC 问题严重，因此怀疑耗时毛刺问题是由 GC 导致；

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyA4t4VibkJa9hrdlUNXzzb1OBq9eL5FlXceicGZOIoOCvsIQyzOF486qQw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAKda3AKnjNkdBC2ODoGLNPjo4CrOLueoictynhMe8pXP7z8a7jicTmcIw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**第二步：从原因出发，进行针对性分析**

- 根据观察计算模块服务平均每 10 秒发生 2 次 GC，GC 频率较低，但在每分钟前 10s 第一次与第二次的 GC 压力大小（做 mark assist 的 goroutine 数）呈明显差距，因此怀疑是在每分钟前 10s 进行第一次 GC 时的压力过高导致了耗时毛刺。
- 根据 Golang GC 原理分析可知，G 被招募去做辅助标记是因为该 G 分配堆内存太快导致，而 计算模块每分钟缓存失效机制会导致大量的下游访问，从而引入更多的对象分配，两者结合互相印证了为何在每分钟前 10s 的第一次 GC 压力超乎寻常；

> 关于 GC 辅助标记 mark assist 为了保证在Marking过程中，其它G分配堆内存太快，导致Mark跟不上Allocate的速度，还需要其它G配合做一部分标记的工作，这部分工作叫辅助标记(mutator assists)。在Marking期间，每次G分配内存都会更新它的”负债指数”(gcAssistBytes)，分配得越快，gcAssistBytes越大，这个指数乘以全局的”负载汇率”(assistWorkPerByte)，就得到这个G需要帮忙Marking的内存大小(这个计算过程叫revise)，也就是它在本次分配的mutator assists工作量(gcAssistAlloc)。引用自：https://wudaijun.com/2020/01/go-gc-keypoint-and-monitor/

**第三步：按照分析结论，设计优化操作**从减少对象分配数角度出发，对 Pprof heap 图进行观察

- 在 inuse_objects 指标下 cache 库占用最大；
- 在 alloc_objects 指标下 json 序列化占用最大；

但无法确定哪一个是真正使分配内存增大的因素，因此着手对这两点进行分开优化；

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAsSicAlsu64icxPhhgSHRc5Nt69x46ETBjdsibFOf8rDrqMNiaYp7icWnCZg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAsvHiaQVpcgB6MOS29sAvneDfZCkZPjn70ibvicxZ6WicOK7Hsic8Dd9JG6w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)通过对业界开源的 json 和 cache 库调研后（调研报告：https://segmentfault.com/a/1190000041591284），采用性能较好、低分配的 GJSON 和 0GC 的 BigCache 对原有库进行替换；

  

#### 效果

- 更换 JSON 序列化库 GJSON 库优化无效果；
- 更换 Cache 库 BigCache 库效果明显，inuse_objects 由 200-300w 下降到 12w，毛刺基本消失；
![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAFlXJyibAicX4tmU5R50989VzZG4ZP858obWxdX1cPs55iaRodaYic1icpvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)计算模块耗时统计图（浅色部分：GJSON，深色部分：BigCache）
![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAg7Vs67qPAibiciaopHHlQ9ic1dxmp9wE4W32JfbUBHBWfkOOsrUxjSftIQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)API 模块返回上游耗时统计图

#### 关于 Golang GC

在通俗意义上常认为，GO GC 触发时机为堆大小增长为上次 GC 两倍时。但在 GO GC 实际实践中会按照 Pacer 调频算法根据堆增长速度、对象标记速度等因素进行预计算，使堆大小在达到两倍大小前提前发起 GC，最佳情况下会只占用 25% CPU 且在堆大小增长为两倍时，刚好完成 GC。

> 关于 Pacer 调频算法：https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/pacing/

但 Pacer 只能在稳态情况下控制 CPU 占用为 25%，一旦服务内部有瞬态情况，例如定时任务、缓存失效等等，Pacer 基于稳态的预判失效，导致 GC 标记速度小于分配速度，为达到 GC 回收目标（在堆大小到达两倍之前完成 GC），会导致大量 Goroutine 被招募去执行 Mark Assist 操作以协助回收工作，从而阻碍到 Goroutine 正常的工作执行。因此目前 GO GC 的 Marking 阶段对耗时影响时最为严重的。

> 关于 gc pacer 调频器
> 
> ![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAIbPjxRwicIYQhClpEIB8ovHnEiave2YCgiaQqYdvgldTt6Gb7BjVUicnpA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)引用自：https://go.googlesource.com/proposal/+/a216b56e743c5b6b300b3ef1673ee62684b5b63b/design/44167-gc-pacer-redesign.md
> 
>   

## 最终效果

API 模块 P99 耗时从 20-50ms 降低至 6ms，访问错误率从 1‰ 降低到 1‱。

![图片](https://mmbiz.qpic.cn/mmbiz_png/YdZzofiato9LJXrEicIt094ztVYzNoAxyAtx8SG3Yfce1LbAWZePfRCNiam9Hwvhwu8njibTRUY3SAVB3oWWbydSwQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)API 返回上游服务耗时统计图

  

## 总结

1. 当分析耗时问题时，观察监控或日志后，可能会发现趋势完全匹配的两种指标，误以为是因果关系，但却有可能这两者都是外部表现，共同受到**第三变量的影响，相关但不是因果**；
    
2. 相对于百毫秒耗时服务，低延时服务的耗时会较大程度上受到 **CPU 使用率**的影响，在做性能优化时切勿忽视这点；（线程排队、调度损耗、资源竞争等）
    
3. 对于高并发、低延时服务，耗时方面受到下游的影响可能只是一个方面，服务**自身开销**如序列化、GC 等都可能会较大程度上影响到服务耗时；
    
4. 性能优化因从**提高可观测性**入手，如链路追踪、标准化的 Metric、go pprof 工具等等，打好排查基础，基于多方可靠数据进行分析与猜测，最后着手进行优化、验证，避免盲人摸象似的操作，妄图通过碰运气的方式解决问题；
    
5. 了解一些简单的**建模知识**对耗时优化的分析与猜测阶段会有不错的帮助；
    
6. **理论结合实际问题**进行思考；多看文章、参与分享、进行交流，了解更多技术，扩展视野；每一次的讨论和质疑都是进一步深入思考的机会，以上多项优化都出自与大佬（特别鸣谢 @李心宇@刘琦@龚勋）的讨论后的实践；
    
7. 同为性能优化，耗时优化不同于 CPU、内存等资源优化，更加复杂，难度较高，在做资源优化时 Go 语言自带了方便易用的 PProf 工具，可以提供很大的帮助，但耗时优化尤其是长尾问题的优化非常艰难，因此在进行优化的过程中一定要稳住心态、耐心观察，**行百里者半九十**。
    
8. 关注请求之间**共享资源**的争用导致的耗时问题，不仅限于下游服务，服务自身的 CPU、内存（引发 GC）等也是共享资源的一部分；
    


# Reference
https://mp.weixin.qq.com/s/aIKNqQAaI37iJPEEi9z8YQ
https://mp.weixin.qq.com/s/83M0j8lIALdF-eOdD5GukA