#golang 

### 一、什么是sync.Once

> 官方文档对它的描述是：一个对象将完全执行一次，不得复制。常常用来单例对象的初始化场景，或者并发访问只需要初始化一次的共享资源。sync.Once只暴露了一个方法Do，可以多次调用，但是只有第一次调用Do方法时f参数才会执行，这里的f是一个无参数无返回值的函数。下面来看下官方给出来的demo

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
// 结果只打印一次：only once
```

### 二、源码分析

> sync.Once的源代码还很少的，直接在代码里面作分析

```go
package sync
 
import (
     "sync/atomic"
 )
type Once struct {
     done uint32 // 初始值为0表示还未执行过，1表示已经执行过
     m    Mutex  // m为互斥锁
}
func (o *Once) Do(f func()) {
    // 判断done是否为0.若为0，表示未执行过，调用doSlow方法
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}
func (o *Once) doSlow(f func()) {
    // 互斥锁加锁解锁
    o.m.Lock()
    defer o.m.Unlock()
    // 加锁判断done是否为0
    if o.done == 0 {
        // 执行完f()函数后，将done值设置为1
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

> Do 方法流程图如下：

![](https://pic2.zhimg.com/80/v2-e4f079a97f92f4ec3bf6981f263eeffd_720w.jpg)

Do方法流程图

-   为什么在do方法的时候没有用到cas原子判断？

> 在do方法源码注释中的时候有这么一段话，说如果是使用atomic.CompareAndSwapUint32(&o.done, 0, 1）此方法不会保证实现f函数调用完成为什么会这么说，同时进行两次调用，cas的获胜者将调用f函数，失败者将立即返回，而无需等待第一个对f的调用完成。此时f函数还在执行过程中，你就已经回复了once.Do成功了。此时全局变量还没有创建出来，行为是无法定义的。那么怎么解决呢？有以下两个思路：1）热路径：用原子读done的值，保证竞态条件正确；2） 加锁：既然不能用cas原子操作，那就用加锁的方式来保证原子性，如果done\==0，那么走慢路径，先加锁，然后在执行f函数，最后将done设置为1，并解锁。

-   千万不要把同一sync.Once用在嵌套结构中，这样非常容易形成死锁！

```go
func once() {
    once := &sync.Once{}
    once.Do(func() {
        once.Do(func() {
            fmt.Println("test nestedDo")
        })
    })
    //输出：fatal error: all goroutines are asleep - deadlock!
```

### 三、sync.Once实现单例模式

> 单例模式可以说是设计模式里面最简单的了，Go可以借助sync.Once来实现，代码如下：

```go
package main
 
 import (
     "fmt"
     "sync"
 )
type Singleton struct{}
var (
    singleton *Singleton
    once      sync.Once
)
func GetSingleton() *Singleton {
    once.Do(func() {
        fmt.Println("Create Obj")
        singleton = new(Singleton)
    })
    return singleton
}
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            obj := GetSingleton()
            fmt.Printf("%p\n", obj)
            wg.Done()
        }()
    }
    wg.Wait()
    // 只打印一次 Create Obj
}
```

