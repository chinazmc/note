---
sr-due: 2022-10-07
sr-interval: 2
sr-ease: 230
---

 #golang/内存 #note 


Go语言内置运行时（就是runtime），抛弃了传统的内存分配方式，改为自主管理。这样可以自主地实现更好的内存使用模式，比如内存池、预分配等等。这样，不会每次内存分配都需要进行系统调用。

Golang运行时的内存分配算法主要源自 Google 为 C 语言开发的`TCMalloc算法`，全称`Thread-Caching Malloc`。核心思想就是把内存分为多级管理，从而降低锁的粒度。它将可用的堆内存采用二级分配的方式进行管理：每个线程都会自行维护一个独立的内存池，进行内存分配时优先从该内存池中分配，当内存池不足时才会向全局内存池申请，以避免不同线程对全局内存池的频繁竞争

# 基础概念

Go在程序启动的时候，会先向操作系统申请一块内存（注意这时还只是一段虚拟的地址空间，并不会真正地分配内存），切成小块后自己进行管理。

申请到的内存块被分配了三个区域，在X64上分别是512MB，16GB，512GB大小。

![堆区总览](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c860e5fa3c~tplv-t2oaga2asx-watermark.awebp)

`arena区域`就是我们所谓的堆区，Go动态分配的内存都是在这个区域，它把内存分割成`8KB`大小的页，一些页组合起来称为`mspan`。

`bitmap区域`标识`arena`区域哪些地址保存了对象，并且用`4bit`标志位表示对象是否包含指针、`GC`标记信息。`bitmap`中一个`byte`大小的内存对应`arena`区域中4个指针大小（指针大小为 8B ）的内存，所以`bitmap`区域的大小是`512GB/(4*8B)=16GB`。

![bitmap arena](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c75a5cd188~tplv-t2oaga2asx-watermark.awebp)

![bitmap arena](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c75a442c75~tplv-t2oaga2asx-watermark.awebp)

从上图其实还可以看到bitmap的高地址部分指向arena区域的低地址部分，也就是说bitmap的地址是由高地址向低地址增长的。

`spans区域`存放`mspan`（也就是一些`arena`分割的页组合起来的内存管理基本单元，后文会再讲）的指针，每个指针对应一页，所以`spans`区域的大小就是`512GB/8KB*8B=512MB`。除以8KB是计算`arena`区域的页数，而最后乘以8是计算`spans`区域所有指针的大小。创建`mspan`的时候，按页填充对应的`spans`区域，在回收`object`时，根据地址很容易就能找到它所属的`mspan`。

# 内存管理单元
## 内存管理概况
> `Golang`的内存管理包含内存管理单元、线程缓存、中心缓存和页堆四个重要的组件，分别对应`runtine.mspan`、`runtime.mcache`、`runtime.mcentral`和`runtime.mheap`。

每一个`Go`程序在启动时都会向操作系统申请一块内存（仅仅是虚拟的地址空间，并不会真正分配内存），在`X64`上申请的内存会被分成`512M`、`16G`和`512G`的三块空间，分别对应`spans`、`bitmap`和`arena`。


![](https://pic4.zhimg.com/80/v2-1aa731ddd77b6cad73c8f68f864ea5ef_1440w.jpg)


![](https://pic1.zhimg.com/80/v2-7c7d27f2fc11b559cd527132546a7988_720w.jpg)

-   `arena`：堆区，运行时该区域每`8KB`会被划分成一个页，存储了所有在堆上初始化的对象
-   `bitmap`：标识`arena`中哪些地址保存了对象，`bitmap`中一个字节的内存对应`arena`区域中`4`个指针大小的内存，并标记了是否包含指针和是否扫描的信息（一个指针大小为`8B`，因此`bitmap`的大小为`512GB/(4*8)=16GB`）
1kb=1024bytes=8129bit

![](https://pic1.zhimg.com/80/v2-0ce962eb75617bac8fa0ad844b57c7ac_720w.jpg)

-   `spans`：存放`mspan`的指针，其中每个`mspan`会包含多个页，`spans`中一个指针（`8B`）表示`arena`中某一个`page`（`8KB`），因此`spans`的大小为`512GB/(1024)=512MB`

![](https://pic2.zhimg.com/80/v2-9aafbba848e15cb8f5ac7ec1d12bd37d_720w.jpg)

## mspan

![](https://pic2.zhimg.com/80/v2-bfb6b3ac165781032490142bf047c835_720w.jpg)

`mspan`：Go中内存管理的基本单元，是由一片连续的`8KB`的页组成的大块内存。注意，这里的页和操作系统本身的页并不是一回事，它一般是操作系统页大小的几倍。一句话概括：`mspan`是一个包含起始地址、`mspan`规格、页的数量等内容的双端链表。

每个`mspan`按照它自身的属性`Size Class`的大小分割成若干个`object`，每个`object`可存储一个对象。并且会使用一个位图来标记其尚未使用的`object`。属性`Size Class`决定`object`大小，而`mspan`只会分配给和`object`尺寸大小接近的对象，当然，对象的大小要小于`object`大小。还有一个概念：`Span Class`，它和`Size Class`的含义差不多，

```
Size_Class = Span_Class / 2
```

这是因为其实每个 `Span Class`有两个`mspan`，也就是有两个`Size Class`。其中一个分配给含有指针的对象，另一个分配给不含有指针的对象。这会给垃圾回收机制带来利好，之后的文章再谈。

如下图，`mspan`由一组连续的页组成，按照一定大小划分成`object`。

![page mspan](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c75a5fa8ae~tplv-t2oaga2asx-watermark.awebp)

Go1.9.2里`mspan`的`Size Class`共有67种，每种`mspan`分割的object大小是8\*2^n的倍数，这个是写死在代码里的：

```go
// path: /usr/local/go/src/runtime/sizeclasses.go

const _NumSizeClasses = 67

var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536,1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}

```

根据`mspan`的`Size Class`可以得到它划分的`object`大小。 比如`Size Class`等于3，`object`大小就是32B。 32B大小的object可以存储对象大小范围在17B~32B的对象。而对于微小对象（小于16B），分配器会将其进行合并，将几个对象分配到同一个`object`中。

数组里最大的数是32768，也就是32KB，超过此大小就是大对象了，它会被特别对待，这个稍后会再介绍。顺便提一句，类型`Size Class`为0表示大对象，它实际上直接由堆内存分配，而小对象都要通过`mspan`来分配。

对于mspan来说，它的`Size Class`会决定它所能分到的页数，这也是写死在代码里的：

```go
// path: /usr/local/go/src/runtime/sizeclasses.go

const _NumSizeClasses = 67

var class_to_allocnpages = [_NumSizeClasses]uint8{0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 2, 1, 2, 1, 3, 2, 3, 1, 3, 2, 3, 4, 5, 6, 1, 7, 6, 5, 4, 3, 5, 7, 2, 9, 7, 5, 8, 3, 10, 7, 4}
```

比如当我们要申请一个`object`大小为`32B`的`mspan`的时候，在class_to_size里对应的索引是3，而索引3在`class_to_allocnpages`数组里对应的页数就是1。

`mspan`结构体定义：

```go
// go1.13.5
type mspan struct {
    next *mspan     // 链表下一个span地址
    prev *mspan     // 链表前一个span地址
    list *mSpanList // 链表地址, 用于DEBUG
​
    startAddr uintptr // 该span在arena区域的起始地址
    npages    uintptr // 该span占用arena区域page的数量
​
    manualFreeList gclinkptr // 空闲对象列表
​
    // freeindex是0~nelems的位置索引, 标记当前span中下一个空对象索引
    freeindex uintptr 
    // 当前span中管理的对象数
    nelems uintptr 
​
    allocCache uint64  // 从freeindex开始的位标记 用于快速查找内存中未被使用的内存
    allocBits  *gcBits // 该mspan中对象的位图
    gcmarkBits *gcBits // 该mspan中标记的位图,用于垃圾回收
    
    sweepgen    uint32
    divMul      uint16        // for divide by elemsize - divMagic.mul
    baseMask    uint16        // if non-0, elemsize is a power of 2, & this will get object allocation base
    allocCount  uint16        // number of allocated objects
    spanclass   spanClass     // size class and noscan (uint8)
    state       mSpanStateBox // mSpanInUse etc; accessed atomically (get/set methods)
    needzero    uint8         // needs to be zeroed before allocation
    divShift    uint8         // for divide by elemsize - divMagic.shift
    divShift2   uint8         // for divide by elemsize - divMagic.shift2
    elemsize    uintptr       // computed from sizeclass or from npages 用于计算mspan管理了多少内存
    limit       uintptr       // end of data in span span的结束地址值
    speciallock mutex         // guards specials list
    specials    *special      // linked list of special records sorted by offset.
```
`runtime.mspan`是内存管理器里面的最小粒度单元，所有的对象都是被管理在mspan下面。
mspan是一个链表，有上下指针；
- npages代表mspan管理的堆页的数量；
- freeindex是空闲对象的索引；
- nelems代表这个mspan中可以存放多少对象，等于`(npages * pageSize)/elemsize`；
- allocCache用于快速的查找未被使用的内存地址；
- elemsize表示一个对象会占用多个个bytes，等于`class_to_size[sizeclass]`，需要注意的是sizeclass每次获取的时候会sizeclass方法，将`sizeclass>>1`；
- limit表示span结束的地址值，等于`startAddr+ npages*pageSize`；
-   `startAddr`和`npages`：确定该`mspan`管理的多个页在`arena`堆中的内存位置
-   `allocCache`和`freeindex`：当用户程序或者线程向`mspan`申请内存时，根据这两个字段在管理的内存中快速查找可以分配的空间，如果查询不到可以分配的空间，`mcache`会调用`runtine.mcache.refill`更新`mspan`以满足为更多对象分配内存的需求
-   `spanclass`：决定`mspan`中存储对象`object`的大小和个数，当前`golang`从`8B`到`32KB`共分`66`个类，下图是每一类大小能存储的`object`大小和数量，以及因为内存按页管理造成的`tail waster`和最大内存浪费率。

 **allocBits,allocCache,freeIndex** allowBits 是一个 bit 数组，大小为 nelems，bit=1 表示对应 index 已经被分配，allocCache 是 allowBits 某一段的映射，其是为了加速在 allocBits 中寻找空闲对象存在的，在 allocCache 中,bit=0 表示对应 index 已经被分配(和 allocBits 相反)，freeIndex 是一个和 allocCache 挂钩的变量，其存储了 allocCache 最后一 bit 在 allocBits 的偏移量，对于 mspan 管理的 nelems 个对象，[0,freeIndex)已经全部被分配出去，[freeIndex,nelems)可能被分配，也可能是空闲的，每次 gc 后 freeIndex 置为 0,相关图如下。
![](https://pic4.zhimg.com/80/v2-6a15c140b0593f0b03383f0604b45e37_720w.webp)


我们将`mspan`放到更大的视角来看：

![mspan更大视角](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c75b17cd6f~tplv-t2oaga2asx-watermark.awebp)

上图可以看到有两个`S`指向了同一个`mspan`，因为这两个`S`指向的`P`是同属一个`mspan`的。所以，通过`arena`上的地址可以快速找到指向它的`S`，通过`S`就能找到`mspan`，回忆一下前面我们说的`mspan`区域的每个指针对应一页。

假设最左边第一个`mspan`的`Size Class`等于10，根据前面的`class_to_size`数组，得出这个`msapn`分割的`object`大小是144B，算出可分配的对象个数是`8KB/144B=56.89`个，取整56个，所以会有一些内存浪费掉了，Go的源码里有所有`Size Class`的`mspan`浪费的内存的大小；再根据`class_to_allocnpages`数组，得到这个`mspan`只由1个`page`组成；假设这个`mspan`是分配给无指针对象的，那么`spanClass`等于20。

`startAddr`直接指向`arena`区域的某个位置，表示这个`mspan`的起始地址，`allocBits`指向一个位图，每位代表一个块是否被分配了对象；`allocCount`则表示总共已分配的对象个数。

这样，左起第一个`mspan`的各个字段参数就如下图所示：

![左起第一个mspan具体值](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c8168adc34~tplv-t2oaga2asx-watermark.awebp)


> 假设一个`mspan`的`spanclass`为`4`，那么它能存储的对象大小为`33~48B`，能存储的对象个数为`170`个，并且会在一页`8KB`的尾部浪费`32B`的内存：  
> ![[Pasted image 20220302182910.png]]
> 假设`mspan`中存储的对象大小均为`33B`，那么最大的内存浪费率 `max waste`为：  
> ![[Pasted image 20220302182952.png]]

![](https://pic1.zhimg.com/80/v2-26d653537e5fa9f285ff67374d37a724_720w.jpg)

`golang`在分配内存时会根据对象的大小来选择不同`spanclass`的`span`，比如`17~32Byte`的对象都会使用`spanclass`为`3`的`span`。超过`32KB`的对象被称为“大对象”，`golang`在分配这类“大对象”时会直接从`heap`中分配一个`spanclass`为`0`的`span`，它只包含一个大对象。

# **Golang内存管理前身**

### **1. 简介**

`tcmalloc`全称为`thread-caching malloc`，是`google`推出的一种内存管理器。按照对象所占内存空间的大小，`tcmalloc`将对象划分为三类：
![[Pasted image 20220302181249.png]]
相关的重要概念：
-   `Page`：`tcmalloc`将虚拟地址空间划分为多个大小相同的页`Page`（大小为`8KB`）
-   `Span`：单个`Span`可能包含一个或多个`Page`，是`tcmalloc`向操作系统申请内存的基本单位
-   `Size Class`：对于`256KB`以内的小对象，`tcmalloc`按照大小划分了不同的`Size Class`，比如`8`字节、`16`字节和`32`字节等，以此类推。应用程序申请小对象需要的内存时，`tcmalloc`会将申请的内存向上取整到某个`Size Class`的大小
-   `ThreadCache`：每个线程自己维护的缓存，里面对于每个`Size Class`都有一个单独的`FreeList`，缓存了`n`个还未被应用程序使用的空闲对象，由于不同线程的`ThreadCache`是相互独立的，因此小对象从`ThreadCache`中的`FreeList`中存储或者回收时是不需要加锁的，提升了执行效率
-   `CentralCache`：同样对于每个`Size Class`都维护了一个`Central Free List`来缓存空闲对象，作为各个`ThreadCache`获取空闲对象，每个线程从`CentralCache`中取用或者回收对象是需要加锁的，为了平摊加锁解锁的时间开销，一般一次会取用或者回收多个空闲对象
-   `PageHeap`：当`Centralache`中的空闲对象不够用时会向`PageHeap`申请一块内存然后再拆分成各个`Size Class`添加到对应的`CentralFreeist`中。`PageHeap`对不同内存块`Span`的大小采用了不同的缓存策略：`128 Page`以内的`Span`每个大小都用一个链表来缓存，超过`128 Page`的`Span`存储在一个有序`set`中

![](https://pic3.zhimg.com/80/v2-f8c3e7291ab73997ac386ef6fb1b2ac6_720w.jpg)

### **2. 小对象分配**

对于小于等于`256KB`的小对象，分配流程如下：

-   将对象大小向上取整到对应的`Size Class`
-   如果`FreeList`非空则直接移除`FreeList`第一个空闲对象并返回，分配结束
-   如果`FreeList`为空，从`CentralCache`中该`SizeClass`对应的`CentralFreeList`加锁一次性获取一堆空闲对象（如果`CentralFreeList`也为空的则向`PageHeap`申请一个`Span`拆分成`Size Class`对应大小的空闲对象，放入`CentralFreeList`中），将这堆对象（除第一个对象外）放到`ThreadCache`中`Size Class`对应的`FreeList`，返回第一个对象，分配结束

回收流程如下：

-   应用程序调用`free()`或者`delete`触发回收
-   将该对象对应的内存插入到`ThreadCache`中该`Size Class`对应的`FreeList`中
-   仅当满足一定条件时，`ThreadCache`中的空闲对象才会回到`CentralCache`中作为线程间的公共缓存

### **3. 中对象分配**

对于超过`256KB`但又不超过`1MB`（即`128 Pages`）的中对象，分配流程如下：

-   将该对象所需要的内存向上取整到`k(k <=128)`个`Page`，因此最多会产生`8KB`的内存碎片
-   从`PageHeap`中的`k pages list`的链表中开始按顺序找到一个非空的链表（假如是`n pages list, n>=k`），取出这个非空链表中的一个`span`并拆分成`k pages`和`n-k pages`的两个`span`，前者作为分配结果返回，后者插入到`n-k pages list`
-   如果一直找到`128 pages list`都没找到非空链表，则把这次分配当成大对象分配

### **4. 大对象分配**

对于超过`128 pages`的大对象，分配策略如下：

-   将该对象所需要的内存向上取整到`k`个`page`
-   搜索`large span set`，找到不小于`k`个`page`的最小`span`（假如是`n pages`），将该`span`拆成`k pages`和`n-k pages`的两个`span`，前者作为结果返回，后者根据是否大于`128 pages`选择插入到`n-k pages list`或者`large span set`
-   如果找不到合适的`span`，使用`sbrk`或者`mmap`向系统申请新的内存生成新的`span`，再执行一次大对象的分配算法
### **2. 对象分级与多级缓存**

`golang`内存管理是在`tcmalloc`基础上实现的，同样实现了对象分级：
![[Pasted image 20220302182832.png]]

> 类似于绝大多数对象是“朝生夕死”的，绝大对数对象的大小都小于`32KB`，对不同大小的对象分级管理并针对性实现对象的分配和回收算法有助于提升程序执行效率。

同样`golang`的内存管理也实现了`Thread Cache`、`Central Cache`和`PageHeap`的三级缓存，这样做的好处包括：

-   对于小对象的分配可以直接从`Thread Cache`获取对应的内存空间，并且每个`Thread Cache`是独立的因此无需加锁，极大提高了内存申请的效率
-   绝对多数对象都是小对象，因此这种做法可以保证大部分内存申请的操作是高效的

# 内存管理组件

内存分配由内存分配器完成。分配器由3种组件构成：`mcache`, `mcentral`, `mheap`。

## mcache
由于同一时间内只能有一个线程访问同一个`P`，因此`P`中的数据不需要加锁。每个`P`都绑定了`span`的缓存（被称为`mcache`）用于缓存用户程序申请的微对象，每一个`mcache`都拥有`67 * 2`个`mspan`。
实例图如下：

![mcache](https://img.luozhiyun.com/20210129154022.png)

**图中alloc是一个拥有134个元素的mspan数组，mspan数组管理数个page大小的内存，每个page是8k，page的数量由spanclass规格决定。**
![](https://pic4.zhimg.com/80/v2-4920c0a0d42f887c6d1924160adb2b47_720w.jpg)

> scan和noscan的区别：  
> 如果对象包含指针，分配对象时会使用scan的span；如果对象不包含指针，则分配对象时会使用noscan的span。如此在垃圾回收时对于noscan的span可以不用查看bitmap来标记子对象，提高GC效率。

`mcache`在初始化时候是没有任何`mspan`资源的，在使用过程中动态地向 `mcentral`申请，之后会缓存下来。 

`mcache`：每个工作线程都会绑定一个mcache，本地缓存可用的`mspan`资源，这样就可以直接给Goroutine分配，因为不存在多个Goroutine竞争的情况，所以不会消耗锁资源。

`mcache`的结构体定义：
```Go
type mcache struct {
    ...
    // 申请小对象的起始地址
    tiny             uintptr
    // 从起始地址tiny开始的偏移量
    tinyoffset       uintptr
    // tiny对象分配的数量
    local_tinyallocs uintptr // number of tiny allocs not counted in other stats
    // mspan对象集合，numSpanClasses=134
    alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass
    ...
}
​
numSpanClasses = _NumSizeClasses << 1
```

`mcache`用`Span Classes`作为索引管理多个用于分配的`mspan`，它包含所有规格的`mspan`。它是`_NumSizeClasses`的2倍，也就是`67*2=134`，为什么有一个两倍的关系，前面我们提到过：为了加速之后内存回收的速度，数组里一半的`mspan`中分配的对象不包含指针，另一半则包含指针。

对于无指针对象的`mspan`在进行垃圾回收的时候无需进一步扫描它是否引用了其他活跃的对象。 后面的垃圾回收文章会再讲到，这次先到这里。

![mcache](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c75aa46b89~tplv-t2oaga2asx-watermark.awebp)

`mcache`在初始化的时候是没有任何`mspan`资源的，在使用过程中会动态地从`mcentral`申请，之后会缓存下来。当对象小于等于32KB大小时，使用`mcache`的相应规格的`mspan`进行分配。

## mcentral
`mcentral`为所有线程的提供切分好的`mspan`资源，每个`mcentral`会保存一种特定大小的全局`mspan`列表，包括已分配出去的和未分配出去的。

```go
type mcentral struct {
    lock mutex         // 互斥锁
    sizeclass int32    // 规格
    nonempty mSpanList // 尚有空闲object的mspan链表
    empty mSpanList    // 没有空闲object的mspan链表，或者是已被mcache取走的msapn链表
    nmalloc uint64     // 已累计分配的对象个数
}
```

`mcache`和`mcentral`的交互：

-   `mcache`没有足够的`mspan`时加锁从`mcentral`的`nonempty`中获取`mspan`并从`nonempty`删除，取出后加入`empty`链表
-   `mcache`归还时将`mspan`从`empty`链表删除并重新添加回`noempty`

`mcentral`：为所有`mcache`提供切分好的`mspan`资源。每个`central`保存一种特定大小的全局`mspan`列表，包括已分配出去的和未分配出去的。 每个`mcentral`对应一种`mspan`，而`mspan`的种类导致它分割的`object`大小不同。当工作线程的`mcache`中没有合适（也就是特定大小的）的`mspan`时就会从`mcentral`获取。

`mcentral`被所有的工作线程共同享有，存在多个Goroutine竞争的情况，因此会消耗锁资源。结构体定义：

```go
//path: /usr/local/go/src/runtime/mcentral.go

type mcentral struct {
    // 互斥锁
    lock mutex 
    
    // 规格
    sizeclass int32 
    
    // 尚有空闲object的mspan链表
    nonempty mSpanList 
    
    // 没有空闲object的mspan链表，或者是已被mcache取走的msapn链表
    empty mSpanList 
    
    // 已累计分配的对象个数
    nmalloc uint64 
}

```

![mcentral](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c75ac81dae~tplv-t2oaga2asx-watermark.awebp)

`empty`表示这条链表里的`mspan`都被分配了`object`，或者是已经被`cache`取走了的`mspan`，这个`mspan`就被那个工作线程独占了。而`nonempty`则表示有空闲对象的`mspan`列表。每个`central`结构体都在`mheap`中维护。

简单说下`mcache`从`mcentral`获取和归还`mspan`的流程：

-   获取 加锁；从`nonempty`链表找到一个可用的`mspan`；并将其从`nonempty`链表删除；将取出的`mspan`加入到`empty`链表；将`mspan`返回给工作线程；解锁。
    
-   归还 加锁；将`mspan`从`empty`链表删除；将`mspan`加入到`nonempty`链表；解锁。
    

## mheap

`mheap`：代表Go程序持有的所有堆空间，Go程序使用一个`mheap`的全局对象`_mheap`来管理堆内存。

当`mcentral`没有空闲的`mspan`时，会向`mheap`申请。而`mheap`没有资源时，会向操作系统申请新内存。`mheap`主要用于大对象的内存分配，以及管理未切割的`mspan`，用于给`mcentral`切割成小对象。

同时我们也看到，`mheap`中含有所有规格的`mcentral`，所以，当一个`mcache`从`mcentral`申请`mspan`时，只需要在独立的`mcentral`中使用锁，并不会影响申请其他规格的`mspan`。

`mheap`结构体定义：

```go
//path: /usr/local/go/src/runtime/mheap.go

type mheap struct {
	lock mutex
	
	// spans: 指向mspans区域，用于映射mspan和page的关系
	spans []*mspan 
	
	// 指向bitmap首地址，bitmap是从高地址向低地址增长的
	bitmap uintptr 

    // 指示arena区首地址
	arena_start uintptr 
	
	// 指示arena区已使用地址位置
	arena_used  uintptr 
	
	// 指示arena区末地址
	arena_end   uintptr 
	//各个规格的mcentral集合
	central [67*2]struct {
		mcentral mcentral
		pad [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
	}
}
```

![mheap](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/13/169755c8b91af501~tplv-t2oaga2asx-watermark.awebp)

上图我们看到，bitmap和arena_start指向了同一个地址，这是因为bitmap的地址是从高到低增长的，所以他们指向的内存位置相同。

# 分配流程

上一篇文章[《Golang之变量去哪儿》](https://link.juejin.cn/?target=https%3A%2F%2Fwww.cnblogs.com%2Fqcrao-2018%2Fp%2F10453260.html "https://www.cnblogs.com/qcrao-2018/p/10453260.html")中我们提到了，变量是在栈上分配还是在堆上分配，是由逃逸分析的结果决定的。通常情况下，编译器是倾向于将变量分配到栈上的，因为它的开销小，最极端的就是"zero garbage"，所有的变量都会在栈上分配，这样就不会存在内存碎片，垃圾回收之类的东西。

Go的内存分配器在分配对象时，根据对象的大小，分成三类：小对象（小于等于16B）、一般对象（大于16B，小于等于32KB）、大对象（大于32KB）。

大体上的分配流程：

-   > 32KB 的对象，直接从mheap上分配；
    
-   <=16B 的对象使用mcache的tiny分配器分配；
-   (16B,32KB] 的对象，首先计算对象的规格大小，然后使用mcache中相应规格大小的mspan分配；
-   如果mcache没有相应规格大小的mspan，则向mcentral申请
-   如果mcentral没有相应规格大小的mspan，则向mheap申请
-   如果mheap中也没有合适大小的mspan，则向操作系统申请

# 总结

-   Go在程序启动时，会向操作系统申请一大块内存，之后自行管理。
-   Go内存管理的基本单元是mspan，它由若干个页组成，每种mspan可以分配特定大小的object。
-   mcache, mcentral, mheap是Go内存管理的三大组件，层层递进。mcache管理线程在本地缓存的mspan；mcentral管理全局的mspan供所有线程使用；mheap管理Go的所有动态分配内存。
-   极小对象会分配在一个object中，以节省资源，使用tiny分配器分配内存；一般小对象通过mspan分配内存；大对象则直接由mheap分配内存。

  
![](https://pic2.zhimg.com/80/v2-1afa6e523c990b22632ef7c8f8553425_720w.jpg)

golang从堆分配对象是通过runtime包中的nowobject函数实现的，函数执行流程如下：

![](https://pic4.zhimg.com/80/v2-52ef10dd13907ccf43bfc0d706e73e4f_720w.jpg)

![](https://upload-images.jianshu.io/upload_images/7515493-bd316b2abaa4965d.png?imageMogr2/auto-orient/strip|imageView2/2/w/850/format/webp)
![](https://upload-images.jianshu.io/upload_images/7515493-0d3c3e96475d29f4.png?imageMogr2/auto-orient/strip|imageView2/2/w/665/format/webp)

流程图

# 源码角度
### 断点调试汇编

目前Go语言支持GDB、LLDB和Delve几种调试器。只有Delve是专门为Go语言设计开发的调试工具。而且Delve本身也是采用Go语言开发，对Windows平台也提供了一样的支持。本节我们基于Delve简单解释如何调试Go汇编程序。项目地址：[https://github.com/go-delve/delve](https://github.com/go-delve/delve)

安装：

```
go get github.com/go-delve/delve/cmd/dlv
```

首先编写一个test.go的一个例子：

```go
package main

import "fmt"

type A struct {
    test string
}
func main() {
    a := new(A)
    fmt.Println(a)
}
```

然后命令行进入包所在目录，然后输入`dlv debug`命令进入调试：

```powershell
PS C:\document\code\test_go\src> dlv debug
Type 'help' for list of commands.
```

然后可以使用break命令在main包的main方法上设置一个断点：

```powershell
(dlv) break main.main
Breakpoint 1 set at 0x4bd30a for main.main() c:/document/code/test_go/src/test.go:8
```

通过breakpoints查看已经设置的所有断点：

```powershell
(dlv) breakpoints
Breakpoint runtime-fatal-throw at 0x4377e0 for runtime.fatalthrow() c:/software/go/src/runtime/panic.go:1162 (0)
Breakpoint unrecovered-panic at 0x437860 for runtime.fatalpanic() c:/software/go/src/runtime/panic.go:1189 (0)
        print runtime.curg._panic.arg
Breakpoint 1 at 0x4bd30a for main.main() c:/document/code/test_go/src/test.go:8 (0)
```

通过continue命令让程序运行到下一个断点处：

```powershell
(dlv) continue
> main.main() c:/document/code/test_go/src/test.go:8 (hits goroutine(1):1 total:1) (PC: 0x4bd30a)
     3: import "fmt"
     4:
     5: type A struct {
     6:         test string
     7: }
=>   8: func main() {
     9:         a := new(A)
    10:         fmt.Println(a)
    11: }
    12:
    13:
```

通过disassemble反汇编命令查看main函数对应的汇编代码：

```powershell
(dlv) disassemble
TEXT main.main(SB) C:/document/code/test_go/src/test.go
        test.go:8       0x4bd2f0        65488b0c2528000000      mov rcx, qword ptr gs:[0x28]
        test.go:8       0x4bd2f9        488b8900000000          mov rcx, qword ptr [rcx]
        test.go:8       0x4bd300        483b6110                cmp rsp, qword ptr [rcx+0x10]
        test.go:8       0x4bd304        0f8697000000            jbe 0x4bd3a1
=>      test.go:8       0x4bd30a*       4883ec78                sub rsp, 0x78
        test.go:8       0x4bd30e        48896c2470              mov qword ptr [rsp+0x70], rbp
        test.go:8       0x4bd313        488d6c2470              lea rbp, ptr [rsp+0x70]
        test.go:9       0x4bd318        488d0581860100          lea rax, ptr [__image_base__+874912]
        test.go:9       0x4bd31f        48890424                mov qword ptr [rsp], rax
        test.go:9       0x4bd323        e8e800f5ff              call $runtime.newobject
        test.go:9       0x4bd328        488b442408              mov rax, qword ptr [rsp+0x8]
        test.go:9       0x4bd32d        4889442430              mov qword ptr [rsp+0x30], rax
        test.go:10      0x4bd332        4889442440              mov qword ptr [rsp+0x40], rax
        test.go:10      0x4bd337        0f57c0                  xorps xmm0, xmm0
        test.go:10      0x4bd33a        0f11442448              movups xmmword ptr [rsp+0x48], xmm0
        test.go:10      0x4bd33f        488d442448              lea rax, ptr [rsp+0x48]
        test.go:10      0x4bd344        4889442438              mov qword ptr [rsp+0x38], rax
        test.go:10      0x4bd349        8400                    test byte ptr [rax], al
        test.go:10      0x4bd34b        488b4c2440              mov rcx, qword ptr [rsp+0x40]
        test.go:10      0x4bd350        488d15099f0000          lea rdx, ptr [__image_base__+815712]
        test.go:10      0x4bd357        4889542448              mov qword ptr [rsp+0x48], rdx
        test.go:10      0x4bd35c        48894c2450              mov qword ptr [rsp+0x50], rcx
        test.go:10      0x4bd361        8400                    test byte ptr [rax], al
        test.go:10      0x4bd363        eb00                    jmp 0x4bd365
        test.go:10      0x4bd365        4889442458              mov qword ptr [rsp+0x58], rax
        test.go:10      0x4bd36a        48c744246001000000      mov qword ptr [rsp+0x60], 0x1
        test.go:10      0x4bd373        48c744246801000000      mov qword ptr [rsp+0x68], 0x1
        test.go:10      0x4bd37c        48890424                mov qword ptr [rsp], rax
        test.go:10      0x4bd380        48c744240801000000      mov qword ptr [rsp+0x8], 0x1
        test.go:10      0x4bd389        48c744241001000000      mov qword ptr [rsp+0x10], 0x1
        test.go:10      0x4bd392        e869a0ffff              call $fmt.Println
        test.go:11      0x4bd397        488b6c2470              mov rbp, qword ptr [rsp+0x70]
        test.go:11      0x4bd39c        4883c478                add rsp, 0x78
        test.go:11      0x4bd3a0        c3                      ret
        test.go:8       0x4bd3a1        e82a50faff              call $runtime.morestack_noctxt
        .:0             0x4bd3a6        e945ffffff              jmp $main.main
```

现在我们可以使用break断点到runtime.newobject函数的调用上：

```powershell
(dlv) break runtime.newobject
Breakpoint 2 set at 0x40d426 for runtime.newobject() c:/software/go/src/runtime/malloc.go:1164
```

输入continue跳到断点的位置：

```powershell
(dlv) continue
> runtime.newobject() c:/software/go/src/runtime/malloc.go:1164 (hits goroutine(1):1 total:1) (PC: 0x40d426)
Warning: debugging optimized function
  1159: }
  1160:
  1161: // implementation of new builtin
  1162: // compiler (both frontend and SSA backend) knows the signature
  1163: // of this function
=>1164: func newobject(typ *_type) unsafe.Pointer {
  1165:         return mallocgc(typ.size, typ, true)
  1166: }
  1167:
  1168: //go:linkname reflect_unsafe_New reflect.unsafe_New
  1169: func reflect_unsafe_New(typ *_type) unsafe.Pointer {
```

print命令来查看typ的数据：

```powershell
(dlv) print typ
*runtime._type {size: 16, ptrdata: 8, hash: 875453117, tflag: tflagUncommon|tflagExtraStar|tflagNamed (7), align: 8, fieldAlign: 8, kind: 25, equal: runtime.strequal, gcdata: *1, str: 5418, ptrToThis: 37472}
```

可以看到这里打印的size是16bytes，因为我们A结构体里面就一个string类型的field。

进入到mallocgc方法后，通过args和locals命令查看函数的参数和局部变量：

```powershell
(dlv) args
size = (unreadable could not find loclist entry at 0x8b40 for address 0x40ca73)
typ = (*runtime._type)(0x4d59a0)
needzero = true
~r3 = (unreadable empty OP stack)
(dlv) locals
(no locals)
```

### 各个对象入口

我们根据汇编可以判断，所有的函数入口都是`runtime.mallocgc`，但是下面两个对象需要注意一下：

### int64对象

`runtime.convT64`

```go
func convT64(val uint64) (x unsafe.Pointer) {
    if val < uint64(len(staticuint64s)) {
        x = unsafe.Pointer(&staticuint64s[val])
    } else {
        x = mallocgc(8, uint64Type, false)
        *(*uint64)(x) = val
    }
    return
}
```

这段代码表示如果一个int64类型的值小于256，直接获取的是缓存值，那么这个值不会进行内存分配。

### string对象

`runtime.convTstring`

```go
func convTstring(val string) (x unsafe.Pointer) {
    if val == "" {
        x = unsafe.Pointer(&zeroVal[0])
    } else {
        x = mallocgc(unsafe.Sizeof(val), stringType, true)
        *(*string)(x) = val
    }
    return
}
```

由这段代码显示，如果是创建一个为”“的string对象，那么会直接返回一个固定的地址值，不会进行内存分配。
### 调试用例

大家在调试的时候也可以使用下面的例子来进行调试，因为go里面的对象分配是分为大对象、小对象、微对象的，所以下面准备了三个方法分别对应三种对象的创建时的调试。

```go
type smallobj struct {
    arr [1 << 10]byte
}

type largeobj struct {
    arr [1 << 26]byte
}

func tiny()   {
    y := 100000
    fmt.Println(y)
}

func large() {
    large := largeobj{}
    println(&large)
}

func small() {
    small := smallobj{}
    print(&small)
}

func main() {
    //tiny()
    //small()
    //large() 
}
```


## 给对象分配内存

我们通过对源码的反编译可以知道，堆上所有的对象都会通过调用`runtime.newobject`函数分配内存，该函数会调用`runtime.mallocgc`:

```go
//创建一个新的对象
func newobject(typ *_type) unsafe.Pointer {
    //size表示该对象的大小
    return mallocgc(typ.size, typ, true)
}

// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if gcphase == _GCmarktermination {
		throw("mallocgc called with gcphase == _GCmarktermination")
	}

	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}

	if debug.sbrk != 0 {
		align := uintptr(16)
		if typ != nil {
			align = uintptr(typ.align)
		}
		return persistentalloc(size, align, &memstats.other_sys)
	}

	// 判断是否要辅助GC工作
	// gcBlackenEnabled在GC的标记阶段会开启
	// assistG is the G to charge for this allocation, or nil if
	// GC is not currently active.
	var assistG *g
	if gcBlackenEnabled != 0 {
		// Charge the current user G for this allocation.
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		// 减去内存值
		// Charge the allocation against the G. We'll account
		// for internal fragmentation at the end of mallocgc.
		assistG.gcAssistBytes -= int64(size)

		// 会按分配的大小判断需要协助GC完成多少工作
		// 具体的算法将在下面讲解收集器时说明
		if assistG.gcAssistBytes < 0 {
			// This G is in debt. Assist the GC to correct
			// this before allocating. This must happen
			// before disabling preemption.
			gcAssistAlloc(assistG)
		}
	}

	// 增加当前G对应的M的lock计数, 防止这个G被抢占
	// Set mp.mallocing to keep from being preempted by GC.
	mp := acquirem()
	if mp.mallocing != 0 {
		throw("malloc deadlock")
	}
	if mp.gsignal == getg() {
		throw("malloc during signal")
	}
	mp.mallocing = 1

	shouldhelpgc := false
	dataSize := size
	// 获取当前G对应的M对应的P的本地span缓存(mcache)
	// 因为M在拥有P后会把P的mcache设到M中, 这里返回的是getg().m.mcache
	// 获取mcache，用于处理微对象和小对象的分配
	c := gomcache()
	var x unsafe.Pointer
	// 表示对象是否包含指针，true表示对象里没有指针
	noscan := typ == nil || typ.kind&kindNoPointers != 0
	// 判断是否小对象, maxSmallSize当前的值是32K
	if size <= maxSmallSize {
		// 如果对象不包含指针, 并且对象的大小小于16 bytes, 可以做特殊处理
		// 这里是针对非常小的对象的优化, 因为span的元素最小只能是8 byte, 如果对象更小那么很多空间都会被浪费掉
		// 非常小的对象可以整合在"class 2 noscan"的元素(大小为16 byte)中
		if noscan && size < maxTinySize {
			// Tiny allocator.
			//
			// Tiny allocator combines several tiny allocation requests
			// into a single memory block. The resulting memory block
			// is freed when all subobjects are unreachable. The subobjects
			// must be noscan (don't have pointers), this ensures that
			// the amount of potentially wasted memory is bounded.
			//
			// Size of the memory block used for combining (maxTinySize) is tunable.
			// Current setting is 16 bytes, which relates to 2x worst case memory
			// wastage (when all but one subobjects are unreachable).
			// 8 bytes would result in no wastage at all, but provides less
			// opportunities for combining.
			// 32 bytes provides more opportunities for combining,
			// but can lead to 4x worst case wastage.
			// The best case winning is 8x regardless of block size.
			//
			// Objects obtained from tiny allocator must not be freed explicitly.
			// So when an object will be freed explicitly, we ensure that
			// its size >= maxTinySize.
			//
			// SetFinalizer has a special case for objects potentially coming
			// from tiny allocator, it such case it allows to set finalizers
			// for an inner byte of a memory block.
			//
			// The main targets of tiny allocator are small strings and
			// standalone escaping variables. On a json benchmark
			// the allocator reduces number of allocations by ~12% and
			// reduces heap size by ~20%.
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			span := c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			// 否则按普通的小对象分配
			// 首先获取对象的大小应该使用哪个span类型
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}
			size = uintptr(class_to_size[sizeclass])
			// 等于sizeclass * 2 + (noscan ? 1 : 0)
			spc := makeSpanClass(sizeclass, noscan)
			span := c.alloc[spc]
			// 尝试快速的从这个span中分配
			v := nextFreeFast(span)
			if v == 0 {
				// 分配失败, 可能需要从mcentral或者mheap中获取
				// 如果从mcentral或者mheap获取了新的span, 则shouldhelpgc会等于true
				// shouldhelpgc会等于true时会在下面判断是否要触发GC
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
		// 大对象直接从mheap分配, 这里的s是一个特殊的span, 它的class是0
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			s = largeAlloc(size, needzero, noscan)
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}

	// 设置arena对应的bitmap, 记录哪些位置包含了指针, GC会使用bitmap扫描所有可到达的对象
	var scanSize uintptr
	if !noscan {
		// If allocating a defer+arg block, now that we've picked a malloc size
		// large enough to hold everything, cut the "asked for" size down to
		// just the defer header, so that the GC bitmap will record the arg block
		// as containing nothing at all (as if it were unused space at the end of
		// a malloc block caused by size rounding).
		// The defer arg areas are scanned as part of scanstack.
		if typ == deferType {
			dataSize = unsafe.Sizeof(_defer{})
		}
		// 这个函数非常的长, 有兴趣的可以看
		// https://github.com/golang/go/blob/go1.9.2/src/runtime/mbitmap.go#L855
		// 虽然代码很长但是设置的内容跟上面说过的bitmap区域的结构一样
		// 根据类型信息设置scan bit跟pointer bit, scan bit成立表示应该继续扫描, pointer bit成立表示该位置是指针
		// 需要注意的地方有
		// - 如果一个类型只有开头的地方包含指针, 例如[ptr, ptr, large non-pointer data]
		//   那么后面的部分的scan bit将会为0, 这样可以大幅提升标记的效率
		// - 第二个slot的scan bit用途比较特殊, 它并不用于标记是否继续scan, 而是标记checkmark
		// 什么是checkmark
		// - 因为go的并行GC比较复杂, 为了检查实现是否正确, go需要在有一个检查所有应该被标记的对象是否被标记的机制
		//   这个机制就是checkmark, 在开启checkmark时go会在标记阶段的最后停止整个世界然后重新执行一次标记
		//   上面的第二个slot的scan bit就是用于标记对象在checkmark标记中是否被标记的
		// - 有的人可能会发现第二个slot要求对象最少有两个指针的大小, 那么只有一个指针的大小的对象呢?
		//   只有一个指针的大小的对象可以分为两种情况
		//   对象就是指针, 因为大小刚好是1个指针所以并不需要看bitmap区域, 这时第一个slot就是checkmark
		//   对象不是指针, 因为有tiny alloc的机制, 不是指针且只有一个指针大小的对象会分配在两个指针的span中
		//               这时候也不需要看bitmap区域, 所以和上面一样第一个slot就是checkmark
		heapBitsSetType(uintptr(x), size, dataSize, typ)
		if dataSize > typ.size {
			// Array allocation. If there are any
			// pointers, GC has to scan to the last
			// element.
			if typ.ptrdata != 0 {
				scanSize = dataSize - typ.size + typ.ptrdata
			}
		} else {
			scanSize = typ.ptrdata
		}
		c.local_scan += scanSize
	}

	// 内存屏障, 因为x86和x64的store不会乱序所以这里只是个针对编译器的屏障, 汇编中是ret
	// Ensure that the stores above that initialize x to
	// type-safe memory and set the heap bits occur before
	// the caller can make x observable to the garbage
	// collector. Otherwise, on weakly ordered machines,
	// the garbage collector could follow a pointer to x,
	// but see uninitialized memory or stale heap bits.
	publicationBarrier()

	// 如果当前在GC中, 需要立刻标记分配后的对象为"黑色", 防止它被回收
	// Allocate black during GC.
	// All slots hold nil so no scanning is needed.
	// This may be racing with GC so do it atomically if there can be
	// a race marking the bit.
	if gcphase != _GCoff {
		gcmarknewobject(uintptr(x), size, scanSize)
	}

	// Race Detector的处理(用于检测线程冲突问题)
	if raceenabled {
		racemalloc(x, size)
	}

	// Memory Sanitizer的处理(用于检测危险指针等内存问题)
	if msanenabled {
		msanmalloc(x, size)
	}

	// 重新允许当前的G被抢占
	mp.mallocing = 0
	releasem(mp)

	// 除错记录
	if debug.allocfreetrace != 0 {
		tracealloc(x, size, typ)
	}

	// Profiler记录
	if rate := MemProfileRate; rate > 0 {
		if size < uintptr(rate) && int32(size) < c.next_sample {
			c.next_sample -= int32(size)
		} else {
			mp := acquirem()
			profilealloc(mp, x, size)
			releasem(mp)
		}
	}

	// gcAssistBytes减去"实际分配大小 - 要求分配大小", 调整到准确值
	if assistG != nil {
		// Account for internal fragmentation in the assist
		// debt now that we know it.
		assistG.gcAssistBytes -= int64(size - dataSize)
	}

	// 如果之前获取了新的span, 则判断是否需要后台启动GC
	// 这里的判断逻辑(gcTrigger)会在下面详细说明
	if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(gcBackgroundMode, t)
		}
	}

	return x
}

```
mallocgc是负责堆分配的关键函数，runtime中的new系列和make系列函数都依赖它。它的主要逻辑可以分成四个部分：

**第一部分：辅助GC**

如果程序申请堆内存时正处于GC标记阶段，那么，当下已分配的堆内存还没标记完，你这边又要分配新的堆内存。万一内存申请的速度超过了GC标记的速度，那就不妙了~

所以，申请一字节内存空间需要做多少扫描工作，或者说，完成一字节扫描工作后，可以分配多大的内存空间，那都是根据GC扫描的进度更新计算的，绝对合理！

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibjI8pEWI9L7iaaviaI8S6PnSuVlYU6AvSw3NOHDMtAAJibU2k26R1kM3uSQLysDfdxfTHgPzqqwkMXq4UjGicmeXibQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

不过既然都开始辅助GC了，也不能白折腾一场，切换来切换去的好不麻烦~

所以每次执行辅助GC，最少要扫描64KB。

先不要替那些申请小块内存的协程感到不公平，因为协程每次执行辅助GC，多出来的部分会作为信用存储到当前G中，就像信用卡的额度一样，后续再执行mallocgc()时，只要信用额度用不完，就不用执行辅助GC了~

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibjI8pEWI9L7iaaviaI8S6PnSuVlYU6AvSwEAsEiaw1TmnApbsQ4UZ5v6d8aqdLYrCTiboCo0aGbW9SeiauOj66sVJ5Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

此外，还有一种偷懒的办法来逃避辅助GC的责任，那就是：

**窃取信用**

后台的GC mark worker执行扫描任务，会在全局gcController的bgScanCredit这里积累信用。如果能够窃取足够多的信用值来抵消当前协程背负的债务，那也就不用执行辅助GC了~

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibjI8pEWI9L7iaaviaI8S6PnSuVlYU6AvSwibicGicSGZdNND6YGmtopO1yskEAcfrjj1vws8iaSlNllZHTqBWpo1AFfQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

  

过了辅助GC这一关，就进入到  

**第二部分：空间分配**
通过mallocgc的代码可以知道，mallocgc在分配内存的时候，会按照对象的大小分为3档来进行分配：

1.  小于16bytes的小对象；
2.  在16bytes与32k之间的微对象；
3.  大于 32 Kb的大对象；

## 大对象分配

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer { 
    ...  
    var s *mspan
    shouldhelpgc = true
    systemstack(func() {
        s = largeAlloc(size, needzero, noscan)
    })
    s.freeindex = 1
    s.allocCount = 1
    x = unsafe.Pointer(s.base())
    size = s.elemsize
    ... 
    return x
}
```

从上面我们可以看到分配大于32KB的空间时，直接使用largeAlloc来分配一个mspan。

```go
func largeAlloc(size uintptr, needzero bool, noscan bool) *mspan {
    // _PageSize=8k,也就是表明对象太大，溢出
    if size+_PageSize < size {
        throw("out of memory")
    }
    // _PageShift==13，计算需要分配的页数
    npages := size >> _PageShift
    // 如果不是整数，多出来一些，需要加1
    if size&_PageMask != 0 {
        npages++
    } 
    ...
    // 从堆上分配
    s := mheap_.alloc(npages, makeSpanClass(0, noscan), needzero)
    if s == nil {
        throw("out of memory")
    }
    ...
    return s
}
```

在分配内存的时候是按页来进行分配的，每个页的大小是_PageSize（8K），然后需要根据传入的size来判断需要分多少页，最后调用alloc从堆上分配。

```go
func (h *mheap) alloc(npages uintptr, spanclass spanClass, needzero bool) *mspan {
    var s *mspan
    systemstack(func() { 
        if h.sweepdone == 0 {
            // 回收一部分内存
            h.reclaim(npages)
        }
        // 进行内存分配
        s = h.allocSpan(npages, false, spanclass, &memstats.heap_inuse)
    }) 
    ...
    return s
}
```

继续看allocSpan的实现：

```go
const pageCachePages = 8 * unsafe.Sizeof(pageCache{}.cache)

func (h *mheap) allocSpan(npages uintptr, manual bool, spanclass spanClass, sysStat *uint64) (s *mspan) {
    // Function-global state.
    gp := getg()
    base, scav := uintptr(0), uintptr(0)

    pp := gp.m.p.ptr()
    // 申请的内存比较小,尝试从pcache申请内存
    if pp != nil && npages < pageCachePages/4 {
        c := &pp.pcache

        if c.empty() {
            lock(&h.lock)
            *c = h.pages.allocToCache()
            unlock(&h.lock)
        } 

        base, scav = c.alloc(npages)
        if base != 0 {
            s = h.tryAllocMSpan()

            if s != nil && gcBlackenEnabled == 0 && (manual || spanclass.sizeclass() != 0) {
                goto HaveSpan
            } 
        }
    } 
    lock(&h.lock)
    // 内存比较大或者线程的页缓存中内存不足，从mheap的pages上获取内存
    if base == 0 { 
        base, scav = h.pages.alloc(npages)
        // 内存也不够，那么进行扩容
        if base == 0 {
            if !h.grow(npages) {
                unlock(&h.lock)
                return nil
            }
            // 重新申请内存
            base, scav = h.pages.alloc(npages)
            // 内存不足，抛出异常
            if base == 0 {
                throw("grew heap, but no adequate free space found")
            }
        }
    }
    if s == nil { 
        // 分配一个mspan对象
        s = h.allocMSpanLocked()
    }

    unlock(&h.lock)

HaveSpan: 
    // 设置参数初始化
    s.init(base, npages) 
    ...
    // 建立mheap与mspan之间的联系
    h.setSpans(s.base(), npages, s)
    ...
    return s
}
```

这里会根据需要分配的内存大小再判断一次：

-   如果要分配的页数小于`pageCachePages/4=64/4=16`页，那么就尝试从pcache申请内存；
-   如果申请的内存比较大或者线程的页缓存中内存不足，会通过`runtime.pageAlloc.alloc`从页堆分配内存；
-   如果页堆上内存不足，那么就mheap的grow方法从系统上申请内存，然后再调用pageAlloc的alloc分配内存；

下面来看看grow的向操作系统申请内存：

```go
func (h *mheap) grow(npage uintptr) bool {
    // We must grow the heap in whole palloc chunks.
    ask := alignUp(npage, pallocChunkPages) * pageSize

    totalGrowth := uintptr(0)
    nBase := alignUp(h.curArena.base+ask, physPageSize)
    // 内存不够则调用sysAlloc申请内存
    if nBase > h.curArena.end { 
    // 调用mheap.sysAlloc函数申请
        av, asize := h.sysAlloc(ask)
        if av == nil {
            print("runtime: out of memory: cannot allocate ", ask, "-byte block (", memstats.heap_sys, " in use)\n")
            return false
        }
        // 重新设置curArena的值
        if uintptr(av) == h.curArena.end { 
            h.curArena.end = uintptr(av) + asize
        } else { 
            if size := h.curArena.end - h.curArena.base; size != 0 {
                h.pages.grow(h.curArena.base, size)
                totalGrowth += size
            } 
            h.curArena.base = uintptr(av)
            h.curArena.end = uintptr(av) + asize
        } 
        nBase = alignUp(h.curArena.base+ask, physPageSize)
    } 
    ...
    return true
}
```

grow会通过curArena的end值来判断是不是需要从系统申请内存；如果end小于nBase那么会调用`runtime.mheap.sysAlloc`方法从操作系统中申请更多的内存；

```go
func (h *mheap) sysAlloc(n uintptr) (v unsafe.Pointer, size uintptr) {
    n = alignUp(n, heapArenaBytes)

    // 在预先保留的内存中申请一块可以使用的空间
    v = h.arena.alloc(n, heapArenaBytes, &memstats.heap_sys)
    if v != nil {
        size = n
        goto mapped
    } 
    // 根据页堆的arenaHints在目标地址上尝试扩容
    for h.arenaHints != nil {
        hint := h.arenaHints
        p := hint.addr
        if hint.down {
            p -= n
        }
        if p+n < p {
            // We can't use this, so don't ask.
            v = nil
        } else if arenaIndex(p+n-1) >= 1<<arenaBits {
            // Outside addressable heap. Can't use.
            v = nil
        } else {
            // 从操作系统中申请内存
            v = sysReserve(unsafe.Pointer(p), n)
        }
        if p == uintptr(v) {
            // Success. Update the hint.
            if !hint.down {
                p += n
            }
            hint.addr = p
            size = n
            break
        } 
        if v != nil {
            sysFree(v, n, nil)
        }
        h.arenaHints = hint.next
        h.arenaHintAlloc.free(unsafe.Pointer(hint))
    }  
    ...
    // 将内存由Reserved转为Prepared
    sysMap(v, size, &memstats.heap_sys)

mapped:
    // Create arena metadata.
    // 初始化一个heapArena来管理刚刚申请的内存
    for ri := arenaIndex(uintptr(v)); ri <= arenaIndex(uintptr(v)+size-1); ri++ {
        l2 := h.arenas[ri.l1()]
        if l2 == nil { 
            l2 = (*[1 << arenaL2Bits]*heapArena)(persistentalloc(unsafe.Sizeof(*l2), sys.PtrSize, nil))
            if l2 == nil {
                throw("out of memory allocating heap arena map")
            }
            atomic.StorepNoWB(unsafe.Pointer(&h.arenas[ri.l1()]), unsafe.Pointer(l2))
        }  
        var r *heapArena
        r = (*heapArena)(h.heapArenaAlloc.alloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gc_sys))
        ...  
        // 将创建heapArena放入到arenas列表中
        h.allArenas = h.allArenas[:len(h.allArenas)+1]
        h.allArenas[len(h.allArenas)-1] = ri
        atomic.StorepNoWB(unsafe.Pointer(&l2[ri.l2()]), unsafe.Pointer(r))
    }
    return
}
```

sysAlloc方法会调用`runtime.linearAlloc.alloc`预先保留的内存中申请一块可以使用的空间；如果没有会调用sysReserve方法会从操作系统中申请内存；最后初始化一个heapArena来管理刚刚申请的内存，然后将创建heapArena放入到arenas列表中。

至此，大对象的分配流程至此结束。

## 小对象分配

对于介于16bytes~32K的对象分配如下：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    ...
    dataSize := size
    // 获取mcache，用于处理微对象和小对象的分配
    c := gomcache()
    var x unsafe.Pointer
    // 表示对象是否包含指针，true表示对象里没有指针
    noscan := typ == nil || typ.ptrdata == 0
    // maxSmallSize=32768 32k
    if size <= maxSmallSize {
        // maxTinySize= 16 bytes 
        if noscan && size < maxTinySize { 
            ...
        } else {
            var sizeclass uint8
            //计算 sizeclass
            // smallSizeMax=1024
            if size <= smallSizeMax-8 {
                // smallSizeDiv=8
                sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
            } else {
                // largeSizeDiv=128,smallSizeMax = 1024
                sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
            }
            size = uintptr(class_to_size[sizeclass])
            spc := makeSpanClass(sizeclass, noscan)
            span := c.alloc[spc]
            //从对应的 span 里面分配一个 object 
            v := nextFreeFast(span)
            if v == 0 {
                // mcache不够用了，则从 mcentral 申请内存到 mcache
                v, span, shouldhelpgc = c.nextFree(spc)
            }
            x = unsafe.Pointer(v)
            if needzero && span.needzero != 0 {
                memclrNoHeapPointers(unsafe.Pointer(v), size)
            }
        } 
        ...
    }  
    ...
    return x
}
```

首先会先计算sizeclass 大小，计算 sizeclass 是通过预先定义两个数组：size_to_class8 和 size_to_class128。小于 1024 – 8 = 1016 （smallSizeMax=1024），使用 size_to_class8，否则使用数组 size_to_class128。

举个例子，比如要分配 20 byte 的内存，那么sizeclass = size_to_calss8[(20+7)/8] = size_to_class8[3] = 3。然后通过class_to_size[3]获取到对应的值32，表示应该要分配32bytes的内存值。

接着会从alloc数组中获取一个span的指针，通过调用nextFreeFast尝试从mcache中获取内存，如果mcache不够用了，则尝试调用nextFree从 mcentral 申请内存到 mcache。

下面看看nextFreeFast：

```go
func nextFreeFast(s *mspan) gclinkptr {
    // 获取allocCache二进制中0的个数
    //获取第一个非0的bit是第几个bit，也就是哪个元素是未分配的
    theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
    //找到未分配的元素
    if theBit < 64 {
        result := s.freeindex + uintptr(theBit)
        //要求索引值小于元素数量
        if result < s.nelems {
	        //下一个freeIndex
            freeidx := result + 1
            // 可以被64整除时需要特殊处理(参考nextFree)
            if freeidx%64 == 0 && freeidx != s.nelems {
                return 0
            }
            // 更新freeindex和allocCache(高位都是0, 用尽以后会更新)
            s.allocCache >>= uint(theBit + 1)
            // 返回元素所在的地址
            s.freeindex = freeidx
            // 添加已分配的元素计数
            s.allocCount++
            return gclinkptr(result*s.elemsize + s.base())
        }
    }
    return 0
}
```

allocCache在初始化的时候会初始化成`^uint64(0)`，换算成二进制，如果为0则表示被占用，通过allocCache可以快速的定位待分配的空间：

![allocCache](https://img.luozhiyun.com/20210129154059.png)

```go
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
//找到下一个freeIndex和更新allocCache
    s = c.alloc[spc]
    shouldhelpgc = false
    // 当前span中找到合适的index索引
    freeIndex := s.nextFreeIndex()
    // 当前span已经满了
    //如果span里面所有元素都已分配，则需要获取新的span
    if freeIndex == s.nelems { 
        if uintptr(s.allocCount) != s.nelems {
            println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
            throw("s.allocCount != s.nelems && freeIndex == s.nelems")
        }
        //申请新的span
        // 从 mcentral 中获取可用的span，并替换掉当前 mcache里面的span
        c.refill(spc)
        //获取申请后的新的span，并设置需要检查是否执行GC
        shouldhelpgc = true
        s = c.alloc[spc]
        // 再次到新的span里面查找合适的index
        freeIndex = s.nextFreeIndex()
    }

    if freeIndex >= s.nelems {
        throw("freeIndex is not valid")
    }
    // 计算出来内存地址，并更新span的属性
    //返回元素所在的地址
    v = gclinkptr(freeIndex*s.elemsize + s.base())
    //添加已分配的元素计数
    s.allocCount++
    if uintptr(s.allocCount) > s.nelems {
        println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
        throw("s.allocCount > s.nelems")
    }
    return
}
```

nextFree中会判断当前span是不是已经满了，如果满了就调用refill方法从 mcentral 中获取可用的span，并替换掉当前 mcache里面的span。

```go
// Gets a span that has a free object in it and assigns it
// to be the cached span for the given sizeclass. Returns this span.
func (c *mcache) refill(spc spanClass) *mspan {
	_g_ := getg()

	// 防止G被抢占
	_g_.m.locks++
	// Return the current cached span to the central lists.
	s := c.alloc[spc]

	// 确保当前的span所有元素都已分配
	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}

	// 设置span的incache属性, 除非是全局使用的空span(也就是mcache里面span指针的默认值)
	if s != &emptymspan {
		s.incache = false
	}

	// 向mcentral申请一个新的span
	// Get a new cached span from the central lists.
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// 设置新的span到mcache中
	c.alloc[spc] = s
	// 允许G被抢占
	_g_.m.locks--
	return s
}
```

Refill 根据指定的sizeclass获取对应的span，并作为 mcache的新的sizeclass对应的span。

```go
// Allocate a span to use in an MCache.
func (c *mcentral) cacheSpan() *mspan {
	// 让当前G协助一部分的sweep工作
	// Deduct credit for this span allocation and sweep if necessary.
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	deductSweepCredit(spanBytes, 0)

	// 对mcentral上锁, 因为可能会有多个M(P)同时访问
	lock(&c.lock)
	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}
	sg := mheap_.sweepgen
retry:
	// mcentral里面有两个span的链表
	// - nonempty表示确定该span最少有一个未分配的元素
	// - empty表示不确定该span最少有一个未分配的元素
	// 这里优先查找nonempty的链表
	// sweepgen每次GC都会增加2
	// - sweepgen == 全局sweepgen, 表示span已经sweep过
	// - sweepgen == 全局sweepgen-1, 表示span正在sweep
	// - sweepgen == 全局sweepgen-2, 表示span等待sweep
	var s *mspan
	for s = c.nonempty.first; s != nil; s = s.next {
		// 如果span等待sweep, 尝试原子修改sweepgen为全局sweepgen-1
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// 修改成功则把span移到empty链表, sweep它然后跳到havespan
			c.nonempty.remove(s)
			c.empty.insertBack(s)
			unlock(&c.lock)
			s.sweep(true)
			goto havespan
		}
		// 如果这个span正在被其他线程sweep, 就跳过
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// span已经sweep过
		// 因为nonempty链表中的span确定最少有一个未分配的元素, 这里可以直接使用它
		// we have a nonempty span that does not require sweeping, allocate from it
		c.nonempty.remove(s)
		c.empty.insertBack(s)
		unlock(&c.lock)
		goto havespan
	}

	// 查找empty的链表
	for s = c.empty.first; s != nil; s = s.next {
		// 如果span等待sweep, 尝试原子修改sweepgen为全局sweepgen-1
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// 把span放到empty链表的最后
			// we have an empty span that requires sweeping,
			// sweep it and see if we can free some space in it
			c.empty.remove(s)
			// swept spans are at the end of the list
			c.empty.insertBack(s)
			unlock(&c.lock)
			// 尝试sweep
			s.sweep(true)
			// sweep以后还需要检测是否有未分配的对象, 如果有则可以使用它
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			lock(&c.lock)
			// the span is still empty after sweep
			// it is already in the empty list, so just retry
			goto retry
		}
		// 如果这个span正在被其他线程sweep, 就跳过
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// 找不到有未分配对象的span
		// already swept empty span,
		// all subsequent ones must also be either swept or in process of sweeping
		break
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}
	unlock(&c.lock)

	// 找不到有未分配对象的span, 需要从mheap分配
	// 分配完成后加到empty链表中
	// Replenish central list if empty.
	s = c.grow()
	if s == nil {
		return nil
	}
	lock(&c.lock)
	c.empty.insertBack(s)
	unlock(&c.lock)

	// At this point s is a non-empty span, queued at the end of the empty list,
	// c is unlocked.
havespan:
	if trace.enabled && !traceDone {
		traceGCSweepDone()
	}
	// 统计span中未分配的元素数量, 加到mcentral.nmalloc中
	// 统计span中未分配的元素总大小, 加到memstats.heap_live中
	cap := int32((s.npages << _PageShift) / s.elemsize)
	n := cap - int32(s.allocCount)
	if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
		throw("span has no free objects")
	}
	// Assume all objects from this span will be allocated in the
	// mcache. If it gets uncached, we'll adjust this.
	atomic.Xadd64(&c.nmalloc, int64(n))
	usedBytes := uintptr(s.allocCount) * s.elemsize
	atomic.Xadd64(&memstats.heap_live, int64(spanBytes)-int64(usedBytes))
	// 跟踪处理
	if trace.enabled {
		// heap_live changed.
		traceHeapAlloc()
	}
	// 如果当前在GC中, 因为heap_live改变了, 重新调整G辅助标记工作的值
	// 详细请参考下面对revise函数的解析
	if gcBlackenEnabled != 0 {
		// heap_live changed.
		gcController.revise()
	}
	// 设置span的incache属性, 表示span正在mcache中
	s.incache = true
	// 根据freeindex更新allocCache
	freeByteBase := s.freeindex &^ (64 - 1)
	whichByte := freeByteBase / 8
	// Init alloc bits cache.
	//更新allocCache
	s.refillAllocCache(whichByte)

	// Adjust the allocCache so that s.freeindex corresponds to the low bit in
	// s.allocCache.
	s.allocCache >>= s.freeindex % 64

	return s
}

```

cacheSpan主要是从mcentral的spanset中去寻找可用的span，如果没找到那么调用grow方法从堆中申请新的内存管理单元。

获取到后更新nmalloc、allocCache等字段。

`runtime.mcentral.grow`触发扩容操作从堆中申请新的内存:

```go
func (c *mcentral) grow() *mspan {
    // 获取待分配的页数
    // 根据mcentral的类型计算需要申请的span的大小(除以8K = 有多少页)和可以保存多少个元素
    npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
    size := uintptr(class_to_size[c.spanclass.sizeclass()])
    // 获取新的span
    // 向mheap申请一个新的span, 以页(8K)为单位
    s := mheap_.alloc(npages, c.spanclass, true)
    if s == nil {
        return nil
    }

    // Use division by multiplication and shifts to quickly compute:
    // n := (npages << _PageShift) / size
    n := (npages << _PageShift) >> s.divShift * uintptr(s.divMul) >> s.divShift2
    // 初始化limit 
    s.limit = s.base() + size*n
    // 分配并初始化span的allocBits和gcmarkBits
    heapBitsForAddr(s.base()).initSpan(s)
    return s
}
```

grow里面会调用`runtime.mheap.alloc`方法获取span，这个方法在上面已经讲过了，不记得的同学可以翻一下文章上面。

到这里小对象的分配就讲解完毕了。

## 微对象分配

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    ...
    dataSize := size
    // 获取mcache，用于处理微对象和小对象的分配
    c := gomcache()
    var x unsafe.Pointer
    // 表示对象是否包含指针，true表示对象里没有指针
    noscan := typ == nil || typ.ptrdata == 0
    // maxSmallSize=32768 32k
    if size <= maxSmallSize {
        // maxTinySize= 16 bytes 
        if noscan && size < maxTinySize { 
            off := c.tinyoffset 
            // 指针内存对齐
            if size&7 == 0 {
                off = alignUp(off, 8)
            } else if size&3 == 0 {
                off = alignUp(off, 4)
            } else if size&1 == 0 {
                off = alignUp(off, 2)
            }
            // 判断指针大小相加是否超过16
            if off+size <= maxTinySize && c.tiny != 0 {
                // 获取tiny空闲内存的起始位置
                x = unsafe.Pointer(c.tiny + off)
                // 重设偏移量
                c.tinyoffset = off + size
                // 统计数量
                c.local_tinyallocs++
                mp.mallocing = 0
                releasem(mp)
                return x
            }  
            // 重新分配一个内存块
            span := c.alloc[tinySpanClass]
            v := nextFreeFast(span)
            if v == 0 {
                v, _, shouldhelpgc = c.nextFree(tinySpanClass)
            }
            x = unsafe.Pointer(v)
            //将申请的内存块全置为 0
            (*[2]uint64)(x)[0] = 0
            (*[2]uint64)(x)[1] = 0 
            // 如果申请的内存块用不完，则将剩下的给 tiny，用 tinyoffset 记录分配了多少。
            if size < c.tinyoffset || c.tiny == 0 {
                c.tiny = uintptr(x)
                c.tinyoffset = size
            }
            size = maxTinySize
        }  
        ...
    }  
    ...
    return x
}
```

在分配对象内存的时候做了一个判断， 如果该对象的大小小于16bytes，并且是不包含指针的，那么就可以看作是微对象。

在分配微对象的时候，会先判断一下tiny指向的内存块够不够用，如果tiny剩余的空间超过了size大小，那么就直接在tiny上分配内存返回；

![mchache2](https://img.luozhiyun.com/20210129154107.png)

这里我再次使用我上面的图来加以解释。首先会去mcache数组里面找到对应的span，tinySpanClass对应的span的属性如下：

```
startAddr: 824635752448,
npages: 1,
manualFreeList: 0,
freeindex: 128,
nelems: 512,
elemsize: 16,
limit: 824635760640,
allocCount: 128,
spanclass: tinySpanClass (5),
...
```

tinySpanClass对应的mspan里面只有一个page，里面的元素可以装512（nelems）个；page里面每个对象的大小是16bytes（elemsize），目前已分配了128个对象（allocCount），当然我上面的page画不了这么多，象征性的画了一下。

上面的图中还画了在page里面其中的一个object已经被使用了12bytes，还剩下4bytes没有被使用，所以会更新tinyoffset与tiny的值。

# 总结

本文先是介绍了如何对go的汇编进行调试，然后分了三个层次来讲解go中的内存分配是如何进行的。对于小于32k的对象来说，go通过无锁的方式可以直接从mcache获取到了对应的内存，如果mcache内存不够的话，先是会到mcentral中获取内存，最后才到mheap中申请内存。对于大对象（>32k）来说可以直接mheap中申请，但是对于大对象来说也是有一定优化，当大对象需要分配的页小于16页的时候会直接从pageCache中分配，否则才会从堆页中获取。

# reference:
https://juejin.cn/post/6844903795739082760
https://zhuanlan.zhihu.com/p/269621141
https://www.luozhiyun.com/archives/434
https://zhuanlan.zhihu.com/p/323915446