对于一些初学者，自知道 Go 里面的 array 以 pass-by-value 方式传递后，就莫名地引起 “恐慌”。外加诸多文章未作说明，就建议用 slice 代替 array，企图避免数据拷贝，提升性能。实际上，此做法有待商榷。某些时候怕会适得其反，倒造成不必要的性能损失。

用个简单的示例说明。

```go
package main

import (
    "fmt"
)

const capacity = 1024

func array() [capacity]int {
    var d [capacity]int

    for i := 0; i < len(d); i++ {
        d[i] = 1
    }

    return d
}

func slice() []int {
    d := make([]int, capacity)

    for i := 0; i < len(d); i++ {
        d[i] = 1
    }

    return d
}

func main() {
    fmt.Println(array())
    fmt.Println(slice())
}
```

代码很简单，两个函数分别返回 “内容相同” 的 array 和 slice。为避免编译器优化，特填充了全部数据，以模拟 “真实” 数据复制行为。接下来，看看性能测试对比。

```go
package main

import (
    "testing"
)

func BenchmarkArray(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = array()
    }
}

func BenchmarkSlice(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = slice()
    }

}
```

![图片描述](https://segmentfault.com/img/bVvb3K "图片描述")

这结果怕是颠覆了最初认知。array 非但拥有更好的性能，还避免了堆内存分配，也就是说减轻了 GC 压力。为什么会这样？

熟悉汇编的，怕是很容易看出来。函数 array 返回值的复制只需用 "CX + REP" 指令就可完成。

![图片描述](https://segmentfault.com/img/bVvb3P "图片描述")

整个 array 函数完全在栈上完成，而 slice 函数则需执行 makeslice，继而在堆上分配内存，这就是问题所在。

![图片描述](https://segmentfault.com/img/bVvb3Q "图片描述")

对于一些短小的对象，复制成本远小于在堆上分配和回收操作。

> Go Proverbs: A little copying is better than a little dependency.

# 二
延迟调用（defer）确实是一种 “优雅” 机制。可简化代码，并确保即便发生 panic 依然会被执行。如将 panic/recover 比作 try/except，那么 defer 似乎可看做 finally。

如同异常保护被滥用一样，defer 被无限制使用的例子比比皆是。

![图片描述](https://segmentfault.com/img/bVvfVC "图片描述")  
![图片描述](https://segmentfault.com/img/bVvfVD "图片描述")

只需稍稍了解 defer 实现机制，就不难理解会有这样的性能差异。

编译器通过 runtime.deferproc “注册” 延迟调用，除目标函数地址外，还会复制相关参数（包括 receiver）。在函数返回前，执行 runtime.deferreturn 提取相关信息执行延迟调用。这其中的代价自然不是普通函数调用一条 CALL 指令所能比拟的。

![图片描述](https://segmentfault.com/img/bVvfVE "图片描述")  
![图片描述](https://segmentfault.com/img/bVvfVH "图片描述")

或许你会觉得 4x 的性能差异算不得什么，但如果是下面这样呢？

![图片描述](https://segmentfault.com/img/bVvfVI "图片描述")

当多个 goroutine 执行该函数时，只怕性能差异就不是 4x，还得算上 httpGet 所需时间。原本的并发设计，因为错误的 defer 调用变成 “串行”。

与之类似的，还有下面这样的写法。

![图片描述](https://segmentfault.com/img/bVvfXg "图片描述")

如果 files 是个 “超大” 列表，只怕在 analysis 结束前，会有不小的隐式 “资源泄露”，这些不能及时回收的对象，会导致 GC 在内的相关性能问题。

解决方法么，要么去掉 f.close 前的 defer，要么将内层处理逻辑重构为独立函数（比如匿名函数调用）。

![图片描述](https://segmentfault.com/img/bVvfVR "图片描述")

除此之外，单个函数里过多的 defer 调用可尝试合并。最起码，在并发竞争激烈时，mutex.Unlock 不应该使用 defer，而应尽快执行，仅保护最短的代码片段。

