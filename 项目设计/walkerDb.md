# data 数据格式
数据文件有
```go
DataFileNameSuffix    = ".data"  
HintFileName          = "hint-index"  
MergeFinishedFileName = "merge-finished"  
SeqNoFileName         = "seq-no"
```

日志格式：
```go
// EncodeLogRecord 对 LogRecord 进行编码，返回字节数组及长度  
//  
// +-------------+-------------+-------------+--------------+-------------+--------------+  
// | crc 校验值  |  type 类型   |    key size |   value size |      key    |      value   |
// +-------------+-------------+-------------+--------------+-------------+--------------+  
//     4字节          1字节        变长（最大5）   变长（最大5）     变长           变长
```
日志类型有：
```go
const (  
   LogRecordNormal LogRecordType = iota  
   LogRecordDeleted  
   LogRecordTxnFinished)
```

crc32是循环冗余校验

读文件的话就是根据key从b树中获取文件id和offset，然后从这个文件中这个offset中根据格式获取数据。

写入流程是先获取activeFile，然后判断活跃文件的剩余大小是否不够写入，如果不够写入，就持久化活跃文件，然后加入旧文件map《fileId》=\*db,然后打开一个新文件再写入，然后再判断用户配置中是否配置了写了多少字节就持久化一次来进行持久化。

内存中会维护一个b树用来表示key对应的文件id和offset的关系。

事务的话就是通过writeBatch来做，writeBatch维护一个map\[key]\*data.LogRecord,等到提交的时候，先用writeBatch加锁，然后再锁db的锁，这个顺序是在全部代码中是唯一的，不可能倒过来的。然后再用一个全局唯一递增的事务id，写到key中，key就会变成 seqno+key写道磁盘中，获取到很多pos，然后再写到内存中，内存中还是原来的key,磁盘中的key就是seqnokey。
然后在db启动的过程中，需要将同一批事务一起写到内存中。

merge的过程是先将当前activeFile置为旧file,然后提供一个新的activeFile用于写入，然后记录下来这个新的activeFile的fileId作为unmergeFile，然后启动一个新的db，并且将旧文件排序从小到大，然后一条条数据判断跟当前内存中的数据是否一致，比如文件id是否一致，key是否一致，offset是否一致，如果一致，就写入mergedb，最后，merge完之后写一个merge完成的数据标记日志。
然后将旧文件关闭并删除，将新文件挪过去。

merge的过程中需要写hint文件用于启动db的时候快速建立内存索引，因为merge的过程中pos会变，所以将这个变动记录下来，后面启动的时候就可以直接读出来key->pos

我们使用内存文件映射mmap io来进行启动的加速，mmap指的是将磁盘文件映射到内存地址空间，操作系统在读文件的时候，会触发缺页中断，将数据加载到内存，这个内存是用户可见的，相较于传统的文件io，避免了内核态到数据态的数据拷贝，但是加载完成之后，要将iomanager切换到原来的io.
## 1. mmap 基础概念

mmap 即 memory map，也就是内存映射。

mmap 是一种内存映射文件的方法，即**将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系**。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用 read、write 等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。如下图所示：

![img](https://spongecaptain.cool/SimpleClearFileIO/images/200501092691998.png)

mmap 具有如下的特点：

1. mmap 向应用程序提供的内存访问接口是内存地址连续的，但是对应的磁盘文件的 block 可以不是地址连续的；
2. mmap 提供的内存空间是虚拟空间（虚拟内存），而不是物理空间（物理内存），因此完全可以分配远远大于物理内存大小的虚拟空间（例如 16G 内存主机分配 1000G 的 mmap 内存空间）；
3. mmap 负责映射文件逻辑上一段连续的数据（物理上可以不连续存储）映射为连续内存，而这里的文件可以是磁盘文件、驱动假造出的文件（例如 DMA 技术）以及设备；
4. mmap 由操作系统负责管理，对同一个文件地址的映射将被所有线程共享，操作系统确保线程安全以及线程可见性；

mmap 的设计很有启发性。基于磁盘的读写单位是 block（一般大小为 4KB），而基于内存的读写单位是地址（虽然内存的管理与分配单位是 4KB）。换言之，CPU 进行一次磁盘读写操作涉及的数据量至少是 4KB，但是进行一次内存操作涉及的数据量是基于地址的，也就是通常的 64bit（64 位操作系统）。mmap 下进程可以采用指针的方式进行读写操作，这是值得注意的。

## 2. mmap 的 I/O 模型

mmap 也是一种零拷贝技术，其 I/O 模型如下图所示：

![mmap](https://spongecaptain.cool/SimpleClearFileIO/images/mmap.jpg)

mmap 技术有如下特点：

1. 利用 DMA 技术来取代 CPU 来在内存与其他组件之间的数据拷贝，例如从磁盘到内存，从内存到网卡；
2. 用户空间的 mmap file 使用虚拟内存，实际上并不占据物理内存，只有在内核空间的 kernel buffer cache 才占据实际的物理内存；
3. `mmap()` 函数需要配合 `write()` 系统调动进行配合操作，这与 `sendfile()` 函数有所不同，后者一次性代替了 `read()` 以及 `write()`；因此 mmap 也至少需要 4 次上下文切换；
4. mmap 仅仅能够避免内核空间到用户空间的全程 CPU 负责的数据拷贝，但是内核空间内部还是需要全程 CPU 负责的数据拷贝；

利用 `mmap()` 替换 `read()`，配合 `write()` 调用的整个流程如下：

1. 用户进程调用 `mmap()`，从用户态陷入内核态，将内核缓冲区映射到用户缓存区；
2. DMA 控制器将数据从硬盘拷贝到内核缓冲区（可见其使用了 Page Cache 机制）；
3. `mmap()` 返回，上下文从内核态切换回用户态；
4. 用户进程调用 `write()`，尝试把文件数据写到内核里的套接字缓冲区，再次陷入内核态；
5. CPU 将内核缓冲区中的数据拷贝到的套接字缓冲区；
6. DMA 控制器将数据从套接字缓冲区拷贝到网卡完成数据传输；
7. `write()` 返回，上下文从内核态切换回用户态。

### 读取过程

当我们通过 mmap 读取文件时，将经历以下步骤：

1. 在当前用户虚拟内存空间中分配一片 **指定映射大小** 的虚拟内存区域。
2. 将磁盘中的文件映射到这片内存区域，等待后续 **按需** 进行页面调度。
3. 当CPU真正访问数据时，触发 **缺页异常** 将所需的数据页从磁盘拷贝到物理内存，并将物理页地址记录到页表。
4. 进程通过页表得到的物理页地址访问文件数据。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/8/17022db530902c43~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

而作为对比，当通过 **标准IO** 读取一个文件时，步骤为：

1. 将 **完整** 的文件从磁盘拷贝到物理内存（内核空间）。
2. 将完整文件数据从 **内核空间** 拷贝到 **用户空间** 以供进程访问。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/8/17022db532250284~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

## 3. mmap 的优势

**1.简化用户进程编程**

在用户空间看来，通过 mmap 机制以后，磁盘上的文件仿佛直接就在内存中，把访问磁盘文件简化为按地址访问内存。这样一来，应用程序自然不需要使用文件系统的 write（写入）、read（读取）、fsync（同步）等系统调用，因为现在只要面向内存的虚拟空间进行开发。

但是，这并不意味着我们不再需要进行这些系统调用，而是说这些系统调用由操作系统在 mmap 机制的内部封装好了。

**（1）基于缺页异常的懒加载**

出于节约物理内存以及 mmap 方法快速返回的目的，mmap 映射采用懒加载机制。具体来说，通过 mmap 申请 1000G 内存可能仅仅占用了 100MB 的虚拟内存空间，甚至没有分配实际的物理内存空间。当你访问相关内存地址时，才会进行真正的 write、read 等系统调用。CPU 会通过陷入缺页异常的方式来将磁盘上的数据加载到物理内存中，此时才会发生真正的物理内存分配。

**（2）数据一致性由 OS 确保**

当发生数据修改时，内存出现脏页，与磁盘文件出现不一致。mmap 机制下由操作系统自动完成内存数据落盘（脏页回刷），用户进程通常并不需要手动管理数据落盘。

**2.读写效率提高：避免内核空间到用户空间的数据拷贝**

简而言之，mmap 被认为快的原因是因为建立了页到用户进程的虚地址空间映射，以读取文件为例，避免了页从内核空间拷贝到用户空间。

**3.避免只读操作时的 swap 操作**

虚拟内存带来了种种好处，但是一个最大的问题在于所有进程的虚拟内存大小总和可能大于物理内存总大小，因此当操作系统物理内存不够用时，就会把一部分内存 swap 到磁盘上。

在 mmap 下，如果虚拟空间没有发生写操作，那么由于通过 mmap 操作得到的内存数据完全可以通过再次调用 mmap 操作映射文件得到。但是，通过其他方式分配的内存，在没有发生写操作的情况下，操作系统并不知道如何简单地从现有文件中（除非其重新执行一遍应用程序，但是代价很大）恢复内存数据，因此必须将内存 swap 到磁盘上。

**4.节约内存**

由于用户空间与内核空间实际上共用同一份数据，因此在大文件场景下在实际物理内存占用上有优势。

## 4. mmap 不是银弹

mmap 不是银弹，这意味着 mmap 也有其缺陷，在相关场景下的性能存在缺陷：

1. 由于 mmap 使用时必须实现指定好内存映射的大小，因此 mmap 并不适合变长文件；
2. 如果更新文件的操作很多，mmap 避免两态拷贝的优势就被摊还，最终还是落在了大量的脏页回写及由此引发的随机 I/O 上，所以在随机写很多的情况下，mmap 方式在效率上不一定会比带缓冲区的一般写快；
3. 读/写小文件（例如 16K 以下的文件），mmap 与通过 read 系统调用相比有着更高的开销与延迟；同时 mmap 的刷盘由系统全权控制，但是在小数据量的情况下由应用本身手动控制更好；
4. mmap 受限于操作系统内存大小：例如在 32-bits 的操作系统上，虚拟内存总大小也就 2GB，但由于 mmap 必须要在内存中找到一块连续的地址块，此时你就无法对 4GB 大小的文件完全进行 mmap，在这种情况下你必须分多块分别进行 mmap，但是此时地址内存地址已经不再连续，使用 mmap 的意义大打折扣，而且引入了额外的复杂性；
5. 1. mmap 本身开销比 read 大，因为mmap涉及更多的系统调用，需要触发缺页中断，更改虚拟内存映射。

## 5. mmap 的适用场景

mmap 的适用场景实际上非常受限，在如下场合下可以选择使用 mmap 机制：

1. 多个线程以只读的方式同时访问一个文件，这是因为 mmap 机制下多线程共享了同一物理内存空间，因此节约了内存。案例：多个进程可能依赖于同一个动态链接库，利用 mmap 可以实现内存仅仅加载一份动态链接库，多个进程共享此动态链接库。
2. mmap 非常适合用于进程间通信，这是因为对同一文件对应的 mmap 分配的物理内存天然多线程共享，并可以依赖于操作系统的同步原语；
3. mmap 虽然比 sendfile 等机制多了一次 CPU 全程参与的内存拷贝，但是用户空间与内核空间并不需要数据拷贝，因此在正确使用情况下并不比 sendfile 效率差；

# 为什么不要在数据库系统中使用mmap?
## 问题引出

作者引出了两种数据库系统中对于文件I/O管理的选择：

- 开发者自己实现buffer pool来管理文件I/O读入内存的数据
- 使用Linux操作系统实现的MMAP系统调用将文件直接映射到用户地址空间，并且利用对开发者透明的page cache来实现页面的换入换出

由于第二种方案，开发者不需要手动管理内存，实现起来简单，因此很多数据库系统曾经使用MMAP来代替buffer pool，但是由于一些问题导致它们最终弃用了MMAP（这一点也是论文的一个重要的论据），改为自己管理文件I/O。

本论文通过理论的引入和实验的结论证明MMAP不适合用在数据库系统中。

## 理论介绍

论文首先介绍了程序是如何通过MMAP系统调用访问到文件，结合着配图，原理十分清晰。

![](https://fanjingbo.com/images/mmap.png)

1. 程序调用MMAP返回了指向文件内容的指针
2. 操作系统保留了一部分虚拟地址空间，但是并没有开始加载文件
3. 程序开始使用指针获取文件的内容
4. 操作系统尝试在物理内存获取内存页
5. 由于内存页此时不存在，因此触发了页错误，开始从物理存储将第3步获取的那部分内容加载到物理内存页中
6. 操作系统将虚拟地址映射到物理地址的页表项（Page Table Entry）加入到页表中
7. 上述操作使用的CPU核心会将页表项加载到页表缓存（TLB）中

然后介绍了MMAP和其相关的几个API

- mmap:介绍了MAP_SHARED和MAP_PRIVATE在可见性上的区别
- madvise:介绍了MADV_NORMAL，MADV_RANDOM和MADV_SEQUENTIAL在预读上的区别
- mlock:可以尝试性的锁住内存中的页面，一定程度上防止被写回存储。（然而并不是确定性的锁住）
- msync:将页面从内存写回存储的接口

论文列举了几种曾经使用过MMAP的数据库例子：

![](https://fanjingbo.com/images/mmap-based-dbms.png)

## 问题陈述

论文列举了三点关于使用MMAP可能引起的问题：

### 问题一 事务安全

由于MMAP中页面写回存储的时机不受程序控制，因此当commit还没有发生时，可能会有一部分脏页面已经写回存储了。此时原子性就会失效，在过程中的查询会看到中间状态。

为了解决事务安全问题，有以下三个解决方式：

- 操作系统写时复制（copy-on-write）：使用MAP_PRIVATE标识位创建一个独立的写空间（物理内存复制），写操作在这个写空间执行，读操作仍读取原来的空间。同时使用WAL（write-ahead log）来保证写入操作被持久化。事务提交之后将写空间新增的内容复制到读空间。

> 这里存在一个疑问，原文是这样表述的： When a transaction commits, the DBMS flushes the corresponding WAL records to secondary storage and uses a separate background thread to apply the committed changes to the primary copy. 我的理解是：持久化的不应该是WAL的记录，而应该是写空间的msync，只有msync意外中断的时候，才需要从WAL恢复。这个理解有可能是错误的，等我之后理解了再来更新吧。

- 用户空间写时复制（copy-on-write）：类似操作系统写时复制，不过写空间在用户空间开辟，同样写入WAL保证持久性。事务提交后，从用户空间写回读空间。

> 这里提到了： Since copying an entire page is wasteful for small changes, some DBMSs support applying WAL records directly to the mmap-backed memory. 特殊提到了一些数据库支持WAL直接写入，说明不是大多数数据库的行为，也验证了上面疑问中我的理解可能是对的。

- 影子分页：影子分页类似第一种方法，不过没有WAL的参与，而是直接出来两份分页，一份只读的，另一份可写，事务提交就是在可写页表做msync之后再让只读页表可以读到最新的页。

### 问题二 I/O停顿

作者主要提到的就是，由于MMAP将文件加载到内存的过程是操作系统控制的，所以无法保证将要查询的页面在内存中，这时就会出现page cache的换入换出，导致I/O停顿。

这一点比较好理解，毕竟操作系统使用page cache并没有针对数据库场景进行特别的优化，那么很有可能在页面将要被使用的时候，由于page cache不足要使用的页面被换出，从而影响性能。

### 问题三 错误处理

首先是MMAP对文件内容的校验要以单个页面为单位，不能基于多个页面来做，因为有可能要使用的页面会被换出。

另外，对于一些使用内存不安全语言写的数据库系统（感觉大部分都是用内存不安全的C++写的），指针错误可能导致页面问题。使用buffer pool可以通过写入前的检查规避这个问题，但是MMAP会默默将错误的页面写到存储中。

还有，MMAP要应对的系统调用会出现的SIGBUS信号报错，相比之下使用其他I/O方式的buffer pool就能比较轻松的处理I/O错误。

### 问题四 性能问题

其实讲到这里，作者才算是“图穷匕见”，因为这片论文后面的实验主要想证明的就是这一点。

业界里面大家普遍认为MMAP比传统read/write更快，这主要基于以下两点原因：

1. 负责文件映射操作，并且处理page fault的是内核，而不是应用程序
2. MMAP帮助避免了用户空间中的额外的复制操作，相应的也减少了内存的占用

接下来作者提出了他们的发现：MMAP相对于传统read/write I/O在目前高带宽的存储设备上是更差的。并指出了三点原因：

1. 页表竞争

> 这一点没有太理解，下面的解释是： Finally, the OS must synchronize the page table, which becomes highly contended with many concurrent threads. 但是我理解线程切换应该不需要切换页表

1. 单线程的页面换出： 页面换出是单线程的（使用kswapd），这点与CPU相关
2. TLB shootdowns 当一个页面失效时，每个核心的TLB都需要一次中断来做刷新操作，这个操作可能会消耗几千个时钟周期，是非常耗时的。

## 实验说明

这里的实验就不详细写了，有兴趣的可以去看原文。作者主要分了随机读写和顺序读写两个实验来做，证明了直接I/O比MMAP有几倍到几十倍的速度优势。

## 总结

Andy和他的学生无疑是buffer pool的拥趸，他们代表着学术界，不建议在数据库系统中使用MMAP。但我们看一下工业界，还是有很多数据库系统在使用MMAP。这可能也是这两类人的立场不同导致的吧。对于学术界来说，性能是最重要的，成本并不是需要考虑的重点。而工业界则是追求在能接受的成本下实现最好的性能。 所以说，计算机科学领域没有银弹，就像I/O方式的选择一样，从传统的read/write到mmap，再到直接I/O和异步I/O，每一种方式都有人在使用，并不是说异步I/O性能好就能吸引所有人使用，毕竟要受到成本和其他因素的限制。 不过呢，还是希望工业界能够在技术追求上内卷起来，毕竟谁不愿意搞出来一个很叼的东西呢。感觉ScyllaDB就是个不错的例子，也希望他们能火起来吧。

# epoll
## **什么是epoll**

epoll是什么？按照man手册的说法：是为处理大批量句柄而作了改进的poll。当然，这不是2.6内核才有的，它是在2.5.44内核中被引进的(epoll(4) is a new API introduced in Linux kernel 2.5.44)，它几乎具备了之前所说的一切优点，被公认为Linux2.6下性能最好的多路I/O就绪通知方法。

## **epoll的相关系统调用**

epoll只有epoll_create,epoll_ctl,epoll_wait 3个系统调用。

**1. int epoll_create(int size);**

创建一个epoll的句柄。自从linux2.6.8之后，size参数是被忽略的。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

**2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);**

epoll的**事件注册函数**，它不同于select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。

第一个参数是epoll_create()的返回值。

第二个参数表示动作，用三个宏来表示：

EPOLL_CTL_ADD：注册新的fd到epfd中；

EPOLL_CTL_MOD：修改已经注册的fd的监听事件；

EPOLL_CTL_DEL：从epfd中删除一个fd；

第三个参数是需要监听的fd。

第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
events可以是以下几个宏的集合：

EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；

EPOLLOUT：表示对应的文件描述符可以写；

EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；

EPOLLERR：表示对应的文件描述符发生错误；

EPOLLHUP：表示对应的文件描述符被挂断；

EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。

EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

**3. int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);**

收集在epoll监控的事件中已经发送的事件。参数events是分配好的epoll_event结构体数组，epoll将会把发生的事件赋值到events数组中（events不可以是空指针，内核只负责把数据复制到这个events数组中，不会去帮助我们在用户态中分配内存）。maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。如果函数调用成功，返回对应I/O上已准备好的文件描述符数目，如返回0表示已超时。

## **epoll工作原理**

epoll同样只告知那些就绪的文件描述符，而且当我们调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可

另一个本质的改进在于epoll采用基于事件的就绪通知方式。在select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。

**Epoll的2种工作方式-水平触发（LT）和边缘触发（ET）**

假如有这样一个例子：

1. 我们已经把一个用来从管道中读取数据的文件句柄(RFD)添加到epoll描述符

2. 这个时候从管道的另一端被写入了2KB的数据

3. 调用epoll_wait(2)，并且它会返回RFD，说明它已经准备好读取操作

4. 然后我们读取了1KB的数据

5. 调用epoll_wait(2)......

Edge Triggered 工作模式：

如果我们在第1步将RFD添加到epoll描述符的时候使用了EPOLLET标志，那么在第5步调用epoll_wait(2)之后将有可能会挂起，因为剩余的数据还存在于文件的输入缓冲区内，而且数据发出端还在等待一个针对已经发出数据的反馈信息。只有在监视的文件句柄上发生了某个事件的时候 ET 工作模式才会汇报事件。因此在第5步的时候，调用者可能会放弃等待仍在存在于文件输入缓冲区内的剩余数据。在上面的例子中，会有一个事件产生在RFD句柄上，因为在第2步执行了一个写操作，然后，事件将会在第3步被销毁。因为第4步的读取操作没有读空文件输入缓冲区内的数据，因此我们在第5步调用 epoll_wait(2)完成后，是否挂起是不确定的。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。最好以下面的方式调用ET模式的epoll接口，在后面会介绍避免可能的缺陷。

i 基于非阻塞文件句柄

ii 只有当read(2)或者write(2)返回EAGAIN时才需要挂起，等待。但这并不是说每次read()时都需要循环读，直到读到产生一个EAGAIN才认为此次事件处理完成，当read()返回的读到的数据长度小于请求的数据长度时，就可以确定此时缓冲中已没有数据了，也就可以认为此事读事件已处理完成。

Level Triggered 工作模式

相反的，以LT方式调用epoll接口的时候，它就相当于一个速度比较快的poll(2)，并且无论后面的数据是否被使用，因此他们具有同样的职能。因为即使使用ET模式的epoll，在收到多个chunk的数据的时候仍然会产生多个事件。调用者可以设定EPOLLONESHOT标志，在 epoll_wait(2)收到事件后epoll会与事件关联的文件句柄从epoll描述符中禁止掉。因此当EPOLLONESHOT设定后，使用带有 EPOLL_CTL_MOD标志的epoll_ctl(2)处理文件句柄就成为调用者必须作的事情。

LT(level triggered)是epoll**缺省的工作方式**，并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你 的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表．

ET (edge-triggered)是高速工作方式，只支持no-block socket，它效率要比LT更高。ET与LT的区别在于，当一个新的事件到来时，ET模式下当然可以从epoll_wait调用中获取到这个事件，可是如果这次没有把这个事件对应的套接字缓冲区处理完，在这个套接字中没有新的事件再次到来时，在ET模式下是无法再次从epoll_wait调用中获取这个事件的。而LT模式正好相反，只要一个事件对应的套接字缓冲区还有数据，就总能从epoll_wait中获取这个事件。

因此，LT模式下开发基于epoll的应用要简单些，不太容易出错。而在ET模式下事件发生时，如果没有彻底地将缓冲区数据处理完，则会导致缓冲区中的用户请求得不到响应。

图示说明：

![](https://ask.qcloudimg.com/http-save/yehe-1408376/dj6ql6an2a.jpeg?imageView2/2/w/1200)

Nginx默认采用ET模式来使用epoll。

## **epoll的优点：**

**1.支持一个进程打开大数目的socket描述符(FD)**

select 最不能忍受的是一个进程所打开的FD是有一定限制的，由FD_SETSIZE设置，默认值是2048。对于那些需要支持的上万连接数目的IM[服务器](https://cloud.tencent.com/product/cvm?from=20065&from_column=20065)来说显然太少了。这时候你一是可以选择修改这个宏然后重新编译内核，不过资料也同时指出这样会带来网络效率的下降，二是可以选择多进程的解决方案(传统的 Apache方案)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。不过 epoll则没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

**2.IO效率不随FD数目增加而线性下降**

传统的select/poll另一个致命弱点就是当你拥有一个很大的socket集合，不过由于网络延时，任一时间只有部分的socket是"活跃"的，但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用 callback函数，其他idle状态socket则不会，在这点上，epoll实现了一个"伪"AIO，因为这时候推动力在os内核。在一些 benchmark中，如果所有的socket基本上都是活跃的---比如一个高速LAN环境，epoll并不比select/poll有什么效率，相反，如果过多使用epoll_ctl,效率相比还有稍微的下降。但是一旦使用idle connections模拟WAN环境,epoll的效率就远在select/poll之上了。


先通过配置判断是否用单机还是raft
# 如果是单机
给数据文件加锁
加载merge数据目录
加载数据文件（加载数据文件使用mmap来加速，但是后面还要切换回来标准io，
因为不想要每个文件来维护mmap的内存，那样子的话内存就一直占用着了，还会一直顺序增长，并且无法控制脏页刷新，有可能一个事务的话只刷了一半的脏页，或者频繁发生缺页中断让系统性能变得复杂。）
加载hint索引文件

使用epoll来做tcp，先通过net.listen获取文件描述符，然后启动工作池，里面有很多协程通过chan来获取数据。


# dapeng的负载均衡

下图引用自[知乎：什么是负载均衡？](https://zhuanlan.zhihu.com/p/32841479)，很好地说明了负载均衡是一件什么事情：

[![v2-3661c2082103036ecb23a3f29be740be_b](https://spongecaptain.cool/images/img_rpc/loadbalance.gif)](https://spongecaptain.cool/images/img_rpc/loadbalance.gif)

取自 [WiKi](https://zh.wikipedia.org/wiki/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1)：

负载平衡（Load balancing）是一种电子计算机技术，用来在多个计算机（计算机集群）、网络连接、CPU、磁盘驱动器或其他资源中分配负载，以达到优化资源使用、最大化吞吐率、最小化响应时间、同时避免过载的目的。 使用带有负载平衡的多个服务器组件，取代单一的组件，可以通过冗余提高可靠性。负载平衡服务通常是由专用软件和硬件来完成。 主要作用是将大量作业合理地分摊到多个操作单元上进行执行，用于解决互联网架构中的高并发和高可用的问题。

**但是，我们该如何设计负载均衡接口呢？**

## 2. 设计一个负载均衡算法接口

在编程过程中，一个好的抽象能够极大地替我们简化问题。关于负载均衡算法，最好的方式就是遵循函数式编程范例，不持有任何状态，输出仅仅与输入有关：

- **输入**：一个 Server List ，即一个能够提供当前 RPC 请求对应服务的服务器列表；
- **输出**：一个服务器地址，该服务器将处理此次请求；

因此，我们可以将负载均衡算法抽象为如下的接口（in Java）：
```java
public interface LoadBalance {
    //String 对应于能够提供服务的服务器地址
    String getServerAddress(RpcRequest request, List<ServiceInfo> serverList);
}
```
常见的负载均衡算法有：

- 随机（包括简单随机以及权重随机）；
- 轮询（包括均等轮询以及权重轮询）；
- 最少调用数优先，相同活跃数则随机；
- 一致性 Hash，相同类型的请求总是会发送到同一个提供者上；

负载均衡算法的优缺点可以参考 [Dubbo2.7 用户文档-成熟度](https://dubbo.apache.org/zh/docs/v2.7/user/maturity/)：

|负载均衡算法|缺点|
|---|---|
|权重随机|在一个截面上碰撞的概率高，重试时，可能出现瞬间压力不均|
|轮询|存在慢的机器累积请求问题，极端情况可能产生雪崩|
|最少活跃调用数|不支持权重，在容量规划时，不能通过权重把压力导向一台机器压测容量|
|一致性 Hash|压力分摊不均|

下面我们依次实现上述负载均衡算法。

## 3. 具体负载均衡算法实现

每一个服务提供方在注册中心对于其服务的每一个服务都可以在 Java 堆中抽象为如下的类：
```java
public class ServiceInfo {
    private String address;
    //-1 代表没有权重设置
    private int weight;
    //-1 代表没有连接数的数据
    private int invokeNumber;
  //省略其构造器、setter/getter 方法
}
```
### [](https://spongecaptain.cool/post/rpc/addloadbalanceintorpc/#31-%E7%AE%80%E5%8D%95%E9%9A%8F%E6%9C%BA)3.1 简单随机

简单随机策略意味着每一个服务提供者有着相等的概率被选中，这种方式实现策略简单。但是，如果 Provider 性能有所差异，那么性能好的机器与性能差的机器处理同理的请求，并不是一件明智的决定。

代码更能有说明能力，简单随机负载均衡器代码如下：
```java
public class RandomLoadBalance implements LoadBalance {

    private Random random;

    public RandomLoadBalance() {
        random = new Random();
    }

    @Override
    public String getServerAddress(RpcRequest rpcRequest, List<ServiceInfo> serverList) {
        //首先，确定随机整数的最大值
        int max = serverList.size();
        //在 0 - size-1 返回内进行随机
        int index = random.nextInt(max);//返回 [0,max)返回内的一个整数，左闭右开
        //返回服务器地址
        return serverList.get(index).getAddress();
    }

}
```
### 3.2 权重随机

权重随机中，每一个服务提供者虽然具有权重的被随机选择。高数老师告诉我们，在大数定理下，权重将最终决定每一个 Provider 被选择的个数。例如，Consumer 有 1000 次调用，服务提供方分别的权重为 [1,2,3,4]，那么最终提供方被选择的次数大致为 100、200、300、400 次。

代码更能有说明能力，权重随机负载均衡器代码如下：
```java
public class WeightRandomLoadBalance implements LoadBalance {

    private Random random;

    public WeightRandomLoadBalance() {
        random = new Random();
    }

    @Override
    public String getServerAddress(RpcRequest rpcRequest, List<ServiceInfo> serverList) {
        int sum = 0;
        int[] weightArray = new int[serverList.size()];
        //遍历 serverList，计算权重和，并初始化 weightArray 数组
        for (int i = 0; i < serverList.size(); i++) {
            int weight = serverList.get(i).getWeight();
            weightArray[i] = weight;
            sum += weight;
        }
        //首先在 [0,sum) 范围内进行随机数的计算
        int rad = random.nextInt(sum);
        int cur_total = 0;

        for (int i = weightArray.length - 1; i >= 0; i--) {
            cur_total += weightArray[i];
            if (cur_total > rad) {
                return serverList.get(i).getAddress();
            }
        }
        return serverList.get(0).getAddress();
    }
}
```
### 3.3 轮询

轮询策略是所有负载均衡策略中唯一一个反函数式编程思想的负载均衡器，因为为了确保轮询，需要额外的保存状态，因此引入了额外的线程不安全性。

代码更能有说明能力，轮询式负载均衡器代码如下：
```java
public class RoundRobinLoadBalance implements LoadBalance {

    //为每一个 RPC 的服务名创建一个轮询的索引计数
    //key 服务名，value 轮询所用的索引
    final Map<String, AtomicInteger> roundRobinMap = new HashMap<>();
    //注意线程安全性！！！！
    @Override
    public String getServerAddress(RpcRequest rpcRequest, List<ServiceInfo> serverList) {

        //得到轮询索引
        AtomicInteger index = roundRobinMap.get(rpcRequest.getInterfaceName());

        //进行 double check 式的空值初始化
        if (index == null) {
            synchronized (roundRobinMap) {
                if (roundRobinMap.get(rpcRequest.getInterfaceName())==null) {
                    index = new AtomicInteger(-1);
                    roundRobinMap.put(rpcRequest.getInterfaceName(), index);
                }
            }
        }
        //进行取余操作
        int serverListIndex = Math.abs(index.incrementAndGet() % serverList.size());
        //返回地址
        return serverList.get(serverListIndex).getAddress();
        /**
         * 注意，虽然 AtomicInteger 会有溢出的危险，但是当 AtomicInteger 达到最大值以后，
         * 并不会继续增大，而是会转换为 -2147483648，因此，只要进行取取余后的值进行取绝对值，就符合本轮询的使用方式（取余无所谓从整数变为负数）
         */
    }
}
```
### 3.4 一致性 Hash

一致性 Hash 比简单 Hash 提供了在增减 Provider 时的稳定性，具体算法思想网上有非常多，这里不再赘述。

一致性 Hash 负载均衡算法大致上分为两个步骤：

1. 将多个服务提供方抽象为 Hash 环上的节点；
2. 将 Consumer 的请求匹配到合适的服务提供方列表，然后将请求映射为 Hash 数，然后打在 Hash 环上，确定服务提供方地址；

> Hash 环为何可以使用 SortedMap 来实现，是一致性 Hash 算法的难点之一。下图来自于：https://segmentfault.com/a/1190000021199728，很好地说明它们之间的联系。

[![hash-ring-hash-objects.jpg](https://spongecaptain.cool/images/img_rpc/hash.jpeg)](https://spongecaptain.cool/images/img_rpc/hash.jpeg)

一致性 Hash 算法负载均衡器的代码如下所示：
```java
```java
public class ConsistentHashLoadBalance implements LoadBalance {

    @Override
    public String getServerAddress(RpcRequest rpcRequest, List<ServiceInfo> serverList) {

        //从入口参数 List<ServiceInfo> serverList 处进行构造 Hash 环
        SortedMap<Integer, String> sortedMap = new TreeMap<>();
        /**
         * 利用 serverList 来构造 Hash 环:
         * 不过事实上并没有单独的 Hash 环作为数据结构，
         * Hash 环的存在重在我们如何来看待 SortedMap<Integer, String> 这一个数据结构
         * 把 Hash 环拉直，就是一个 key 为 hash，value 为服务器地址的有序的 HashMap(SortedMap)
         * 可以参考 URL:https://segmentfault.com/a/1190000021199728
         */
        for (int i = serverList.size() - 1; i >= 0; i--) {
            int hash = getHash(serverList.get(i).getAddress());
            sortedMap.put(hash,serverList.get(i).getAddress());
        }

        /**
         * 首先，我们利用 rpcRequest 请求的服务名进行 Hash 运算
         * 服务名是 接口名 还是 接口名+方法名 取决于 Provider 在服务注册中心的服务注册粒度，这里就仅仅以接口
         */
        //1.得到请求的 hash
        int hashOfRequest = getHash(rpcRequest.getInterfaceName());

        //2.得到大于 hashOfRequest 的所有 Map

        SortedMap<Integer,String> subMap = sortedMap.tailMap(hashOfRequest);

        if (subMap.isEmpty()) {
            //如果没有比该key的hash值大的，则从第一个node开始
            Integer i = sortedMap.firstKey();
            //返回对应的服务器
            return sortedMap.get(i);
        } else {
            //第一个Key就是顺时针过去离node最近的那个结点
            Integer i = subMap.firstKey();
            //返回对应的服务器
            return subMap.get(i);
        }
    }

    //一个基于 FNV1_32_HASH 的 Hash 算法，用于计算服务名的 hash 值
    private static int getHash(String str) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < str.length(); i++)
            hash = (hash ^ str.charAt(i)) * p;
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;

        // 如果算出来的值为负数则取其绝对值
        if (hash < 0)
            hash = Math.abs(hash);
        return hash;
    }
}
```

### 3.5 最少调用数负载均衡器

最少调用数是最值得考虑的负载均衡设计，因为其存在很大的性能问题。

最少调用数算法看起来很完美（似乎是最优解），但是也存在隐患：如果客户端在一个时间段的请求数激增，快到服务端都来不及将调用数异步更新到注册中心（或者类似的分布式组件上）。那么可能会导致原本最少调用数的服务器上的调用数激增，甚至超过原本最高调用数的服务器。

## 4. 将负载均衡器融入到现有 RPC 框架中

现有的 RPC 框架的地址为：https://github.com/Spongecaptain/RPCFramework，原本的框架设计中是没有负载均衡器的，新的组件融入旧框架是一个重大的问题。

下面按照不同组件来说明一下上述问题的解决方案。

### 4.1 修改 ZooKeeper 注册中心的使用方式

ZooKeeper 服务中心如下图所示：

[![/user-guide/images/zookeeper.jpg](https://spongecaptain.cool/images/img_rpc/zookeeper.jpg)](https://spongecaptain.cool/images/img_rpc/zookeeper.jpg)

我将上述注册中心模型进行简化（我们没有必要如此复杂），将 consumers 节点去除，便是我们的 ZooKeeper 使用策略。

在根节点 `/rpc` 下的每一个节点都是服务节点：

- 节点的 path：一个接口的完全限定名，例如图中的 cool.spongecaptain.Foo 接口；
- 服务节点的子节点 path：能够提供此接口的服务的服务器地址，例如 localhost:2222，其 value 对应于服务权重（当其 value 为 -1 时，意味着我们不对权重值进行设计）。

另一方面，伴随着注册中心的使用策略修改，框架中的服务注册（[registry](https://github.com/Spongecaptain/RPCFramework/blob/main/rpc-core/src/main/java/cool/spongecaptain/registry/ServiceRegistry.java)）/服务发现（[ServiceDiscovery](https://github.com/Spongecaptain/RPCFramework/blob/main/rpc-core/src/main/java/cool/spongecaptain/registry/ServiceDiscovery.java)）组件也需要进行修改。

### [](https://spongecaptain.cool/post/rpc/addloadbalanceintorpc/#42-%E9%87%8D%E6%96%B0%E8%AE%BE%E8%AE%A1-nettyrpcclient-%E7%B1%BB)4.2 重新设计 NettyRpcClient 类

[NettyRpcClient](https://github.com/Spongecaptain/RPCFramework/blob/main/rpc-core/src/main/java/cool/spongecaptain/client/NettyRpcClient.java) 类现如今发出一个 RPC 调用分为 3 步：

1. 利用服务发现组件，为当前请求匹配一个服务器地址列表；
2. 利用负载均衡组件，为当前请求确定一个服务器地址；
3. 发出请求；

负载均衡为新添加的一个步骤。
