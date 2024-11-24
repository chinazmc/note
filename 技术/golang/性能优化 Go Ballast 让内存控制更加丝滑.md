#golang 


关于 Go GC 优化的手段你知道的有哪些？比较常见的是通过调整 GC 的步调，以调整 GC 的触发频率。

-   设置 GOGC
-   设置 debug.SetGCPercent()

这两种方式的原理和效果都是一样的，GOGC 默认值是 100，也就是下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍。

我们都知道 GO 的 GC 是标记-清除方式，当 GC 会触发时全量遍历变量进行标记，当标记结束后执行清除，把标记为白色的对象执行垃圾回收。值得注意的是，这里的**回收**仅仅是标记内存可以返回给操作系统，并不是立即回收，这就是你看到 Go 应用 RSS 一直居高不下的原因。在整个垃圾回收过程中会暂停整个 Go 程序（STW），Go 垃圾回收的耗时还是主要取决于**标记**花费的时间的长短，清除过程是非常快的。

## **设置 GOGC 的弊端**

### **1. GOGC 设置比率的方式不精确**

设置 GOGC 基本上我们比较常用的 Go GC 调优的方式，大部分情况下其实我们并不需要调整 GOGC 就可以，一方面是不涉及内存密集型的程序本身对内存敏感程度太低，另外就是 GOGC 这种设置比率的方式不精确，我们很难精确的控制我们想要的触发的垃圾回收的阈值。

### **2. GOGC 设置过小**

GOGC 设置的非常小，会频繁触发 GC 导致太多无效的 CPU 浪费，反应到程序的表现就会特别明显。举个例子，对于 API 接口来说，导致的结果的就是接口周期性的耗时变化。这个时候你抓取 CPU profile 来看，大部分的耗时都集中在 GC 的相关处理上。

![](https://ask.qcloudimg.com/http-save/yehe-5709307/066399d39aafdd34cff619c93a4d894d.png?imageView2/2/w/1620)

如上图，这是一次 prometheus 的查询操作，我们看到大部分的 CPU 都消耗在 GC 的操作上。这也是生产环境遇到的，由于 GOGC 设置的过小，导致过多的消耗都耗费在 GC 上。

### **3. 对某些程序本身占用内存就低，容易触发 GC**

对 API 接口耗时比较敏感的业务，如果 GOGC 置默认值的时候，也可能也会遇到接口的周期性的耗时波动。这是为什么呢？

因为这种接口本身占用内存比较低，每次 GC 之后本身占的内存比较低，如果按照上次 GC 后的 heap 的一倍的 GC 步调来设置 GOGC 的话，这个阈值其实是很容易就能够触发，于是就很容出现接口因为 GC 的触发导致额外的消耗。

### **4. GOGC 设置很大，有的时候又容易触发 OOM**

那如何调整呢？是不是把 GOGC 设置的越大越好呢？这样确实能够降低 GC 的触发频率，但是这个值需要设置特别大才有效果。这样带来的问题，GOGC 设置的过大，如果这些接口突然接受到一大波流量，由于长时间无法触发 GC 可能导致 OOM。

由此，GOGC 对于某些场景并不是很友好，那有没有能够精确控制内存，让其在 10G 的倍数时准确控制 GC 呢？

## **GO 内存 ballast**

这就需要 Go ballast 出场了。什么是 Go ballast，其实很简单就是初始化一个生命周期贯穿整个 Go 应用生命周期的超大 slice。

```javascript
func main() {
  ballast := make([]byte, 10*1024*1024*1024) // 10G 
  
  // do something
  
  runtime.KeepAlive(ballast)
}
```

上面的代码就初始化了一个 ballast，利用 runtime.KeepAlive 来保证 ballast 不会被 GC 给回收掉。

利用这个特性，就能保证 GC 在 10G 的一倍时才能被触发，这样就能够比较精准控制 GO GC 的触发时机。

**这里你可能有一个疑问，这里初始化一个 10G 的数组，不就占用了 10 G 的物理内存呢？** 答案其实是不会的。

```javascript
package main

import (
    "runtime"
    "math"
    "time"
)

func main() {
    ballast := make([]byte, 10*1024*1024*1024)

    <-time.After(time.Duration(math.MaxInt64))
    runtime.KeepAlive(ballast)
}
```

```javascript
$ ps -eo pmem,comm,pid,maj_flt,min_flt,rss,vsz --sort -rss | numfmt --header --to=iec --field 5 | numfmt --header --from-unit=1024 --to=iec --field 6 | column -t | egrep "[t]est|[P]I"

%MEM  COMMAND   PID    MAJFL      MINFL  RSS    VSZ
0.1   test      12859  0          1.6K   344M   11530184
```

这个结果是在 CentOS Linux release 7.9 验证的，我们看到占用的 RSS 真实的物理内存只有 344M，但是 VSZ 虚拟内存确实有 10G 的占用。

延伸一点，当怀疑我们的接口的耗时是由于 GC 的频繁触发引起的，我们需要怎么确定呢？首先你会想到周期性的抓取 pprof 的来分析，这种方案其实也可以，但是太麻烦了。其实可以根据 GC 的触发时间绘制这个曲线图，GC 的触发时间可以利用 runtime.Memstats 的 LastGC 来获取。

## **生产环境验证**

-   绿线 调整前 GOGC = 30
-   黄线 调整后 GOGC 默认值，ballast = 50G

![](https://ask.qcloudimg.com/http-save/yehe-5709307/4eee2ab04a30c1ff500d082efc082794.png?imageView2/2/w/1620)

![](https://ask.qcloudimg.com/http-save/yehe-5709307/da9dc2b09183fa09ec88d9400a735f6c.png?imageView2/2/w/1620)

这张图相同的流量压力下，ballast 的表现明显偏好

![](https://ask.qcloudimg.com/http-save/yehe-5709307/2a50fff21a20eb8257a31faa328a2ea6.png?imageView2/2/w/1620)


# Reference
twitch.tv 这篇文章: _https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/_
https://cloud.tencent.com/developer/article/1903097
