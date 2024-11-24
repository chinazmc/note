本章，我来带大家学习一下 Kafka 源码中的索引对象。在Kafka中索引类的继承关系如下图：

  
![](https://files.tpvlog.com/tpvlog/kafka/source/20210617232334076.png)  

-   **AbstractIndex：**定义了最顶层的抽象类，这个类封装了所有索引类型的公共操作；
-   **LazyIndex：** AbstractIndex 的一个包装类，实现索引项延迟加载，这个类主要是为了提升性能；
-   **OffsetIndex：**位移索引，保存`<相对位移值，文件磁盘物理位置 >`；
-   **TimeIndex：**时间戳索引，保存`< 时间戳，相对位移值 >`；
-   **TransactionIndex：**事务索引，为已中止事务（Aborted Transcation）保存重要的元数据信息，只有启用 Kafka 事务后，这个索引才有可能出现。

## 一、索引属性

我们先来看下AbstractIndex中定义的一些重要属性：

```
// AbstractIndex.scala
abstract class AbstractIndex[K, V](@volatile var file: File, val baseOffset: Long,
                                   val maxIndexSize: Int = -1) extends Logging {
}

```

AbstractIndex 定义了 3 个属性字段，它的所有子类自动地继承了这 3 个字段：

-   **索引文件（file）：**每个索引对象在磁盘上都对应了一个索引文件；
-   **起始位移（baseOffset）：**日志段的起始位移值。举个例子，日志文件是 00000000000000000123.log，正常情况下，还有一组索引文件 00000000000000000123.index、00000000000000000123.timeindex 等。这里的“123”就是这组文件的起始位移值，也就是 baseOffset；
-   **索引文件最大字节数（maxIndexSize）**：控制索引文件的大小。通过参数`segment.index.bytes` 设置，默认10MB。

## 二、索引写入

LogSegment的append方法中会完成稀疏索引的写入，内部实际是调用了`OffsetIndex.append()`方法：

```
// LogSegment.scala

def append(firstOffset: Long, largestOffset: Long, largestTimestamp: Long, 
           shallowOffsetOfMaxTimestamp: Long, records: MemoryRecords) {
  if (records.sizeInBytes > 0) {
    //...
    if(bytesSinceLastIndexEntry > indexIntervalBytes) {
      // 重点是这里写入索引
      index.append(firstOffset, physicalPosition)     
      timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestamp)
      bytesSinceLastIndexEntry = 0
    }
    bytesSinceLastIndexEntry += records.sizeInBytes
  }
}

```

### 2.1 内存映射

我们来看下OffsetIndex的append方法， 它的内部引用了一个`mmap`变量，事实上，这个`mmap`就是Java中的**MappedByteBuffer**：

```
// OffsetIndex.scala

def append(offset: Long, position: Int) {
  inLock(lock) {
    // 1.判断索引文件是否写满
    require(!isFull, "Attempt to append to a full index (size = " + _entries + ").")
    // 2.必须满足以下任意条件才允许写入索引项： 
    // CASE ONE：当前索引文件为空 
    // CASE TWO：要写入的位移大于当前所有已写入的索引项的位移
    if (_entries == 0 || offset > _lastOffset) {
      // 3.1 向mmap中写入相对位移值
      mmap.putInt((offset - baseOffset).toInt)
      // 3.2 向mmap中写入物理位置信息
      mmap.putInt(position)
      // 4.更新其他元数据统计信息，如当前索引项计数器_entries和当前索引项最新位移值_lastOffset
      _entries += 1
      _lastOffset = offset
      // 5.校验写入的索引项格式
      require(_entries * entrySize == mmap.position, entries + " entries but file position in index is " + mmap.position + ".")
    } else {
      throw new InvalidOffsetException("Attempt to append an offset (%d) to position %d no larger than the last offset appended (%d) to %s."
        .format(offset, entries, _lastOffset, file.getAbsolutePath))
    }
  }
}

```

整个写入流程可以用下面这张图描述：

  
![](https://files.tpvlog.com/tpvlog/kafka/source/20210617232347690.png)  

我们再来看下mmap的创建过程：

```
// AbstractIndex.scala

protected var mmap: MappedByteBuffer = {
  // 1.创建索引文件
  val newlyCreated = file.createNewFile()
  // 2.以writable指定的方式（读写方式或只读方式）打开索引文件
  val raf = new RandomAccessFile(file, "rw")
  try {
    if(newlyCreated) {
      if(maxIndexSize < entrySize) // 预设的索引文件大小不能太小，如果连一个索引项都保存不了，直接抛出异常
        throw new IllegalArgumentException("Invalid max index size: " + maxIndexSize)
      // 3.设置索引文件长度，roundDownToExactMultiple方法计算的是不超过maxIndexSize的最大整数倍entrySize 
      // 比如maxIndexSize=1234567，entrySize=8，那么调整后的文件长度为1234560
      raf.setLength(roundDownToExactMultiple(maxIndexSize, entrySize))
    }

    // 4.更新索引长度字段_length
    val len = raf.length()
    // 5.创建MappedByteBuffer对象
    val idx = raf.getChannel.map(FileChannel.MapMode.READ_WRITE, 0, len)

    // 6.如果是新创建的索引文件，将MappedByteBuffer对象的当前位置置成0 
    // 如果索引文件已存在，将MappedByteBuffer对象的当前位置置成最后一个索引项所在的位置
    if(newlyCreated)
      idx.position(0)
    else
      idx.position(roundDownToExactMultiple(idx.limit, entrySize))
    // 7.返回创建的MappedByteBuffer对象
    idx
  } finally {
    CoreUtils.swallow(raf.close())
  }
}

```

索引的写入**使用了内存映射文件** ，使用内存映射文件的优势在于，它有很高的 I/O 性能。**对于索引这种小文件，文件内存可以被直接映射到一段虚拟内存（Page Cache）上**，这样访问内存映射文件的速度要远远快于普通的读写文件速度。

另外，在Linux操作系统中，映射的内存区域实际上就是内核的页缓存（Page Cache），这意味着数据不需要重复在内核态空间和用户态空间之间拷贝，避免了不必要的上下文切换消耗。

## 三、索引查找

Kakfa索引查找的过程我已经详细讲解过了，当时讲的是Kafka使用了二分查找算法，但这并不完全准确，事实上，Kafka对索引的二分查找进行了优化。

我们知道，Linux使用了页缓存（Page Cache）来实现内存映射，并且一般使用LRU（Least Recently Used）机制来管理页缓存。

索引写入时使用了Page Cahce，那么当 Kafka 查询索引时，就很可能出现**缺页中断（Page Fault）问题**。所谓Page Fault，就是说Kafka 线程会被阻塞，等待对应的索引项从磁盘读出并放入到页缓存中，这种加载过程可能长达 1 秒。

### 3.1 冷热分离

那么，Kafka是如何解决这个问题的呢？

Kafka对二分查找算法进行了改进，采用了**缓存友好的搜索算法**。总体的思路是：  
将所有索引项分成两个部分：**热区（Warm Area）**和**冷区（Cold Area）**，由于大部分查询集中在索引项尾部，那么把尾部的_8192字节_设置为热区，永远保存在缓存中，然后分别在这两个区域内执行二分查找算法，如下图所示：

  
![](https://files.tpvlog.com/tpvlog/kafka/source/20210617232400500.png)  

  
这个改进版算法的**最大好处**在于：**查询最热那部分数据所遍历的 Page 永远是固定的，因此大概率在页缓存中，从而避免无意义的 Page Fault**。

## 四、总结

本章，我对Kafka中的索引进行了深入讲解。有两个重点：

-   **AbstractIndex：**这是 Kafka 所有类型索引的抽象父类，里面的 mmap 变量是实现索引机制的核心，Kakfa采用了内存映射文件实现了索引的写入，极大提升了IO性能；
-   **改进版二分查找算法：**Kafka对二分查找算法进行了改进，将所有索引项分成两个部分：热区（Warm Area）和冷区（Cold Area），然后分别在这两个区域内执行二分查找算法，从而提升页缓存的使用率，避免缺页中断（Page Fault）问题。

