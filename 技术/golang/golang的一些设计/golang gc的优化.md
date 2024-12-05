#review/golang性能优化 
## 理论

GC 和内存分配方式是强相关的两个技术，因此在分析两者的设计原理之时，要结合起来一起看。

### GC 算法

标记-清除  
标记-整理  
标记-复制  
分代收集  
[关于以上算法的简单介绍](https://link.segmentfault.com/?enc=eW%2Bcszj0zrSoxjya%2Fk%2BWtA%3D%3D.TJhdBh5TF7ycoOJpDYYBUNq%2FCBmQ8l5eznE5pzFy03KreD%2BI1tS5nxTInnsaquHKRPlVb%2Bwti2PUvfPE3ycqag%3D%3D)

### 内存分配方式

#### 线性分配

> 线性分配（Bump Allocator）是一种高效的内存分配方法，但是有较大的局限性。当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，即移动下图中的指针：  
> ![图片](https://segmentfault.com/img/remote/1460000021948710 "图片")  
> 线性分配器虽然线性分配器实现为它带来了较快的执行速度以及较低的实现复杂度，但是线性分配器无法在内存被释放时重用内存。如下图所示，如果已经分配的内存被回收，线性分配器无法重新利用红色的内存：  
> ![图片](https://segmentfault.com/img/remote/1460000021948712 "图片")  
> 线性分配器回收内存因为线性分配器具有上述特性，所以需要与合适的垃圾回收算法配合使用，例如：标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法，它们可以通过拷贝的方式整理存活对象的碎片，将空闲内存定期合并，这样就能利用线性分配器的效率提升内存分配器的性能了。因为线性分配器需要与具有拷贝特性的垃圾回收算法配合，所以 C 和 C++ 等需要直接对外暴露指针的语言就无法使用该策略。
> 
> 引用自：[https://draveness.me/golang/d...](https://link.segmentfault.com/?enc=FJ2923Pcv0AYMgOZ6TQgxg%3D%3D.uBqealMYFKrgz84Mvv2pkDKJRtLgwy293LWATAVJTwm7UP4yLxCSLynpbZbDGbm%2Fy3qcIq8nkxC2w1QcQed6OqB8THgpembphf3PZ4y8reFrOkjmO4yadM3XetUG%2BNKNqwQIaU5GNj3nOX4KvwXUbscYQna%2ByKrHBmswuXWmLmQ%3D)

应用代表：Java（如果使用 Serial, ParNew 等带有 Compact 过程的收集器时，采用分配的方式为线性分配）  
问题：内存碎片  
解决方式：GC 算法中加入「复制/整理」阶段

#### 空闲链表分配

> 空闲链表分配器（Free-List Allocator）可以重用已经被释放的内存，它在内部会维护一个类似链表的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表：  
> ![图片](https://segmentfault.com/img/remote/1460000021948713 "图片")  
> 空闲链表分配器因为不同的内存块通过指针构成了链表，所以使用这种方式的分配器可以重新利用回收的资源，但是因为分配内存时需要遍历链表，所以它的时间复杂度是 𝑂(𝑛)。空闲链表分配器可以选择不同的策略在链表中的内存块中进行选择，最常见的是以下四种：
> 
> -   首次适应（First-Fit）— 从链表头开始遍历，选择第一个大小大于申请内存的内存块；
> -   循环首次适应（Next-Fit）— 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块；
> -   最优适应（Best-Fit）— 从链表头遍历整个链表，选择最合适的内存块；
> -   隔离适应（Segregated-Fit）— 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；
> 
> 引用自：[https://draveness.me/golang/d...](https://link.segmentfault.com/?enc=j4eIGBL6wR3EVF81OUNq8A%3D%3D.%2By7eAFA6tld6aUKAdkk%2BAmNRLBVgu%2FmCBhzdtZiWHUAjsO1Koi6oQeKDAIc0cLa9jR%2FfjS6jl5NCS2jABb1B5YBgLt%2BaFt1FVrCAzWFZCepoRUi%2FTRxpzAq7OK%2BXD%2B8u1RgFuGapaseiDEjby5OU%2FOC%2FVGFvhSr6FOzUTfU0sBU%3D)

应用代表：GO、Java（如果使用 CMS 这种基于标记-清除，采用分配的方式为空闲链表分配）  
问题：相比线性分配方式的 bump-pointer 分配操作（top += size），空闲链表的分配操作过重，例如在 GO 程序的 pprof 图中经常可以看到 mallocgc() 占用了比较多的 CPU；

## 在 Golang 里的应用

完整讲解：[https://time.geekbang.org/col...](https://link.segmentfault.com/?enc=ZMUQUb8h5KaxRsrZ1toGwg%3D%3D.oYEl9vW1XO6USIzaxSebPNRVkzBNXVwLhq8tMC9pzom77qYZ%2BPa9fW8duKSUJLVu)

### 内存分配方式

Golang 采用了基于空闲链表分配方式的 TCMalloc 算法。

> **关于 TCMalloc**  
> [官方文档](https://link.segmentfault.com/?enc=rv91xbOEDNtLk3xf%2F0MiJg%3D%3D.thDniKb2qC1jsBZUF9Ls%2FXloW1sVnB5h44ltLF6AtoUw3Dw%2Bw8mhw3eDtoJDtnnV8zo4rJS50pI4DLG07tcYEA%3D%3D)
> 
> ![图片](https://segmentfault.com/img/remote/1460000042910691 "图片")
> 
> -   Front-end：它是一个内存缓存，提供了快速分配和重分配内存给应用的功能。它主要有2部分组成：Per-thread cache 和 Per-CPU cache。
> -   Middle-end：职责是给Front-end提供缓存。也就是说当Front-end缓存内存不够用时，从Middle-end申请内存。它主要是 Central free list 这部分内容。
> -   Back-end：这一块是负责从操作系统获取内存，并给Middle-end提供缓存使用。它主要涉及 Page Heap 内容。
> 
> TCMalloc将整个虚拟内存空间划分为n个同等大小的Page。将n个连续的page连接在一起组成一个Span。  
> PageHeap向OS申请内存，申请的span可能只有一个page，也可能有n个page。ThreadCache内存不够用会向CentralCache申请，CentralCache内存不够用时会向PageHeap申请，PageHeap不够用就会向OS操作系统申请。
> 
> 引用自：  
> [https://www.cnblogs.com/jiuju...](https://link.segmentfault.com/?enc=wsiswDkPYFdNmp4%2Fs1CXqA%3D%3D.nPBiMPxhMEFnAKhvHqd6v2Nu3neAhkUmhG81ppZx4mWH5Ypyka6eENF3Rgj%2BB9uo)

### GC 算法

Golang 采用了基于并发标记与清扫算法的三色标记法。

1.  Golang GC 的四个阶段
    
    > Mark Prepare - STW  
    > 做标记阶段的准备工作，需要停止所有正在运行的goroutine(即STW)，标记根对象，启用内存屏障，内存屏障有点像内存读写钩子，它用于在后续并发标记的过程中，维护三色标记的完备性(三色不变性)，这个过程通常很快，大概在10-30微秒。
    > 
    > Marking - Concurrent  
    > 标记阶段会将大概25%(gcBackgroundUtilization)的P用于标记对象，逐个扫描所有G的堆栈，执行三色标记，在这个过程中，所有新分配的对象都是黑色，被扫描的G会被暂停，扫描完成后恢复，这部分工作叫后台标记(gcBgMarkWorker)。这会降低系统大概25%的吞吐量，比如MAXPROCS=6，那么GC P期望使用率为6*0.25=1.5，这150%P会通过专职(Dedicated)/兼职(Fractional)/懒散(Idle)三种工作模式的Worker共同来完成。  
    > 这还没完，为了保证在Marking过程中，其它G分配堆内存太快，导致Mark跟不上Allocate的速度，还需要其它G配合做一部分标记的工作，这部分工作叫辅助标记(mutator assists)。在Marking期间，每次G分配内存都会更新它的”负债指数”(gcAssistBytes)，分配得越快，gcAssistBytes越大，这个指数乘以全局的”负载汇率”(assistWorkPerByte)，就得到这个G需要帮忙Marking的内存大小(这个计算过程叫revise)，也就是它在本次分配的mutator assists工作量(gcAssistAlloc)。
    > 
    > Mark Termination - STW  
    > 标记阶段的最后工作是Mark Termination，关闭内存屏障，停止后台标记以及辅助标记，做一些清理工作，整个过程也需要STW，大概需要60-90微秒。在此之后，所有的P都能继续为应用程序G服务了。
    > 
    > Sweeping - Concurrent  
    > 在标记工作完成之后，剩下的就是清理过程了，清理过程的本质是将没有被使用的内存块整理回收给上一个内存管理层级(mcache -> mcentral -> mheap -> OS)，清理回收的开销被平摊到应用程序的每次内存分配操作中，直到所有内存都Sweeping完成。当然每个层级不会全部将待清理内存都归还给上一级，避免下次分配再申请的开销，比如Go1.12对mheap归还OS内存做了优化，使用NADV_FREE延迟归还内存。
    > 
    > 引用自：[https://wudaijun.com/2020/01/...](https://link.segmentfault.com/?enc=6Elwv1qnMntUtwXfPMPhSw%3D%3D.j1IPSoNxcbQgfZZEorv77MT72SFmOIaCCodx6wphCzj%2B66k%2BUwfi4Tz2ps2lpLbxwbGhpZ3wBHuo%2ByHwimcCfg%3D%3D)
    
2.  关于 GC 触发阈值
    
    > ![图片](https://segmentfault.com/img/remote/1460000042910692 "图片")  
    > 对应关系如下：
    > 
    > -   GC开始时内存使用量：GC trigger；
    > -   GC标记完成时内存使用量：Heap size at GC completion；
    > -   GC标记完成时的存活内存量：图中标记的Previous marked heap size为上一轮的GC标记完成时的存活内存量；
    > -   本轮GC标记完成时的预期内存使用量：Goal heap size；
    > 
    > 引用自: [https://www.jianshu.com/p/406...](https://link.segmentfault.com/?enc=pNNE%2B1EBdu%2Bi62S8Sdiz2Q%3D%3D.zcVp6Lrgn%2BLa36eUmbMODxDtKefoCGm0A2V2%2BNwmgYViGGcfvNA2SveZfjSd3GI8)
    

### 存在问题

1.  GC Marking - Concurrent 阶段，这个阶段有三个问题：  
    a. GC 协程和业务协程是并行运行的，大概会占用 25% 的CPU，使得程序的吞吐量下降；  
    b. 如果业务 goroutine 分配堆内存太快，导致 Mark 跟不上 Allocate 的速度，那么业务 goroutine 会被招募去做协助标记，暂停对业务逻辑的执行，这会影响到服务处理请求的耗时。  
    c. GO GC 在稳态场景下可以很好的工作，但是在瞬态场景下，如定时的缓存失效，定时的流量脉冲，GC 影响会急剧上升。一个典型例子：[IO 密集型服务 耗时优化](https://segmentfault.com/a/1190000041637173)  
    ![图片](https://segmentfault.com/img/bVcYRQq "图片")
2.  GC Mark Prepare、Mark Termination - STW 阶段，这两个阶段虽然按照官方说法时间会很短，但是在实际的线上服务中，有时会在 trace 图中观测到长达十几 ms 的停顿，原因可能为：OS 线程在做内存申请的时候触发内存整理被“卡住”，Go Runtime 无法抢占处于这种情况的 goroutine ，进而阻塞 STW 完成。（内存申请卡住原因：HugePage配置不合理）  
    ![image.png](https://segmentfault.com/img/bVc4dxT "image.png")
3.  过于关注 STW 的优化，带来服务吞吐量的下降（高峰期内存分配和 GC 时间的 CPU 占用超过 30% ）；  
    ![image.png](https://segmentfault.com/img/bVc4dxX "image.png")
    
    > **性能问题之 GC**  
    > 这里谈一下 GC 的问题，或者说内存管理的问题。  
    > 内存管理包括了内存分配和垃圾回收两个方面，对于 Go 来说，GC 是一个并发 - 标记 - 清除（CMS）算法收集器。但是需要注意一点，Go 在实现 GC 的过程当中，过多地把重心放在了暂停时间——也就是 Stop the World（STW）的时间方面，但是代价是牺牲了 GC 中的其他特性。  
    > 我们知道，GC 有很多需要关注的方面，比如吞吐量——GC 肯定会减慢程序，那么它对吞吐量有多大的影响；还有，在一段固定的 CPU 时间里可以回收多少垃圾；另外还有 Stop the World 的时间和频率；以及新申请内存的分配速度；还有在分配内存时，空间的浪费情况；以及在多核机器下，GC 能否充分利用多核等很多方面问题。非常遗憾的是，Golang 在设计和实现时，过度强调了暂停时间有限。但这带来了其他影响：比如在执行的过程当中，堆是不能压缩的，也就是说，对象也是不能移动的；还有它也是一个不分代的 GC。所以体现在性能上，就是内存分配和 GC 通常会占用比较多 CPU 资源。  
    > 我们有同事进行过一些统计，很多微服务在晚高峰期，内存分配和 GC 时间甚至会占用超过 30% 的 CPU 资源。占用这么高资源的原因大概有两点，一个是 Go 里面比较频繁地进行内存分配操作；另一个是 Go 在分配堆内存时，实现相对比较重，消耗了比较多 CPU 资源。比如它中间有 acquired M 和 GC 互相抢占的锁；它的代码路径也比较长；指令数也比较多；内存分配的局部性也不是特别好。
    > 
    > 引用自：[https://mp.weixin.qq.com/s/0X...](https://link.segmentfault.com/?enc=opQPoTnVYluDsX1V8qBhQw%3D%3D.yuFDHdWVsmZY73Yz%2B5RYFq5iDbXREuu2nfElhSJkvevm6ba7J7ODKx4%2BO3P83H0HfDh8Nmi5TQO8l11s55%2F3fA%3D%3D)
    
4.  由于 GC 不分代，每次 GC 都要扫描全量的存活对象，导致 GC 开销较高。（解决方式：[GO 的分代 GC](https://link.segmentfault.com/?enc=MbRNLrpgBZ5mrDulWxTSJQ%3D%3D.PBR5RF3nTGKmnDgjtlfrcGIRp41JRmLaBQNgRdjsSPeiezB3EXUXDFF33JuaizVe)）

## 优化

强烈建议阅读官方这篇 [Go 垃圾回收指南（翻译）](https://link.segmentfault.com/?enc=wSwZ8aIDirEopT%2FB49se1A%3D%3D.doZB%2BRqbJt8LfmImbHKIooMacd%2BV%2BGiJ7%2FS3kbTm3rovHCcXi0lXjYvTSRHokPvQ)

### 目标

-   降低 CPU 占用；
-   降低服务接口延时；

### 方向

1.  降低 GC 频率；
2.  减少堆上对象数量；

> **问题：为什么降低 GC 频率可以改善延迟**  
> 那么关键的一点是，降低 GC 频率也可能会改善延迟。 这不仅适用于通过修改调整参数来降低 GC 频率，例如增加 GOGC 和/或内存限制，还适用于[优化指南](https://link.segmentfault.com/?enc=J7mPRNCNox4GZU%2BlP25mGw%3D%3D.IrHSvbiOau%2FwuOiMOlPPc2Bru7sa2cY7ewCGMg3GsjQW9fQbJ7HO2rqFG3ScUufDxG6fCzwyFSy38uhjBidTZQ%3D%3D)中描述的优化。  
> 然而，理解延迟通常比理解吞吐量更复杂，因为它是程序即时执行的产物，而不仅仅是成本的聚合之物。 因此，延迟和GC频率之间的联系更加脆弱，可能不那么直接。 下面是一个可能导致延迟的来源列表，供那些倾向于深入研究的人使用。
> 
> -   当 GC 在标记和扫描阶段之间转换时，短暂的 stop-the-world 暂停
> -   调度延迟是因为 GC 在标记阶段占用了 25% 的 CPU 资源
> -   用户 goroutine 在高内存分配速率下的辅助标记
> -   当 GC 处于标记阶段时，指针写入需要额外的处理（write barrier）
> -   运行中的 goroutine 必须被暂停，以便扫描它们的根。
> 
> 引用自：[https://blog.leonard.wang/arc...](https://link.segmentfault.com/?enc=Wcx9wtgy5UggIYDzHgbkUw%3D%3D.ezGLhx0huMXFrzPCbQw5j7aCAiIsWFZ8StoTVByqz1cWK7hyCREssM4fgqygKfrK)

### 手段

#### sync.pool

**原理：** 使用 sync.pool() 缓存对象，减少堆上对象分配数；

> Pool's purpose is to cache allocated but unused items for later reuse, relieving pressure on the garbage collector.

**注意：** sync.pool 是全局对象，读写存在竞争问题，因此在这方面会消耗一定的 CPU，但之所以通常用它优化后 CPU 会有提升，是因为它的对象复用功能对 GC 和内存分配带来的优化，因此 sync.pool 的优化效果取决于锁竞争增加的 CPU 消耗与优化 GC 与内存分配减少的 CPU 消耗这两者的差值；

#### 设置 GOGC 参数（go 1.19 之前）

**原理：** GOGC 默认值是 100，也就是下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍，通过调大 GOGC 值（gcpercent）的方式，达到减少 GC 次数的目的；

> 公式：gc_trigger = heap_marked * (1+gcpercent/100)  
> heap_marked：上一个 GC 中被标记的(存活的)字节数；  
> gcpercent：通过 GOGC 来设置，默认是 100，也就是当前内存分配到达上次存活堆内存 2 倍时，触发 GC；
> 
> 在 go 1.19 及之后，这个公式变为了 heap_marked + (heap_marked + GC roots) * gcpercent / 100  
> GC roots：全局变量和goroutine的栈

**存在问题：** GOGC 参数不易控制，设置较小提升有限，设置较大容易有 OOM 风险，因为堆大小本身是在实时变化的，在任何流量下都设置一个固定值，是一件有风险的事情。

#### ballast 内存控制（go 1.19 之前）

**原理：** 仍然是从利用了下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍这一原理，初始化一个生命周期贯穿整个 Go 应用生命周期的超大 slice，用于内存占位，增大 heap_marked 值降低 GC 频率；实际操作有以下两种方式

> 公式：gc_trigger = heap_marked * (1+gcpercent/100)  
> gcpercent：通过 GOGC 来设置，默认是 100，也就是当前内存分配到达上次存活堆内存 2 倍时，触发 GC；  
> heap_marked：上一个 GC 中被标记的(存活的)字节数；

**相比于设置 GOGC 的优势**

-   安全性更高，OOM 风险小；
-   效果更好，可以从 pprof 图看出，后者的优化效果更大；

**负面考量**  
问：虽然通过大切片占位的方式可以有效降低 GC 频率，但是每次 GC 需要扫描和回收的对象数量变多了，是否会导致进行 GC 的那一段时间产生耗时毛刺？  
答：不会，GC 有两个阶段 mark 与 sweep，unused_objects 只与 sweep 阶段有关，但这个过程是非常快速的；mark 阶段是 GC 时间占用最主要的部分，但其只与当前的 inuse_objects 有关，与 unused_objects 无太大关系；因此，综上所述，降低频率确实会让每次 GC 时的 unused_objects 有所增长，但并不会对 GC 增加太多负担；

> 关于 ballast 内存控制更详细的内容：[https://blog.twitch.tv/en/201...](https://link.segmentfault.com/?enc=mO%2FSD4lRrAWjFH7maTVO1Q%3D%3D.Pjv%2FyZ0XymF8vexz8U1eFAKkxMfdxOZh2vS2X4FAd7bD%2FXm0apeWkCqLz%2FbA1GO1buqqeUY2TqZAIt%2FwmCeSGtLco9qgSHD%2FQ0R3lzjh8C1y5SXMpKh%2BZBf%2BTL0HqMc3lsPBgFaTT5Bbg%2FXTna9TEQ%3D%3D)

**以上三种优化操作的相关实践：** [Go 语言-计算密集型服务 性能优化](https://segmentfault.com/a/1190000041602269)

#### GCTuner（go 1.19 之前）

**原理：** [https://eng.uber.com/how-we-s...](https://link.segmentfault.com/?enc=yb6QNGFqwzu6b3iQ%2BFn2ug%3D%3D.9kxXm%2BAZCYgcFbJYNp9omZEb7lB2shmGWLPx1G4GAPPX5GwCv225ldLiaMhJdT%2FxWY1vjjSUCjfteordBP6%2By18mg6ZLcbiiKYp7ig22AvfKgJZfaJRUI8pG3GGAc9UV)  
**实现：** [https://github.com/bytedance/...](https://link.segmentfault.com/?enc=fwpypWTTD49U1f2nEZA9uw%3D%3D.9Ejpq9iC%2BJ6Kty%2FXTQjCOBj1Pj7iFPH7hixi%2Fq6J9Y8EZ8sHtddjkVHWsa%2BDW1vYiZSuQ4z5uE993o9hz%2FUwwg%3D%3D)  
**简述：** 同上文讲到的设置 GOGC 参数的思路相同，但增加了自动调整的设计，而非在程序初始设置一个固定值，可以有效避免高峰期的 OOM 问题。  
**优点：** 不需要修改 GO 源码，通用性较强；  
**缺点：** 对内存的控制不够精准。

#### GO SetMemoryLimit（go 1.19 及之后）

**原理：** [https://github.com/golang/pro...](https://link.segmentfault.com/?enc=FiioQBzsM9qWF3ZagVU5zg%3D%3D.ZsIr28T8pA%2FEADX0%2Bruk3wzRjO9D5NVlDsfeccfOdGH9%2FWYbRUckEPdTVLKtruoAduDmaO9A%2F2PY9V4RB46nnQjWAPTrvoxc8gDMg2vePf7W7Z56ngS3qmWrDVtrFTLT)  
**简述：**

> 通过对 Go 使用的内存总量设置软内存限制来调整 Go 垃圾收集器的行为。  
> 此选项有两种形式：runtime/debug调用的新函数SetMemoryLimit和GOMEMLIMIT环境变量。总之，运行时将尝试通过限制堆的大小并通过更积极地将内存返回给底层平台来维持此内存限制。这包括一种有助于减轻垃圾收集死循环的机制。最后，通过设置GOGC=off，Go 运行时将始终将堆增长到满内存限制。  
> 这个新选项使应用程序可以更好地控制其资源经济性。它使用户能够：
> 
> -   更好地利用他们已经拥有的内存；
> -   自信地降低他们的内存限制，知道 Go 会遵守他们；
> -   避免使用不受支持的 GC 调整方式；

**效果测试**：[https://colobu.com/2022/06/20...](https://link.segmentfault.com/?enc=unX%2Bal6ETplvQcfDw5ADAQ%3D%3D.7QZoOC4V%2BDHF9US%2BYF02zcBE%2BrJBBbaP%2FJxNyiCd6BtUDUCBKGx2ATZYrXptL37zN1udrFsSUhBOPNbK9Zp%2Btg%3D%3D)  
**其他：**

1.  与 GCTuner 的区别：  
    a. 两者都是通过调节 heapgoal 和 gctrigger 的值（GC 触发阈值），达到影响 GC 触发时机的目的；  
    b. GCTuner 对于 heapgoal 值的调整，依赖 SetFinalizer 的执行时机，在执行时通过设置 GOGC 参数间接调整的，在每个 GC 周期时最多调整一次；而 SetMemoryLimit 是一直在实时动态调整的，在每次检查是否需要触发GC的时候重新算的，不仅是每一轮 GC 完时决定，因此对于内存的控制更加精准。
2.  对内存的控制非常精准，可以关注到所有由 runtime 管理的内存，包括全局 Data 段、BSS 段所占用的内存；goroutine 栈的内存；被GC管理的内存；非 GC 管理的内存，如 trace、GC 管理所需的 mspan 等数据结构；缓存在 Go Heap 中没有被使用，也暂时未归还操作系统的内存；
3.  一般配合 GOGC = off 一起使用，可以达到最好的效果。

#### Bigcache

**原理：** [https://colobu.com/2019/11/18...](https://link.segmentfault.com/?enc=mAo4SBwF8euucYefyV8rSQ%3D%3D.xFgLuh%2FeWTvV%2Fq3%2BbjCs2E9zrSQs8PebH63G1MiKpOgDuwLnY2a%2F%2BxM4kCbO1vPmJ9Z3Z%2FbaBrzv8oUUHyaqYg%3D%3D)  
会在内存中分配大数组用以达到 0 GC 的目的，并使用 map[int]int，维护对对象的引用；

> 当 map 中的 key 和 value 都是基础类型时，GC 就不会扫到 map 里的 key 和 value

![image.png](https://segmentfault.com/img/bVc4dxO "image.png")  
**存在问题：** 由于大数组的存在，会起到同 ballast 内存控制手段的效果，一定程度上会影响到 GC 频率；  
**相关实践：** [IO 密集型服务 耗时优化](https://segmentfault.com/a/1190000041637173)

#### fastcache

与 bigcache 类似，但使用 syscall.mmap 申请堆外内存，避免了像 bigcache 影响 GC 的问题；

#### 堆外分配

绕过 Go runtime 直接分配内存，使 runtime 感知不到此块内存，从而不增加 GC 开销。

-   fastcache：直接调用 syscall.mmap 申请堆外内存使用；
-   offheap：使用 cgo 管理堆外内存；

问题：管理成本高，灵活性低；

#### GAB（Goroutine allocation buffer）

优化对象分配效率，并减少 GC 扫描的工作量。

> 经过调研发现，很多微服务进行内存分配时，分配的对象大部分都是比较小的对象。基于这个观测，我们设计了 GAB（Goroutine allocation buffer）机制，用来优化小对象内存分配。Go 的内存分配用的是 tcmalloc 算法，传统的 tcmalloc，会为每个分配请求执行一个比较完整的 malloc GC 方法，而我们的 Gab 为每个 Goroutine 预先分配一个比较大的 buffer，然后使用 bump-pointer 的方式，为适合放进 Gab 里的小对象来进行快速分配。我们算法和 tcmalloc 算法完全兼容，而且它的分配操作可以随意被 Stop the world 打断。虽然我们的 Gab 优化可能会造成一些空间浪费，但是在很多微服务上测试后，发现 CPU 性能大概节省了 5% 到 12%。
> 
> 引用自：[https://mp.weixin.qq.com/s/0X...](https://link.segmentfault.com/?enc=T2sjTCqTAs%2BOWvEn%2Fs%2FaSA%3D%3D.%2B2gyKhix96RxsKfWK%2Bhr61LxzcFtY99KBUh2JSElxQlTjKtF01nGxgkH5fTkRPwpLKNBwdG2QDjLKVvRKH8wYw%3D%3D)

#### Go1.20 arena

**官方文档：** [https://github.com/golang/go/...](https://link.segmentfault.com/?enc=Y91STNWVvCZAfrDJBJCUYw%3D%3D.bJJSp6kqOTM6YiFqidnb1DCSlyZ7JX30h76EsNSV7U5rFqvIOzDuhWTpUHB8YTGH)  
**简述：** 可以经由 runtime 申请内存，但由用户手动管理此块堆内存。因为是经由 runtime 申请的，可以被 runtime 感知到，因此可以纳入 GC 触发条件中的内存计算里，有效降低 OOM 风险。

## 其他

Java 中如何优化 GC：[ZGC在去哪儿机票运价系统实践](https://link.segmentfault.com/?enc=P4C90fgrYw0cpLeQ%2BleWPw%3D%3D.SzXUFWZBkA1sP5du%2FjtWL9HYORF2ECjR942uZvOvOJASpYSntwtP9gwNQ5doVnXY43jF%2F4Xi5yXUUVL%2B8liZLA%3D%3D)、[Java中9种常见的CMS GC问题分析与解决](https://link.segmentfault.com/?enc=bFa%2BYyRVI0qCjet9RG7wAQ%3D%3D.Oktf02ULs1rWGfxHD4vd8ftLOYGuQYdFStFnJgOtlR7JX%2BAGN7pcvTycb9Ft%2BHw1UP4Q1m%2BfcsiotXm861dM1Q%3D%3D)


# [Go 语言-计算密集型服务 性能优化](https://segmentfault.com/a/1190000041602269)

![](https://sponsor.segmentfault.com/lg.php?bannerid=0&campaignid=0&zoneid=25&loc=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000041602269&referer=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000042910688&cb=a4c87d303a)

## 背景

### 项目背景

![image.png](https://segmentfault.com/img/bVcYIDe "image.png")

                            worker 服务数据链路图

worker 服务消费上游数据（工作日高峰期产出速度达近 **200 MB/s**，节假日高峰期可达 **300MB/s** 以上），进行中间处理后，写入多个下游。在实践中结合业务场景，基于**快慢隔离**的思想，以三个不同的 consumer group 消费同一 Topic，隔离三种数据处理链路。

### 面对问题

-   worker 服务在高峰期时 CPU Idle 会降至 60%，因其属于数据处理类计算密集型服务，CPU Idle 过低会使服务吞吐降低，在数据处理上产生较大延时，且受限于 Kafka 分区数，无法进行横向扩容；
-   对上游数据的采样率达 **30%**，业务方对数据的完整性有较大诉求，但系统 CPU 存在瓶颈，无法满足；

## 性能优化

针对以上问题，开始着手对服务 CPU Idle 进行优化；  
抓取服务 pprof profile 图如下：  
go tool pprof -http=:6061 http://「ip:port」/debug/pprof/profile  
![image.png](https://segmentfault.com/img/bVcYIVA "image.png")

                              优化前的 pprof profile 图

### 服务与存储之间置换压力

#### 背景

worker 服务消费到上游数据后，会先写全部写入 Apollo 的数据库，之后每分钟定时捞取处理，但消息体大小 P99 分位达近 **1.5M**，对于 Apollo 有较严重的大 Key 问题，再结合 RocksDB 特有的写放大问题，会进一步加剧存储压力。在这个背景下，我们采用 zlib 压缩算法([压缩算法测试报告](https://link.segmentfault.com/?enc=Gj19QAYOA4Mc2Ldo4Akjkg%3D%3D.ZkwECQYaMFw%2BPdzSXnWObaD4NiFGWQ2ctSkxTEOmntw2likQq9Iy2J20R%2BvDk8xm0qGI7tgPgUPzSlCf99c0SebdhGGKN6t3U0eO5humWUw%3D))，对消息体先进行压缩后再写入 Apollo，减缓读写大 Key 对 Apollo 的压力。

#### 优化

在 CPU 的优化过程中，我们发现服务在压缩操作上占用了较多的 CPU，于是对压缩等级进行调整，以减小压缩率、增大下游存储压力为代价，减少压缩操作对服务 CPU 的占用，提升服务 CPU 。  
这一操作本质上是在服务与存储之间进行压力置换，是一种**空间换时间**的做法。

#### 关于压缩等级

这里需要特别注意的是，在压缩等级的设置上可能存在较为严重的**边际效用递减**问题。  
在进行基准测试时发现，将压缩等级由 BestCompression 调整为 DefaultCompression 后，压缩率只有近 **1‱** 的下降，但压缩方面的 CPU 占用却相对提高近 **50%**。  
此结论不能适用于所有场景，需从实际情况出发，但在使用时应注意这个问题，选择相对最优的压缩方式。  
![image.png](https://segmentfault.com/img/bVcYIDm "image.png")

                                zlib 可设置的压缩等级

### 使用更高效的序列化库

#### 背景

worker 服务在设计之初基于快慢隔离的思想，使用三个不同的 consumer group 进行分开消费，导致对同一份数据会重复消费三次，而上游产出的数据是在 PB 序列化之后写入 Kafka，消费侧亦需要进行 PB 反序列化方能使用，因此导致了 PB 反序列化操作在 CPU 上的较大开销。

#### 优化

经过探讨和调研后发现，gogo/protobuf 三方库相较于原生的 golang/protobuf 库性能更好，在 CPU 上占用更低，速度更快，因此采用 gogo/protobuf 库替换掉原生的 golang/protobuf 库。

#### gogo/protobuf 为什么快

-   通过对每一个字段都生成代码的方式，**取消了对反射的使用**；
-   采用预计算方式，在序列化时能够减少内存分配次数，进而**减少了内存分配**带来的系统调用、锁和 GC 等代价。

> 用过去或未来换现在的时间：页面静态化、池化技术、预编译、代码生成等都是提前做一些事情，用过去的时间，来降低用户在线服务的响应时间；另外对于一些在线服务非必须的计算、存储的耗时操作，也可以异步化延后进行处理，这就是用未来的时间换现在的时间。  
> 出处：[https://mp.weixin.qq.com/s/S8KVnG0NZDrylenIwSCq8g](https://link.segmentfault.com/?enc=PIxofDjAMo%2BpjxYcZfVkrg%3D%3D.NiG2FHPx8JHzwOn%2BWTxYxWibce7E9ckd15anFD7i77Zzf1aRcRhSSekfqlSA%2BZAHKc6GMzqQcobt%2FGeWsi071w%3D%3D)

#### 关于序列化库

这里只列举的了 PB 序列化库的优化 Case，但在 JSON 序列化方面也存在一样的优化手段，如 json-iterator、sonic、gjson 等等，我们在 Feature 服务中先后采用了 json-iterator 与 gjson 库替换原有的标准库 JSON 序列化方式，均取得了显著效果。  
JSON 库调研报告：[https://segmentfault.com/a/1190000041591284](https://segmentfault.com/a/1190000041591284)  
![image.png](https://segmentfault.com/img/bVcYIEJ "image.png")

                        调整压缩等级与更换 PB 序列化库之后

### 数据攒批 减少调用

#### 背景

在观察 pprof 图后发现写 hbase 占用了近 50% 的相对 CPU，经过进一步分析后，发现每次在序列化一个字段时 Thrift 都会调用一次 socket->syscall，带来频繁的上下文切换开销。

#### 优化

阅读代码后发现， 原代码中使用了 Thrift 的 TTransport 实现，其功能是包装 TSocket，裸调 Syscall，每次 Write 时都会调用 socket 写入进而调用 Syscall。这与通常我们的编码习惯不符，认为应该有一个 buffer 充当中间层进行数据攒批，当 buffer 写完或者写满后再向下层写入。于是进一步阅读 Thrift 源码，发现其中有多种 Transport 实现，而 TTBufferedTransport 是符合我们编码习惯的。  
![image.png](https://segmentfault.com/img/bVcYIFa "image.png")

        对 Thrift 调用进行优化，使用带 buffer 的 transport，大大减少对 Syscall的调用

![image.png](https://segmentfault.com/img/bVcYIFR "image.png")

                更换 transport 之后，对 HBase 的调用消耗只剩最右侧的一条了

![image.png](https://segmentfault.com/img/bVcYIFm "image.png")

                            对 HBase 的访问耗时大幅下降

#### Thrift Client 部分源码分析

Transport 使用了装饰器模式

Transport 实现

作用

TTransport

包装 TSocket，裸调 Syscall，每次 Write 都会调用 syscall；

TTBufferedTransport

需要提前声明 buffer 的大小，在调用 Socket 之前加了一层 buffer，写满或者写完之后再调用 Syscall；

TFramedTransport

与 TTBufferedTransport 类似，但只会在全部写入 buffer 后，再调用 Syscall。数据格式为：size+content，客户端与服务端必须都使用该实现，否则会因为格式不兼容报错；

streamTransport

传入自己实现的 IO 接口；

TMemoryBufferTransport

纯内存交换，不与网络交互；

Protocol

Protocol 实现

作用

TBinaryProtocol

直接的二进制格式；

TCompactProtocol

紧凑型、高效和压缩的二进制格式；

TJSONProtocol

JSON 格式；

TSimpleJSONProtocol

SimpleJSON 产生的输出适用于 AJAX 或脚本语言，它不保留Thrift的字段标签，不能被 Thrift 读回，它不应该与全功能的 TJSONProtocol 相混淆；[https://cwiki.apache.org/confluence/display/THRIFT/ThriftUsag...](https://link.segmentfault.com/?enc=L6v5R8wJTlh3UOxXpo%2F0QQ%3D%3D.Xh8J%2F5CWQEI2bel8BNLZm%2BpajQrDhZ%2FKopp%2BV9l8Q6g%2BkP6q%2F3Q1Zx5z3FUy212SGaT0qWj0BmxsavsEgVVCHpa%2BiR%2Fz0I7BlACA5jsvqIQ%3D)

#### 关于数据攒批

数据攒批：将数据先写入用户态内存中，而后统一调用 syscall 进行写入，常用在数据落盘、网络传输中，可降低系统调用次数、利用磁盘顺序写特性等，是一种空间换时间的做法。有时也会牺牲一定的数据实时性，如 kafka producer 侧。  
相似优化可见：[https://mp.weixin.qq.com/s/ntNGz6mjlWE7gb_ZBc5YeA](https://link.segmentfault.com/?enc=FXVBYdnjRsqwDcg2cbKQdA%3D%3D.bcWwBBzAjvI%2B7fEegp%2BJX8yNAvwL9lVYAkGZq6cnLT9yuOaogXYoWol0kZbLiqa2o%2Be8ayhCN98U4%2FnHNVedeQ%3D%3D)

### 语法调整

除在对库的使用上进行优化外，在 GO 语言本身的使用上也存在一些优化方式；

-   slice、map 预初始化，减少频繁扩容导致的内存拷贝与分配开销；
-   字符串连接使用 strings.builder(预初始化) 代替 fmt.Sprintf()；

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

-   buffer 修改返回 string([]byte) 操作为 []byte，减少内存 []byte -> string 的内存拷贝开销；

![image.png](https://segmentfault.com/img/bVcYIGu "image.png")

-   string <-> []byte 的另一种优化，需确保 []byte 内容后续不会被修改，否则会发生 panic；

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

#### 关于语法调整

更多语法调整，见以下文章

-   [https://www.bacancytechnology.com/blog/golang-performance](https://link.segmentfault.com/?enc=flmVqaJPZigoFJdqnjK0TA%3D%3D.6vKRKxt%2BJ5YOtuCztbjXqHPM6i6Xqz4rZfelWdHHaODQAhu5ED9gai28B4BwX6t%2Fhrjtu0J5XuXDlHea%2BBF9MA%3D%3D)
-   [https://mp.weixin.qq.com/s/Lv2XTD-SPnxT2vnPNeREbg](https://link.segmentfault.com/?enc=xgO2ccpIzbuu5ia3PWuB%2Bw%3D%3D.hBGQLNZ6PdXoSH9ytRlO5iju34Sci0yAUpCLU6Yh9Dj5eAH8Us6y2%2F3MgNrkfDfulLPGwlChtEoIqguB%2FpnhPw%3D%3D)

### GC 调优

#### 背景

在上次优化完成之后，系统已经基本稳定，CPU Idle 高峰期也可以维持在 80% 左右，但后续因业务诉求对上游数据采样率调整至 100%，CPU.Idle 高峰期指标再次下降至近 70%，且由于定时任务的问题，存在 CPU.Idle 掉 0 风险；

#### 优化

经过对 pprof 的再次分析，发现 runtime.gcMarkWorker 占用不合常理，达到近 30%，于是开始着手对 GC 进行优化；  
![image.png](https://segmentfault.com/img/bVcYIMx "image.png")

                                GC 优化前 pprof 图

##### 方法一：使用 sync.pool()

通常来说使用 sync.pool() 缓存对象，减少对象分配数，是优化 GC 的最佳方式，因此我们在项目中使用其对 bytes.buffer 对象进行缓存复用，意图减少 GC 开销，但实际上线后 **CPU Idle 却略微下降，且 GC 问题并无缓解**。原因有二：

1.  sync.pool 是全局对象，读写存在竞争问题，因此在这方面会消耗一定的 CPU，但之所以通常用它优化后 CPU 会有提升，是因为它的对象复用功能对 GC 带来的优化，因此 sync.pool 的优化效果取决于锁竞争增加的 CPU 消耗与优化 GC 减少的 CPU 消耗这两者的差值；
2.  GC 压力的大小通常取决于 inuse_objects，与 inuse_heap 无关，也就是说与正在使用的对象数有关，与正在使用的堆大小无关；

本次优化时选择对 bytes.buffer 进行复用，是想做到减少堆大小的分配，出发点错了，对 GC 问题的理解有误，对 GC 的优化因从 pprof heap 图 inuse_objects 与 alloc_objects 两个指标出发。  
![image.png](https://segmentfault.com/img/bVcYIMR "image.png")

                        甚至没有依赖经验， 只是单纯的想当然了🤦‍♂️

##### 方法二：设置 GOGC

**原理**：GOGC 默认值是 100，也就是下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍，通过调大 GOGC 值（gcpercent）的方式，达到减少 GC 次数的目的；

> 公式：gc_trigger = heap_marked * (1+gcpercent/100)  
> gcpercent：通过 GOGC 来设置，默认是 100，也就是当前内存分配到达上次存活堆内存 2 倍时，触发 GC；  
> heap_marked：上一个 GC 中被标记的(存活的)字节数；

**问题**：GOGC 参数不易控制，设置较小提升有限，设置较大容易有 OOM 风险，因为堆大小本身是在实时变化的，在任何流量下都设置一个固定值，是一件有风险的事情。这个问题目前已经有[解决方案](https://link.segmentfault.com/?enc=JWb1dlMrYk6EpfllfSR9TQ%3D%3D.Rd6%2Bqc3AF6%2BsqpAwzBK3lwbzg6hZO5BNctyRS%2BjcKxLrxMOi4Lz5wzswFOAk646r)，Uber 发表的文章中提到了一种自动调整 GOGC 参数的方案，用于在这种方式下优化 GO 的 GC CPU 占用，不过业界还没有开源相关实现。  
![image.png](https://segmentfault.com/img/bVcYIM6 "image.png")  
![image.png](https://segmentfault.com/img/bVcYINA "image.png")

                        设置 GOGC 至 1000% 后，GC 占用大幅缩小

![image.png](https://segmentfault.com/img/bVcYIND "image.png")

                                内存利用率接近 100%

##### 方法三：GO ballast 内存控制

> ballast 压舱物--航海。为提供所需的吃水和稳定性而临时或永久携带在船上的重型材料。  
> 来源：Dictionary.com

**原理**：仍然是从利用了下次 GC 触发的 heap 的大小是这次 GC 之后的 heap 的一倍这一原理，初始化一个生命周期贯穿整个 Go 应用生命周期的超大 slice，用于内存占位，增大 heap_marked 值降低 GC 频率；实际操作有以下两种方式

> 公式：gc_trigger = heap_marked * (1+gcpercent/100)  
> gcpercent：通过 GOGC 来设置，默认是 100，也就是当前内存分配到达上次存活堆内存 2 倍时，触发 GC；  
> heap_marked：上一个 GC 中被标记的(存活的)字节数；

![image.png](https://segmentfault.com/img/bVcYINO "image.png")

                                    方式一

![image.png](https://segmentfault.com/img/bVcYINP "image.png")

                                    方式二

两种方式都可以达到同样的效果，但是方式一会实际占用物理内存，在可观测性上会更舒服一点，方式二并不会实际占用物理内存。

> 原因：  
> Memory in ‘nix (and even Windows) systems is virtually addressed and mapped through page tables by the OS. When the above code runs, the array the ballast slice points to will be allocated in the program’s virtual address space. Only if we attempt to read or write to the slice, will the page fault occur that causes the physical RAM backing the virtual addresses to be allocated.  
> 引用自：[https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-...](https://link.segmentfault.com/?enc=KxLj9SLrk6zCITN0AYMZGQ%3D%3D.qoBPaJvEdze2bC9dprSdc4xsr0l7EAamCr31oHtXqfB0B1LlBkzesMY2T8PvxExQccVXk%2BKIjiM%2BPUiDBdX8OCJNv4qFD6dCjl8oBLq4hVTOytC0Jr5SKzbhcc6h8xfkZn122YXATXzCc5m5iLp%2FcA%3D%3D)

![image.png](https://segmentfault.com/img/bVcYIOd "image.png")

                    优化后 GC 频率由每秒 5 次降低到了每秒 0.1 次

![image.png](https://segmentfault.com/img/bVcYIOq "image.png")

                    使用 ballast 内存控制后，GC 占用缩小至红框大小

![image.png](https://segmentfault.com/img/bVcYIOj "image.png")

                    使用方式一后，内存始终稳定在 25% -30%，即 3G 大小

**相比于设置 GOGC 的优势**

-   安全性更高，OOM 风险小；
-   效果更好，可以从 pprof 图看出，后者的优化效果更大；

**负面考量**  
问：虽然通过大切片占位的方式可以有效降低 GC 频率，但是每次 GC 需要扫描和回收的对象数量变多了，是否会导致进行 GC 的那一段时间产生耗时毛刺？  
答：不会，GC 有两个阶段 mark 与 sweep，unused_objects 只与 sweep 阶段有关，但这个过程是非常快速的；mark 阶段是 GC 时间占用最主要的部分，但其只与当前的 inuse_objects 有关，与 unused_objects 无太大关系；因此，综上所述，降低频率确实会让每次 GC 时的 unused_objects 有所增长，但并不会对 GC 增加太多负担；

> 关于 ballast 内存控制更详细的内容请看：[https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-...](https://link.segmentfault.com/?enc=0uKG%2Fl%2FCaVkeXKktvc1ATw%3D%3D.ru%2BK9VZNuHpz7oEw0pvhp0M4aNNFHt9N5%2FpHSge0OZKtvH08LGpJU6v5%2BrWA%2BLPpNnVeLcr9Qr5la22c02pJ8OuHPHxZ4fznS4B9vdYjCcP6c4R7tWaR%2FtKCVseYLZCoz4%2FRuT6BD26Tll6%2FjPj9xQ%3D%3D)

#### 关于 GC 调优

-   GC 优化手段的优先级：设置 GOGC、GO ballast 内存控制等操作是一种治标不治本略显 trick 的方式，在做 GC 优化时还应先从对象复用、减少对象分配角度着手，在确无优化空间或优化成本较大时，再选择此种方式；
-   设置 GOGC、GO ballast 内存控制等操作本质上也是一种空间换时间的做法，在内存与 CPU 之间进行压力置换；
-   在 GC 调优方面，还有很多其他优化方式，如 bigcache 在堆内定义大数组切片自行管理、fastcache 直接调用 syscall.mmap 申请堆外内存使用、offheap 使用 cgo 管理堆外内存等等。

## 优化效果

![image.png](https://segmentfault.com/img/bVcYIHy "image.png")

                    黄色线：调整压缩等级与更换 PB 序列化库；
                    绿色线：Thrift 序列化更换带 buffer 的 transport；

![image.png](https://segmentfault.com/img/bVcYIH3 "image.png")

            蓝色曲线抖动是因为上游业务放量，后又做垂直伸缩将 CPU 由 8 核提至 16 核
                                    GC 优化（红框部分）

-   先后将 CPU 提升 **25%、10%**（假设不做伸缩）；
-   支持上游数据 **100%** 放量；
-   通过对 CPU 瓶颈的解决，顺利合并服务，下掉 **70 台**容器。

## 总结

**经验分享**

-   做性能优化经验很重要，其次在优化之前掌握一部分前置知识更好；
-   平时多看一些资料学习，有优化机会就抓住实践，避免书到用时方恨少；
-   仔细观察 pprof 图，分析大块部分；
-   观察问题点的 api 使用，可能具有更高效的使用方式
-   记录优化过程和优化效果，以后分享、吹逼用的上；
-   最好可以构建稳定的基准环境，验证效果；
-   空间换时间是万能钥匙，工程问题不要只 case by case 的看，很多解决方案都是同一种思想的不同落地，尝试去总结和掌握这种思想，最后达到迁移复用的效果；
-   多和大佬讨论，非常重要，以上多项优化都出自与大佬（特别鸣谢 @李小宇@曹春晖）讨论后的实践；

# Reference
https://segmentfault.com/a/1190000041602269