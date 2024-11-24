# bitcask存储引擎

bitcask是一个使用Erlang写的key-value存储引擎。Bitcask的起源和一个分布式key-value数据库 Riak有很密切的关系。在Riak的集群里，每个node使用插件式的存储引擎，几乎所有key-value类型的存储引擎都可以作为单个node节点的存储引擎。关于Riak的详细介绍，有机会后面再讲。

# 设计理念

在MySQL和postgresql中，除了保存在disk上的真正的数据库数据外，还有额外的日志文件，MySQL中是binlog，pg中是wal 文件。这些日志文件在备份、还原、建立从库的时候非常有用。

在bitcask中的设计中，相对就比较简单，日志文件本身就是数据库。备份起来也相当简单，只要把数据目录的所有文件拷贝一份，在另一个服务器上重建索引就行了。简要说起来有下面几点：

-   使用RAM（内存）存储一个哈希表，哈希表上的value指向文件系统上的文件，以及该key对应的值在该文件中的具体位置。
-   无论是插入、更新还是删除，都是append一条记录到一个特殊格式的文件。
-   每次append记录之后，更新内存里的那个哈希表
-   每个文件有最大空间限制，这个文件写满之后，写下一个，写过的之后永远不会再改变。
-   有一个merge进程，用来合并**老数据**，防止文件大小无限积累。
-   读缓存：其他的数据库系统，比如MySQL，PostgreSql有相当复杂的设计，bitcask没有相关设计，而是依赖于操作系统内核的文件系统缓存。

# 优势

-   写操作：由于所有的操作都是append，所以速度是极快的，因为不会涉及到seek（不知道如何用中文表述，手动捂脸）。
-   读操作：O(1)复杂度的disk查询，因为内存里的哈希表已经明确表名这个值所在的位置了。
-   设计简单，很容易理解，这也是一个优势

# 详细解释

一个bitcask实例就是一个目录，在设计上强制在任意时刻，只有一个操作系统进程可以打开bitcask进行写操作，这个进程就可以看作是bitcask服务。在任意时刻，这个目录中只有一个文件是active的，只有这个文件是可以被写入的。当这个active的文件大小达到一个临界值的时候，bitcask就会创建一个新的文件，用来取代当前的active文件。被取代的文件被称为老文件，之后永远都是不可变的，不会再有任何进程往里面写入数据。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/7/168c5f689be8831c~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

Active文件写操作全是append，意味着顺序写入操作不需要disk seeking。每一个记录的格式如下图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/7/168c5f689c951b46~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

-   crc：一个检验值，在这里可以忽略
-   tstamp：时间戳
-   ksz: key的大小
-   value_sz : value的大小。

需要注意的是，删除操作，不过是插入一条value为某个特殊值的记录。

这些记录组合起来，就构成了一个bitcask文件：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/7/168c5f68a387f710~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

当append操作完成时，内存里的一个叫做keydir的数据结构就会被更新。一个keydir就是一个简单的哈希表，key就是插入数据的key，value指向了插入数据value在文件系统中的具体位置。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/7/168c5f68a165231c~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

-   file_id : 前面说过，bitcask有active文件，也有老数据文件，这个file_id就是指向了一个其中的文件。
-   value_pos : value在该文件的起始位置。
-   value_sz : value大小

写入发生时，原来的老数据依然还是保存在磁盘上，但是新的读取这个key的请求都会使用keydir上最新的数据。后面会说到，有一个合并进程，最终会删除不会再使用到的老数据。

读取操作是很简单的，根本不需要操作一次的disk seek。从内存中的keydir查询key，从这里知道了value所在的file_id，位置，大小，然后只要调用系统的读取接口就行了。一般操作系统都还会有自己独立的disk读缓存，所以这个操作实际上可以更快。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/7/168c5f689cd6d767~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

现在来说说前面提到的合并操作。其实很简单，**合并进程遍历所有的老文件，产生一个只包含当前最新版本数据的文件。**这部分完成的时候，还会产生一个hint file，这个文件本质上和data 文件一样，只不过他们保存的是value所在文件上的位置，而不是value本身。这个文件可以加速从目录文件重建keydir的过程。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/7/168c5f68a1eef56d~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

当bitcask被一个Erlang进程打开时，它会检查同一个VM里，是不是有另一个Erlang进程在使用这个bitcask，如果是的话，就会和那个进程共享同一个keydir。如果没有的话，它就会扫描所有的数据文件，构建一个新的keydir，对于那些有hint file的文件，这个构建过程会有显著提高。

# 总结

我接触的第一个key-value存储引擎是redis。redis的所有数据都是装在内存的（每隔一段时间会把数据持久化保存到磁盘），这意味着redis的读写速度都非常快。但是这有一个限制，那就是单机redis存储的数据不能大于内存本身。而bitcask的最大限制是内存必须装得下所有的key，因为bitcask的value是存在磁盘上的。所以相比redis，bitcask的存在意义不是和redis比速度，而是当你的数据用redis存不下的时候，可以考虑稍微损失一丢丢速度，试试bitcask。


# Reference
https://juejin.cn/post/6844903774377476109
https://blog.shunzi.tech/post/vldbj-2018lsm-based-storage-techniques-a-survey/