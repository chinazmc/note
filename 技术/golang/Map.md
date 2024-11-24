#golang 


本文是三篇系列文章的第一篇，每一篇将涵盖Go map的一部分，建议你按顺序阅读。  
map类型是Go语言的一种内置类型，通过映射表（hash table）原理来实现。在本文当中将探索map的不同部分的具体实现，包括：桶（buckets）存储键值对的结构、散列（键值对的索引）和负载因子（定义map容量应该增长的度量值）。

# 桶（buckets）

Go将键值对存储在一个桶列表中，每个桶将保存8个键值对，当map耗尽容量时，散列桶将加倍扩容。下面一张图粗略的表示了四个桶：

  

![](https://upload-images.jianshu.io/upload_images/21436181-98202ad26adb372a.png?imageMogr2/auto-orient/strip|imageView2/2/w/439/format/webp)

map的buckets列表

  

我们将在下一篇文章中介绍存储桶中的键值对是如何存放的。如果map容量增加，桶的数量将翻倍至8个、16个等等。

当一个key/value对存入map当中，将根据key的散列值分配到对于的桶里。

# hash

当key/value对赋值到map时，Go将基于key值生成一个hash值。我们以插入"foo=1"键值对为例，生成的hash值可能为15491954468309821754，将该值用于一个位操作，其掩码等于桶的数量值减1。在下图中给出了桶数为4的例子，可以得到掩码3，然后执行按位与操作：

  

![](https://upload-images.jianshu.io/upload_images/21436181-b540288e809d75af.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/650/format/webp)

value在桶中的分配

  

散列值不仅用于分配桶的值，还会有其他的操作。根据散列值的高8位，可以确认一个桶内的数组存储value的位置。如下图：

  

![](https://upload-images.jianshu.io/upload_images/21436181-7aa3fc079405c336.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

桶中的top hash table

  
因为桶内存在这个表，Go能够快速访问和比较key对于的value值。  
根据程序中map的使用，Go需要一种可扩容的机制来存放更多的key/value值。

# Map扩容

如果桶需要存储一个key/value，将为存储在内部可用的8个桶对于的槽内。如果没有桶可用的话，一个溢出桶会被创建并链接到当前桶中，并链接到当前桶上。

  

![](https://upload-images.jianshu.io/upload_images/21436181-9b404534f1469476.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

溢出桶

  

这种溢出的特性概括了桶的内部结构。然而增加溢出桶会降低map的性能。为了解决性能问题，Go将分配新的桶（当前数量的两倍）将在旧的桶和新桶之间建立连接。

Go使用它的负载系数来知道何时应该开始分配新桶和这个疏散过程。


map的内部设计向我们展示了它是如何为性能和内存管理进行优化的。下面从map的定义开始。

# Map初始化

Go提供了两种方式来初始化map和内部的桶：

-   用户显示定义map长度

```go
m := make(map[string]int, 10)
```

-   按需更新map

```go
m := make(map[string]int)
m[`foo`] = 1
```

在第二个例子当中，创建map没有指定长度，因此不会创建桶，Go将等待第一次更新来初始化map，第二行代码会创建桶。

两个例子当中，map都会根据需要扩容。第一个例子当中桶长度为10不会阻止map的扩容如果超过10个键值对，设定长度优化了map的使用，因为扩容是有成本的。

# map按需扩容的影响

Go灵活的按需扩容是有代价的，我们可以通过初始化两个map并创建100和1000个键值对，运行这些简单的基准测试来对比看看扩容的影响。前两个基准测试map的长度是会定义为100和1000，另个一例子map的长度是没有定义的，按需扩容。

```cpp
// 100 allocations
name                   time/op
LazyInitMap100Keys-8   6.67µs ± 0%
InitMap100Keys-8       3.57µs ± 0%
name                   alloc/op
LazyInitMap100Keys-8   5.59kB ± 0%
InitMap100Keys-8       2.97kB ± 0%
name                   allocs/op
LazyInitMap100Keys-8     18.0 ± 0%
InitMap100Keys-8         7.00 ± 0%
// 1000 allocations
name                   time/op
LazyInitMap1000Keys-8  77.8µs ± 0%
InitMap1000Keys-8      32.2µs ± 0%
name                   alloc/op
LazyInitMap1000Keys-8  86.8kB ± 0%
InitMap1000Keys-8      41.2kB ± 0%
name                   allocs/op
LazyInitMap1000Keys-8    66.0 ± 0%
InitMap1000Keys-8        7.00 ± 0%
```

可以看到map扩容和桶的扩展产生的代价，执行时间增加80-140%。内存分配也相应的受到影响。关于内存，Go提供了灵活设计来节省内存的消耗。

# 桶和边界对齐

如前所述每个桶只包含8个键值对，有趣的是为了节省内存Go会先保存key值然后在保存value：

  

![](https://upload-images.jianshu.io/upload_images/21436181-eae6ce21db1ad55e.png?imageMogr2/auto-orient/strip|imageView2/2/w/254/format/webp)


这个可以防止因内存的边界对齐而浪费内存。事实上，因为keys和values不一定大小相同的，将导致内存大量的边界对齐出现。下面是一个两对sting/bool的键值对例子，如果keys和values混合存储将导致很多边界对齐出现：

![](https://upload-images.jianshu.io/upload_images/21436181-9cd02e4def704537.png?imageMogr2/auto-orient/strip|imageView2/2/w/837/format/webp)

混合存储边界对齐
如果key和value单独分组的话：  

![](https://upload-images.jianshu.io/upload_images/21436181-986b31ebf5ec006d.png?imageMogr2/auto-orient/strip|imageView2/2/w/847/format/webp)

可以清楚的看到内存对齐消除了，下面看看如何访问这些值。

# value值的读取

```go
m := make(map[string]int)
v := m[`my_key`]
v, ok := m[`my_key`]
```

我们可以单独读取value值，也可以使用一个布尔值来表示键值对是否在map中存在。我们可能会想，这怎么可能。因为所有的返回结果要么显示映射到对应的变量或者用_忽略。实际上，通过汇编代码我们可以发现一些线索：

```c
(main.go:3) CALL    runtime.mapaccess1_faststr(SB)
(main.go:4) CALL    runtime.mapaccess2_faststr(SB)
```

正如我们看到的，根据读取key/value的方式，编译器会使用两个参数不同的内部函数：

```go
func mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer

func mapaccess2_faststr(t *maptype, h *hmap, ky string) (unsafe.Pointer, bool)
```

编译器的这个技巧非常有用，为我们提供了读取数据的灵活性。编译器甚至可以根据map的key值类型来使用对应的函数。在以上例子当中，我们使用一个以string作为key的map，所选择的方法是mapaccess1_faststr。后缀str表示字符串作为key进行了优化。下面看下整数key：

```go
m := make(map[int]int)

v := m[1]
v, ok := m[1]
```

汇编代码显示了所选择的方法：

```css
(main.go:3) CALL    runtime.mapaccess1_fast64(SB)
(main.go:4) CALL    runtime.mapaccess2_fast64(SB)
```

现在，编译器将使用一种专用的方法来映射，将int64作为key。如果开发人员没有定义map的容量，那么每个方法都是针对相关类型的hash比较进行了优化。

# 实现

Map的实现主要有两种方式：哈希表（hash table）和搜索树（search tree）。例如Java中的hashMap是基于哈希表实现，而C++中的Map是基于一种平衡搜索二叉树——红黑树而实现的。以下是不同实现方式的时间复杂度对比。

![](https://pic1.zhimg.com/80/v2-1fb624ee09df8b0ac30520708affbde4_720w.jpg)

可以看到，对于元素查找而言，二叉搜索树的平均和最坏效率都是O(log n)，哈希表实现的平均效率是O(1)，但最坏情况下能达到O(n)，不过如果哈希表设计优秀，最坏情况基本不会出现（所以，读者不想知道Go是如何设计的Map吗）。另外二叉搜索树返回的key是有序的，而哈希表则是乱序。

# 哈希表

由于Go中map的基于哈希表（也被称为散列表）实现，本文不探讨搜索树的map实现。以下是Go官方博客对map的说明。

_One of the most useful data structures in computer science is the hash table. Many hash table implementations exist with varying properties, but in general they offer fast lookups, adds, and deletes. Go provides a built-in map type that implements a hash table._

学习哈希表首先要理解两个概念：哈希函数和哈希冲突。

# 哈希函数

哈希函数（常被称为散列函数）是可以用于将任意大小的数据映射到固定大小值的函数，常见的包括MD5、SHA系列等。

![](https://pic4.zhimg.com/80/v2-3519c704cf3af67f8553a34b55be0d0f_720w.jpg)

一个设计优秀的哈希函数应该包含以下特性：

-   **均匀性**：一个好的哈希函数应该在其输出范围内尽可能均匀地映射，也就是说，应以大致相同的概率生成输出范围内的每个哈希值。
-   **效率高**：哈希效率要高，即使很长的输入参数也能快速计算出哈希值。
-   **可确定性**：哈希过程必须是确定性的，这意味着对于给定的输入值，它必须始终生成相同的哈希值。
-   **雪崩效应**：微小的输入值变化也会让输出值发生巨大的变化。
-   **不可逆**：从哈希函数的输出值不可反向推导出原始的数据。

# 哈希冲突

重复一遍，哈希函数是将任意大小的数据映射到固定大小值的函数。那么，我们可以预见到，即使哈希函数设计得足够优秀，几乎每个输入值都能映射为不同的哈希值。但是，当输入数据足够大，大到能超过固定大小值的组合能表达的最大数量数，冲突将不可避免！（参见抽屉原理）

![](https://pic3.zhimg.com/80/v2-10560afb7b26467a603854528a377922_720w.jpg)

抽屉原理：桌上有十个苹果，要把这十个苹果放到九个抽屉里，无论怎样放，我们会发现至少会有一个抽屉里面放不少于两个苹果。抽屉原理有时也被称为鸽巢原理。

# 如何解决哈希冲突

比较常用的Has冲突解决方案有链地址法和开放寻址法。

在讲链地址法之前，先说明两个概念。

1.  哈希桶。哈希桶（也称为槽，类似于抽屉原理中的一个抽屉）可以先简单理解为一个哈希值，所有的哈希值组成了哈希空间。
2.  装载因子。装载因子是表示哈希表中元素的填满程度。它的计算公式：装载因子=填入哈希表中的元素个数/哈希表的长度。装载因子越大，填入的元素越多，空间利用率就越高，但发生哈希冲突的几率就变大。反之，装载因子越小，填入的元素越少，冲突发生的几率减小，但空间浪费也会变得更多，而且还会提高扩容操作的次数。装载因子也是决定哈希表是否进行扩容的关键指标，在java的HashMap的中，其默认装载因子为0.75；Python的dict默认装载因子为2/3。

# 链地址法

链地址法的思想就是将映射在一个桶里的所有元素用链表串起来。

下面以一个简单的哈希函数 H(key) = key MOD 7（除数取余法）对一组元素 [50, 700, 76, 85, 92, 73, 101] 进行映射，通过图示来理解链地址法处理Hash冲突的处理逻辑。

![](https://pic1.zhimg.com/80/v2-d9983d0db110adac1e4cd5a7c17034e4_720w.jpg)

链地址法解决冲突的方式与图的邻接表存储方式在样式上很相似，发生冲突，就用单链表组织起来。

# 开放寻址法

对于链地址法而言，槽位数m与键的数目n是没有直接关系的。但是对于开放寻址法而言，所有的元素都是存储在Hash表当中的，所以无论任何时候都要保证哈希表的槽位数m大于或等于键的数据n（必要时，需要对哈希表进行动态扩容）。

开放寻址法有多种方式：线性探测法、平方探测法、随机探测法和双重哈希法。这里以线性探测法来帮助读者理解开放寻址法思想。

-   **线性探测法**

设 `Hash(key)` 表示关键字 `key` 的哈希值， 表示哈希表的槽位数（哈希表的大小）。

线性探测法则可以表示为：

如果 `Hash(x) % M` 已经有数据，则尝试 `(Hash(x) + 1) % M` ;

如果 `(Hash(x) + 1) % M` 也有数据了，则尝试 `(Hash(x) + 2) % M` ;

如果 `(Hash(x) + 2) % M` 也有数据了，则尝试 `(Hash(x) + 3) % M` ;

......

我们同样以哈希函数 `H(key) = key MOD 7` （除数取余法）对 `**[50, 700, 76, 85, 92, 73, 101]**` 进行映射，通过图示来理解线性探测法处理 Hash 碰撞。

![](https://pic2.zhimg.com/80/v2-41ffc2fe0bf167eab48c5d975ad370bd_720w.jpg)

其中，empty代表槽位为空，occupied代表槽位已被占（后续映射到该槽，则需要线性向下继续探测），而lazy delete则代表将槽位里面的数据清除，并不释放存储空间。

# 两种解决方案比较

对于开放寻址法而言，它只有数组一种数据结构就可完成存储，继承了数组的优点，对CPU缓存友好，易于序列化操作。但是它对内存的利用率不如链地址法，且发生冲突时代价更高。当数据量明确、装载因子小，适合采用开放寻址法。

链表节点可以在需要时再创建，不必像开放寻址法那样事先申请好足够内存，因此链地址法对于内存的利用率会比开方寻址法高。链地址法对装载因子的容忍度会更高，并且适合存储大对象、大数据量的哈希表。而且相较于开放寻址法，它更加灵活，支持更多的优化策略，比如可采用红黑树代替链表。但是链地址法需要额外的空间来存储指针。

值得一提的是，在Python中dict在发生哈希冲突时采用的开放寻址法，而java的HashMap采用的是链地址法。

# Go Map实现

同python与java一样，Go语言中的map是也基于哈希表实现的，它解决哈希冲突的方式是链地址法，即通过使用数组+链表的数据结构来表达map。

注意：本文后续出现的map统一代指Go中实现的map类型。

# map数据结构

map中的数据被存放于一个数组中的，数组的元素是桶（bucket），每个桶至多包含8个键值对数据。**哈希值低位（low-order bits）用于选择桶，哈希值高位（high-order bits）用于在一个独立的桶中区别出键。**哈希值高低位示意图如下

![](https://pic1.zhimg.com/80/v2-08da07f60e7ce3a5f8c315ad9da81938_720w.jpg)

本文基于go 1.15.2 darwin/amd64分析，源码位于src/runtime/map.go.

-   **map的结构体为hmap**

```go
// A header for a Go map.
type hmap struct {
    count     int // 代表哈希表中的元素个数，调用len(map)时，返回的就是该字段值。
    flags     uint8 // 状态标志，下文常量中会解释四种状态位含义。
    B         uint8  // buckets（桶）的对数log_2（哈希表元素数量最大可达到装载因子*2^B）
    noverflow uint16 // 溢出桶的大概数量。
    hash0     uint32 // 哈希种子。

    buckets    unsafe.Pointer // 指向buckets数组的指针，数组大小为2^B，如果元素个数为0，它为nil。
    oldbuckets unsafe.Pointer // 如果发生扩容，oldbuckets是指向老的buckets数组的指针，老的buckets数组大小是新的buckets的1/2。非扩容状态下，它为nil。
    nevacuate  uintptr        // 表示扩容进度，小于此地址的buckets代表已搬迁完成。

    extra *mapextra // 这个字段是为了优化GC扫描而设计的。当key和value均不包含指针，并且都可以inline时使用。extra是指向mapextra类型的指针。
```

-   **mapextra的结构体**

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
    // 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
    // 就使用 hmap的extra字段 来存储 overflow buckets，这样可以避免 GC 扫描整个 map
    // 然而 bmap.overflow 也是个指针。这时候我们只能把这些 overflow 的指针
    // 都放在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中了
    // overflow 包含的是 hmap.buckets 的 overflow 的 buckets
    // oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
    overflow    *[]*bmap
    oldoverflow *[]*bmap

    // 指向空闲的 overflow bucket 的指针
    nextOverflow *bmap
}
```

-   **bmap结构体**

```go
// A bucket for a Go map.
type bmap struct {
    // tophash包含此桶中每个键的哈希值最高字节（高8位）信息（也就是前面所述的high-order bits）。
    // 如果tophash[0] < minTopHash，tophash[0]则代表桶的搬迁（evacuation）状态。
    tophash [bucketCnt]uint8
}
```

bmap也就是bucket（桶）的内存模型图解如下（相关代码逻辑可查看src/cmd/compile/internal/gc/reflect.go中的bmap()函数）。

![](https://pic1.zhimg.com/80/v2-c3e9f9dea45043f7da1208a13fbd1af0_720w.jpg)

在以上图解示例中，该桶的第7位cell和第8位cell还未有对应键值对。需要注意的是，key和value是各自存储起来的，并非想象中的key/value/key/value...的形式。这样做虽然会让代码组织稍显复杂，但是它的好处是能让我们消除例如map[int64]int所需要的填充（padding）。此外，在8个键值对数据后面有一个overflow指针，因为桶中最多只能装8个键值对，如果有多余的键值对落到了当前桶，那么就需要再构建一个桶（称为溢出桶），通过overflow指针链接起来。

-   **重要常量标志**

```go
const (
    // 一个桶中最多能装载的键值对（key-value）的个数为8
    bucketCntBits = 3
    bucketCnt     = 1 << bucketCntBits

  // 触发扩容的装载因子为13/2=6.5
    loadFactorNum = 13
    loadFactorDen = 2

    // 键和值超过128个字节，就会被转换为指针
    maxKeySize  = 128
    maxElemSize = 128

    // 数据偏移量应该是bmap结构体的大小，它需要正确地对齐。
  // 对于amd64p32而言，这意味着：即使指针是32位的，也是64位对齐。
    dataOffset = unsafe.Offsetof(struct {
        b bmap
        v int64
    }{}.v)


  // 每个桶（如果有溢出，则包含它的overflow的链接桶）在搬迁完成状态（evacuated* states）下，要么会包含它所有的键值对，要么一个都不包含（但不包括调用evacuate()方法阶段，该方法调用只会在对map发起write时发生，在该阶段其他goroutine是无法查看该map的）。简单的说，桶里的数据要么一起搬走，要么一个都还未搬。
  // tophash除了放置正常的高8位hash值，还会存储一些特殊状态值（标志该cell的搬迁状态）。正常的tophash值，最小应该是5，以下列出的就是一些特殊状态值。
    emptyRest      = 0 // 表示cell为空，并且比它高索引位的cell或者overflows中的cell都是空的。（初始化bucket时，就是该状态）
    emptyOne       = 1 // 空的cell，cell已经被搬迁到新的bucket
    evacuatedX     = 2 // 键值对已经搬迁完毕，key在新buckets数组的前半部分
    evacuatedY     = 3 // 键值对已经搬迁完毕，key在新buckets数组的后半部分
    evacuatedEmpty = 4 // cell为空，整个bucket已经搬迁完毕
    minTopHash     = 5 // tophash的最小正常值

    // flags
    iterator     = 1 // 可能有迭代器在使用buckets
    oldIterator  = 2 // 可能有迭代器在使用oldbuckets
    hashWriting  = 4 // 有协程正在向map写人key
    sameSizeGrow = 8 // 等量扩容

    // 用于迭代器检查的bucket ID
    noCheck = 1<<(8*sys.PtrSize) - 1
)
```

综上，我们以B等于4为例，展示一个完整的map结构图。

![](https://pic4.zhimg.com/80/v2-fceb2feda6a4dd151f0730e751436b3f_720w.jpg)

# 创建map

map初始化有以下两种方式

```go
make(map[k]v)
// 指定初始化map大小为hint
make(map[k]v, hint)
```

对于不指定初始化大小，和初始化值hint<=8（bucketCnt）时，go会调用makemap_small函数（源码位置src/runtime/map.go），并直接从堆上进行分配。

```go
func makemap_small() *hmap {
    h := new(hmap)
    h.hash0 = fastrand()
    return h
}
```

当hint>8时，则调用makemap函数

```go
// 如果编译器认为map和第一个bucket可以直接创建在栈上，h和bucket可能都是非空
// 如果h != nil，那么map可以直接在h中创建
// 如果h.buckets != nil，那么h指向的bucket可以作为map的第一个bucket使用
func makemap(t *maptype, hint int, h *hmap) *hmap {
  // math.MulUintptr返回hint与t.bucket.size的乘积，并判断该乘积是否溢出。
    mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
// maxAlloc的值，根据平台系统的差异而不同，具体计算方式参照src/runtime/malloc.go
    if overflow || mem > maxAlloc {
        hint = 0
    }

// initialize Hmap
    if h == nil {
        h = new(hmap)
    }
  // 通过fastrand得到哈希种子
    h.hash0 = fastrand()

    // 根据输入的元素个数hint，找到能装下这些元素的B值
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 分配初始哈希表
  // 如果B为0，那么buckets字段后续会在mapassign方法中lazily分配
    if h.B != 0 {
        var nextOverflow *bmap
    // makeBucketArray创建一个map的底层保存buckets的数组，它最少会分配h.B^2的大小。
        h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
        if nextOverflow != nil {
    h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }

    return h
}
```

分配buckets数组的makeBucketArray函数

```go
// makeBucket为map创建用于保存buckets的数组。
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    base := bucketShift(b)
    nbuckets := base
  // 对于小的b值（小于4），即桶的数量小于16时，使用溢出桶的可能性很小。对于此情况，就避免计算开销。
    if b >= 4 {
    // 当桶的数量大于等于16个时，正常情况下就会额外创建2^(b-4)个溢出桶
        nbuckets += bucketShift(b - 4)
        sz := t.bucket.size * nbuckets
        up := roundupsize(sz)
        if up != sz {
            nbuckets = up / t.bucket.size
        }
    }

 // 这里，dirtyalloc分两种情况。如果它为nil，则会分配一个新的底层数组。如果它不为nil，则它指向的是曾经分配过的底层数组，该底层数组是由之前同样的t和b参数通过makeBucketArray分配的，如果数组不为空，需要把该数组之前的数据清空并复用。
    if dirtyalloc == nil {
        buckets = newarray(t.bucket, int(nbuckets))
    } else {
        buckets = dirtyalloc
        size := t.bucket.size * nbuckets
        if t.bucket.ptrdata != 0 {
            memclrHasPointers(buckets, size)
        } else {
            memclrNoHeapPointers(buckets, size)
        }
}

  // 即b大于等于4的情况下，会预分配一些溢出桶。
  // 为了把跟踪这些溢出桶的开销降至最低，使用了以下约定：
  // 如果预分配的溢出桶的overflow指针为nil，那么可以通过指针碰撞（bumping the pointer）获得更多可用桶。
  // （关于指针碰撞：假设内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”）
  // 对于最后一个溢出桶，需要一个安全的非nil指针指向它。
    if base != nbuckets {
        nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
        last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
        last.setoverflow(t, (*bmap)(buckets))
    }
    return buckets, nextOverflow
}
```

根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 `hmap` 中的不同字段引用而已。

# 哈希函数

在初始化go程序运行环境时（src/runtime/proc.go中的schedinit），就需要通过alginit方法完成对哈希的初始化。

```go
func schedinit() {
    lockInit(&sched.lock, lockRankSched)

    ...

    tracebackinit()
    moduledataverify()
    stackinit()
    mallocinit()
    fastrandinit() // must run before mcommoninit
    mcommoninit(_g_.m, -1)
    cpuinit()       // must run before alginit
    // 这里调用alginit()
    alginit()       // maps must not be used before this call
    modulesinit()   // provides activeModules
    typelinksinit() // uses maps, activeModules
    itabsinit()     // uses activeModules

    ...

    goargs()
    goenvs()
    parsedebugvars()
    gcinit()

  ...
}
```

对于哈希算法的选择，程序会根据当前架构判断是否支持AES，如果支持就使用AES hash，其实现代码位于src/runtime/asm_{386,amd64,arm64}.s中；若不支持，其hash算法则根据xxhash算法（[https://code.google.com/p/xxhash/](https://link.zhihu.com/?target=https%3A//code.google.com/p/xxhash/)）和cityhash算法（[https://code.google.com/p/cityhash/](https://link.zhihu.com/?target=https%3A//code.google.com/p/cityhash/)）启发而来，代码分别对应于32位（src/runtime/hash32.go）和64位机器（src/runtime/hash32.go）中，对这部分内容感兴趣的读者可以深入研究。

```go
func alginit() {
    // Install AES hash algorithms if the instructions needed are present.
    if (GOARCH == "386" || GOARCH == "amd64") &&
        cpu.X86.HasAES && // AESENC
        cpu.X86.HasSSSE3 && // PSHUFB
        cpu.X86.HasSSE41 { // PINSR{D,Q}
        initAlgAES()
        return
    }
    if GOARCH == "arm64" && cpu.ARM64.HasAES {
        initAlgAES()
        return
    }
    getRandomData((*[len(hashkey) * sys.PtrSize]byte)(unsafe.Pointer(&hashkey))[:])
    hashkey[0] |= 1 // make sure these numbers are odd
    hashkey[1] |= 1
    hashkey[2] |= 1
    hashkey[3] |= 1
}
```

上文在创建map的时候，我们可以知道map的哈希种子是通过h.hash0 = fastrand()得到的。它是在以下maptype中的hasher中被使用到，在下文内容中会看到hash值的生成。

```go
type maptype struct {
    typ    _type
    key    *_type
    elem   *_type
    bucket *_type
  // hasher的第一个参数就是指向key的指针，h.hash0 = fastrand()得到的hash0，就是hasher方法的第二个参数。
  // hasher方法返回的就是hash值。
    hasher     func(unsafe.Pointer, uintptr) uintptr
    keysize    uint8  // size of key slot
    elemsize   uint8  // size of elem slot
    bucketsize uint16 // size of bucket
    flags      uint32
}
```

# map操作

假定key经过哈希计算后得到64bit位的哈希值。如果B=5，buckets数组的长度，即桶的数量是32（2的5次方）。

例如，现要置一key于map中，该key经过哈希后，得到的哈希值如下：

![](https://pic1.zhimg.com/80/v2-086420a9bba7556ff0cd4d554cfb639c_720w.jpg)

前面我们知道，哈希值低位（low-order bits）用于选择桶，哈希值高位（high-order bits）用于在一个独立的桶中区别出键。当B等于5时，那么我们选择的哈希值低位也是5位，即01010，它的十进制值为10，代表10号桶。再用哈希值的高8位，找到此key在桶中的位置。最开始桶中还没有key，那么新加入的key和value就会被放入第一个key空位和value空位。

注意：对于高低位的选择，该操作的实质是取余，但是取余开销很大，在实际代码实现中采用的是位操作。以下是tophash的实现代码。

```go
func tophash(hash uintptr) uint8 {
    top := uint8(hash >> (sys.PtrSize*8 - 8))
    if top < minTopHash {
        top += minTopHash
    }
    return top
}
```

当两个不同的key落在了同一个桶中，这时就发生了哈希冲突。go的解决方式是链地址法：在桶中按照顺序寻到第一个空位，若有位置，则将其置于其中；否则，判断是否存在溢出桶，若有溢出桶，则去该桶的溢出桶中寻找空位，如果没有溢出桶，则添加溢出桶，并将其置溢出桶的第一个空位（非扩容的情况）。

![](https://pic2.zhimg.com/80/v2-2c06dae33e3cd7375fe6584070b6e3b5_720w.jpg)

上图中的B值为5，所以桶的数量为32。通过哈希函数计算出待插入key的哈希值，低5位哈希00110，对应于6号桶；高8位10010111，十进制为151，由于桶中前6个cell已经有正常哈希值填充了(遍历)，所以将151对应的高位哈希值放置于第7位cell，对应将key和value分别置于相应的第七个空位。

如果是查找key，那么我们会根据高位哈希值去桶中的每个cell中找，若在桶中没找到，并且overflow不为nil，那么继续去溢出桶中寻找，直至找到，如果所有的cell都找过了，还未找到，则返回key类型的默认值（例如key是int类型，则返回0）。

# 查找key

对于map的元素查找，其源码实现如下

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  // 如果开启了竞态检测 -race
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        pc := funcPC(mapaccess1)
        racereadpc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
  // 如果开启了memory sanitizer -msan
    if msanenabled && h != nil {
        msanread(key, t.key.size)
    }
  // 如果map为空或者元素个数为0，返回零值
    if h == nil || h.count == 0 {
        if t.hashMightPanic() {
            t.hasher(key, 0) // see issue 23734
        }
        return unsafe.Pointer(&zeroVal[0])
    }
  // 注意，这里是按位与操作
  // 当h.flags对应的值为hashWriting（代表有其他goroutine正在往map中写key）时，那么位计算的结果不为0，因此抛出以下错误。
  // 这也表明，go的map是非并发安全的
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
  // 不同类型的key，会使用不同的hash算法，可详见src/runtime/alg.go中typehash函数中的逻辑
    hash := t.hasher(key, uintptr(h.hash0))
    m := bucketMask(h.B)
  // 按位与操作，找到对应的bucket
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  // 如果oldbuckets不为空，那么证明map发生了扩容
  // 如果有扩容发生，老的buckets中的数据可能还未搬迁至新的buckets里
  // 所以需要先在老的buckets中找
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            m >>= 1
        }
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    // 如果在oldbuckets中tophash[0]的值，为evacuatedX、evacuatedY，evacuatedEmpty其中之一
    // 则evacuated()返回为true，代表搬迁完成。
    // 因此，只有当搬迁未完成时，才会从此oldbucket中遍历
        if !evacuated(oldb) {
            b = oldb
        }
    }
  // 取出当前key值的tophash值
    top := tophash(hash)
  // 以下是查找的核心逻辑
  // 双重循环遍历：外层循环是从桶到溢出桶遍历；内层是桶中的cell遍历
  // 跳出循环的条件有三种：第一种是已经找到key值；第二种是当前桶再无溢出桶；
  // 第三种是当前桶中有cell位的tophash值是emptyRest，这个值在前面解释过，它代表此时的桶后面的cell还未利用，所以无需再继续遍历。
bucketloop:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
      // 判断tophash值是否相等
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
      }
      // 因为在bucket中key是用连续的存储空间存储的，因此可以通过bucket地址+数据偏移量（bmap结构体的大小）+ keysize的大小，得到k的地址
      // 同理，value的地址也是相似的计算方法，只是再要加上8个keysize的内存地址
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
      // 判断key是否相等
            if t.key.equal(key, k) {
                e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                if t.indirectelem() {
                    e = *((*unsafe.Pointer)(e))
                }
                return e
            }
        }
    }
  // 所有的bucket都未找到，则返回零值
    return unsafe.Pointer(&zeroVal[0])
}
```

以下是mapaccess1的查找过程图解

![](https://pic1.zhimg.com/80/v2-8547e5fdbc7f51e6d5aec5d51ed658b0_720w.jpg)

map的元素查找，对应go代码有两种形式

```go
// 形式一
    v := m[k]
    // 形式二
    v, ok := m[k]
```

形式一的代码实现，就是上述的mapaccess1方法。此外，在源码中还有个mapaccess2方法，它的函数签名如下。

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {}
```

与mapaccess1相比，mapaccess2多了一个bool类型的返回值，它代表的是是否在map中找到了对应的key。因为和mapaccess1基本一致，所以详细代码就不再贴出。

同时，源码中还有mapaccessK方法，它的函数签名如下。

```go
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {}
```

与mapaccess1相比，mapaccessK同时返回了key和value，其代码逻辑也一致。

# 赋值key

对于写入key的逻辑，其源码实现如下

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  // 如果h是空指针，赋值会引起panic
  // 例如以下语句
  // var m map[string]int
    // m["k"] = 1
    if h == nil {
        panic(plainError("assignment to entry in nil map"))
    }
  // 如果开启了竞态检测 -race
    if raceenabled {
        callerpc := getcallerpc()
        pc := funcPC(mapassign)
        racewritepc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
  // 如果开启了memory sanitizer -msan
    if msanenabled {
        msanread(key, t.key.size)
    }
  // 有其他goroutine正在往map中写key，会抛出以下错误
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
  // 通过key和哈希种子，算出对应哈希值
    hash := t.hasher(key, uintptr(h.hash0))

  // 将flags的值与hashWriting做按位或运算
  // 因为在当前goroutine可能还未完成key的写入，再次调用t.hasher会发生panic。
    h.flags ^= hashWriting

    if h.buckets == nil {
        h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
}

again:
  // bucketMask返回值是2的B次方减1
  // 因此，通过hash值与bucketMask返回值做按位与操作，返回的在buckets数组中的第几号桶
    bucket := hash & bucketMask(h.B)
  // 如果map正在搬迁（即h.oldbuckets != nil）中,则先进行搬迁工作。
    if h.growing() {
        growWork(t, h, bucket)
    }
  // 计算出上面求出的第几号bucket的内存位置
  // post = start + bucketNumber * bucketsize
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    top := tophash(hash)

    var inserti *uint8
    var insertk unsafe.Pointer
    var elem unsafe.Pointer
bucketloop:
    for {
    // 遍历桶中的8个cell
        for i := uintptr(0); i < bucketCnt; i++ {
      // 这里分两种情况，第一种情况是cell位的tophash值和当前tophash值不相等
      // 在 b.tophash[i] != top 的情况下
      // 理论上有可能会是一个空槽位
      // 一般情况下 map 的槽位分布是这样的，e 表示 empty:
      // [h0][h1][h2][h3][h4][e][e][e]
      // 但在执行过 delete 操作时，可能会变成这样:
      // [h0][h1][e][e][h5][e][e][e]
      // 所以如果再插入的话，会尽量往前面的位置插
      // [h0][h1][e][e][h5][e][e][e]
      //          ^
      //          ^
      //       这个位置
      // 所以在循环的时候还要顺便把前面的空位置先记下来
      // 因为有可能在后面会找到相等的key，也可能找不到相等的key
            if b.tophash[i] != top {
        // 如果cell位为空，那么就可以在对应位置进行插入
                if isEmpty(b.tophash[i]) && inserti == nil {
                    inserti = &b.tophash[i]
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
                    elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                }
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
      // 第二种情况是cell位的tophash值和当前的tophash值相等
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
      // 注意，即使当前cell位的tophash值相等，不一定它对应的key也是相等的，所以还要做一个key值判断
            if !t.key.equal(key, k) {
                continue
            }
            // 如果已经有该key了，就更新它
            if t.needkeyupdate() {
                typedmemmove(t.key, k, key)
            }
      // 这里获取到了要插入key对应的value的内存地址
      // pos = start + dataOffset + 8*keysize + i*elemsize
            elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
      // 如果顺利到这，就直接跳到done的结束逻辑中去
            goto done
        }
    // 如果桶中的8个cell遍历完，还未找到对应的空cell或覆盖cell，那么就进入它的溢出桶中去遍历
        ovf := b.overflow(t)
    // 如果连溢出桶中都没有找到合适的cell，跳出循环。
        if ovf == nil {
            break
        }
        b = ovf
    }

    // 在已有的桶和溢出桶中都未找到合适的cell供key写入，那么有可能会触发以下两种情况
  // 情况一：
  // 判断当前map的装载因子是否达到设定的6.5阈值，或者当前map的溢出桶数量是否过多。如果存在这两种情况之一，则进行扩容操作。
  // hashGrow()实际并未完成扩容，对哈希表数据的搬迁（复制）操作是通过growWork()来完成的。
  // 重新跳入again逻辑，在进行完growWork()操作后，再次遍历新的桶。
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again // Growing the table invalidates everything, so try again
    }

  // 情况二：
// 在不满足情况一的条件下，会为当前桶再新建溢出桶，并将tophash，key插入到新建溢出桶的对应内存的0号位置
    if inserti == nil {
        // all current buckets are full, allocate a new one.
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        elem = add(insertk, bucketCnt*uintptr(t.keysize))
    }

  // 在插入位置存入新的key和value
    if t.indirectkey() {
        kmem := newobject(t.key)
        *(*unsafe.Pointer)(insertk) = kmem
        insertk = kmem
    }
    if t.indirectelem() {
        vmem := newobject(t.elem)
        *(*unsafe.Pointer)(elem) = vmem
    }
    typedmemmove(t.key, insertk, key)
    *inserti = top
  // map中的key数量+1
    h.count++

done:
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    h.flags &^= hashWriting
    if t.indirectelem() {
        elem = *((*unsafe.Pointer)(elem))
    }
    return elem
}
```

通过对mapassign的代码分析之后，发现该函数并没有将插入key对应的value写入对应的内存，而是返回了value应该插入的内存地址。为了弄清楚value写入内存的操作是发生在什么时候，分析如下map.go代码。

```go
package main

func main() {
    m := make(map[int]int)
    for i := 0; i < 100; i++ {
        m[i] = 666
    }
}
```

m[i] = 666对应的汇编代码

```bash
$ go tool compile -S map.go
...
        0x0098 00152 (map.go:6) LEAQ    type.map[int]int(SB), CX
        0x009f 00159 (map.go:6) MOVQ    CX, (SP)
        0x00a3 00163 (map.go:6) LEAQ    ""..autotmp_2+184(SP), DX
        0x00ab 00171 (map.go:6) MOVQ    DX, 8(SP)
        0x00b0 00176 (map.go:6) MOVQ    AX, 16(SP)
        0x00b5 00181 (map.go:6) CALL    runtime.mapassign_fast64(SB) // 调用函数runtime.mapassign_fast64，该函数实质就是mapassign（上文示例源代码是该mapassign系列的通用逻辑）
        0x00ba 00186 (map.go:6) MOVQ    24(SP), AX 24(SP), AX // 返回值，即 value 应该存放的内存地址
        0x00bf 00191 (map.go:6) MOVQ    $666, (AX) // 把 666 放入该地址中
...
```

赋值的最后一步实际上是编译器额外生成的汇编指令来完成的，可见靠 runtime 有些工作是没有做完的。所以，在go中，编译器和 runtime 配合，才能完成一些复杂的工作。同时说明，在平时学习go的源代码实现时，必要时还需要看一些汇编代码。

# 删除key

理解了赋值key的逻辑，删除key的逻辑就比较简单了。本文就不再讨论该部分内容了，读者感兴趣可以自行查看src/runtime/map.go的mapdelete方法逻辑。

# 遍历map

**结论：迭代 map 的结果是无序的**

```go
m := make(map[int]int)
    for i := 0; i < 10; i++ {
        m[i] = i
    }
    for k, v := range m {
        fmt.Println(k, v)
    }
```

运行以上代码，我们会发现每次输出顺序都是不同的。

map遍历的过程，是按序遍历bucket，同时按需遍历bucket中和其overflow bucket中的cell。但是map在扩容后，会发生key的搬迁，这造成原来落在一个bucket中的key，搬迁后，有可能会落到其他bucket中了，从这个角度看，遍历map的结果就不可能是按照原来的顺序了（详见下文的map扩容内容）。

但其实，go为了保证遍历map的结果是无序的，做了以下事情：map在遍历时，并不是从固定的0号bucket开始遍历的，每次遍历，都会从一个**随机值序号的bucket**，再从其中**随机的cell**开始遍历。然后再按照桶序遍历下去，直到回到起始桶结束。

![](https://pic3.zhimg.com/80/v2-f418d848668c62eac039507d4f460ec2_720w.jpg)

上图的例子，是遍历一个处于未扩容状态的map。如果map正处于扩容状态时，需要先判断当前遍历bucket是否已经完成搬迁，如果数据还在老的bucket，那么就去老bucket中拿数据。

注意：在下文中会讲解到增量扩容和等量扩容。当发生了增量扩容时，一个老的bucket数据可能会分裂到两个不同的bucket中去，那么此时，如果需要从老的bucket中遍历数据，例如1号，则不能将老1号bucket中的数据全部取出，仅仅只能取出老 1 号 bucket 中那些在裂变之后，分配到新 1 号 bucket 中的那些 key（这个内容，请读者看完下文map扩容的讲解之后再回头理解）。

鉴于篇幅原因，本文不再对map遍历的详细源码进行注释贴出。读者可自行查看源码src/runtime/map.go的`mapiterinit()`和`mapiternext()`方法逻辑。

这里注释一下`mapiterinit()`中随机保证的关键代码

```go
// 生成随机数
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
   r += uintptr(fastrand()) << 31
}
// 决定了从哪个随机的bucket开始
it.startBucket = r & bucketMask(h.B)
// 决定了每个bucket中随机的cell的位置
it.offset = uint8(r >> h.B & (bucketCnt - 1))
```

# map扩容

在文中讲解装载因子时，我们提到装载因子是决定哈希表是否进行扩容的关键指标。在go的map扩容中，除了装载因子会决定是否需要扩容，溢出桶的数量也是扩容的另一关键指标。

为了保证访问效率，当map将要添加、修改或删除key时，都会检查是否需要扩容，扩容实际上是以空间换时间的手段。在之前源码mapassign中，其实已经注释map扩容条件，主要是两点:

1.  判断已经达到装载因子的临界点，即元素个数 >= 桶（bucket）总数 * 6.5，这时候说明大部分的桶可能都快满了（即平均每个桶存储的键值对达到6.5个），如果插入新元素，有大概率需要挂在溢出桶（overflow bucket）上。

```go
func overLoadFactor(count int, B uint8) bool {
    return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

1.  判断溢出桶是否太多，当桶总数 < 2 ^ 15 时，如果溢出桶总数 >= 桶总数，则认为溢出桶过多。当桶总数 >= 2 ^ 15 时，直接与 2 ^ 15 比较，当溢出桶总数 >= 2 ^ 15 时，即认为溢出桶太多了。

```go
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
    if B > 15 {
        B = 15
    }
    return noverflow >= uint16(1)<<(B&15)
}
```

对于第2点，其实算是对第 1 点的补充。因为在装载因子比较小的情况下，有可能 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是桶数量多（真实分配的桶数量多，包括大量的溢出桶）。

在某些场景下，比如不断的增删，这样会造成overflow的bucket数量增多，但负载因子又不高，未达不到第 1 点的临界值，就不能触发扩容来缓解这种情况。这样会造成桶的使用率不高，值存储得比较稀疏，查找插入效率会变得非常低，因此有了第 2 点判断指标。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

![](https://pic3.zhimg.com/80/v2-33636572ffb353268b2162940514a1ce_720w.jpg)

如上图所示，由于对map的不断增删，以0号bucket为例，该桶链中就造成了大量的稀疏桶。

两种情况官方采用了不同的解决方案

-   针对 1，将 B + 1，新建一个buckets数组，新的buckets大小是原来的2倍，然后旧buckets数据搬迁到新的buckets。该方法我们称之为**增量扩容**。
-   针对 2，并不扩大容量，buckets数量维持不变，重新做一遍类似增量扩容的搬迁动作，把松散的键值对重新排列一次，以使bucket的使用率更高，进而保证更快的存取。该方法我们称之为**等量扩容**。

对于 2 的解决方案，其实存在一个极端的情况：如果插入 map 的 key 哈希都一样，那么它们就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 `O(n)`。但 Go 的每一个 map 都会在初始化阶段的 makemap时定一个随机的哈希种子，所以要构造这种冲突是没那么容易的。

在源码中，和扩容相关的主要是`hashGrow()`函数与`growWork()`函数。`hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在`mapassign()` 和 `mapdelete()` 函数中。也就是插入（包括修改）、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。它们会先检查 oldbuckets 是否搬迁完毕（检查 oldbuckets 是否为 nil），再决定是否进行搬迁工作。

`hashGrow()`函数

```go
func hashGrow(t *maptype, h *hmap) {
  // 如果达到条件 1，那么将B值加1，相当于是原来的2倍
  // 否则对应条件 2，进行等量扩容，所以 B 不变
    bigger := uint8(1)
    if !overLoadFactor(h.count+1, h.B) {
        bigger = 0
        h.flags |= sameSizeGrow
    }
  // 记录老的buckets
    oldbuckets := h.buckets
  // 申请新的buckets空间
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
  // 注意&^ 运算符，这块代码的逻辑是转移标志位
    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
    // 提交grow (atomic wrt gc)
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
  // 搬迁进度为0
    h.nevacuate = 0
  // overflow buckets 数为0
    h.noverflow = 0

  // 如果发现hmap是通过extra字段 来存储 overflow buckets时
    if h.extra != nil && h.extra.overflow != nil {
        if h.extra.oldoverflow != nil {
            throw("oldoverflow is not nil")
        }
        h.extra.oldoverflow = h.extra.overflow
        h.extra.overflow = nil
    }
    if nextOverflow != nil {
        if h.extra == nil {
            h.extra = new(mapextra)
        }
        h.extra.nextOverflow = nextOverflow
    }
}
```

`growWork()`函数

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
  // 为了确认搬迁的 bucket 是我们正在使用的 bucket
  // 即如果当前key映射到老的bucket1，那么就搬迁该bucket1。
    evacuate(t, h, bucket&h.oldbucketmask())

    // 如果还未完成扩容工作，则再搬迁一个bucket。
    if h.growing() {
        evacuate(t, h, h.nevacuate)
    }
}
```

从`growWork()`函数可以知道，搬迁的核心逻辑是`evacuate()`函数。这里读者可以思考一个问题：为什么每次至多搬迁2个bucket？这其实是一种性能考量，如果map存储了数以亿计的key-value，一次性搬迁将会造成比较大的延时，因此才采用逐步搬迁策略。

在讲解该逻辑之前，需要读者先理解以下两个知识点。

-   知识点1：bucket序号的变化

前面讲到，增量扩容（条件1）和等量扩容（条件2）都需要进行bucket的搬迁工作。对于等量扩容而言，由于buckets的数量不变，因此可以按照序号来搬迁。例如老的的0号bucket，仍然搬至新的0号bucket中。

![](https://pic4.zhimg.com/80/v2-9d60c8b3096e2dab7a87c64bf349097b_720w.jpg)

但是，对于增量扩容而言，就会有所不同。例如原来的B=5，那么增量扩容时，B就会变成6。那么决定key值落入哪个bucket的低位哈希值就会发生变化（从取5位变为取6位），取新的低位hash值得过程称为rehash。

![](https://pic4.zhimg.com/80/v2-15b4f455a970e9d3406cd47765bcca8b_720w.jpg)

因此，在增量扩容中，某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于低 hash 值第倒数第B+1位是 0 还是 1。

![](https://pic2.zhimg.com/80/v2-c50f4e8f7e2769313b4f2d680fa91925_720w.jpg)

如上图所示，当原始的B = 3时，旧buckets数组长度为8，在编号为2的bucket中，其2号cell和5号cell，它们的低3位哈希值相同（不相同的话，也就不会落在同一个桶中了），但是它们的低4位分别是0010、1010。当发生了增量扩容，2号就会被搬迁到新buckets数组的2号bucket中去，5号被搬迁到新buckets数组的10号bucket中去，它们的桶号差距是2的3次方。

-   知识点2：确定搬迁区间

在源码中，有bucket x 和bucket y的概念，其实就是增量扩容到原来的 2 倍，桶的数量是原来的 2 倍，前一半桶被称为bucket x，后一半桶被称为bucket y。一个 bucket 中的 key 可能会分裂到两个桶中去，分别位于bucket x的桶，或bucket y中的桶。所以在搬迁一个 cell 之前，需要知道这个 cell 中的 key 是落到哪个区间（而对于同一个桶而言，搬迁到bucket x和bucket y桶序号的差别是老的buckets大小，即2^old_B）。

这里留一个问题：为什么确定key落在哪个区间很重要？

![](https://pic1.zhimg.com/80/v2-197a831e09d60dd04566163be0458b5c_720w.jpg)

确定了要搬迁到的目标 bucket 后，搬迁操作就比较好进行了。将源 key/value 值 copy 到目的地相应的位置。设置 key 在原始 buckets 的 tophash 为 `evacuatedX` 或是 `evacuatedY`，表示已经搬迁到了新 map 的bucket x或是bucket y，新 map 的 tophash 则正常取 key 哈希值的高 8 位。

下面正式解读搬迁核心代码`evacuate()`函数。

`evacuate()`函数

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  // 首先定位老的bucket的地址
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
  // newbit代表扩容之前老的bucket个数
    newbit := h.noldbuckets()
  // 判断该bucket是否已经被搬迁
    if !evacuated(b) {
    // 官方TODO，后续版本也许会实现
        // TODO: reuse overflow buckets instead of using new ones, if there
        // is no iterator using the old buckets.  (If !oldIterator.)

    // xy 包含了高低区间的搬迁目的地内存信息
    // x.b 是对应的搬迁目的桶
    // x.k 是指向对应目的桶中存储当前key的内存地址
    // x.e 是指向对应目的桶中存储当前value的内存地址
        var xy [2]evacDst
        x := &xy[0]
        x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        x.k = add(unsafe.Pointer(x.b), dataOffset)
        x.e = add(x.k, bucketCnt*uintptr(t.keysize))

    // 只有当增量扩容时才计算bucket y的相关信息（和后续计算useY相呼应）
        if !h.sameSizeGrow() {
            y := &xy[1]
            y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
            y.k = add(unsafe.Pointer(y.b), dataOffset)
            y.e = add(y.k, bucketCnt*uintptr(t.keysize))
        }

    // evacuate 函数每次只完成一个 bucket 的搬迁工作，因此要遍历完此 bucket 的所有的 cell，将有值的 cell copy 到新的地方。
    // bucket 还会链接 overflow bucket，它们同样需要搬迁。
    // 因此同样会有 2 层循环，外层遍历 bucket 和 overflow bucket，内层遍历 bucket 的所有 cell。

    // 遍历当前桶bucket和其之后的溢出桶overflow bucket
    // 注意：初始的b是待搬迁的老bucket
        for ; b != nil; b = b.overflow(t) {
            k := add(unsafe.Pointer(b), dataOffset)
            e := add(k, bucketCnt*uintptr(t.keysize))
      // 遍历桶中的cell，i，k，e分别用于对应tophash，key和value
            for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
                top := b.tophash[i]
        // 如果当前cell的tophash值是emptyOne或者emptyRest，则代表此cell没有key。并将其标记为evacuatedEmpty，表示它“已经被搬迁”。
                if isEmpty(top) {
                    b.tophash[i] = evacuatedEmpty
                    continue
                }
        // 正常不会出现这种情况
        // 未被搬迁的 cell 只可能是emptyOne、emptyRest或是正常的 top hash（大于等于 minTopHash）
                if top < minTopHash {
                    throw("bad map state")
                }
                k2 := k
        // 如果 key 是指针，则解引用
                if t.indirectkey() {
                    k2 = *((*unsafe.Pointer)(k2))
                }
                var useY uint8
        // 如果是增量扩容
                if !h.sameSizeGrow() {
          // 计算哈希值，判断当前key和vale是要被搬迁到bucket x还是bucket y
                    hash := t.hasher(k2, uintptr(h.hash0))
                    if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
            // 有一个特殊情况：有一种 key，每次对它计算 hash，得到的结果都不一样。
            // 这个 key 就是 math.NaN() 的结果，它的含义是 not a number，类型是 float64。
            // 当它作为 map 的 key时，会遇到一个问题：再次计算它的哈希值和它当初插入 map 时的计算出来的哈希值不一样！
            // 这个 key 是永远不会被 Get 操作获取的！当使用 m[math.NaN()] 语句的时候，是查不出来结果的。
            // 这个 key 只有在遍历整个 map 的时候，才能被找到。
            // 并且，可以向一个 map 插入多个数量的 math.NaN() 作为 key，它们并不会被互相覆盖。
            // 当搬迁碰到 math.NaN() 的 key 时，只通过 tophash 的最低位决定分配到 X part 还是 Y part（如果扩容后是原来 buckets 数量的 2 倍）。如果 tophash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。
                        useY = top & 1
                        top = tophash(hash)
          // 对于正常key，进入以下else逻辑  
                    } else {
                        if hash&newbit != 0 {
                            useY = 1
                        }
                    }
                }

                if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
                    throw("bad evacuatedN")
                }

        // evacuatedX + 1 == evacuatedY
                b.tophash[i] = evacuatedX + useY
        // useY要么为0，要么为1。这里就是选取在bucket x的起始内存位置，或者选择在bucket y的起始内存位置（只有增量同步才会有这个选择可能）。
                dst := &xy[useY]

        // 如果目的地的桶已经装满了（8个cell），那么需要新建一个溢出桶，继续搬迁到溢出桶上去。
                if dst.i == bucketCnt {
                    dst.b = h.newoverflow(t, dst.b)
                    dst.i = 0
                    dst.k = add(unsafe.Pointer(dst.b), dataOffset)
                    dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
                }
                dst.b.tophash[dst.i&(bucketCnt-1)] = top
        // 如果待搬迁的key是指针，则复制指针过去
                if t.indirectkey() {
                    *(*unsafe.Pointer)(dst.k) = k2 // copy pointer
        // 如果待搬迁的key是值，则复制值过去  
                } else {
                    typedmemmove(t.key, dst.k, k) // copy elem
                }
        // value和key同理
                if t.indirectelem() {
                    *(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
                } else {
                    typedmemmove(t.elem, dst.e, e)
                }
        // 将当前搬迁目的桶的记录key/value的索引值（也可以理解为cell的索引值）加一
                dst.i++
        // 由于桶的内存布局中在最后还有overflow的指针，多以这里不用担心更新有可能会超出key和value数组的指针地址。
                dst.k = add(dst.k, uintptr(t.keysize))
                dst.e = add(dst.e, uintptr(t.elemsize))
            }
        }
    // 如果没有协程在使用老的桶，就对老的桶进行清理，用于帮助gc
        if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
            b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
      // 只清除bucket 的 key,value 部分，保留 top hash 部分，指示搬迁状态
            ptr := add(b, dataOffset)
            n := uintptr(t.bucketsize) - dataOffset
            memclrHasPointers(ptr, n)
        }
    }

  // 用于更新搬迁进度
    if oldbucket == h.nevacuate {
        advanceEvacuationMark(h, t, newbit)
    }
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
  // 搬迁桶的进度加一
    h.nevacuate++
  // 实验表明，1024至少会比newbit高出一个数量级（newbit代表扩容之前老的bucket个数）。所以，用当前进度加上1024用于确保O(1)行为。
    stop := h.nevacuate + 1024
    if stop > newbit {
        stop = newbit
    }
  // 计算已经搬迁完的桶数
    for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
        h.nevacuate++
    }
  // 如果h.nevacuate == newbit，则代表所有的桶都已经搬迁完毕
    if h.nevacuate == newbit {
    // 搬迁完毕，所以指向老的buckets的指针置为nil
        h.oldbuckets = nil
    // 在讲解hmap的结构中，有过说明。如果key和value均不包含指针，则都可以inline。
    // 那么保存它们的buckets数组其实是挂在hmap.extra中的。所以，这种情况下，其实我们是搬迁的extra的buckets数组。
    // 因此，在这种情况下，需要在搬迁完毕后，将hmap.extra.oldoverflow指针置为nil。
        if h.extra != nil {
            h.extra.oldoverflow = nil
        }
    // 最后，清除正在扩容的标志位，扩容完毕。
        h.flags &^= sameSizeGrow
    }
}
```

代码比较长，但是文中注释已经比较清晰了，如果对map的扩容还不清楚，可以参见以下图解。

![](https://pic2.zhimg.com/80/v2-3df0b0907c1a309a58c7d40d0fc41a59_720w.jpg)

针对上图的map，其B为3，所以原始buckets数组为8。当map元素数变多，加载因子超过6.5，所以引起了增量扩容。

以3号bucket为例，可以看到，由于B值加1，所以在新选取桶时，需要取低4位哈希值，这样就会造成cell会被搬迁到新buckets数组中不同的桶（3号或11号桶）中去。注意，在一个桶中，搬迁cell的工作是有序的：它们是依序填进对应新桶的cell中去的。

当然，实际情况中3号桶很可能还有溢出桶，在这里为了简化绘图，假设3号桶没有溢出桶，如果有溢出桶，则相应地添加到新的3号桶和11号桶中即可，如果对应的3号和11号桶均装满，则给新的桶添加溢出桶来装载。

![](https://pic2.zhimg.com/80/v2-47aafdcac1db08845444e14f55adb3a5_720w.jpg)

对于上图的map，其B也为3。假设整个map中的overflow过多，触发了等量扩容。注意，等量扩容时，新的buckets数组大小和旧buckets数组是一样的。

以6号桶为例，它有一个bucket和3个overflow buckets，但是我们能够发现桶里的数据非常稀疏，等量扩容的目的就是为了把松散的键值对重新排列一次，以使bucket的使用率更高，进而保证更快的存取。搬迁完毕后，新的6号桶中只有一个基础bucket，暂时并不需要溢出桶。这样，和原6号桶相比，数据变得紧密，使后续的数据存取变快。

最后回答一下上文中留下的问题：为什么确定key落在哪个区间很重要？因为对于增量扩容而言，原本一个bucket中的key会被分裂到两个bucket中去，它们分别处于bucket x和bucket y中，但是它们之间存在关系 bucket x + 2^B = bucket y （其中，B是老bucket对应的B值）。假设key所在的老bucket序号为n，那么如果key落在新的bucket x，则它应该置入 bucket x起始位置 + n*bucket 的内存中去；如果key落在新的bucket y，则它应该置入 bucket y起始位置 + n*bucket的内存中去。因此，确定key落在哪个区间，这样就很方便进行内存地址计算，快速找到key应该插入的内存地址。

# 总结

Go语言的map，底层是哈希表实现的，通过链地址法解决哈希冲突，它依赖的核心数据结构是数组加链表。

map中定义了2的B次方个桶，每个桶中能够容纳8个key。根据key的不同哈希值，将其散落到不同的桶中。哈希值的低位（哈希值的后B个bit位）决定桶序号，高位（哈希值的前8个bit位）标识同一个桶中的不同 key。

当向桶中添加了很多 key，造成元素过多，超过了装载因子所设定的程度，或者多次增删操作，造成溢出桶过多，均会触发扩容。

扩容分为增量扩容和等量扩容。增量扩容，会增加桶的个数（增加一倍），把原来一个桶中的 keys 被重新分配到两个桶中。等量扩容，不会更改桶的个数，只是会将桶中的数据变得紧凑。不管是增量扩容还是等量扩容，都需要创建新的桶数组，并不是原地操作的。

扩容过程是渐进性的，主要是防止一次扩容需要搬迁的 key 数量过多，引发性能问题。触发扩容的时机是增加了新元素， 桶搬迁的时机则发生在赋值、删除期间，每次最多搬迁两个 桶。查找、赋值、删除的一个很核心的内容是如何定位到 key 所在的位置，需要重点理解。一旦理解，关于 map 的源码就可以看懂了。

# **使用建议**

从map设计可以知道，它并不是一个并发安全的数据结构。同时对map进行读写时，程序很容易出错。因此，要想在并发情况下使用map，请加上锁（sync.Mutex或者sync.RwMutex）。其实，Go标准库中已经为我们实现了并发安全的map——sync.Map，我之前写过文章对它的实现进行讲解，详情可以查看公号：Golang技术分享——《深入理解sync.Map》一文。

遍历map的结果是无序的，在使用中，应该注意到该点。

通过map的结构体可以知道，它其实是通过指针指向底层buckets数组。所以和slice一样，尽管go函数都是值传递，但是，当map作为参数被函数调用时，在函数内部对map的操作同样会影响到外部的map。

另外，有个特殊的key值math.NaN，它每次生成的哈希值是不一样的，这会造成m[math.NaN]是拿不到值的，而且多次对它赋值，会让map中存在多个math.NaN的key。不过这个基本用不到，知道有这个特殊情况就可以了。

Go中的map在底层是用哈希表实现的，你可以在 $GOROOT/src/pkg/runtime/hashmap.goc 找到它的实现。

# map设计中的性能优化

读完map源代码发现作者还是做了很多设计上的选择的。本人水平有限，谈不上优劣的点评，这里只是拿出来与读者分享。

HMap中是Bucket的数组，而不是Bucket指针的数组。好的方面是可以一次分配较大内存，减少了分配次数，避免多次调用mallocgc。但相应的缺点，其一是可扩展哈希的算法并没有发生作用，扩容时会造成对整个数组的值拷贝(如果实现上用Bucket指针的数组就是指针拷贝了，代价小很多)。其二是首个bucket与后面产生了不一致性。这个会使删除逻辑变得复杂一点。比如删除后面的溢出链可以直接删除，而对于首个bucket，要等到evalucated完毕后，整个oldbucket删除时进行。

没有重用设freelist重用删除的结点。作者把这个加了一个TODO的注释，不过想了一下觉得这个做的意义不大。因为一方面，bucket大小并不一致，重用比较麻烦。另一方面，下层存储已经做过内存池的实现了，所以这里不做重用也会在内存分配那一层被重用的，

bucket直接key/value和间接key/value优化。这个优化做得蛮好的。注意看代码会发现，如果key或value小于128字节，则它们的值是直接使用的bucket作为存储的。否则bucket中存储的是指向实际key/value数据的指针，

bucket存8个key/value对。查找时进行顺序比较。第一次发现高位居然不是用作offset，而是用于加快比较的。定位到bucket之后，居然是一个顺序比较的查找过程。后面仔细想了想，觉得还行。由于bucket只有8个，顺序比较下来也不算过分。仍然是O(1)只不过前面系数大一点点罢了。相当于hash到一个小范围之后，在这个小范围内顺序查找。

插入删除的优化。前面已经提过了，插入只要找到相同的key或者第一个空位，bucket中如果存在一个以上的相同key，前面覆盖后面的(只是如果，实际上不会发生)。而删除就需要遍历完所有bucket溢出链了。这样map的设计就是为插入优化的。考虑到一般的应用场景，这个应该算是很合理的。

作者还列了另个2个TODO：将多个几乎要empty的bucket合并；如果table中元素很少，考虑shrink table。(毕竟现在的实现只是单纯的grow)。

# Reference
https://www.jianshu.com/p/e5de7b978946
https://zhuanlan.zhihu.com/p/273666774
https://tiancaiamao.gitbooks.io/go-internals/content/zh/02.3.html
