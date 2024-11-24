
> [boltdb](https://github.com/boltdb/bolt) 是市面上为数不多的纯 go 语言开发的、单机 KV 库。boltdb 基于 [Howard Chu’s](https://twitter.com/hyc_symas) [LMDB 项目](http://symas.com/mdb/) ，实现的比较清爽，去掉单元测试和适配代码，核心代码大概四千多行。简单的 API、简约的实现，也是作者的意图所在。由于作者精力所限，原 boltdb 已经封版，不再更新。若想改进，提交新的 pr，建议去 etcd 维护的 fork 版本 [bbolt](https://github.com/etcd-io/bbolt)。
> 
> 为了方便，本系列导读文章仍以不再变动的原 repo 为基础。该项目麻雀虽小，五脏俱全，仅仅四千多行代码，就实现了一个基于 B+ 树索引、支持一写多读事务的单机 KV 引擎。代码本身简约朴实、注释得当，如果你是 go 语言爱好者、如果对 KV 库感兴趣，那 boltdb 绝对是不可错过的一个 repo。
> 
> 本系列计划分成三篇文章，依次围绕**数据组织**、**索引设计**、**事务实现**等三个主要方面对 boltdb 源码进行剖析。由于三个方面不是完全正交解耦的，因此叙述时会不可避免的产生交织，读不懂时，暂时略过即可，待有全貌，再回来梳理。本文是第一篇， boltdb 数据组织。

# 引子

一个存储引擎最底层的构成，就是处理数据在各种物理介质（比如在磁盘上、在内存里）上的组织。而这些数据组织也体现了该存储引擎在设计上的取舍哲学。

在文件系统上，boltdb 采用**页**（page）的组织方式，将一切数据都**对齐**到页；在内存中，boltdb 按 B+ 树组织数据，其基本单元是**节点**（node），一个内存中的树节点对应文件系统上一个或者多个**连续的**页。boltdb 就在数据组织上就只有这两种核心抽象，可谓设计简洁。当然，这种简洁必然是有代价的，后面文章会进行详细分析。

本文首先对节点和页的关系进行总体说明，然后逐一分析四种页的格式及其载入内存后的表示，最后按照 db 的生命周期串一下 db 文件的增长过程以及载入内存的策略。
# 概述

本文主要涉及到 page.go 和 freelist.go 两个源文件，主要分析了 boltdb 各种 page 在磁盘上的格式和其加载到内存中后的表示。

## 顶层组织

boltdb 的数据组织，自上而下来说：

1.  每个 db 对应一个文件。
2.  在逻辑上：
    -   一个 db 包含多个桶（bucket），n 相当于多个命名空间（namespace），桶可以无限嵌套
    -   每个桶对应一棵 B+ 树
3.  在物理上：
    -   一个 db 文件是按页为单位进行顺序存储
    -   一个页大小和操作系统的页大小保持一致（通常是 4KB）

## 页和节点

页分为四种类型：

-   **元信息页**：全局有且仅有两个 meta 页，保存在文件；它们是 boltdb 实现事务的关键
-   **空闲列表页**：有一种特殊的页，存放空闲页（freelist） id 列表；他们在文件中表现为一段一段的连续的页
-   **两种数据页**：剩下的页都是数据页，有两种类型，分别对应 B+ 树中的分支节点和叶子节点

页和节点的关系在于：

1.  页是 db 文件存储的基本单位，节点是 B+ 树的基本构成节点
2.  一个数据节点对应一到多个**连续的**数据页
3.  连续的**数据页**序列化加载到内存中就成为一个数据节点

总结一下：在文件系统上线性组织的**数据页**，通过页内指针，在逻辑上组织成了一棵二维的 B+ 树，该树的树根保存在**元信息**页中，而文件中所有其他没有用到的页的 id 列表，保存在**空闲列表页**中。

# 页格式和内存表示

boltdb 中的页分四种类型：元信息页、空闲列表页、分支节点页和叶子节点页。boltdb 使用常量枚举标记：

```go
const (  
  branchPageFlag   = 0x01  
  leafPageFlag     = 0x02  
  metaPageFlag     = 0x04  
  freelistPageFlag = 0x10  
)  
```

每个页都由定长 header 和数据部分组成：

![boltdb 页结构](https://i.loli.net/2020/11/29/scJkTp5GSdPlHW1.png)

其中 ptr 指向的是页的数据部分，为了避免载入内存和写入文件系统时的序列化和反序列化操作，boltdb 使用了大量的 go unsafe 包中的指针操作。

```go
type pgid uint64  
type page struct {  
  id       pgid  
  flags    uint16  // 页类型，值为四种类型之一  
  count    uint16  // 对应的节点包含元素个数，比如说包含的 kv 对  
  overflow uint32  // 对应节点溢出页的个数，即使用 overflow+1 个页来保存对应节点  
  ptr      uintptr // 指向数据对应的 byte 数组，当 overlay>0 时会跨越多个连续页；不过多个物理也在内存中也只会用一个 page 结构体来表示  
}  
```

##  元信息页（metaPage）

boltdb 中有且仅有两个元信息页，保存在 db 文件的开头（pageid = 0 和 1）。但是在元信息页中，ptr 指向的内容并非元素列表，而是整个 db 的元信息的各个字段。

![meta-page.png](https://i.loli.net/2020/11/29/BjTF382uakiy4QA.png)

元信息页加载到内存后数据结构如下：

```go
type meta struct {  
  magic    uint32  
  version  uint32  
  pageSize uint32 // 该 db 页大小，通过 syscall.Getpagesize() 获取，通常为 4k  
  flags    uint32 //   
  root     bucket // 各个子 bucket 根所组成的树  
  freelist pgid   // 空闲列表所存储的起始页 id  
  pgid     pgid   // 当前用到的最大 page id，也即用到 page 的数量  
  txid     txid   // 事务版本号，用以实现事务相关  
  checksum uint64 // 校验和，用于校验 meta 页是否写完整  
}  
```

##  空闲列表页（freelistPage）

空闲列表页是 db 文件中一组连续的页（一个或者多个），用于保存在 db 使用过程中由于修改操作而释放的页的 id 列表。

![freelist-page.png](https://i.loli.net/2020/11/29/d84eqFEGwiLQZUu.png)

在内存中表示时分为两部分，一部分是可以分配的空闲页列表 ids，另一部分是按事务 id 分别记录了在对应事务期间新增的空闲页列表。

```go
// 表示当前已经释放的 page 列表  
// 和写事务刚释放的 page  
type freelist struct {  
  ids        []pgid            // all free and available free page ids.  
  pending    map[txid][]pgid   // mapping of soon-to-be free page ids by tx.  
  cache      map[pgid]bool     // fast lookup of all free and pending page ids.  
}  
```

其中 `pending` 部分需要单独记录主要是为了做 MVCC 的事务：

1.  写事务回滚时，对应事务待释放的空闲页列表要从 `pending` 项中删除。
2.  某个写事务（比如 txid=7）已经提交，但可能仍有一些读事务（如 txid <=7）仍然在使用其刚释放的页，因此不能立即用作分配。

这部分内容会在 boltdb 事务中详细说明，这里只需有个印象即可。

##  空闲列表转化为 page

freelist 通过 `write` 函数，在**事务提交时**将自己写入给定的页，进行持久化。在写入时，会将 `pending` 和 `ids` 合并后写入，这是因为：

1.  `write` 函数是在写事务提交时调用，写事务是串行的，因此 `pending` 中对应的写事务都已经提交。
2.  写入文件是为了应对崩溃后重启，而重启时没有任何读操作，自然不用担心有读事务还在用刚释放的页。

```go
func (f *freelist) write(p *page) error {  
  // 设置页类型  
  p.flags |= freelistPageFlag  
  
  // page.count 是 uint16 类型，其能表示的范围为 [0, 64k-1] 。如果空闲页 id 列表长度超出了此范围，就需要另想办法。  
  // 这里用了个 trick，将 page.count 置为 64k 即 0xFFF，然后在数据部分的第一个元素存实际数量（以 pgid 为类型，即 uint64）。  
  lenids := f.count()  
  if lenids == 0 {  
    p.count = uint16(lenids)  
  } else if lenids < 0xFFFF {  
    p.count = uint16(lenids)  
    // copyall 会将 pending 和 ids 合并并排序  
    f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[:])   
  } else {  
    p.count = 0xFFFF  
    ((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0] = pgid(lenids)  
    f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[1:])  
  }  
  
  return nil  
}  
```

**注意**本步骤只是将 `freelist` 转化为内存中的页结构，需要额外的操作，比如 `tx.write()` 才会将对应的页真正持久化到文件。

##  空闲列表从 page 中加载

在数据库重启时，会首先从前两个元信息页恢复出一个合法的元信息。然后根据元信息中的 `freelist` 字段，找到存储 freelist 页的起始地址，进而将其恢复到内存中。

```go
func (f *freelist) read(p *page) {  
  // count == 0xFFFF 表明实际 count 存储在 ptr 所指向的内容的第一个元素  
  idx, count := 0, int(p.count)  
  if count == 0xFFFF {  
    idx = 1  
    count = int(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0])  
  }  
  
  // 将空闲列表从 page 拷贝内存中 freelist 结构体中  
  if count == 0 {  
    f.ids = nil  
  } else {  
    ids := ((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[idx:count]  
    f.ids = make([]pgid, len(ids))  
    copy(f.ids, ids)  
  
    // 保证 ids 是有序的  
    sort.Sort(pgids(f.ids))  
  }  
  
  // 重新构建 freelist.cache 这个 map.  
  f.reindex()  
}  
```

##  空闲列表分配

作者原版的空闲列表分配异常简单，**分配单位**是页，分配策略是 ** [首次适应](https://baike.baidu.com/item/%E9%A6%96%E6%AC%A1%E9%80%82%E5%BA%94%E7%AE%97%E6%B3%95/10147334)**：即从排好序的空闲页列表 `ids` 中，找到第一段等于指定长度的连续空闲页，然后返回起始页 id。

```go
// 如果可以找到连续 n 个空闲页，则返回起始页 id  
// 否则返回 0  
func (f *freelist) allocate(n int) pgid {  
  if len(f.ids) == 0 {  
    return 0  
  }  
  
  // 遍历寻找连续空闲页，并判断是否等于 n  
  var initial, previd pgid  
  for i, id := range f.ids {  
    if id <= 1 {  
      panic(fmt.Sprintf("invalid page allocation: %d", id))  
    }  
  
    // 如果不连续，则重置 initial  
    if previd == 0 || id-previd != 1 {  
      initial = id  
    }  
  
    if (id-initial)+1 == pgid(n) {  
      // 当正好分配到 ids 中前 n 个 page 时，仅简单往前调整 f.ids 切片即可。  
      // 尽管一时会造成空间浪费，但是在对 f.ids append/free 操作时，会按需  
      // 重新空间分配，重新分配会导致这些浪费空间被回收掉  
      if (i + 1) == n {  
        f.ids = f.ids[i+1:]  
      } else {  
        copy(f.ids[i-n+1:], f.ids[i+1:])  
        f.ids = f.ids[:len(f.ids)-n]  
      }  
  
      // 从 cache 中删除对应 page id  
      for i := pgid(0); i < pgid(n); i++ {  
        delete(f.cache, initial+i)  
      }  
  
      return initial  
    }  
  
    previd = id  
  }  
  return 0  
}  
```

这个 GC 策略相当简单直接，是线性的时间复杂度。阿里似乎做过一个 patch，将所有空闲 page 按其连续长度 group by 了一下。

## 叶子节点页（leafPage）

这种页对应 **B+ 树**中叶子节点，叶子节点包含的元素有两种类型：普通 KV 数据、subbucket。

对于前者来说，页中存储的基本元素为某个 bucket 中一条用户数据。对于后者来说，页中的一个元素为该 db 中的某个 subbucket 。

```go
// page ptr 指向的字节数组中的单个元素  
type leafPageElement struct {   
  flags         uint32    // 普通 kv （flags=0）还是 subbucket（flags=bucketLeafFlag）  
  pos           uint16    // kv header 与对应 kv 的距离  
  ksize         uint32    // key 的字节数  
  vsize         uint32    // val 字节数  
}  
```

其详细结构如下：

![leaf-page-element.png](https://i.loli.net/2020/11/29/4eGQqLtk5fmNvH7.png)

可以看出，leaf page 在组织数据时，将**元素头**（`leafPageElement`）和**元素本身**（`key value`）分开存储。这样的好处在于 `leafPageElement` 是定长的，可以按下标访问对应元素。在二分查找指定 key 时，只需按需加载相应页到内存（访问 page 时是通过 mmap 进行的，因此只有访问时才会真正将数据从文件系统中加载到内存）即可。

```go
inodes := p.leafPageElements()  
index := sort.Search(int(p.count), func(i int) bool {  
  return bytes.Compare(inodes[i].key(), key) != -1  
})  
```

如果元素头和对应元素紧邻存储，则需将 `leafPageElement` 数组对应的所有页顺序读取，全部加载到内存，才能进行二分。

另外一个小优化是 pos 存储的是元素头的起始地址到元素的起始地址的**相对偏移量**，而非以 `ptr` 指针为起始地址的**绝对偏移量**。这样可以用尽量少的位数（`pos` 是 `uint16`） 表示尽量长的距离。
```go
func (n *branchPageElement) key() []byte {  
  buf := (*[maxAllocSize]byte)(unsafe.Pointer(n)) // buf 是元素头起始地址  
  return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos]))[:n.ksize]  
}  
```

##  分支节点页（branchPage）

分支节点页和叶子节点页的结构大体相同。不同之处在于，页中保存的数据的 value 是 page id，即该分支节点在哪些 key 上的分支分别指向的 page 。

![branch-element.png](https://i.loli.net/2020/12/05/6SPHqWmLXkwiusj.png)

`branchPageElement` 中的 key 存的是其指向的页中的起始 key。

#  转换流程

boltdb 使用 mmap 将 db 文件映射到内存空间。在构建树并且访问过程中，按需将对应的页加载到内存里，并且利用操作系统的页缓存策略进行替换。

##  文件增长

当我们打开一个 db 时，如果发现该 db 文件为空，会在内存中初始化四个页（4\*4k=16K），分别是两个元信息页、一个空的空闲列表页和一个空的叶子节点页，然后将其写入 db 文件，然后走正常打开流程。

```go
func (db *DB) init() error {  
  // 设置页大小与操作系统一致  
  db.pageSize = os.Getpagesize()  
  
  buf := make([]byte, db.pageSize*4)  
  // 在 buffer 中创建两个元信息页.  
  for i := 0; i < 2; i++ {  
    p := db.pageInBuffer(buf[:], pgid(i))  
    p.id = pgid(i)  
    p.flags = metaPageFlag  
  
    // 初始化元信息页.  
    m := p.meta()  
    m.magic = magic  
    m.version = version  
    m.pageSize = uint32(db.pageSize)  
    m.freelist = 2  
    m.root = bucket{root: 3}  
    m.pgid = 4  
    m.txid = txid(i)  
    m.checksum = m.sum64()  
  }  
  
  // 在 pgid=2 的页写入一个空的空闲列表.  
  p := db.pageInBuffer(buf[:], pgid(2))  
  p.id = pgid(2)  
  p.flags = freelistPageFlag  
  p.count = 0  
  
  // 在 pgid=3 的页写入一个空的叶子元素.  
  p = db.pageInBuffer(buf[:], pgid(3))  
  p.id = pgid(3)  
  p.flags = leafPageFlag  
  p.count = 0  
  
  // 将 buffer 中的这四个页写入数据文件并刷盘  
  if _, err := db.ops.writeAt(buf, 0); err != nil {  
    return err  
  }  
  if err := fdatasync(db); err != nil {  
    return err  
  }  
  
  return nil  
}  
```

随着数据的不断写入，需要申请新的页。boltdb 首先会去 freelist 中找有无可重复利用的页，如果没有，就只能进行 re-mmap（先 mumap 在 mmap），扩大 db 文件。每次扩大会进行倍增（因此从 16K * 2 = 32K 开始），到达 1G 后，再次扩大会每次新增 1G。

```go
func (db *DB) mmapSize(size int) (int, error) {  
  // 从 32KB 开始，直到 1GB.  
  for i := uint(15); i <= 30; i++ {  
    if size <= 1<<i {  
      return 1 << i, nil  
    }  
  }  
  
  // Verify the requested size is not above the maximum allowed.  
  if size > maxMapSize {  
    return 0, fmt.Errorf("mmap too large")  
  }  
  
  // 对齐到 maxMmapStep = 1G  
  sz := int64(size)  
  if remainder := sz % int64(maxMmapStep); remainder > 0 {  
    sz += int64(maxMmapStep) - remainder  
  }  
  
  // 对齐到 db.pageSize  
  pageSize := int64(db.pageSize)  
  if (sz % pageSize) != 0 {  
    sz = ((sz / pageSize) + 1) * pageSize  
  }  
  
  // 不能超过 maxMapSize  
  if sz > maxMapSize {  
    sz = maxMapSize  
  }  
  
  return int(sz), nil  
}  
```

在 32 位 机器上文件最大不能超过 `maxMapSize` = 2G；在 64 位机器上，文件上限为 256T。

##  读写流程

在打开一个已经存在的 db 时，会首先将 db 文件映射到内存空间，然后解析元信息页，最后加载空闲列表。

在 db 进行读取时，会按需将访问路径上的 page 加载到内存，并转换为 node，进行缓存。

在 db 进行修改时，使用 COW 原则，所有修改不在原地，而是在改动前先复制一份。如果叶子节点 node 需要修改，则 root bucket 到该 node 路径上所涉及的所有节点都需要修改。这些节点都需要新申请空间，然后持久化，这些和事务的实现息息相关，之后会在本系列事务文章中做详细说明。

#  小结

boltdb 在数据组织方面只使用了两个概念：页（page） 和节点 （node）。每个数据库对应一个文件，每个文件中包含一系列线性组织的页。页的大小固定，依其性质不同，分为四种类型：元信息页、空闲列表页、叶子节点页、分支节点页。打开数据库时，会渐次进行以下操作：

1.  利用 mmap 将数据库文件映射到内存空间。
2.  解析元信息页，获取空闲列表页 id 和 root bucket 页 id。
3.  依据空闲列表页 id ，将所有空闲页列表载入内存。
4.  依据 root bucket 起始页地址，解析 root bucket 根节点。
5.  根据读写需求，从树根开始遍历，按需将访问路径上的数据页（分支节点页和叶子节点页）载入内存成为节点（node）。

可以看出，节点分两种类型：分支节点（branch node）和叶子节点（leaf node）。

另外需要注意的是，由于嵌套 bucket 的存在，导致这一块稍微有点不好理解。在下一篇 boltdb 的**索引设计**中，将详细剖析 boltdb 是如何组织多个 bucket 以及单个 bucket 内的 B+ 树索引的。

# tt
boltdb是一个纯粹的key Value数据库，其宗旨是提供一个简单，快速，可信的数据库。此数据库广泛应用于各大开源组件中。

[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)目录为：

![](https://img-blog.csdn.net/20180408093842950)

源码比较多，且其内部逻辑比较复杂，本文只分析其中的page结构。

github.com/boltdb/bolt/page.go

![](https://img-blog.csdn.net/20180408093857817)

page结构体。page指的是内存中的页，这个结构体其实是用来对应页，然后将其管理起来的数据结构。

id：是pgid类型，是给page的编号。

flags：是指的此页中保存的具体数据类型。（有好几种）

count：记录具体数据类型中的计数，不同的类型具有不同的含义

overflow：用来记录是否有跨页

ptr：是具体的数据类型。这种用法让我想起来了c语言中的用法。在Linux内核中经常用到。比如典型的场景就是虚拟文件系统的接口。

![](https://img-blog.csdn.net/20180408093913748)

![](https://img-blog.csdn.net/20180408093927484)

这个是具体的数据类型。上面写的有4种。

branchPageFlag：分支节点

leafPageFlag：叶子节点

以上是用树结构来管理页的关系

metaPageFlag：meta页

freelistPageFlag：freelist页（这个以后文章再说吧）

![](https://img-blog.csdn.net/20180408093942138)

上面的pageHeaderSize很难解释。还是画图来看吧。

![](https://img-blog.csdn.net/20180408093956806)

从上面的图就可以很直接的看到了。page的头部包含了一些信息，最后的ptr是具体的数据结构。

![](https://img-blog.csdn.net/20180408094012712)

获取到meta格式的数据结构

![](https://img-blog.csdn.net/20180408094028281)

上面是叶子节点的转换

主要是数组的整个转化[0x7FFFFFFFF]leafpageElement 类型，

指针是p.ptr，返回了数组转化后的[:]

![](https://img-blog.csdn.net/20180408094044397)

pos就是相对的偏移

ksize，key的大小

vsize，value的大小

具体可以看下图结构

![](https://img-blog.csdn.net/20180408094059066)

![](https://img-blog.csdn.net/20180408094113916)

分支节点的类似的转化

![](https://img-blog.csdn.net/20180408094131250)

pos为相对于branchPageElement的偏移位置

看下这个具体的内存分布结构图

![](https://img-blog.csdn.net/20180408094143764)

github.com/boltdb/bolt/db.go

![](https://img-blog.csdn.net/20180408094202403)

meta相对比结构会固定，但内容更多

# boltdb源码分析系列-文件结构

发布于2022-08-15 15:01:59阅读 3470

小编在阅读etcd源码的时候，看到它的存储storage使用了BoltDB[数据库](https://cloud.tencent.com/solution/database?from=20065&from_column=20065)。查阅资料发现它是一个go语言开发的单机的KV数据库，是awesome-go推荐学习的项目，它的代码组织结构非常清晰，代码量也不是很大，于是小编阅读了BoltDB的源码，希望有两方面的收获，一是学习代码的组织分层，提升代码品味；二是加深对数据库知识的理解，数据快速查找是怎么实现的，MVCC机制具体是怎么实现的。

本系列文章是小编学习完BoltDB源码的总结输出，通过图示、理论说明和源码分析结合的方式，讲述bolt是怎么实现的，用了哪些黑科技等等。分析的BoltDB源码地址在`https://github.com/boltdb/bolt`。

## **bolt简介**

BoltDB是根据LMDB项目开发的一个go语言版的key-value存储，它的目标是为应用提供一个简单、高效和可靠的嵌入式[数据库存储](https://cloud.tencent.com/product/crs?from=20065&from_column=20065)。github地址为`https://github.com/boltdb/bolt`, 由于作者精力和鉴于当前版本已经稳定，作者不在更新此项目。etcd项目中使用的bolt单独拉取了分支(`https://github.com/etcd-io/bbolt`)，相应的由etcd维护.

![](https://ask.qcloudimg.com/http-save/yehe-9955853/3b560730cfb4dbfef8cd73dbebe262a9.jpeg?imageView2/2/w/2560/h/7000)

readme介绍比较长,这里对BoltDB关键性概念做一个说明。

-   支持事务操作 BoltDB支持两种事务类型：只读事务和读写事务，多个只读事务可以并发执行，多个只读事务和一个读写事务也可以并发执行，但是多个读写事务不能并发执行
-   单机型 BoltDB是单机型数据库，不是分布式数据库，比较使用的场景一般是做日志存储
-   key-value存储 也就是说BotdDB是一款nosql数据库，与redis同类型。不过，它存储的key和value都是[]byte类型，不像redis提供了丰富的数据类型。
-   文件型 BoltDB属于文件型数据库，因为它的所有数据都是存储在磁盘上，这与redis是内存数据库有差别。

BoltDB具有不错的性能，得益于它实现中使用了一些黑科技，总结有以下几点：

-   采用mmap技术 使用操作系统提供的mmap技术，减少了文件内容的复制次数，从而提高了文件的读写效率
-   COW技术 为了提高读操作的并发性，使用了写时复制技术（copy on write）
-   B+树 通过使用B+树实现索引，提高了随机读操作的性能

## **BoltDB文件**

boltDB是一个文件型数据库，写入的数据存储在磁盘的文件上。下面的test.db文件就是一个BoltDB文件，以十六进制查看它，长的是下面这样的，第一列是文件偏移量地址，后面的是内容。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/1b61e0d22cc080bb13ea7389a80dfcbf.jpeg?imageView2/2/w/2560/h/7000)

db文件是按页(page)为单位进行顺序存储的，页的大小和操作系统页的大小是相同的，一般都是4KB. 为什么要将db文件划分成页处理，小编认为是方便管理和减少存储空间的浪费。

一个db文件文件由多个page构成，每个page大小为4KB。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/41256b1e762e21284f1496311582e2f0.jpeg?imageView2/2/w/2560/h/7000)

所以BoltDB文件的大小一般都是4KB的整数倍，像32KB,64KB,256KB。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/d3cd7e773218114e71ec43ac75dd0a62.jpeg?imageView2/2/w/2560/h/7000)

![](https://ask.qcloudimg.com/http-save/yehe-9955853/e6b1ec4a135e0d4ed341d6727b22a81c.jpeg?imageView2/2/w/2560/h/7000)

## **page**

前面介绍了BoltDB文件是按page组织的，本小节详细介绍page有哪几种类型，以及page的构成。

page有四种类型，分别是meta page、freelist page、branch page和leaf page,翻译成中文是元数据页、空闲列表页、分支页和叶子节点页。

```javascript
const (
 // 分支节点页
 branchPageFlag   = 0x01
 // 叶子节点页
 leafPageFlag     = 0x02
 // 元数据页
 metaPageFlag     = 0x04
 // 空闲列表页
 freelistPageFlag = 0x10
)
```

复制

page数据结构定义如下, id为页id，每页有唯一的id值，flags表示页的类型，对应的就是上面的四种类型之一，count表示页中元素的数量，overflow数据是否有溢出，主要在空闲列表上有用, ptr存储真实的数据。

```javascript
type page struct {
 id       pgid
 flags    uint16
 count    uint16
 overflow uint32
 ptr      uintptr
}

type pgid uint64
```

复制

一个page分为两部分，分别是page header和page data。header部分是固定的，它描述了page的基本信息，它的id值，页类型等，data部分是真正存储数据的。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/33ed8df120fcb19765084fe3d4a128e8.jpeg?imageView2/2/w/2560/h/7000)

### **meta page**

1个BoltDB文件中meta page有两个，分别是meta0和meta1，它们的结构是一样的。meta0和meta1是文件开头的两个page. header部分前面已经介绍过了，这里我们主要关注它的data部分。data部分包含字段信息如下：

-   magic: 魔数，用于判断文件是否是BoltDB文件
-   version:版本号，固定为2
-   pageSize:描述页的大小，为4KB
-   root: BoltDB文件虽然是按page线性存储在磁盘上的，但是它的组织结构是B+树的形式，是为了快速检索查找。root存储的树根所在的page id信息
-   freelist:空闲列表页的page id值
-   txid:数据库事务操作序号
-   checksum:data数据的hash摘要，用于判断data部分数据是否损坏

![](https://ask.qcloudimg.com/http-save/yehe-9955853/d423a27c7c559b0144ff57d08b2a5479.jpeg?imageView2/2/w/2560/h/7000)

至于为什么有两个meta page页以及meta页的作用，留着在后续的事务介绍文章中做说明。这里我们对meta page结构有个整体认识即可。

### **freelist page**

空闲列表页中的data部分存储的是一个一个的pgid值，pgid即为page id，它是uint64类型。这些pgid对应的page是空闲的，可以进行复用存储数据。header中的count描述了data中存储pgid数量，因为count类型为uint16,所以它的最大值为0xFFFF。当count的值小于0xFFFF时候，它的值即为data中真实的pgid数量，对应下图。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/9085639dc84cb3937b87b34e91b94de7.jpeg?imageView2/2/w/2560/h/7000)

那数量超过了uint16怎么处理呢? BoltdDB将真实数量存储在data部分最开头的8字节中，此时header中的count值为0xFFFF是一个标记，表示真实的数量在data部分开头。后面的部分在开始存储pgid值。如下图所示。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/564ebdcb167ba65fe5b6da30dc8ebfb6.jpeg?imageView2/2/w/2560/h/7000)

### **branch page**

分支节点页存储索引信息，BoltDB通过B+索引来加快数据查询检索，所以除了要存储key-value数据，还要存储B+内部分支节点信息。分支节点中的data存储的是索引key和pgid值，格式如下图。整个data分为branchPageElements和keys两部分，branchPageElements描述了key的大小，每个key起始位置信息。通过branchPageElement刻画了每个key的信息。

-   pos:key的起始位置距离它的branchPageElement头的差值
-   ksize:描述key的长度
-   pgid:子page页的id

![](https://ask.qcloudimg.com/http-save/yehe-9955853/ea5b6fe7a09262aa9e6e8b7cd7140e44.jpeg?imageView2/2/w/2560/h/7000)

### **leaf page**

叶子节点页是真正存储key-value数据的地方，向BoltDB中存储的数据都存放在叶子节点上。它的结构如下，与branchPageElement有点类似，data部分也分为leafPageElement和key-value两部分，leafPageElement描述了每个key的起始位置，key的大小和value的大小。

BoltdDB中有Bucket（桶）的概念，这里先做一点说明，一个Bucket可以看做关系型数据像[MySQL](https://cloud.tencent.com/product/cdb?from=20065&from_column=20065)中的一个表，BoltDB中比较独特的是Bucket中可以嵌套Bucket，Bucket的子Bucket信息存储在叶子节点上。所以我们看到下图中leafPageElement中有个flags字段，它的值为0表示存储的key-value数据，值为1表示这个是一个子Bucket.Bucket有一个名称，一个Bucket中的存储的key-value数据中的key值不能和它的子Bucket的名称相同。Bucket结构和功能在后续的文章中做详细介绍。leafPageElement中pos的值描述的是key距离它的差值，ksize和vsize分别表示key的长度和value长度。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/3d70b51d9df49db4b33776a015ff60b7.jpeg?imageView2/2/w/2560/h/7000)

## **总结**

本文主要介绍了BoltDB物理存储，一个BoltDB文件是一系列page的集合，通过图示介绍了每种page的结构以及各个字段的含义，下篇将介绍BoltDB文件加载到内存之后，它的逻辑组织结构。


# boltdb源码分析系列-内存结构


发布于2022-08-15 15:02:47阅读 1330

## **BoltDB内存结构**

在[boltdb源码分析系列-文件结构](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488511&idx=1&sn=995690cd1c546c46412455f5717ec2dd&chksm=873a6dbeb04de4a878f14c282ecc57248ffad855856f583d046ad8d7bb0da298e7a3a1bbe2d8&scene=21#wechat_redirect)文章中讲述了BoltdDB文件是按page位单位进行物理存储的，当我们Open打开一个BoltDB文件之后，文件内容会被加载到内存。文件中的page页在内存中以node来组织，如果说page是BoltDB物理存储单元，那么node是BoltDB的逻辑单元，是page在内存中的表示。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/ee754bcc0c45e50ae7de86fe631cac3f.jpeg?imageView2/2/w/2560/h/7000)

## **node**

node是page在内存中逻辑表示，现在我们来看node的构成，它的结构体定义如下。

```javascript
type node struct {
 // 该node节点属于哪个桶
 bucket     *Bucket
 // 是否是叶子节点
 isLeaf     bool
 // 是否平衡，即节点中的元素是否在一定范围，不平衡，则需要合并
 unbalanced bool
 // 节点中的元素是否太多了，需要分裂, spilled为true表示已经分裂过了
 spilled    bool
 key        []byte
 pgid       pgid
 // 当前节点的父节点
 parent     *node
 // 孩子节点
 children   nodes
 // 节点中存储的数据
 inodes     inodes
}
```

复制

node中各个字段与page构成各字段没有明确的对应关系。为了好理解，小编将node中所有的字段分成两类，一类是描述的是page中的内容，另一类是操作控制信息。下面对这两类字段分别进行详细介绍。

在介绍node是如何描述page内容之前，先说一点，就是这里的node描述的只是branch page和leaf page在内存中表示。在[boltdb源码分析系列-文件结构](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488511&idx=1&sn=995690cd1c546c46412455f5717ec2dd&chksm=873a6dbeb04de4a878f14c282ecc57248ffad855856f583d046ad8d7bb0da298e7a3a1bbe2d8&scene=21#wechat_redirect)中介绍了BoltDB文件由meta page、freelist page、branch page和leaf page。那meta page和freelist page在内存是怎么表示的呢？因为meta page只有固定的两个page,freelist page一般是一个，并且meta page在db文件的最开头位置，所以meta和freelist page直接放在了DB结构体中，也很好理解，一个db文件它们是固定的，放在这里操作也方便。

下面是DB的结构体定义，这里只抽取出了与meta page和freelist page相关信息，可以看到，两个meta page内容保存在meta0和meta1中，freelist page保存在freelist。

```javascript
type DB struct {
 ...
    // meta0 保存 mate page1信息
 meta0    *meta
    // meta1 保存 mate page2信息
 meta1    *meta
 ...
    // freelist存储freelist page信息
 freelist *freelist
 ...
}
```

复制

BoltDB有四种类型的page,现在就剩下branch page和leaf page了。它们在内存中的信息保存在node结构中。

现在开始分析node是如何描述page内容的，branch page和leaf page格式基本是相似的。page header中的id对应到node中是pgid，node中isLeaf表示是分支节点还是叶子节点，对应到page header中的flags字段。page中的data信息存在node中的inodes中，因为inodes信息是结构化的，它是一个切片数组，可以直接定位第几个key以及它的内容。通过inodes切片的长度我们就知道了元素的个数，所以page header中的count值隐含在inodes中，同理branchPageElements和leafPageElements信息也隐含在inodes中。所以node结构完整的描述了page内容。

node结构中另一类是字段是方便控制操作用的，这个属于附加信息，用在读写、查询各种逻辑操作上。像指向父节点node的parent指针，以及标识节点是否进行过平衡操作的unbalanced，节点是否进行过分裂的spilled。

## **node与page相互转换**

当我们打开一个BoltDB文件之后，它的信息(page)会被加载到内存以node表示。进行事务操作，无论执行写入key-value，还是修改已有key的value值等操作，都是对内存中的node进行修改，最后执行事务Commit操作时，内存中的node才会写入磁盘永久存储。这个过程涉及到node和page相互转换，下面结合源码分析这个过程。

**「page转成node的过程可以看做node的反序列化，node转成page的过程可以看做node序列化」**，就像对内存中的结构体对象进行json序列化和反序列化操作那样，可以将一个对象序列化成二进制，相反，也可以将二进制反序列化成结构体对象。

### **page转成node**

将page转成node操作由方法`func (n *node) read(p *page)`完成，该方法在node.go文件中，具体实现如下：

```javascript
// 根据一个page页初始化节点n，相当于对node进行反序列化
func (n *node) read(p *page) {
 // 赋值page id
 n.pgid = p.id
 // 是否是叶子节点
 n.isLeaf = ((p.flags & leafPageFlag) != 0)
 // 节点中元素的大小，开辟空间
 n.inodes = make(inodes, int(p.count))

 // 填充节点中的元素数据
 for i := 0; i < int(p.count); i++ {
  inode := &n.inodes[i]
  if n.isLeaf {
   // 叶子节点有flags-key-value信息
   elem := p.leafPageElement(uint16(i))
   inode.flags = elem.flags
   inode.key = elem.key()
   inode.value = elem.value()
  } else {
   // 内部节点只有key-pgid信息，可以把pgid理解为内部节点的value信息，它指向的是孩子节点的page id
   elem := p.branchPageElement(uint16(i))
   inode.pgid = elem.pgid
   inode.key = elem.key()
  }
  _assert(len(inode.key) > 0, "read: zero-length inode key")
 }

 // 初始节点n的key值为元素中第一个key-value中的key值
 if len(n.inodes) > 0 {
  n.key = n.inodes[0].key
  _assert(len(n.key) > 0, "read: zero-length node key")
 } else {
  n.key = nil
 }
}
```

复制

read处理过程非常简单，根据page信息给node各字段赋值。根据page中count的值，开辟对应大小的inodes空间，inode中填充的信息根据page的类型有所不同，如果是branch page，inode中装载的是key和pgid信息，如果是leaf page，inode中装载的是key-value信息。

核心点在于从page中获取key-value数据的过程，得益于page数据结构非常好的定义，运用了unsafe.Pointer将p.ptr指针，先转成branchPageElement或leafPageElement数组的**「数组指针」**，最后获取下标为index处的地址返回。对应到下面的源码。

```javascript
func (p *page) branchPageElement(index uint16) *branchPageElement {
 return &((*[0x7FFFFFF]branchPageElement)(unsafe.Pointer(&p.ptr)))[index]
}

func (p *page) leafPageElement(index uint16) *leafPageElement {
 n := &((*[0x7FFFFFF]leafPageElement)(unsafe.Pointer(&p.ptr)))[index]
 return n
}
```

复制

成功获取到page中每个branchPageElement或leafPageElement之后，就可以通过它直接来获取key-value数据了。这个过程利用elem中记录的偏移量pos，以及key的长度，运用unsafe.Pointer黑科技，提取出key-value数据。

```javascript
func (n *leafPageElement) key() []byte {
 buf := (*[maxAllocSize]byte)(unsafe.Pointer(n))
 return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos]))[:n.ksize:n.ksize]
}

func (n *leafPageElement) value() []byte {
 buf := (*[maxAllocSize]byte)(unsafe.Pointer(n))
 return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos+n.ksize]))[:n.vsize:n.vsize]
}

func (n *branchPageElement) key() []byte {
 buf := (*[maxAllocSize]byte)(unsafe.Pointer(n))
 return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos]))[:n.ksize]
}
```

复制

### **node转成page**

node转成page的过程与上面的过程刚好相反，在`func (n *node) write(p *page)`方法中实现，源码也在node.go文件中。

write操作是根据node结构中字段值初始化page的过程，例如根据n.isLeaf是否是叶子节点初始化p(page)的flags信息，根据n.inodes的大小初始化p.count。这里可以看到，inodes的大小是小于65535的，如果超过了，直接panic，这说明page中的元素的个数是不能超过这个数值的。

最核心的操作就是定位到page data中的位置，将inode中的信息写入，通过`elem:=p.leafPageElement(uint16(i))`或`elem:=p.branchPageElement(uint16(i))`操作，定位到了要操作的内存地址，返回的elem是指针，所以后续对elem的赋值内容，都填充到page中。在填充page data的时候，先填充描述key-value的元数据信息branchPageElements和leafPageElements，最后通过copy深拷贝填充key-value数据。

```javascript
// 上面过程的逆过程，根据节点n的内容初始page,相当于对node进行序列化
func (n *node) write(p *page) {
 // 初始化是否是页节点标记位
 if n.isLeaf {
  p.flags |= leafPageFlag
 } else {
  p.flags |= branchPageFlag
 }
    // 节点中的元素个数超过2^16=65535个，溢出了
 if len(n.inodes) >= 0xFFFF {
  panic(fmt.Sprintf("inode overflow: %d (pgid=%d)", len(n.inodes), p.id))
 }
 p.count = uint16(len(n.inodes))

 // 节点n中没有元素，直接返回，因为没有要处理的元素
 if p.count == 0 {
  return
 }

 // 定位到key-value数据要写入的位置，循环写入key-value
 b := (*[maxAllocSize]byte)(unsafe.Pointer(&p.ptr))[n.pageElementSize()*len(n.inodes):]
 for i, item := range n.inodes {
  _assert(len(item.key) > 0, "write: zero-length inode key")

  // 叶子节点
  if n.isLeaf {
   // elem是个指针，给elem赋值，相当于对p中的数据进行修改
   // 修改的是p中描述元素的元素头信息，即元素的key-value的起始位置，类型，以及key和value的大小
   elem := p.leafPageElement(uint16(i))
   elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))
   elem.flags = item.flags
   elem.ksize = uint32(len(item.key))
   elem.vsize = uint32(len(item.value))
  } else {
   // 处理分支节点
   elem := p.branchPageElement(uint16(i))
   elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))
   elem.ksize = uint32(len(item.key))
   elem.pgid = item.pgid
   _assert(elem.pgid != p.id, "write: circular dependency occurred")
  }
        
  // See: https://github.com/boltdb/bolt/pull/335
  klen, vlen := len(item.key), len(item.value)
  // b的空间不能装下key-value数据，进行扩容
  if len(b) < klen+vlen {
   b = (*[maxAllocSize]byte)(unsafe.Pointer(&b[0]))[:]
  }

  // 拷贝数据key
  copy(b[0:], item.key)
  b = b[klen:]
  // 拷贝数据value
  copy(b[0:], item.value)
  b = b[vlen:]
 }

 // DEBUG ONLY: n.dump()
}
```

复制

上面详细分析了branch page、leaf page与node转换过程，此过程概括起来用下面的这种图描述。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/fe944bdad0c61342fecd5d205ba22ac0.jpeg?imageView2/2/w/2560/h/7000)

## **DB.meta与meta page相互转换**

前面说的node与page相互转换实际是branch page和leaf page与node相互转换，因为meta page和freelist page中的data比较特别，所以它们加载到内存后是存放在DB.meta字段和DB.freelist字段中的。

本小节分析meta page与DB.meta之间的相互转换。两个meta page与DB.meta0和DB.meta1是一一对应的。将meta page转换为DB.meta的处理在db.go文件`func (db *DB) mmap(minsz int) error`方法中。下面只抽取出了相关的逻辑。

```javascript
func (db *DB) mmap(minsz int) error {
 ...
 // Save references to the meta pages.
 // 加载meta page到内存中的db.meta0和meta1中
 db.meta0 = db.page(0).meta()
 db.meta1 = db.page(1).meta()

 err0 := db.meta0.validate()
 err1 := db.meta1.validate()
 ...
}

// 根据id，计算出数据偏移地址，获取起始地址处page大小的数据
func (db *DB) page(id pgid) *page {
 pos := id * pgid(db.pageSize)
 return (*page)(unsafe.Pointer(&db.data[pos]))
}
```

复制

两个meta page的id分别是0和1，所以直接通过db.page()传入pgid，通过unsafe.Pointer获取到了meta page0和meta page1。然后调用page的meta方法获取page data数据。

上面分析了meta page转换为DB.meta过程，下面分析它的逆过程，即如何将DB.meta转为meta page.对[数据库](https://cloud.tencent.com/solution/database?from=20065&from_column=20065)有更新操作，元数据meta才会有变化，才会将DB.meta转成page，刷新到磁盘中。

BoltDB事务操作分为只读事务和读写事务，只读事务不会更新数据库内容，所以不会更新meta page, 只有读写事务会更新meta page. 所以内存中DB.meta转换为meta page，最后写入磁盘，这个过程在事务提交tx.Commit处理的。

```javascript
// 元数据写入磁盘
func (tx *Tx) writeMeta() error {
 // 开辟一个4KB大小的字节数组
 buf := make([]byte, tx.db.pageSize)
 // 将字节数组转为page,page和buf都是同一片空间，转成page，是为了
 // 方便向里面填充值
 p := tx.db.pageInBuffer(buf, 0)
 // 将DB.meta写入p(page)中
 tx.meta.write(p)

 // 将buf,也就是meta page写入到磁盘文件上
 if _, err := tx.db.ops.writeAt(buf, int64(p.id)*int64(tx.db.pageSize)); err != nil {
  return err
 }
 ...
}

```

复制

下面是真正meta中的数据转为page过程，处理非常简单，就是将m.txid赋值给p.id，设置p的flags属性，填充p的ptr内容。

```javascript
// 将meta中的内容输出到page中
func (m *meta) write(p *page) {
 ...
    
 // 轮流写入，一次写入meat page0，另一次写入meta page1
 p.id = pgid(m.txid % 2)
 // 设置flags为元数据页标识
 p.flags |= metaPageFlag

 // 计算数据校验和
 m.checksum = m.sum64()
 // 将m中的内容填充到p.ptr中
 m.copy(p.meta())
}
```

复制

## **DB.freelist与freelist page相互转换**

DB.freelist与freelist page的相互转换逻辑在freelist.go文件中，现在来看详细的转换处理流程。

将freelist page转换为DB.freelist的过程是通过`func (f *freelist) read(p *page)`实现的。处理过程就是根据page中字段值填充freelist，对应p.count有一种特殊情况，如果它的值为0xFFFF，这是一个特殊的标记，真正的数量存储在p.ptr的第一个8byte中。

```javascript
func (f *freelist) read(p *page) {
 // 获取page data中存储的pgid的数量，如果p.count的值为0xFFFF，这是一个特殊的标记
 // 真正的数量存储在p.ptr的第一个8byte中
 idx, count := 0, int(p.count)
 if count == 0xFFFF {
  idx = 1
  count = int(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0])
 }

 // count为0，说明db文件中没有空闲页，即没有任何page的浪费
 if count == 0 {
  f.ids = nil
 } else {
  // 获取所有的pgid值
  ids := ((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[idx:count]
  // 为f.ids开辟装载pgid的内存空间
  f.ids = make([]pgid, len(ids))
  // pgid值保存到f.ids中，缓存起来
  copy(f.ids, ids)

  // 将f.ids值按升序排序
  sort.Sort(pgids(f.ids))
 }

 // 将f.ids中的pgid缓存到f.cache中，f.cache是一个map类型，方便查找
 f.reindex()
}

```

复制

将DB.freelist转换为freelist page的过程，即freelist的序列化过程是在`func (f *freelist) write(p *page) error`实现的。处理过程非常简单，下面代码做了注解，相信每个同学都看得懂，这里就不在进一步说明了。

```javascript
func (f *freelist) write(p *page) error {
 // 设置flags为空闲列表页标识
 p.flags |= freelistPageFlag

 // 获取空闲的pgid数量
 lenids := f.count()
 if lenids == 0 {
  // 没有空闲页，p.count为0
  p.count = uint16(lenids)
 } else if lenids < 0xFFFF {
  // 空闲页的数量小于65535，数量直接保存在page的count中
  p.count = uint16(lenids)
  // 填充page的ptr
  f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[:])
 } else {
  // 空闲页的数量大于等于65535,将空闲页的数量保存在page的ptr的第一个8byte处
  p.count = 0xFFFF
  ((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0] = pgid(lenids)
  // 填充page的ptr,偏移8个字节开始
  f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[1:])
 }

 return nil
}
```



# Reference
https://blog.csdn.net/screscent/article/details/79807625?spm=1001.2014.3001.5502
https://cloud.tencent.com/developer/article/2072928?areaSource=&traceId=

https://cloud.tencent.com/developer/article/2072930?areaSource=&traceId=