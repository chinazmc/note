#golang 


# 源码剖析

## Add()

![add](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd5b885429534a51bdfceb5c39996b0f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## Wait()

![wait](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/011e3568904743cbb68a8442b9d2cd20~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

```go
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}
```

WaitGroup 底层结构看起来简单，但 WaitGroup.state1 其实代表三个字段：counter，waiter，sema。

-   counter ：可以理解为一个计数器，计算经过 wg.Add(N), wg.Done() 后的值。
-   waiter ：当前等待 WaitGroup 任务结束的等待者数量。其实就是调用 wg.Wait() 的次数，所以通常这个值是 1 。
-   sema ： 信号量，用来唤醒 Wait() 函数。

## 为什么要将 counter 和 waiter 放在一起 ？

其实是为了保证 WaitGroup 状态的完整性。举个例子，看下面的一段源码

```go
// sync/waitgroup.go:L79 --> Add()
if v > 0 || w == 0 { // v => counter, w => waiter
    return
}
// ...
*statep = 0
for ; w != 0; w-- {
    runtime_Semrelease(semap, false, 0)
}
复制代码
```

当同时发现 wg.counter <= 0 && wg.waiter != 0 时，才会去唤醒等待的 waiters，让等待的协程继续运行。但是使用 WaitGroup 的调用方一般都是并发操作，如果不同时获取的 counter 和 waiter 的话，就会造成获取到的 counter 和 waiter 可能不匹配，造成程序 deadlock 或者程序提前结束等待。

## 如何获取 counter 和 waiter ?

对于 wg.state 的状态变更，WaitGroup 的 Add()，Wait() 是使用 atomic 来做原子计算的(为了避免锁竞争)。但是由于 atomic 需要使用者保证其 64 位对齐，所以将 counter 和 waiter 都设置成 uint32，同时作为一个变量，即满足了 atomic 的要求，同时也保证了获取 waiter 和 counter 的状态完整性。但这也就导致了 32位，64位机器上获取 state 的方式并不相同。如下图： ![waitgroup state](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a28df16cd0384dcbadd803b8db245a46~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp) 简单解释下：

因为 64 位机器上本身就能保证 64 位对齐，所以按照 64 位对齐来取数据，拿到 state1[0], state1[1] 本身就是64 位对齐的。但是 32 位机器上并不能保证 64 位对齐，因为 32 位机器是 4 字节对齐，如果也按照 64 位机器取 state[0]，state[1] 就有可能会造成 atmoic 的使用错误。

于是 32 位机器上空出第一个 32 位，也就使后面 64 位天然满足 64 位对齐，第一个 32 位放入 sema 刚好合适。早期 WaitGroup 的实现 sema 是和 state1 分开的，也就造成了使用 WaitGroup 就会造成 4 个字节浪费，不过 go1.11 之后就是现在的结构了。

## 为什么流程图里缺少了 Done ?

其实并不是，是因为 Done 的实现就是 Add. 只不过我们常规用法 wg.Add(1) 是加 1 ，wg.Done() 是减 1，即 wg.Done() 可以用 wg.Add(-1) 来代替。 尽管我们知道 wg.Add 可以传递负数当 wg.Done 使用，但是还是别这么用。

## 退出waitgroup的条件

其实就一个条件， WaitGroup.counter 等于 0

# 日常开发中特殊需求

## 1. 控制超时/错误控制

虽说 WaitGroup 能够让主 Goroutine 等待子 Goroutine 退出，但是 WaitGroup 遇到一些特殊的需求，如：超时，错误控制，并不能很好的满足，需要做一些特殊的处理。

用户在电商平台中购买某个货物，为了计算用户能优惠的金额，需要去获取 A 系统（权益系统），B 系统（角色系统），C 系统（商品系统），D 系统（xx系统）。为了提高程序性能，可能会同时发起多个 Goroutine 去访问这些系统，必然会使用 WaitGroup 等待数据的返回，但是存在一些问题：

1.  当某个系统发生错误，等待的 Goroutine 如何感知这些错误？
2.  当某个系统响应过慢，等待的 Goroutine 如何控制访问超时？

这些问题都是直接使用 WaitGroup 没法处理的。如果直接使用 channel 配合 WaitGroup 来控制超时和错误返回的话，封装起来并不简单，而且还容易出错。我们可以采用 ErrGroup 来代替 WaitGroup。

有关 ErrGroup 的用法这里就不再阐述。golang.org/x/sync/errgroup

```go
package main

import (
	"context"
	"fmt"
	"golang.org/x/sync/errgroup"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	defer cancel()
	errGroup, newCtx := errgroup.WithContext(ctx)

	done := make(chan struct{})
	go func() {
		for i := 0; i < 10; i++ {
			errGroup.Go(func() error {
				time.Sleep(time.Second * 10)
				return nil
			})
		}
		if err := errGroup.Wait(); err != nil {
			fmt.Printf("do err:%v\n", err)
			return
		}
		done <- struct{}{}
	}()

	select {
	case <-newCtx.Done():
		fmt.Printf("err:%v ", newCtx.Err())
		return
	case <-done:
	}
	fmt.Println("success")
}
复制代码
```

## 2. 控制 Goroutine 数量

场景模拟： 大概有 2000 - 3000 万个数据需要处理，根据对服务器的测试，当启动 200 个 Goroutine 处理时性能最佳。如何控制？

遇到诸如此类的问题时，单纯使用 WaitGroup 是不行的。既要保证所有的数据都能被处理，同时也要保证同时最多只有 200 个 Goroutine。这种问题需要 WaitGroup 配合 Channel 一块使用。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg = sync.WaitGroup{}
	manyDataList := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	ch := make(chan bool, 3)
	for _, v := range manyDataList {
		wg.Add(1)
		go func(data int) {
			defer wg.Done()

			ch <- true
			fmt.Printf("go func: %d, time: %d\n", data, time.Now().Unix())
			time.Sleep(time.Second)
			<-ch
		}(v)
	}
	wg.Wait()
}
```

  
# 二、源码分析

![](https://pic1.zhimg.com/80/v2-42374414d1fb69d44606f07b16a93c48_720w.jpg)

WaitGroup结构体

## onCopy机制

> Go中没有原生的禁止拷贝的方式，所以如果有的结构体，你希望使用者无法拷贝，只能指针传递保证全局唯一的话，可以这么干，定义一个结构体叫noCopy，要实现sync.Locker 这个接口。

```go
type noCopy struct{}

// nocopy 只有在使用 go vet 检查时才能显示错误，编译正常
func (*noCopy) Lock() {}
func (*noCopy) UnLock() {}
```

## state1字段处理

> 总共分配了12个字节，在这里被设计成三种状态。其中对齐的8个字节作为状态位(state)，高32位为记录计数器的数量，低32位为等待goroutine的数量值。其余的4个字节作为信号量存储(sema)。由于操作系统分为32位和64位，64位的原子操作需要64位对齐，但是32位编译器保证不了，于是这里就采用了动态识别当前我们操作的64位数到底是不是在8字节对齐的位置上面。具体见源码state方法

这个操作巧妙在哪里呢？

-   如果是 64 位的机器那肯定是 8 字节对齐了的，所以是上面第一种方式
-   如果在 32 位的机器上
    -   如果恰好 8 字节对齐了，那么也是第一种方式取前面的 8 字节数据
    -   如果是没有对齐，但是 32 位 4 字节是对齐了的，所以我们只需要后移四个字节，那么就 8 字节对齐了，所以是第二种方式

所以通过 sema 信号量这四个字节的位置不同，保证了 counter 这个字段无论在 32 位还是 64 为机器上都是 8 字节对齐的，后续做 64 位原子操作的时候就没问题了。  
这个实现是在 `state` 方法实现的
```go
 // 得到state1的状态位和信号量
 func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
     if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
         // 如果地址是64bit对齐的，数组前两个元素做state，后一个元素做信号量
         return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
     } else {
         // 如果地址是32bit对齐的，数组后两个元素用来做state，它可以用来做64bit的原子操作，第一个元素32bit用来做信号量
         return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
     }
}
```

![](https://pic1.zhimg.com/80/v2-6b277a75ca2b8aaa3d1e0f328cf85f64_720w.jpg)

## Add方法实现

> 主要操作的state1字段中计数值部分，计数器部分的逻辑主要是通过state()，在上面有提及。每次调用Add方法就会增加相应数量的计数器。如果计数器为零，则释放等待时阻塞的所有goroutine。如果计数器变为负数，请添加恐慌。如果计数器值大于0，说明此时还有任务没有完成，那么调用者就变成等待者，需要加入wait队列，并且阻塞自己。参数可正可负数。

```go
func (wg *WaitGroup) Add(delta int) {
    //获取state1中的状态位和信号量位
    statep, semap := wg.state()
    //用来goroutine的竞争检测，可忽略。
    if race.Enabled {
        _ = *statep 
        if delta < 0 {
            race.ReleaseMerge(unsafe.Pointer(wg))
        }
        race.Disable()
        defer race.Enable()
    }
    // uint64(delta)<<32 将delta左移32
    // 因为高32位表示计数器，所以delta左移32位，
    // 增加到计数位。
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    // 当前计数器的值
    v := int32(state >> 32)
    // 阻塞的wait goroutine数量
    w := uint32(state)
    if race.Enabled && delta > 0 && v == int32(delta) {
        race.Read(unsafe.Pointer(semap))
    }
    // 计数器的值<0,panic
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    // 当wait goroutine数量不为0时，累加后的counter值和delta相等,
    // 说明Add()和Wait()同时调用了,所以发生panic,
    // 因为正确的做法是先Add()后Wait()，
    // 也就是已经调用了wait()就不允许再添加任务了
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // add调用结束
    if v > 0 || w == 0 {
        return
    }
    // 能走到这里说明当前Goroutine Counter计数器为0，
    // Waiter Counter计数器大于0, 
    // 到这里数据也就是允许发生变动了，如果发生变动了，则出发panic
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 所有的状态位清0
    *statep = 0
    for ; w != 0; w-- {
        // 首先让信号量加一，然后检查是否有正在等待的Goroutine，如果没有，直接返回；
        // 如果有，调用goready函数唤醒一个Goroutine。
        runtime_Semrelease(semap, false, 0)
    }
}
```

![](https://pic2.zhimg.com/80/v2-7467f34d2932b2a3a74cb9c1df191b3d_720w.jpg)

## Done方法实现

> 内部调用了Add(-1)的操作，具体看Add方法实现部分

```go
//Done方法其实就是Add（-1）
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

## Wait方法实现

> 阻塞主goroutine直到WaitGroup计数器变为0。

```go
// 等待并阻塞，直到WaitGroup计数器为0
func (wg *WaitGroup) Wait() {
    // 获取waitgroup状态位和信号量
    statep, semap := wg.state() 
    if race.Enabled { 
        _ = *statep 
        race.Disable()
    }
    for {
        // 使用原子操作读取state，是为了保证Add中的写入操作已经完成
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32) //获取计数器(高32位)
        w := uint32(state) //获取wait goroutine数量（低32位）
        if v == 0 { // 计数器为0，跳出死循环,不用阻塞
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }
        // 使用CAS操作对`waiter Counter`计数器进行+1操作，
        // 外面有for循环保证这里可以进行重试操作
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            if race.Enabled && w == 0 {
                race.Write(unsafe.Pointer(semap))
            }
            // 在这里获取信号量，使线程进入睡眠状态，
            // 与Add方法中runtime_Semrelease增加信号量相对应，
            // 也就是当最后一个任务调用Done方法
            // 后会调用Add方法对goroutine counter的值减到0，
            // 就会走到最后的增加信号量
            runtime_Semacquire(semap)
            // 在Add方法中增加信号量时已经将statep的值设为0了，
            // 如果这里不是0，说明在wait之后又调用了Add方法，
            // 使用时机不对，触发panic
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }
    }
}
```

![](https://pic4.zhimg.com/80/v2-1820064b332c2efb80c7314a70130387_720w.jpg)

# Reference
https://zhuanlan.zhihu.com/p/365288361