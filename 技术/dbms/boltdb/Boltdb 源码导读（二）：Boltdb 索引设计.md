
> [boltdb](https://github.com/boltdb/bolt) 是市面上为数不多的纯 go 语言开发的、单机 KV 库。boltdb 基于 [Howard Chu’s](https://twitter.com/hyc_symas) [LMDB 项目](http://symas.com/mdb/) ，实现的比较清爽，去掉单元测试和适配代码，核心代码大概四千多行。简单的 API、简约的实现，也是作者的意图所在。由于作者精力所限，原 boltdb 已经封版，不再更新。若想改进，提交新的 pr，建议去 etcd 维护的 fork 版本 [bbolt](https://github.com/etcd-io/bbolt)。
> 
> 为了方便，本系列导读文章仍以不再变动的原 repo 为基础。该项目麻雀虽小，五脏俱全，仅仅四千多行代码，就实现了一个基于 B+ 树索引、支持一写多读事务的单机 KV 引擎。代码本身简约朴实、注释得当，如果你是 go 语言爱好者、如果对 KV 库感兴趣，那 boltdb 绝对是不可错过的一个 repo。
> 
> 本系列计划分成三篇文章，依次围绕**数据组织**、**索引设计**、**事务实现**等三个主要方面对 boltdb 源码进行剖析。由于三个方面不是完全正交解耦的，因此叙述时会不可避免的产生交织，读不懂时，暂时略过即可，待有全貌，再回来梳理。本文是第一篇， boltdb 数据组织。

# 概览

数据库中常用的索引设计有两种，一个是 B+ 树，一个是 LSM-tree。B+ 树比较经典，比如说传统单机数据库 mysql 就是 B+ 树索引，它对快速读取和范围查询（range query）比较友好。 LSM-tree 是近年来比较流行的索引结构，Bigtable、LevelDB、RocksDB 都有它的影子；[前面文章](https://www.qtmuniao.com/2020/11/18/leveldb-data-structures-bloom-filter/)也有提到，LSM-tree 使用 WAL 和多级数据组织以牺牲部分读性能，换来强悍的随机写性能。因此，这也是一个经典的取舍问题。

BoltDB 在逻辑上以桶来组织数据，一个桶可以看做一个命名空间，是一组 KV 对的集合，和对象存储的桶概念类似。每个桶对应一棵 B+ 树，命名空间是可以嵌套的，因此 BoltDB 的 Bucket 间也是允许嵌套的。在实现上来说，子 bucket 的 root node 的 page id 保存在父 bucket 叶子节点上实现嵌套。

每个 db 文件，是一组树形组织的 B+ 树。我们知道对于 B+ 树来说，分支节点用于查找，叶子节点存数据。

1.  顶层 B+ 树，比较特殊，称为 root bucket，其所有叶子节点保存的都是子 bucket B+ 树根的 page id 。
2.  其他 B+ 树，不妨称之为 data bucket，其叶子节点可能是正常用户数据，也可能是子 bucket B+ 树根的 page id。

![boltdb-buckets-organised.png](https://i.loli.net/2021/01/02/QBwKIpNOHkRqzrE.png)

相比普通 B+ 树，boltdb 的 B+ 树有几点特殊之处：

2.  节点的分支个数不是一个固定范围，而是依据其所存元素大小之和来限制的，这个上限即为页大小。
3.  其分支节点（branch node）所存分支的 key，是其所指向分支的最小 key。
4.  所有叶子节点并没有通过链表首尾相接起来。
5.  没有保证所有的叶子节点都在同一层。

在代码组织上，boltdb 索引相关的源文件如下：

1.  **bucket.go**：对 bucket 操作的高层封装。包括 kv 的增删改查、子 bucket 的增删改查以及 B+ 树拆分和合并。
2.  **node.go**：对 node 所存元素和 node 间关系的相关操作。节点内所存元素的增删、加载和落盘，访问孩子兄弟元素、拆分与合并的详细逻辑。
3.  **cursor.go**：实现了类似迭代器的功能，可以在 B+ 树上的叶子节点上进行随意游走。

本文主要分三部分，由局部到整体来一步步揭示 BoltDB 是如何进行索引设计的。首先会拆解树的基本单元，其次剖析 bucket 的遍历实现，最后分析树的生长和平衡过程。

_作者：木鸟杂记 [https://www.qtmuniao.com/2020/12/14/bolt-index-design](https://www.qtmuniao.com/2020/12/14/bolt-index-design), 转载请注明出处_

#  基本单元 —— 节点（Node）

B+ 树的基本构成单元是节点（node），对应在[上一篇](https://www.qtmuniao.com/2020/11/29/bolt-data-organised/)中提到过文件系统中存储的页（page），节点包括两种类型，分支节点（branch node）和叶子节点（leaf node）。但在实现时，他们复用了同一个结构体，并通过一个标志位 `isLeaf` 来区分：

```go
// node 表示内存中一个反序列化后的 page  
type node struct {  
  bucket     *Bucket   // 其所在 bucket 的指针  
  isLeaf     bool      // 是否为叶子节点  
    
  // 调整、维持 B+ 树时使用  
  unbalanced bool      // 是否需要进行合并  
  spilled    bool      // 是否需要进行拆分和落盘  
    
  key        []byte    // 所含第一个元素的 key  
  pgid       pgid      // 对应的 page 的 id  
    
  parent     *node     // 父节点指针  
  children   nodes     // 孩子节点指针（只包含加载到内存中的部分孩子）  
    
  inodes     inodes    // 所存元素的元信息；对于分支节点是 key+pgid 数组，对于叶子节点是 kv 数组  
}  
```

##  node/page 转换

page 和 node 的对应关系为：文件系统中一组连续的物理 page，加载到内存成为一个逻辑 page ，进而转化为一个 node。下图为一个在文件系统上占用两个 pagesize 空间的一段连续数据 ，首先 mmap 到内存空间，转换为 `page` 类型，然后通过 `node.read` 转换为 `node` 的过程。

![boltdb-node-load-and-dump.png](https://i.loli.net/2021/01/02/N7OqmibTklP5DLg.png)

其中 `node.read` 将 page 转换为 node 的代码如下：

```go
// read 函数通过 mmap 读取 page，并转换为 node  
func (n *node) read(p *page) {  
  // 初始化元信息  
  n.pgid = p.id  
  n.isLeaf = ((p.flags & leafPageFlag) != 0)  
  n.inodes = make(inodes, int(p.count))  
  
  // 加载所包含元素 inodes  
  for i := 0; i < int(p.count); i++ {  
    inode := &n.inodes[i]  
    if n.isLeaf {  
      elem := p.leafPageElement(uint16(i))  
      inode.flags = elem.flags  
      inode.key = elem.key()  
      inode.value = elem.value()  
    } else {  
      elem := p.branchPageElement(uint16(i))  
      inode.pgid = elem.pgid  
      inode.key = elem.key()  
    }  
    _assert(len(inode.key) > 0, "read: zero-length inode key")  
  }  
    
  // 用第一个元素的 key 作为该 node 的 key，以便父节点以此作为索引进行查找和路由  
  if len(n.inodes) > 0 {  
    n.key = n.inodes[0].key  
    _assert(len(n.key) > 0, "read: zero-length node key")  
  } else {  
    n.key = nil  
  }  
}  
```

## node 元素 inode

`inode` 表示 node 所含的内部元素，分支节点和叶子节点也复用同一个结构体。对于分支节点，单个元素为 key + 引用；对于叶子节点，单个元素为用户 kv 数据。

注意到这里对其他节点的引用类型为 `pgid` ，而非 `node*`。这是因为 `inode` 中指向的元素并不一定加载到了内存。 boltdb 在访问 B+ 树时，会按需将访问到的 page 转化为 node，并将其指针存在父节点的 `children` 字段中， 具体的加载顺序和逻辑在后面小结会详述。

```go
type inode struct {  
  // 共有变量  
  flags uint32  
  key   []byte  
    
  // 分支节点使用  
  pgid  pgid   // 指向的分支/叶子节点的 page id  
  // 叶子节点使用  
  value []byte // 叶子节点所存的数据  
}  
```

`inode` 会在 B+ 树中进行路由 —— 二分查找时使用。

##  新增元素

所有的数据新增都发生在叶子节点，如果新增数据后 B+ 树不平衡，之后会通过 `node.spill` 来进行拆分调整。主要代码如下：

```go
// put inserts a key/value.  
func (n *node) put(oldKey, newKey, value []byte, pgid pgid, flags uint32) {  
  // 找到插入点  
  index := sort.Search(len(n.inodes), func(i int) bool { return bytes.Compare(n.inodes[i].key, oldKey) != -1 })  
    
  // 如果 key 是新增而非替换，则需为待插入节点腾空儿  
  exact := (len(n.inodes) > 0 && index < len(n.inodes) && bytes.Equal(n.inodes[index].key, oldKey))  
  if !exact {  
    n.inodes = append(n.inodes, inode{})  
    copy(n.inodes[index+1:], n.inodes[index:])  
  }  
  
  // 给要替换/插入的元素赋值  
  inode := &n.inodes[index]  
  inode.key = newKey  
  // ...  
}  
```

该函数的主干逻辑比较简单，就是二分查找到待插入位置，如果是覆盖写则直接覆盖；否则就要新增一个元素，并整体右移，腾出插入位置。 但是该函数签名很有意思，同时有 `oldKey` 和 `newKey` ，开始时感觉很奇怪。其调用代码有两处：

1.  在叶子节点插入用户数据时，`oldKey` 等于 `newKey`，此时这两个参数是有冗余的。
2.  在 `spill` 阶段调整 B+ 树时，`oldKey` 可能不等于 `newKey`，此时是有用的，但从代码上来看，用处仍然很隐晦。

在和朋友讨论后，大致得出如下结论：为了避免在叶子节点最左侧插入一个很小的值时，引起祖先节点的 `node.key` 的链式更新，而将更新延迟到了最后 B+ 树调整阶段（`spill` 函数）进行统一处理 。此时，需要利用 `node.put` 函数将最左边的 `node.key` 更新为 `node.inodes[0].key`：

```go
// 该代码片段在 node.spill 函数中  
if node.parent != nil { // 如果不是根节点  
  var key = node.key // split 后，最左边 node  
  if key == nil {    // split 后，非最左边 node  
    key = node.inodes[0].key  
  }  
  
  node.parent.put(key, node.inodes[0].key, nil, node.pgid, 0)  
  node.key = node.inodes[0].key  
}  
```

![bolt-recursive-change.png](https://i.loli.net/2021/01/02/woJQh7ZSOr3yGWq.png)

#  节点遍历

由于 Golang 不支持 Python 中类似 `yield` 机制，boltdb 使用栈保存遍历上下文实现了一个树节点**顺序遍历**的迭代器：`cursor`。在逻辑上可以理解为对某 B+ 树**叶子节点所存元素遍历**的迭代器。之前提到，boltdb 的 B+ 树没有使用链表将所有叶子节点串起来，因此需要一些额外逻辑来进行遍历中各种细节的处理。

这么实现增加了遍历的复杂度，但是减少了维持 B+ 树平衡性质的难度，也是一种取舍。当然，最重要的是为了通过 COW 实现事务时，避免链式更新所有前驱节点。

![cursor-implementation.png](https://i.loli.net/2021/01/02/m4WE7fgn8OazSND.png)

`cursor` 和某个 bucket 绑定，实现了以下功能，需要注意，当遍历到的元素为嵌套 bucket 时，`value = nil`。

```go
type Cursor  
func (c *Cursor) Bucket() *Bucket // 返回绑定的 bucket  
func (c *Cursor) Delete() error // 删除 cursor 指向的 key  
func (c *Cursor) First() (key []byte, value []byte) // 定位到并返回该 bucket 第一个 KV   
func (c *Cursor) Last() (key []byte, value []byte)  // 定位到并返回该 bucket 最后一个 KV   
func (c *Cursor) Next() (key []byte, value []byte)  // 定位到并返回该 bucket 下一个 KV  
func (c *Cursor) Prev() (key []byte, value []byte)  // 定位到并返回该 bucket 前一个 KV  
func (c *Cursor) Seek(seek []byte) (key []byte, value []byte) // 定位到并返回该 bucket 内指定的 KV  
```

由于 boltdb 中 B+ 树左右叶子节点并没有通过链表串起来，因此遍历时需要记下遍历**路径**以进行回溯。其结构体如下：

```go
type Cursor struct {  
  bucket *Bucket    // 使用该句柄来进行 node 的加载  
  stack  []elemRef  // 保留路径，方便回溯  
}  
```

`elemRef` 结构体如下，page 和 node 是一一对应的，如果 page 加载到了内存中（通过 page 转换而来），则优先使用 node，否则使用 page；index 表示路径经过该节点时在 `inodes` 中的位置。

```go
type elemRef struct {  
  page  *page  
  node  *node  
  index int  
}  
```

##  辅助函数

这几个 API 在实现的时候是有一些通用逻辑可以进行复用的，因此可以作为构件拆解出来。其中**移动 cursor** 在实现上，就是调整 `cursor.stack` 数组所表示的路径。

```go
// 尾递归，查询 key 所在的 node，并且在 cursor 中记下路径  
func (c *Cursor) search(key []byte, pgid pgid)  
  
// 借助 search，查询 key 对应的 value  
// 如果 key 不存在，则返回待插入位置的 kv 对：  
//   1. ref.index < ref.node.count() 时，则返回第一个比给定 key 大的 kv  
//   2. ref.index == ref.node.count() 时，则返回 nil  
// 上层 Seek 需要处理第二种情况。  
func (c *Cursor) seek(seek []byte) (key []byte, value []byte, flags uint32)  
  
// 移动 cursor 到以栈顶元素为根的子树中最左边的叶子节点  
func (c *Cursor) first()  
// 移动 cursor 到以栈顶元素为根的子树中最右边的叶子节点  
func (c *Cursor) last()  
  
// 移动 cursor 到下一个叶子元素  
//   1. 如果当前叶子节点后面还有元素，则直接返回  
//   2. 否则需要借助保存的路径找到下一个非空叶子节点  
//   3. 如果当前已经指向最后一个叶子节点的最后一个元素，则返回 nil  
func (c *Cursor) next() (key []byte, value []byte, flags uint32)  
```

##  组合遍历

有了以上几个基本构件，我们来梳一下主要 API 函数的实现： 

```go
// 1. 将根节点放入 stack 中  
// 2. 调用 c.first() 定位到根的第一个叶子节点  
// 3. 如果该节点为空，则调用 c.next() 找到下一个非空叶子节点  
// 4. 返回其该叶子节点第一个元素  
func (c *Cursor) First() (key []byte, value []byte)  
  
// 1. 将根节点放入 stack 中  
// 2. 调用 c.last() 定位到根的最后一个叶子节点  
// 3. 返回其最后一个元素，不存在则返回空  
func (c *Cursor) Last() (key []byte, value []byte)   
  
// 1. 直接调用 c.next 即可  
func (c *Cursor) Next() (key []byte, value []byte)  
  
// 1. 遍历 stack，回溯到第一个有前驱元素的分支节点  
// 2. 将节点的 ref.index--  
// 3. 调用 c.last()，定位到该子树的最后一个叶子节点  
// 4. 返回其最后一个元素  
func (c *Cursor) Prev() (key []byte, value []byte)  
  
// 1. 调用 c.seek()，找到第一个 >= key 的节点中元素。  
// 2. 如果 key 正好落在两个叶子节点中间，调用 c.next() 找到下一个非空节点  
func (c *Cursor) Seek(seek []byte) (key []byte, value []byte) // 定位到并返回该 bucket 内指定的 KV  
```

这些 API 的实现叙述起来稍显繁琐，但只要抓住其主要思想，并且把握一些边界和细节，便能很容易看懂。**主要思想**比较简单： cursor 最终目的是在所有叶子节点的元素进行遍历，但是叶子节点并没有通过链表串起来，因此需要借助一个 stack 数组记下遍历上下文 —— 路径，来实现对前驱后继的快速（\_因为前驱后继与当前叶子节点大概率共享前缀路径_）访问。

另外需要注意的一些**边界和细节**如下：

1.  每次移动，需要先找节点，再找节点中元素。
2.  如果节点已经转换为 node，则优先访问 node；否则，访问 mmap 出来的 page。
3.  分支节点所记元素的 key 为其指向的节点的 key，也即其节点所包含元素的最小 key。
4.  使用 Golang 的 `sort.Search` 获取第一个小于给定 key 的元素下标需要做一些额外处理。
5.  几个边界判断，node 中是否有元素、index 是否大于元素值、该元素是否为子 bucket。
6.  如果 key 不存在时，seek/search 定位到的是 key 应当插入的点。

#  树的生长

我们分几个时间节点来展开说明下 boltdb 中 B+ 树的生命周期：

1.  数据库初始化时
2.  事务开启后
3.  事务提交时

最后在理解这几个阶段的状态的基础上，整个串一下其生长过程。

##  初始化时

数据库初始化时，B+ 树只包含一个空的叶子节点，该叶子节点即为 root bucket 的 root node。

```go
func (db *DB) init() error {  
  // ...  
  // 在 pageid = 3 的地方写入一个空的叶子节点.  
  p = db.pageInBuffer(buf[:], pgid(3))  
  p.id = pgid(3)  
  p.flags = leafPageFlag  
  p.count = 0  
  // ...  
}  
```

之后 B+ 树的生长都由**写事务**中对 B+ 树节点的增删、调整来完成。按 boltdb 的设计，写事务只能串行进行。boltdb 使用 COW 的方式对节点进行修改，以保证不影响并发的读事务。即，将要修改的 page 读到内存，修改并调整后，申请新的 page 将变动后的 node 落盘。

这种方式可以方便的实现读写并发和事务，在本系列文章下一篇中将详细分析其原因。但无疑，其代价比较高昂，即使一个 key 的修改 / 删除，都会引起对应叶子节点所在 B+ 树路径上所有节点的修改和落盘。因此如果修改较频繁，最好在单个事务中做 Batch。

##  Bucket 较小时

在 bucket 包含的数据还很少时，不会给 bucket 开辟新的 page，而是将其**内嵌**（inline）在父 bucket 的叶子节点中。是否能内嵌的判断逻辑在 `bucket.inlineable` 中：

1.  只包含一个叶子节点
2.  数据尺寸不大于 1/4 个页
3.  不包含子 bucket

## 事务开启后

在每次事务**初始化时**，会在内存中拷贝一份 root bucket 的句柄，以此作为之后动态加载修改路径上 node 的入口。

```go
func (tx *Tx) init(db *DB) {  
  //...  
    
  // 拷贝并初始化 root bucket  
  tx.root = newBucket(tx)  
  tx.root.bucket = &bucket{}  
  *tx.root.bucket = tx.meta.root  
    
  // ...   
}  
```

在_读写事务_中，用户调用 `bucket.Put` 新增或者修改数据时，涉及两个阶段：

1.  **查找定位**：利用 cursor 定位到指定 key 所在 page
2.  **加载插入**：加载路径上所有节点，并在叶子节点插入 kv

```go
func (b *Bucket) Put(key []byte, value []byte) error {  
  // ...  
  
  // 1. 将 cursor 定位到 key 所在位置  
  c := b.Cursor()  
  k, _, flags := c.seek(key)  
  
  // ...  
  
  // 2. 加载路径上节点为 node，然后插入 key value  
  key = cloneBytes(key)  
  c.node().put(key, key, value, 0, 0)  
  
  return nil  
}  
```

**查找定位**。该实现逻辑在上一节所探讨的 cursor 的 `cursor.seek` 中，其主要逻辑为从根节点出发，不断二分查找，访问对应 node/page，直到定位到待插入 kv 的叶子节点。这里面有个关键的节点访问函数 `bucket.pageNode`，该函数不会加载 page 为 node，而只是复用已经缓存的 node ，或者直接访问 mmap 到内存空间中的相关 page。

**加载插入**。在该阶段，首先通过 `cursor.node()` 将 cursor 栈中保存的所有节点索引加载为 `node`，并缓存起来，避免重复加载以进行复用；然后通过 `node.put` 通过二分查找将 key value 数据插入叶子 `node`。

在_只读事务_中，只会有查找定位的过程，因此只会通过 mmap 对 `page` 访问，而不会有 `page` 到 `node` 的转换过程。

对于 `bucket.Delete` 操作，和上述两个阶段类似，只不过**加载插入**变成了**加载删除**。可以看出，这个阶段所有的修改都发生在内存中，文件系统中保存的之前 page 组成的 B+ 树结构并未遭到破坏，因此读事务可以并发进行。

## 事务提交前

在写事务开启后，用户可能会进行一系列的新增 / 删除，大量的相关节点被转化为 node 加载到内存中，改动后的 B+ 树由文件系统中的 page 和内存中的 node 共同构成，且由于数据变动，可能会造成某些节点元素数量过少、而另外一些节点元素数量过多。

因此在事务提交前，会先按一定策略调整 B+ 树，使其维持较好的查询性质，然后将所有改动的 `node` 序列化为 `page` 增量的写入文件系统中，构成一棵新的、持久化的、平衡的 B+ 树。

这个阶段涉及到两个核心逻辑：`bucket.rebalance` 和 `bucket.spill`， 他们分别对应节点的 merge 和 split，是维持 boltdb B+ 树查找性质的关键函数，下面来分别梳理下。

**rebalance**。该函数旨在将过小（key 数太少或者总体尺寸太小）的节点合并到邻居节点上。调整的主要逻辑在 `node.rebalance` 中， `bucket.rebalance` 主要是个外包装：

```go
func (b *Bucket) rebalance() {  
  // 对所有缓存的 node 进行调整  
  for _, n := range b.nodes {  
    n.rebalance()  
  }  
    
  // 对所有子 bucket 进行调整  
  for _, child := range b.buckets {  
    child.rebalance()  
  }  
}  
```

`node.rebalance` 主要逻辑如下：

1.  判断 `n.unbalanced` 标记，避免对某个节点进行重复调整。使用标记的原因有二，一是按需调整，二是避免重复调整。
2.  判断该节点是需要 merge，判断标准为：`n.size() > pageSize / 4 && len(n.inodes) > n.minKeys()` ，不需要则结束。
3.  如果该节点是根节点，且只有一个孩子节点，则将其和其唯一的孩子合并。
4.  如果该节点没有孩子节点，则删除该节点。
5.  否则，将该节点合并到左邻。如果该节点是第一个节点，没左邻，则将右邻合并过来。
6.  由于此次调整可能会导致节点的删除，因此向上递归看是否需要进一步调整。

需要明确的是，只有 `node.del()` 的调用才会导致 `n.unbalanced` 被标记。只有两个地方会调用 `node.del()`:

1.  用户调用 `bucket.Delete` 函数删除数据。
2.  子节点调用 `node.rebalance` 进行调整时，删除被合并的节点。

而 2 又是由 1 引起的，因此可以说，只有用户在某次写事务中删除数据时，才会引起 `node.rebanlance` 主逻辑的实际执行。

**spill**，该函数功能有二，将过大（尺寸大于一个 page）节点拆分、将节点写入脏页（dirty page）。与 rebalance 一样，主要逻辑在 `node.spill` 中。不同的是，`bucket.spill` 中也有相当一部分逻辑，主要是处理 inline bucket 的问题。前面提到，如果一个 bucket 内容过少，就会直接内嵌在父 bucket 的叶子节点中。否则，则先调用子 bucket 的 spill 函数，然后将子 bucket 的根节点 pageid 放在父 bucket 叶子节点中。可以看出， spill 调整是一个自下而上的过程，类似于树的后序遍历。

```go
// spill 将尺寸大于一个节点的 page 拆分，并将调整后的节点写入脏页  
func (b *Bucket) spill() error {  
    
  // 自下而上，先递归调整子 bucket  
  for name, child := range b.buckets {  
    // 如果子 bucket 可以内嵌，则将其所有数据序列化后内嵌到父 bucket 相应叶子节点  
    var value []byte  
    if child.inlineable() {  
      child.free()  
      value = child.write()  
    // 否则，先调整子 bucket，然后将其根节点 page id 作为值写入父 bucket 相应叶子节点  
    } else {  
      if err := child.spill(); err != nil {  
        return err  
      }  
  
      value = make([]byte, unsafe.Sizeof(bucket{}))  
      var bucket = (*bucket)(unsafe.Pointer(&value[0]))  
      *bucket = *child.bucket  
    }  
  
    // 如果该子 bucket 没有缓存任何 node（说明没有数据变动），则直接跳过  
    if child.rootNode == nil {  
      continue  
    }  
  
    // 更新 child 父 bucket（即本 bucket）的对该子 bucket 的引用  
    var c = b.Cursor()  
    k, _, flags := c.seek([]byte(name))  
    if !bytes.Equal([]byte(name), k) {  
      panic(fmt.Sprintf("misplaced bucket header: %x -> %x", []byte(name), k))  
    }  
    if flags&bucketLeafFlag == 0 {  
      panic(fmt.Sprintf("unexpected bucket header flag: %x", flags))  
    }  
    c.node().put([]byte(name), []byte(name), value, 0, bucketLeafFlag)  
  }  
  
  // 如果该 bucket 没有缓存任何 node（说明没有数据变动），则终止调整  
  if b.rootNode == nil {  
    return nil  
  }  
  
  // 调整本 bucket  
  if err := b.rootNode.spill(); err != nil {  
    return err  
  }  
  b.rootNode = b.rootNode.root()  
  
  // 由于调整会增量写，造成本 bucket 根节点引用变更，因此需要更新 b.root  
  if b.rootNode.pgid >= b.tx.meta.pgid {  
    panic(fmt.Sprintf("pgid (%d) above high water mark (%d)", b.rootNode.pgid, b.tx.meta.pgid))  
  }  
  b.root = b.rootNode.pgid  
  
  return nil  
}  
```

`node.spill` 的主要逻辑如下：

1.  判断 `n.spilled` 标记，默认为 false，表明所有节点都需要调整。如果调整过，则跳过。
2.  由于是自下而上调整，因此需要递归调用以先调整子节点，再调节本节点。
3.  调整本节点时，将节点按照 pagesize 进行拆分。
4.  为所有新节点申请新的合适尺寸的 pages，然后将 node 写入 page（此时还没有写回文件系统）。
5.  如果 spilt 生成了新的根节点，则需要向上递归调用调整其他分支。为了避免重复调整，会将本节点 `children` 置空。

可以看出，boltdb 维持 B+ 树查找性质，并非像教科书 B+ 树一样，将所有分支节点的分支树维护在一个固定范围，而是直接按节点元素是否能够保存到一个 page 中来做的。这样做可以减少 page 内部碎片，实现也相对简单。

这两个函数都通过 put/delete 后标记来实现按需调整，但不同的是，rebalance 先 merge 本身，再 merge 子 bucket；而 spill 先 split 子 bucket，再 split 本身。另外，调整时对他们调用顺序是有要求的，需要先调用 balance 进行无脑 merge，然后在调用 spill，按 pagesize 进行拆分后，写入脏页。

总结一下， 在 db 初始化时，只有一个页保存 root bucket 的根节点。之后的 B+ 树在 `bucket.Create` 的时候进行创建。初始时内嵌在父 bucket 的叶子节点中，读事务不会对 B+ 树结构造成任何改变，写事务中所有变动，会先写到内存中，在事务提交时，会进行平衡调整，然后增量的写入文件系统。随着写入数据的增多，B+ 树会不断进行拆分，变深，不在内嵌于父 bucket 中。

# 小结

boltdb 使用类 B+ 树组织数据库索引，所有数据存在叶子节点，分支节点只用于路由查找。boltdb 支持 bucket 间的嵌套，在实现上表现为 B+ 树的嵌套，通过 page id 来维持父子 bucket 间的引用。

boltdb 中的 B+ 树为了实现简单，没有使用链表将所有叶子节点串在一起。为了支持对数据的顺序遍历，额外实现了一个 curosr 遍历逻辑，通过保存遍历栈来提高遍历效率、快速跳转。

boltdb 对 B+ 树的生长以事务为周期，而且生长只发生在写事务中。在写事务开始后，会复制 root bucket 的根节点，然后将改动涉及到的节点按需加载到内存，并在内存中进行修改。在写事务结束前，在对 B+ 树调整后，将所有改动涉及到的 node 申请新的 page，写回文件系统，完成 B+ 树一次生长。释放的树节点，在没有读事务占用后，会进入 freelist 供之后使用。

# 注解

**节点**：B+ 树中的节点，在文件系统或者 mmap 后表现为 page，在内存中转换后成为 node。

**路径**：树中的路径指树的根节点到当前节点的顺序经过的所有节点。


# boltdb源码分析系列-Bucket

发布于2022-08-15 15:03:25阅读 1870

## **Bucket是什么**

Bucket翻译成中文为桶，正如其名，桶里面可以放东西。本文要介绍的Bucket是boltdb中的桶，它是一个数据结构。boltdb中每个Bucket是一颗B+树，它里面存放一批key-value数据对。

类比[关系型数据库](https://cloud.tencent.com/product/cdb-overview?from=20065&from_column=20065)，每个Bucket可以看做关系型[数据库](https://cloud.tencent.com/solution/database?from=20065&from_column=20065)中的table。一个boltdb中可以有多个Bucket, Bucket中可以嵌套Bucket.

在[boltdb源码分析系列-内存结构](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488539&idx=1&sn=3215a13742e9c2e2eff185bb33108a55&chksm=873a6ddab04de4cc7a4558ebb4308f21839fab7a7967a6ca85402991e701235bb0f746bc48c6&scene=21#wechat_redirect)文章中介绍了boltdb内存结构node，一个Bucket可以看做node的集合。总结起来，Bucket有如下要点：

-   每个Bucket是一颗B+Tree
-   一个Bucket中存储了一批key-value数据
-   Bucket类比做关系型数据库中的table概念
-   Bucket可以看做node的集合
-   Bucket中可以嵌套Bucket

#### **Bucket结构体定义**

Bucket结构中各个字段含义如下，关键的字段有*bucket和rootNode,它们描述的是的Bucket对应B+Tree的树根信息。

rootNode是这颗B+Tree的根节点node. 只要知道了树的根节点，就可以从根节点遍历获取所有的其他节点。在[boltdb源码分析系列-内存结构](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488539&idx=1&sn=3215a13742e9c2e2eff185bb33108a55&chksm=873a6ddab04de4cc7a4558ebb4308f21839fab7a7967a6ca85402991e701235bb0f746bc48c6&scene=21#wechat_redirect)文章中node结构定义中可以看到，它是一个递归的定义，node节点中的children记录它的孩子节点。所以Bucket只要记录根节点，便拥有了整颗树的信息。

*bucket是一个内嵌结构，它里面的核心字段是root，root是该Bucket所在page的id值。page描述的是boltdb的文件结构，即物理存储；node描述的是boltdb的内存结构，即逻辑结构。Bucket结构体中上面的两个字段分别从物理和逻辑层面描述了boltdb信息。

```javascript
type Bucket struct {
 // B+树的根节点信息,主要是根节点的pgid
 *bucket
 // 操作Bucket的事务句柄
 tx       *Tx                
 // 子Bucket
 buckets  map[string]*Bucket 
 // 内联page
 page     *page              
 // Bucket管理的树的根节点
 rootNode *node              
 // node缓存
 nodes    map[pgid]*node    
    // 填充率
 FillPercent float64
}
```

复制

这里再对Bucket结构体中其他字段做一个详细说明：

-   tx: 当前Bucket所属的事务
-   page: 内联Bucket的页引用，内置Bucket只有一个节点，即它的根节点，并且节点不存在独立的页面中，而是作为Bucket的value存在父Bucket所在页上，页引用是指向value中的内置页
-   nodes: Bucket中的node集合
-   FillPercent: Bucket中节点的填充百分比，该值与B+Tree的分裂有关，当节点中key的个数或者占用的空间大小超过整个node容量的某个值之后，节点必须分裂为两个节点
-   buckets是子Bucket的集合，因为Bucket中可以嵌套Bucket，所以需要一个字段记录子Bucket信息，这里通过一个map来记录，map的key为子Bucket的桶名，value为子Bucket对象指针。
-   内联page是指创建Bucket的时候，没有为它申请新的page来存储它，而是将它的信息存储在它的父Bucket中的叶子page中。内联的意义是，如果Bucket中存储key-value数据不大，将其内容存放在父Bucket的leaf page中，这样能够提升page的填充率，减少空间的浪费。
-   nodes缓存的是可能有影响的节点信息，当我们向Bucket中写入数据、删除数据或者更新数据的时候，并不是直接更新boltdb文件，而是更新它在内存中的node信息。因为所有对key-value的操作都是发生在叶子节点上。所以叶子节点会缓存到nodes，从根节点到叶子节点路径上的节点可能也会受到影响，它们也会缓存到nodes。这些nodes在事务提交的时候，会转换成dirty page，最后刷到磁盘上。

#### **bucket与node关系**

每个db文件，是一组树形组织的B+树。对于B+树来说，分支节点（branch node）用于查找，叶子节点(leaf node)存数据。

每个node都会属于某个桶，所以在node的结构体定义中有一个指向它所属桶的bucket字段。向boltdb中写入数据、修改数据或是删除数据，都是通过调用Bucket的方法，因为在逻辑上，Bucket是承载数据的地方。

通过Bucket查询key的value值

```javascript
// 在桶中查找键值对为key的value
func (b *Bucket) Get(key []byte) []byte {
 ...
}
```

复制

通过Bucket写入key-value数据

```javascript
func (b *Bucket) Put(key []byte, value []byte) error {
 ...
}
```

复制

通过Bucket删除key-value数据

```javascript
func (b *Bucket) Delete(key []byte) error {
 ...
}
```

复制

node结构体中记录有孩子节点的信息，所以node可以组成一颗树形结构，而Bucket只要记录了B+Tree的root node，相当于掌控了整颗B+树,所以Bucket结构中有一个rootNode字段，记录的是根节点信息。一个bolt db文件可以创建多个Bucket,并且Bucket可以嵌套，而每个Bucket是一颗B+Tree, 所以一个bolt db文件相当于多个B+Tree的集合。多个Bucket也需要一个伪根Bucket记录它们的信息，这个根Bucket就是tx.root，本文称之为根Bucket, 剩下的Bucket称之为普通Bucket.

经过前面的分析，将Bucket分为了根Bucket和普通Bucket两类。根Bucket所有叶子节点保存的都是子Bucket B+树根的page id.普通Bucket叶子节点可能是正常用户数据，也可能是子Bucket B+树根的page id.

根Bucket与普通Bucket关系如下,根Bucket即tx.root对外是不可见的。普通Bucket是用户程序可见的。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/8b21996de2b9f9a3cf0b56fb081f4a1d.jpeg?imageView2/2/w/2560/h/7000)

Bucket与node关系如下，下图中有4个Bucket,一个是根Bucket,其他3个都是普通Bucket. Bucket3是Bucket2的子Bucket.它们形成父子关系，从而所有Bucket形成树结构，通过根Bucket可以遍历所有子Bucket，但是注意，Bucket之间的树结构并不是B+Tree，而是一个逻辑树结构，如Bucket3是Bucket2的子Bucket，但并不是说Bucket3所在的节点就是Bucket2所在节点的子节点。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/f011839b47221be0837fed8865f2426f.jpeg?imageView2/2/w/2560/h/7000)

## **Bucket核心方法及实现**

![](https://ask.qcloudimg.com/http-save/yehe-9955853/c30b073da884bd013bee5d39a3e397e1.jpeg?imageView2/2/w/2560/h/7000)

#### **构造函数**

返回一个Bucket对象，默认设置了Bucket填充率为50%，如果是读写事务，初始化两个map,它们分别记录子Bucket和Bucket中的node信息。

```javascript
// Bucket构造函数
func newBucket(tx *Tx) Bucket {
 var b = Bucket{tx: tx, FillPercent: DefaultFillPercent}
 if tx.writable {
  b.buckets = make(map[string]*Bucket)
  b.nodes = make(map[pgid]*node)
 }
 return b
}
```

复制

#### **在桶中查找给定名称的Bucket**

查找Bucket过程如下：

1.  先查找缓存，Bucket对象中的map会记录它里面子Bucket信息，如果缓存中有，直接返回
2.  遍历Bucket查找，需要先创建一个Bucket迭代器(Cursor),定位到叶子节点，然后通过key判断Bucket是否存在，以及是否是Bucket类型
3.  创建一个Bucket对象缓存起来并返回

```javascript
// 在Bucket中查找给定名称的bucket
func (b *Bucket) Bucket(name []byte) *Bucket {
 // 先从缓存中查找
 if b.buckets != nil {
  // 如果缓存中有，直接取缓存中的返回
  if child := b.buckets[string(name)]; child != nil {
   return child
  }
 }

 // 创建一个游标对象，用于对bucket进行遍历
 c := b.Cursor()
 // 在B+树中查找key为name的节点
 k, v, flags := c.seek(name)

 // 如果不存在或者不是一个bucket节点
 if !bytes.Equal(name, k) || (flags&bucketLeafFlag) == 0 {
  return nil
 }

 // 走到这里，说明找到了给定名称的桶，然后创建一个Bucket
 var child = b.openBucket(v)
 if b.buckets != nil {
  // 缓存起来
  b.buckets[string(name)] = child
 }

 return child
}
```

复制

#### **创建桶**

创建桶的处理过程比较复杂：

1.  对异常情况进行判断，包括db是否已关闭，是否是读写事务，桶的名称是否合法
2.  检查桶是否已经存在，创建一个Bucket迭代器，对Bucket进行遍历，查找Bucket是否存在，如果桶已经存在，返回错误
3.  桶不存在，创建一个Bucket, 加入迭代器游标位置，其中key是子Bucket的名字，value是子Bucket的序列化结果
4.  将当前Bucket的page字段置空，因为当前Bucket包含了刚创建的子Bucket，它不会是内置Bucket
5.  通过b.Bucket()方法按子Bucket的名字查找子Bucket并返回结果,为啥不直接返回上面的bucket呢？因为要设置bucket.root值和bucket.page

```javascript
// 创建一个桶，如果名称已存在，则会返回错误
func (b *Bucket) CreateBucket(key []byte) (*Bucket, error) {
 // 异常情况处理：1 db已关闭   2 非读写事务，不支持创建桶  3 桶的名称为空
 if b.tx.db == nil {
  return nil, ErrTxClosed
 } else if !b.tx.writable {
  return nil, ErrTxNotWritable
 } else if len(key) == 0 {
  return nil, ErrBucketNameRequired
 }

 // 创建一个游标对象
 c := b.Cursor()
 // 在B+中查找key
 k, _, flags := c.seek(key)

 // key已存在
 if bytes.Equal(key, k) {
  // key已经存在并且是个桶
  if (flags & bucketLeafFlag) != 0 {
   return nil, ErrBucketExists
  }
  // 桶的名称不能和数据中的key重名
  return nil, ErrIncompatibleValue
 }

 // 创建一个新的bucket
 var bucket = Bucket{
  bucket:      &bucket{},
  rootNode:    &node{isLeaf: true},
  FillPercent: DefaultFillPercent,
 }
 var value = bucket.write()

 // 对key进行深度拷贝，传入的key是用户空间的，防止用户后续修改key
 key = cloneBytes(key)
 // 将bucket加入到节点c.node()中
 c.node().put(key, key, value, 0, bucketLeafFlag)

 // 当前桶中已有子桶，不能在被内联
 b.page = nil
 // 为啥不直接返回上面的bucket呢？主要是要设置bucket.root的值
 return b.Bucket(key), nil
}

```

复制

#### **删除桶**

删除桶先检查桶是否存在，如果桶存在，需要递归将要删除桶中包含的子桶信息删除，然后才能删除，并且需要释放待删除桶关联的page,放入freelist page中。

```javascript
func (b *Bucket) DeleteBucket(key []byte) error {
 // 异常情况检查
 if b.tx.db == nil {
  return ErrTxClosed
 } else if !b.Writable() {
  return ErrTxNotWritable
 }

 // 创建一个迭代器对桶进行遍历查找给定的key
 c := b.Cursor()
 k, _, flags := c.seek(key)

 // 桶不存在
 if !bytes.Equal(key, k) {
  return ErrBucketNotFound
 } else if (flags & bucketLeafFlag) == 0 {
  // 存在给定名称的key,但它不是桶的名字
  return ErrIncompatibleValue
 }

 // 递归删除即将要删除桶的所有的子bucket
 child := b.Bucket(key)
 err := child.ForEach(func(k, v []byte) error {
  if v == nil {
   if err := child.DeleteBucket(k); err != nil {
    return fmt.Errorf("delete bucket: %s", err)
   }
  }
  return nil
 })
 if err != nil {
  return err
 }

 // 删除缓存map中桶
 delete(b.buckets, string(key))

 child.nodes = nil
 child.rootNode = nil
 // 释放桶关联的page
 child.free()

 // 从叶子节点c中删除该桶的key
 c.node().del(key)

 return nil
}
```

复制

#### **查找桶中数据**

查找桶中数据，只是在当前桶中查找，并不会递归查找子桶，整个查找过程是通过迭代器完成的，迭代器工作方法在下一篇文章中详细介绍。

```javascript
// 在桶中查找键值对为key的value
func (b *Bucket) Get(key []byte) []byte {
 k, v, flags := b.Cursor().seek(key)

 // key是一个桶的名字
 if (flags & bucketLeafFlag) != 0 {
  return nil
 }

 // key不存在
 if !bytes.Equal(key, k) {
  return nil
 }
 return v
}
```

复制

#### **向桶中放入数据**

向桶中放入数据都是添加在B+Tree的叶子节点上，所以先要定位到待插入的叶子节点，这个过程是通过迭代器Cursor实现的。如果key已存在，相当于更新value，否则插入key-value， 注意一点，对key是深拷贝，value是浅拷贝。

```javascript
func (b *Bucket) Put(key []byte, value []byte) error {
 // 异常情况的校验：1 bucket关联的是读写事务，2 key不能为空 不能过长  3 value不能过长
 if b.tx.db == nil {
  return ErrTxClosed
 } else if !b.Writable() {
  return ErrTxNotWritable
 } else if len(key) == 0 {
  return ErrKeyRequired
 } else if len(key) > MaxKeySize {
  return ErrKeyTooLarge
 } else if int64(len(value)) > MaxValueSize {
  return ErrValueTooLarge
 }

 // 创建一个游标对象，对Bucket进行遍历，查找键值对为key的数据
 c := b.Cursor()
 k, _, flags := c.seek(key)

 // key是一个桶的名字，返回不兼容
 if bytes.Equal(key, k) && (flags&bucketLeafFlag) != 0 {
  return ErrIncompatibleValue
 }

 // 运行到这里，key有可能存在，也有可能不存在，不存在会创建一个key-value键值对，
 // 存在会更新原来key的value，这些处理都是在put函数中实现的
 // NOTE 对key是深拷贝，对value是浅拷贝
 key = cloneBytes(key)
 c.node().put(key, key, value, 0, 0)

 return nil
}
```

复制

#### **删除桶中数据**

删除数据，也需要定位到key所在的叶子节点，然后将其删除。

```javascript
func (b *Bucket) Delete(key []byte) error {
 if b.tx.db == nil {
  return ErrTxClosed
 } else if !b.Writable() {
  return ErrTxNotWritable
 }

 // 创建一个游标对象进行查询
 c := b.Cursor()
 _, _, flags := c.seek(key)

 // 如果存在同名的bucket
 if (flags & bucketLeafFlag) != 0 {
  return ErrIncompatibleValue
 }

 // 进行真正删除
 c.node().del(key)

 return nil
}
```


# boltdb源码分析系列-迭代器

发布于2022-08-15 15:03:48阅读 1820

Cursor是boltdb中的迭代器，它能够按顺序访问Bucket中的数据。在前面的文章中说过，一个Bucket是一颗B+Tree. Cursor任务是对B+Tree中节点中数据的访问，因为B+Tree树中的数据是保存在叶子节点，所以严格来说Cursor遍历访问的是叶子节点中的存储的数据。

## **Cursor是什么**

前文对Cursor进行了概括说明，本小节将结合Cursor的数据结构定义，对其进行更详细的说明。

Cursor是boltdb中的一个结构体，定义在cursor.go文件中。它只有2个字段，如下所示。

-   bucket:Cursor关联的桶，Cursor的核心工作是对桶中的数据进行顺序访问，bucket记录的就是要访问的桶
-   stack:Bucket是一颗B+Tree，也就是一颗多叉树，Cursor在访问的时候是按照二叉树那种先序遍历的方式进行的，访问到的每个节点都会加入到stack中。所以它是切片结构，存储的元素是elemRef复合类型。elemRef中两个指针和一个整数字段。page和node是一一对应的，它们都描述的是B+Tree节点中的node和page信息，index用于表示当前遍历的是节点中的第几个元素。

一颗B+Tree包含多个节点，而每个节点中又装有很多数据。所以这是一个二维结构，定位一个数据，需要先定位到某个节点，在定位到节点中的第几个元素。通过下标索引访问stack锁定到某个节点（elemRef），在通过elemRef中的index便可锁定到具体的元素。

```javascript
type Cursor struct {
 bucket *Bucket
 stack  []elemRef
}

type elemRef struct {
 page  *page
 node  *node
 index int
}
```

复制

![](https://ask.qcloudimg.com/http-save/yehe-9955853/cf0712f2f1245c4e668de2bdbd6e9c9f.jpeg?imageView2/2/w/2560/h/7000)

## **为什么需要Cursor**

在说需要Cursor原因之前，我们先来看boltdb中B+Tree的结构。标准的B+Tree结构如下图所示，像[MySQL](https://cloud.tencent.com/product/cdb?from=20065&from_column=20065)中索引组织也是B+Tree，它的结构就是下面这样的。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/3ddcf4aada5b8f6db1ec0767aeb0353e.png?imageView2/2/w/2560/h/7000)

所有的数据都存储在叶子节点上，对应到图中的绿色内容。图中一共有9个叶子节点，它们之间通过指针连接了起来，构成了一个链表结构。

而boltdb中B+Tree的叶子节点并没有通过链表连接起来，这样做的原因小编觉得是为了减少了维持B+Tree平衡性质的难度,但在遍历的时候增加了一定的困难。所以这里抽象了Cursor结构来进行遍历，在遍历的时候记下了遍历的路径，方便进行快速回溯。

遍历路径上的每步是elemRef结构，在前面Cursor结构体定义介绍中有说明。这里在强调下处理时node和page的差别，处理时优先使用node，否则使用page。因为node中的数据可能是更新过的，page中的数据来自mmap，是没有被更新的。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/c1edcabf07dcc86700e0c413657b2bfd.jpeg?imageView2/2/w/2560/h/7000)

总结起来，因为boltdb中的B+Tree没有将叶子节点通过链表串联起来，为了能够方便对其访问，抽象处理了Cursor迭代器，来对其进行遍历操作。

## **Cursor工作原理**

Cursor的功能是对B+Tree的遍历访问，提供了First、Last、Next、Prev和Seek等方法，分别是查找第一个数据，最后一个数据，下一个数据，前一个数据以及定位到某个位置。

First、Last、Seek方法查询的是一个绝对位置，也就是说，调用它们不会受到当前Cursor游标位置的影响，都是从根节点开始查询。体现在代码中是它们都有重置`c.stack = c.stack[:0]`操作，

Next、Prev查询的是相对于当前游标位置来说的，所以调用这两个方法，依赖于调用前游标所在的位置。它们实现中不会有重置c.stack操作。

为了代码复用，抽取了first、last、next、seek等内部方法，供对外提供方法的接口First、Last、Next、Prev和Seek调用。

#### **获取第一个数据**

First方法返回B+Tree中第一个数据，就是最左侧（如果里面元素非空）的叶子节点的第一个元素。处理逻辑概括起来分为3步：

1.  加载根节点到迭代器stack
2.  调用c.first方法，一路向下向左，走到叶子节点, 并将路径上的节点加入stack
3.  取stack最后一个元素（目标叶子节点）它里面的第一个数据

![](https://ask.qcloudimg.com/http-save/yehe-9955853/187d806a59ce3b60299f28233b93d5c7.jpeg?imageView2/2/w/2560/h/7000)

```javascript
func (c *Cursor) First() (key []byte, value []byte) {
 _assert(c.bucket.tx.db != nil, "tx closed")
 // 重置stack
 c.stack = c.stack[:0]
 // 加载根节点
 p, n := c.bucket.pageNode(c.bucket.root)
 // 根节点信息加入stack
 c.stack = append(c.stack, elemRef{page: p, node: n, index: 0})
 // 一路向下走到第一个叶子节点，内部通过for循环走到最左侧的叶子节点
 c.first()

 // https://github.com/boltdb/bolt/issues/450
 // 叶子节点中没有元素，移动到下一个叶子节点
 if c.stack[len(c.stack)-1].count() == 0 {
  c.next()
 }
    // 获取最左侧叶子节点中的第一个key-value
 k, v, flags := c.keyValue()
 // 如果是一个桶节点，value为空
 if (flags & uint32(bucketLeafFlag)) != 0 {
  return k, nil
 }
 // 存储的是数据，返回key-value
 return k, v

}
```

复制

#### **获取最后一个数据**

Last方法获取B+Tree中最后一个数据，处理过程与First类型，只不过在查找的时候一路向下向右走，定位到最右侧的叶子节点。具体处理过程也分为3步，与First类似，这里不在展开说明了。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/85f7bb1e9b697a0ec409ca05f6177ef9.jpeg?imageView2/2/w/2560/h/7000)

```javascript
func (c *Cursor) Last() (key []byte, value []byte) {
 _assert(c.bucket.tx.db != nil, "tx closed")
 c.stack = c.stack[:0]
 // 加载根节点
 p, n := c.bucket.pageNode(c.bucket.root)
 ref := elemRef{page: p, node: n}
 // 设置index为最后一个元素位置
 ref.index = ref.count() - 1
 // 加入路径缓存stack
 c.stack = append(c.stack, ref)
 // 一路向下走到最右侧的节点
 c.last()
 // 获取游标位置（最右边节点的最后一个）元素
 k, v, flags := c.keyValue()
 // 如果是Bucket，返回value为空
 if (flags & uint32(bucketLeafFlag)) != 0 {
  return k, nil
 }
 // 返回key-value
 return k, v
}
```

复制

#### **获取下一个数据**

Next方法获取下一个数据，是基于当前游标位置的下一个位置数据，实际调用的是内部的next方法，next方法分析见下文。

```javascript
func (c *Cursor) Next() (key []byte, value []byte) {
 _assert(c.bucket.tx.db != nil, "tx closed")
 k, v, flags := c.next()
 if (flags & uint32(bucketLeafFlag)) != 0 {
  return k, nil
 }
 return k, v
}
```

复制

next方法真正实现将游标移动到下一个位置，并返回它的数据。有两种情况需要考虑：

-   情况1：当前游标index在节点（叶子节点）元素的内部，即index还没有走到节点中的最后一个元素位置，这种很好处理，就是index++即可
-   情况2：当前游标index已经走到节点中的最后一个元素位置，需要将游标移动到当前节点的下一个邻近节点的第一个元素位置。因为boltdb中B+Tree叶子节点没有构成链表结构，所以需要回溯到当前节点的父节点，然后走到它的邻近右节点。

情况1的处理在下面for循环中的`elem.index < elem.count()-1`,直接elem.index++即游走到了下一个元素位置。此种情况，`c.stack = c.stack[:i+1]`会保持不变，调用`c.first()`会立即返回，因为当前已在叶子节点。

情况2的处理也在for循环中，for循环会执行两次，情况1中for循环只会执行一次就break掉了。循环完之后，执行`c.stack = c.stack[:i+1]`会切掉右边数据，相当于剪掉了B+Tree已处理过的节点，这个过程就是回溯。然后调用c.first()定位到它的邻近叶子节点。

```javascript
func (c *Cursor) next() (key []byte, value []byte, flags uint32) {
 for {
  var i int
  // 切片c.stack中保存的是访问的前缀信息，i从大到小循环，先处理游标最后位置
  // 先在节点内部找到下一个元素，如果节点内部元素已全部访问完，i--走到下一个节点
  for i = len(c.stack) - 1; i >= 0; i-- {
   elem := &c.stack[i]
   // 移动到节点中的下一个元素
   if elem.index < elem.count()-1 {
    elem.index++
    break
   }
  }

  // i为-1说明整颗树已经处理完了
  if i == -1 {
   return nil, nil, 0
  }

  // 清理掉已经处理过的节点，保留剩下未访问处理的
  c.stack = c.stack[:i+1]
  // 定位到剩下未处理完成的节点，c.stack[i]即为待处理的节点
  // c.stack[i].index已经定位到节点中位置
  c.first()

  // https://github.com/boltdb/bolt/issues/450
  // 节点中没有数据，跳出
  if c.stack[len(c.stack)-1].count() == 0 {
   continue
  }
  // 返回key-value
  return c.keyValue()
 }
}
```

复制

#### **获取前一个数据**

Prev方法是Next方法的反过程，与Next方法类似，也有两种情况需要考虑，分别是当前游标位置处于节点中的内部，不是节点的第一个元素位置，和已处于第一个运算位置。

-   情况1：当前游标位置不在节点的中第一个元素位置，处理很简单，将index减一即可
-   情况2：这个时候要找到当前节点的左邻近叶子节点，将index移动左邻近叶子节点的最后一个元素位置。

代码中的for循环，融合处理了上面两种情况，当循环结束的时候，调用c.last（）方法定位到游标节点的最后一个元素位置。

```javascript
func (c *Cursor) Prev() (key []byte, value []byte) {
 _assert(c.bucket.tx.db != nil, "tx closed")

 // 现在当前节点内部移动到前一个元素，特殊情况，当前已经位于节点的第一个元素位置
 // 这时需要找到当前节点的前驱叶子节点
 for i := len(c.stack) - 1; i >= 0; i-- {
  elem := &c.stack[i]
  if elem.index > 0 {
   elem.index--
   break
  }
  // 调整stack，c.stack[i]为当前活动的节点
  c.stack = c.stack[:i]
 }

 // 已经遍历完整颗树
 if len(c.stack) == 0 {
  return nil, nil
 }

 // 调用c.last定位到c.stack游标位置节点的最后一个节点的最后一个元素
 c.last()
 k, v, flags := c.keyValue()
 // 是桶，value为nil
 if (flags & uint32(bucketLeafFlag)) != 0 {
  return k, nil
 }
 return k, v
}
```

复制

#### **迭代器定位到给定key的位置**

Seek方法将迭代器定位到给定key的位置，并返回key-value值。如果key不存在，迭代器会移动到下一个数据位置。处理的核心调用了内部的seek方法，下面分析这个处理流程。

```javascript
func (c *Cursor) Seek(seek []byte) (key []byte, value []byte) {
 k, v, flags := c.seek(seek)

 if ref := &c.stack[len(c.stack)-1]; ref.index >= ref.count() {
  k, v, flags = c.next()
 }

 if k == nil {
  return nil, nil
 } else if (flags & uint32(bucketLeafFlag)) != 0 {
  return k, nil
 }
 return k, v
}
```

复制

seek方法调用search方法完成key的定位，即查找key所在的位置，因为B+Tree中数据有序，search的时候采用二分查找`sort.Search`实现快速定位。

查询流程是一个递归的过程，首先将根节点加载到stack中。递归的出口是走到了叶子节点，并查询给定的key在叶子节点中是否存在。如果当前是非叶子节点，优先从node中查找，如果node为空，则从page中查找。它们的内部也都会又调用search方法，所以形成了两条隐含的递归过程。

```javascript
search-->searchNode-->search

serach-->searchPage-->search
```

复制

searchNode和searchPage处理过程非常类似，我觉得源码实现有点重复，可以提取复用😁，处理过程非常简单，这里不再分析。

```javascript
func (c *Cursor) seek(seek []byte) (key []byte, value []byte, flags uint32) {
 _assert(c.bucket.tx.db != nil, "tx closed")

 // 清理stack
 c.stack = c.stack[:0]
 // 从根节点开始查找
 c.search(seek, c.bucket.root)
 ref := &c.stack[len(c.stack)-1]

 // key不存在
 if ref.index >= ref.count() {
  return nil, nil, 0
 }

 return c.keyValue()
}

func (c *Cursor) search(key []byte, pgid pgid) {
 p, n := c.bucket.pageNode(pgid)
 if p != nil && (p.flags&(branchPageFlag|leafPageFlag)) == 0 {
  panic(fmt.Sprintf("invalid page type: %d: %x", p.id, p.flags))
 }
 e := elemRef{page: p, node: n}
 c.stack = append(c.stack, e)

 // e是叶子节点
 if e.isLeaf() {
  // 在叶子节点中查找key
  c.nsearch(key)
  return
 }

 if n != nil {
  // 先从node中查找
  c.searchNode(key, n)
  return
 }
 // 从page中查找
 c.searchPage(key, p)
}

```


# Reference
https://www.qtmuniao.com/2020/12/14/bolt-index-design/
https://cloud.tencent.com/developer/article/2072931?areaSource=&traceId=
https://cloud.tencent.com/developer/article/2072932?areaSource=&traceId=