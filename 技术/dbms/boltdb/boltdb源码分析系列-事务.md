
#### **事务的四大特性**

事务有ACID四个属性，它们的含义如下，虽然通常都说事务有这四大特性，但是这些特性之间并不是完全正交的。小编认为A、I、D是因，C是果，因为有AID的保障，所有才有一致性C.

-   原子性（Atomic):在同一项业务处理过程中，事务保证了对多个数据的修改，要么同时成功，要么同时撤销。
-   隔离性（Isolation):在不同的业务处理过程中，事务保证了各业务正在读、写的数据相互独立，不会彼此影响。
-   持久性（Durability):事务应当保证所有成功被提交的数据修改都能够正确地被持久化，不丢失数据。
-   一致性（Consistency):指事务在开始之前和事务结束以后，系统中所有的数据是符合预期的，相关联的数据之间不会产生矛盾。

注意，这里所说的事务的ACID是单一数据源的情况，在分布式环境中，一致性是指多个副本间数据的一致性。我们知道磁盘并不是长久不坏的，所以仅凭单个磁盘不能保证数据的可靠，在分布式环境下衍生出多个副本等冗余策略，来提升数据可靠性。

对于隔离性，在[数据库](https://cloud.tencent.com/solution/database?from=20065&from_column=20065)系统中，一般有下面四种隔离级别，它们的隔离强度依次递减。

1.  可串行性化（Serializable):串性化提供了最高强度的隔离性，它保证了读写操是串行执行的，没有并发冲突，在实现上是通过加读锁、写锁和间隙锁技术手段，但是它存在吞吐量低缺点，所以在实际中一般不采用。
2.  可重复读（Repeatable Read):可重复读确保在同一个事务中读取同一行多次，得到的数据是稳定的，不会变化，即使这行数据被并发执行的事务修改了。但是存在幻读的问题，例如执行select查询某条数据是否存在，不存在，准备插入此数据，但在执行insert时发现此记录已存在，无法插入，这时产生了幻读，关键的原因在写并发，虽然读不到数据，但不代表其他事务的影响不存在。
3.  读已提交（Read Committed): 读已提交指一个事务提交之后便会对其他事务可见，例如有两个事务A和B,事务A先开启一个查询，然后事务B对数据进行更新后并提交，这时事务A再进行查询，查询的结果与前一次不同。读已提交对事务涉及的数据加的写锁会一直持续到事务结束，但加的读锁在查询操作完成之后立即释放。
4.  读未提交（Read Uncommitted):读未提交指事务执行过程中，一个事务读取到了另一个事务未提交的数据，存在脏读的问题。是因为它只会对事务涉及的数据加写锁，且一直持续到事务结束，但完全不加读锁。

#### **事务有哪些实现方法**

根据场景的不同，事务可以划分为多种，例如在分布下有[分布式事务](https://cloud.tencent.com/product/dtf?from=20065&from_column=20065)，本文这里介绍单数据源下两种典型实现方法。一种是通过写日志，另一种是通过shadow paging方式。

-   写日志：写日志即提交日志的方法，对数据的操作以日志的形式记录下来，日志中内容包含修改了什么数据、数据物理上位于哪个内存页和磁盘中等信息，日志写入通过追加形式写文件。在日志记录全部安全写入磁盘，数据库在日志中看到代表事务成功提交的提交记录（Commit Record）后，才会根据日志上的信息对真正的数据进行修改，修改完成之后，再在日志中加入一条结束记录（End Record)表示事务已完成持久化。
-   shadow paging: shadow pagin有翻译为影子分页，它的实现是对数据的变动会写入磁盘的数据中，但并不是直接原地修改原先的数据，而是先复制一份副本，保留原数据，修改副本数据。在事务处理过程中，被修改的数据会同时存在两份，一份是修改前的数据，一份是修改后的数据。当事务成功提交，所有数据的修改都成功持久化之后，最后一步是修改数据的引用指针，将引用从原数据改为新复制并修改后的副本。

通过日志实现事务的原子性和持久性是现在的主流方案，像MySQL采用的就是写日志的方法。shadow paging方法在轻量级数据库SQLite version3中被采用，本文的boltdb数据库也采用的是这种方法。两种方法对比起来，shadow paging要比写日志更简单，但涉及隔离性与并发锁时，通过shadow paging实现事务的能力要弱些。

#### **boltdb是如何实现事务原子性的？**

事务的原子性即一组数据库操作，要么全部修改成功，要么全部撤销，不存在部分操作成功部分失败的情况。boltdb是如何实现事务原子性的，可以从两个方面来分析。一方面在我们遇到问题的时候，可以主动调用tx.Rollback进行回滚，例如在执行tx.Commit过程中出现错误，调用回滚操作，undo该事务到目前为止的所有操作。另一方面是在执行事务的过程中，例如在向磁盘写数据的过程中，出现设备掉电等导致boltdb实例挂掉，这是通过boltdb采用的shadow paging方法实现的，在tx.Commit的时候，元信息page最后写磁盘，只有元信息写入成功，所有的改动才对用户可见。如果在写数据的时候宕机，这时元信息还没有写入，改动不会生效。

在没有执行tx.Commit操作之前，读写事务的写入操作都是修改内存中的node节点，内存中的node是原数据的副本，修改node不影响原始数据。如下图所示，Put操作是在叶子节点node d上，这时候修改的数据都在内存，如果此时执行tx.Rollback操作，只要丢弃这些内存中的node，不写入磁盘即可。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/f211aaa4a152e544730d8efff78855c5.jpeg?imageView2/2/w/2560/h/7000)

对于只读事务来说，不能执行tx.Commit操作，否则会panic, 但必须执行tx.Rollback操作，用于释放一些资源，清理一些引用，GC尽快进行垃圾回收。对于读写事务，执行tx.Rollback操作，会恢复空闲数据页freelist，恢复元数据页的txid,还有释放读写锁这步关键操作，使得其他读写事务可以获取到锁可以运行。

```javascript
func (tx *Tx) Rollback() error {
 _assert(!tx.managed, "managed tx rollback not allowed")
 if tx.db == nil {
  return ErrTxClosed
 }
 tx.rollback()
 return nil
}

func (tx *Tx) rollback() {
 if tx.db == nil {
  return
 }
 if tx.writable {
  // 恢复暂存的缓存
  tx.db.freelist.rollback(tx.meta.txid)
  // 从meta数据中恢复freelist
  tx.db.freelist.reload(tx.db.page(tx.db.meta().freelist))
 }
 tx.close()
}

```

复制

tx.close操作释放一些资源，清理所有的引用。如果是只读事务，会将事务句柄tx从db中移除，如果是读写事务，会释放读写锁。

```javascript
func (tx *Tx) close() {
 if tx.db == nil {
  return
 }
 if tx.writable {
  ...
  tx.db.rwtx = nil
  // 释放进行读写操作的读写锁
  tx.db.rwlock.Unlock()
        ...
 } else {
  // 溢出当前的只读事务tx
  tx.db.removeTx(tx)
 }

 // 清理所有引用
 tx.db = nil
 tx.meta = nil
 tx.root = Bucket{tx: tx}
 tx.pages = nil
}
```

复制

如果在执行tx.Commit过程中出现异常情况，boltdb是如何保证操作原子性的呢？下面源码抽取关键的4步操作：

1.  释放旧的freelist占有的页面，并分配新的freelist页面，并将新分配的页面加入缓存，对应到下面的`tx.db.freelist.free`语句。
2.  刷新脏页（包含为新分配的freelist页）到磁盘，对应到下面的`tx.allocate`和`tx.db.freelist.write`操作，前者新分配合适大小的freelist页，并将当前哪些页是空闲的填入到新的freelist中。
3.  刷新元数据页到磁盘，这步操作是最为关键的一步，在后面单独分析。
4.  关闭事务，执行`tx.close`操作

![](https://ask.qcloudimg.com/http-save/yehe-9955853/671a7af67659b9edc1e239bc6fd26dec.jpeg?imageView2/2/w/2560/h/7000)

```javascript
func (tx *Tx) Commit() error {
 ...
 tx.db.freelist.free(tx.meta.txid, tx.db.page(tx.meta.freelist))
 // 申请一片空间用于存储新的freelist信息
 p, err := tx.allocate((tx.db.freelist.size() / tx.db.pageSize) + 1)
 if err != nil {
  tx.rollback()
  return err
 }
 // 将freelist页中的信息序列化到内存p中，并将脏页p加入到缓存中
 if err := tx.db.freelist.write(p); err != nil {
  tx.rollback()
  return err
 }
 tx.meta.freelist = p.id
    ...
 // 刷新脏页（包含新的freelist）到磁盘
 if err := tx.write(); err != nil {
  tx.rollback()
  return err
 }
    ...
 // 刷新元数据到磁盘
 if err := tx.writeMeta(); err != nil {
  tx.rollback()
  return err
 }
 tx.stats.WriteTime += time.Since(startTime)

 // 关闭事务
 tx.close()

 ...
}
```

复制

**「注意，上面关键的4步操作的顺序是不能调整的,刷新元数据落盘的操作一定要放到后面，元信息页作为 “全局指针”，以该指针的写入原子性来保证事务的原子性」**。

第3步刷新元数据页到磁盘是最为关键的一步，writeMeta只有一处有返回错误，就是在执行tx.db.ops.writeAt的时候，其他操作都是内存中操作。writeMeta处理逻辑是先在内存中开辟一个4KB的空间，然后向里面填充page字段，在然后将tx.meta中的信息写入刚申请的空间中。注意这里pgid的确定，通过`p.id = pgid(m.txid % 2)`操作，轮流写入，一次写入meat page0，另一次写入meta page1，相当于灾备写入，如果出现异常，可以通过另一个正常的meta恢复。

```javascript
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
 if !tx.db.NoSync || IgnoreNoSync {
  if err := fdatasync(tx.db); err != nil {
   return err
  }
 }

 // Update statistics.
 tx.stats.Write++

 return nil
}

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

总结起来，对于tx.Commit执行过程中出现的异常，是通过tx.Commit特定的处理逻辑顺序巧妙处理，降低了出现不原子性结果的概率。

#### **boltdb是如何实现事务隔离性的？**

对于boltdb来说，不存在并发的写操作，所以事务的隔离被限定在了多个只读事务的隔离性和1个写事务操作和读事务操作的隔离性上。

本小节，主要来学习两面内容，一是怎么实现写事务操作只有一个，读事务有多个限制的；二是事务隔离，通过什么隔离，是怎么隔离的。

写事务只有一个，读事务操作有多个是通过锁来实现的。DB结构体中有5把锁，batchMu是执行批量事务操作用的，保护的对象是batch. rwlock是一个互斥锁，它是用来确保只有一个写事务的，开启一个写事务，需要获取rwlock的写锁，如果写锁已被别人获取，会被阻塞。metalock用于保护元素数据信息meta对象。mmaplock是一个读写锁，开启只读事务的时候，需要获取它的读锁，可以同时获取多个读锁，所以读事务操作是可以并发的。statlock是保护boltdb统计分析对象用的，这里不用过多关心。

```javascript
type DB struct {
 ...
 batchMu sync.Mutex
 batch   *batch
    
 rwlock   sync.Mutex   
 metalock sync.Mutex   
 mmaplock sync.RWMutex 
 statlock sync.RWMutex 
    ...
}

```

复制

开启只读事务，需要获取mmap读锁

```javascript
func (db *DB) beginTx() (*Tx, error) {
 ...
 // 获取mmap的读锁
 db.mmaplock.RLock()

 ...
}
```

复制

关闭只读事务，即执行tx.Rollback操作，释放mmaplock读锁. tx.Rollback-->tx.rollback-->tx.close-->tx.db.removeTx.

```javascript
func (db *DB) removeTx(tx *Tx) {
 db.mmaplock.RUnlock()
    ...
}
```

复制

开启写事务，需要获取rwlock的写锁

```javascript
func (db *DB) beginRWTx() (*Tx, error) {
 ...
 db.rwlock.Lock()

 ...
}
```

复制

执行回滚操作tx.Rollback，或者提交事务操作tx.Commit，释放rwlock的写锁

```javascript
func (tx *Tx) close() {
 ...
 if tx.writable {
     ...
  // 释放进行读写操作的读写锁
  tx.db.rwlock.Unlock()

  ...
 } else {
  ...
 }
    ...
}

```

复制

**「强调一点，只读事务必须执行tx.Rollback操作，否则可能阻塞写事务的tx.Commit操作」**。前面分析了，只读事务会进行db.mmaplock.RLock()操作，也就是对mmaplock获取读锁，而在tx.Commit中会为修改的数据分配新page和分配新的freelist page都是通过`tx.db.allocate(count)`来分配的,它里面会调用db.map，db.map中会获取db.mmaplock写锁。两条调用路径如下，所以如果只读事务中的读锁没有释放，下面获取写锁会阻塞。

```javascript
1.
tx.root.spill()-->b.rootNode.spill()-->tx.allocate((node.size() / tx.db.pageSize) + 1)-->tx.db.allocate(count)-->db.mmap(minsz)-->db.mmaplock.Lock()

2.
tx.allocate((tx.db.freelist.size() / tx.db.pageSize) + 1)-->tx.db.allocate(count)-->db.mmap(minsz)-->db.mmaplock.Lock()
```

复制

#### **boltdb是如何实现事务持久性的？**

事务持久性需要将[数据存储](https://cloud.tencent.com/product/cdcs?from=20065&from_column=20065)到硬盘等长久保存的设备中，boltdb采用的是写磁盘的方式，每次执行tx.Commit都将当前的修改的数据刷新到磁盘中。而写磁盘操作相比内存是很费时的，存在下面的问题：

1.  改动很少的数据也写磁盘，代价比较高
2.  如果一次改动的数据不在同一个page，这需要写多个page,多个page不一定是连续的，也就是随机写IO,代价也是比较高

所以boltdb使用于读多写少的场景，对于写操作多并且性能要求高的情况，boltdb就不太适合。boltdb采用B+Tree组织，节点的degree高达几百，所以它是个矮胖小子，加载叶子节点到内存不需要进行很多次IO操作，并且采用mmap映射，提升性能。

在事务提交时，需要将用户进行的一系列更新、插入和删除操作相关的node进行调整，按一定策略调整为B+Tree,使得它维持好的查询性质，最后将所有的node序列化为page写入磁盘，构成一颗新的平衡的B+Tree.上述核心实现在`tx.root.rebalance()`和`tx.root.spill()`,下面分析下tx.Commit中的这两个关键操作.

rebalance是再平衡操作，主要是如果node过小（key很少或者整体占用的空间不大），将其他与邻近的节点进行合并，可以提高页面的利用率。

rebalance先对当前Bucket缓存的所有node进行调整，在对它的孩子Bucket进行平衡调整。

```javascript
func (b *Bucket) rebalance() {
 // 对当前Bucket所有缓存的node进行调整
 for _, n := range b.nodes {
  n.rebalance()
 }
 // 在对孩子Bucket进行平衡操作
 for _, child := range b.buckets {
  child.rebalance()
 }
}
```

复制

核心在于对node进行合并操作，下面对源码关键做了详细注解，总结起来，有以下操作步骤：

1.  判断节点是否已经进行过平衡处理了，如果已经是平衡的，直接跳过，避免对节点重复处理。
2.  判断节点是否满足合并的条件，如果不满足，即n.size() > threshold && len(n.inodes) > n.minKeys()，则直接结束
3.  处理特殊情况1，如果当前节点是根节点，并且只有一个孩子节点，将它和它的唯一孩子合并
4.  处理特殊情况2，如果节点没有孩子节点，将当前节点删除
5.  对节点进行合并，默认是将当前节点合并到它的左兄弟上，如果它是第一个节点，没有左兄弟，将其与右兄弟进行合并
6.  上面的调整可能会导致节点的删除，因此向上递归看是否需要进一步进行平衡调整

```javascript
func (n *node) rebalance() {
 // 如果节点node n是平衡的，直接返回
 if !n.unbalanced {
  return
 }
 // 标记节点n是平衡的，经过下面的处理之后就是平衡的了
 n.unbalanced = false

 n.bucket.tx.stats.Rebalance++

 // 阈值threshold为page页大小的1/4,page大小为4KB,即threshold为1KB
 var threshold = n.bucket.tx.db.pageSize / 4
 // 当节点n所占空间的大小大于1KB 并且 节点中的元素的个数超过最小值，对于页节点是1个
 // 对于分支节点是2个，即叶子节点元素最少为2个，分支节点元素最少为3个
 // 不用进行合并平衡处理
 if n.size() > threshold && len(n.inodes) > n.minKeys() {
  return
 }

 // n是根节点
 if n.parent == nil {
  // 如果该节点是根节点，且只有一个孩子节点，则将其和其唯一的孩子合并
  if !n.isLeaf && len(n.inodes) == 1 {
   child := n.bucket.node(n.inodes[0].pgid, n)
   n.isLeaf = child.isLeaf
   n.inodes = child.inodes[:]
   n.children = child.children

   // 修改inodes的父节点，从原来的child改为现在的n
   for _, inode := range n.inodes {
    if child, ok := n.bucket.nodes[inode.pgid]; ok {
     child.parent = n
    }
   }

   // 解除child关联的父节点
   child.parent = nil
   delete(n.bucket.nodes, child.pgid)
   // 释放child节点，加入freelist
   child.free()
  }

  return
 }
    
 // 节点n中没有任何元素
 if n.numChildren() == 0 {
  n.parent.del(n.key)
  // 从节点n的父节点中将节点n移除
  n.parent.removeChild(n)
  // 删除bucket nodes中的缓存
  delete(n.bucket.nodes, n.pgid)
  // 释放节点n
  n.free()
  // 对节点n的父节点继续进行平衡操作，因为节点n被删除，可能会导致它的父节点存在不平衡
  n.parent.rebalance()
  return
 }

 _assert(n.parent.numChildren() > 1, "parent must have at least 2 children")

 // 下面将节点n和它的兄弟节点进行合并，默认是与节点n的左边的兄弟节点合并，但是如果节点n
 // 本来就是第一个节点，也就是它没有左兄弟，这种情况下，将节点n与它的右兄弟进行合并
 var target *node
 // 判断节点n是不是第一个节点，如果是则将它与它右边的兄弟节点合并
 var useNextSibling = (n.parent.childIndex(n) == 0)
 if useNextSibling {
  // 选择与右兄弟节点进行合并
  target = n.nextSibling()
 } else {
  // 选择与左兄弟节点进行合并
  target = n.prevSibling()
 }

 // 与右兄弟进行合并
 if useNextSibling {
  // 调整target中所有孩子的节点父节点，从target调整为n，因为target和n会合并
  // 合并之后的节点为n
  for _, inode := range target.inodes {
   if child, ok := n.bucket.nodes[inode.pgid]; ok {
    // 从旧父节点中移除child
    child.parent.removeChild(child)
    // 调整child的新的父节点为当前的节点n
    child.parent = n
    // 将child加入到节点n的孩子节点集合中
    child.parent.children = append(child.parent.children, child)
   }
  }

  // 将target中的数据合并到节点n中
  n.inodes = append(n.inodes, target.inodes...)
  // 从当前节点n的父节点中删除target key
  n.parent.del(target.key)
  // 将target从B+Tree中移除
  n.parent.removeChild(target)
  // 缓存的node集合中删除target
  delete(n.bucket.nodes, target.pgid)
  // 释放target占用的空间
  target.free()
 } else {
  // 操作与上述一致，只不过是将当前的节点n合并到target中
  for _, inode := range n.inodes {
   if child, ok := n.bucket.nodes[inode.pgid]; ok {
    child.parent.removeChild(child)
    child.parent = target
    child.parent.children = append(child.parent.children, child)
   }
  }

  target.inodes = append(target.inodes, n.inodes...)
  n.parent.del(n.key)
  n.parent.removeChild(n)
  delete(n.bucket.nodes, n.pgid)
  n.free()
 }

 // 上面的调整可能会导致节点的删除，因此向上递归看是否需要进一步进行平衡调整
 n.parent.rebalance()
}
```

复制

**「对于n.parent.rebalance操作，也许有同学有疑问，如果处理的时候先处理的n.parent节点，这时不是将n.parent的unbalanced已经设置为false吗，那执行n.parent.rebalance操作不是在开头判断直接退出了吗？」**，不会的，因为前面有执行`n.parent.del(target.key)`操作，它的实现中会将unbalanced设置为true.

```javascript
func (n *node) del(key []byte) {
    ...
 // 给该节点n做一个不平衡的标记，在事务进行提交的时候，决定是否要对该节点进行rebalance操作
 n.unbalanced = true
}
```

复制

只有node.del()的调用才会导致n.unbalanced被标记为true。只有两个地方会调用node.del,一个地方就上面的rebalance操作的时候，另一个地方是在用户调用bucket.Delete函数删除数据的时候，而前者又是因为后者引起的，所以说只有用户在某次写事务中删除数据时，才会引起node.rebanlance逻辑的实际执行。

spill方法核心功能有两点，一是将占用空间过大的节点进行拆分，二是将节点转成page（脏页），为稍后写入磁盘做准备。采用自底向上的处理方式，先处理当前Bucket的子Bucket，然后再处理当前Bucket.如果一个Bucket中的内容很少，将会直接内嵌在父Bucket的叶子节点中。否则，先调用子Bucket的spill函数，将子Bucket的根节点pgid放在父节Bucket的叶子节点中。对于桶内的处理调用node.spill方法。

```javascript
func (b *Bucket) spill() error {
 // 先对子Bucket进行spill处理
 for name, child := range b.buckets {
  var value []byte
  // 判断child是否可以被内联
  if child.inlineable() {
   // 可以被内联，释放child
   child.free()
   // 将child序列化内容放入到它父节点page中
   value = child.write()
  } else {
   // 递归对child的子Bucket进行spill处理
   if err := child.spill(); err != nil {
    return err
   }

   // 重新获取child.bucket信息，因为在对child进行spill处理后
   // 因为有分裂操作，可能会导致child.bucket发生变化
   value = make([]byte, unsafe.Sizeof(bucket{}))
   var bucket = (*bucket)(unsafe.Pointer(&value[0]))
   *bucket = *child.bucket
  }

  if child.rootNode == nil {
   continue
  }

  var c = b.Cursor()
  k, _, flags := c.seek([]byte(name))
  if !bytes.Equal([]byte(name), k) {
   panic(fmt.Sprintf("misplaced bucket header: %x -> %x", []byte(name), k))
  }
  if flags&bucketLeafFlag == 0 {
   panic(fmt.Sprintf("unexpected bucket header flag: %x", flags))
  }
  // 更新name的value
  c.node().put([]byte(name), []byte(name), value, 0, bucketLeafFlag)
 }

 if b.rootNode == nil {
  return nil
 }

 // 从根节点开始进行溢出处理
 if err := b.rootNode.spill(); err != nil {
  return err
 }
 b.rootNode = b.rootNode.root()

 if b.rootNode.pgid >= b.tx.meta.pgid {
  panic(fmt.Sprintf("pgid (%d) above high water mark (%d)", b.rootNode.pgid, b.tx.meta.pgid))
 }
 // 更新b的根pgid
 b.root = b.rootNode.pgid

 return nil
}
```

复制

node.spill方法是对Bucket内部节点按tx.db.pageSize进行切分，转成对应的page. 主要处理逻辑如下：

1.  判断n.spilled标记，默认为false,表示需要对该节点进行调整，如果调整过，它的spilled为true,会跳过处理。
2.  递归调整处理孩子节点，然后再处理当前节点，是一个自底向上的递归处理过程。
3.  对节点按照tx.db.pagesize进行拆分
4.  为所有新节点分配合适的大小的page,将node内容写入page中
5.  如果spilt产生了新的根节点，需要向上递归处理，调整其他分支。

```javascript
func (n *node) spill() error {
 var tx = n.bucket.tx
 // 如果节点已经处理过了，直接返回
 if n.spilled {
  return nil
 }

 // 先对孩子节点进行排序
 sort.Sort(n.children)
 // 先处理当前节点n的孩子节点，处理过程是一个自底向上的递归
 for i := 0; i < len(n.children); i++ {
  if err := n.children[i].spill(); err != nil {
   return err
  }
 }

 // 处理到这里的时候，节点n的孩子节点已经全部处理完并保存到分配的脏页page中，脏页page已经被保存到
 // tx.pages中了，所以可以将n.children设置为nil,GC可以回收孩子节点node了
 n.children = nil
    
 // 按给定的pageSize对节点n进行切分
 var nodes = n.split(tx.db.pageSize)
 for _, node := range nodes {
  // 不是新分配的node,也就是该node在db中有关联的page,需要将node对应的page加入到freelist中
  // 在下一次开启读写事务时，此page可以被重新复用
  if node.pgid > 0 {
   tx.db.freelist.free(tx.meta.txid, tx.page(node.pgid))
   node.pgid = 0
  }

  // 为节点n分配page,注意分配的page页此时还是位于内存中，尚未刷新到磁盘，
  // 这里的count为啥要+1？ 因为前面对节点n是按pageSize切分的，切分后node节点的大小有可能
  // 小于pageSize，也有可能比pageSize大一点点，split逻辑里面是 size>threshold 才break的
  // 所以需要+1才能满足。count的值最小为1，也有可能比较大，例如节点n中存在一个超大的key-value，
  // 直接导致node.size()很大
  p, err := tx.allocate((node.size() / tx.db.pageSize) + 1)
  if err != nil {
   return err
  }

  if p.id >= tx.meta.pgid {
   panic(fmt.Sprintf("pgid (%d) above high water mark (%d)", p.id, tx.meta.pgid))
  }
  // 分配的page id值赋值给node
  node.pgid = p.id
  // 将node中的元素内容序列化化到page p中
  node.write(p)
  // 标记节点node已经spill处理过了
  node.spilled = true

  // 将节点node添加到它的父节点中
  if node.parent != nil {
   var key = node.key
   if key == nil {
    key = node.inodes[0].key
   }

   node.parent.put(key, node.inodes[0].key, nil, node.pgid, 0)
   node.key = node.inodes[0].key
   _assert(len(node.key) > 0, "spill: zero-length node key")
  }

  tx.stats.Spill++
 }

 // 对根节点进行spill
 if n.parent != nil && n.parent.pgid == 0 {
  // 先将children设置nil,防止又递归spill
  n.children = nil
  // 对根节点进行spill
  return n.parent.spill()
 }

 return nil
}
```

复制

通过上面的spill实现可以看到，boltdb在维护B+Tree查找性质的时候，并不是像数据库上介绍的那样将分支节点的数量保持在一个固定的范围，而是直接按节点数据是否能够保存到一个page中来实现的，这样做的好处是可以减少page内部碎片，提升空间利用率。


# Boltdb 源码导读（三）：Boltdb 事务实现

 发表于 2021-04-02   更新于 2022-04-30   分类于 [源码阅读](https://www.qtmuniao.com/categories/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/)  阅读次数： 692

> boltdb 是市面上为数不多的纯 go 语言开发的、单机 KV 库。boltdb 基于 Howard Chu’s LMDB 项目 ，实现的比较清爽，去掉单元测试和适配代码，核心代码大概四千多行。简单的 API、简约的实现，也是作者的意图所在。由于作者精力所限，原 boltdb 已经封版，不再更新。若想改进，提交新的 pr，建议去 etcd 维护的 fork 版本 bbolt。
> 
> 为了方便，本系列导读文章仍以不再变动的原 repo 为基础。该项目麻雀虽小，五脏俱全，仅仅四千多行代码，就实现了一个基于 B+ 树索引、支持一写多读事务的单机 KV 引擎。代码本身简约朴实、注释得当，如果你是 go 语言爱好者、如果对 KV 库感兴趣，那 boltdb 绝对是不可错过的一个 repo。
> 
> 本系列计划分成三篇文章，依次围绕[**数据组织**](https://www.qtmuniao.com/2020/11/29/bolt-data-organised/)、[**索引设计**](https://www.qtmuniao.com/2020/12/14/bolt-index-design/)、**事务实现**等三个主要方面对 boltdb 源码进行剖析。由于三个方面不是完全正交解耦的，因此叙述时会不可避免的产生交织，读不懂时，暂时略过即可，待有全貌，再回来梳理。本文是第三篇， boltdb 事务实现。

## [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E5%BC%95%E5%AD%90 "引子")引子

在分析 boltd 的事务之前，我们有必要对事务概念做一个界定，以此来明确我们的讨论范围。**数据库事务**（简称：**事务**）是数据库管理系统执行过程中的一个逻辑单位，由一个有限的[数据库](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93)操作序列构成 [^1](https://www.qtmuniao.com/2021/04/02/bolt-transaction/%E7%BB%B4%E5%9F%BA%E7%99%BE%E7%A7%91%E4%BA%8B%E5%8A%A1%EF%BC%9Ahttps://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1)。wiki 上的定义有点拗口，理解时只需抓住几个关键点即可：

1.  执行：计算层面
2.  逻辑单位：意味着不可分割
3.  操作序列有限：一般粒度不会太大

那为什么要有事务呢？事务出现于上世纪七十年代，是为了解放数据库用户的心智而出现的：事务帮助用户组织一组操作、并在出错时自动进行扫尾。后来，NoSQL 和一些分布式存储为了高性能而舍弃了完整事务的支持。然而，历史是螺旋上升的，事务的便利性让 NewSQL 等新一代分布式数据库又将其重新请回。

提起事务，最脍炙人口的便是 ACID 四大特性。 其实 ACID 更像一种易于记忆的口号而非严格的描述，因为他们在概念上并不怎么对称，而且依赖于一些上下文阐释。本文仍然会按照这四个方面对 boltdb 对事务的支持进行剖析，但在每个小结开始，会先参考 Martin Kleppmann 的演讲 [^2]，试着从不同角度先阐释其内涵；然后在再分析 boltdb 对其实现。

_作者：木鸟杂记 [https://www.qtmuniao.com/2021/04/02/bolt-transaction/](https://www.qtmuniao.com/2021/04/02/bolt-transaction/) , 转载请注明出处_

![transaction.png](https://i.loli.net/2021/04/10/vpQ6KmdieFGyAjV.png)

boltdb 只支持一写多读的事务，即同时至多有一个**读写**事务，而可以有多个**只读**事务，算是一种弱化的事务模型，好处在于容易实现，坏处在于牺牲了写并发的性能。也因此，boltdb 适合读多写少的应用。

boltdb 事务实现的主要代码在 `tx.go` 中，但这个源文件大抵算一个事务实现入口，事务提交时的一些行为，主要在数据库索引逻辑中实现，可以参考[之前文章](https://www.qtmuniao.com/2020/12/14/bolt-index-design/)。

## [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E6%8C%81%E4%B9%85%E6%80%A7%EF%BC%88Durability%EF%BC%89 "持久性（Durability）")持久性（Durability）

在早期，数据库将数据刷到**磁带**上，即获得持久性，断电重启后数据不会丢失。后来磁带变成**磁盘**，再后来到分布式系统时代，海量磁盘场景下单盘不可靠，便又衍生出**多副本**等冗余策略。虽然策略一直在演变，但其目大抵相同：事务一旦提交，对应更改便会无视各种**常见**故障，而进行长久保持。

boltdb 是一个单机数据库引擎，因此暂不必考虑磁盘故障，数据刷到磁盘上即可认为完成了持久化。其实现代码在函数 `func (tx *Tx) Commit() error` 中。需要说明的是，boltdb 中只有读写事务才须提交，只读事务提交会报错，但只读事务需要在结束时调用 `tx.Rollback` 以释放资源（比如锁）。这个设定有点反直觉，毕竟对于只读事务来说，明明是关闭，却叫 `Rollback`。

读写事务提交时，为了保证事务的持久性，boltdb 主要做了两方面的工作：

1.  改动数据刷盘
2.  元信息刷盘

### [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E6%94%B9%E5%8A%A8%E6%95%B0%E6%8D%AE%E5%88%B7%E7%9B%98 "改动数据刷盘")改动数据刷盘

在一个读写事务中，所有用户的直接改动（增加、删除、改动）都发生在**叶子节点**，但为了维持 B+ 树的性质，会在 `Commit` 前进行调整，会引起**中间节点**的级联变动。所有这些节点（Node）在 `spill` 阶段通过 `node.write(p)` 转化为页（Page），所有变动的页（包括复用 freelist 中的和新申请的）称为**脏页**（dirty pages）。在 `spill` 为 page 后，boltdb 会通过 `func (tx *Tx) write() error` 将这些脏页进行刷盘，大体逻辑为：

1.  将脏页按 page id 排序后逐个遍历
2.  将 page id 转化为 offset
3.  通过 `db.ops.writeAt` 将脏页在 offset 处刷盘
4.  通过 page pool 复用 page size = 1 的脏页，以备 allocate 时复用

需要注意的是，这个过程中有个可配置项 `db.NoSync`。如果 `db.Nosync = true` ，每次 Commit 时不会立即刷盘，只是写到操作系统的缓冲区，由操作系统决定真正落盘时机，性能较好。但是意外宕机会导致缓冲区数据丢失，从而不能保证严格持久性。

### [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E5%85%83%E4%BF%A1%E6%81%AF%E5%88%B7%E7%9B%98 "元信息刷盘")元信息刷盘

元信息包括 freelist 表和整个 db 的元信息页刷盘，刷盘过程不再赘述。需要注意的是元信息页刷盘一定在最后，以保证事务所有改动生效的原子性，这个点在后面也会强调。

## [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E4%B8%80%E8%87%B4%E6%80%A7%EF%BC%88Consistency%EF%BC%89 "一致性（Consistency）")一致性（Consistency）

此处的一致性要和分布式系统中的一致性区分开来。在分布式系统中，一致性主要指多副本间的数据一致性。而此处的 C，更像是为了让 ACID 念着顺口来凑数的，他的官方表述是：在事务开始之前和事务结束以后，数据库能够保持某些不变性（invariants）。这表示写入的数据必须完全符合所有的预设[约束](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%AE%8C%E6%95%B4%E6%80%A7)、[触发器](https://zh.wikipedia.org/wiki/%E8%A7%A6%E5%8F%91%E5%99%A8_(%E6%95%B0%E6%8D%AE%E5%BA%93))、[级联回滚](https://zh.wikipedia.org/wiki/%E7%BA%A7%E8%81%94%E5%9B%9E%E6%BB%9A)等 [^3]。举个例子来说，A 给 B 转账，转账前后，A 和 B 的账户总额应该保持不变。

该性质描述侧重于应用层面，而非数据库本身。boltdb 是一个简单的 KV 引擎，不支持用户自定义约束，这里不再展开。

## [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E5%8E%9F%E5%AD%90%E6%80%A7%EF%BC%88Atomicity%EF%BC%89 "原子性（Atomicity）")原子性（Atomicity）

原子性，从字面意义上来理解是将事务所包含的一组操作打包为一个逻辑单元。但这个角度很容易和并发编程中的原子性相混淆。在事务中，原子性其实更侧重于出现问题时的**可回滚性**（`rollback`），或者说**可丢弃性**（`abortability`），即事务中的操作不能部分执行，要么都成功执行，要么都未执行。

那么 boltdb 是如何实现原子性的呢？可以从主动和被动两个方面来分析。

**主动方面**。用户遇到一些问题，可以主动调用 `tx.Rollback` 进行回滚，undo 该事务到目前为止的所有操作。其主要逻辑包括回滚使用的 freelist，释放一些资源（如锁和节点内存引用）。只读事务结束时必须要调用回滚函数，以关闭事务，防止对读写事务的阻塞，之前文章分析过原因（主要是争抢 remap 时候的锁）。

```go
// Rollback 关闭事务，并且放弃之前的更新. 只读事务结束时必须调用 Rollback。  
func (tx *Tx) rollback() {  
  if tx.db == nil {  
    return  
  }  
  if tx.writable {  
    tx.db.freelist.rollback(tx.meta.txid)  
    tx.db.freelist.reload(tx.db.page(tx.db.meta().freelist))  
  }  
  tx.close()  
}  
  
func (tx *Tx) close() {  
  // ...   
    
  if tx.writable {  
    // ...  
  
    // 释放锁  
    tx.db.rwtx = nil  
    tx.db.rwlock.Unlock()  
    tx.db.statlock.Unlock()  
  } else {  
    // 删除读事务  
    tx.db.removeTx(tx)  
  }  
  
  // 清除引用，释放相关内存.  
  tx.db = nil  
  tx.meta = nil  
  tx.root = Bucket{tx: tx}  
  tx.pages = nil  
}  
```

**被动方面**。在读写事务进行到一半时，如果 boltdb 实例意外挂掉重启后，boltdb 如何保证事务的原子性？这个不体现在某些具体细节的代码中，而是体现在 boltdb 整体的设计里：

1.  读写事务执行过程中，所有的改动都是增量改动，不影响其他只读事务
2.  最后提交时，元信息页落盘成功，才会使得所有增量改动对用户可见

也就是说，使用元信息页作为 “全局指针”，以该指针的写入原子性来保证事务的原子性。如果宕机时，元信息页没有写入完成，所有改动便不会生效，达到了自动回滚的效果。

## [](https://www.qtmuniao.com/2021/04/02/bolt-transaction/#%E9%9A%94%E7%A6%BB%E6%80%A7%EF%BC%88Isolation%EF%BC%89 "隔离性（Isolation）")隔离性（Isolation）

隔离性在数据库系统中是一块重要内容。说起隔离性，一般会提到四个隔离级别：

1.  读未提交（Read uncommitted）
2.  读已提交（Read committed）
3.  可重复读（Repeatable read）
4.  序列化（Serializable ）

从上到下，四个级别的隔离性依次变强，性能依次变差。我在初学这几个隔离级别时，看过好几次都没有记住。后来才了解到描述的不是是什么（概念特征），而是怎么做（实现细节），而且是上个世纪特定数据库的实现。只不过这些名词后来延续了下来，所以如果你也曾为这些名词而苦恼，不要自我怀疑，是这些概念本身有问题 —— 他们英文名字就没起好，中文翻译就更差了。

这里不会详细展开，只是粗略说下他们直觉上的理解，改天有时间单开一篇文章来说说，其中牵扯的东西还挺多。

隔离性是由并发引起的，最好的隔离 —— 序列化，性能最差。理解隔离性的关键，是要注意到，每个事务有起止时间，不是瞬间完成。我们可以把每个事务执行过程看作是时间维度上的一个线段，多个线段并发交错，就会引出各种隔离性问题。而隔离性越差，用户代码编写就越难受，需要自行处理各种不一致的情况。比如你开始读到的一个记录，在后面使用时，还得再次进行检查读出的值是否和数据库当前状态仍然一致。

![isolation-levels.png](https://i.loli.net/2021/04/10/SzeTKMk1glF5Pwu.png)

下面依次简单介绍下四种隔离级别：

1.  读未提交 ：对应**脏读**，在本事务的线段内，会读到其他线段的中间状态。
    
2.  读已提交：对应**不可重复读**，比上个好一些。该级别下不能读到其他事务的未提交状态。但如上图，如果事务 t2 在执行时，多次读某个记录 x 的状态，在事务 t1 未启动前，发现 x = 2，在事务 t1 提交后，发现 x = 3，这便出现了不一致。
    
3.  可重复读：如上图，事务 t2 在整个执行期间，多次读取数据库 x 的状态，无论他事务（如 t1）是否改变 x 状态并提交，事务 t2 都不会感觉到。但是会存在**幻读**的风险。怎么理解呢？最关键的原因在于**写**并发。因为读不到，不代表其他事务的影响不存在。比如事务 t2 开始时，通过查询发现 `id = "qtmuniao"` 的记录为空，于是创建了 `id="qtmuniao"` 的记录，然而在提交时，发现报错说该 id 已经存在。这可能是因为有一个开始的比较晚的事务 t2，也创建了一个 `id="qtmuniao"` 的记录，但是先提交了。于是用户就郁闷了，明明你说没有，但我写又报错，难道我出现幻觉了？这就太扯淡了，但是此级别就只能做到这样了。反而，因为兼顾了性能和隔离性，他是大多数据库的默认级别。
    
    ![phantom-problem.png](https://i.loli.net/2021/04/10/3S7jYQwvKfGJO46.png)
    
4.  序列化：最简单的实现办法就是一把锁来串行化所有的事务。在此基础上如果能提高并发，就需要做很多优化。
    

对于 boltdb 来说，因为不允许并发写，可重复读和序列化在此含义就是一样的。总结来说，boltdb 实现隔离性的方法是：

1.  增量写内存。
2.  穿透读磁盘。

读写事务的变动都在内存中，而只读事务通过 mmap 直接读取的磁盘上的内容，因此读写事务的改动不会为只读事务所见。多个读写事务是串行的，也不会互相影响。而每个只读事务期间所看到的状态，就是该只读事务开始执行时的状态。

# Reference
https://cloud.tencent.com/developer/article/2072934?areaSource=&traceId=
https://www.qtmuniao.com/2021/04/02/bolt-transaction/