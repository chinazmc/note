#golang 

本篇文章会从源码角度去深入剖析下 sync.Cond。Go 日常开发中 sync.Cond 可能是我们用的较少的控制并发的手段，因为大部分场景下都被 Channel 代替了。还有就是 sync.Cond 使用确实也蛮复杂的。

比如下面这段代码：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	done := make(chan int, 1)

	go func() {
		time.Sleep(5 * time.Second)
		done <- 1
	}()

	fmt.Println("waiting")
	<-done
	fmt.Println("done")
}
复制代码
```

同样可以使用 sync.Cond 来实现

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	cond := sync.NewCond(&sync.Mutex{})
	var flag bool
	go func() {
		time.Sleep(time.Second * 5)
		cond.L.Lock()
		flag = true
		cond.Signal()
		cond.L.Unlock()
	}()

	fmt.Println("waiting")
	cond.L.Lock()
	for !flag {
		cond.Wait()
	}
	cond.L.Unlock()
	fmt.Println("done")
}
复制代码
```

大部分场景下使用 channel 是比 sync.Cond方便的。不过我们要注意到，sync.Cond 提供了 Broadcast 方法，可以通知所有的等待者。想利用 channel 实现这个方法还是不容易的。我想这应该是 sync.Cond 唯一有用武之地的地方。

先列出来一些问题吧，可以带着这些问题来阅读本文：

1.  cond.Wait本身就是阻塞状态，为什么 cond.Wait 需要在循环内 ？
2.  sync.Cond 如何触发不能复制的 panic ?
3.  为什么 sync.Cond 不能被复制 ？
4.  cond.Signal 是如何通知一个等待的 goroutine ?
5.  cond.Broadcast 是如何通知等待的 goroutine 的？

## 源码剖析

![cond wait](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c73e6e28543049f6b694cd7c24b29411~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![cond signal](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92904b1d8f3d4bc780970698346ecc45~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![cond broadcast](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdfddbe288004efe93b80db3746810aa~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![cond 排队](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/260df4f4ad5445baa833fda00beca6eb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### cond.Wait 是阻塞的吗？是如何阻塞的？

是阻塞的。不过不是 sleep 这样阻塞的。

调用 `goparkunlock` 解除当前 goroutine 的 m 的绑定关系，将当前 goroutine 状态机切换为等待状态。等待后续 goready 函数时候能够恢复现场。

### cond.Signal 是如何通知一个等待的 goroutine ?

1.  判断是否有没有被唤醒的 goroutine，如果都已经唤醒了，直接就返回了
2.  将已通知 goroutine 的数量加1
3.  从等待唤醒的 goroutine 队列中，获取 head 指针指向的 goroutine，将其重新加入调度
4.  被阻塞的 goroutine 可以继续执行

### cond.Broadcast 是如何通知等待的 goroutine 的？

1.  判断是否有没有被唤醒的 goroutine，如果都已经唤醒了，直接就返回了
2.  将等待通知的 goroutine 数量和已经通知过的 goroutine 数量设置成相等
3.  遍历等待唤醒的 goroutine 队列，将所有的等待的 goroutine 都重新加入调度
4.  所有被阻塞的 goroutine 可以继续执行

### cond.Wait本身就是阻塞状态，为什么 cond.Wait 需要在循环内 ？

我们能注意到，调用 cond.Wait 的位置，使用的是 for 的方式来调用 wait 函数，而不是使用 if 语句。

这是由于 wait 函数被唤醒时，存在虚假唤醒等情况，导致唤醒后发现，条件依旧不成立。因此需要使用 for 语句来循环地进行等待，直到条件成立为止。

## 使用中注意点

### 1. 不能不加锁直接调用 cond.Wait

```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
复制代码
```

我们看到 Wait 内部会先调用 c.L.Unlock()，来先释放锁。如果调用方不先加锁的话，会触发“fatal error: sync: unlock of unlocked mutex”。关于 mutex 的使用方法，推荐阅读下《这可能是最容易理解的 Go Mutex 源码剖析》

### 2. 为什么不能 sync.Cond 不能复制 ？

sync.Cond 不能被复制的原因，并不是因为 sync.Cond 内部嵌套了 Locker。因为 NewCond 时传入的 Mutex/RWMutex 指针，对于 Mutex 指针复制是没有问题的。

主要原因是 sync.Cond 内部是维护着一个 notifyList。如果这个队列被复制的话，那么就在并发场景下导致不同 goroutine 之间操作的 notifyList.wait、notifyList.notify 并不是同一个，这会导致出现有些 goroutine 会一直堵塞。

这里有留下一个问题，sync.Cond 内部是有一段代码 check sync.Cond 是不能被复制的，下面这段代码能触发这个 panic 吗？

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	cond1 := sync.NewCond(new(sync.Mutex))
	cond := *cond1
	fmt.Println(cond)
}
```

# Reference
https://juejin.cn/post/6954183614099619854  
