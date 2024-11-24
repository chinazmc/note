#golang 


struct 是我们写 Go 必然会用到的关键字, 不过当 struct 遇上一些比较特殊类型的时候, 你注意过你的程序是否正常吗 ?

### 一段代码

```go
type URL struct {
	Ip       string
	Port     string
	mux   	 sync.RWMutex
	params    url.Values
}

func (c *URL) Clone() URL {
	newUrl := URL{}
	newUrl.Ip = c.Ip
	newUrl.params = url.Values{}
	return newUrl
}

```

这段代码你能看出来问题所在吗 ?

```
A: 程序正常
B: 编译失败
C: panic
D: 有可能发生 data race
E: 有可能发生死锁

```

如果你看出来问题在哪里的话, 那我再悄悄告诉你, 这段代码是 github 某 3k star Go 框架的底层核心代码, 那你是不是就觉得这个话题开始有意思了 ?

## 先说结论

上面那段代码的问题是 `sync.RWMutex` 引起的. 如果你看过有关 sync 相关类型的介绍或者相关源码时, 在 `sync` 包里面的所有类型都有句这样的注释: `must not be copied after first use`, 可能很多人却并不知道这句话有什么作用, 顶多看到相关介绍时还记得 `sync` 相关类型的变量不能复制, 可能真正使用 Mutex, WaitGroup, Cond时, 早把这个注释忘的一干二净.

究其原因, 我觉得有下面两点原因:

1.  不明白什么叫 sync 类型变量复制
2.  sync 类型的变量复制了会出现怎样的结果

下面的例子都以 Mutex 来举例

1.  最容易看出来的情形

```go
func main() {
	var amux sync.Mutex
	b := amux
	b.Lock()
	b.Unlock()
}

```

其实这种情况一般情况下, 没人这么用. 问题不大, 略过

2.  嵌套在 struct 里面, struct 变量间的互相赋值

```go
type URL struct {
	Ip       string
	Port     string
	mux   	 sync.RWMutex
	params    url.Values
}

func main() {
	var url1 URL
	url2 := url1
}

```

当 struct 嵌套 **不可复制** 类型时, 就需要开始小心了. 当 struct 嵌套层次过深或者 struct 变量随着值传递对外扩散时, 这个时候就会变得不可控了, 就需要特别小心了.

3.  struct 类型变量的值传递作为返回值

```go
type URL struct {
	Ip       string
	mux   	 sync.RWMutex
}

func (c *URL) Clone() URL {
	newUrl := URL{}
	newUrl.Ip = c.Ip
	return newUrl
}

```

4.  struct 类型变量的值传递作为 receiver

```go
type URL struct {
	Ip       string
	mux   	 sync.RWMutex
}

func (c URL) String() string {
	c.paramsLock.Lock()
	defer c.paramsLock.Unlock()
	buf.WriteString(c.params.Encode())
	return buf.String()
}

```

### 复制后出现的结果

例子1:

```go
import (
	"fmt"
	"sync"
)

var wg sync.WaitGroup
var age int

type Person struct {
	mux sync.Mutex
}

func (p Person) AddAge() {
	defer wg.Done()
	p.mux.Lock()
	age++
	defer p.mux.Unlock()

}

func main() {
	p1 := Person{
		mux: sync.Mutex{},
	}
	wg.Add(100)
	for i := 0; i < 100; i++ {
		go p1.AddAge()
	}
	wg.Wait()
	fmt.Println(age)
}


```

结果: 结果有可能是 100, 也有可能是99....

例子2:

```go
package main

import (
	"fmt"
	"sync"
)

type Person struct {
	mux sync.Mutex
}

func Reduce(p Person) {
	fmt.Println("step...", )
	p.mux.Lock()
	fmt.Println(p)
	defer p.mux.Unlock()
	fmt.Println("over...")
}

func main() {
	var p Person
	p.mux.Lock()
	go Reduce(p)
	p.mux.Unlock()
	fmt.Println(111)
	for {
	}
}

```

结果: Reduce 协程会死锁.

看到这里我们就能发现, 当 struct 嵌套了 Mutex, 如果以值传递的方式使用时, 有可能造成程序死锁, 有可能需要互斥的变量并不能达到互斥.

所以不管是单独使用 **不能复制** 类型的变量, 还是嵌套在 struct 里面都不能值传递的方式使用.

### 不能复制原因

以 Mutex 为例,

```go
type Mutex struct {
	state int32
	sema  uint32
}

```

我们使用 Mutex 是为了不同 goroutine 之间共享某个变量, 所以需要让这个变量做到能够互斥, 不然该变量就会被互相被覆盖. Mutex 底层是由 `state` `sema` 控制的, 当 Mutex 变量被复制时, Mutex 的 state, sema 当时的状态也被复制走了, 但是由于不同 goroutine 之间的 Mutex 已经不是同一个变量了, 这样就会造成要么某个 goroutine 死锁或者不同 goroutine 共享的变量达不到互斥

## struct 如何与 不可复制 的类型一块使用 ?

由上面可以看到不只是 sync 相关类型变量自身不能被复制，而且 sturct 嵌套 **不可复制** 类型变量时, 同样也不能被复制. 但是如果我将嵌套的不可复制变量改成指针类型变量呢, 是不是就解决了不能复制的问题 ?

```go
type URL struct {
	Ip       string
	mux   	 *sync.RWMutex
}

```

这样确实解决了上述的不能复制问题. 但也引出了另外一个问题. 众所周知 Go 没有构造函数, 这就导致我们使用 URL 的时候都需要先去初始化 RWMutex, 不然就会造成同样很严重的空指针问题, 这个问题同样很棘手，也许哪个位置就忘了初始化这个 RWMutex.

根据 google groups 的讨论 [How to copy a struct which contains a mutex?](https://link.juejin.cn/?target=https%3A%2F%2Fgroups.google.com%2Fg%2Fgolang-nuts%2Fc%2FimxjBLNJ9OY "https://groups.google.com/g/golang-nuts/c/imxjBLNJ9OY"), 以及我查看了[Kubernets 的相关源码(这里只是一个例子, 里面还有很多)](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Fblob%2Fd20e3246bade17564860655e7fa57ead922d7307%2Fpkg%2Fcontroller%2Fendpointslicemirroring%2Fmetrics%2Fcache.go%23L28 "https://github.com/kubernetes/kubernetes/blob/d20e3246bade17564860655e7fa57ead922d7307/pkg/controller/endpointslicemirroring/metrics/cache.go#L28"), 发现大家的观点基本上都是一致的, 都不会去选用 struct 去嵌套指针类型的变量, 由此不建议 struct 去嵌套 **不可复制的** 的指针类型变量. 最重要的原因: 没有一个工具能去准确的检测空指针.

所以一般情况下, 当 struct 嵌套了 **不可复制** 类型的变量时, 都需要传递的是 struct 类型变量的指针.

## 如何防止复制了不该复制的变量呢?

由于 Go 并不提供`重载`的功能, 所以并不能做到去重载 struct 的相关的被复制的方法. 但是 Go 的槽点就来了, Go 本身还不提供不能被复制的相关的编译强约束. 这样就有可能导致出现**不能被复制的类型**被复制过后蒙混过关. 那我们需要怎么做呢 ?

Go 提供了另外一个工具 `go vet` 来做补充, 用这个工具是能检测出来不可复制的类型是否被复制过.

```go
func main() {
	var amux sync.Mutex
	b := amux
	b.Lock()
	b.Unlock()
}

```

```
$ go vet main.go
# command-line-arguments
./main.go:7:7: assignment copies lock value to b: sync.Mutex

```

我们怎么把 go vet 与 日常开发结合起来呢?

1.  目前的 Goland, Vscode 都会集成 go vet 的相关功能, 如果你强迫症比较严重的话, 你就能发现有相关提示.
2.  把 go vet 与 CI 流程结合起来, 其实更推荐使用 `golangci-lint` 这个 lint 工具来做 CI

Go 还提供一段 noCopy 的代码, 当你的 struct 有不能被复制的需求的时候, 可以加入这段代码

```go
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

```

这段代码依然是给 go vet 来使用的.

说到这里, 禁止复制不能被复制的变量, 这个明明能在 **编译期** 就杜绝的事情, 为啥非要搞出来工具来做这个事情呢? 有点想不通.

## 不可复制的类型有哪些?

Go 提供的不可复制的类型基本上就是 sync 包内的所有类型: `atomic.Value`, `sync.Mutex`, `sync.Cond`, `sync.RWMutex`, `sync.Map`, `sync.Pool`, `sync.WaitGroup`.

这些内置的不可被复制的类型当被复制时配合 go vet是能够发现的. 但是下面这种场景你是否遇见过?

```go
package main

import "fmt"

type Books struct {
	someImportantData []int
}

func DoSomething(otherBook Books) Books {
	newBook := otherBook
	// do something
	for k := range newBook.someImportantData {
		newBook.someImportantData[k]++ // just like this
	}
	return otherBook
}

func main() {
	oldBook := Books{
		someImportantData: make([]int, 0, 100),
	}

	oldBook.someImportantData = append(oldBook.someImportantData, 1, 2, 3)
	fmt.Println("before DoSomething, old book:", oldBook.someImportantData)
	DoSomething(oldBook)
	fmt.Println("after DoSomething, old book:", oldBook.someImportantData)
	// 使用oldBook.someImportantData 继续做某些事情
}


```

结果:

```
before DoSomething, old book: [1 2 3]
after DoSomething, old book: [2 3 4]

```

这个场景其实我们可能不经意间就会遇到. oldBook 是我们要操作的数据, 但是通过 DoSomething` 后, oldBook.someImportantData 的值可能就被改掉了, 这可能并不是我们所期待的. 由于 DoSomething 是通过复制传递的, 可能我们并不能很敏感关注到这个点, 导致程序继续往下走逻辑可能就错了. 我们是不是可以设置 Books 为不可复制呢 ? 这样可以让 go vet 帮助我们发现这些问题

# Reference
https://juejin.cn/post/6912265131619516430