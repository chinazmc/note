#golang 
# 一、什么是Context

> Context是goroutine的上下文，包含了goroutine的运行状态，环境，现场等信息，主要用来在goroutine之间传递上下文信息，包括：取消信号，超时时间，截止时间，以及K-V键值对等。下面详细来了解下Context的运行机制。

# 二、Context

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

> `_Deadline()_`：是获取设置的截止时间，第一个参数返回截止时间，意思到了这个时间点context会自动发起取消请求，第二个参数返回值ok\==false，表示没有设置截止时间，如果需要取消的话，需要调用取消函数来进行取消。  
> `_Done()_`：返回一个只读的chan，类型为struct{}，如果在select语句中，该方法返回的chan是可以读取的，则意味着parent context已经发起了取消请求；该contex没有被取消，则当前方法返回nil，不可以读取。因此我们可以通过判断是否可读,来操作当前goroutine。  
> `_Err()_`：返回取消的错误原因，因为什么context被取消了。  
> `_Value()_`：获取该context上绑定的值，如果没有对应的key，则返回nil。

# 三、emptyCtx

> 实现了context的接口，但其实是一个空的context。emptyCtx没有超时时间，不能取消，也不能存储任何额外信息，所以emptyCtx用来作为context树的根节点。但在使用中一般不直接使用emptyCtx，而是使用emptyCtx实现的两个实例变量Background和TODO，这两个本质是一样的，源码注释部分的解释说：Background通常被用于主函数、初始化以及测试中，作为一个顶层的context，也就是说一般我们创建的context都是基于Background；而TODO是在不确定使用什么context的时候才会使用。

```go
type emptyCtx int
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}
func (*emptyCtx) Done() <-chan struct{} {
	return nil
}
func (*emptyCtx) Err() error {
	return nil
}
func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
func Background() Context {
	return background
}
func TODO() Context {
	return todo
}
```

# 四、cancelCtx

> 定义一个可以被取消的上下文，取消后，同时还会取消继承它的子项。

```go
type cancelCtx struct {
	Context
	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

> `_Context_`：为匿名接口类型，继承了Context类型的方法  
> `_mu_`：互斥锁，用来保护里面的字段  
> `_done_`：表示一个channel，用来表示传递关闭信号  
> `_children_`：表示一个map，存储了当前context节点下的子节点  
> `_err_`：用于存储错误信息表示任务结束的原因

### Done

> c.done只有调用了Done()方法才创建，返回一个可读的channel，而且没有向这个channel中写入数据，直接调用读这个 channel，协程会被 block 住。一般通过搭配select 来使用。一旦关闭，就会立即读出零值。

```go
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock() //加锁
	if c.done == nil {
		c.done = make(chan struct{}) //创建chan
	}
	d := c.done
	c.mu.Unlock()//解锁
	return d
}
```

### WithCancel

> 返回一个cancelCtx和一个CancelFunc，调用CancelFunc即可触发cancel操作,如果父级为cancelCtx时，将子级context填充在父级的children中，以便在触发cancel操作时，同时将子级也一并取消和删除掉。

```go
// 返回从传入的父context派生的新context，cancelFunc
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent}
}
func propagateCancel(parent Context, child canceler) {
    //如果父级为emptyCtx,则直接返回。
    done := parent.Done() 
    if done == nil { 
        return 
    }
    //反之父级为cancelCtx时。
    select {
    case <-done: //证明父级的done通道已经关闭，即父级发生了取消操作。
        child.cancel(false, parent.Err()) //子级调用取消操作。
        return
    default:
    }
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil { //父级发生取消操作。
            child.cancel(false, p.err) //子级调用取消操作。
        } else {
            // 将父级children字段填充传过来child
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else { //如果不是cancelCtx类型
        atomic.AddInt32(&goroutines, +1)
        //开启协议来监控父级与子级是否有取消操作
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}
// 父级为cancelCtx时，才调用该方法。
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done() //返回只读的通道。
    if done == closedchan || done == nil {
        return nil, false
    }
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx) 
    if !ok {
        return nil, false
    }
    p.mu.Lock()
    ok = p.done == done //为父级的done赋值
    p.mu.Unlock()
    if !ok {
        return nil, false
    }
    return p, true
}
```

### cancel

> 继承自canceler接口，关闭自身的done通道；字节点递归调用取消操作，关闭自身的done通道，从父节点中删除自己。达到的效果是通过关闭channel，将取消信号传递给了它的所有子节点。goroutine 接收到取消信号的方式就是 select 语句中的读 c.done 被选中。

```go
// 调用cancel实现取消操作。
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil { //说明已经取消
        c.mu.Unlock()
        return 
    }
    c.err = err
    if c.done == nil {
        c.done = closedchan
    } else {
        //关闭done通道，让select中调用ctx.Done()感知
        close(c.done) 
    }
    for child := range c.children {
        child.cancel(false, err) //子级递归调用取消操作
    }
    c.children = nil //将所有子级赋值为nil
    c.mu.Unlock()
    // 是否从父级移除，默认true
    if removeFromParent {
        removeChild(c.Context, c) //从父级中删除
    }
}
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        return nil, false
    }
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        return nil, false
    }
    p.mu.Lock()
    ok = p.done == done
    p.mu.Unlock()
    if !ok {
        return nil, false
    }
    return p, true
}
```

### 五、valueCtx

> 定义一个带有键值对的contetx，并且将其传导到该context的子项。

```go
type valueCtx struct {
	Context
	key, val interface{}
}
```

> `_Context_`：为匿名接口类型，继承了Context类型的方法。  
> `_key,val_`：在valueCtx中定义一组键值对。

### 六、timerCtx

> 是基于cancelCtx的，只是多了一个计时器（timer）和截止时间（deadline），原理是通过停止计时器来实现取消，然后委托给cancelCtx.cancel方法。

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer 
	deadline time.Time
}
```

> `_cancelCtx_`： 匿名接口，继承该cancelCtx的所有方法。  
> `_timer_`： 计时器。  
> `_deadline_`： 截止时间。

### WithDeadline

> 返回一个timerCtx和cancel方法，超时逻辑主要用到了timer.AfterFunc。在构建后必须手动defer cancel() 下，不然即使提前执行完了，也要等d的时间才会被释放，切记。

### WithTimeout

> 直接调用了WithDeadline，传入的deadline是当前时间加上timeout的时间，也就是从现在开始在经过timeout时间就算是超时。

```go
// 返回一个timerCtx和cancel方法。
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    // 父节点的deadline时间早于指定的d时间。直接构建一个取消的context
   // 通过调用cancel来取消context。
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        return WithCancel(parent)
    }
    // 构建timerCtx
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    // 挂靠到父节点上，见cancel逻辑
    propagateCancel(parent, c)
    // 计算当前距离 d 的时间
    dur := time.Until(d)
    if dur <= 0 {
        // 直接取消，表示已经d已经过去了
        c.cancel(true, DeadlineExceeded)
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        // c.timer会在d时间间隔后，自动调用cancel函数
        // 并且传入的错误就是 DeadlineExceeded
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
// 在WithDeadline中调用，不需要直接执行
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err) //调用cancelCtx的取消操作
    if removeFromParent {
        removeChild(c.cancelCtx.Context, c) //从父节点删除自身节点
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop() //计时器停止
        c.timer = nil
    }
    c.mu.Unlock()
}
// 直接调用了WithDeadline，传入的deadline是当前时间加上timeout的时间，
// 也就是从现在开始在经过timeout时间就算是超时
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```


# Golang 笔记（二）：Context 源码剖析

![go-context-tree-construction.png](https://i.loli.net/2020/07/25/Yjbq6ZSkEHQ3X1u.png)

## 概述

Context 是 Go 中一个比较独特而常用的概念，用好了往往能事半功倍。但如果不知其然而滥用，则往往变成 “为赋新词强说愁”，轻则影响代码结构，重则埋下许多 bug。

Golang 使用树形派生的方式构造 Context，通过在不同过程 _[1]_ 中传递 deadline 和 cancel 信号，来管理处理某个任务所涉及到的一组 goroutine 的生命周期，防止 goroutine 泄露。并且可以通过附加在 Context 上的 Value 来传递 / 共享一些跨越整个请求间的数据。

Context 最常用来追踪 RPC/HTTP 等耗时的、跨进程的 IO 请求的生命周期，从而让外层调用者可以主动地或者自动地取消该请求，进而告诉子过程回收用到的所有 goroutine 和相关资源。

Context 本质上是一种在 API 间树形嵌套调用时传递信号的机制。本文将从接口、派生、源码分析、使用等几个方面来逐一解析 Context。

## Context 接口

Context 接口如下：
```go
// Context 用以在多 API 间传递 deadline、cancelation 信号和请求的键值对。  
// Context 中的方法能够安全的被多个 goroutine 并发调用。  
type Context interface {  
    // Done 返回一个只读 channel，该 channel 在 Context 被取消或者超时时关闭  
    Done() <-chan struct{}  
  
    // Err 返回 Context 结束时的出错信息  
    Err() error  
  
    // 如果 Context 被设置了超时，Deadline 将会返回超时时限。  
    Deadline() (deadline time.Time, ok bool)  
  
    // Value 返回关联到相关 Key 上的值，或者 nil.  
    Value(key interface{}) interface{}  
}  
```

上面是简略注释，接口详细信息可以访问 Context 的 [godoc](https://golang.org/pkg/context/)。

-   `Done()` 方法返回一个只读的 channel，当 Context 被主动取消或者超时自动取消时，该 Context 所有派生 Context 的 done channel 都被 close 。所有子过程通过该字段收到 close 信号后，应该立即中断执行、释放资源然后返回。
-   `Err()` 在上述 channel 被 close 前会返回 nil，在被 close 后会返回该 Context 被关闭的信息，error 类型，只有两种，\_被取消_或者_超时_：
    
```go
var Canceled = errors.New("context canceled")  
var DeadlineExceeded error = deadlineExceededError{} 
``` 

-   `Deadline()` 如果本 Context 被设置了时限，则该函数返回 `ok=true` 和对应的到期时间点。否则，返回 `ok=false` 和 nil。
    
-   `Value()` 返回绑定在该 Context 链（我称为回溯链，下文会展开说明）上的给定的 Key 的值，如果没有，则返回 nil。注意，不要用于在函数中传参，其本意在于共享一些横跨整个 Context 生命周期范围的值。Key 可以是任何可比较的类型。为了防止 Key 冲突，最好将 Key 的类型定义为非导出类型，然后为其定义访问器。看一个通过 Context 共享用户信息的例子： 

```go
package user  
  
import "context"  
  
// User 是要存于 Context 中的 Value 类型.  
type User struct {...}  
  
// key 定义为了非导出类型，以避免和其他 package 中的 key 冲突  
type key int  
  
// userKey 是 Context 中用来关联 user.User 的 key，是非导出变量  
// 客户端需要用 user.NewContext 和 user.FromContext 构建包含  
// user 的 Context 和从 Context 中提取相应 user   
var userKey key  
  
// NewContext 返回一个带有用户值 u 的 Context.  
func NewContext(ctx context.Context, u *User) context.Context {  
  return context.WithValue(ctx, userKey, u)  
}  
  
// FromContext 从 Context 中提取 user，如果有的话.  
func FromContext(ctx context.Context) (*User, bool) {  
  u, ok := ctx.Value(userKey).(*User)  
  return u, ok  
}  
```

## Context 派生

Context 设计之妙在于可以从已有 Context 进行树形派生，以管理一组过程的生命周期。我们上面说了单个 Context 实例是不可变的，但可以通过 context 包提供的三种方法：`WithCancel` 、 `WithTimeout` 和 `WithValue` 来进行派生并附加一些属性（可取消、时限、键值），以构造一组树形组织的 Context。

![go-context-tree.png](https://i.loli.net/2020/07/25/d5Slcasj1FAzItu.png)

当根 Context 结束时，所有由其派生出的 Context 也会被一并取消。也就是说，父 Context 的生命周期涵盖所有子 Context 的生命周期。

`context.Background()` 通常用作根节点，它不会超时，不能被取消。 

```go
// Background 返回一个空 Context。它不能被取消，没有时限，没有附加键值。Background 通常用在  
// main函数、init 函数、test 入口，作为某个耗时过程的根 Context。  
func Background() Context  
```

`WithCancel` 和 `WithTimeout` 可以从父 Context 进行派生，返回受限于父 Context 生命周期的新 Context。

通过 `WithCancel` 从 `context.Background()` 派生出的 Context 要注意在对应过程完结后及时 cancel，否则会造成 Context 泄露。

使用 `WithTimeout` 可以控制某个过程的处理时限。具体过程为，到点后， Context 发送信号到 Done Channel，子过程检测到 Context Done Channel _[2]_ 中的信号，会立即退出。

```go
// WithCancel 返回一份父 Context 的拷贝，和一个 cancel 函数。当父 Context 被关闭或者   
// 此 cancel 函数被调用时，该 Context 的 Done Channel 会立即被关闭.  
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)  
  
// 调用 CancelFunc 取消对应 Context.  
type CancelFunc func()  
  
// WithTimeout 返回一份父 Context 的拷贝，和一个 cancel 函数。当父 Context 被关闭、  
// cancel 函数被调用或者设定时限到达时，该 Context 的 Done Channel 会立即关闭。在 cancel 函数  
// 被调用时，如果其内部 timer 仍在运行，将会被停掉。  
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)  
```

`WithValue` 可以给 Context 附加上整个处理过程中的键值。

```go
// WithValue 返回一个父 Context 的副本，并且附加上给定的键值对.  
func WithValue(parent Context, key interface{}, val interface{}) Context  
```

## Context 源码解析

Go context 使用嵌入类，以类似继承的方式组织几个 Context 类：`emptyCtx`、`valueCtx`、 `cancelCtx`、`timerCtx`。

![go-context-implementation.png](https://i.loli.net/2020/07/25/9s6qG8RUkCQWLlc.png)

形象的来说，通过嵌入的方式，Go 对树形组织的 Context 体系中的每个 Context 节点都构造了一个指向父亲实例” 指针”。从另一个角度来说，这是一种经典代码组织模式 —— [组合模式](https://sourcemaking.com/design_patterns/composite)，每一层只增量 or 覆盖实现自己所关注的功能，然后通过路由调用来复用已有的实现。

### 空实现 emptyCtx

`emptyCtx` 实现了一个空的 `Context`，所有接口函数都是空实现。

```go
type emptyCtx int  
  
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {  
  return  
}  
func (*emptyCtx) Done() <-chan struct{} {  
  return nil // 返回 nil，从语法上说是空实现，从语义上说是该 Context 永远不会被关闭。  
}  
//... 其他的省略，类似都是满足语法要求的空函数体  
  
func (e *emptyCtx) String() string {  
  switch e {  
  case background:  
    return "context.Background"  
  case todo:  
    return "context.TODO"  
  }  
  return "unknown empty Context"  
}  
```

`context.Background()` 和 `context.TODO()` 返回的都是 `emptyCtx` 的实例。但其语义略有不同。前者做为 Context 树的根节点，后者通常在不知道用啥时用。 

```go
var (  
  background = new(emptyCtx)  
  todo       = new(emptyCtx)  
)  
  
func Background() Context {  
  return background  
}  
  
func TODO() Context {  
  return todo  
} 
``` 

### 附加单键值 valueCtx

`valueCtx` 嵌入了一个 `Context` 接口以进行 Context 派生，并且附加了一个 KV 对。从 `context.WithValue` 函数可以看出，每附加一个键值对，都得套上一层新的 `valueCtx`。在使用 `Value(key interface)` 接口访问某 Key 时，会沿着 Context 树回溯链不断向上遍历所有 Context 直到 `emptyCtx`：

1.  如果遇到 `valueCtx` 实例，则比较其 key 和给定 key 是否相等
2.  如果遇到其他 Context 实例，就直接向上转发。但这里有个特例，为了获取给定 Context 所有祖先节点中最近的 `cancelCtx`，go 用了一个特殊的 key：`cancelCtxKey`，遇到该 key 时，cancelCtx 会返回自身。这个在 `cancelCtx` 实现中会提到。

对于其他的接口调用（`Done`, `Err`, `Deadline`），会路由到嵌入的 `Context` 上去。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  

type valueCtx struct {  
  Context // 嵌入，指向父 Context  
  key, val interface{}  
}  
  
func (c *valueCtx) Value(key interface{}) interface{} {  
  if c.key == key {  
    return c.val  
  }  
  return c.Context.Value(key)  
}  
  
func WithValue(parent Context, key, val interface{}) Context {  
  if key == nil {  
    panic("nil key")  
  }  
  if !reflectlite.TypeOf(key).Comparable() {  
    panic("key is not comparable")  
  }  
  return &valueCtx{parent, key, val} // 附加上 kv，并引用父 Context  
}  

### [](https://www.qtmuniao.com/2020/07/12/go-context/#%E5%8F%AF%E5%8F%96%E6%B6%88%E7%9A%84-cancelCtx "可取消的 cancelCtx")可取消的 cancelCtx

context 包中核心实现在 `cancelCtx` 中，包括构造树形结构、进行级联取消。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  

type cancelCtx struct {  
  Context  
  
  mu       sync.Mutex            // 保证下面三个字段的互斥访问  
  done     chan struct{}         // 惰式初始化，被第一个 cancel() 调用所关闭  
  children map[canceler]struct{} // 被第一个 cancel() 调用置 nil  
  err      error                 // 被第一个 cancel() 调用置非 nil  
}  
  
func (c *cancelCtx) Value(key interface{}) interface{} {  
  if key == &cancelCtxKey {   
    return c  
  }  
  return c.Context.Value(key)  
}  
  
func (c *cancelCtx) Done() <-chan struct{} {  
  c.mu.Lock()  
  if c.done == nil {  
    c.done = make(chan struct{})  
  }  
  d := c.done  
  c.mu.Unlock()  
  return d  
}  

`Value()` 函数的实现有点意思，遇到特殊 key：`cancelCtxKey` 时，会返回自身。这个其实是复用了 Value 函数的回溯逻辑，从而在 Context 树回溯链中遍历时，可以找到给定 Context 的第一个祖先 `cancelCtx` 实例。

`children` 保存的是子树中所有路径向下走的第一个可以 cancel 的 Context (实现了 `canceler` 接口，比如 `cancelCtx` 或 `timerCtx` 节点)，可以参考后面的图来形象理解。

下面将逐一详细说明。

#### [](https://www.qtmuniao.com/2020/07/12/go-context/#%E5%9B%9E%E6%BA%AF%E9%93%BE "回溯链")回溯链

回溯链是各个 context 包在实现时利用 go 语言嵌入（_embedding_）的特性来构造的，主要用于：

1.  `Value()` 函数被调用时沿着回溯链向上查找匹配的键值对。
2.  复用 `Value()` 的逻辑查找最近 `cancelCtx` 祖先，以构造 Context 树。

在 `valueCtx`、`cancelCtx`、`timerCtx` 中只有 `cancelCtx` **直接**（`valueCtx` 和 `timerCtx` 都是通过嵌入实现，调用该方法会直接转发到 `cancelCtx` 或者 `emptyCtx` ）实现了非空 `Done()` 方法，因此 `done := parent.Done()` 会返回第一个祖先 `cancelCtx` 中的 done channel。但如果 Context 树中有第三方实现的 Context 接口的实例时，`parent.Done()` 就有可能返回其他 channel。

因此，如果 `p.done != done` ，说明在回溯链中遇到的第一个实现非空 `Done()` Context 是第三方 Context ，而非 `cancelCtx`。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  

// parentCancelCtx 返回 parent 的第一个祖先 cancelCtx 节点  
func parentCancelCtx(parent Context) (*cancelCtx, bool) {  
  done := parent.Done() // 调用回溯链中第一个实现了 Done() 的实例(第三方Context类/cancelCtx)  
  if done == closedchan || done == nil {  
    return nil, false  
  }  
  p, ok := parent.Value(&cancelCtxKey).(*cancelCtx) // 回溯链中第一个 cancelCtx 实例  
  if !ok {  
    return nil, false  
  }  
  p.mu.Lock()  
  ok = p.done == done  
  p.mu.Unlock()  
  if !ok { // 说明回溯链中第一个实现 Done() 的实例不是 cancelCtx 的实例  
    return nil, false  
  }  
  return p, true  
}  

#### [](https://www.qtmuniao.com/2020/07/12/go-context/#%E6%A0%91%E6%9E%84%E5%BB%BA "树构建")树构建

Context 树的构建是在调用 `context.WithCancel()` 调用时通过 `propagateCancel` 进行的。

1  
2  
3  
4  
5  

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {  
  c := newCancelCtx(parent)  
  propagateCancel(parent, &c)  
  return &c, func() { c.cancel(true, Canceled) }  
}  

Context 树，本质上可以细化为 `canceler` （`*cancelCtx` 和 `*timerCtx`）树，因为在级联取消时只需找到子树中所有的 `canceler` ，因此在实现时只需在树中保存所有 `canceler` 的关系即可（跳过 `valueCtx`），简单且高效。

1  
2  
3  
4  
5  
6  

// A canceler is a context type that can be canceled directly. The  
// implementations are *cancelCtx and *timerCtx.  
type canceler interface {  
  cancel(removeFromParent bool, err error)  
  Done() <-chan struct{}  
}  

具体实现为，沿着回溯链找到第一个实现了 `Done()` 方法的实例，

1.  如果为 `canceler` 的实例，则其必有 children 字段，并且实现了 cancel 方法（canceler），将该 context 放进 children 数组即可。此后，父 cancelCtx 在 cancel 时会递归遍历所有 children，逐一 cancel。
2.  如果为非 `canceler` 的第三方 Context 实例，则我们不知其内部实现，因此只能为每个新加的子 Context 启动一个守护 goroutine，当 父 Context 取消时，取消该 Context。

需要注意的是，由于 Context 可能会被多个 goroutine 并行访问，因此在更改类字段时，需要再一次检查父节点是否已经被取消，若父 Context 被取消，则立即取消子 Context 并退出。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  
26  
27  
28  
29  
30  
31  
32  
33  
34  
35  
36  
37  

func propagateCancel(parent Context, child canceler) {  
  done := parent.Done()  
  if done == nil {  
    return // 父节点不可取消  
  }  
  
  select {  
  case <-done:  
    // 父节点已经取消  
    child.cancel(false, parent.Err())  
    return  
  default:  
  }  
  
  if p, ok := parentCancelCtx(parent); ok { // 找到一个 cancelCtx 实例  
    p.mu.Lock()  
    if p.err != nil {  
      // 父节点已经被取消  
      child.cancel(false, p.err)  
    } else {  
      if p.children == nil {  
        p.children = make(map[canceler]struct{}) // 惰式创建  
      }  
      p.children[child] = struct{}{}  
    }  
    p.mu.Unlock()  
  } else {                                // 找到一个非 cancelCtx 实例  
    atomic.AddInt32(&goroutines, +1)  
    go func() {  
      select {  
      case <-parent.Done():  
        child.cancel(false, parent.Err())  
      case <-child.Done():   
      }  
    }()  
  }  
}  

下面用一张图来解释下回溯链和树组织， _C0_ 是 `emptyCtx`，通常由 `context.Background()` 得来，作为 Context 树的根节点。_C1_~_C4_ 依次通过嵌入的方式从各自父节点派生而来。图中的虚线是由嵌入（_embedded_）而构成的回溯链，实线是由 `cancelCtx` children 数组而保存的父子关系。

`parentCancelCtx(C2)` 和 `parentCancelCtx(C4)` 都为 _C1_，则 _C1_ 的 children 数组中保存的为 _C2_ 和 _C4_。构建了这两层关系后，就可以沿着回溯链向上查询 Value 值，包括找到第一个祖先 `cancelCtx`；也可以沿着 children 关系往下进行级联取消。

![go-context-tree-construction.png](https://i.loli.net/2020/07/25/Yjbq6ZSkEHQ3X1u.png)

当然，图中所有 Context 都是针对 go 包中的系统 Context，没有画出有第三方 Context 的情况。而实际代码由于增加了对第三方 Context 的处理逻辑，稍微难懂一些。区分系统 Context 实现和用户自定义 Context 的关键点在于是否实现了 `canceler` 接口。

第三方 Context 实现了此接口就可以进行树形组织，并且在上游 `cancelCtx` 取消时，递归调用 children 的 cancel 进行级联取消。否则只能通过为每个第三方 Context 启动一个 goroutine 来监听上游取消事件，以对第三方 Context 进行取消了。

#### [](https://www.qtmuniao.com/2020/07/12/go-context/#%E7%BA%A7%E8%81%94%E5%8F%96%E6%B6%88 "级联取消")级联取消

下面是级联取消中的关键函数 `cancelCtx.cancel` 的实现。在本 `cancelCtx` 取消时，需要级联取消以该 `cancelCtx` 为根节点的 Context 树中的所有 Context，并将根 `cancelCtx` 从其从父节点中摘除，以让 GC 回收该 `cancelCtx` 子树所有节点的资源。

`cancelCtx.cancel` 是非导出函数，不能在 context 包外调用，因此持有 Context 的内层过程不能自己取消自己，须由返回的 `CancelFunc` （简单的包裹了 `cancelCtx.cancel` ）来取消，其句柄一般为外层过程所持有。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  
26  
27  
28  
29  
30  
31  
32  

func (c *cancelCtx) cancel(removeFromParent bool, err error) {  
  if err == nil { // 需要给定取消的理由，Canceled or DeadlineExceeded  
    panic("context: internal error: missing cancel error")  
  }  
    
  c.mu.Lock()  
  if c.err != nil {  
    c.mu.Unlock()  
    return // 已经被其他 goroutine 取消  
  }  
    
  // 记下错误，并关闭 done  
  c.err = err  
  if c.done == nil {  
    c.done = closedchan  
  } else {  
    close(c.done)  
  }  
    
  // 级联取消  
  for child := range c.children {  
    // NOTE: 持有父 Context 的同时获取了子 Context 的锁  
    child.cancel(false, err)  
  }  
  c.children = nil  
  c.mu.Unlock()  
  
  // 子树根需要摘除，子树中其他节点则不再需要  
  if removeFromParent {  
    removeChild(c.Context, c)  
  }  
}  

### [](https://www.qtmuniao.com/2020/07/12/go-context/#timerCtx "timerCtx")timerCtx

`timerCtx` 在嵌入 `cancelCtx` 的基础上增加了一个计时器 timer，根据用户设置的时限，到点取消。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  
26  
27  
28  

type timerCtx struct {  
  cancelCtx  
  timer *time.Timer // Under cancelCtx.mu  
  
  deadline time.Time  
}  
  
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {  
  return c.deadline, true  
}  
  
func (c *timerCtx) cancel(removeFromParent bool, err error) {  
  // 级联取消子树中所有 Context  
  c.cancelCtx.cancel(false, err)  
    
  if removeFromParent {  
    // 单独调用以摘除此节点，因为是摘除 c，而非 c.cancelCtx  
    removeChild(c.cancelCtx.Context, c)  
  }  
    
  // 关闭计时器  
  c.mu.Lock()  
  if c.timer != nil {  
    c.timer.Stop()  
    c.timer = nil  
  }  
  c.mu.Unlock()  
}  

设置超时取消是在 `context.WithDeadline()` 中完成的。如果祖先节点时限早于本节点，只需返回一个 `cancelCtx` 即可，因为祖先节点到点后在级联取消时会将其取消。

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  
26  
27  

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {  
  if cur, ok := parent.Deadline(); ok && cur.Before(d) {  
    // 祖先节点的时限更早  
    return WithCancel(parent)  
  }  
    
  c := &timerCtx{               
    cancelCtx: newCancelCtx(parent), // 使用一个新的 cancelCtx 实现部分 cancel 功能  
    deadline:  d,  
  }  
  propagateCancel(parent, c) // 构建 Context 取消树，注意传入的是 c 而非 c.cancelCtx  
  dur := time.Until(d)       // 测试时限是否设的太近以至于已经结束了  
  if dur <= 0 {  
    c.cancel(true, DeadlineExceeded)  
    return c, func() { c.cancel(false, Canceled) }  
  }  
    
  // 设置超时取消  
  c.mu.Lock()  
  defer c.mu.Unlock()  
  if c.err == nil {  
    c.timer = time.AfterFunc(dur, func() {  
      c.cancel(true, DeadlineExceeded)  
    })  
  }  
  return c, func() { c.cancel(true, Canceled) }  
}  

## [](https://www.qtmuniao.com/2020/07/12/go-context/#Context-%E4%BD%BF%E7%94%A8 "Context 使用")Context 使用

使用了 Context 的子过程**须保证**在 Context 被关闭时及时退出并释放资源。也就是说，使用 Context 需要遵循上述原则才能保证级联取消时释放资源的效果。因此，Context 本质上是一种树形分发信号的机制，可以用 Context 树追踪过程调用树，当外层过程取消时，使用 Context 级联通知所有被调用过程。

以下是一个典型子过程的检查 Context 以确定是否需要退出的代码片段：

1  
2  
3  
4  
5  
6  
7  
8  
9  

for ; ; time.Sleep(time.Second) {  
    select {  
    case <-context.Done():  
      return  
    default:  
    }  
    
    // 一些耗时操作  
}  

可以看出，Context 接口本身并没有 Cancel 方法，这和 `Done()` 返回的 channel 是只读的是一个道理：Context 关闭信号的发送方和接收方通常不在一个函数中。比如，当父 goroutine 启动了一些子 goroutine 来干活时，只能是父 goroutine 来关闭 done channel，子 goroutine 来检测 channel 的关闭信号。即不能在子 goroutine 中 取消父 goroutine 中传递过来的 Context。

## [](https://www.qtmuniao.com/2020/07/12/go-context/#Context-%E6%B3%A8%E6%84%8F "Context 注意")Context 注意

Context 有一些使用实践需要遵循：

1.  Context 通常作为函数中第一个参数
2.  不要在 struct 中存储 Context，每个函数都要显式的传递 Context。不过实践中可以根据 struct 的生命周期来灵活组合。
3.  不要使用 nil Context，尽管语法上允许。不知道使用什么值合适时，可以使用 `context.TODO()` 。
4.  Context value 是为了在请求生命周期中共享数据，而非作为函数中传递额外参数的方法。因为这是一种隐式的语义，极易造成 bug；要想传额外参数，还是要在函数中显式声明。
5.  Context 是 immutable 的，因此是线程安全的，可以在多个 goroutine 中传递并使用同一个 Context。

## [](https://www.qtmuniao.com/2020/07/12/go-context/#%E6%B3%A8 "注")注

[1] 文中的过程，指的是计算密集型或者 IO 密集型的耗时函数，或者 goroutine。

[2] Context 的 Done Channel，指的是 `context.Done()` 返回的 channel。它是 Context 内的关键数据结构，作为沟通不同过程的的渠道。需要结束时，父过程向该 channel 发送信号，子过程读取该 channel 信号后做扫尾工作并且退出。

## [](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%B8%80%E3%80%81Context-%E4%BB%8B%E7%BB%8D "一、Context 介绍")一、Context 介绍[](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%B8%80%E3%80%81Context-%E4%BB%8B%E7%BB%8D)

标准库中的 Context 是一个接口，其具体实现有很多种；Context 在 Go 1.7 中被添加入标准库，主要用于跨多个 Goroutine 设置截止时间、同步信号、传递上下文请求值等。

由于需要跨多个 Goroutine 传递信号，所以多个 Context 往往需要关联到一起，形成类似一个树的结构。这种树状的关联关系需要有一个根(root) Context，然后其他 Context 关联到 root Context 成为它的子(child) Context；这种关联可以是多级的，所以在角色上 Context 分为三种:

-   root(根) Context
-   parent(父) Context
-   child(子) Context

## [](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%BA%8C%E3%80%81Context-%E7%B1%BB%E5%9E%8B "二、Context 类型")二、Context 类型[](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%BA%8C%E3%80%81Context-%E7%B1%BB%E5%9E%8B)

### [](https://mritd.com/2021/06/27/golang-context-source-code/#2-1%E3%80%81Context-%E7%9A%84%E5%88%9B%E5%BB%BA "2.1、Context 的创建")2.1、Context 的创建[](https://mritd.com/2021/06/27/golang-context-source-code/#2-1%E3%80%81Context-%E7%9A%84%E5%88%9B%E5%BB%BA)

标准库中定义的 Context 创建方法大致如下:

-   context.Background(): 该方法用于创建 root Context，且不可取消
-   context.TODO(): 该方法同样用于创建 root Context(不准确)，也不可取消，TODO 通常代表不知道要使用哪个 Context，所以后面可能有调整
-   context.WithCancel(parent Context): 从 parent Context 创建一个带有取消方法的 child Context，该 Context 可以手动调用 cancel
-   context.WithDeadline(parent Context, d time.Time): 从 parent Context 创建一个带有取消方法的 child Context，不同的是当到达 d 时间后该 Context 将自动取消
-   context.WithTimeout(parent Context, timeout time.Duration): 与 WithDeadline 类似，只不过指定的是一个从当前时间开始的超时时间
-   context.WithValue(parent Context, key, val interface{}): 从 parent Context 创建一个 child Context，该 Context 可以存储一个键值对，同时这是一个不可取消的 Context

### [](https://mritd.com/2021/06/27/golang-context-source-code/#2-2%E3%80%81Context-%E5%86%85%E9%83%A8%E7%B1%BB%E5%9E%8B "2.2、Context 内部类型")2.2、Context 内部类型[](https://mritd.com/2021/06/27/golang-context-source-code/#2-2%E3%80%81Context-%E5%86%85%E9%83%A8%E7%B1%BB%E5%9E%8B)

在阅读源码后会发现，Context 各种创建方法其实主要只使用到了 4 种类型的 Context 实现:

#### [](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-1%E3%80%81emptyCtx "2.2.1、emptyCtx")2.2.1、emptyCtx[](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-1%E3%80%81emptyCtx)

`emptyCtx` 实际上就是个 int，其对 Context 接口的主要实现(`Deadline`、`Done`、`Err`、`Value`)全部返回了 `nil`，也就是说其实是一个 “啥也不干” 的 Context；**它通常用于创建 root Context，标准库中 `context.Background()` 和 `context.TODO()` 返回的就是这个 `emptyCtx`。**

```
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
	return background
}

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
func TODO() Context {
	return todo
}
```

#### [](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-2%E3%80%81cancelCtx "2.2.2、cancelCtx")2.2.2、cancelCtx[](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-2%E3%80%81cancelCtx)

`cancelCtx` 内部包含一个 `Context` 接口实例，还有一个 `children map[canceler]struct{}`；这两个变量的作用就是保证 `cancelCtx` 可以在 parent Context 和 child Context 两种角色之间转换:

-   作为其他 Context 实例的 parent Context 时，将其他 Context 实例存储在 `children map[canceler]struct{}` 中建立关联关系
-   作为其他 Context 实例的 child Context 时，将其他 Context 实例存储在 “Context” 变量里建立关联

**`cancelCtx` 被定义为一个可以取消的 Context，而由于 Context 的树形结构，当作为 parent Context 取消时需要同步取消节点下所有 child Context，这时候只需要遍历 `children map[canceler]struct{}` 然后逐个取消即可。**

```
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Value(key interface{}) interface{} {
	if key == &cancelCtxKey {
		return c
	}
	return c.Context.Value(key)
}

func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}

type stringer interface {
	String() string
}

func contextName(c Context) string {
	if s, ok := c.(stringer); ok {
		return s.String()
	}
	return reflectlite.TypeOf(c).String()
}

func (c *cancelCtx) String() string {
	return contextName(c.Context) + ".WithCancel"
}

// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

#### [](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-3%E3%80%81timerCtx "2.2.3、timerCtx")2.2.3、timerCtx[](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-3%E3%80%81timerCtx)

`timerCtx` 实际上是在 `cancelCtx` 之上构建的，唯一的区别就是增加了计时器和截止时间；**有了这两个配置以后就可以在特定时间进行自动取消，`WithDeadline(parent Context, d time.Time)` 和 `WithTimeout(parent Context, timeout time.Duration)` 方法返回的都是这个 `timerCtx`。**

```
// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}

func (c *timerCtx) String() string {
	return contextName(c.cancelCtx.Context) + ".WithDeadline(" +
		c.deadline.String() + " [" +
		time.Until(c.deadline).String() + "])"
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

#### [](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-4%E3%80%81valueCtx "2.2.4、valueCtx")2.2.4、valueCtx[](https://mritd.com/2021/06/27/golang-context-source-code/#2-2-4%E3%80%81valueCtx)

`valueCtx` 内部同样包含了一个 `Context` 接口实例，目的也是可以作为 child Context，同时为了保证其 “Value” 特性，其内部包含了两个无限制变量 `key, val interface{}`；**在调用 `valueCtx.Value(key interface{})` 会进行递归向上查找，但是这个查找只负责查找 “直系” Context，也就是说可以无限递归查找 parent Context 是否包含这个 key，但是无法查找兄弟 Context 是否包含。**

```
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}

// stringify tries a bit to stringify v, without using fmt, since we don't
// want context depending on the unicode tables. This is only used by
// *valueCtx.String().
func stringify(v interface{}) string {
	switch s := v.(type) {
	case stringer:
		return s.String()
	case string:
		return s
	}
	return "<not Stringer>"
}

func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(type " +
		reflectlite.TypeOf(c.key).String() +
		", val " + stringify(c.val) + ")"
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

## [](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%B8%89%E3%80%81cancelCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90 "三、cancelCtx 源码分析")三、cancelCtx 源码分析[](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%B8%89%E3%80%81cancelCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)

### [](https://mritd.com/2021/06/27/golang-context-source-code/#3-1%E3%80%81cancelCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E8%A2%AB%E5%88%9B%E5%BB%BA%E7%9A%84 "3.1、cancelCtx 是如何被创建的")3.1、cancelCtx 是如何被创建的[](https://mritd.com/2021/06/27/golang-context-source-code/#3-1%E3%80%81cancelCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E8%A2%AB%E5%88%9B%E5%BB%BA%E7%9A%84)

cancelCtx 在调用 `context.WithCancel` 方法时创建(暂不考虑其他衍生类型)，创建方法比较简单:

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

`newCancelCtx` 方法就是将 parent Context 设置到内部变量中，**值得分析的是 `propagateCancel(parent, &c)` 方法和被其调用的 `parentCancelCtx(parent Context) (*cancelCtx, bool)` 方法，这两个方法保证了 Context 链可以从顶端到底端的及联 cancel**，关于这两个方法的分析如下:

```
// propagateCancel arranges for child to be canceled when parent is.
// propagateCancel 这个方法主要负责保证当 parent Context 被取消时，child Context 也会被及联取消
func propagateCancel(parent Context, child canceler) {
	// 针对于 context.Background()/TODO() 创建的 Context(emptyCtx)，其 done channel 将永远为 nil
	// 对于其他的标准的可取消的 Context(cancelCtx、timerCtx) 调用 Done() 方法将会延迟初始化 done channel(调用时创建)
	// 所以 done channel 为 nil 时说明 parent context 必然永远不会被取消，所以就无需及联到 child Context
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	// 如果 done channel 不是 nil，说明 parent Context 是一个可以取消的 Context
	// 这里需要立即判断一下 done channel 是否可读取，如果可以读取说明上面无锁阶段
	// parent Context 已经被取消了，那么应该立即取消 child Context
	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	// parentCancelCtx 用于获取 parent Context 的底层可取消 Context(cancelCtx)
	//
	// 如果 parent Context 本身就是 *cancelCtx 或者是标准库中基于 cancelCtx 衍生的 Context 会返回 true
	// 如果 parent Context 已经取消/或者根本无法取消 会返回 false
	// 如果 parent Context 无法转换为一个 *cancelCtx 也会返回 false
	// 如果 parent Context 是一个自定义深度包装的 cancelCtx(自己定义了 done channel) 则也会返回 false
	if p, ok := parentCancelCtx(parent); ok { // ok 为 true 说明 parent Context 为 标准库的 cancelCtx 或者至少可以完全转换为 *cancelCtx
		// 先对 parent Context 加锁，防止更改
		p.mu.Lock()
		// 因为 ok 为 true 就已经确定了 parent Context 一定为 *cancelCtx，而 cancelCtx 取消时必然设置 err
		// 所以并发加锁情况下如果 parent Context 的 err 不为空说明已经被取消了
		if p.err != nil {
			// parent has already been canceled
			// parent Context 已经被取消，则直接及联取消 child Context
			child.cancel(false, p.err)
		} else {
			// 在 ok 为 true 时确定了 parent Context 一定为 *cancelCtx，此时 err 为 nil
			// 这说明 parent Context 还没被取消，这时候要在 parent Context 的 children map 中关联 child Context
			// 这个 children map 在 parent Context 被取消时会被遍历然后批量调用 child Context 的取消方法
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else { // ok 为 false，说明: "parent Context 已经取消" 或 "根本无法取消" 或 "无法转换为一个 *cancelCtx" 或 "是一个自定义深度包装的 cancelCtx"
		atomic.AddInt32(&goroutines, +1)
		// 由于代码在方法开始时就判断了 parent Context "已经取消"、"根本无法取消" 这两种情况
		// 所以这两种情况在这里不会发生，因此 <-parent.Done() 不会产生 panic
		// 
		// 唯一剩下的可能就是 parent Context "无法转换为一个 *cancelCtx" 或 "是一个被覆盖了 done channel 的自定义 cancelCtx"
		// 这种两种情况下无法通过 parent Context 的 children map 建立关联，只能通过创建一个 Goroutine 来完成及联取消的操作
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}

// parentCancelCtx returns the underlying *cancelCtx for parent.
// It does this by looking up parent.Value(&cancelCtxKey) to find
// the innermost enclosing *cancelCtx and then checking whether
// parent.Done() matches that *cancelCtx. (If not, the *cancelCtx
// has been wrapped in a custom implementation providing a
// different done channel, in which case we should not bypass it.)
// parentCancelCtx 负责从 parent Context 中取出底层的 cancelCtx
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	// 如果 parent context 的 done 为 nil 说明不支持 cancel，那么就不可能是 cancelCtx
	// 如果 parent context 的 done 为 可复用的 closedchan 说明 parent context 已经 cancel 了
	// 此时取出 cancelCtx 没有意义(具体为啥没意义后面章节会有分析)
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}

	// 如果 parent context 属于原生的 *cancelCtx 或衍生类型(timerCtx) 需要继续进行后续判断
	// 如果 parent context 无法转换到 *cancelCtx，则认为非 cancelCtx，返回 nil,fasle
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	p.mu.Lock()
	// 经过上面的判断后，说明 parent context 可以被转换为 *cancelCtx，这时存在多种情况:
	//   - parent context 就是 *cancelCtx
	//   - parent context 是标准库中的 timerCtx
	//   - parent context 是个自己自定义包装的 cancelCtx
	//
	// 针对这 3 种情况需要进行判断，判断方法就是: 
	//   判断 parent context 通过 Done() 方法获取的 done channel 与 Value 查找到的 context 的 done channel 是否一致
	// 
	// 一致情况说明 parent context 为 cancelCtx 或 timerCtx 或 自定义的 cancelCtx 且未重写 Done()，
	// 这种情况下可以认为拿到了底层的 *cancelCtx
	// 
	// 不一致情况说明 parent context 是一个自定义的 cancelCtx 且重写了 Done() 方法，并且并未返回标准 *cancelCtx 的
	// 的 done channel，这种情况需要单独处理，故返回 nil, false
	ok = p.done == done
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```

### [](https://mritd.com/2021/06/27/golang-context-source-code/#3-2%E3%80%81cancelCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E5%8F%96%E6%B6%88%E7%9A%84 "3.2、cancelCtx 是如何取消的")3.2、cancelCtx 是如何取消的[](https://mritd.com/2021/06/27/golang-context-source-code/#3-2%E3%80%81cancelCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E5%8F%96%E6%B6%88%E7%9A%84)

在上面的 cancelCtx 创建源码中可以看到，cancelCtx 内部跨多个 Goroutine 实现信号传递其实靠的就是一个 done channel；**如果要取消这个 Context，那么就需要让所有 `<-c.Done()` 停止阻塞，这时候最简单的办法就是把这个 channel 直接 close 掉，或者干脆换成一个已经被 close 的 channel，**事实上官方也是怎么做的。

```
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    // 首先判断 err 是不是 nil，如果不是 nil 则直接 panic
    // 这么做的目的是因为 cancel 方法是个私有方法，标准库内任何调用 cancel
    // 的方法保证了一定会传入 err，如果没传那就是非正常调用，所以可以直接 panic
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	// 对 context 加锁，防止并发更改
	c.mu.Lock()
	// 如果加锁后有并发访问，那么二次判断 err 可以防止重复 cancel 调用
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	// 这里设置了内部的 err，所以上面的判断 c.err != nil 与这里是对应的
	// 也就是说加锁后一定有一个 Goroutine 先 cannel，cannel 后 c.err 一定不为 nil
	c.err = err
	// 判断内部的 done channel 是不是为 nil，因为在 context.WithCancel 创建 cancelCtx 的
	// 时候并未立即初始化 done channel(延迟初始化)，所以这里可能为 nil
	// 如果 done channel 为 nil，那么就把它设置成共享可重用的一个已经被关闭的 channel
	if c.done == nil {
		c.done = closedchan
	} else { // 如果 done channel 已经被初始化，则直接 close 它
		close(c.done)
	}
	// 如果当前 Context 下面还有关联的 child Context，且这些 child Context 都是
	// 可以转换成 *cancelCtx 的 Context(见上面的 propagateCancel 方法分析)，那么
	// 直接遍历 childre map，并且调用 child Context 的 cancel 即可
	// 如果关联的 child Context 不能转换成 *cancelCtx，那么由 propagateCancel 方法
	// 中已经创建了单独的 Goroutine 来关闭这些 child Context
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	// 清除 c.children map 并解锁
	c.children = nil
	c.mu.Unlock()

    // 如果 removeFromParent 为 true，那么从 parent Context 中清理掉自己
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

### [](https://mritd.com/2021/06/27/golang-context-source-code/#3-3%E3%80%81parentCancelCtx-%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E5%8F%96%E5%87%BA%E5%B7%B2%E5%8F%96%E6%B6%88%E7%9A%84-cancelCtx "3.3、parentCancelCtx 为什么不取出已取消的 cancelCtx")3.3、parentCancelCtx 为什么不取出已取消的 cancelCtx[](https://mritd.com/2021/06/27/golang-context-source-code/#3-3%E3%80%81parentCancelCtx-%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E5%8F%96%E5%87%BA%E5%B7%B2%E5%8F%96%E6%B6%88%E7%9A%84-cancelCtx)

在上面的 3.1 章节中分析 `parentCancelCtx` 方法时有这么一段:

```
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	// 如果 parent context 的 done 为 nil 说明不支持 cancel，那么就不可能是 cancelCtx
	// 如果 parent context 的 done 为 可复用的 closedchan 说明 parent context 已经 cancel 了
	// 此时取出 cancelCtx 没有意义(具体为啥没意义后面章节会有分析)
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}
	
	// ...... 省略
}
```

现在来仔细说明一下 “为什么没有意义？” 这个问题:

首先是调用 `parentCancelCtx` 方法的位置，在 context 包中只有两个位置调用了 `parentCancelCtx` 方法；一个是在创建 cancelCtx 的 `func WithCancel(parent Context)` 的 `propagateCancel(parent, &c)` 方法中，另一个就是 `cancel` 方法的 `removeChild(c.Context, c)` 调用中；下面分析一下这两个方法的目的。

#### [](https://mritd.com/2021/06/27/golang-context-source-code/#3-3-1%E3%80%81propagateCancel-parent-amp-c "3.3.1、propagateCancel(parent, &c)")3.3.1、propagateCancel(parent, &c)[](https://mritd.com/2021/06/27/golang-context-source-code/#3-3-1%E3%80%81propagateCancel-parent-amp-c)

`propagateCancel` 负责保证当 parent cancelCtx 在取消时能正确传递到 child Context；**那么它需要通过 `parentCancelCtx` 来确定 parent Context 是否是一个 cancelCtx，如果是那就把 child Context 加到 parent Context 的 children map 中，然后 parent Context 在 cancel 时会自动遍历 map 调用 child Context 的 cancel；如果不是那就开 Goroutine 阻塞读 parent Context 的 done channel然后再调用 child Context 的 cancel。**

```
if p, ok := parentCancelCtx(parent); ok {
    p.mu.Lock()
    if p.err != nil {
        // parent has already been canceled
        child.cancel(false, p.err)
    } else {
        if p.children == nil {
            p.children = make(map[canceler]struct{})
        }
        p.children[child] = struct{}{}
    }
    p.mu.Unlock()
} else {
    atomic.AddInt32(&goroutines, +1)
    go func() {
        select {
        case <-parent.Done():
            child.cancel(false, parent.Err())
        case <-child.Done():
        }
    }()
}
```

所以在这个方法调用时，**如果 `parentCancelCtx` 取出一个已取消的 cancelCtx，那么 parent Context 的 children map 在 cancel 时已经清空了，这时要是再给设置上就有问题了，同样业务需求中 `propagateCancel` 为了就是控制传播，明明 parent Context 已经 cancel 了，再去传播就没意义了。**

#### [](https://mritd.com/2021/06/27/golang-context-source-code/#3-3-2%E3%80%81removeChild-c-Context-c "3.3.2、removeChild(c.Context, c)")3.3.2、removeChild(c.Context, c)[](https://mritd.com/2021/06/27/golang-context-source-code/#3-3-2%E3%80%81removeChild-c-Context-c)

同上面的 3.3.1 一样，**`removeChild(c.Context, c)` 目的是在 cancel 时断开与 parent Context 的关联，同样是为了处理 children map 的问题；此时如果 `parentCancelCtx` 也取出一个已经 cancel 的 parent Context，由于 parent Context 在 cancel 时已经清空了 childre map，这里再尝试 remove 也没有任何意义。**

```
// removeChild removes a context from its parent.
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent)
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)
	}
	p.mu.Unlock()
}
```

## [](https://mritd.com/2021/06/27/golang-context-source-code/#%E5%9B%9B%E3%80%81timerCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90 "四、timerCtx 源码分析")四、timerCtx 源码分析[](https://mritd.com/2021/06/27/golang-context-source-code/#%E5%9B%9B%E3%80%81timerCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)

### [](https://mritd.com/2021/06/27/golang-context-source-code/#4-1%E3%80%81timerCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E5%88%9B%E5%BB%BA%E7%9A%84 "4.1、timerCtx 是如何创建的")4.1、timerCtx 是如何创建的[](https://mritd.com/2021/06/27/golang-context-source-code/#4-1%E3%80%81timerCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E5%88%9B%E5%BB%BA%E7%9A%84)

timerCtx 的创建主要通过 `context.WithDeadline` 方法，同时 `context.WithTimeout` 实际上也是调用的 `context.WithDeadline`:

```
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    // 与 cancelCtx 一样先检查一下 parent Context
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    
    // 判断 parent Context 是否支持 Deadline，如果支持的话需要判断 parent Context 的截止时间
    // 假设 parent Context 的截止时间早于当前设置的截止时间，那就意味着 parent Context 肯定会先
    // 被 cancel，同样由于 parent Context 的 cancel 会导致当前这个 child Context 也会被 cancel
    // 所以这时候直接返回一个 cancelCtx 就行了，计时器已经没有必要存在了
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}

    // 创建一个 timerCtx
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}

    // 与 cancelCtx 一样的传播操作
	propagateCancel(parent, c)

    // 判断当前时间已经已经过了截止日期，如果超过了直接 cancel
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}

    // 所有 check 都没问题的情况下，创建一个定时器，在到时间后自动 cancel
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

### [](https://mritd.com/2021/06/27/golang-context-source-code/#4-2%E3%80%81timerCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E5%8F%96%E6%B6%88%E7%9A%84 "4.2、timerCtx 是如何取消的")4.2、timerCtx 是如何取消的[](https://mritd.com/2021/06/27/golang-context-source-code/#4-2%E3%80%81timerCtx-%E6%98%AF%E5%A6%82%E4%BD%95%E5%8F%96%E6%B6%88%E7%9A%84)

了解了 cancelCtx 的取消流程以后再来看 timerCtx 的取消就相对简单的多，主要就是调用一下里面的 cancelCtx 的 cancel，然后再把定时器停掉:

```
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

## [](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%BA%94%E3%80%81valueCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90 "五、valueCtx 源码分析")五、valueCtx 源码分析[](https://mritd.com/2021/06/27/golang-context-source-code/#%E4%BA%94%E3%80%81valueCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)

相对于 cancelCtx 还有 timerCtx，valueCtx 实在是过于简单，因为它没有及联的取消逻辑，也没有过于复杂的 kv 存储:

```
// WithValue returns a copy of parent in which the value associated with key is
// val.
//
// Use context Values only for request-scoped data that transits processes and
// APIs, not for passing optional parameters to functions.
//
// The provided key must be comparable and should not be of type
// string or any other built-in type to avoid collisions between
// packages using context. Users of WithValue should define their own
// types for keys. To avoid allocating when assigning to an
// interface{}, context keys often have concrete type
// struct{}. Alternatively, exported context key variables' static
// type should be a pointer or interface.
// WithValue 方法负责创建 valueCtx
func WithValue(parent Context, key, val interface{}) Context {
    // parent 检测
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    // key 检测
	if key == nil {
		panic("nil key")
	}
    // key 必须是可比较的
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}

// stringify tries a bit to stringify v, without using fmt, since we don't
// want context depending on the unicode tables. This is only used by
// *valueCtx.String().
func stringify(v interface{}) string {
	switch s := v.(type) {
	case stringer:
		return s.String()
	case string:
		return s
	}
	return "<not Stringer>"
}

func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(type " +
		reflectlite.TypeOf(c.key).String() +
		", val " + stringify(c.val) + ")"
}

func (c *valueCtx) Value(key interface{}) interface{} {
    // 先判断当前 Context 里有没有这个 key
	if c.key == key {
		return c.val
	}
    // 如果没有递归向上查找
	return c.Context.Value(key)
}
```

## [](https://mritd.com/2021/06/27/golang-context-source-code/#%E5%85%AD%E3%80%81%E7%BB%93%E5%B0%BE "六、结尾")六、结尾[](https://mritd.com/2021/06/27/golang-context-source-code/#%E5%85%AD%E3%80%81%E7%BB%93%E5%B0%BE)

分析 Context 源码断断续续经历了 3、4 天，说心里话发现里面复杂情况有很多，网上其他文章很多都是只提了一嘴，但是没有深入具体逻辑，尤其是 cancelCtx 的相关调用；我甚至觉得我有些地方可能理解的也不完全正确，目前就先写到这里，如果有不对的地方欢迎补充。


# Reference
https://zhuanlan.zhihu.com/p/366050223
https://www.qtmuniao.com/2020/07/12/go-context/
https://mritd.com/2021/06/27/golang-context-source-code/#%E4%B8%89%E3%80%81cancelCtx-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90
