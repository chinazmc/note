想写这篇文章，主要源于两篇文章（论文）中的对mmap在存储引擎中使用的两种截然不同的观点讨论：

-   反方（mmap不应该用于存储引擎）：[Are You Sure You Want to Use MMAP in Your Database Management System? (CIDR 2022)](https://db.cs.cmu.edu/mmap-cidr2022/)
-   正方（mmap可以用于存储引擎）：[re: Are You Sure You Want to Use MMAP in Your Database Management System? - Ayende @ Rahien](https://ayende.com/blog/196161-C/re-are-you-sure-you-want-to-use-mmap-in-your-database-management-system)

由于刚好看过这两种方式的btree存储引擎：sqlite的btree实现以及boltdb，所以可以结合我的认知来聊一聊这个问题。这两个存储引擎的实现都已经整理成了系列博客，这两个系列的第一篇分别是：

-   [sqlite3.36版本 btree实现（零）- 起步及概述 - codedump的网络日志](https://www.codedump.info/post/20211217-sqlite-btree-0/)
-   [boltdb 1.3.0实现分析（一） - codedump的网络日志](https://www.codedump.info/post/20200625-boltdb-1/)

先来看看一个存储引擎实现时的大体分层，以sqlite为例分为三层：

![btree架构](https://www.codedump.info/media/imgs/20211217-sqlite-btree-0/btree-arch.png "btree架构")

自下而上，这三个层次分别是：

-   os层：封装系统级API实现文件的读写等操作。
-   页面管理层：提供以页面为单位的读、写、加载、缓存等操作。
-   btree实现：btree以物理页面为单位向下一层的页面管理层来读写页面，而物理页面内部的逻辑组织（比如父子关系），以及页面内的数据组织（比如一个页面中管理的数据）由这一层负责。

可以这样来简单区别理解“页面管理”模块和btree模块的功能：

-   页面管理：顾名思义，页面管理模块的最基本单位是”页面“，页面的读、写、缓存、落盘、恢复、回滚等，都由页面模块负责。上一层依赖页面管理模块的btree模块，不需要关心一个页面何时缓存、何时落盘等细节。即：**页面模块负责页面的物理级别的操作**。
    
-   btree：
    
    -   负责按照btree算法，来组织页面，即负责的是页面之间逻辑关系维护。
    -   除此以外，一个页面内部的数据的物理、逻辑组织，也是btree模块来负责的。
    
    即：**btree负责维护页面间的逻辑关系，以及一个页面内数据的组织。**
    

![以页面物理、逻辑关系的维护看模块划分](https://www.codedump.info/media/imgs/20211217-sqlite-btree-0/page-module.png "以页面物理、逻辑关系的维护看模块划分")

在数据库文件中，通常按照页面为单位来划分文件，比如sqlite一般是4KB大小为一个物理页面，所以一个数据库文件可以看做是一个大的“物理页面数组”，这样的话每个物理页面都有一个对应的编号（从1开始），这个编号通常简称为PID（page id）。

从上面的功能划分可以看到，“页面管理器（也被称为“buffer pool）”的功能是非常复杂的，这里列举几个最关键的：

-   读页面：上层的btree要读一个数据库文件中的页面时，通常传入一个PID，由页面管理器去加载这个页面的数据。而页面数据并不是每次都会到数据库文件中一次磁盘IO读出来，也很可能在内存中，此时就不需要读磁盘操作了。
    
-   写页面：当一个页面被修改后，就被称为“脏页面（dirty page）”，需要落盘；但并不是每一次修改了一个页面的内容之后就马上落盘，其原因在于：
    
    -   一次写事务可能修改了不止一个页面，需要以事务为单位去落盘脏页面。
    -   即便是落盘脏页面，由于涉及到写磁盘操作，所以还会用其他方式减少写磁盘的次数。比如sqlite的wal备份文件机制中，脏页面的内容是首先写入wal文件的，由于写wal文件是一次append操作而不是随机写，所以效率会更高，如果一个脏页面的内容被写入wal文件的话，那么这部分页面内容是不急于马上写入数据库文件的。
-   缓存页面：由于页面缓存的功能，所以还需要一个页面缓存管理的功能，主要负责：
    
    -   使用缓存控制算法（sqlite中是LRU）将经常读到的页面缓存在内存中，这样不必每次都读磁盘加载数据。
    -   当缓存大小不够时，将脏页面落盘，空出来空间加载要读的页面。

看了上面对页面管理器这个模块功能的描述，可以看到：

-   由于有页面缓存的作用，所以能够精准的控制页面缓存的大小。
-   将“脏页面落盘”这个操作，是与具体的事务有关，并不是修改完毕就能直接落盘，否则的话可能会涉及到脏写等问题。比如一个事务修改了1、2、3三个PID的页面，修改页面1之后并不能马上落盘这个修改，需要等到三个页面都改完了才行。

我们来看看如果使用mmap技术来代替上面的“页面管理器”会面对什么问题。

首先，无法做到对内存容量的精准控制。

其次，写事务如何处理，因为当使用mmap技术修改了一个页面时，实际上这个被修改的页面内容何时被OS内核落到硬盘，已经不由使用者来控制了，那么如何解决上面提到的一个事务修改了多个页面需要同时落盘的问题？

以boltdb为例，它使用的是类似COW的机制来解决：

![boltdb实现写事务](https://www.codedump.info/media/imgs/20220327-weekly-11/boltdb-write.png "boltdb实现写事务")

-   boltdb的数据库文件中，前面三个页面依次是：meta1页面、meta2页面、freelist页面，前两者是存储数据库元信息的页面，freelist页面可以认为是存储当前空闲页面数据。之所以需要两个meta页面，是为了保存写信息：即便一次写操作失败，因为还有另一个meta页面，这样就还能恢复回来。每个meta页面都有一个单调递增的事务id，事务id越大说明是越近的写操作。
    
-   图中紧跟着前面三个元页面的是页面1、2、3，假设这里的写事务就修改了这三个页面。
    
-   在进行修改时，首先拿到当前保存较大事务id的meta页面，这里假设是meta1页面。
    
-   然后进行写事务的页面修改，此时的修改还只是在内存中进行。
    
-   写事务完成之后，需要将修改落盘，将按照如下的顺序来进行：
    
    -   首先，数据库文件中新增三个页面保存修改后的内容，而原来的三个页面内容不动。
    -   然后，将原先的三个页面加入freelist空闲页面链表中，这样下一次就可以回收利用了。
    -   最后，将meta1页面的修改落地。
    
    这个修改的顺序必须严格遵守，否则中间一个过程失败整个数据库文件就损坏了。只有当完成meta页面的修改，才认为这次修改完成。
    

这就是boltdb这个使用了mmap来做页面管理的存储引擎，应对写事务操作的手段，本质上算一个COW的做法。

## 总结

-   “页面管理器”在整个存储引擎的架构里，处于承上启下的作用：对上提供物理页面级别的读、写、缓存控制等功能，对下使用OS系统API来实现文件读写操作。
-   其缓存功能，能够达到精准控制页面缓存这部分内容容量的作用；另外，由于一次写事务通常不止修改了一个页面，所以还需要精准控制脏页面的落盘的时机，否则会出现写坏数据库的情况。
-   使用mmap来替代自实现的页面管理器最大的就是这两个问题：
    -   无法做到精准控制页面缓存容量。
    -   采用类COW的做法来解决写事务问题。
-   上面的第二个问题有解决方案，但是问题一貌似没有。所以一个存储引擎如果使用mmap来实现页面管理，可以说这个存储引擎可能只适用于“内存不敏感”的业务场景。