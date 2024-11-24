
# 01 为什么要进行性能优化

对 Golang 程序进行性能优化，可以在提升业务收益的同时，起到降低成本的作用。笔者在做一次代码重构时发现过一个问题，DeepCopy 占据了大量 CPU 时间，其处理逻辑如下：  

```
x1 := DeepCopy(x)       // 对x进行deep copy
Modify(x)                       // 对x进行修改
Read(x1)                        // 读取旧x
.........
```

我们完全可以通过简单业务逻辑调整，比如调整处理的先后顺序等移除DeepCopy。优化前后性能对比如下：
![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe95nic51Nzh4xWn97sPrnzIzD1diaCIy2RHZGn6OficeuZUwDOPbdX3bO6OZ4QZqMOysOOEpI0zOXMWoQ/640?wx_fmt=png&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1)

性能有5倍左右提升，折算到成本上的收益是巨大的。  

# 02 Go 中如何对性能进行度量与分析
####    2.1 Benchmark
**Benchmark 示例**

```go
func BenchmarkConvertReflect(b *testing.B) {
var v interface{} = int32(64)
for i:=0;i<b.N;i++{
        f := reflect.ValueOf(v).Int()
if f != int64(64){
            b.Error("errror")
        }
    }
}
```

> 函数固定以 Benchmark 开头，其位于_test.go 文件中，入参为 testing.B 业务逻辑应放在 for 循环中，因为 b.N 会依次取值 1, 2, 3, 5, 10, 20, 30, 50,100.........，直至执行时间超过 1s

可以运行 go test -bench 命令执行 benchmark，其结果如下：
```
➜  gotest666 go test -bench='BenchmarkConvertReflect' -run=none
goos: darwin
goarch: amd64
pkg: gotest666
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkConvertReflect-12      520200014            2.291 ns/op
```  

> `--bench='BenchmarkConvertReflect'`， 要执行的 benchmark。需注意:该参数支持模糊匹配，如--bench='Get|Set' ，支持./...`-run=none`，只进行 Benchmark，不执行单测

BenchmarkConvertReflect, 在12核下，1s内执行了520200014次，每次约2.291ns。

**高级用法**

```bash
➜  gotest666 go test -bench='Convert' -run=none -benchtime=2s -count=3 -cpu='2,4' -benchmem -cpuprofile=cpu.profile -memprofile=mem.profile -blockprofile=blk.profile -trace=trace.out -gcflags=all=-l
goos: darwin
goarch: amd64
pkg: gotest666
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkConvertReflect-2       1000000000           2.286 ns/op           0 B/op          0 allocs/op
BenchmarkConvertReflect-2       1000000000           2.302 ns/op           0 B/op          0 allocs/op
BenchmarkConvertReflect-2       1000000000           2.239 ns/op           0 B/op          0 allocs/op
BenchmarkConvertReflect-4       1000000000           2.244 ns/op           0 B/op          0 allocs/op
BenchmarkConvertReflect-4       1000000000           2.236 ns/op           0 B/op          0 allocs/op
BenchmarkConvertReflect-4       1000000000           2.247 ns/op           0 B/op          0 allocs/op
PASS
```


> `-benchtime=2s'`， 依次递增 b.N 直至运行时间超过 2s`-count=3`，执行 3 轮`-benchmem,b.ReportAllocs`，展示堆分配信息，0 B/op, 0 allos/op 分别代表每次分配了多少空间，每个 op 有多少次空间分配`-cpu='2,4'`，依次在 2 核、4 核下进行测试`-cpuprofile=xxxx -memprofile=xxx -trace=trace.out`，benmark 时生成 profile、trace 文件`-gcflags=all=-l`，停止编译器的内联优化`b.ResetTimer, b.StartTimer/b.StopItmer`，重置定时器`b.SetParallelism、b.RunParallel`， 并发执行，设置并发的协程数

目前对 Go 性能进行分析的主要工具包含:profile、trace，以下是对二者的介绍。
####    2.2 profile
go profile 主要是通过对快照中数据进行采样实现，采样命中越多说明函数越是热点 Go 中 profile 包括: cpu、heap、mutex、goroutine。要在 Go 中启用 profile 数据采集，主要包含以下几种方式:

1. 通过运行时函数，pprof.StartCPUProfile、pprof.WriteHeapProfile 等；
2. 通过导入 net/http/pprof 包，请求相关接口(debug/pprof/\*)；
3. go test 中使用-cpuprofile、-memprofile、-mutexprofile、-blockprofile等。

对 Profile 数据的解析，Go 提供了命令行工具 pprof、web 服务，以命令行工具为例，如下：

```
go tool pprof cpu.profile
(pprof) top 15
Showing nodes accounting for 14680ms, 99.46% of 14760ms total
Dropped 30 nodes (cum <= 73.80ms)
      flat  flat%   sum%        cum   cum%
2900ms 19.65% 19.65%     4590ms 31.10%  reflect.unpackEface (inline)
2540ms 17.21% 36.86%    13280ms 89.97%  gotest666.BenchmarkConvertReflect
1680ms 11.38% 48.24%     1680ms 11.38%  reflect.(*rtype).Kind (inline)

(pprof) list gotest666.BenchmarkConvertReflect
Total: 14.76s
ROUTINE ======================== gotest666.BenchmarkConvertReflect in /Users/zhangyuxin/go/src/gotest666/a_test.go
2.54s     13.28s (flat, cum) 89.97% of Total
         .          .      8:func BenchmarkConvertReflect(b *testing.B) {
         .          .      9:   var v interface{} = int32(64)
1.30s      1.41s     10:   for i:=0;i<b.N;i++{
         .     10.63s     11:       f := reflect.ValueOf(v).Int()
1.24s      1.24s     12:       if f != int64(64){
         .          .     13:           b.Error("errror")
         .          .     14:       }
         .          .     15:   }
         .          .     16:}
         .          .     17:
(pprof)
```

> `flat,cum` 分别代表了当前函数、当前函数调用函数的统计信息`top、list、tree`是用的最多的命令

Go 对 profile 进行解析的 web 服务包含调用图、火焰图等，可以通过 -http 参数打开。
```
go tool pprof -http=":8081" cpu.profile
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe95nic51Nzh4xWn97sPrnzIzDq8KAtr0d2vePAueeQiakS1ITznF36hGnAGVQLmiaWPB3RNuUqEzgc7QQ/640?wx_fmt=png&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1)

> 对于调用图，边框、字体的颜色越深，代表消耗资源越多。实线代表直接调用，虚线代表非直接调用（中间还有其他调用） 火焰图代表了调用层级，函数调用栈越长，火焰越高。同一层级，框越长、颜色越深占用资源越多。

profile 是通过采样实现，存在精度问题、且会对性能有影响。

####    2.3 trace
profile 工具基于快照的统计信息，存在精度问题。

为此 Go 还提供了 trace 工具，其基于事件的统计能够提供更加详细的信息。此外 trace 还把 P、G、M 等相关信息聚合在一起，从全局对问题进行一个更加直观的解释，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe95nic51Nzh4xWn97sPrnzIzDdBv6Ltv5P907ia5TXLH9SLvOqUO5GLG2ATtFTsib3ic8lrjeKXK5aYiaYQ/640?wx_fmt=png&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1)

Go 中启用 trace 数据采集，可以通过以下方式：
1. 通过runtime/trace函数，trace.Start、trace.Stop；
2. 通过导入net/http/pprof，请求debug/pprof/\*相关接口；
3. 通过 go test 中 trace 参数。
以 runtime/trace 为例，如下：
```go
import (
"os"
"runtime/trace"
)

func main() {
    f, _ := os.Create("trace.out")
    trace.Start(f)
defer trace.Stop()

    ch := make(chan string)
go func() {
        ch <- "this is a test"
    }()

    <-ch
}
```

go tool trace trace.out，会打开 web 页面，结果包含如下信息：
```
View trace         // 按照时间查看thread、goroutine分析、heap等相关信息
Goroutine analysis // goroutine相关分析
Syscall blocking profile    // syscall 相关
Scheduler latency profile   // 调度相关
........
```

需要注意，基于事件的数据采集方式，会导致性能有25%左右下降。
# 03 常用结构、用法背后的故事
####    3.1 interface、reflect
Go 中较多的 interface、reflect 会对性能有影响，但 interface、reflect 为什么会对性能有影响？
#### **interface**
Go 中 interface 包含2种，eface(empty face)、iface， eface 代表了不含方法的 interface 类型、iface 标识包含方法的 interface。

iface、eface 的定义位于 **runtime2.go、type.go**，其定义如下：

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type eface struct {
    _type *_type            // 类型信息
    data  unsafe.Pointer    // 数据
}

type itab struct {
  ........
    _type *_type
    .......
}

type _type struct {
    size       uintptr    // 大小信息
    .......
    hash       uint32     // 类型信息
    tflag      tflag        
    align      uint8      // 对齐信息
    .......
}
```

因为同时包含类型、数据，Go 中所有类型都可以转换为 interface。interface 赋值的过程，即为 iface、eface 生成的过程。如果编译阶段编译器无法确定 interface 类型（比如 :iface 入参）会通过 conv 完成打包，有可能会导致逃逸。conv 系列函数定义位于 iface.go，如下:

```go
// convT converts a value of type t, which is pointed to by v, to a pointer that can
// be used as the second word of an interface value.
func convT(t *_type, elem unsafe.Pointer) (e eface) {
    .....
    x := mallocgc(t.size, t, true)      // 空间的分配
    typedmemmove(t, x, elem)                    // memove
    e._type = t
    e.data = x
return
}

func convT64(val uint64) (x unsafe.Pointer) {
if val < uint64(len(staticuint64s)) {
        x = unsafe.Pointer(&amp;staticuint64s[val])
    } else {
        x = mallocgc(8, uint64Type, false)
        *(*uint64)(x) = val
    }
return
}

var staticuint64s = [...]uint64{....}   // 长度256的数组
```

> 很多对 interface 类型的赋值(并非所有)，都会导致空间的分配和拷贝，这也是 Interface 函数为什么可能会导致逃逸的原因 go 这么做的主要原因：逃逸的分析位于编译阶段，对于不确定的类型在堆上分配最为合适。

**Reflect.Value**

Go 中 reflect 机制涉及到2个类型，reflect.Type、reflect.Value，reflect.Type 是 Interface。

reflect.Value 定义位于 **value.go、type.go**，其定义与 eface 类似：  

```go
type Value struct {
    typ *rtype  // type._type
    ptr unsafe.Pointer
    flag
}

// rtype must be kept in sync with ../runtime/type.go:/^type._type.
type rtype struct {
    ....
}
```

> 相似的实现，即为 interface 和 reflect 可以相互转换的原因。

reflect.Value 是通过 reflect.ValueOf 生成，reflect.ValueOf 也可能会导致数据逃逸，其定义位于 value.go 中，如下：
```go
func ValueOf(i interface{}) Value {
if i == nil {
	return Value{}
}
// TODO: Maybe allow contents of a Value to live on the stack.
// For now we make the contents always escape to the heap.
    escapes(i) // 逃逸
return unpackEface(i) // unpack eface
}

// dummy为全局变量，作用域不确定可能会逃逸
func escapes(x any) {
if dummy.b {
        dummy.x = x
    }
}
```

> 再次强调：逃逸的分析是在编译阶段进行的。

一个简单的例子：

```go
func main() {
var x = "xxxx"
_ = reflect.ValueOf(x)
}
```

结果如下：
```go
➜  gotest666 go build -gcflags="-m -l" main.go
# command-line-arguments
./main.go:26:21: inlining call to reflect.ValueOf
./main.go:26:21: inlining call to reflect.escapes
./main.go:26:21: inlining call to reflect.unpackEface
./main.go:26:21: inlining call to reflect.(*rtype).Kind
./main.go:26:21: inlining call to reflect.ifaceIndir
./main.go:26:22: x escapes to heap
```

需要注意，逃逸的检测是通过-gcflags=-m，一般还需要关闭内联比如-gcflags="-m -l"。

#### **类型的选择：强类型 vs interface**

为降低可能的空间分配、拷贝，建议只在必要情况下使用 interface、reflect。
针对函数定义中强类型、interface 的性能对比，测试如下：

```go
type testStruct struct {
    Data [8192]byte
}

func StrongType(t testStruct) {
    t.Data[0] = 1
}

func InterfaceType(ti interface{}) {
    ts := ti.(testStruct)
    ts.Data[0] = 1
}

func BenchmarkTypeStrong(b *testing.B) {
    t := testStruct{}
    t.Data[0] = 2
for i := 0; i < b.N; i++ {
        StrongType(t)
    }
}

func BenchmarkTypeInterface(b *testing.B) {
    t := testStruct{}
    t.Data[0] = 2
for i := 0; i < b.N; i++ {
        InterfaceType(t)
    }
}
```

会导致逃逸时(sizeof(testStruct.Dat)\==8192)：

```go
➜  test go test -bench='Type' -run=none -benchmem
goos: darwin
goarch: amd64
pkg: gotest666/test
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkTypeStrong-12          1000000000           0.2546 ns/op          0 B/op          0 allocs/op
BenchmarkTypeInterface-12         799846          1399 ns/op        8192 B/op          1 allocs/op
PASS
```

没有逃逸时(sizeof(testStruct.Dat)\==1)：

```
➜  test go test -bench='Type' -run=none -benchmem
goos: darwin
goarch: amd64
pkg: gotest666/test
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkTypeStrong-12          1000000000           0.2549 ns/op          0 B/op          0 allocs/op
BenchmarkTypeInterface-12       1000000000           0.2534 ns/op          0 B/op          0 allocs/op
PASS
```

> 在一些会导致逃逸的情况下，不建议使用 Interface。

目前一些可能会导致逃逸的函数：

|函数|应用场景|
|---|---|
|fmt系列，包括:fmt.Sprinf、fmt.Sprint等|数据转换、格式化打印|
|binary.Read/binary.Write|二级制数据读写|
|Json.Marshal/json.UnMarshal|json相关|
  
# 06 其他
需要注意：语言层面只能解决单点的性能问题，良好的架构设计才能从全局解决问题。本文所有 benchmark、源码都是基于1.18。

# Reference
https://mp.weixin.qq.com/s/ZDWJyRG_56GQVItjNGBujQ