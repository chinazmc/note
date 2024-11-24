

Log是LogSegment日志段的容器，里面定义了很多管理日志段的操作。Log 对象是 Kafka Broker最核心的部分：

```
// Log.scala

class Log(@volatile var dir: File,
          @volatile var config: LogConfig,
          @volatile var recoveryPoint: Long = 0L,
          scheduler: Scheduler,
          time: Time = Time.SYSTEM) extends Logging with KafkaMetricsGroup {
}

```

Log中包含两个核心属性：**dir** 和 **logStartOffset**。`dir`是主题分区日志所在的文件夹路径，比如"topic-0"、"topic-1"；而 logStartOffset，表示日志的当前最早位移。

## 一、核心方法

Log 的常见操作分为 4 大部分：

-   **高水位管理操作：**高水位（HW）的概念在 Kafka 中举足轻重，对它的管理，是 Log 最重要的功能之一；
-   **日志段管理：**Log 是日志段的容器，高效组织与管理其下辖的所有日志段对象，是源码要解决的核心问题；
-   **关键位移值管理：**日志定义了很多重要的位移值，比如 Log Start Offset 和 LEO 等，确保这些位移值的正确性，是构建消息引擎一致性的基础；
-   **读写操作：**所谓的操作日志，大体上就是指读写日志，读写操作的作用之大，不言而喻。

我只讲解日志段管理和写日志操作，其余操作读者可参阅源码或胡夕的这个专栏：[《Kafka核心源码解读》](https://time.geekbang.org/column/intro/304)。

### 1.1 日志段管理

Log 是LogSegment的容器，它使用 J.U.C中的 **_ConcurrentSkipListMap_**类来保存所有日志段对象。Kafka 将每个日志段的**起始位移值作为 Key**，这样一来，我们就能够很方便地根据所有日志段的起始位移值对它们进行排序和比较，同时还能快速地找到与给定位移值相近的前后两个日志段：

```
// Log.scala

private val segments: ConcurrentNavigableMap[java.lang.Long, LogSegment] = new ConcurrentSkipListMap[java.lang.Long, LogSegment]

```

> 对于ConcurrentSkipListMap不了解的童鞋，可以去看我的专栏：[《透彻理解Java并发编程》](https://www.tpvlog.com/article/308)。

Kafka 提供了很多策略（包括基于时间维度、空间维度、Log Start Offset 维度），可以根据一定的规则决定哪些日志段可以删除：

  
![](https://files.tpvlog.com/tpvlog/kafka/source/20210615232653585.png)  

### 1.2 写日志操作

我们重点看下Log的写日志操作，也就是append方法。整个流程可以用下面这张图表述：

  
![](https://files.tpvlog.com/tpvlog/kafka/source/20210615232706009.png)  

```
// Log.scala

def append(records: MemoryRecords, assignOffsets: Boolean = true): LogAppendInfo = {
  // 1.分析和校验待写入消息集合，并返回校验结果
  val appendInfo = analyzeAndValidateRecords(records)

  // 如果不需要写入任何消息，直接返回
  if (appendInfo.shallowCount == 0)
    return appendInfo

  // 2.消息格式规整，即删除无效格式消息或无效字节
  var validRecords = trimInvalidBytes(records, appendInfo)

  try {
    lock synchronized {
      if (assignOffsets) {
        // 3.使用当前LEO值作为待写入消息集合中第一条消息的位移值
        val offset = new LongRef(nextOffsetMetadata.messageOffset)
        appendInfo.firstOffset = offset.value
        val now = time.milliseconds
        val validateAndOffsetAssignResult = try {
          LogValidator.validateMessagesAndAssignOffsets(validRecords,
                                                        offset,
                                                        now,
                                                        appendInfo.sourceCodec,
                                                        appendInfo.targetCodec,
                                                        config.compact,
                                                        config.messageFormatVersion.messageFormatVersion,
                                                        config.messageTimestampType,
                                                        config.messageTimestampDifferenceMaxMs)
        } catch {
          case e: IOException => throw new KafkaException("Error in validating messages while appending to log '%s'".format(name), e)
        }
        // 更新校验结果对象类LogAppendInfo
        validRecords = validateAndOffsetAssignResult.validatedRecords
        appendInfo.maxTimestamp = validateAndOffsetAssignResult.maxTimestamp
        appendInfo.offsetOfMaxTimestamp = validateAndOffsetAssignResult.shallowOffsetOfMaxTimestamp
        appendInfo.lastOffset = offset.value - 1
        if (config.messageTimestampType == TimestampType.LOG_APPEND_TIME)
          appendInfo.logAppendTime = now

        // 4.验证消息，确保消息大小不超限
        if (validateAndOffsetAssignResult.messageSizeMaybeChanged) {
          for (logEntry <- validRecords.shallowEntries.asScala) {
            if (logEntry.sizeInBytes > config.maxMessageSize) { BrokerTopicStats.getBrokerTopicStats(topicPartition.topic).bytesRejectedRate.mark(records.sizeInBytes)
              BrokerTopicStats.getBrokerAllTopicsStats.bytesRejectedRate.mark(records.sizeInBytes)
              throw new RecordTooLargeException("Message size is %d bytes which exceeds the maximum configured message size of %d."
                .format(logEntry.sizeInBytes, config.maxMessageSize))
            }
          }
        }

      } else {    // 直接使用给定的位移值，无需自己分配位移值；
                  // 确保消息位移值的单调递增性
        if (!appendInfo.offsetsMonotonic || appendInfo.firstOffset < nextOffsetMetadata.messageOffset)
          throw new IllegalArgumentException("Out of order offsets found in " + records.deepEntries.asScala.map(_.offset))
      }

      // 5.确保消息大小不超限
      if (validRecords.sizeInBytes > config.segmentSize) {
        throw new RecordBatchTooLargeException("Message set size is %d bytes which exceeds the maximum configured segment size of %d."
          .format(validRecords.sizeInBytes, config.segmentSize))
      }

      // 6.执行日志切分，当前日志段剩余容量可能无法容纳新消息集合，因此有必要创建一个新的日志段来保存待写入的所有消息
      val segment = maybeRoll(messagesSize = validRecords.sizeInBytes,
        maxTimestampInMessages = appendInfo.maxTimestamp,
        maxOffsetInMessages = appendInfo.lastOffset)

      // 7.执行真正的消息写入操作，主要调用LogSegment.append方法实现
      segment.append(firstOffset = appendInfo.firstOffset,
        largestOffset = appendInfo.lastOffset,
        largestTimestamp = appendInfo.maxTimestamp,
        shallowOffsetOfMaxTimestamp = appendInfo.offsetOfMaxTimestamp,
        records = validRecords)

      // 8.更新LEO对象，其中，LEO值是消息集合中最后一条消息位移值+1
      updateLogEndOffset(appendInfo.lastOffset + 1)

      // 9.是否需要手动落盘（根据消息数判断）
      // 一般情况下我们不需要设置Broker端参数log.flush.interval.messages, 落盘操作由OS完成
      // 但某些情况下，可以设置该参数来确保高可靠性
      if (unflushedMessages >= config.flushInterval)
        flush()

      appendInfo
    }
  } catch {
    case e: IOException => throw new KafkaStorageException("I/O exception in append to log '%s'".format(name), e)
  }
}

```

日志的写入操作，是通过`LogSegment.append()`方法完成的，我下一章节会对LogSegment进行分析。

### 1.3 日志切分

Log在写日志的过程中，有一个很重要的`maybeRoll`方法，负责进行**日志切分**。所谓日志切分，就是说如果当前日志段的剩余容量无法容纳新消息时，就需要新创建一个日志段：

```
// Log.scala

private def maybeRoll(messagesSize: Int, maxTimestampInMessages: Long, maxOffsetInMessages: Long): LogSegment = {
  val segment = activeSegment
  val now = time.milliseconds
  val reachedRollMs = segment.timeWaitedForRoll(now, maxTimestampInMessages) > config.segmentMs - segment.rollJitterMs
  // 容量不足，执行切分
  if (segment.size > config.segmentSize - messagesSize ||
      (segment.size > 0 && reachedRollMs) ||
      segment.index.isFull || segment.timeIndex.isFull || !segment.canConvertToRelativeOffset(maxOffsetInMessages)) {
    roll(maxOffsetInMessages - Integer.MAX_VALUE)
  } else {
    segment
  }
}

def roll(expectedNextOffset: Long = 0): LogSegment = {
  val start = time.nanoseconds
  lock synchronized {
    val newOffset = Math.max(expectedNextOffset, logEndOffset)
    val logFile = Log.logFile(dir, newOffset)
    val indexFile = indexFilename(dir, newOffset)
    val timeIndexFile = timeIndexFilename(dir, newOffset)
    for(file <- List(logFile, indexFile, timeIndexFile); if file.exists) {
      warn("Newly rolled segment file " + file.getName + " already exists; deleting it first")
      file.delete()
    }

    segments.lastEntry() match {
      case null =>
      case entry => {
        val seg = entry.getValue
        seg.onBecomeInactiveSegment()
        seg.index.trimToValidSize()
        seg.timeIndex.trimToValidSize()
        seg.log.trim()
      }
    }
    // 新建一个日志段
    val segment = new LogSegment(dir,
                                 startOffset = newOffset,
                                 indexIntervalBytes = config.indexInterval,
                                 maxIndexSize = config.maxIndexSize,
                                 rollJitterMs = config.randomSegmentJitter,
                                 time = time,
                                 fileAlreadyExists = false,
                                 initFileSize = initFileSize,
                                 preallocate = config.preallocate)
    val prev = addSegment(segment)
    if(prev != null)
      throw new KafkaException("Trying to roll a new log segment for topic partition %s with start offset %d while it already exists.".format(name, newOffset))
    updateLogEndOffset(nextOffsetMetadata.messageOffset)
    scheduler.schedule("flush-log", () => flush(newOffset), delay = 0L)
    segment
  }
}

```

> 可以通过Kafka Broker端的参数`segment.bytes`设置日志分段的大小，默认为1G，当前正在写入的LogSegment称为_Active LogSegment_。

## 二、总结

本章，我对Log对象进行了简单的讲解。Log是LogSegment日志段的容器，里面定义了很多管理日志段的操作，我们目前只需要关注它的写日志方法即可。

