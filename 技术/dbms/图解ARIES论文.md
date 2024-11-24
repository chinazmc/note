
# page oriented类存储引擎里可能同时存在多种树形结构

## 存储引擎的分类

目前接触到的存储引擎，以向磁盘读写方式来分类的话，大体可以分为两类：

-   LSM-Tree结构。
-   Page oriented类。

LSM-Tree是“Log-Structured Merge-Tree”的简称，这类存储引擎写入一条数据的流程大体如下：

-   向内存以及WAL日志中写入完成，即可认为写入成功。
-   内存中的数据写满之后，将落盘到所谓的sstable中。
-   sstable分为多层，随着写入进行，不同层次的sstable数据将进行合并。

![LSM](https://www.codedump.info/media/imgs/20220410-weekly-12/LSM.jpeg "LSM")

（图片引用自[LSM树详解 - 知乎](https://zhuanlan.zhihu.com/p/181498475))

从上面简单的写入LSM的流程可以看到：无论是写入内存还是磁盘，这类存储引擎在写入新数据时（不是合并sstable流程），磁盘操作的单位是一条记录。而一条记录的长度，是不定长的。

与LSM-Tree类的结构不同的是，Page oriented类的存储引擎，向磁盘发起读写操作的基本单位是页面（page），一个页面通常的大小是2的次方，最小一般是1024字节，比如sqlite的存储，其页面大小为4K（可以修改编译选项配置页面大小）。

以一个物理页面为读写磁盘的基本单位，这也是这一类存储引擎之所以被称为”Page oriented类存储引擎“的原因。本文重点是介绍Page oriented类存储引擎的结构。

## Page oriented存储引擎的结构

还是以之前介绍过的sqlite的架构图来开头：

![btree架构](https://www.codedump.info/media/imgs/20211217-sqlite-btree-0/btree-arch.png "btree架构")

这个架构由下往上依次是：

-   系统层：提供不同系统级API的封装，比如文件读写、加解锁操作等。
-   物理页面管理层：提供物理页面读写、缓存等功能。
-   树形结构的实现：根据具体的树形算法，组织物理页面之间的逻辑关系（比如父子页面、兄弟页面），以及单个物理页面之内的数据的组织。

这里的重点是页面管理层和树形结构的实现这两部分：

-   物理页面管理相当于是磁盘文件的”原材料供应商“，负责对它的客户也就是各种不同结构的实现提供物理页面这一”原材料“的读写、缓存管理，而它对这些材料被客户拿去做成了什么，一无所知。
-   树形结构的实现，从页面管理器拿到了”物理页面“这个原材料之后，可以按照自己的算法和数据结构任意塑造成任何合理的结构。

![数据库文件的物理页面组织和逻辑页面结构](https://www.codedump.info/media/imgs/20220201-sqlite-btree-5-btree/database-file.png "数据库文件的物理页面组织和逻辑页面结构")

可以看到，Page oriented存储引擎，在满足“向磁盘读写的基本单位是物理页面”这个大前提下，这类存储引擎的可能同时存在多种结构：可能只有B-Tree，也可能只有B+Tree。还有另一种情况是：这类存储引擎内部同时存在多种结构。

以sqlite为例，内部其实就存在两种结构：

-   存储索引的index tree：结构为B-Tree，键为表索引，值为这一行数据的`rowid`，其中`rowid`为隐藏列，创建数据表时自动生成，这一列是自增整数。
-   存储数据的table tree：结构为B+Tree，键为`rowid`，值为一行数据。

这两类存储引擎，由于同属于“Page oriented类存储引擎”，因此可以共用同一个物理页面管理器。

![数据库文件的rowid全量数据表和索引表](https://www.codedump.info/media/imgs/20220201-sqlite-btree-5-btree/btree-rowid.png "数据库文件的rowid全量数据表和索引表")

下面，以sqlite中的一个表为例来解释上面这个流程。

首先，创建一个表以及索引：

```sql
// 创建数据库COMPANY
CREATE TABLE COMPANY(
   ID             INT      NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
);
// 创建索引
CREATE INDEX id_index ON COMPANY (id);
```

上面这个建表以及创建索引之后，对应的在这个数据文件中就有了两个树形结构：

-   存储`COMPANY`表数据的table-tree。
-   存储索引`id`到`rowid`关系的index-tree。

如果向这个表插入数据，比如：

```sql
INSERT INTO COMPANY (ID,NAME,AGE) VALUES (1, 'Paul', 32 );
```

那么，这个插入操作背后实际对应了向这两棵树的插入操作：

-   首先，将这一行数据插入到table-tree中，同时得到`rowid`以及插入时候的`id`。
-   再将第一步得到的`rowid`以及`id`插入到index-tree中。

如果使用`id_index`索引来查询`COMPANY`表，比如：

```sql
select * from COMPANY where id = 1;
```

这个查询操作也实际上经过了上面这两个表：

-   首先，通过存储`id_index`到`rowid`关系的index-tree，找到`id=1`对应的`rowid`。
-   然后，再根据第一步得到的`rowid`到table-tree中查询到这一行数据。

## 总结

-   存储引擎，按照对磁盘读写方式的不同，大体可以分为以下两类：
    -   LSM-Tree：写磁盘的基本单位是一条记录，而一条记录大小是不定长的。
    -   Page oriented：读写磁盘的基本单位是页面，页面大小为2的次方。
-   “Page oriented”类存储引擎的核心模块是页面管理器和树形结构的实现，前者提供物理页面这一“原材料”的读写操作，对页面内部的结构一无所知；后者组织管理物理页面间的逻辑关系，以及物理页面内部的数据。
-   在满足“读写磁盘的基本单位是页面”的大前提下，“Page oriented”类存储引擎可以使用各种树形结构，还可能在同一个存储引擎中同时存在多种树形结构。
-   sqlite的实现，内部存在两种不同的树形结构：使用B-Tree来管理索引数据，以B+Tree来管理表数据。这是因为：
    -   索引数据的值只有`rowid`这样的整型数据，所以单个物理页面内能存储更多的索引数据，适合使用B-Tree这样“高而瘦”的结构来管理这类单条数据很小的数据。
    -   而B+Tree的树形结构是“矮而胖”的结构，更适合存储管理多种不定长的数据。

引言：ARIES(Algorithm for Recovery and Isolation Exploiting Semantics的简称）是论文[《ARIES: A Transaction Recovery Method Supporting Fine-Franularity Locking and Partial Rollbacks Using Write-Ahead Logging》](https://cs.stanford.edu/people/chrismre/cs345/rl/aries.pdf)中提到的一种存储引擎中数据恢复的算法。这篇论文可以说是存储引擎数据恢复领域必读的一篇论文，这两期的周刊就是对这篇论文的图解，这是其中的上篇。
# 图解ARIES论文（上）

在展开解释ARIES算法原理之前，需要对[Page oriented类存储引擎](https://www.codedump.info/post/20220410-weekly-12/)的日志系统有一定的了解，才能继续解释基于这个日志系统之上做的恢复算法。

## 问题

在一个存储系统中，出错是非常常见的情况的，这就涉及到出错了之后系统恢复时还需要能继续工作，即数据不能发生破坏导致整个系统跑不起来。

于是，当系统出错需要重启恢复时，就涉及到以下两个动作：

-   撤销（Undo）：未完成或者由于各种原因发生回滚（rollback）、中断（abort）的事务，其修改需要被撤销，即回滚为事务之前的旧值。
-   重做（Redo）：已经提交的事务，其修改操作的效果需要体现为新值。

来看下图中提出的问题：

![bufferpool](https://www.codedump.info/media/imgs/20220514-weekly-15/bufferpool.png "bufferpool")

在上图中：

-   存在事务T1和T2在同时执行：
    -   事务T1：修改A值为3，但是在事务还未提交前，事务T2开始执行。
    -   事务T2：修改B值为8，并且成功提交。
    -   事务T1终止：在事务T2成功提交之后，事务T1终止。

这个事务调度的执行顺序引发了以下几个问题：

-   回滚未提交的事务T1需要做什么？
-   对于未提交的事务T1，是否允许其修改操作在持久化存储上生效（即将A修改为3）？
-   在磁盘的数据库文件中，已成功提交的事务T2，其修改操作是否应该立即落盘（即从buffer pool中同步修改的内容到硬盘）。

第一个问题当前暂且放到一边，来看后面两个问题。

“是否允许未提交事务的修改在持久化存储上生效”（Whether the DBMS allows an uncommitted txn to overwrite the most recent committed value of an object in non-volatile storage），被称为`Steal policy`：

-   steal：允许未提交事务的修改持久化存储上生效。
-   no steal：反之。

一个事务在提交之前是否需要将所有修改同步到持久化存储上（Whether the DBMS requires that all updates made by a txn are reflected on non-volatile storage before the txn is allowed to commit.），也有两种策略：

-   force：必须将事务的所有修改都同步到持久化存储上，事务才被允许提交。
-   no-force：反之。

围绕着这两个不同的维度，有了不同的做法。

我们来按照上面的定义，使用`no-steal+force`的策略，来重新看看前面的问题。

![no-steal+force](https://www.codedump.info/media/imgs/20220514-weekly-15/no-steal+force.png "no-steal+force")

在上图中：

-   事务T1：由于采用no-steal策略，所以对于未能提交的事务T1而言，该事务的修改顶多到内存的buffer pool中，不会在磁盘上生效，因此回滚事务的操作只需要将内存的buffer pool中的数据撤销即可。
-   事务T2：由于采用force策略，因此在事务提交之前，必须将所有修改都同步到磁盘的数据库文件上。

可以看到，使用`no-steal+force`的策略是相对简单的，因为：

-   `undo`操作只需要回滚内存中的数据即可，因为还未同步到磁盘的数据库文件上。
-   对于已提交事务而言，无需`redo`操作，这是因为事务提交之前必须强制同步所有修改到数据库文件上。

但是，这个策略组合的问题在于：无法支撑一个修改了大量数据的事务。比如假设内存中的buffer pool仅能容纳1K大小的修改缓冲区，而某一个事务一共修改了2K的数据，这时候超过buffer pool大小的数据，是否应该即时落盘？按照这个策略是解决不了这个问题的。

接下来看看两种最常见的做法：

-   影子页面（shadow page）。
-   WAL（Write Ahead Log）。

## 影子页面（shadow page）

影子页面的原理是这样的，维护两个独立的数据库文件的拷贝：

-   主拷贝：只保存已提交事务的修改。
-   影子拷贝：拷贝待修改页面的内容，保存未提交事务的修改。
-   有了两份不同的数据，在进行中的事务只需要修改影子拷贝即可，当未提交事务需要提交时，切换影子拷贝变成主拷贝。

以下图来解释影子页面的工作原理：

![shadow-page](https://www.codedump.info/media/imgs/20220514-weekly-15/shadow-page.png "shadow-page")

上图中：

-   分别用三种颜色标记了不同类型的页面：主页面、影子页面、空闲页面，其中空闲页面是被替换出去的主页面，由于已经有新的主页面产生，所以之前的主页面就变成了空闲页面可以重新在下一次生成影子页面的时候使用上。
-   在修改之前，已经存在3个主页面，其中的页面1为当前的根页面。
-   假设一个事务，需要同时修改这三个主页面，系统发现当前没有可用的空闲页面，于是扩展数据库文件生成三个新的物理页面，首先拷贝原页面内容，然后再这个基础上进行事务的修改。
-   当事务修改完成之后，只需要修改根页面为最新的页面1即可，而这次修改中涉及到几个之前的主页面就变成了空闲页面，下一次修改时可以回收利用。

根据前面对两种策略的定义，影子页面属于采用`no-steal+force`策略的方案：

-   no-steal：未提交的时候，不能直接修改变成主页面。
-   force：事务提交之前，必须把修改后的影子页面切换为新的主页面。

我们看一看，影子页面的重做和撤销是怎么做的：

-   撤销：只需要将写事务时多分配出来的物理页面再次变成空闲页面即可，下一次可以回收利用。由于没有切换根页面编号，并不需要多做什么其它的工作了。
-   重做：不需要重做操作，因为只要完成了影子页面切换到主页面的操作，就相当于这个事务的修改可见了。

来看看影子页面的问题：

-   既然是“影子”，那么就要求这些页面在修改之前，必须先拷贝一份原始页面的数据，这个代价是很重的。
-   对于类似B-Tree这样的数据结构，这个修改会一直从下往上进行，路径中经过的节点都需要一个影子页面来存储修改。
-   被替换的主页面会产生空闲页面，对应的就需要垃圾回收策略把这些空闲页面给回收利用上。
-   页面内容会碎片化，因为写入的影子页面可能是新增的物理页面，还可能是被回收利用的空闲页面，如果逻辑上相邻的页面在物理上分散，对于读写缓存是不友好的。
-   最后，当数据落盘到影子页面时，本质上还是针对文件的一个随机写，随机写的代价是很大的：需要首先寻址到指定位置，然后再进行修改。
-   同一时间，只允许有一个写事务在进行，因为如果存在多个写事务，而不同写事务又修改了同一个页面的话，是无法处理的。

### 采用影子页面的例子

之前的博客中分析过boltdb的实现，以及sqlite的journal实现，这两者都是影子页面的例子：

-   [boltdb 1.3.0实现分析（一）](https://www.codedump.info/post/20200625-boltdb-1/)：在boltdb中，修改页面之前，首先使用mmap机制复制出来待修改的页面，修改完毕之后统一切换主页面和影子页面，同时将修改之前的主页面设置为空闲页面可供下一次回收利用。
-   [sqlite3.36版本 btree实现（三）- journal文件备份机制](https://www.codedump.info/post/20211222-sqlite-btree-3-journal/)：sqlite的journal文件，负责备份修改的页面在修改前的数据，事务提交之后这些备份的数据就无用了，如果中间事务中断可以直接用journal备份文件的内容来恢复数据库。

感兴趣的，可以由这两篇文章了解更详细的影子页面的实现。不过，影子页面机制的使用已经不多，更多的是下面介绍的WAL。

## WAL（Write Ahead Log）

除了shadow paging策略以外，另一种保存修改的策略就是使用WAL（Write Ahead Log），对比前者而言，WAL最大的好处是：由于将修改添加到日志中就可以认为保存修改完成，而在文件尾追加（append）操作对比随机写是相对快很多的。

对于一个写事务而言，如果采用WAL机制，WAL日志的格式有以下三类（后面还会扩展）：

-   事务开始时，写入一条`BEGIN`表示写事务开始。
-   记录事务的修改操作：对于写事务中的每一条修改操作，都要记下以下四个数据：
    -   事务ID，这样在同一时间进行多个写事务时，可以知道对应的是哪个事务。
    -   修改的键值。
    -   修改之前的值，用于撤掉事务时使用。
    -   修改之后的值，用于重做事务时使用。
-   事务提交时，写入一条`COMMIT`表示事务结束。

如下图中演示了之前的两个写事务使用WAL策略：

![WAL](https://www.codedump.info/media/imgs/20220514-weekly-15/wal.png "WAL")

在上图中：

-   同时并发了两个写事务，每个写事务的三类不同的日志，都需要写到内存的buffer pool中，日志的格式已经在上面给出来。
-   磁盘上存储的文件，除了之前的数据库文件之外，新增了一个叫WAL的日志文件，这个文件不会随机写，只会按照buffer pool中记录的日志顺序添加到文件结尾处。
-   一个写事务的操作日志，首先会记录到buffer pool中，而不必立即同步到wal日志中，buffer pool中的日志同步到wal的操作，可以批量一次性写多多条记录，又进一步提升了写操作的效率。
-   采用WAL策略之后，一个写操作完成不要求一定要将修改同步到数据库文件中，只需要所有该事务的操作都写入WAL文件即可。

这里需要说明的是，前面提到`steal`以及`force`这两个维度的策略时，都与是否需要同步数据到持久化存储有关。但是，在WAL中，由于新增了WAL文件，而这个文件本身也是保存在持久化存储中的，所以`持久化存储`文件这个概念的范畴，从仅有数据库文件，扩展到了数据库文件和WAL文件。

于是，从上面对WAL的分析可以看到，WAL是`steal+no-force`的策略：

-   steal：允许未提交事务的修改操作同步到磁盘的WAL文件中。
-   no-force：事务提交之前，必须所有的修改操作都写入磁盘的WAL中才能提交，但并不要求一定要落入数据库文件。

由于`持久化存储`包括了WAL文件和数据库文件，即可以认为任意时刻的数据为：`数据库文件` + `WAL中的已提交事务数据`，于是查询一个数据时就需要在这两类文件中依次查询：

-   首先到WAL中查询已提交的事务数据。
-   如果第一步没有查到，说明WAL中并没有对这个数据的修改，于是到数据库文件中查询数据。

从上面对影子页面和WAL的分析可以看出来，以`steal`和`force`两个策略的维度组合（一共四个组合，见下图）来看：

-   运行时效率：`steal` + `no-force`是最快的，而`no-steal` + `force`则是最慢的。
-   恢复时效率：对比运行时效率，恢复时的效率就反过来了，`steal` + `no-force`是最慢的，而`no-steal` + `force`则是最快的，原因在于前者需要执行`undo`、`redo`操作，而后者则不需要。

![policies](https://www.codedump.info/media/imgs/20220514-weekly-15/policies.png "policies")

## Checkpoint（检查点）

前面提到过，采用了WAL策略的数据库，任意时刻的数据为：`数据库文件` + `WAL中的已提交事务数据`。

但是WAL文件不可能一直增长下去，否则一来文件太大，导致影响读性能，二来数据恢复的时候时间过长。所以，每隔一段时间，都需要将WAL文件中的已提交事务数据落盘到数据库文件中，这个流程被称为`checkpoint`。

为了做`checkpoint`操作，需要新增一类`checkpoint`日志，结合前面提到的几类WAL日志，如下：

-   事务开始时，写入一条`BEGIN`表示写事务开始。
-   记录事务的修改操作：对于写事务中的每一条修改操作，都要记下以下四个数据：
    -   事务ID，这样在同一时间进行多个写事务时，可以知道对应的是哪个事务。
    -   修改的键值。
    -   修改之前的值，用于撤掉事务时使用。
    -   修改之后的值，用于重做事务时使用。
-   事务提交时，写入一条`COMMIT`表示事务结束。
-   做`checkpoint`时，写入一条`CHECKPOINT`表示在这个点做了`checkpoint`操作。

来看看有了`checkpoint`操作之后，如何处理恢复问题，如下图所示：

![checkpoint](https://www.codedump.info/media/imgs/20220514-weekly-15/checkpoint.png "checkpoint")

在上图中：

-   事务T1提交之后，又开始了事务T2、T3，在T2、T3未提交之前，开始了一个`checkpoint`操作。
-   `checkpoint`完成之后，T2完成提交。但是在T3修改之后，就发生了宕机。

此时，如果进行数据恢复的话：

-   数据库文件中已经有事务T1的修改了，因为T1是在`checkpoint`之前提交的事务，所以这个事务的修改会在`checkpoint`的时候同步到数据库文件中。但是T2、T3这两个在`checkpoint`时还未完成的事务，并不会在这一次`checkpoint`时同步到数据库文件。
-   由于WAL中存在事务T2的`COMMIT`日志，表示事务T2已经完成提交，所以在系统恢复时：
    -   找到崩溃之前最后一次`checkpoint`的位置，针对这个`checkpoint`之后的日志，进行如下的操作。
    -   对于存在`COMMIT`日志的事务，比如事务T2，只需重放其WAL日志即可。
    -   对于不存在`COMMIT`日志的事务，比如事务T3，不能重放这个事务，因为在崩溃之前没有落盘所有的修改在WAL中。

这里的`checkpoint`流程中，存在一个明显的问题：要求`checkpoint`在进行的时候，不得有任何的写事务在进行，换言之`checkpoint`操作会影响写事务的进行。所以，如果`checkpoint`的频率太高，影响写事务；频率太低，又会让单次`checkpoint`时间很长。

下一期周刊会在此基础上介绍ARIES论文的思想，论文中给出了`checkpoint`的优化做法。

## 使用WAL的例子

在sqlite的3.7.0版本之后，就采用了WAL机制，之前也写过文章分析，见[sqlite3.36版本 btree实现（四）- WAL的实现](https://www.codedump.info/post/20220106-sqlite-btree-4-wal/)

## 总结

-   按照落盘的策略，分为了`steal`和`force`这两个维度。
-   shadow page采用的是`no-steal+force`策略，而WAL采用的是`steal+no-force`策略。

---

# 图解ARIES论文（下）

## 前情回顾

在[周刊（第15期）：图解ARIES论文（上）](https://www.codedump.info/post/20220514-weekly-15/)中，讨论了存储引擎面临的问题，如果存储引擎宕机重启，将要进行以下两个操作：

-   撤销（Undo）：未完成或者由于各种原因发生回滚（rollback）、中断（abort）的事务，其修改需要被撤销，即回滚为事务之前的旧值。
-   重做（Redo）：已经提交的事务，其修改操作的效果需要体现为新值。

为了这两个操作，存储引擎就需要回答这两个问题：

-   “是否允许未提交事务的修改在持久化存储上生效”（Whether the DBMS allows an uncommitted txn to overwrite the most recent committed value of an object in non-volatile storage），被称为`Steal policy`。  
    “是否允许未提交事务的修改在保持持久化存储上生成有效”（DBMS 是否允许未提交的 txn 覆盖非易失性存储中对象的最近提交值），被称为 `Steal policy` 。
-   一个事务在提交之前是否需要将所有修改同步到持久化存储上（Whether the DBMS requires that all updates made by a txn are reflected on non-volatile storage before the txn is allowed to commit.），称为`force policy`。

两个问题合并起来一共有四种组合：

![policies](https://www.codedump.info/media/imgs/20220514-weekly-15/policies.png "policies")

最常见的组合策略实现有以下两种：

-   shadow paging（影子页面）：采用`no-steal+force`策略。
    
-   WAL：采用`steal+no-force`策略。
    

两者的优缺点：

-   `steal+no-force`策略：运行时效率高，即读写数据的效率高，但是数据恢复的效率低。以WAL为例，运行时写入WAL日志即可，而写WAL是append操作而不是随机写操作，速度会更快；但是恢复时需要重放（replay）WAL日志。
-   `no-steal+force`策略：运行时效率低，但是数据恢复的效率高。

目前更常见的做法是WAL。在WAL这个策略中，任意时刻都满足以下条件：

```undefined
全量数据 = 数据库文件数据 + WAL中已提交但还未转入数据库文件的数据
```

即可以认为，WAL中的`已提交但还未转入数据库文件的数据`是增量的数据。

另外，为了避免WAL文件无限增大，还引入了`checkpoint`流程，但是就目前来看在`checkpoint`时，需要终止写事务，这导致了写操作的卡顿。

前情回顾完毕，本篇正式开始介绍ARIES论文的思想，ARIES在此基础上提供了更高效的事务回滚、`checkpoint`算法的实现。

为了能够回滚事务，以及在`checkpoint`操作时不阻塞写事务，ARIES算法引入了一种重要的变量LSN（Log Sequence Number），下面几乎所有的相关算法、数据结构都跟这个变量有关系，从写事务开始说起。

## 写事务流程

### 扩展WAL格式

回忆一下上一节中wal日志的格式，有以下三类：

-   事务开始时，写入一条`BEGIN`表示写事务开始。
-   记录事务的修改操作：对于写事务中的每一条修改操作，都要记下以下四个数据：
    -   事务ID，这样在同一时间进行多个写事务时，可以知道对应的是哪个事务。
    -   修改的键值。
    -   修改之前的值，用于撤销事务时使用。
    -   修改之后的值，用于重做事务时使用。
-   事务提交时，写入一条`COMMIT`表示事务结束。
-   做`checkpoint`时，写入一条`CHECKPOINT`表示在这个点做了`checkpoint`操作。

ARIES算法要求扩展wal日志的格式：

-   每条日志需要带上一个全局唯一且单调递增的LSN序列号（Log Sequence Number），用于区分每一条日志。
-   同时，除了上面三类日志以外，还需要在事务提交成功之后，再写一条`TXN-END`的日志表示事务的结束。

总结起来，到目前为止，ARIES算法中日志有如下几类，其格式如下：

-   事务开始：`LSN:<事务编号,BEGIN>`。
-   事务的修改操作：`LSN:<事务编号,修改的键值,修改之前的值,修改之后的值>`。
-   事务提交操作：`LSN:<事务编号,COMMIT>`。
-   事务提交成功：`LSN:<事务编号,TXN-END>`。
-   `checkpoint`操作：`LSN:checkpoint`。

有了LSN之后，各类数据就需要记录相关的LSN以便记录当前的进度：

### 几类记录LSN的数据

![[Pasted image 20230423140340.png]]

以下图来展开解释这几个值的使用。

### 例子

![aries-1](https://www.codedump.info/media/imgs/20220521-weekly-16/aries-1.png "aries-1")

在上图中：

-   在LSN=07的位置，做了一次`checkpoint`，这意味着：
    -   在07之前的所有wal的内容，都同步到了数据库文件中，于是数据库文件中的`A=3`，这是07日志之前最后一次成功提交的事务对A的修改值。
    -   因为LSN=07做了一次`checkpoint`，所以`MasterRecord=07`，如果这时候发生宕机，那么重启恢复的时候，数据库将不会扫描在07这个LSN之前的wal日志，因为根据`MasterRecord=07`意味着这之前的日志都已经落盘从wal转存到数据库文件中了。
-   内存中的`flushLSN=11`，这是因为最后一条落盘的WAL日志序列号为11。
-   这个例子中仅有一个页面，为了讨论问题的方便，假设这个例子中的所有修改都在同一个页面中：
    -   右下角表示磁盘上在数据库文件中的页面，它始终反映的是最后一次`checkpoint`的修改，因此有： `pageLSN=06`和`A=3`。
    -   左下角表示该页面的内存中的修改，也被称为”脏页面“，内存中的脏页面一定反映的是当前最新的对该页面的修改，因此`pageLSN=15`和`A=12`、`B=10`。

以上几点都比较明确了，来看一个疑难问题：假如内存中有一个页面，如何判断这个页面是否能被释放？

这取决于这个页面的`pageLSN`与`flushLSN`之间的关系，因为`pageLSN`表示最后一次更新该页面时的LSN，因此唯有当将所有满足`pageLSN<=flushLSN`的日志都从内存落到磁盘时，这个页面才能被释放。

在上图中，对脏页面而言，`pageLSN=15` > `flushLSN=11`，说明脏页面中仍然有日志还未落盘，也就是LSN=12到15这几条日志还在内存里，所以这时候不能释放这个脏页面。

来看何时才能安全释放这个脏页，如下图所示：

![aries-2](https://www.codedump.info/media/imgs/20220521-weekly-16/aries-2.png "aries-2")

上图中，将LSN=12到15这几条日志从内存中落盘到磁盘上，这时候`flushLSN`修改成15，而内存中脏页面的`pageLSN=15`，这表示内存中脏页的修改都以日志形式落盘，于是这个脏页可以被安全释放了。

上面的流程里，还需要特别注意几点：

-   内存中的脏页不能直接落盘为磁盘上数据库文件中的页面，一定是经过这样的流程：
    -   对页面的修改日志先写入到内存中的WAL日志中。
    -   内存中的WAL日志落到磁盘上的WAL日志。
    -   对磁盘上的WAL日志进行`checkpoint`，在这个流程里依次回放WAL的操作，将修改落盘到数据库文件的页面里。

## 中断事务流程

事务中断时，就需要回滚这个事务前面的修改，为了记录一个事务都依次进行了哪些修改，需要在每一条记录上加入一个`prevLSN`字段表示在同一个事务中上一个日志的LSN，之所以加上`prevLSN`字段，是因为同一个事务的写日志，物理上不见得就是连续的，`prevLSN`在这里扮演了逻辑指针的作用。

有了`prevLSN`字段之后，继续扩展前面的日志格式：

-   事务开始：`LSN|prevLSN:<事务编号,BEGIN>`。
-   事务的修改操作：`LSN|prevLSN:<事务编号,修改的键值,修改之前的值,修改之后的值>`。
-   事务提交操作：`LSN|prevLSN:<事务编号,COMMIT>`。
-   事务提交成功：`LSN|prevLSN:<事务编号,TXN-END>`。

可以看到，有了`prevLSN`这个字段以后，可以将同一个事务的修改按照逆序串联起来，于是就可以根据这个逆序使用保存的`undo`值来回滚事务之前的修改。

只有这样还不够，因为还有可能在中断事务回滚的过程中也发生了宕机，这时候需要记录下来回滚进行到哪里了。记录回滚进度的日志被称为`CLR（Compensation Log Record，补偿日志记录）`日志，`CLR`日志和事务的修改操作日志格式大体相同，但是还需要新增一个`UndoNext`字段，表示回滚操作中下一个需要回滚的日志，显然有如下的关系：

```objectivec
假如CLR(A)是针对WAL(B)的回滚操作，那么CLR(A).UndoNext = WAL(B).prevLSN
```

于是在前面扩展之后的日志格式继续进行扩展新增`CLR`类型的日志：

-   事务开始：`LSN|prevLSN:<事务编号,BEGIN>`。
-   事务的修改操作：`LSN|prevLSN:<事务编号,修改的键值,修改之前的值,修改之后的值>`。
-   CLR日志：`LSN|prevLSN:<事务编号,修改的键值,撤销之前的值,撤销之后的值,下一个需要撤掉操作的LSN>`。
-   事务提交操作：`LSN|prevLSN:<事务编号,COMMIT>`。
-   事务提交成功：`LSN|prevLSN:<事务编号,TXN-END>`。

以一个例子来说明情况：

![[Pasted image 20230423140356.png]]

在上面的表格中：

-   事务T1：
    -   `LSN=001`：开始事务。
    -   `LSN=002`：修改键值A=40，修改之前的值为30。
    -   `LSN=011`：中断事务T1。
    -   `LSN=26`：由于前面开始中断事务T1的操作，于是从中断的那条日志开始进行回滚操作。中断日志的LSN为011，其`prevLSN=002`，于是新增一条CLR日志：
        -   该CLR日志的`prevLSN`为11，即中断开始的日志LSN。
        -   将回滚操作前后（即LSN=002的这条日志）的值记录下来。
        -   将回滚操作的`prevLSN`记录为这条CLR日志的`UndoNext`。
        -   继续这个流程，直到回滚完毕这个事务的所有操作（即一直回滚到`prevLSN=nil`的日志）。
    -   最后，在回滚操作结束之后，同样也需要一条`Txn-End`日志来标记事务回滚结束。

有了CLR日志之后，因为知道了事务中哪些操作已经被回滚过了，这样就不会在程序宕机重启时重复执行了，另外CLR日志本身并不需要回滚操作。

## fuzzy checkpoint流程

### 问题

上一篇讲解`checkpoint`流程时，有一个重要的问题：即一旦开始`checkpoint`，则当前所有事务，要么没有开始，要么全都已经结束，即此时不允许存在还未结束的事务。

这带来几个问题：

-   `checkpoint`运行时，不允许同时并发有其它的写事务。
-   由于上面这一点，一旦`checkpoint`要花费很长的时间，会导致系统在这段时间内都无法进行写操作。

于是，首先想到的策略，是将事务运行时的状态加入到`checkpoint`流程中，这样就允许同时并发有写事务操作了。

### 带状态的checkpoint

为了在`checkpoint`过程中保存事务运行时的状态，引入了以下两个数据结构。

-   Active Transaction Table（简称ATT，活跃事务表）：用于保存同时在运行的事务信息，表里存储的每个事务信息包括如下字段：
    -   txnId：事务ID。
    -   status：事务当前状态，包括Running（R，在运行）、Commit（C，在提交）、Candidate for Undo（U，准备撤销事务）。
    -   lastLSN：该事务最后的日志LSN。
-   Dirty Page Table（简称DPT，脏页表），这个表中存储的每个脏页信息，有`recLSN`：第一条修改这个页变成脏页的日志LSN。

**注意：这两个表的内容，要做为`checkpoint`记录的一部分，写入WAL日志中。 **

有了这些状态信息之后，一个重大的改进是：

-   之前一次`checkpoint`要求把这之前的所有WAL落盘到数据库文件中，所以不能在`checkpoint`时同时存在有写事务。
-   有了这些状态信息之后，不再要求将`checkpoint`之前的所有WAL落盘，对于还在进行的事务，只要将其信息保存到`ATT`以及`DPT`中即可，这些在进行事务的修改这一次`checkpoint`可以不进行落盘。

仍然以一个例子来说明这个流程。

![checkpoint](https://www.codedump.info/media/imgs/20220521-weekly-16/checkpoint.png "checkpoint")

如上图中：

-   事务T1：修改了页面P1，之后提交。
-   事务T2：修改了页面P2，还未提交之前就开始了`checkpoint`操作。
-   第一次`checkpoint`操作：在开始`checkpoint`时，系统中事务T2还在进行，P2为脏页面，于是`ATT={T2}`，以及`DPT={P2}`，于是这一次`checkpoint`操作，仅把已提交的事务T1的修改同步到了数据库文件上，即`A=120`。
-   继续到第二次`checkpoint`时，这时候事务T2已经完成，而事务T3还在进行，于是这一次`checkpoint`时，`ATT={T3}`，以及`DPT={P3}`，这样会把第一次`checkpoint`到第二次`checkpoint`之间的操作落盘，如何落盘呢？这时候会遍历这中间的WAL日志，来修改`ATT`和`DPT`这两个表，流程如下：
    -   第二次`checkpoint`开始的时候，`ATT={T2}`，以及`DPT={P2}`，这是上一次`checkpoint`记录下来的值。
    -   遍历两次`checkpoint`中间的所有WAL：
        -   完成的事务：从`ATT`表删除，同时从`DPT`中删除被该事务修改的页面。这个例子中，事务T2就是这样一个位于两次`checkpoint`之间且完成的事务，于是`ATT={T2}`修改为`ATT={}`，`DPT={P2}`修改为`DPT={}`
        -   开始的事务：将事务ID加入`ATT`表，当前被修改的页面编号加入`DPT`表中。在这个例子中，事务T3就是这样一个在第二次`checkpoint`时还在进行的事务，于是修改`ATT={}`为`ATT={T3}`，以及`DPT={}`为`DPT={P3}`。

可以看到，引入了`ATT`和`DPT`表之后，`checkpoint`操作带了状态， 这样就不要求`checkpoint`要等到所有事务都完成（比如这里的T2和T3）才能开始，因为记录了在进行的事务的状态。

但是这个改进还并未解决`checkpoint`时停止所有写事务的问题。原因在于，目前的`checkpoint`机制，是一个`STW（Stop The World）`的设计，一旦触发无法中断，下面的`fuzzy checkpoint`算法就针对这一点做了改进。

### fuzzy checkpoint

`fuzzy checkpoint`算法主要有以下的重点：

-   对比之前的`checkpoint`流程，`fuzzy checkpoint`算法有两条相关的日志：
    -   `checkpoint-begin`：标记着`checkpoint`操作的开始，在这次`checkpoint`操作成功之后，这条日志的`LSN`将保存到`MasterRecord`记录中。
    -   `checkpoint-end`：标记着`checkpoint`操作的结束。
-   `ATT`和`DPT`表：任何在`checkpoint-begin`之前开始但还未完成的事务ID以及脏页面会加入到这两个表中，但是任何在`checkpoint-begin`之后的不做记录。另外，对这两个表的内容是记录在`checkpoint-end`日志中的。

还是以一个例子来说明这个算法的流程：

![fuzzy-checkpoint](https://www.codedump.info/media/imgs/20220521-weekly-16/fuzzy-checkpoint.png "fuzzy-checkpoint")

-   事务T1：修改了页面P1，之后提交。
-   事务T2：修改了页面P2，还未提交之前就开始了`checkpoint`操作。
-   第一次`checkpoint`操作：在开始`checkpoint`时，系统中事务T2还在进行，P2为脏页面，于是`ATT={T2}`，以及`DPT={P2}`，于是这一次`checkpoint-end`日志，仅把已提交的事务T1的修改同步到了数据库文件上，即`A=120`。注意，事务T3在`checkpoint-begin`操作之后，所以并不会加入到这两个表中。
-   结束了第一次`checkpoint`操作之后，`MasterRecord`修改为`checkpoint-begin`操作的LSN。
-   继续到第二次`checkpoint`时，这时候事务T2已经完成，而事务T3还在进行，于是这一次`checkpoint`时，`ATT={T3}`，以及`DPT={P3}`，这样会把第一次`checkpoint`到第二次`checkpoint`之间的操作落盘，如何落盘呢？这时候会遍历这中间的WAL日志，来修改`ATT`和`DPT`这两个表，流程与上面大体一样，就不做解释了。

`fuzzy-checkpoint`操作之所以名为`fuzzy（模糊）`，意思就是并不知道`checkpoint`操作何时结束，只知道何时开始，只需要在`checkpoint-end`结束的时候，把`checkpoint-begin`时计算的ATT，DPT内容一起写入即可。

可以看到，**这个算法既不要求开始时所有写事务都结束，也不要求进行时停止所有写事务，极大提升了并发性能。**

## 数据恢复流程

以上，已经完整的介绍了ARIES算法的核心，最后来谈谈数据恢复的流程。

数据恢复分为以下三个步骤：

-   数据分析阶段
-   重做。
-   撤销。

下面来逐个讲解这三个阶段的工作。

### 数据分析阶段

找到最近的一次`checkpoint`日志，分析出在这之后成功和失败的事务，从这条日志之后开始构建ATT，DPT表内容：

-   `MasterRecord`中保存了最近一次成功的`checkpoint-begin`的`LSN`，从这条日志之后开始扫描。
-   扫描过程中，针对不同类型的WAL处理如下：
    -   修改：将对应事务的ID加入`ATT`表中，设置状态为`Undo`，意为需要撤销这个事务的操作。同时，这个修改操作修改到的页面编号，如果不在`DPT`表，就需要加入到`DPT`表中，同时设置这个页面的`recLSN`为这个操作的LSN。
    -   事务提交：对应事务的ID加入`ATT`表中，同时修改状态为`Commit`，意为需要这个事务的状态为提交。
    -   `CLR`、`checkpoint-end`日志：忽略，这几类日志并不影响`ATT`和`DPT`表的构建。

在数据分析阶段完毕之后：

-   `ATT`表：保存了在崩溃时所有还在活跃的事务以及其状态（需要撤销或者已经提交）。
-   `DPT`表：保存了在崩溃时所有的脏页面编号。

有了这些信息，就能进行重做和撤销操作。

### Redo（重做）阶段

第一步扫描完毕之后，`DPT`表中存储了所有崩溃时的脏页面编号，同时每个脏页面也有一条`recLSN`记录，存储的是这个页面被修改时的最小`LSN`编号，从这些编号中选出编号最小的那条日志，从这条日志开始做重放操作，但并不是所有日志都需要重放，满足以下条件的就不需要重放：

-   修改操作，同时这个修改操作修改的页面编号不在`DPT`表中，或者：
-   虽然页面编号在`DPT`表中，但是这条记录的`LSN`小于`DPT`中记录的该页面的`recLSN`，这说明这个修改的效果已经在`checkpoint`操作中体现，不需要重放了。

### Undo（撤销）阶段

对于所有在`ATT`表中，且状态为`Undo`状态的事务，从后往前进行回滚操作（`ATT`表中每个事务保存了`lastLSN`，可以用这个字段快速定位这个事务的最后一个日志LSN），每回滚一条日志还需要补上对应的`CLR`日志。

### 例子

仍然以一个例子来说明数据恢复流程：

![recovery-1](https://www.codedump.info/media/imgs/20220521-weekly-16/recovery-1.png "recovery-1")

上图中：

-   `MasterRecord`保存的是`LSN`为1的`checkpoint`操作，于是从这条日志开始逐条解析WAL日志，构建`ATT`和`DPT`表。
-   在这个`checkpoint`流程中，T1和T2事务还在活跃，分析结果如下：
    -   `ATT`表（表中每个元素的格式为`(事务编号，状态(U:Undo,C:cOMMIT),lastLSN)`）：
        -   事务T1：因为事务T1最后提交了，因此值为`（T1,C,4）`，表示T1的状态为已提交，且`lastLSN=4`。
        -   事务T2：事务T2没有提交，因此值为`（T2,U,7）`，表示T1的状态为已提交，且`lastLSN=7`。
    -   `DPT`表（表中每个元素的格式为`(页面编号,recLSN)`）：
        -   (P1,4)：表示页面`P1`在LSN=4处修改。
        -   (P2,6)：页面`P2`有两个地方被修改（6、7），但是只需要保存第一处就好了。
-   有了`ATT`和`DPT`数据之后，可以开始重做和撤销流程了：
    -   重做：这个例子中不存在不需要重做的日志，不再阐述。
    -   撤销：可以看到事务T2没有完成，因此需要撤销这个事务的操作，并且补上对应的`CLR`日志。由于`ATT`表中，保存的T2事务的`lastLSN`为7，因此从这条日志开始进行撤销。

数据恢复完成之后，补齐了`CLR`日志如下图所示：

![recovery-2](https://www.codedump.info/media/imgs/20220521-weekly-16/recovery-2.png "recovery-2")

## 总结

-   ARIES算法在每条WAL日志中引入`LSN`，有了这个变量之后：
    
    -   可以把同一个事务的多个操作，在逻辑上串联起来。
    -   可以知道每个页面的最后一次修改。
    -   可以知道数据库文件的最后一次`checkpoint`操作。
    
    简而言之，`LSN`引入之后，给后面处理撤销操作、`checkpoint`、恢复操作都提供了基石。
    
-   `CLR`日志是针对撤销（`Undo`）操作的日志。
    
-   `checkpoint`操作引入`ATT`和`DPT`表用于保存状态，解决了两个问题：
    
    -   不再要求在`checkpoint`开始时要完成所有写事务。
    -   写事务和`checkpoint`可以并发进行。



# Reference
https://www.codedump.info/post/20220410-weekly-12/
https://www.codedump.info/post/20220514-weekly-15/
https://www.codedump.info/post/20220521-weekly-16/#%E9%97%AE%E9%A2%98
