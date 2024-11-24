![](https://zhuanlan.zhihu.com/p/181498475)  

# LSM树详解

LSM树(Log-Structured-Merge-Tree)的名字往往会给初识者一个错误的印象，事实上，LSM树并不像B+树、红黑树一样是一颗严格的树状数据结构，它其实是一种存储结构，目前HBase,LevelDB,RocksDB这些NoSQL存储都是采用的LSM树。

LSM树的核心特点是利用顺序写来提高写性能，但因为分层(此处分层是指的分为内存和文件两部分)的设计会稍微降低读性能，但是通过牺牲小部分读性能换来高性能写，使得LSM树成为非常流行的存储结构。

## **1、LSM树的核心思想**

![](https://pic2.zhimg.com/80/v2-37576525d52091fd713bb13556c92861_720w.webp)

如上图所示，LSM树有以下三个重要组成部分：

_**1) MemTable**_

MemTable是在**_内存_**中的数据结构，用于保存最近更新的数据，会按照Key有序地组织这些数据，LSM树对于具体如何组织有序地组织数据并没有明确的数据结构定义，例如Hbase使跳跃表来保证内存中key的有序。

因为数据暂时保存在内存中，内存并不是可靠存储，如果断电会丢失数据，因此通常会通过WAL(Write-ahead logging，预写式日志)的方式来保证数据的可靠性。

_**2) Immutable MemTable**_

当 MemTable达到一定大小后，会转化成Immutable MemTable。Immutable MemTable是将转MemTable变为SSTable的一种中间状态。写操作由新的MemTable处理，在转存过程中不阻塞数据更新操作。

_**3) SSTable(Sorted String Table)**_

**_有序键值对_**集合，是LSM树组在**_磁盘_**中的数据结构。为了加快SSTable的读取，可以通过建立key的索引以及布隆过滤器来加快key的查找。

![](https://pic3.zhimg.com/80/v2-9eeda5082f56b1df20fa555d36b0e0ae_720w.webp)

这里需要关注一个重点，LSM树(Log-Structured-Merge-Tree)正如它的名字一样，LSM树会将所有的数据插入、修改、删除等操作记录(注意是操作记录)保存在内存之中，当此类操作达到一定的数据量后，再批量地顺序写入到磁盘当中。这与B+树不同，B+树数据的更新会直接在原数据所在处修改对应的值，但是LSM数的数据更新是日志式的，当一条数据更新是直接append一条更新记录完成的。这样设计的目的就是为了顺序写，不断地将Immutable MemTable flush到持久化存储即可，而不用去修改之前的SSTable中的key，保证了顺序写。

因此当MemTable达到一定大小flush到持久化存储变成SSTable后，在不同的SSTable中，可能存在相同Key的记录，当然最新的那条记录才是准确的。这样设计的虽然大大提高了写性能，但同时也会带来一些问题：

> 1）冗余存储，对于某个key，实际上除了最新的那条记录外，其他的记录都是冗余无用的，但是仍然占用了存储空间。因此需要进行Compact操作(合并多个SSTable)来清除冗余的记录。  
> 2）读取时需要从最新的倒着查询，直到找到某个key的记录。最坏情况需要查询完所有的SSTable，这里可以通过前面提到的索引/布隆过滤器来优化查找速度。

## 2、LSM树的Compact策略

从上面可以看出，Compact操作是十分关键的操作，否则SSTable数量会不断膨胀。在Compact策略上，主要介绍两种基本策略：size-tiered和leveled。

不过在介绍这两种策略之前，先介绍三个比较重要的概念，事实上不同的策略就是围绕这三个概念之间做出权衡和取舍。

> 1）读放大:读取数据时实际读取的数据量大于真正的数据量。例如在LSM树中需要先在MemTable查看当前key是否存在，不存在继续从SSTable中寻找。  
> 2）写放大:写入数据时实际写入的数据量大于真正的数据量。例如在LSM树中写入时可能触发Compact操作，导致实际写入的数据量远大于该key的数据量。  
> 3）空间放大:数据实际占用的磁盘空间比数据的真正大小更多。上面提到的冗余存储，对于一个key来说，只有最新的那条记录是有效的，而之前的记录都是可以被清理回收的。

_**1) size-tiered 策略**_

![](https://pic2.zhimg.com/80/v2-bedb057fde7a4ce4d5be2ea34fe86f59_720w.webp)

size-tiered策略保证每层SSTable的大小相近，同时限制每一层SSTable的数量。如上图，每层限制SSTable为N，当每层SSTable达到N后，则触发Compact操作合并这些SSTable，并将合并后的结果写入到下一层成为一个更大的sstable。

由此可以看出，_当层数达到一定数量时，最底层的单个SSTable的大小会变得非常大_。并且_size-tiered策略会导致空间放大比较严重_。即使对于同一层的SSTable，每个key的记录是可能存在多份的，只有当该层的SSTable执行compact操作才会消除这些key的冗余记录。

_**2) leveled策略**_

![](https://pic3.zhimg.com/80/v2-5f8de2e435e979936693631617a60d16_720w.webp)

每一层的总大小固定，从上到下逐渐变大

leveled策略也是采用分层的思想，每一层限制总文件的大小。

但是跟size-tiered策略不同的是，leveled会将每一层切分成多个大小相近的SSTable。这些SSTable是这一层是**全局有序**的，意味着一个key在每一层至多只有1条记录，不存在冗余记录。之所以可以保证全局有序，是因为合并策略和size-tiered不同，接下来会详细提到。

![](https://pic1.zhimg.com/80/v2-8274669affe5b9602aff45ddff29e628_720w.webp)

每一层的SSTable是全局有序的

假设存在以下这样的场景:

1) L1的总大小超过L1本身大小限制：

![](https://pic1.zhimg.com/80/v2-2546095c6b6e02af02de10cd236302f8_720w.webp)

此时L1超过了最大阈值限制

2) 此时会从L1中选择至少一个文件，然后把它跟L2_**有交集的部分(非常关键)**_进行合并。生成的文件会放在L2:

![](https://pic2.zhimg.com/80/v2-663d136cefaaf6f8301833bf29c833e9_720w.webp)

如上图所示，此时L1第二SSTable的key的范围覆盖了L2中前三个SSTable，那么就需要将L1中第二个SSTable与L2中前三个SSTable执行Compact操作。

3) 如果L2合并后的结果仍旧超出L5的阈值大小，需要重复之前的操作 —— 选至少一个文件然后把它合并到下一层:

![](https://pic1.zhimg.com/80/v2-715d76154b33abbe51e158b0cfcdc2bc_720w.webp)

需要注意的是，**_多个不相干的合并是可以并发进行的_**：

![](https://pic3.zhimg.com/80/v2-2065db94c8837edd583b6ec639eaae6e_720w.webp)

leveled策略相较于size-tiered策略来说，每层内key是不会重复的，即使是最坏的情况，除开最底层外，其余层都是重复key，按照相邻层大小比例为10来算，冗余占比也很小。因此空间放大问题得到缓解。但是写放大问题会更加突出。举一个最坏场景，如果LevelN层某个SSTable的key的范围跨度非常大，覆盖了LevelN+1层所有key的范围，那么进行Compact时将涉及LevelN+1层的全部数据。

## 3、总结

LSM树是非常值得了解的知识，理解了LSM树可以很自然地理解Hbase，LevelDb等存储组件的架构设计。ClickHouse中的MergeTree也是LSM树的思想，Log-Structured还可以联想到Kafka的存储方式。

虽然介绍了上面两种策略，但是各个存储都在自己的Compact策略上面做了很多特定的优化，例如Hbase分为Major和Minor两种Compact，这里不再做过多介绍，推荐阅读文末的RocksDb合并策略介绍。

PS:封面是在当时百度搜索lsm树的截图，真实截图，非PS。


# [论文笔记] WiscKey: Separating Keys from Values in SSD-Conscious Storage

Jul 13, 2020

阅读 WiscKey 论文时随手记录一些笔记。

这篇论文的核心思想理解起来还是很简单的，但是具体涉及到实现还有一些想不明白的地方，后来看到 TiKV 的 Titan 实现也很有趣，索性把这些问题都记录下来并抛出来。

本文中和论文相关的内容，_斜体_均为我个人的主观想法，关于 Titan 的实现，我只看过几篇公开文章以及粗浅的扫过一遍代码，如果这两部分的内容有理解错误欢迎指出，感谢！

## [](https://www.scienjus.com/wisckey/#%E8%83%8C%E6%99%AF "背景")背景

基于 LSM 树（Log-Structured Merge-Trees）的键值存储已经广泛应用，其特点是保持了数据的顺序写入和存储，利用磁盘的顺序 IO 得到了很高的性能（在 HDD 上尤其显著）。但是同一份数据会在生命周期中写入多次，随之带来高额的写放大。

![](https://www.scienjus.com/uploads/15946493854361.jpg)

以 LevelDB 为例，数据写入的整个流程为：

1.  数据首先会被写入 memtable 和 WAL
2.  当 memtable 达到上限后，会转换为 immutable memtable，之后持久化到 L0（称为 flush），L0 中每个文件都是一个持久化的 immutable memtable，多个文件间可以有相互重叠的 Key
3.  当 L0 中的文件达到一定数量时，便会和 L1 中的文件进行合并（称为 compaction）
4.  自 L1 开始所有文件都不会再有相互重叠的 Key，并且每个文件都会按照 Key 的顺序存储。每一层通常是上一层大小的 10 倍，当一层的大小超过限制时，就会挑选一部分文件合并到下一层

由此可以计算出 LevelDB 的写放大比率：

1.  由于每一层是上一层大小的 10 倍，所以在最坏情况下，上一层的一个文件可能需要和下一层的十个文件进行合并，所以合并的写放大是 10
2.  假设每行数据经过一系列 compaction 最终都会落入最终层，每层都需要重新写一次，那么从 L1 到 L6 的写放大为 5
3.  加上 WAL 和 L0，最终写放大可能会超过 50

另一方面，由于数据在 LevelDB 中的每一层（memtable/L0/L1~L6）都有可能存在，所以对于读请求，也会有一定的读放大：

1.  由于 L0 的多个文件允许有数据重叠，所以最坏情况需要查询所有文件
2.  对于 L1 到 L6，因为数据有序且不重叠，所以每层需要查询一个文件
3.  为了确认 Key 是否存在，对于每个文件都需要读取 index block、bloom-filter blocks 和 data block

论文中提供了一个实际的数据：

![](https://www.scienjus.com/uploads/15946502022182.jpg)

## [](https://www.scienjus.com/wisckey/#WiscKey-%E4%BB%8B%E7%BB%8D "WiscKey 介绍")WiscKey 介绍

### [](https://www.scienjus.com/wisckey/#%E8%AE%BE%E8%AE%A1%E7%9B%AE%E6%A0%87 "设计目标")设计目标

WiscKey 的核心思想是将数据中的 Key 和 Value 分离，只在 LSM-Tree 中有序存储 Key，而将 Value 存放在单独的 Log 中。这样带来了两点好处：

1.  当 LSM-Tree 进行 compaction 时，只会对 Key 进行排序和重写，不会影响到没有改变的 Value，也就显著降低了写放大
2.  将 Value 分离后，LSM-Tree 本身会大幅减小，所以对应磁盘中的层级会更少，可以减少查询时从磁盘读取的次数，并且可以更好的利用缓存的效果

另外，WiscKey 的设计很大一部分还建立在 SSD 的普及上，相比 HDD，SSD 有一些变化：

1.  SSD 的随机 IO 和顺序 IO 的性能差距并不像 HDD 那么大，所以 LSM-Tree 为了避免随机 IO 而采用了大量的顺序 IO，反而可能会造成了带宽浪费
2.  SSD 拥有内部并行性，但 LSM-Tree 并没有利用到该特性
3.  SSD 会因为大量的重复写入而产生硬件损耗，LSM-Tree 的高写入放大率会降低设备的寿命

下图展示了在不同请求大小和并发度时，随机读和顺序读的吞吐量，可以看到在请求大于 16KB 时，32 线程的随机读已经接近了顺序读的吞吐：

![](https://www.scienjus.com/uploads/15946506532648.jpg)

### [](https://www.scienjus.com/wisckey/#%E5%AE%9E%E7%8E%B0 "实现")实现

在 LSM-Tree 的基础上，WiscKey 引入了一个额外的存储用于存储分离出的值，称为 Value Log。整体的读写路径为：

1.  当用户添加一个 KV 时，WiscKey 会先将 Value 写入到 Value-Log 中（顺序写），然后将 Key 和 Value 在 Value Log 中的地址写入 LSM-Tree
2.  当用户删除一个 Key 时，仅在 LSM-Tree 中删除 Key 的记录，之后通过 GC 清理掉 Value Log 中的数据
3.  当用户查询一个 Key 时，会先从 LSM-Tree 中查询到 Value 的地址，再根据地址将 Value 真正从 Value-Log 中读取出来（随机读）

假设 Key 的大小为 16 Bytes，Value 的大小为 1KB，优化后的效果为：

1.  如果在 LSM-Tree 中的单层的写放大率是 10，那么使用 WiscKey 后单层的写放大率将变为 ((16 x 10) + (1024 x 1)) / (16 + 1024) = 1.14，远小于之前的 10 倍
2.  如果一个标准的 LSM-Tree 的大小为 100G，那么将 Value 分离后 LSM-Tree 本身的大小将会减少到 2G，层级会减少 1~2 级，并且缓存到内存中的比例会更高，从而降低读放大

看上去实现很简单，效果也很好，但是背后也存在了一些挑战和优化。

### [](https://www.scienjus.com/wisckey/#%E6%8C%91%E6%88%98%E4%B8%80%EF%BC%9A%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2 "挑战一：范围查询")挑战一：范围查询

在标准的 LSM-Tree 中，由于 Key 和 Value 是按照顺序存储在一起的，所以范围查询只需要顺序读即可遍历整个 SSTable 的所有数据。但是在 WiscKey 中，每个 Key 都需要额外的一次随机读才能读取到对应的 Value，因此效率会很差。

论文中的解决方案是利用上文中所提到的 SSD 内部的并行能力。WiscKey 内部会有一个 32 线程的线程池，当用户使用迭代器迭代一行时，迭代器会预先取出多个 Key，并放入到一个队列中，线程池会从队列中读取 Key 并行的查找对应的 Value。

![](https://www.scienjus.com/uploads/15946512255415.jpg)

_疑问：_

1.  _预取在某些场景是否会有浪费？（用户不准备迭代完所有数据的场景，例如 Limit 或是 Filter）_
2.  _为什么用线程池 + 队列，而不是直接用异步 IO？_

### [](https://www.scienjus.com/wisckey/#%E6%8C%91%E6%88%98%E4%BA%8C%EF%BC%9A%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%EF%BC%88GC%EF%BC%89 "挑战二：垃圾收集（GC）")挑战二：垃圾收集（GC）

上文中提到了，当用户删除一个 Key 时，WiscKey 只会将 LSM-Tree 中的 Key 删除掉，所以需要一个额外的方式清理 Value-Log 中的值。

最简单的方法是定期扫描整个 LSM-Tree，获得所有还有引用的 Value 地址，然后将没有引用的 Value 删除，但是这个逻辑非常重。

![](https://www.scienjus.com/uploads/15946515427042.jpg)

论文中介绍的方式是通过维护一个 Value Log 的有效区间（由 head 和 tail 两个地址组成），通过不断地搬运有效数据来达到淘汰无效数据。整个流程为：

1.  对于 Value-Log 中的每个值，需要额外存储 Key，为了方便从 LSM-Tree 中进行反查（相对 Value，Key 会比较小，所以写入放大不会增加太多）
2.  从 tail 的位置读取 KV，通过 Key 在 LSM-Tree 中查询 Value 是否还在被引用
3.  如果 Value 还在被引用，则将 Value 写入到 head，并将新的 Value 地址写回 LSM-Tree 中
4.  如果 Value 已经没有被引用，则跳过这行数据，接着读取下一个 KV
5.  当已经确认数据写入 head 之后，就可以将 tail 之后的数据都删除掉了

_因为需要重新写入一次 Value，并且需要将 Key 回填到 LSM-Tree 中，所以这个 GC 策略会造成额外的写放大。并且即使不做 GC，也只会影响到空间放大（删除的数据没有真正清理），所以感觉可以配置一些策略：_

1.  _根据磁盘负载和 LSM-Tree 的负载计算，仅在低峰期执行_
2.  _计算每一段数据中被删除的比例有多少，当空洞变得比较大的时候才触发 GC_

### [](https://www.scienjus.com/wisckey/#%E6%8C%91%E6%88%98%E4%B8%89%EF%BC%9A%E5%B4%A9%E6%BA%83%E4%B8%80%E8%87%B4%E6%80%A7 "挑战三：崩溃一致性")挑战三：崩溃一致性

当系统崩溃时，LSM-Tree 可以保证数据写入的原子性和恢复的有序性，所以 WiscKey 也需要保证这两点。

WiscKey 通过查询时的容错机制保证 Key 和 Value 的原子性：

1.  当用户查询时，如果在 LSM-Tree 中找不到 Key，则返回 Key 不存在
2.  如果在 LSM-Tree 中可以找到 Key，但是通过地址在 Value-Log 中无法找到匹配的 Value，则说明 Value 在写入时丢失了，同样返回不存在

这个前提建立在于 WiscKey 通过一个 Write Buffer 批量提交 Value Log（下面有详细介绍），所以才会出现 Key 写入成功后 Value 丢失的场景，用户也可以通过设置同步写入，这样在刷新 Value Log 之后，才会将 Key 写入 LSM-Tree 中。

另外，WiscKey 通过现代的文件系统的特性保证了写入的有序性，即写入一个字节序列 b1, b2, b3…bn，如果 b3 在写入时丢失了，那么 b3 之后的所有值也一定会丢失。

### [](https://www.scienjus.com/wisckey/#%E4%BC%98%E5%8C%96%E4%B8%80%EF%BC%9AWrite-Buffer "优化一：Write Buffer")优化一：Write Buffer

为了提高写入效率，WiscKey 首先会将 Value 写入到 Write Buffer 中，等待 Write Buffer 达到一定大小再一起刷新到文件中。所以查询时首先也要先从 WriteBuffer 中查询。当崩溃时，Write Buffer 中的数据会丢失，此时的行为就是上文中的崩溃一致性。

_疑问：_

1.  _根据这个描述，Value-Log 似乎是异步写入？结合上文中崩溃一致性的介绍，会有给用户返回成功但是数据丢失的情况？_

### [](https://www.scienjus.com/wisckey/#%E4%BC%98%E5%8C%96%E4%BA%8C%EF%BC%9AWAL-%E4%BC%98%E5%8C%96 "优化二：WAL 优化")优化二：WAL 优化

LSM-Tree 通过 WAL 保证了在系统崩溃时 memtable 中的数据可恢复，但是也带来了额外的一倍写放大。

而在 WiscKey 中，Value-Log 和 WAL 都是基于用户的写入顺序进行存储的，并且也具备了恢复数据的所有内容（前提是基于上文中的 GC 实现，Value Log 里存有 Key），所以理论上 Value-Log 是可以同时作为 WAL 的，从而减少 WAL 的写放大。

由于 Value Log 的 GC 比 WAL 更加低频，并且包含了大量已经持久化的数据，直接通过 Value-Log 进行恢复的话可能会导致回放大量已经持久化到 SST 的数据。所以 WiscKey 会定期将已经持久化到 SST 的 head 写入到 LSM-Tree 中，这样当恢复时只需要从最新持久化的 head 开始恢复即可。

_疑问：_

1.  _Delete 操作只需要写 LSM-Tree，但如果需要 Value Log 作为 WAL，则 Delete 也需要写入到 Value Log 中_
2.  _如果不应用这个优化，则可以做到只将大 Value 分离出 LSM-Tree，应用此优化后，小 Value 也必须要额外存到 Value Log 中了_
3.  _与其说是用 Value Log 替代 WAL，不如说是让 WAL 支持读 Value…_

## [](https://www.scienjus.com/wisckey/#%E6%95%88%E6%9E%9C "效果")效果

说完实现再看看效果，论文中有 db_bench 和 YCSB 的数据，为了节约篇幅，只贴一部分 db_bench 的数据。

db_bench 的场景分两种，一种是所有 Key 按顺序写入（这样写放大会更低，数据在每一层会更紧凑），另一种是随机写入（写放大更高，数据在每一层分布更均匀）。

### [](https://www.scienjus.com/wisckey/#%E9%A1%BA%E5%BA%8F%E5%86%99%E5%85%A5 "顺序写入")顺序写入

![](https://www.scienjus.com/uploads/15946525528572.jpg)

_效果应该来自两部分：_

1.  _WAL 没了直接省了一倍写放大_
2.  _顺序写入，每一层合并可以认为没有写放大，但是数据依旧要在每一层写一次，100 G 可能是 4~5 次（对应 L5 的大小）_

### [](https://www.scienjus.com/wisckey/#%E9%9A%8F%E6%9C%BA%E5%86%99%E5%85%A5 "随机写入")随机写入

![](https://www.scienjus.com/uploads/15946526136478.jpg)

_效果对比顺序写入，如果说为什么差距会这么大，只有可能是每一层合并造成的写放大了。_

### [](https://www.scienjus.com/wisckey/#%E7%82%B9%E6%9F%A5 "点查")点查

![](https://www.scienjus.com/uploads/15946526501920.jpg)

1.  _当 Value 比较小时，WiscKey 的劣势在于额外的一次随机读，而 LevelDB 的劣势在于读放大。当 Value 变得更大时，基于 SSD 内部的并行能力，随机读依旧能读满带宽，但是 LevelDB 读放大造成的带宽浪费却没有改善。_
2.  _另外这个测试场景是数据库大小为 100G，对于 LevelDB 来说，层级和 KV 大小挂钩，对于 WiscKey 来说，层级和 Key 大小挂钩，所以当 Value 越大，WiscKey 中的 LSM-Tree 反而更小，层级也就更低，甚至可能仅在内存中 (例如 Value 为 256KB 时，Key 加起来才 100G / (16 + 256_ 1024) _16 ~= 610KB)_

![](https://www.scienjus.com/uploads/15946527132087.jpg)

_这个有点看不懂…：_

1.  _在数据集是顺序写的场景下，LevelDB 的性能随着 Value 的增大反而降低了，这个不太理解原因（理论上读放大不会很大，而且是顺序读，很容易就能读满带宽），WiscKey 因为是随机读，并且有上文中提到的 LSM-Tree 本身很小，随着 Value 变大性能越高是符合预期的_
2.  _在数据集是随机写的场景下，一开始 WiscKey 性能低是因为随机读的延迟，随着 Value 增大，优势应该和点查类似_

### [](https://www.scienjus.com/wisckey/#GC "GC")GC

![](https://www.scienjus.com/uploads/15946527662942.jpg)

上文提到了 GC 会重写 Value 以及写回 LSM-Tree，造成额外的写入。当空余空间的占比越高时（大部分数据都已经被删了），回写的数据越少，对性能的影响也就越小。

## [](https://www.scienjus.com/wisckey/#Titan-%E7%9A%84%E5%AE%9E%E7%8E%B0 "Titan 的实现")Titan 的实现

BlobDB 和 Badger 的实现都和论文比较接近，并且也都是玩具。反而 TiKV 的 Titan 有一些独特的设计可以学习和讨论，所以下面只介绍这一案例。

### [](https://www.scienjus.com/wisckey/#%E6%A0%B8%E5%BF%83%E5%AE%9E%E7%8E%B0 "核心实现")核心实现

![](https://www.scienjus.com/uploads/15946528563659.jpg)

和 WiscKey 的主要区别在于：Titan 在 flush/compaction 时才开始分离键值，并且用于存储分离后 Value 的文件（BlobFile）会按照 Key 的顺序存储，而不是写入的顺序（其实在这个阶段，已经没有写入顺序了）。

因此导致实现上的差异有：

1.  范围查询：由于 Value-Log 没有按照 Key 排序，所以 WiscKey 需要将一个范围查询拆解为多个随机读。而 Titan 保证了局部有序，在单个 BlobFile 内部可以顺序读，但是会有多个 BlobFile 的范围有重叠，需要额外做归并。另外对于预取策略，WiscKey 建立在 SSD 并行的优势上，可以靠增加并发预取增加吞吐，而 Titan 暂时没有如此激进的预取策略
2.  WAL 优化：在 WiscKey 的实现中通过 Value Log 替代 WAL 减少了一倍写放大，而 Titan 在 flush/compaction 时才进行键值分离，肯定是没办法做这个优化的，不过这一点在 Titan 的设计文档里也提到了：「假设 LSM-tree 的 max level 是 5，放大因子为 10，则 LSM-tree 总的写放大大概为 1 + 1 + 10 + 10 + 10 + 10，其中 Flush 的写放大是 1，其比值是 42 : 1，因此 Flush 的写放大相比于整个 LSM-tree 的写放大可以忽略不计。」，个人觉得还是还比较信服的
3.  GC 策略：Titan 目前有两个版本的 GC 策略，会在下面详细介绍

### [](https://www.scienjus.com/wisckey/#GC-%E7%AD%96%E7%95%A5 "GC 策略")GC 策略

第一种策略（传统 GC）：

1.  首先挑选一些需要合并的 BlobFile，在 flush/compaction 时可以统计出每个 BlobFile 中已经删除的数据大小，从而挑选出空洞较大的文件
2.  迭代这些 BlobFile，通过 Key 查询 LSM-Tree，判断 Value 是否还在被引用
3.  如果 Value 还在被引用就写到新的 BlobFile 中，并把更新后的地址回填到 LSM-Tree 中

这个实现和论文中的 GC 方案类似，只不过论文为了 WAL 需要写入一条完整的 Value Log，所以需要维护 head 和 tail。Titan 的实现只需要每次都生成新的 BlobFile 即可。

不同点在于：WiscKey 是随机读，Value Log 的大小不会影响到读 Value 的成本。GC 策略在于写放大和空间放大的权衡，所以 GC 可以更加低频。而 BlobFile 是顺序读，如果 BlobFile 中的无效数据太多，会影响到预取的效率，间接也会影响到读的性能。

第二种策略（Level-Merge）：

1.  不存在单独的 GC，由 LSM-Tree 的 compaction 触发
2.  compaction 时，如果遍历到的值已经是一个 BlobIndex（代表值已经写入了某个 BlobFile），依旧将其读出来重新写入新的 BlobFile，也就是说每次 compaction 都会生成一批与新的 SST 完全对应的 BlobFile
3.  目前 Level Merge 只在最后两层开启

开启 Level Merge 后相当于 GC 频率和 compaction 频率持平了（GC 频率最多也只能和 compaction 持平），并且在这个基础上，直接在 compaction 里做 GC，可以减少一次回写 LSM-Tree 的成本（因为在 compaction 的过程中就能将老的 Value 地址替换掉）。

这种策略的优点在于 BlobFile 中不再有无效数据，可以用更加激进的预取策略提高范围查询的性能，缺点是写放大肯定会比之前更大（个人觉得开启后，写放大就和标准 LSM-Tree 完全一样了吧（一次 compaction 需要合并的 Key 和 Value 都需要重写一遍）？），所以只在最后两层开启。

### [](https://www.scienjus.com/wisckey/#%E6%95%88%E6%9E%9C-1 "效果")效果

Titan 的性能测试结果摘自[官网的文章](https://pingcap.com/blog-cn/titan-design-and-implementation/)，大部分结论都和 WiscKey 类似，并且文章中也分析了原因，就不在此赘述了。

因为文章是 19 年初的，所以还没有上文中的 Level Merge GC，不过 GC 策略理论上只影响范围查询的性能，所以在此贴一下范围查询的性能：

![](https://www.scienjus.com/uploads/15946541646207.jpg)

在实现 Level Merge GC 的策略之前，Titan 的范围查询只有 RocksDB 的 40%，主要原因应该还是分离后需要额外读一次 Value，以及没办法并行预取增加吞吐。 这点文章最后也提到了：

> 我们通过测试发现，目前使用 Titan 做范围查询时 IO Util 很低，这也是为什么其性能会比 RocksDB 差的重要原因之一。因此我们认为 Titan 的 Iterator 还存在着巨大的优化空间，最简单的方法是可以通过更加激进的 prefetch 和并行 prefetch 等手段来达到提升 Iterator 性能的目的。

另外在 [TiDB in Action](https://book.tidb.io/session1/chapter8/titan-in-action.html) 也提到了 Level Merge GC 可以「大幅提升 Titan 的范围查询性能」，不知道除了完全去掉无效数据之外，是否还有其他的优化，还需要再看下代码。

## [](https://www.scienjus.com/wisckey/#%E6%84%9F%E5%8F%97 "感受")感受

个人认为 WiscKey 的核心思想还是比较有意义的，毕竟适用的场景很典型而且还比较常见：大 Value、写多读少、点查多范围查询少，只要业务场景命中一个特点，效果应该就会非常显著了。

对于论文中的具体实现是否能套用在一个真实的工业实现中，我觉得大部分实现还是简单有效的，但是也有一些设计个人不太喜欢，例如使用 Value Log 替代 WAL 的方案，感觉有些过于追求减少写放大了，可能反而会引入其他问题，以及默认的 GC 策略还要写回 LSM-Tree 也有些别扭。

在和其他同事讨论内部项目的实现时，也畅想过一些其他玩法，例如只将 Value 中的一部分分离出来单独存储，或是一个分布式的 WAL 是否也能转换为 Value Log，会有哪些问题。包括看到 Titan 的实现时，我也很好奇设计成 BlobFile 这种顺序读的方式是否有什么深意（毕竟论文都把利用 SSD 写到标题里了），或者只是因为从 compaction 才开始分离键值最简单的做法就是按顺序存储 KV。

总之，期待将来能有更多工业实现落地，看到更多有趣的案例。


# Reference
https://zhuanlan.zhihu.com/p/181498475
https://www.scienjus.com/wisckey/
