
# kafka熟悉的参数
- `auto.create.topics.enable`：是否允许自动创建Topic。
- `unclean.leader.election.enable`：是否允许Unclean Leader选举。
- `auto.leader.rebalance.enable`：是否允许定期进行Leader选举。

unclean参数这个参数在最新版的Kafka中默认就是false，本来不需要我特意提的，但是比较搞笑的是社区对这个参数的默认值来来回回改了好几版了，鉴于我不知道你用的是哪个版本的Kafka，所以建议你还是显式地把它设置成false吧。
- `log.retention.{hours|minutes|ms}`：这是个“三兄弟”，都是控制一条消息数据被保存多长时间。从优先级上来说ms设置最高、minutes次之、hours最低。
- `log.retention.bytes`：这是指定Broker为消息保存的总磁盘容量大小。
- `message.max.bytes`：控制Broker能够接收的最大消息大小。


# kafka可以不用zk吗
分片太多的话，zk的watch压力太大，写入的数据量过大，ZooKeeper 的性能和稳定性就会下降，
 Kafka 集群比较大，分区数很多的时候，ZooKeeper 存储的元数据就会很多，性能就差了。
基于 Zookeeper 实现的单个 Controller 在分区数太大的时候还有个问题，故障转移太慢了。

最重要的是不想捆绑

选举的话kafka用kraft协议

# redis6.0为什么要用多线程
因为现在硬件软件的速度越来越快了，socket读写的消耗比重越来越大了。
![[Pasted image 20230616105913.png]]
默认没有开期，开启多线程后，还需要设置线程数，否则是不生效的。同样修改redis.conf配置文件。关于线程数的设置，官方有一个建议：4 核的机器建议设置为 2 或 3 个线程，8核的建议设置为 6 个线程，线程数一定要小于机器核数。线程数并不是越大越好，官方认为超过了 8 个基本就没什么意义了。

# kafka怎么做到消息顺序
单分区，或者用消息键指定发送。消费者单线程消费。

# redis的bigkey定位和解决
**1 大Key带来的常见问题**

- Client发现Redis变慢；
- Redis内存不断变大引发OOM，或达到maxmemory设置值引发写阻塞或重要Key被逐出；
- Redis Cluster中的某个node内存远超其余node，但因Redis Cluster的数据迁移最小粒度为Key而无法将node上的内存均衡化；
- 大Key上的读请求使Redis占用服务器全部带宽，自身变慢的同时影响到该服务器上的其它服务；
- 删除一个大Key造成主库较长时间的阻塞并引发同步中断或主从切换；

**2 热Key带来的常见问题**

- 热Key占用大量的Redis CPU时间使其性能变差并影响其它请求；
- Redis Cluster中各node流量不均衡造成Redis Cluster的分布式优势无法被Client利用，一个分片负载很高而其它分片十分空闲从而产生读/写热点问题；
- 在抢购、秒杀活动中，由于商品对应库存Key的请 求量过大超出Redis处理能力造成超卖；
- 热Key的请求压力数量超出Redis的承受能力造成缓存击穿，此时大量强求将直接指向后端存储将其打挂并影响到其它业务；

**通过Redis官方客户端redis-cli的bigkeys参数发现各个类型的大Key，然后再用memory看内存**
**通过Redis官方客户端redis-cli的hotkeys参数发现热Key**
Redis自4.0起提供了hotkeys参数来方便用户进行实例级的热Key分析功，该参数能够返回所有Key的被访问次数，它的缺点同样为不可定制化输出报告，大量的信息会使你在分析结果时复杂度较大，另外，使用该方案的前提条件是将redis-server的maxmemory-policy参数设置为LFU。

# innodb如何管理page页
**Buffer Pool**  
InnoDB内存页的缓冲池，其中的内存页实际是被各种链表给管理起来的。Buffer Pool分为多个Instance，每个Instance的数据是独立的，可以并发的进行读写。  
**Page**  
MySQL中内存管理的最小单位。  
**Free List**  
其上的节点都是未被使用的节点，如果需要从数据库中分配新的数据页，直接从上获取即可。InnoDB需要保证Free List有足够的节点，提供给用户线程用，否则需要从Flush List或者LRU List淘汰一定的节点。InnoDB初始化后，Buffer Chunks中的所有数据页都被加入到Free List，表示所有节点都可用。  
**Flush List**  
保存所有脏页，也就是被修改但还没有被刷到磁盘上的数据页。  
**LRU List**  
保存从磁盘中读取到的数据页，并用一种优化后的LRU算法策略进行淘汰，优化后的算法被称为midpoint insertion strategy。这个算法在LRU列表中加入了midpoint的位置。新读取的页，并不直接放到LRU列表的首部，而是放入到LRU列表的midpoint位置，midpoint默认为5/8的位置。midpoint之前的链表被称为young list, midpoint之后的链表被称为old list。在old list的page在一定时间内被第二次访问时才会被移到young list，当需要淘汰数据时，从old list的尾部进行淘汰。  
为什么需要这种优化策略呢？  
对于普通的LRU算法，如果有索引或表扫描操作，就会导致大量的页被加入到内存cache中，但是这些页可能接下来并不会被频繁使用到，热点的页被这些页逐出内存，导致性能的下降。使用midpoint insertion strategy，这些扫描产生的大量的页，只会被加入到old列表，从而不会影响到热点数据所在的new列表。通常old列表都比较短（取决于你认为数据库的热点数据的比例），这些页就会很快被逐出内存。  
**page hash表**  
通过它，使用space_id和page_no就能快速找到已经被读入内存的数据页，而不用线性遍历LRU List去查找。

我们从一个数据页的访问流程来看下上面介绍的这些内存模块是怎么相互配合来实现高效的内存页管理的。  

1. 首先根据读取的数据页的page no和space id，找到对应的buffer pool instance
2. 根据page hash表查找对应的页是否已经在内存中，如果已经在内存中，则调整该内存页在LRU list中的位置，并返回，否则从磁盘中读取对应的数据页
3. 在free list中找到一个空闲页面来缓存读取到的数据页，如果找到了，将这个页面加入到LRU List中并成功返回，否则代表free list中没有空闲的数据页了
4. 尝试从LRU List的末尾找到一个可以被淘汰的页面，加入到free list中，然后从free list中找到一个空闲的页面来存储当前的数据页
5. 如果在第4步没有找到可以被淘汰的页面，则要进行单页刷脏操作，刷脏后，将释放的内存块加入到free list中


如果redo log的大小达到上限或缓冲池的内存大小到达上限，就会触发脏页刷新逻辑。MySQL也有后台异步线程来定时的刷新一定比例的脏页。

# 为什么写缓冲区，仅仅适用于非唯一普通索引页
**「在《Buffer Pool》中介绍了buffer pool会缓存热的数据页和索引页，减少磁盘读操作，而对于磁盘的写操作，innoDB同样也有类似的策略，即通过change buffer缓解磁盘写操作产生的磁盘IO」**。

- **「Change Buffer是在【非唯一普通索引页】不在buffer pool中时，当对页进行了写操作时，在不影响数据一致性的前提下。InnoDB会将数据先写入Change Buffer中，等未来数据被读取时，再将 change buffer 中的操作merge到原数据页中」**。   
- 在MySQL5.5之前，只针对insert做了优化，叫插入缓冲(insert buffer)，后面进行了优化，对delete和update也有效，叫做写缓冲(change buffer)。

二、Change buffer 执行过程

**「我们知道当执行写操作时，数据页存在于Buffer Pool中时，会直接修改数据页。那如果数据页不存在于Buffer Pool中时，过程会有一些不一样，这种情况会将写操作缓存到Change Buffer中，等未来在特定条件下其合并到Buffer Pool中」**。因此当需要执行一个写入操作时，一般分为走Change buffer和不走Change buffer两种情况。

如果没有Change buffer，那更新可能会变成

- 第一步：先从磁盘读取所在数据页加载到缓冲池，一次磁盘随机读操作；
    
- 第二步：更新Buffer Pool中的数据页，一次内存操作；
    
- 第三步：将更新操作顺序写Redo log，一次磁盘顺序写操作；
    

也就是会多一次磁盘IO，磁盘IO相比较内存操作时很慢的，并发下性能就会急剧下降。
**「是否会产生数据一致性问题」**

- 读取数据，会将Change Buffer中的数据合并到Buffer Pool中。
    
- 如果没有读取，Change也会被被定期刷盘到写缓冲系统表空间。
    
- 数据库奔溃，redo log可以恢复数据。
    

**「因此写入的数据页不在内存中这中情况也不会产生数据一致性问题」**。
**从下图中可以看出，Change Buffer被包含在了Buffer Pool中的，change buffer用的是buffer pool里的内存，由于Buffer Pool的内存大小是有限制的，所以change buffer大小也是有限制的，可通过参数innodb_change_buffer_max_size设置，一般是25%**
**前面说到Change Buffer在MySQL5.5之后可以支持新增、删除、修改的写入，对于受I/O限制的操作（大量DML、如批量插入）有很大的性能提升价值。但是对于一些特定的场景，可以通过修改innodb_change_buffering来变更Change Buffer支持的类型，分别为插入，删除，清除启用或禁用缓冲，更新操作是插入和删除的组合**

既然Change buffer是单独内存中，写入之后会被合并到Buffer Pool中,那么是时候时候会被merge呢？

**「Change buffer会被merge触发时机」**

- 读取Change buffer中记录的数据页时，会将Change buffer合并到buffer Pool 中，然后被刷新到磁盘。
    
- 当系统空闲或者slow shutdown时，后台master线程发起merge。
    
- change buffer的内存空间用完了，后台master线程会发起merge。
    
- redo log写满了，但是一般不会发生。

**「不知道大家有没有印象，在本文第一节就重点说了一个词【非唯一普通索引页】，Change buffer只有在非唯一普通索引页时才生效，这是为什么呢？」**

相信大家在日常工作中会经常遇到一个问题：**「主键冲突」**。

- **「主键索引，唯一索引」**
    
    实际上对于【唯一索引】的更新，插入操作都会**「先判断当前操作是否违反唯一性约束」**，而这个操作就必须要将索引页读取到内存中，此时既然已经读取到内存了，那直接更新即可，没有需要在用Change buffer了。
    
- **「非唯一普通索引」**
    
    **「不需要判断当前操作是否违反唯一性约束」**，也就不需要将数据页读取到内存，因此可以直接使用 change buffer 更新。
    

**「基于此，Change buffer只有对普通索引可以使用，对唯一索引的更新无法生效」**。


# 使用了索引一定提高效率吗
简单说说哪些情况不建议为字段添加索引， 如果我们的数据量比较少的情况下，也就没有必要添加索引，比如就只有几十几百的数据量，这种为其添加索引没意义，还降低了除查询操作之外的操作的性能。

然后就是增删改的操作十分频繁的表，不建议为其字段建索引，上一章也说了，这样不仅没有充分利用其优点，还大大降低了增删改的性能。因为增删改难免会对B+树的顺序造成破坏，也就还会花时间去维护B+树。相反对于增删改操作不频繁的，而查询操作十分频繁的表，那么我们就可以建立与之对应的索引了。

还有一个点，不建议对唯一字段建索引，也就是设置了unique关键字的字段。为啥呢？ 这里不建议主要是针对添加操作，因为在添加操作时，还要去进行查询，看有无与之重复的值，随后再去进行添加。但如果查询次数远远大于其他操作，我觉得还是问题不大。

# redo log和undo log的持久化策略
- **innodb_flush_log_at_trx_commit**

- innodb_flush_log_at_trx_commit=1，每次commit都会把redo log从redo log buffer写入到system，并fsync刷新到磁盘文件中。
- innodb_flush_log_at_trx_commit=2，每次事务提交时MySQL会把日志从redo log buffer写入到system，但只写入到file system buffer，由系统内部来fsync到磁盘文件。如果数据库实例crash，不会丢失redo log，但是如果服务器crash，由于file system buffer还来不及fsync到磁盘文件，所以会丢失这一部分的数据。
- innodb_flush_log_at_trx_commit=0，事务发生过程，日志一直激励在redo log buffer中，跟其他设置一样，但是在事务提交时，不产生redo 写操作，而是MySQL内部每秒操作一次，从redo log buffer，把数据写入到系统中去。如果发生crash，即丢失1s内的事务修改操作。

- 注意：由于进程调度策略问题,这个“每秒执行一次 flush(刷到磁盘)操作”并不是保证100%的“每秒”。

前面说到未提交的事务和回滚了的事务也会记录Redo Log，因此在进行恢复时,这些事务要进行特殊的的处理。有2种不同的恢复策略：

恢复
前面说到未提交的事务和回滚了的事务也会记录Redo Log，因此在进行恢复时,这些事务要进行特殊的的处理。有2种不同的恢复策略：

  A. 进行恢复时，只重做已经提交了的事务。  
  B. 进行恢复时，重做所有事务包括未提交的事务和回滚了的事务。然后通过Undo Log回滚那些未提交的事务。

  **MySQL数据库InnoDB存储引擎使用了B策略,** InnoDB存储引擎中的恢复机制有几个特点：

  A. 在重做Redo Log时，并**不关心事务性**。 恢复时，没有BEGIN，也没有COMMIT,ROLLBACK的行为。也不关心每个日志是哪个事务的。尽管事务ID等事务相关的内容会记入Redo Log，这些内容只是被当作要操作的数据的一部分。

  B. 使用B策略就必须要将Undo Log持久化，而且必须要在写Redo Log之前将对应的Undo Log写入磁盘。Undo和Redo Log的这种关联，使得持久化变得复杂起来。为了降低复杂度，InnoDB将Undo Log看作数据，因此记录Undo Log的操作也会记录到redo log中。这样undo log就可以象数据一样缓存起来，而不用在redo log之前写入磁盘了。

     包含Undo Log操作的Redo Log，看起来是这样的：  
     记录1: <trx1, **Undo log insert** <undo_insert …>>  
     记录2: <trx1, insert …>  
     记录3: <trx2, **Undo log insert** <undo_update …>>  
     记录4: <trx2, update …>  
     记录5: <trx3, ****Undo log insert**** <undo_delete …>>  
     记录6: <trx3, delete …>  
  C. 到这里，还有一个问题没有弄清楚。既然Redo没有事务性，那岂不是会重新执行被回滚了的事务？  
     确实是这样。同时Innodb也会将事务回滚时的操作也记录到redo log中。回滚操作本质上也是  
     对数据进行修改，因此回滚时对数据的操作也会记录到Redo Log中。  
     一个回滚了的事务的Redo Log，看起来是这样的：  
     记录1: <trx1, Undo log insert <undo_insert …>>  
     记录2: <trx1, **insert A**…>  
     记录3: <trx1, Undo log insert <undo_update …>>  
     记录4: <trx1, **update B**…>  
     记录5: <trx1, Undo log insert <undo_delete …>>  
     记录6: <trx1, **delete C**…>  
     记录7: <trx1, **insert C**>  
     记录8: <trx1, **update B** to old value>

     记录9: <trx1, **delete A>**

     一个被回滚了的事务在恢复时的操作就是先redo再undo，因此不会破坏数据的一致性。

     未提交的事务。

之所以能同时保证原子性和持久化，是因为以下特点：

更新数据前记录undo log。
为了保证持久性，必须将数据在事务提交前写到磁盘，只要事务成功提交，数据必然已经持久化到磁盘。
undo log必须先于数据持久化到磁盘。如果在G,H之间发生系统崩溃，undo log是完整的，可以用来回滚。
如果在A - F之间发生系统崩溃，因为数据没有持久化到磁盘，所以磁盘上的数据还是保持在事务开始前的状态。
缺陷：每个事务提交前将数据和undo log写入磁盘，这样会导致大量的磁盘IO，因此性能较差。 如果能够将数据缓存一段时间，就能减少IO提高性能，但是这样就会失去事务的持久性。

# binlog日志格式
binlog格式分为statement，row以及mixed三种，mysql5.5默认的还是statement模式，当然我们在主从同步中一般是不建议用statement模式的，因为会有些语句不支持，`比如语句中包含UUID函数，以及LOAD DATA IN FILE语句`等，一般推荐的是mixed格式。暂且不管这三种格式的区别，看看binlog的存储格式是什么样的。binlog是一个二进制文件集合，当然除了我们看到的mysql-bin.xxxxxx这些binlog文件外，还有个binlog索引文件mysql-bin.index。如官方文档中所写，binlog格式如下：  

- binlog文件以一个值为0Xfe62696e的魔数开头，这个魔数对应0xfe 'b''i''n'。
- binlog由一系列的binlog event构成。每个binlog event包含header和data两部分。

- header部分提供的是event的公共的类型信息，包括event的创建时间，服务器等等。
- data部分提供的是针对该event的具体信息，如具体数据的修改。

  
- 从mysql5.0版本开始，binlog采用的是v4版本，第一个event都是`format_desc event` 用于描述binlog文件的格式版本，这个格式就是event写入binlog文件的格式。关于之前版本的binlog格式，可以参见[http://dev.mysql.com/doc/internals/en/binary-log-versions.html](https://link.zhihu.com/?target=https%3A//link.jianshu.com/%3Ft%3Dhttp%3A//dev.mysql.com/doc/internals/en/binary-log-versions.html)
- 接下来的event就是按照上面的格式版本写入的event。
- 最后一个`rotate event`用于说明下一个binlog文件。
- binlog索引文件是一个文本文件，其中内容为当前的binlog文件列表。比如下面就是一个mysql-bin.index文件的内容。

- statement：基于SQL语句的模式，某些语句和函数如UUID, LOAD DATA INFILE等在复制过程可能导致数据不一致甚至出错。
- row：基于行的模式，记录的是行的变化，很安全。但是binlog会比其他两种模式大很多，在一些大表中清除大量数据时在binlog中会生成很多条语句，可能导致从库延迟变大。
- mixed：混合模式，根据语句来选用是statement还是row模式。

# kafka生产者的sticky分区策略有什么优势
Kafka Range RoundRobin 和Sticky 三种 分区分配策略
一、Kafka默认分区分配策略
1、1 consumer 订阅 1 topic ( 7 partition )
按照Kafka默认的消费逻辑设定，一个分区只能被同一个消费组（ConsumerGroup）内的一个消费者消费。

假设目前某消费组内只有一个消费者C0，订阅了一个topic，这个topic包含7个分区，也就是说这个消费者C0订阅了7个分区，参考下图。
![[Pasted image 20230616142451.png]]

2、2 consumer 订阅 1 topic ( 7 partition )
此时消费组内又加入了一个新的消费者C1，按照既定的逻辑需要将原来消费者C0的部分分区分配给消费者C1消费，情形如下图，消费者C0和C1各自负责消费所分配到的分区，相互之间并无实质性的干扰。
![[Pasted image 20230616142643.png]]

3、3 consumer 订阅 1 topic ( 7 partition )
接着消费组内又加入了一个新的消费者C2，如此消费者C0、C1和C2按照上图（3）中的方式各自负责消费所分配到的分区。
![[Pasted image 20230616142705.png]]

4、8 consumer 订阅 1 topic ( 7 partition )
如果消费者过多，出现了消费者的数量大于分区的数量的情况，就会有消费者分配不到任何分区。参考下图，一共有8个消费者，7个分区，那么最后的消费者C7由于分配不到任何分区进而就无法消费任何消息。
![[Pasted image 20230616142715.png]]
上面各个示例中的整套逻辑是按照Kafka中默认的分区分配策略来实施的。Kafka提供了消费者客户端参数partition.assignment.strategy用来设置消费者与订阅主题之间的分区分配策略。

默认情况下，此参数的值为：org.apache.kafka.clients.consumer.RangeAssignor，即采用RangeAssignor分配策略。除此之外，Kafka中还提供了另外两种分配策略： RoundRobinAssignor和StickyAssignor。消费者客户端参数partition.asssignment.strategy可以配置多个分配策略，彼此之间以逗号分隔。

二、RangeAssignor分配策略
RangeAssignor策略的原理是按照消费者总数和分区总数进行整除运算来获得一个跨度，然后将分区按照跨度进行平均分配，以保证分区尽可能均匀地分配给所有的消费者。对于每一个topic，RangeAssignor策略会将消费组内所有订阅这个topic的消费者按照名称的字典序排序，然后为每个消费者划分固定的分区范围，如果不够平均分配，那么字典序靠前的消费者会被多分配一个分区。

① 假设n=一个topic的分区数 / 订阅此topic消费者数量，
② m=分区数%消费者数量，
③ 那么前m个消费者每个分配n+1个分区，后面的（消费者数量-m）个消费者每个分配n个分区。

假设一个 topic 有13 分区，订阅此topic 有 3 个消费者 C0、C1、C2，那么每个消费者先分配 13 / 3 = 4 个分区，剩下 1 ( 13 % 3 = )个分区，给第一个消费者 C0。
此时各个消费则分配 topic 的分区如下:
C0 : 4 + 1个
C1 : 4 个
C2 : 4 个

为了更加通俗的讲解 RangeAssignor 策略，我们不妨再举一些示例。
假设消费组内有2个消费者C0和C1，都订阅了主题t0和t1，并且每个主题都有4个分区，那么所订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t0p3、t1p0、t1p1、t1p2、t1p3。最终的分配结果为：
![[Pasted image 20230616142727.png]]

消费者C0：t0p0、t0p1、t1p0、t1p1
消费者C1：t0p2、t0p3、t1p2、t1p3

这样分配的很均匀，那么此种分配策略能够一直保持这种良好的特性呢？我们再来看下另外一种情况。假设上面例子中2个主题都只有3个分区，那么所订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：
![[Pasted image 20230616142754.png]]
消费者C0：t0p0、t0p1、t1p0、t1p1
消费者C1：t0p2、t1p2

可以明显的看到这样的分配并不均匀，如果将类似的情形扩大，有可能会出现部分消费者过载的情况。对此我们再来看下另一种RoundRobinAssignor策略的分配效果如何。

三、RoundRobinAssignor分配策略
RoundRobinAssignor策略的原理是将消费组内所有消费者以及消费者所订阅的所有topic的partition按照字典序排序，然后通过轮询消费者方式逐个将分区分配给每个消费者。RoundRobinAssignor策略对应的partition.assignment.strategy参数值为：org.apache.kafka.clients.consumer.RoundRobinAssignor。

消费者订阅相同 Topic
如果同一个消费组内所有的消费者的订阅信息都是相同的，那么RoundRobinAssignor策略的分区分配会是均匀的。
举例，假设消费组中有2个消费者C0和C1，都订阅了主题t0和t1，并且每个主题都有3个分区，那么所订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：
![[Pasted image 20230616142909.png]]
消费者C0：t0p0、t0p2、t1p1
消费者C1：t0p1、t1p0、t1p2

2. 消费者订阅不同 Topic
如果同一个消费组内的消费者所订阅的Topic 是不相同的，那么在执行分区分配的时候就不是完全的轮询分配，有可能会导致分区分配的不均匀。如果某个消费者没有订阅消费组内的某个topic，那么在分配分区的时候此消费者将分配不到这个topic的任何分区。

举例，假设消费组内有3个消费者C0、C1和C2，它们共订阅了3个主题：t0、t1、t2，这3个主题分别有1、2、3个分区，即整个消费组订阅了t0p0、t1p0、t1p1、t2p0、t2p1、t2p2这6个分区。
![[Pasted image 20230616142927.png]]

具体而言，消费者C0订阅的是主题t0，消费者C1订阅的是主题t0和t1，消费者C2订阅的是主题t0、t1和t2，那么最终的分配结果为：
![[Pasted image 20230616142933.png]]

消费者C0：t0p0
消费者C1：t1p0
消费者C2：t1p1、t2p0、t2p1、t2p2

可以看到RoundRobinAssignor策略也不是十分完美，这样分配其实并不是最优解，因为完全可以将分区t1p1分配给消费者C1，如下图：
![[Pasted image 20230616142939.png]]

四、StickyAssignor分配策略
我们再来看一下StickyAssignor策略，“sticky”这个单词可以翻译为“粘性的”，Kafka从0.11.x版本开始引入这种分配策略，它主要有两个目的：

① 分区的分配要尽可能的均匀；
② 分区的分配尽可能的与上次分配的保持相同。

当两者发生冲突时，第一个目标优先于第二个目标。鉴于这两个目标，StickyAssignor策略的具体实现要比RangeAssignor和RoundRobinAssignor这两种分配策略要复杂很多。我们举例来看一下StickyAssignor策略的实际效果。

1. 消费者订阅相同 Topic
假设消费组内有3个消费者：C0、C1和C2，它们都订阅了4个主题：t0、t1、t2、t3，并且每个主题有2个分区，也就是说整个消费组订阅了t0p0、t0p1、t1p0、t1p1、t2p0、t2p1、t3p0、t3p1这8个分区。最终的分配结果如下：
![[Pasted image 20230616142952.png]]

消费者C0：t0p0、t1p1、t3p0
消费者C1：t0p1、t2p0、t3p1
消费者C2：t1p0、t2p1

这样初看上去似乎与采用RoundRobinAssignor策略所分配的结果相同，但事实是否真的如此呢？再假设此时消费者C1脱离了消费组，那么消费组就会执行再平衡操作，进而消费分区会重新分配。如果采用RoundRobinAssignor策略，那么此时的分配结果如下：
![[Pasted image 20230616143004.png]]

消费者C0：t0p0、t1p0、t2p0、t3p0
消费者C2：t0p1、t1p1、t2p1、t3p1

如分配结果所示，RoundRobinAssignor策略会按照消费者C0和C2进行重新轮询分配。而如果此时使用的是StickyAssignor策略，那么分配结果为：
![[Pasted image 20230616143012.png]]
消费者C0：t0p0、t1p1、t3p0、t2p0
消费者C2：t1p0、t2p1、t0p1、t3p1
可以看到分配结果中保留了上一次分配中对于消费者C0和C2的所有分配结果，并将原来消费者C1的“负担”分配给了剩余的两个消费者C0和C2，最终C0和C2的分配还保持了均衡。

如果发生分区重分配，那么对于同一个分区而言有可能之前的消费者和新指派的消费者不是同一个，对于之前消费者进行到一半的处理还要在新指派的消费者中再次复现一遍，这显然很浪费系统资源。StickyAssignor策略如同其名称中的“sticky”一样，让分配策略具备一定的“粘性”，尽可能地让前后两次分配相同，进而减少系统资源的损耗以及其它异常情况的发生。

2. 消费者订阅不同 Topic
到目前为止所分析的都是消费者的订阅信息都是相同的情况，我们来看一下订阅信息不同的
情况下的处理。

举例，同样消费组内有3个消费者：C0、C1和C2，集群中有3个主题：t0、t1和t2，这3个主题分别有1、2、3个分区，也就是说集群中有t0p0、t1p0、t1p1、t2p0、t2p1、t2p2这6个分区。消费者C0订阅了主题t0，消费者C1订阅了主题t0和t1，消费者C2订阅了主题t0、t1和t2。

如果此时采用RoundRobinAssignor策略，那么最终的分配结果如下所示（和讲述RoundRobinAssignor策略时的一样，这样不妨赘述一下）：
( 红线是订阅，其他颜色的线是分配分区 )
![[Pasted image 20230616143023.png]]

【分配结果集1】
消费者C0：t0p0
消费者C1：t1p0
消费者C2：t1p1、t2p0、t2p1、t2p2

如果此时采用的是StickyAssignor策略，那么最终的分配结果为：
( 红线是订阅，其他颜色的线是分配分区 )
![[Pasted image 20230616143035.png]]

【分配结果集2】
消费者C0：t0p0
消费者C1：t1p0、t1p1
消费者C2：t2p0、t2p1、t2p2
可以看到这是一个最优解（消费者C0没有订阅主题t1和t2，所以不能分配主题t1和t2中的任何分区给它，对于消费者C1也可同理推断）。
3. 消费者脱离消费组的情况 RoundRobin
假如此时消费者C0脱离了消费组，那么RoundRobinAssignor策略的分配结果为：
( 红线是订阅，其他颜色的线是分配分区 )
![[Pasted image 20230616143043.png]]


消费者C1：t0p0、t1p1
消费者C2：t1p0、t2p0、t2p1、t2p2
可以看到RoundRobinAssignor策略保留了消费者C1和C2中原有的3个分区的分配：t2p0、t2p1和t2p2（针对结果集1, 保留了三个绿色的,结果集1如下图，做参照）。
![[Pasted image 20230616143053.png]]

消费者脱离消费组的情况 Sticky
而如果采用的是StickyAssignor策略，那么分配结果为：
( 红线是订阅，其他颜色的线是分配分区 )
![[Pasted image 20230616143058.png]]

消费者C1：t1p0、t1p1、t0p0
消费者C2：t2p0、t2p1、t2p2

可以看到StickyAssignor策略保留了消费者C1和C2中原有的5个分区的分配：t1p0、t1p1、t2p0、t2p1、t2p2。（针对结果集2, 保留了三个绿色的,结果集2如下图，做参照）。
![[Pasted image 20230616143110.png]]

从结果上看StickyAssignor策略比另外两者分配策略而言显得更加的优异，这个策略的代码实现也是异常复杂，如果大家在一个 group 里面，不同的 Consumer 订阅不同的 topic, 那么设置Sticky 分配策略还是很有必要的。
https://blog.csdn.net/u010022158/article/details/106271208

# kafka如何基于多层时间轮实现延迟功能，并且解决空转问题
![[Pasted image 20230619143632.png]]
Kafka中存在一些定时任务(DelayedOperation)，如DelayedFetch、DelayedProduce、DelayedHeartbeat等，在Kafka中，定时任务的添加、轮转、执行、消亡等是通过时间轮来实现的。(时间轮并不是Kafka独有的设计，而是一种通用的实现方式，Netty中也有用到时间轮的方式)
延时请求（Delayed Operation），也称延迟请求，是指因未满足条件而暂时无法被处理的 Kafka 请求。举个例子，配置了 acks=all 的生产者发送的请求可能一时无法完成，因为 Kafka 必须确保 ISR 中的所有副本都要成功响应这次写入。因此，通常情况下，这些请求没法被立即处理。只有满足了条件或发生了超时，Kafka 才会把该请求标记为完成状态。这就是所谓的延时请求。

## 1. 时间轮是什么

参考网上的两张图（摘自 [blog.csdn.net/u013256816/…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fu013256816%2Farticle%2Fdetails%2F80697456%25EF%25BC%2589 "https://blog.csdn.net/u013256816/article/details/80697456%EF%BC%89")
![[Pasted image 20230619143735.png]]
这两张图就比较清楚的说明了Kafka时间轮的结构了：类似现实中的钟表，由多个环形数组组成，每个环形数组包含20个时间单位，表示一个时间维度（一轮），如：第一层时间轮，数组中的每个元素代表1ms，一圈就是20ms，当延迟时间大于20ms时，就“进位”到第二层时间轮，第二层中，每“一格”表示20ms，依此类推…

对于一个延迟任务，大体包含三个过程：进入时间轮、降级和到期执行。

- 进入时间轮

1. 根据延迟时间计算对应的时间轮“层次”（如钟表中的“小时级”还是“分钟级”还是“秒级”，实际上是一个不断“升级”的过程，直到找到合适的“层次”）
2. 计算在该轮中的位置，并插入该位置（每个bucket是一个双向链表，可能包含多个延迟任务，这也是时间轮提高效率的一大原因，后面会提到）
3. 若该bucket是首次插入，需要将该bucket加入DelayQueue中（DelayQueue的引入是为了解决“空推进”，后面会提到）
![[Pasted image 20230619143758.png]]
- 降级

1. 当时间“推进”到某个bucket时，说明该bucket中的任务在当前时间轮中的时间已经走完，需要进行“降级”，即进入更小粒度的时间轮中，reinsert的过程和进入时间轮是类似的
![[Pasted image 20230619143814.png]]
- 到期执行

1. 在reinsert的过程中，若发现已经到期，则执行这些任务
![[Pasted image 20230619143825.png]]
整体过程大致如下：
![[Pasted image 20230619143851.png]]
## 2. 时间的“推进”

一种直观的想法是，像现实中的钟表一样，“一格一格”地走，这样就需要有一个线程一直不停的执行，而大多数情况下，时间轮中的bucket大部分是空的，指针的“推进”就没有实质作用，因此，为了减少这种“空推进”，Kafka引入了DelayQueue，以bucket为单位入队，每当有bucket到期，即queue.poll能拿到结果时，才进行时间的“推进”，减少了 ExpiredOperationReaper 线程空转的开销。
![[Pasted image 20230619143902.png]]
## 3. 为什么要用时间轮

用到延迟任务时，比较直接的想法是DelayQueue、ScheduledThreadPoolExecutor 这些，而时间轮相比之下，最大的优势是在时间复杂度上：

时间复杂度对比：
![[Pasted image 20230619143910.png]]
因此，理论上，当任务较多时，TimingWheel的时间性能优势会更明显
但是，DelayQueue 有一个弊端：它插入和删除队列元素的时间复杂度是 O(logN)。对于 Kafka 这种非常容易积攒几十万个延时请求的场景来说，该数据结构的性能是瓶颈。当然，这一版的设计还有其他弊端，比如，它在清除已过期的延迟请求方面不够高效，可能会出现内存溢出的情形。后来，社区改造了延时请求的实现机制，采用了基于时间轮的方案。
时间轮有简单时间轮（Simple Timing Wheel）和分层时间轮（Hierarchical Timing Wheel）两类。两者各有利弊，也都有各自的使用场景。Kafka 采用的是分层时间轮，这是我们重点学习的内容。
总结一下Kafka时间轮性能高的几个主要原因：

（1）时间轮的结构+双向列表bucket，使得插入操作可以达到O(1)的时间复杂度

（2）Bucket的设计让多个任务“合并”，使得同一个bucket的多次插入只需要在delayQueue中入队一次，同时减少了delayQueue中元素数量，堆的深度也减小，delayqueue的插入和弹出操作开销也更小

## 时间轮的复用
![[Pasted image 20230619152820.png]]
取模运算
![[Pasted image 20230619153024.png]]
33的话太大了就按需创建一个新的上层时间轮（时间轮升级）
![[Pasted image 20230619153146.png]]
第一层时间轮走完一圈的时候，第二层就向前一格

![[Pasted image 20230619154011.png]]
方法的第 1 步是获取定时任务的过期时间戳。所谓过期时间戳，就是这个定时任务过期时的时点。

第 2 步是看定时任务是否已被取消。如果已经被取消，则无需加入到时间轮中。如果没有被取消，就接着看这个定时任务是否已经过期。如果过期了，自然也不用加入到时间轮中。如果没有过期，就看这个定时任务的过期时间是否能够被涵盖在本层时间轮的时间范围内。如果可以，则进入到下一步。

第 3 步，首先计算目标 Bucket 序号，也就是这个定时任务需要被保存在哪个 TimerTaskList 中。我举个实际的例子，来说明一下如何计算目标 Bucket。

前面说过了，第 1 层的时间轮有 20 个 Bucket，每个滴答时长是 1 毫秒。那么，第 2 层时间轮的滴答时长应该就是 20 毫秒，总时长是 400 毫秒。第 2 层第 1 个 Bucket 的时间范围应该是[20，40)，第 2 个 Bucket 的时间范围是[40，60），依次类推。假设现在有个延时请求的超时时间戳是 237，那么，它就应该被插入到第 11 个 Bucket 中。

在确定了目标 Bucket 序号之后，代码会将该定时任务添加到这个 Bucket 下，同时更新这个 Bucket 的过期时间戳。在刚刚的那个例子中，第 11 号 Bucket 的起始时间就应该是小于 237 的最大的 20 的倍数，即 220。

第 4 步，如果这个 Bucket 是首次插入定时任务，那么，还同时要将这个 Bucket 加入到 DelayQueue 中，方便 Kafka 轻松地获取那些已过期 Bucket，并删除它们。如果定时任务的过期时间无法被涵盖在本层时间轮中，那么，就按需创建上一层时间戳，然后在上一层时间轮上完整地执行刚刚所说的所有逻辑。

举个例子，目前，Kafka 第 1 层的时间轮固定时长是 20 毫秒（interval），即有 20 个 Bucket（wheelSize），每个 Bucket 涵盖 1 毫秒（tickMs）的时间范围。第 2 层的总时长是 400 毫秒，同样有 20 个 Bucket，每个 Bucket 20 毫秒。依次类推，那么第 3 层的时间轮时长就是 8 秒，因为这一层单个 Bucket 的时长是 400 毫秒，共有 20 个 Bucket。

基于这种设计，每个延迟请求需要根据自己的超时时间，来决定它要被保存于哪一层时间轮上。我们假设在 t=0 时创建了第 1 层的时间轮，那么，该层第 1 个 Bucket 保存的延迟请求就是介于[0，1）之间，第 2 个 Bucket 保存的是介于[1，2) 之间的请求。现在，如果有两个延迟请求，超时时刻分别在 18.5 毫秒和 123 毫秒，那么，第 1 个请求就应该被保存在第 1 层的第 19 个 Bucket（序号从 1 开始）中，而第 2 个请求，则应该被保存在第 2 层时间轮的第 6 个 Bucket 中。
这基本上就是 Kafka 中分层时间轮的实现原理。Kafka 不断向前推动各个层级的时间轮的时钟，按照时间轮的滴答时长，陆续接触到 Bucket 下的各个延迟任务，从而实现了对请求的延迟处理。

![[Pasted image 20230619154237.png]]
DelayQueue队列中的元素按到期时间排序，队列头部的元素2s以后到期。消费者线程１查看了头部元素以后，发现还需要2s才到期，于是它进入等待状态，2s以后醒来，等待头部元素到期的线程称为Leader线程。（这个解决了空转的问题）
advanceClock 方法要做的事情，就是遍历 delayQueue 中的所有 Bucket，并将时间轮的时钟依次推进到它们的过期时间点，令它们过期。然后，再将这些 Bucket 下的所有定时任务全部重新插入回时间轮。

我用一张图来说明这个重新插入过程。
![[Pasted image 20230619154603.png]]
从这张图中，我们可以看到，在 T0 时刻，任务①存放在 Level 0 的时间轮上，而任务②和③存放在 Level 1 的时间轮上。此时，时钟推进到 Level 0 的第 0 个 Bucket 上，以及 Level 1 的第 0 个 Bucket 上。

当时间来到 T19 时刻，时钟也被推进到 Level 0 的第 19 个 Bucket，任务①会被执行。但是，由于一层时间轮是 20 个 Bucket，因此，T19 时刻 Level 0 的时间轮尚未完整走完一圈，此时，Level 1 的时间轮状态没有发生任何变化。

当 T20 时刻到达时，Level 0 的时间轮已经执行完成，Level 1 的时间轮执行了一次滴答，向前推进一格。此时，Kafka 需要将任务②和③插入到 Level 0 的时间轮上，位置是第 20 个和第 21 个 Bucket。这个将高层时间轮上的任务插入到低层时间轮的过程，是由 advanceClock 中的 reinsert 方法完成。

至于为什么要重新插入回低层次的时间轮，其实是因为，随着时钟的推进，当前时间逐渐逼近任务②和③的超时时间点。它们之间差值的缩小，足以让它们被放入到下一层的时间轮中。

总的来说，SystemTimer 类实现了 Timer 接口的方法，它封装了底层的分层时间轮，为上层调用方提供了便捷的方法来操作时间轮。那么，它的上层调用方是谁呢？答案就是 DelayedOperationPurgatory 类。这就是我们建模 Purgatory 的地方。

## 总结

今天，我们重点学习了分层时间轮的上层组件，包括 Timer 接口及其实现类 SystemTimer、DelayedOperation 类以及 DelayedOperationPurgatory 类。你基本上可以认为，它们是逐级被调用的关系，即 DelayedOperation 调用 SystemTimer 类，DelayedOperationPurgatory 管理 DelayedOperation。它们共同实现了 Broker 端对于延迟请求的处理，基本思想就是，能立即完成的请求马上完成，否则就放入到名为 Purgatory 的缓冲区中。后续，DelayedOperationPurgatory 类的方法会自动地处理这些延迟请求。

我们来回顾一下重点。

SystemTimer 类：Kafka 定义的定时器类，封装了底层分层时间轮，实现了时间轮 Bucket 的管理以及时钟向前推进功能。它是实现延迟请求后续被自动处理的基础。

DelayedOperation 类：延迟请求的高阶抽象类，提供了完成请求以及请求完成和过期后的回调逻辑实现。

DelayedOperationPurgatory 类：Purgatory 实现类，该类定义了 WatcherList 对象以及对 WatcherList 的操作方法，而 WatcherList 是实现延迟请求后续自动处理的关键数据结构。

总的来说，延迟请求模块属于 Kafka 的冷门组件。毕竟，大部分的请求还是能够被立即处理的。了解这部分模块的最大意义在于，你可以学习 Kafka 这个分布式系统是如何异步循环操作和管理定时任务的。这个功能是所有分布式系统都要面临的课题，因此，弄明白了这部分的原理和代码实现，后续我们在自行设计类似的功能模块时，就非常容易了。

# kafka的controller保存了什么
![[Pasted image 20230619164855.png]]

在我们公司的 Kafka 集群环境上，曾经出现了一个比较“诡异”的问题：某些核心业务的主题分区一直处于“不可用”状态。

通过使用“kafka-topics”命令查询，我们发现，这些分区的 Leader 显示是 -1。之前，这些 Leader 所在的 Broker 机器因为负载高宕机了，当 Broker 重启回来后，Controller 竟然无法成功地为这些分区选举 Leader，因此，它们一直处于“不可用”状态。

由于是生产环境，我们的当务之急是马上恢复受损分区，然后才能调研问题的原因。有人提出，重启这些分区旧 Leader 所在的所有 Broker 机器——这很容易想到，毕竟“重启大法”一直很好用。但是，这一次竟然没有任何作用。

之后，有人建议升级重启大法，即重启集群的所有 Broker——这在当时是不能接受的。且不说有很多业务依然在运行着，单是重启 Kafka 集群本身，就是一件非常缺乏计划性的事情。毕竟，生产环境怎么能随意重启呢？！

后来，我突然想到了 Controller 组件中重新选举 Controller 的代码。一旦 Controller 被选举出来，它就会向所有 Broker 更新集群元数据，也就是说，会“重刷”这些分区的状态。

那么问题来了，我们如何在避免重启集群的情况下，干掉已有 Controller 并执行新的 Controller 选举呢？答案就在源码中的 ControllerZNode.path 上，也就是 ZooKeeper 的 /controller 节点。倘若我们手动删除了 /controller 节点，Kafka 集群就会触发 Controller 选举。于是，我们马上实施了这个方案，效果出奇得好：之前的受损分区全部恢复正常，业务数据得以正常生产和消费。
![[Pasted image 20230619164927.png]]

# controller如何管理请求
![[Pasted image 20230619165301.png]]
LeaderAndIsrRequest：最主要的功能是，告诉 Broker 相关主题各个分区的 Leader 副本位于哪台 Broker 上、ISR 中的副本都在哪些 Broker 上。在我看来，它应该被赋予最高的优先级，毕竟，它有令数据类请求直接失效的本领。试想一下，如果这个请求中的 Leader 副本变更了，之前发往老的 Leader 的 PRODUCE 请求是不是全部失效了？因此，我认为它是非常重要的控制类请求。

StopReplicaRequest：告知指定 Broker 停止它上面的副本对象，该请求甚至还能删除副本底层的日志数据。这个请求主要的使用场景，是分区副本迁移和删除主题。在这两个场景下，都要涉及停掉 Broker 上的副本操作。

UpdateMetadataRequest：顾名思义，该请求会更新 Broker 上的元数据缓存。集群上的所有元数据变更，都首先发生在 Controller 端，然后再经由这个请求广播给集群上的所有 Broker。在我刚刚分享的案例中，正是因为这个请求被处理得不及时，才导致集群 Broker 无法获取到最新的元数据信息。

# kafka 的controler选举和分片leader选举
如图所示，集群上所有的 Broker 都在实时监听 ZooKeeper 上的这个节点。这里的“监听”有两个含义。

监听这个节点是否存在。倘若发现这个节点不存在，Broker 会立即“抢注”该节点，即创建 /controller 节点。创建成功的那个 Broker，即当选为新一届的 Controller。

监听这个节点数据是否发生了变更。同样，一旦发现该节点的内容发生了变化，Broker 也会立即启动新一轮的 Controller 选举。
![[Pasted image 20230619170133.png]]

## controller kraft选举
### **消息定义**

经典Raft协议只定义了两种RPC消息，即AppendEntries与RequestVote，并且是以推模式交互的。为了适应Kafka环境，KRaft协议以拉模式交互，定义的RPC消息有如下几种。

- Vote：选举的选票信息，由Candidate发送；
- BeginQuorumEpoch：新Leader当选时发送，告知其他节点当前的Leader信息；
- EndQuorumEpoch：当前Leader退位时发送，触发重新选举，用于graceful shutdown；
- Fetch：复制Leader日志，由Follower/Observer发送——可见，经典Raft协议中的AppendEntries消息是Leader将日志推给Follower，而KRaft协议中则是靠Fetch消息从Leader拉取日志。同时Fetch也可以作为Follower对Leader的活性探测。

根据KIP-595的说明，采用拉模式可以将一致性检查的工作放在Leader端，同时也可以更快速地bootstrap一个全新的Follower(直接从offset 0开始复制日志即可)以及淘汰过期的Follower。拉模式的缺点主要是处理僵尸Leader和Fetch的延迟可能较大。

### **领导选举**

当满足以下三个条件之一时，Quorum中的某个节点就会触发选举：

1. 向Leader发送Fetch请求后，在超时阈值quorum.fetch.timeout.ms之后仍然没有得到Fetch响应，表示Leader疑似失败；
2. 从当前Leader收到了EndQuorumEpoch请求，表示Leader已退位；
3. Candidate状态下，在超时阈值quorum.election.timeout.ms之后仍然没有收到多数票，也没有Candidate赢得选举，表示此次选举作废，重新进行选举。

接下来的投票流程与经典Raft协议相同，不再赘述。当然，选举过程中仍然要对无效的选票进行处理，如集群ID或纪元值过期的选票。

## 分区leader选举
## **2.Leader Replica选举触发时机**

常见的有以下几种情况会触发Partition的Leader Replica选举：

1. **Leader Replica 失效：**当 Leader Replica 出现故障或者失去连接时，Kafka 会触发 Leader Replica 选举。
2. **Broker 宕机：**当 Leader Replica 所在的 Broker 节点发生故障或者宕机时，Kafka 也会触发 Leader Replica 选举。
3. **新增 Broker：**当集群中新增 Broker 节点时，Kafka 还会触发 Leader Replica 选举，以重新分配 Partition 的 Leader。
4. **新建分区：**当一个新的分区被创建时，需要选举一个 Leader Replica。
5. **ISR 列表数量减少：**当 Partition 的 ISR 列表数量减少时，可能会触发 Leader Replica 选举。当 ISR 列表中副本数量小于 **Replication Factor（副本因子）**时，为了保证数据的安全性，就会触发 Leader Replica 选举。
6. **手动触发：**通过 Kafka 管理工具（kafka-preferred-replica-election.sh），可以手动触发选举，以平衡负载或实现集群维护。

## **3. Leader Replica选举策略**

在 Kafka 集群中，常见的 Leader Replica 选举策略有以下三种：

1. ISR 选举策略：默认情况下，Kafka 只会从 ISR 集合的副本中选举出新的 Leader Replica，OSR 集合中的副本不具备参选资格。  
    
2. 首选副本选举策略（Preferred Replica Election）：首选副本选举策略也是 Kafka 默认的选举策略。在这种策略下，每个分区都有一个首选副本（Preferred Replica），通常是副本集合中的第一个副本。当触发选举时，控制器会优先选择该首选副本作为新的 Leader Replica，只有在首选副本不可用的情况下，才会考虑其他副本。  
    当然，也可以使用命令手动指定每个分区的首选副本：  
      
    \_bin/kafka-topics.sh --zookeeper localhost:2181 --topic my-topic-name --replica-assignment 0:1,1:2,2:0 --partitions 3_  
      
    意思是：my-topic-name有3个partition，partition0的首选副本是Broker1，partition1首选副本是Broker2，partition2的首选副本是Broker0。  
    
3. 不干净副本选举策略（Unclean Leader Election）：在某些情况下，ISR 选举策略可能会失败，例如当所有 ISR 副本都不可用时。在这种情况下，可以使用 Unclean Leader 选举策略。Unclean Leader 选举策略会从所有副本中（包含OSR集合）选择一个副本作为新的 Leader 副本，即使这个副本与当前 Leader 副本不同步。这种选举策略可能会导致数据丢失，因此只应在紧急情况下使用。  
    修改下面的配置，可以开启 Unclean Leader 选举策略，默认关闭。  
    unclean.leader.election.enable=true  
      
    

## **4. Leader Replica选举过程**

当Leader Replica宕机或失效时，就会触发 Leader Replica 选举，分为两个阶段，第一个阶段是候选人的提名和投票阶段，第二个阶段是Leader的确认阶段。具体过程如下：

1. 候选人提名和投票阶段  
    在Leader Replica失效时，ISR集合中所有Follower Replica都可以成为新的Leader Replica候选人。每个Follower Replica会在选举开始时向其他Follower Replica发送成为候选人的请求，并附带自己的元数据信息，包括自己的当前状态和Lag值。而Preferred replica优先成为候选人。  
    其他Follower Replica在收到候选人请求后，会根据请求中的元数据信息，计算每个候选人的Lag值，并将自己的选票投给Lag最小的候选人。如果多个候选人的Lag值相同，则随机选择一个候选人。  
    
2. Leader确认阶段  
    在第一阶段结束后，所有的Follower Replica会重新计算每位候选人的Lag值，并投票给Lag值最小的候选人。此时，选举的结果并不一定出现对候选人的全局共识。为了避免出现这种情况，Kafka中使用了ZooKeeper来实现分布式锁，确保只有一个候选人能够成为新的Leader Replica。  
    当ZooKeeper确认有一个候选人已经获得了分布式锁时，该候选人就成为了新的Leader Replica，并向所有的Follower Replica发送一个LeaderAndIsrRequest请求，更新Partition的元数据信息。其他Follower Replica接收到请求后，会更新自己的Partition元数据信息，将新的Leader Replica的ID添加到ISR列表中。

# innodb行锁如何实现
## innodb行锁实现方式

**InnoDB行锁是通过给索引上的索引项加锁来实现的，这一点MySQL与Oracle不同，后者是通过在数据块中对相应数据行加锁来实现的。** InnoDB这种行锁实现特点意味着：只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁！  
在实际应用中，要特别注意InnoDB行锁的这一特性，不然的话，可能导致大量的锁冲突，从而影响并发性能。下面通过一些实际例子来加以说明.

# kafka生产和消费数据时，产生的pageCache污染是什么
长尾消费者导致的pageCache，某个消费者非常慢，导致broker的脏页都刷新到磁盘了，结果还被读出来了，而且还加上了预读，一次性读出来很多只被 它用一次的页。但是这个问题的某种程度的减轻，kafka中已经分了冷热链表了，先判断offset是否在 热区，如果在热区就只在热区做二分。在冷区就只对冷区做二分。
# zab算法原理
选择机制中的概念

**1、Serverid：服务器ID**

比如有三台服务器，编号分别是1,2,3。

编号越大在选择算法中的权重越大。

**2、Zxid：数据ID**

服务器中存放的最大数据ID。【zxid实际上是一个64位的数字，高32位是epoch（时期; 纪元; 世; 新时代）用来标识leader是否发生改变，如果有新的leader产生出来，epoch会自增，低32位用来递增计数。】

值越大说明数据越新，在选举算法中数据越新权重越大。

**3、Epoch：逻辑时钟**

或者叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的。每投完一次票这个数据就会增加，然后与接收到的其它服务器返回的投票信息中的数值相比，根据不同的值做出不同的判断。

4、Server状态：选举状态

LOOKING，竞选状态。

FOLLOWING，随从状态，同步leader状态，参与投票。

OBSERVING，观察状态,同步leader状态，不参与投票。

LEADING，领导者状态。

zookeeper是如何保证事务的顺序一致性的（保证消息有序） 在整个消息广播中，Leader会将每一个事务请求转换成对应的 proposal 来进行广播，并且在广播 事务Proposal 之前，Leader服务器会首先为这个事务Proposal分配一个全局单递增的唯一ID，称之为事务ID（即zxid），**由于Zab协议需要保证每一个消息的严格的顺序关系，因此必须将每一个proposal按照其zxid的先后顺序进行排序和处理。**

**消息广播具体步骤**

**1）客户端发起一个写操作请求。**

**2）Leader 服务器将客户端的请求转化为事务 Proposal 提案，同时为每个 Proposal 分配一个全局的ID，即zxid。**

**3）Leader 服务器为每个 Follower 服务器分配一个单独的队列，然后将需要广播的 Proposal 依次放到队列中取，并且根据 FIFO 策略进行消息发送。**

**4）Follower 接收到 Proposal 后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后向 Leader 反馈一个 Ack 响应消息。**

**5）Leader 接收到超过半数以上 Follower 的 Ack 响应消息后，即认为消息发送成功，可以发送 commit 消息。**

**6）Leader 向所有 Follower 广播 commit 消息，同时自身也会完成事务提交。Follower 接收到 commit 消息后，会将上一条事务提交。**

zookeeper 采用 Zab 协议的核心，就是只要有一台服务器提交了 Proposal，就要确保所有的服务器最终都能正确提交 Proposal。这也是 CAP/BASE 实现最终一致性的一个体现。

**Leader 服务器与每一个 Follower 服务器之间都维护了一个单独的 FIFO 消息队列进行收发消息，使用队列消息可以做到异步解耦。Leader 和 Follower 之间只需要往队列中发消息即可。如果使用同步的方式会引起阻塞，性能要下降很多。**

接受投票消息就是判断epoch和zxidf最大值，如果比我大，就投票给大的那个。发送投票会先发给自己，然后搜集到大多数就成功。

Zab 协议崩溃恢复要求满足以下两个要求：

**1）确保已经被 Leader 提交的 Proposal 必须最终被所有的 Follower 服务器提交。**

**2）确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal。**

根据上述要求 Zab协议需要保证选举出来的Leader需要满足以下条件：

**1）新选举出来的 Leader 不能包含未提交的 Proposal 。即新选举的 Leader 必须都是已经提交了 Proposal 的 Follower 服务器节点。**

**2）新选举的 Leader 节点中含有最大的 zxid 。这样做的好处是可以避免 Leader 服务器检查 Proposal 的提交和丢弃工作。**
#### 发起投票的契机

1.  节点启动
2.  节点运行期间无法与 Leader 保持连接，
3.  Leader 失去一半以上节点的连接

# raft和zab异同点
## 1.2 leader选举的效率

Raft中的每个server在某个term轮次内只能投一次票，哪个candidate先请求投票谁就可能先获得投票，这样就可能造成split vote，即各个candidate都没有收到过半的投票，Raft通过candidate设置不同的超时时间，来快速解决这个问题，使得先超时的candidate（在其他人还未超时时）优先请求来获得过半投票

**ZooKeeper中的每个server，在某个electionEpoch轮次内，可以投多次票，只要遇到更大的票就更新，然后分发新的投票给所有人。**这种情况下不存在split vote现象，同时有利于选出含有更新更多的日志的server，但是选举时间理论上相对Raft要花费的多。

## 1.3 加入一个已经完成选举的集群

-   1.3.1 怎么发现已完成选举的leader？
-   1.3.2 加入过程是否阻塞整个请求？

1.3.1 怎么发现已完成选举的leader？

一个server启动后（该server本来就属于该集群的成员配置之一，所以这里不是新加机器），如何加入一个已经选举完成的集群

-   Raft：比较简单，该server启动后，会收到leader的AppendEntries RPC,这时就会从RPC中获取leader信息，识别到leader，即使该leader是一个老的leader，之后新leader仍然会发送AppendEntries RPC,这时就会接收到新的leader了（因为新leader的term比老leader的term大，所以会更新leader）
-   ZooKeeper：该server启动后，会向所有的server发送投票通知，这时候就会收到处于LOOKING、FOLLOWING状态的server的投票（这种状态下的投票指向的leader），则该server放弃自己的投票，判断上述投票是否过半，过半则可以确认该投票的内容就是新的leader。

1.3.2 加入过程是否阻塞整个请求？

这个其实还要看对日志的设计是否是连续的

-   如果是连续的，则leader中只需要保存每个follower上一次的同步位置，这样在同步的时候就会自动将之前欠缺的数据补上，不会阻塞整个请求过程目前Raft的日志是依靠index来实现连续的
-   如果不是连续的，则在确认follower和leader当前数据的差异的时候，是需要获取leader当前数据的读锁，禁止这个期间对数据的修改。差异确定完成之后，释放读锁，允许leader数据被修改，每一个修改记录都要被保存起来，最后一一应用到新加入的follower中。目前ZooKeeper的日志zxid并不是严格连续的，允许有空洞。
## 1.4 leader选举的触发

触发一般有如下2个时机

-   server刚开始启动的时候，触发leader选举
-   leader选举完成之后，检测到超时触发，谁来检测？

-   Raft：目前只是follower在检测。follower有一个选举时间，在该时间内如果未收到leader的心跳信息，则follower转变成candidate，自增term发起新一轮的投票，leader遇到新的term则自动转变成follower的状态
-   ZooKeeper：leader和follower都有各自的检测超时方式，leader是检测是否过半follower心跳回复了，follower检测leader是否发送心跳了。一旦leader检测失败，则leader进入LOOKING状态，其他follower过一段时间因收不到leader心跳也会进入LOOKING状态，从而出发新的leader选举。一旦follower检测失败了，则该follower进入LOOKING状态，此时leader和其他follower仍然保持良好，则该follower仍然是去学习上述leader的投票，而不是触发新一轮的leader选举
## 2.1 上一轮次的leader的残留的数据怎么处理？

首先看下上一轮次的leader在挂或者失去leader位置之前，会有哪些数据？

-   已过半复制的日志
-   未过半复制的日志

一个日志是否被过半复制，是否被提交，这些信息是由leader才能知晓的，那么下一个leader该如何来判定这些日志呢？

下面分别来看看Raft和ZooKeeper的处理策略：

-   Raft：对于之前term的过半或未过半复制的日志采取的是保守的策略，全部判定为未提交，只有当当前term的日志提交了，才会顺便将之前term的日志进行提交
-   ZooKeeper：采取激进的策略，对于所有过半还是未过半的日志都判定为提交，都将其应用到状态机中

Raft的保守策略更多是因为Raft在leader选举完成之后，没有同步更新过程来保持和leader一致（在可以对外处理请求之前的这一同步过程）。而ZooKeeper是有该过程的

这一点其实在上面已经说了，`ZAB`通过选择过程中，选取拥有数据最多的`Leader`，在选举完`Leader`之后，增加了一步**同步阶段**，在当选后，需要将自身所有的数据同步到大多数`Follower`之后，才能开始对外提供服务，通过有两种方式`DIFF`和`SNAP`

这里想要说下，很多资料都表明，为了使`Leader`同步的数据保证正确，`ZAB`保证：

- `Leader`已经`Commite`的消息，一定要同步到其他服务器中。
- `Leader`未`Commite`的消息，保证要丢弃。

让人纠结的是新的`Leader`怎么知道这条消息究竟有没有提交？如果这个消息在第一阶段完成后，还没来的及发送`commite`消息，就挂掉，那么集群中其他服务器都不知道这条消息已经提交。

比较有说服力的说话是，当新的`Leader`当选后，会检查当前是否大多数`Follower`都有这条消息，如果有，就提交，如果没有，就丢弃。

但是查看`Zookeeper`的源码，发现同步消息阶段并没有这一步，而是直接选择最大的`zxid`然后开始同步。其实这也能理解，因为一般这这个过程正好`Leader`挂掉，客户端便会收到异常，异常恢复之后，需要客户端自己检查当前操作是否已经完成，这样设计起来也更加简单。

## 2.2 怎么阻止上一轮次的leader假死的问题

这其实就和实现有密切的关系了。

-   Raft的copycat实现为：每个follower开通一个复制数据的RPC接口，谁都可以连接并调用该接口，所以Raft需要来阻止上一轮次的leader的调用。每一轮次都会有对应的轮次号，用来进行区分，Raft的轮次号就是term，一旦旧leader对follower发送请求，follower会发现当前请求term小于自己的term，则直接忽略掉该请求，自然就解决了旧leader的干扰问题
-   ZooKeeper：一旦server进入leader选举状态则该follower会关闭与leader之间的连接，所以旧leader就无法发送复制数据的请求到新的follower了，也就无法造成干扰了
## 3.3 如何保证顺序

具体顺序是什么？

这个就是先到达leader的请求，先被应用到状态机。这就需要看正常运行过程、异常出现过程都是怎么来保证顺序的

3.3.1 正常同步过程的顺序

-   Raft对请求先转换成entry，复制时，也是按照leader中log的顺序复制给follower的，对entry的提交是按index进行顺序提交的，是可以保证顺序的
-   ZooKeeper在提交议案的时候也是按顺序写入各个follower对应在leader中的队列，然后follower必然是按照顺序来接收到议案的，对于议案的过半提交也都是一个个来进行的

3.3.2 异常过程的顺序保证

如follower挂掉又重启的过程：

-   Raft：重启之后，由于leader的AppendEntries RPC调用，识别到leader，leader仍然会按照leader的log进行顺序复制，也不用关心在复制期间新的添加的日志，在下一次同步中自动会同步
-   ZooKeeper：重启之后，需要和当前leader数据之间进行差异的确定，同时期间又有新的请求到来，所以需要暂时获取leader数据的读锁，禁止此期间的数据更改，先将差异的数据先放入队列，差异确定完毕之后，还需要将leader中已提交的议案和未提交的议案也全部放入队列，即ZooKeeper的如下2个集合数据然后再释放读锁，允许leader进行处理写数据的请求，该请求自然就添加在了上述队列的后面，从而保证了队列中的数据是有序的，从而保证发给follower的数据是有序的，follower也是一个个进行确认的，所以对于leader的回复也是有序的

-   ConcurrentMap<Long, Proposal> outstandingProposalsLeader拥有的属性，每当提出一个议案，都会将该议案存放至outstandingProposals，一旦议案被过半认同了，就要提交该议案，则从outstandingProposals中删除该议案
-   ConcurrentLinkedQueue<\Proposal> toBeAppliedLeader拥有的属性，每当准备提交一个议案，就会将该议案存放至该列表中，一旦议案应用到ZooKeeper的内存树中了，然后就可以将该议案从toBeApplied中删除

如果是leader挂了之后，重新选举出leader，会不会有乱序的问题？

-   Raft：Raft对于之前term的entry被过半复制暂不提交，只有当本term的数据提交了才能将之前term的数据一起提交，也是能保证顺序的
-   ZooKeeper:ZooKeeper每次leader选举之后都会进行数据同步，不会有乱序问题


# iface 和 eface 的区别是什么

`iface` 和 `eface` 都是 Go 中描述接口的底层结构体，区别在于 `iface` 描述的接口包含方法，而 `eface` 则是不包含任何方法的空接口：`interface{}`。

从源码层面看一下：

```go
type iface struct {

	tab *itab
	
	data unsafe.Pointer

}

type itab struct {

	inter *interfacetype
	
	_type *_type
	
	link *itab
	
	hash uint32 // copy of _type.hash. Used for type switches.
	
	bad bool // type does not implement interface
	
	inhash bool // has this itab been added to hash?
	
	unused [2]byte
	
	fun [1]uintptr // variable sized

}
```

`iface` 内部维护两个指针，`tab` 指向一个 `itab` 实体， 它表示接口的类型以及赋给这个接口的实体类型。`data` 则指向接口具体的值，一般而言是一个指向堆内存的指针。

再来仔细看一下 `itab` 结构体：`_type` 字段描述了实体的类型，包括内存对齐方式，大小等；`inter` 字段则描述了接口的类型。`fun` 字段放置和接口方法对应的具体数据类型的方法地址，实现接口调用方法的动态分派，一般在每次给接口赋值发生转换时会更新此表，或者直接拿缓存的 itab。

这里只会列出实体类型和接口相关的方法，实体类型的其他方法并不会出现在这里。如果你学过 C++ 的话，这里可以类比虚函数的概念。

另外，你可能会觉得奇怪，为什么 `fun` 数组的大小为 1，要是接口定义了多个方法可怎么办？实际上，这里存储的是第一个方法的函数指针，如果有更多的方法，在它之后的内存空间里继续存储。从汇编角度来看，通过增加地址就能获取到这些函数指针，没什么影响。顺便提一句，这些方法是按照函数名称的字典序进行排列的。

再看一下 `interfacetype` 类型，它描述的是接口的类型：

```go
type interfacetype struct {

	typ _type
	
	pkgpath name
	
	mhdr []imethod

}
```

可以看到，它包装了 `_type` 类型，`_type` 实际上是描述 Go 语言中各种数据类型的结构体。我们注意到，这里还包含一个 `mhdr` 字段，表示接口所定义的函数列表， `pkgpath` 记录定义了接口的包名。

这里通过一张图来看下 `iface` 结构体的全貌：

![](https://user-images.githubusercontent.com/7698088/56564826-82527600-65e1-11e9-956d-d98a212bc863.png)

iface 结构体全景

接着来看一下 `eface` 的源码：

```go
type eface struct {

_type *_type

data unsafe.Pointer

}
```

相比 `iface`，`eface` 就比较简单了。只维护了一个 `_type` 字段，表示空接口所承载的具体的实体类型。`data` 描述了具体的值。

![](https://user-images.githubusercontent.com/7698088/56565105-318f4d00-65e2-11e9-96bd-4b2e192791dc.png)

eface 结构体全景

我们来看个例子：

```go
package main

​

import "fmt"

​

func main() {

x := 200

var any interface{} = x

fmt.Println(any)

​

g := Gopher{"Go"}

var c coder = g

fmt.Println(c)

}

​

type coder interface {

code()

debug()

}

​

type Gopher struct {

language string

}

​

func (p Gopher) code() {

fmt.Printf("I am coding %s language\n", p.language)

}

​

func (p Gopher) debug() {

fmt.Printf("I am debuging %s language\n", p.language)

}
```

执行命令，打印出汇编语言：

go tool compile -S ./src/main.go

可以看到，main 函数里调用了两个函数：

```go
func convT2E64(t *_type, elem unsafe.Pointer) (e eface)

func convT2I(tab *itab, elem unsafe.Pointer) (i iface)
```

上面两个函数的参数和 `iface` 及 `eface` 结构体的字段是可以联系起来的：两个函数都是将参数`组装`一下，形成最终的接口。

作为补充，我们最后再来看下 `_type` 结构体：

```go
type _type struct {

// 类型大小

size uintptr

ptrdata uintptr

// 类型的 hash 值

hash uint32

// 类型的 flag，和反射相关

tflag tflag

// 内存对齐相关

align uint8

fieldalign uint8

// 类型的编号，有bool, slice, struct 等等等等

kind uint8

alg *typeAlg

// gc 相关

gcdata *byte

str nameOff

ptrToThis typeOff

}

Go 语言各种数据类型都是在 `_type` 字段的基础上，增加一些额外的字段来进行管理的：

type arraytype struct {

typ _type

elem *_type

slice *_type

len uintptr

}

​

type chantype struct {

typ _type

elem *_type

dir uintptr

}

​

type slicetype struct {

typ _type

elem *_type

}

​

type structtype struct {

typ _type

pkgPath name

fields []structfield

}
```

这些数据类型的结构体定义，是反射实现的基础。

# golang鸭子类型
Go 语言作为一门现代静态语言，是有后发优势的。它引入了动态语言的便利，同时又会进行静态语言的类型检查，写起来是非常 Happy 的。Go 采用了折中的做法：不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。
顺带再提一下动态语言的特点：

> 变量绑定的类型是不确定的，在运行期间才能确定 函数和方法可以接收任何类型的参数，且调用时不检查参数类型 不需要实现接口

总结一下，鸭子类型是一种动态语言的风格，在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由它"当前方法和属性的集合"决定。Go 作为一种静态语言，通过接口实现了 `鸭子类型`，实际上是 Go 的编译器在其中作了隐匿的转换工作。

# context.Value 的查找过程是怎样的

```go
type valueCtx struct {

Context

key, val interface{}

}

它实现了两个方法：

func (c *valueCtx) String() string {

return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)

}

​

func (c *valueCtx) Value(key interface{}) interface{} {

if c.key == key {

return c.val

}

return c.Context.Value(key)

}
```

由于它直接将 Context 作为匿名字段，因此仅管它只实现了 2 个方法，其他方法继承自父 context。但它仍然是一个 Context，这是 Go 语言的一个特点。

创建 valueCtx 的函数：

```go
func WithValue(parent Context, key, val interface{}) Context {

if key == nil {

panic("nil key")

}

if !reflect.TypeOf(key).Comparable() {

panic("key is not comparable")

}

return &valueCtx{parent, key, val}

}
```

对 key 的要求是可比较，因为之后需要通过 key 取出 context 中的值，可比较是必须的。

通过层层传递 context，最终形成这样一棵树：

![](https://user-images.githubusercontent.com/7698088/59154893-5e72c300-8aaf-11e9-9b78-3c34b5e73a45.png)

valueCtx

和链表有点像，只是它的方向相反：Context 指向它的父节点，链表则指向下一个节点。通过 WithValue 函数，可以创建层层的 valueCtx，存储 goroutine 间可以共享的变量。

取值的过程，实际上是一个递归查找的过程：

```go
func (c *valueCtx) Value(key interface{}) interface{} {

if c.key == key {

return c.val

}

return c.Context.Value(key)

}
```

它会顺着链路一直往上找，比较当前节点的 key 是否是要找的 key，如果是，则直接返回 value。否则，一直顺着 context 往前，最终找到根节点（一般是 emptyCtx），直接返回一个 nil。所以用 Value 方法的时候要判断结果是否为 nil。

因为查找方向是往上走的，所以，父节点没法获取子节点存储的值，子节点却可以获取父节点的值。

`WithValue` 创建 context 节点的过程实际上就是创建链表节点的过程。两个节点的 key 值是可以相等的，但它们是两个不同的 context 节点。查找的时候，会向上查找到最后一个挂载的 context 节点，也就是离得比较近的一个父节点 context。所以，整体上而言，用 `WithValue` 构造的其实是一个低效率的链表。

如果你接手过项目，肯定经历过这样的窘境：在一个处理过程中，有若干子函数、子协程。各种不同的地方会向 context 里塞入各种不同的 k-v 对，最后在某个地方使用。

你根本就不知道什么时候什么地方传了什么值？这些值会不会被“覆盖”（底层是两个不同的 context 节点，查找的时候，只会返回一个结果）？你肯定会崩溃的。

而这也是 `context.Value` 最受争议的地方。很多人建议尽量不要通过 context 传值。

# Context如何被取消
`Context` 是一个接口，定义了 4 个方法，它们都是`幂等`的。也就是说连续多次调用同一个方法，得到的结果都是相同的。

`Done()` 返回一个 channel，可以表示 context 被取消的信号：当这个 channel 被关闭时，说明 context 被取消了。注意，这是一个只读的channel。 我们又知道，读一个关闭的 channel 会读出相应类型的零值。并且源码里没有地方会向这个 channel 里面塞入值。换句话说，这是一个 `receive-only` 的 channel。因此在子协程里读这个 channel，除非被关闭，否则读不出来任何东西。也正是利用了这一点，子协程从 channel 里读出了值（零值）后，就可以做一些收尾工作，尽快退出。

`Err()` 返回一个错误，表示 channel 被关闭的原因。例如是被取消，还是超时。

`Deadline()` 返回 context 的截止时间，通过此时间，函数就可以决定是否进行接下来的操作，如果时间太短，就可以不往下做了，否则浪费系统资源。当然，也可以用这个 deadline 来设置一个 I/O 操作的超时时间。

`Value()` 获取之前设置的 key 对应的 value。
### canceler[](#canceler)

再来看另外一个接口：

type canceler interface {

cancel(removeFromParent bool, err error)

Done() <-chan struct{}

}

实现了上面定义的两个方法的 Context，就表明该 Context 是可取消的。源码中有两个类型实现了 canceler 接口：`*cancelCtx` 和 `*timerCtx`。注意是加了 `*` 号的，是这两个结构体的指针实现了 canceler 接口。

Context 接口设计成这个样子的原因：

- “取消”操作应该是建议性，而非强制性
    

caller 不应该去关心、干涉 callee 的情况，决定如何以及何时 return 是 callee 的责任。caller 只需发送“取消”信息，callee 根据收到的信息来做进一步的决策，因此接口并没有定义 cancel 方法。

- “取消”操作应该可传递
    

“取消”某个函数时，和它相关联的其他函数也应该“取消”。因此，`Done()` 方法返回一个只读的 channel，所有相关函数监听此 channel。一旦 channel 关闭，通过 channel 的“广播机制”，所有监听者都能收到。
# context 有什么作用

Go 常用来写后台服务，通常只需要几行代码，就可以搭建一个 http server。

在 Go 的 server 里，通常每来一个请求都会启动若干个 goroutine 同时工作：有些去数据库拿数据，有些调用下游接口获取相关数据……

![](https://user-images.githubusercontent.com/7698088/59235934-643ee480-8c26-11e9-8931-456333900657.png)

request

这些 goroutine 需要共享这个请求的基本数据，例如登陆的 token，处理请求的最大超时时间（如果超过此值再返回数据，请求方因为超时接收不到）等等。当请求被取消或是处理时间太长，这有可能是使用者关闭了浏览器或是已经超过了请求方规定的超时时间，请求方直接放弃了这次请求结果。这时，所有正在为这个请求工作的 goroutine 需要快速退出，因为它们的“工作成果”不再被需要了。在相关联的 goroutine 都退出后，系统就可以回收相关的资源。

再多说一点，Go 语言中的 server 实际上是一个“协程模型”，也就是说一个协程处理一个请求。例如在业务的高峰期，某个下游服务的响应变慢，而当前系统的请求又没有超时控制，或者超时时间设置地过大，那么等待下游服务返回数据的协程就会越来越多。而我们知道，协程是要消耗系统资源的，后果就是协程数激增，内存占用飙涨，甚至导致服务不可用。更严重的会导致雪崩效应，整个服务对外表现为不可用，这肯定是 P0 级别的事故。这时，肯定有人要背锅了。

其实前面描述的 P0 级别事故，通过设置“允许下游最长处理时间”就可以避免。例如，给下游设置的 timeout 是 50 ms，如果超过这个值还没有接收到返回数据，就直接向客户端返回一个默认值或者错误。例如，返回商品的一个默认库存数量。注意，这里设置的超时时间和创建一个 http client 设置的读写超时时间不一样，这里不详细展开。可以去看看参考资料`【Go 在今日头条的实践】`一文，有很精彩的论述。

context 包就是为了解决上面所说的这些问题而开发的：在 一组 goroutine 之间传递共享的值、取消信号、deadline……

![](https://user-images.githubusercontent.com/7698088/59235969-a405cc00-8c26-11e9-9448-2c6c86e8263b.png)

request with context

用简练一些的话来说，在Go 里，我们不能直接杀死协程，协程的关闭一般会用 `channel+select` 方式来控制。但是在某些场景下，例如处理一个请求衍生了很多协程，这些协程之间是相互关联的：需要共享一些全局变量、有共同的 deadline 等，而且可以同时被关闭。再用 `channel+select` 就会比较麻烦，这时就可以通过 context 来实现。

一句话：context 用来解决 goroutine 之间`退出通知`、`元数据传递`的功能。

【引申1】举例说明 context 在实际项目中如何使用。

context 使用起来非常方便。源码里对外提供了一个创建根节点 context 的函数：

func Background() Context

background 是一个空的 context， 它不能被取消，没有值，也没有超时时间。

有了根节点 context，又提供了四个函数创建子节点 context：

func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context

context 会在函数传递间传递。只需要在适当的时间调用 cancel 函数向 goroutines 发出取消信号或者调用 Value 函数取出 context 中的值。

在官方博客里，对于使用 context 提出了几点建议：

- 1.
    
    Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx.
    

- 2.
    
    Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use.
    

- 3.
    
    Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
    

- 4.
    
    The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.
    

我翻译一下：

- 1.
    
    不要将 Context 塞到结构体里。直接将 Context 类型作为函数的第一参数，而且一般都命名为 ctx。
    

- 2.
    
    不要向函数传入一个 nil 的 context，如果你实在不知道传什么，标准库给你准备好了一个 context：todo。
    

- 3.
    
    不要把本应该作为函数参数的类型塞到 context 中，context 存储的应该是一些共同的数据。例如：登陆的 session、cookie 等。
    

- 4.
    
    同一个 context 可能会被传递到多个 goroutine，别担心，context 是并发安全的。
    

## 传递共享的数据
对于 Web 服务端开发，往往希望将一个请求处理的整个过程串起来，这就非常依赖于 Thread Local（对于 Go 可理解为单个协程所独有） 的变量，而在 Go 语言中并没有这个概念，因此需要在函数调用的时候传递 context。

```
package main

import (

"context"

"fmt"

)

​

func main() {

ctx := context.Background()

process(ctx)

​

ctx = context.WithValue(ctx, "traceId", "qcrao-2019")

process(ctx)

}

​

func process(ctx context.Context) {

traceId, ok := ctx.Value("traceId").(string)

if ok {

fmt.Printf("process over. trace_id=%s\n", traceId)

} else {

fmt.Printf("process over. no trace_id\n")

}

}
```

运行结果：

process over. no trace_id

process over. trace_id=qcrao-2019

第一次调用 process 函数时，ctx 是一个空的 context，自然取不出来 traceId。第二次，通过 `WithValue` 函数创建了一个 context，并赋上了 `traceId` 这个 key，自然就能取出来传入的 value 值。

当然，现实场景中可能是从一个 HTTP 请求中获取到的 Request-ID。所以，下面这个样例可能更适合：

const requestIDKey int = 0

​

func WithRequestID(next http.Handler) http.Handler {

return http.HandlerFunc(

func(rw http.ResponseWriter, req *http.Request) {

// 从 header 中提取 request-id

reqID := req.Header.Get("X-Request-ID")

// 创建 valueCtx。使用自定义的类型，不容易冲突

ctx := context.WithValue(

req.Context(), requestIDKey, reqID)

​

// 创建新的请求

req = req.WithContext(ctx)

​

// 调用 HTTP 处理函数

next.ServeHTTP(rw, req)

}

)

}

​

// 获取 request-id

func GetRequestID(ctx context.Context) string {

ctx.Value(requestIDKey).(string)

}

​

func Handle(rw http.ResponseWriter, req *http.Request) {

// 拿到 reqId，后面可以记录日志等等

reqID := GetRequestID(req.Context())

...

}

​

func main() {

handler := WithRequestID(http.HandlerFunc(Handle))

http.ListenAndServe("/", handler)

}

## 取消 goroutine[](https://qcrao91.gitbook.io/go/biao-zhun-ku/context/context-you-shi-mo-zuo-yong#qu-xiao-goroutine)

我们先来设想一个场景：打开外卖的订单页，地图上显示外卖小哥的位置，而且是每秒更新 1 次。app 端向后台发起 websocket 连接（现实中可能是轮询）请求后，后台启动一个协程，每隔 1 秒计算 1 次小哥的位置，并发送给端。如果用户退出此页面，则后台需要“取消”此过程，退出 goroutine，系统回收资源。

后端可能的实现如下：

func Perform() {

for {

calculatePos()

sendResult()

time.Sleep(time.Second)

}

}

如果需要实现“取消”功能，并且在不了解 context 功能的前提下，可能会这样做：给函数增加一个指针型的 bool 变量，在 for 语句的开始处判断 bool 变量是发由 true 变为 false，如果改变，则退出循环。

上面给出的简单做法，可以实现想要的效果，没有问题，但是并不优雅，并且一旦协程数量多了之后，并且各种嵌套，就会很麻烦。优雅的做法，自然就要用到 context。

func Perform(ctx context.Context) {

for {

calculatePos()

sendResult()

​

select {

case <-ctx.Done():

// 被取消，直接返回

return

case <-time.After(time.Second):

// block 1 秒钟

}

}

}

主流程可能是这样的：

ctx, cancel := context.WithTimeout(context.Background(), time.Hour)

go Perform(ctx)

​

// ……

// app 端返回页面，调用cancel 函数

cancel()

注意一个细节，WithTimeOut 函数返回的 context 和 cancelFun 是分开的。context 本身并没有取消函数，这样做的原因是取消函数只能由外层函数调用，防止子节点 context 调用取消函数，从而严格控制信息的流向：由父节点 context 流向子节点 context。

## 

防止 goroutine 泄漏[](https://qcrao91.gitbook.io/go/biao-zhun-ku/context/context-you-shi-mo-zuo-yong#fang-zhi-goroutine-xie-lou)

前面那个例子里，goroutine 还是会自己执行完，最后返回，只不过会多浪费一些系统资源。这里改编一个“如果不用 context 取消，goroutine 就会泄漏的例子”，来自参考资料：`【避免协程泄漏】`。

func gen() <-chan int {

ch := make(chan int)

go func() {

var n int

for {

ch <- n

n++

time.Sleep(time.Second)

}

}()

return ch

}

这是一个可以生成无限整数的协程，但如果我只需要它产生的前 5 个数，那么就会发生 goroutine 泄漏：

func main() {

for n := range gen() {

fmt.Println(n)

if n == 5 {

break

}

}

// ……

}

当 n == 5 的时候，直接 break 掉。那么 gen 函数的协程就会执行无限循环，永远不会停下来。发生了 goroutine 泄漏。

用 context 改进这个例子：

func gen(ctx context.Context) <-chan int {

ch := make(chan int)

go func() {

var n int

for {

select {

case <-ctx.Done():

return

case ch <- n:

n++

time.Sleep(time.Second)

}

}

}()

return ch

}

​

func main() {

ctx, cancel := context.WithCancel(context.Background())

defer cancel() // 避免其他地方忘记 cancel，且重复调用不影响

​

for n := range gen(ctx) {

fmt.Println(n)

if n == 5 {

cancel()

break

}

}

// ……

}

增加一个 context，在 break 前调用 cancel 函数，取消 goroutine。gen 函数在接收到取消信号后，直接退出，系统回收资源。


# 如何利用unsafe包修改私有成员
这样在 main 函数中，它的三个字段都是私有成员变量，不能直接修改。但我通过 unsafe.Sizeof() 函数可以获取成员大小，进而计算出成员的地址，直接修改内存。

# Go指针和unsafe.Pointer有什么区别

Go 语言的作者之一 Ken Thompson 也是 C 语言的作者。所以，Go 可以看作 C 系语言，它的很多特性都和 C 类似，指针就是其中之一。

然而，Go 语言的指针相比 C 的指针有很多限制。这当然是为了安全考虑，要知道像 Java/Python 这些现代语言，生怕程序员出错，哪有什么指针（这里指的是显式的指针）？更别说像 C/C++ 还需要程序员自己清理“垃圾”。所以对于 Go 来说，有指针已经很不错了，仅管它有很多限制。

相比于 C 语言中指针的灵活，Go 的指针多了一些限制。但这也算是 Go 的成功之处：既可以享受指针带来的便利，又避免了指针的危险性。

限制一：`Go 的指针不能进行数学运算`。

来看一个简单的例子：

a := 5

p := &a

​

p++

p = &a + 3

上面的代码将不能通过编译，会报编译错误：`invalid operation`，也就是说不能对指针做数学运算。

限制二：`不同类型的指针不能相互转换`。

例如下面这个简短的例子：

```
func main() {

a := int(100)

var f *float64

​

f = &a

}
```

也会报编译错误：

cannot use &a (type *int) as type *float64 in assignment

限制三：`不同类型的指针不能使用 == 或 != 比较`。

只有在两个指针类型相同或者可以相互转换的情况下，才可以对两者进行比较。另外，指针可以通过 `==` 和 `!=` 直接和 `nil` 作比较。

限制四：`不同类型的指针变量不能相互赋值`。

这一点同限制三。

unsafe.Pointer 在 unsafe 包：

```
type ArbitraryType int

type Pointer *ArbitraryType
```

从命名来看，`Arbitrary` 是任意的意思，也就是说 Pointer 可以指向任意类型，实际上它类似于 C 语言里的 `void*`。

unsafe 包提供了 2 点重要的能力：

> - 1.
>     
>     任何类型的指针和 unsafe.Pointer 可以相互转换。
>     
> 
> - 2.
>     
>     uintptr 类型和 unsafe.Pointer 可以相互转换。
>     

![](https://user-images.githubusercontent.com/7698088/58747453-1dbaee80-849e-11e9-8c75-2459f76792d2.png)

type pointer uintptr

pointer 不能直接进行数学运算，但可以把它转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型。

// uintptr 是一个整数类型，它足够大，可以存储

type uintptr uintptr

还有一点要注意的是，uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收。而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。

unsafe 包中的几个函数都是在编译期间执行完毕，毕竟，编译器对内存分配这些操作“了然于胸”。在 `/usr/local/go/src/cmd/compile/internal/gc/unsafe.go` 路径下，可以看到编译期间 Go 对 unsafe 包中函数的处理。

# 如何实现字符串和byte切片的零拷贝转换

这是一个非常精典的例子。实现字符串和 bytes 切片之间的转换，要求是 `zero-copy`。想一下，一般的做法，都需要遍历字符串或 bytes 切片，再挨个赋值。

完成这个任务，我们需要了解 slice 和 string 的底层数据结构：

```
type StringHeader struct {

Data uintptr

Len int

}

​

type SliceHeader struct {

Data uintptr

Len int

Cap int

}

上面是反射包下的结构体，路径：src/reflect/value.go。只需要共享底层 Data 和 Len 就可以实现 `zero-copy`。

func string2bytes(s string) []byte {

return *(*[]byte)(unsafe.Pointer(&s))

}

func bytes2string(b []byte) string{

return *(*string)(unsafe.Pointer(&b))

}
```

原理上是利用指针的强转，代码比较简单，不作详细解释。

# 如何比较两个对象完全相同

Go 语言中提供了一个函数可以完成此项功能：

func DeepEqual(x, y interface{}) bool

`DeepEqual` 函数的参数是两个 `interface`，实际上也就是可以输入任意类型，输出 true 或者 flase 表示输入的两个变量是否是“深度”相等。

先明白一点，如果是不同的类型，即使是底层类型相同，相应的值也相同，那么两者也不是“深度”相等。

type MyInt int

type YourInt int

​

func main() {

m := MyInt(1)

y := YourInt(1)

​

fmt.Println(reflect.DeepEqual(m, y)) // false

}

上面的代码中，m, y 底层都是 int，而且值都是 1，但是两者静态类型不同，前者是 `MyInt`，后者是 `YourInt`，因此两者不是“深度”相等。

在源码里，有对 DeepEqual 函数的非常清楚地注释，列举了不同类型，DeepEqual 的比较情形，这里做一个总结：

|   |   |
|---|---|
|类型|深度相等情形|
|Array|相同索引处的元素“深度”相等|
|Struct|相应字段，包含导出和不导出，“深度”相等|
|Func|只有两者都是 nil 时|
|Interface|两者存储的具体值“深度”相等|
|Map|1、都为 nil；2、非空、长度相等，指向同一个 map 实体对象，或者相应的 key 指向的 value “深度”相等|
|Pointer|1、使用 == 比较的结果相等；2、指向的实体“深度”相等|
|Slice|1、都为 nil；2、非空、长度相等，首元素指向同一个底层数组的相同元素，即 &x[0] == &y[0] 或者 相同索引处的元素“深度”相等|
|numbers, bools, strings, and channels|使用 == 比较的结果为真|

一般情况下，DeepEqual 的实现只需要递归地调用 == 就可以比较两个变量是否是真的“深度”相等。

但是，有一些异常情况：比如 func 类型是不可比较的类型，只有在两个 func 类型都是 nil 的情况下，才是“深度”相等；float 类型，由于精度的原因，也是不能使用 == 比较的；包含 func 类型或者 float 类型的 struct， interface， array 等。

对于指针而言，当两个值相等的指针就是“深度”相等，因为两者指向的内容是相等的，即使两者指向的是 func 类型或者 float 类型，这种情况下不关心指针所指向的内容。

同样，对于指向相同 slice， map 的两个变量也是“深度”相等的，不关心 slice， map 具体的内容。

对于“有环”的类型，比如循环链表，比较两者是否“深度”相等的过程中，需要对已比较的内容作一个标记，一旦发现两个指针之前比较过，立即停止比较，并判定二者是深度相等的。这样做的原因是，及时停止比较，避免陷入无限循环。

来看源码：

func DeepEqual(x, y interface{}) bool {

if x == nil || y == nil {

return x == y

}

v1 := ValueOf(x)

v2 := ValueOf(y)

if v1.Type() != v2.Type() {

return false

}

return deepValueEqual(v1, v2, make(map[visit]bool), 0)

}

首先查看两者是否有一个是 nil 的情况，这种情况下，只有两者都是 nil，函数才会返回 true

接着，使用反射，获取x，y 的反射对象，并且立即比较两者的类型，根据前面的内容，这里实际上是动态类型，如果类型不同，直接返回 false。

最后，最核心的内容在子函数 `deepValueEqual` 中。

代码比较长，思路却比较简单清晰：核心是一个 switch 语句，识别输入参数的不同类型，分别递归调用 deepValueEqual 函数，一直递归到最基本的数据类型，比较 int，string 等可以直接得出 true 或者 false，再一层层地返回，最终得到“深度”相等的比较结果。

实际上，各种类型的比较套路比较相似，这里就直接节选一个稍微复杂一点的 `map` 类型的比较：

// deepValueEqual 函数

// ……

​

case Map:

if v1.IsNil() != v2.IsNil() {

return false

}

if v1.Len() != v2.Len() {

return false

}

if v1.Pointer() == v2.Pointer() {

return true

}

for _, k := range v1.MapKeys() {

val1 := v1.MapIndex(k)

val2 := v2.MapIndex(k)

if !val1.IsValid() || !val2.IsValid() || !deepValueEqual(v1.MapIndex(k), v2.MapIndex(k), visited, depth+1) {

return false

}

}

return true

​

// ……

和前文总结的表格里，比较 map 是否相等的思路比较一致，也不需要多说什么。说明一点，`visited` 是一个 map，记录递归过程中，比较过的“对”：

type visit struct {

a1 unsafe.Pointer

a2 unsafe.Pointer

typ Type

}

​

map[visit]bool

比较过程中，一旦发现比较的“对”，已经在 map 里出现过的话，直接判定“深度”比较结果的是 `true`。


## 场景题：如果我有100G文件，但是只有 500 M 的内存，这些文件存着一行行的数字，如何获取最小的10个？

这是经典的 Top K 问题，分治+堆排，我们可以先进行一个将文件进行分治，将100G的文件的分成一个个的 500 M，利用hash路由来分配，以此放到内存中每个500M的文件取出进行最小堆的排序，取出前10个写到新文件中。再对着一个个的新文件进行合并成 500 M 再进行最小堆的排序，最终获取最小的10个数字。


# http2的队头阻塞
HTTP/2 并没有解决 TCP 的队首阻塞问题，它仅仅是通过**多路复用**解决了以前 HTTP1.1 **管线化**请求时的队首阻塞。

比如 HTTP/1.1 时代建立一个 TCP 连接，三个请求组成一个队列发出去，服务器接收到这个队列之后会依次响应，一旦前面的请求阻塞，后面的请求就会无法响应。

HTTP/2 是通过**分帧**并且给每个帧打上**流**的 ID 去避免依次响应的问题，对方接收到帧之后根据 ID 拼接出流，这样就可以做到乱序响应从而避免请求时的队首阻塞问题。但是 TCP 层面的队首阻塞是 HTTP/2 无法解决的（HTTP 只是应用层协议，TCP 是传输层协议），TCP 的阻塞问题是因为传输阶段可能会丢包，一旦丢包就会等待重新发包，阻塞后续传输，这个问题虽然有**滑动窗口（Sliding Window）**这个方案，但是只能增强抗干扰，并没有彻底解决。

# 最近一个小时内访问频率最高的10个IP
实时输出最近一个小时内访问频率最高的10个IP，要求：

- 实时输出
- 从当前时间向前数的1个小时
- QPS可能会达到10W/s

这道题乍一看很像[Top K 频繁项](https://www.bookstack.cn/read/system-design/cn-bigdata-heavy-hitters.md)，是不是需要 Lossy Count 或 Count-Min Sketch 之类的算法呢？

其实杀鸡焉用牛刀，这道题用不着上述算法，请听我仔细分析。

1. QPS是 10万/秒，即一秒内最高有 10万个请求，那么一个小时内就有 100000\*3600=360000000≈$$2^{28.4}$$，向上取整，大概是 $$2^{29}$$个请求，也不是很大。我们在内存中建立3600个`HashMap<Int,Int>`，放在一个数组里，每秒对应一个HashMap，IP地址为key, 出现次数作为value。这样，一个小时内最多有$$2^{29}$$个pair，每个pair占8字节，总内存大概是 $$2^{29} \times 8=2^{32}$$字节，即4GB，单机完全可以存下。
2. 同时还要新建一个固定大小为10的小根堆，用于存放当前出现次数最大的10个IP。堆顶是10个IP里频率最小的IP。
3. 每次来一个请求，就把该秒对应的HashMap里对应的IP计数器增1，并查询该IP是否已经在堆中存在，
    
    - 如果不存在，则把该IP在3600个HashMap的计数器加起来，与堆顶IP的出现次数进行比较，如果大于堆顶元素，则替换掉堆顶元素，如果小于，则什么也不做
    - 如果已经存在，则把堆中该IP的计数器也增1，并调整堆
4. 需要有一个后台常驻线程，每过一秒，把最旧的那个HashMap销毁，并为当前这一秒新建一个HashMap，这样维持一个一小时的窗口。
    
5. 每次查询top 10的IP地址时，把堆里10个IP地址返回来即可。

以上就是该方案的全部内容。

有的人问，可不可以用”IP + 时间”作为key, 把所有pair放在单个HashMap里？如果把所有数据放在一个HashMap里，有两个巨大的缺点，

- 第4步里，怎么淘汰掉一个小时前的pair呢？这时候后台线程只能每隔一秒，全量扫描这个HashMap里的所有pair,把过期数据删除，这是线性时间复杂度，很慢。
- 这时候HashMap里的key存放的是”IP + 时间”组合成的字符串，占用内存远远大于一个int。而前面的方案，不用存真正的时间，只需要开一个3600长度的数组来表示一个小时时间窗口。

# 基数估计
如何计算数据流中不同元素的个数？例如，独立访客(Unique Visitor，简称UV)统计。这个问题称为基数估计(Cardinality Estimation)，也是一个很经典的题目。

### 方案1: HashSet

首先最容易想到的办法是用HashSet，每来一个元素，就往里面塞，HashSet的大小就所求答案。但是在大数据的场景下，HashSet在单机内存中存不下。

### 方案2: bitmap

HashSet耗内存主要是由于存储了元素的真实值，可不可以不存储元素本身呢？bitmap就是这样一个方案，假设已经知道不同元素的个数的上限，即基数的最大值，设为N，则开一个长度为N的bit数组，地位跟HashSet一样。每个元素与bit数组的某一位一一对应，该位为1，表示此元素在集合中，为0表示不在集合中。那么bitmap中1的个数就是所求答案。

这个方案的缺点是，bitmap的长度与实际的基数无关，而是与基数的上限有关。假如要计算上限为1亿的基数，则需要12.5MB的bitmap，十个网站就需要125M。关键在于，这个内存使用与集合元素数量无关，即使一个网站仅仅有一个1UV，也要为其分配12.5MB内存。该算法的空间复杂度是O(N_{max})。

实际上目前还没有发现在大数据场景中准确计算基数的高效算法，因此在不追求绝对准确的情况下，使用近似算法算是一个不错的解决方案。

### 方案3: Linear Counting

Linear Counting的基本思路是：

- 选择一个哈希函数h，其结果服从均匀分布
- 开一个长度为m的bitmap，均初始化为0(m设为多大后面有讨论)
- 数据流每来一个元素，计算其哈希值并对m取模，然后将该位置为1
- 查询时，设bitmap中还有u个bit为0，则不同元素的总数近似为 $$-m\log\dfrac{u}{m}$$

在使用Linear Counting算法时，主要需要考虑的是bitmap长度`m`。m主要由两个因素决定，基数大小以及容许的误差。假设基数大约为n，允许的误差为ϵ，则m需要满足如下约束，

m > \dfrac{\epsilon^t-t-1}{(\epsilon t)^2}, 其中 t=\dfrac{n}{m}

精度越高，需要的m越大。

Linear Counting 与方案1中的bitmap很类似，只是改善了bitmap的内存占用，但并没有完全解决，例如一个网站只有一个UV，依然要为其分配m位的bitmap。该算法的空间复杂度与方案2一样，依然是O(N_{max})。

### 方案4: LogLog Counting

LogLog Counting的算法流程：

1. 均匀随机化。选取一个哈希函数h应用于所有元素，然后对哈希后的值进行基数估计。哈希函数h必须满足如下条件，
    
    1. 哈希碰撞可以忽略不计。哈希函数h要尽可能的减少冲突
    2. h的结果是均匀分布的。也就是说无论原始数据的分布如何，其哈希后的结果几乎服从均匀分布（完全服从均匀分布是不可能的，D. Knuth已经证明不可能通过一个哈希函数将一组不服从均匀分布的数据映射为绝对均匀分布，但是很多哈希函数可以生成几乎服从均匀分布的结果，这里我们忽略这种理论上的差异，认为哈希结果就是服从均匀分布）。
    3. 哈希后的结果是固定长度的
2. 对于元素计算出哈希值，由于每个哈希值是等长的，令长度为L
    
3. 对每个哈希值，从高到低找到第一个1出现的位置，取这个位置的最大值，设为p，则基数约等于$$2^p$$

如果直接使用上面的单一估计量进行基数估计会由于偶然性而存在较大误差。因此，LLC采用了分桶平均的思想来降低误差。具体来说，就是将哈希空间平均分成m份，每份称之为一个桶（bucket）。对于每一个元素，其哈希值的前k比特作为桶编号，其中2^k=m，而后L-k个比特作为真正用于基数估计的比特串。桶编号相同的元素被分配到同一个桶，在进行基数估计时，首先计算每个桶内最大的第一个1的位置，设为`p[i]`，然后对这m个值取平均后再进行估计，即基数的估计值为2^{\frac{1}{m}\Sigma_{i=0}^{m-1} p[i]}。这相当于做多次实验然后去平均值，可以有效降低因偶然因素带来的误差。

LogLog Counting 的空间复杂度仅有O(\log_2(\log_2(N_{max})))，内存占用极少，这是它的优点。不过LLC也有自己的缺点，当基数不是很大的时候，误差比较大。

关于该算法的数学证明，请阅读原始论文和参考资料里的链接，这里不再赘述。

### 方案5: HyperLogLog Counting

HyperLogLog Counting（以下简称HLLC）的基本思想是在LLC的基础上做改进，

- 第1个改进是使用调和平均数替代几何平均数，调和平均数可以有效抵抗离群值的扰。注意LLC是对各个桶取算术平均数，而算术平均数最终被应用到2的指数上，所以总体来看LLC取的是几何平均数。由于几何平均数对于离群值（例如0）特别敏感，因此当存在离群值时，LLC的偏差就会很大，这也从另一个角度解释了为什么基数n不太大时LLC的效果不太好。这是因为n较小时，可能存在较多空桶，这些特殊的离群值干扰了几何平均数的稳定性。使用调和平均数后，估计公式变为 $$\hat{n}=\frac{\alpha_mm^2}{\Sigma_{i=0}^{m-1} p[i]}$$，其中$$\alpha_m=(m\int_0^{\infty}(log_2(\frac{2+u}{1+u}))^mdu)^{-1}$$
- 第2个改进是加入了分段偏差修正。具体来说，设e为基数的估计值，
    
    - 当 $$e \leq \frac{5}{2}m$$时，使用 Linear Counting
    - 当 $$\frac{5}{2}m<e\leq \frac{1}{30}2^{32}$$时，使用 HyperLogLog Counting
    - 当 $$e>\frac{1}{30}2^{32}$$时，修改估计公式为$$\hat{n}=-2^{32}\log(1-e/2^{32})$$

关于分段偏差修正的效果分析也可以在原论文中找到。

# 频率估计
如何计算数据流中任意元素的频率？

这个问题也是大数据场景下的一个经典问题，称为频率估计(Frequency Estimation)问题。

### 方案1: HashMap

用一个HashMap记录每个元素的出现次数，每来一个元素，就把相应的计数器增1。这个方法在大数据的场景下不可行，因为元素太多，单机内存无法存下这个巨大的HashMap。

### 方案2: 数据分片 + HashMap

既然单机内存存不下所有元素，一个很自然的改进就是使用多台机器。假设有8台机器，每台机器都有一个HashMap，第1台机器只处理`hash(elem)%8==0`的元素，第2台机器只处理`hash(elem)%8==1`的元素，以此类推。查询的时候，先计算这个元素在哪台机器上，然后去那台机器上的HashMap里取出计数器。

方案2能够scale, 但是依旧是把所有元素都存了下来，代价比较高。

如果允许近似计算，那么有很多高效的近似算法，单机就可以处理海量的数据。下面讲几个经典的近似算法。

### 方案3: Count-Min Sketch

Count-Min Sketch 算法流程：

1. 选定d个hash函数，开一个 dxm 的二维整数数组作为哈希表
2. 对于每个元素，分别使用d个hash函数计算相应的哈希值，并对m取余，然后在对应的位置上增1，二维数组中的每个整数称为sketch
3. 要查询某个元素的频率时，只需要取出d个sketch, 返回最小的那一个（其实d个sketch都是该元素的近似频率，返回任意一个都可以，该算法选择最小的那个）

![频率估计 - 图1](https://static.bookstack.cn/projects/system-design/images/count-min-sketch.jpg)

这个方法的思路和 Bloom Filter 比较类似，都是用多个hash函数来降低冲突。

- 空间复杂度`O(dm)`。Count-Min Sketch 需要开一个 `dxm` 大小的二位数组，所以空间复杂度是`O(dm)`
- 时间复杂度`O(n)`。Count-Min Sketch 只需要一遍扫描，所以时间复杂度是`O(n)`

Count-Min Sketch算法的优点是省内存，缺点是对于出现次数比较少的元素，准确性很差，因为二维数组相比于原始数据来说还是太小，hash冲突比较严重，导致结果偏差比较大。

### 方案4: Count-Mean-Min Sketch

Count-Min Sketch算法对于低频的元素，结果不太准确，主要是因为hash冲突比较严重，产生了噪音，例如当m=20时，有1000个数hash到这个20桶，平均每个桶会收到50个数，这50个数的频率重叠在一块了。Count-Mean-Min Sketch 算法做了如下改进：

- 来了一个查询，按照 Count-Min Sketch的正常流程，取出它的d个sketch
- 对于每个hash函数，估算出一个噪音，噪音等于该行所有整数(除了被查询的这个元素)的平均值
- 用该行的sketch 减去该行的噪音，作为真正的sketch
- 返回d个sketch的中位数

1. `class CountMeanMinSketch {`
2.     `// initialization and addition procedures as in CountMinSketch`
3.     `// n is total number of added elements`
4.     `long estimateFrequency(value) {`
5.         `long e[] = new long[d]`
6.         `for(i = 0; i < d; i++) {`
7.             `sketchCounter = estimators[i][ hash(value, i) ]`
8.             `noiseEstimation = (n - sketchCounter) / (m - 1)`
9.             `e[i] = sketchCounter – noiseEstimator`
10.         `}`
11.         `return median(e)`
12.     `}`
13. `}`

Count-Mean-Min Sketch算法能够显著的改善在长尾数据上的精确度。

# Top K 频繁项
寻找数据流中出现最频繁的k个元素(find top k frequent items in a data stream)。这个问题也称为 Heavy Hitters.

这题也是从实践中提炼而来的，例如搜索引擎的热搜榜，找出访问网站次数最多的前10个IP地址，等等。

### 方案1: HashMap + Heap

用一个 `HashMap<String, Long>`，存放所有元素出现的次数，用一个小根堆，容量为k，存放目前出现过的最频繁的k个元素，

1. 每次从数据流来一个元素，如果在HashMap里已存在，则把对应的计数器增1，如果不存在，则插入，计数器初始化为1
2. 在堆里查找该元素，如果找到，把堆里的计数器也增1，并调整堆；如果没有找到，把这个元素的次数跟堆顶元素比较，如果大于堆丁元素的出现次数，则把堆丁元素替换为该元素，并调整堆

- 空间复杂度`O(n)`。HashMap需要存放下所有元素，需要`O(n)`的空间，堆需要存放k个元素，需要`O(k)`的空间，跟`O(n)`相比可以忽略不急，总的时间复杂度是`O(n)`
- 时间复杂度`O(n)`。每次来一个新元素，需要在HashMap里查找一下，需要`O(1)`的时间；然后要在堆里查找一下，`O(k)`的时间，有可能需要调堆，又需要`O(logk)`的时间，总的时间复杂度是`O(n(k+logk))`，k是常量，所以可以看做是O(n)。

如果元素数量巨大，单机内存存不下，怎么办？ 有两个办法，见方案2和3。

### 方案2: 多机HashMap + Heap

- 可以把数据进行分片。假设有8台机器，第1台机器只处理`hash(elem)%8==0`的元素，第2台机器只处理`hash(elem)%8==1`的元素，以此类推。
- 每台机器都有一个HashMap和一个 Heap, 各自独立计算出 top k 的元素
- 把每台机器的Heap，通过网络汇总到一台机器上，将多个Heap合并成一个Heap，就可以计算出总的 top k 个元素了

### 方案3: Count-Min Sketch + Heap

既然方案1中的HashMap太大，内存装不小，那么可以用[Count-Min Sketch算法](https://www.bookstack.cn/read/system-design/cn-bigdata-frequency-estimation.md)代替HashMap，

- 在数据流不断流入的过程中，维护一个标准的Count-Min Sketch 二维数组
- 维护一个小根堆，容量为k
- 每次来一个新元素，
    - 将相应的sketch增1
    - 在堆中查找该元素，如果找到，把堆里的计数器也增1，并调整堆；如果没有找到，把这个元素的sketch作为钙元素的频率的近似值，跟堆顶元素比较，如果大于堆丁元素的频率，则把堆丁元素替换为该元素，并调整堆

这个方法的时间复杂度和空间复杂度如下：

- 空间复杂度`O(dm)`。m是二维数组的列数，d是二维数组的行数，堆需要`O(k)`的空间，不过k通常很小，堆的空间可以忽略不计
- 时间复杂度`O(nlogk)`。每次来一个新元素，需要在二维数组里查找一下，需要`O(1)`的时间；然后要在堆里查找一下，`O(logk)`的时间，有可能需要调堆，又需要`O(logk)`的时间，总的时间复杂度是`O(nlogk)`。

### 方案4: Lossy Counting

Lossy Couting 算法流程：

1. 建立一个HashMap，用于存放每个元素的出现次数
2. 建立一个窗口（窗口的大小由错误率决定，后面具体讨论）
3. 等待数据流不断流进这个窗口，直到窗口满了，开始统计每个元素出现的频率，统计结束后，每个元素的频率减1，然后将出现次数为0的元素从HashMap中删除
4. 返回第2步，不断循环

Lossy Counting 背后朴素的思想是，出现频率高的元素，不太可能减一后变成0，如果某个元素在某个窗口内降到了0，说明它不太可能是高频元素，可以不再跟踪它的计数器了。随着处理的窗口越来越多，HashMap也会不断增长，同时HashMap里的低频元素会被清理出去，这样内存占用会保持在一个很低的水平。

很显然，Lossy Counting 算法是个近似算法，但它的错误率是可以在数学上证明它的边界的。假设要求错误率不大于ε，那么窗口大小为1/ε，对于长度为N的流，有N／（1/ε）＝εN 个窗口，由于每个窗口结束时减一了，那么频率最多被少计数了窗口个数εN。

该算法只需要一遍扫描，所以时间复杂度是`O(n)`。

该算法的内存占用，主要在于那个HashMap, Gurmeet Singh Manku 在他的论文里，证明了HashMap里最多有 `1/ε log (εN)`个元素，所以空间复杂度是`O(1/ε log (εN))`。

### 方案5: SpaceSaving

TODO, 原始论文 “Efficient Computation of Frequent and Top-k Elements in Data Streams”

先拿10000个数建堆，然后一次添加剩余元素，如果大于堆顶的数（10000中最小的），将这个数替换堆顶，并调整结构使之仍然是一个最小堆，这样，遍历完后，堆中的10000个数就是所需的最大的10000个。建堆时间复杂度是O（mlogm），算法的时间复杂度为O（nmlogm）（n为10亿，m为10000）。

优化的方法：可以把所有10亿个数据分组存放，比如分别放在1000个文件中。这样处理就可以分别在每个文件的10^6个数据中找出最大的10000个数，合并到一起在再找出最终的结果。

以上就是面试时简单提到的内容，下面整理一下这方面的问题：

## top K问题

在大规模数据处理中，经常会遇到的一类问题：在海量数据中找出出现频率最好的前k个数，或者从海量数据中找出最大的前k个数，这类问题通常被称为top K问题。例如，在搜索引擎中，统计搜索最热门的10个查询词；在歌曲库中统计下载最高的前10首歌等。

针对top K类问题，通常比较好的方案是分治+Trie树/hash+小顶堆（就是上面提到的最小堆），即先将数据集按照Hash方法分解成多个小数据集，然后使用Trie树活着Hash统计每个小数据集中的query词频，之后用小顶堆求出每个数据集中出现频率最高的前K个数，最后在所有top K中求出最终的top K。

### eg：有1亿个浮点数，如果找出期中最大的10000个？

最容易想到的方法是将数据全部排序，然后在排序后的集合中进行查找，最快的排序算法的时间复杂度一般为O（nlogn），如快速排序。但是在32位的机器上，每个float类型占4个字节，1亿个浮点数就要占用400MB的存储空间，对于一些可用内存小于400M的计算机而言，很显然是不能一次将全部数据读入内存进行排序的。其实即使内存能够满足要求（我机器内存都是8GB），该方法也并不高效，因为题目的目的是寻找出最大的10000个数即可，而排序却是将所有的元素都排序了，做了很多的无用功。

第二种方法为局部淘汰法，该方法与排序方法类似，用一个[容器](https://cloud.tencent.com/product/tke?from=20065&from_column=20065)保存前10000个数，然后将剩余的所有数字——与容器内的最小数字相比，如果所有后续的元素都比容器内的10000个数还小，那么容器内这个10000个数就是最大10000个数。如果某一后续元素比容器内最小数字大，则删掉容器内最小元素，并将该元素插入容器，最后遍历完这1亿个数，得到的结果容器中保存的数即为最终结果了。此时的时间复杂度为O（n+m^2），其中m为容器的大小，即10000。

第三种方法是分治法，将1亿个数据分成100份，每份100万个数据，找到每份数据中最大的10000个，最后在剩下的100\*10000个数据里面找出最大的10000个。如果100万数据选择足够理想，那么可以过滤掉1亿数据里面99%的数据。100万个数据里面查找最大的10000个数据的方法如下：用快速排序的方法，将数据分为2堆，如果大的那堆个数N大于10000个，继续对大堆快速排序一次分成2堆，如果大的那堆个数N大于10000个，继续对大堆快速排序一次分成2堆，如果大堆个数N小于10000个，就在小的那堆里面快速排序一次，找第10000-n大的数字；递归以上过程，就可以找到第1w大的数。参考上面的找出第1w大数字，就可以类似的方法找到前10000大数字了。此种方法需要每次的内存空间为10^6\*4=4MB，一共需要101次这样的比较。

第四种方法是Hash法。如果这1亿个书里面有很多重复的数，先通过Hash法，把这1亿个数字去重复，这样如果重复率很高的话，会减少很大的内存用量，从而缩小运算空间，然后通过分治法或最小堆法查找最大的10000个数。

第五种方法采用最小堆。首先读入前10000个数来创建大小为10000的最小堆，建堆的时间复杂度为O（mlogm）（m为数组的大小即为10000），然后遍历后续的数字，并于堆顶（最小）数字进行比较。如果比最小的数小，则继续读取后续数字；如果比堆顶数字大，则替换堆顶元素并重新调整堆为最小堆。整个过程直至1亿个数全部遍历完为止。然后按照中序遍历的方式输出当前堆中的所有10000个数字。该算法的时间复杂度为O（nmlogm），空间复杂度是10000（常数）。

### 实际运行：

实际上，最优的解决方案应该是最符合实际设计需求的方案，在时间应用中，可能有足够大的内存，那么直接将数据扔到内存中一次性处理即可，也可能机器有多个核，这样可以采用多线程处理整个数据集。

下面针对不容的应用场景，分析了适合相应应用场景的解决方案。

#### （1）单机+单核+足够大内存

如果需要查找10亿个查询次（每个占8B）中出现频率最高的10个，考虑到每个查询词占8B，则10亿个查询次所需的内存大约是10^9 * 8B=8GB内存。如果有这么大内存，直接在内存中对查询次进行排序，顺序遍历找出10个出现频率最大的即可。这种方法简单快速，使用。然后，也可以先用HashMap求出每个词出现的频率，然后求出频率最大的10个词。

#### （2）单机+多核+足够大内存

这时可以直接在内存总使用Hash方法将数据划分成n个partition，每个partition交给一个线程处理，线程的处理逻辑同（1）类似，最后一个线程将结果归并。

该方法存在一个瓶颈会明显影响效率，即数据倾斜。每个线程的处理速度可能不同，快的线程需要等待慢的线程，最终的处理速度取决于慢的线程。而针对此问题，解决的方法是，将数据划分成c×n个partition（c>1），每个线程处理完当前partition后主动取下一个partition继续处理，知道所有数据处理完毕，最后由一个线程进行归并。

#### （3）单机+单核+受限内存

这种情况下，需要将原数据文件切割成一个一个小文件，如次啊用hash(x)%M，将原文件中的数据切割成M小文件，如果小文件仍大于内存大小，继续采用Hash的方法对数据文件进行分割，知道每个小文件小于内存大小，这样每个文件可放到内存中处理。采用（1）的方法依次处理每个小文件。

#### （4）多机+受限内存

这种情况，为了合理利用多台机器的资源，可将数据分发到多台机器上，每台机器采用（3）中的策略解决本地的数据。可采用hash+socket方法进行数据分发。

从实际应用的角度考虑，（1）（2）（3）（4）方案并不可行，因为在大规模数据处理环境下，作业效率并不是首要考虑的问题，算法的扩展性和容错性才是首要考虑的。算法应该具有良好的扩展性，以便数据量进一步加大（随着业务的发展，数据量加大是必然的）时，在不修改算法框架的前提下，可达到近似的线性比；算法应该具有容错性，即当前某个文件处理失败后，能自动将其交给另外一个线程继续处理，而不是从头开始处理。 top K问题很适合采用MapReduce框架解决，用户只需编写一个Map函数和两个Reduce 函数，然后提交到Hadoop（采用Mapchain和Reducechain）上即可解决该问题。具体而言，就是首先根据数据值或者把数据hash(MD5)后的值按照范围划分到不同的机器上，最好可以让数据划分后一次读入内存，这样不同的机器负责处理不同的数值范围，实际上就是Map。得到结果后，各个机器只需拿出各自出现次数最多的前N个数据，然后汇总，选出所有的数据中出现次数最多的前N个数据，这实际上就是Reduce过程。对于Map函数，采用Hash算法，将Hash值相同的数据交给同一个Reduce task；对于第一个Reduce函数，采用HashMap统计出每个词出现的频率，对于第二个Reduce 函数，统计所有Reduce task，输出数据中的top K即可。 直接将数据均分到不同的机器上进行处理是无法得到正确的结果的。因为一个数据可能被均分到不同的机器上，而另一个则可能完全聚集到一个机器上，同时还可能存在具有相同数目的数据。


# 范围查询
给定一个无限的整数数据流，如何查询在某个范围内的元素出现的总次数？例如数据库常常需要SELECT count(v) WHERE v >= l AND v < u。这个经典的问题称为范围查询(Range Query)。

### 方案1: Array of Count-Min Sketches

有一个简单方法，既然[Count-Min Sketch](https://soulmachine.gitbooks.io/system-design/content/cn/bigdata/frequency-estimation.html)可以计算每个元素的频率，那么我们把指定范围内所有元素的sketch加起来，不就是这个范围内元素出现的总数了吗？要注意，由于每个sketch都是近似值，多个近似值相加，误差会被放大，所以这个方法不可行。

解决的办法就是使用多个“分辨率”不同的Count-Min Sketch。第1个sketch每个格子存放单个元素的频率，第2个sketch每个格子存放2个元素的频率（做法很简答，把该元素的哈希值的最低位bit去掉，即右移一位，等价于除以2，再继续后续流程），第3个sketch每个格子存放4个元素的频率（哈希值右移2位即可），以此类推，最后一个sketch有2个格子，每个格子存放一半元素的频率总数，即第1个格子存放最高bit为0的元素的总次数，第2个格子存放最高bit为1的元素的总次数。Sketch的个数约等于`log(不同元素的总数)`。

- 插入元素时，算法伪代码如下，
    
    1.   `def insert(x):`
    2.       `for i in range(1, d+1):`
    3.           `M1[i][h[i](x)] += 1`
    4.           `M2[i][h[i](x)/2] += 1`
    5.           `M3[i][h[i](x)/4] += 1`
    6.           `M4[i][h[i](x)/8] += 1`
    7.           `# ...`
    
- 查询范围\[l, u)时，从粗粒度到细粒度，找到多个区间，能够不重不漏完整覆盖区间\[l, u)，将这些sketch的值加起来，就是该范围内的元素总数。举个例子，给定某个范围，如下图所示，最粗粒度的那个sketch里找不到一个格子，就往细粒度找，最后找到第1个sketch的2个格子，第2个sketch的1个格子和第3个sketch的1个格子，共4个格子，能够不重不漏的覆盖整个范围，把4个红线部分的值加起来就是所求结果
    
    ![范围查询 - 图1](https://static.bookstack.cn/projects/system-design/images/array-of-count-min-sketch.png)
# 成员查询
给定一个无限的数据流和一个有限集合，如何判断数据流中的元素是否在这个集合中？

在实践中，我们经常需要判断一个元素是否在一个集合中，例如垃圾邮件过滤，爬虫的网址去重，等等。这题也是一道很经典的题目，称为成员查询(Membership Query)。

答案: Bloom Filter

#  海量日志数据, 提取出某日访问百度次数最多的那个IP
具体做法如下：

1. 按照IP地址的Hash(IP)%1024值, 把海量IP日志分别存储到1024个小文件中.
2. 对于每一个小文件, 构建一个以IP为key, 出现次数为value的HashMap, 同时记录当前出现次数最多的那个IP地址;
3. 得到1024个小文件中的出现次数最多的IP, 再依据常规的排序算法得到总体上出现次数最多的IP.


-   出现大量 TIME_WAIT 的话应用层有什么优化方案？
-   连接池的中心思想是什么？主要解决的是什么问题？
-   了解 TCP 中的粘包现象吗？
-   如果有一个请求需要发送数据，但是我想把包拆分开（不等待），在应用层应该怎么做？


### 6、golang共享内存（互斥锁）方法实现发送多个get请求

待补充

利用 golang 特性，设计一个 QPS 为 500 的服务器



#### 项⽬经验

-   项⽬整体架构(能画出来)
-   项⽬上下游关系(能将明⽩)
-   项⽬实现细节
-   项⽬主要亮点
//todo

##  系统设计
todo
-   RPC的设计
    
-   架构设计分单系统，每秒3000订单有效期15分钟，50W司机进行抢单操作，如果一直没有抢单，则订单失效
    
-   字符串hash算法的实现
    
-   敏感词过滤
    
-   设计一个高可用的稳定的并发模型处理HTTP请求


### [](https://github.com/go-share-team/go_interview/blob/master/%E9%9D%A2%E7%BB%8F/%E5%AD%97%E8%8A%82/3.md#%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E7%AE%97%E6%B3%95)数据结构与算法：

1、倒排索引和B+树 2、判断链表是否有环，时间复杂度要求0(1) 3、LeetCode上合并区间的题 4、leetcode的股票买卖的题 5、二叉树的最近公共祖先 6、有序数组合并 7、什么是平衡二叉树、最小堆 8、大文件的top10问题 9、golang实现栈、队列


# 如何优雅的实现一个 `goroutine` 池

##  如果有一个排行榜，用 zset，根据积分和时间来排序，积分高的，时间最近的拍前面，怎么实现？

## go语言实现的真正的多层时间轮

使用时间轮算法实现延迟队列 (一)（Golang）
https://raymondjiang.net/2020/12/10/implement-delay-queue-use-golang/#top


# 堆。什么是最小堆？什么是最大堆？在堆中怎么插入一个元素？
```go
func NewMinHeap() *MinHeap {  
   //第一个元素仅用于结束insert中的for循环  
   h := &MinHeap{  
      Element: []int{math.MinInt64},  
   }  
   return h  
}  
func (h *MinHeap) Insert(v int) {  
   h.Element = append(h.Element, v)  
   i := len(h.Element) - 1  
   //上浮  
   for ; h.Element[i/2] > v; i /= 2 {  
      h.Element[i] = h.Element[i/2]  
   }  
   h.Element[i] = v  
}  
func (h *MinHeap) DeleteMin() (int, error) {  
   if len(h.Element) <= 1 {  
      return 0, fmt.Errorf("MinHeap is empty")  
   }  
   minElement := h.Element[1]  
   lastElement := h.Element[len(h.Element)-1]  
   var i, child int  
   for i = 1; i*2 < len(h.Element); i = child {  
      child = i * 2  
      if child < len(h.Element)-1 && h.Element[child+1] < h.Element[child] {  
         child++  
      }  
      // 下滤一层  
      if lastElement > h.Element[child] {  
         h.Element[i] = h.Element[child]  
      } else {  
         break  
      }  
   }  
   h.Element[i] = lastElement  
   h.Element = h.Element[:len(h.Element)-1]  
   return minElement, nil  
}
```

# # 解Go标准库http包是如何处理keep-alive连接的

### 1. http包默认启用keep-alive

按照HTTP/1.1的规范，Go http包的http server和client的实现默认将所有连接视为长连接，无论这些连接上的初始请求是否带有**Connection: keep-alive**。

### 2. http client端基于非keep-alive连接发送请求

有时候http client在一条连接上的数据请求密度并不高，因此client端并不想长期保持这条连接(占用端口资源)，那么client端如何协调Server端在处理完请求返回应答后就关闭这条连接呢？我们看看在Go中如何实现这一场景需求：

http.Client底层的数据连接建立和维护是由http.Transport实现的，http.Transport结构有一个DisableKeepAlives字段，其默认值为false，即启动keep-alive。这里我们将其置为false，即关闭keep-alive，然后将该Transport实例作为初值，赋值给http Client实例的Transport字段。

### 3. 建立一个不支持keep-alive连接的http server

假设我们有这样的一个需求，server端完全不支持keep-alive的连接，无论client端发送的请求header中是否显式带有**Connection: keep-alive**，server端都会在返回应答后关闭连接。那么在Go中，我们如何来实现这一需求呢？

我们看到在ListenAndServe前，我们调用了http.Server的SetKeepAlivesEnabled方法，并传入false参数，这样我们就在全局层面关闭了该Server对keep-alive连接的支持


### 4. 支持长连接闲置超时关闭的http server

显然上面的server处理方式“太过霸道”，对于想要复用连接，提高请求和应答传输效率的client而言，上面的“一刀切”机制并不合理。那么是否有一种机制可以让http server即可以对高密度传输数据的连接保持keep-alive，又可以及时清理掉那些长时间没有数据传输的idle连接，释放占用的系统资源呢？我们来看下面这个go实现的server：
从代码中我们看到，我们仅在创建http.Server实例时显式为其字段IdleTimeout做了一次显式赋值，设置idle连接的超时时间为5s。下面是Go标准库中关于http.Server的字段IdleTimeout的注释：

```
// $GOROOT/src/net/server.go

// IdleTimeout是当启用keep-alive时等待下一个请求的最大时间。
// 如果IdleTimeout为零，则使用ReadTimeout的值。如果两者都是
// 零，则没有超时。
IdleTimeout time.Duration
```

# io timeout的坑
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net"
    "net/http"
    "time"
)

var tr *http.Transport

func init() {
    tr = &http.Transport{
        MaxIdleConns: 100,
        Dial: func(netw, addr string) (net.Conn, error) {
            conn, err := net.DialTimeout(netw, addr, time.Second*2) //设置建立连接超时
            if err != nil {
                return nil, err
            }
            err = conn.SetDeadline(time.Now().Add(time.Second * 3)) //设置发送接受数据超时
            if err != nil {
                return nil, err9
            }
            return conn, nil
        },
    }
}

func main() {
    for {
        _, err := Get("http://www.baidu.com/")
        if err != nil {
            fmt.Println(err)
            break
        }
    }
}


func Get(url string) ([]byte, error) {
    m := make(map[string]interface{})
    data, err := json.Marshal(m)
    if err != nil {
        return nil, err
    }
    body := bytes.NewReader(data)
    req, _ := http.NewRequest("Get", url, body)
    req.Header.Add("content-type", "application/json")

    client := &http.Client{
        Transport: tr,
    }
    res, err := client.Do(req)
    if res != nil {
        defer res.Body.Close()
    }
    if err != nil {
        return nil, err
    }
    resBody, err := ioutil.ReadAll(res.Body)
    if err != nil {
        return nil, err
    }
    return resBody, nil
}
```
实际生产中发生的现象是，golang 服务在发起 http 调用时，虽然 http.Transport 设置了 3s 超时，会偶发出现 i/o timeout 的报错。

但是查看下游服务的时候，发现下游服务其实 100ms 就已经返回了。

也就是说，这里的 3s 超时，其实是在建立连接之后开始算的，而不是单次调用开始算的超时。

看注释里写的是

SetDeadline sets the read and write deadlines associated with the connection.

超时原因
大家知道 HTTP 是应用层协议，传输层用的是 TCP 协议。

HTTP 协议从 1.0 以前，默认用的是短连接，每次发起请求都会建立 TCP 连接。收发数据。然后断开连接。

TCP 连接每次都是三次握手。每次断开都要四次挥手。

其实没必要每次都建立新连接，建立的连接不断开就好了，每次发送数据都复用就好了。

于是乎，HTTP 协议从 1.1 之后就默认使用长连接。具体相关信息可以看之前的 这篇文章。

那么 golang标准库里也兼容这种实现。

通过建立一个连接池，针对每个域名建立一个 TCP 长连接，比如 http://baidu.com 和 http://golang.com 就是两个不同的域名。

第一次访问 http://baidu.com 域名的时候会建立一个连接，用完之后放到空闲连接池里，下次再要访问 http://baidu.com 的时候会重新从连接池里把这个连接捞出来复用。

![[Pasted image 20230615154853.png]]

插个题外话：这也解释了之前这篇文章里最后的疑问，为什么要强调是同一个域名：一个域名会建立一个连接，一个连接对应一个读 goroutine 和一个写 goroutine。正因为是同一个域名，所以最后才会泄漏 3 个 goroutine，如果不同域名的话，那就会泄漏 1+2\*N 个协程，N 就是域名数。

假设第一次请求要 100ms，每次请求完 http://baidu.com 后都放入连接池中，下次继续复用，重复 29 次，耗时 2900ms。

第 30 次请求的时候，连接从建立开始到服务返回前就已经用了 3000ms，刚好到设置的 3s 超时阈值，那么此时客户端就会报超时 i/o timeout 。

虽然这时候服务端其实才花了 100ms，但耐不住前面 29次加起来的耗时已经很长。

也就是说只要通过 http.Transport 设置了 err = conn.SetDeadline(time.Now().Add(time.Second * 3))，并且你用了长连接，哪怕服务端处理再快，客户端设置的超时再长，总有一刻，你的程序会报超时错误。

正确姿势
原本预期是给每次调用设置一个超时，而不是给整个连接设置超时。

另外，上面出现问题的原因是给长连接设置了超时，且长连接会复用。

基于这两点，改一下代码。

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"
    "time"
)

var tr *http.Transport

func init() {
    tr = &http.Transport{
        MaxIdleConns: 100,
        // 下面的代码被干掉了
        //Dial: func(netw, addr string) (net.Conn, error) {
        //    conn, err := net.DialTimeout(netw, addr, time.Second*2) //设置建立连接超时
        //    if err != nil {
        //        return nil, err
        //    }
        //    err = conn.SetDeadline(time.Now().Add(time.Second * 3)) //设置发送接受数据超时
        //    if err != nil {
        //        return nil, err
        //    }
        //    return conn, nil
        //},
    }
}


func Get(url string) ([]byte, error) {
    m := make(map[string]interface{})
    data, err := json.Marshal(m)
    if err != nil {
        return nil, err
    }
    body := bytes.NewReader(data)
    req, _ := http.NewRequest("Get", url, body)
    req.Header.Add("content-type", "application/json")

    client := &http.Client{
        Transport: tr,
        Timeout: 3*time.Second,  // 超时加在这里，是每次调用的超时
    }
    res, err := client.Do(req) 
    if res != nil {
        defer res.Body.Close()
    }
    if err != nil {
        return nil, err
    }
    resBody, err := ioutil.ReadAll(res.Body)
    if err != nil {
        return nil, err
    }
    return resBody, nil
}

func main() {
    for {
        _, err := Get("http://www.baidu.com/")
        if err != nil {
            fmt.Println(err)
            break
        }
    }
}
```
看注释会发现，改动的点有两个

http.Transport 里的建立连接时的一些超时设置干掉了。

在发起 http 请求的时候会场景 http.Client，此时加入超时设置，这里的超时就可以理解为单次请求的超时了。同样可以看下注释

Timeout specifies a time limit for requests made by this Client.

到这里，代码就改好了，实际生产中问题也就解决了。

实例代码里，如果拿去跑的话，其实还会下面的错

Get http://www.baidu.com/: EOF
这个是因为调用得太猛了，http://www.baidu.com 那边主动断开的连接，可以理解为一个限流措施，目的是为了保护服务器，毕竟每个人都像这么搞，服务器是会炸的。。。

解决方案很简单，每次 HTTP 调用中间加个 sleep 间隔时间就好。


# es优化
禁止swap
使用g1垃圾搜集
使用ssd
批量提交
增加refresh间隔
### 修改 index_buffer_size 的设置

索引缓冲的设置可以控制多少内存分配给索引进程。这是一个全局配置，会应用于一个节点上所有不同的分片上。

修改translog时机，比如几秒刷一次，多少mb刷一次

查询时多用filter
少用深度分页，多用scroll-scan

mapping中字段少些，还有就是能不索引就不索引，能不评分就不评分，

# 有了mvcc为什么还需要gap
可能有读者会疑惑，事务的隔离级别其实都是对于读数据的定义，但到了这里，就被拆成了读和写两个模块来讲解。这主要是因为MySQL中的读，和事务隔离级别中的读，是不一样的。

我们且看，在RR级别中，通过MVCC机制，虽然让数据变得可重复读，但我们读到的数据可能是历史数据，是不及时的数据，不是数据库当前的数据！这在一些对于数据的时效特别敏感的业务中，就很可能出问题。

对于这种读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库当前版本数据的方式，叫当前读 (current read)。很显然，在MVCC中：

- 快照读：就是select
    - select * from table ….;
- 当前读：特殊的读操作，插入/更新/删除操作，属于当前读，处理的都是当前的数据，需要加锁。
    - select * from table where ? lock in share mode;
    - select * from table where ? for update;
    - insert;
    - update ;
    - delete;  
        

事务的隔离级别实际上都是定义了当前读的级别，MySQL为了减少锁处理（包括等待其它锁）的时间，提升并发能力，引入了快照读的概念，使得select不用加锁。而update、insert这些“当前读”，就需要另外的模块来解决了。

### 写（”当前读”）

事务的隔离级别中虽然只定义了读数据的要求，实际上这也可以说是写数据的要求。上文的“读”，实际是讲的快照读；而这里说的“写”就是当前读了。

为了解决当前读中的幻读问题，MySQL事务使用了Next-Key锁。

# mysql rr隔离级别存在丢失更新的情况
要加锁才行

# mysql 添加索引会锁表吗

将分类讨论在业务场景不同版本Mysql修改表结构添加索引是否会锁表 alter table add index 操作

## 数据量小

当数据量较小时，即使锁表也没有关系，其他的DML等待执行即可，业务中可以以一千万作为一个判定值，可以直接执行修改表结构操作，短暂性锁表无伤大雅

## 数据量大

当数据量大时，例如几千万、几亿数据，mysql建立索引默认需要锁住整个表，然后再建立索引，由于数据量巨大，从磁盘读取数据创建索引所需时间会比较长，此时业务是不能停的，因此不允许这样操作，会导致业务无法使用该数据库。

## 解决办法：使用inplace算法

在mysql 7.5版本及以后支持了设置algrothm=default/copy/inplace

- default 默认算法，会锁表
- copy 算法，允许DQL不支持DML，能读不能写，操作是创建一个新的空表，然后从旧表逐个读数据修改表结构写入新表，最后重命名
- inplace算法 支持DQL、DML，操作时能读写 文档里没有写具体是怎么操作的，后续找到再补充
- algrothm参数为可选参数，默认使用default算法，若支持copy算法则默认用copy算法，若支持inplace算法则默认使用inplace算法，默认选取优先级 inplace>copy>default


# 如何优化group
group by 总结
group by与order by的索引优化基本一样，group by实质是先排序后分组，也就是分组之前必排序，遵照索引的最佳左前缀原则可以大大提高group by的效率。

当无法使用索引列排序时，适当增大sort_buffer_size参数 + 适当增大max_length_for_sort_data参数可以提高filesort排序的效率。注意：可能会出现Using temporary，也就是说mysql在对查询结果排序时使用了临时表。

where高于having，能写在where限定条件中的就尽量写在where中。

# 数据库用datetime还是timestamp
![[Pasted image 20230624205045.png]]


## 算法题：
 快排一定最快吗？
不一定，当待排序的序列已经有序，不管是升序还是降序。此时快速排序最慢，一般当数据量很大的时候，用快速排序比较好，为了避免原来的序列有序。

1.  B树 B+树 红黑树 跳表描述
8.  LRU实现--针对hash存储的方式，如何改进？
    
-   二叉树的右视图
    
-   LRU缓存机制 （考虑并发访问）
    
-   高并发的生产者消费者模式
    
-   通过中序遍历序列和先序序列恢复二叉树
    
-   爬楼梯问题
    
-   单链表逆序
    
-   单向链表排序
    
-   string1 = 1234dsafaserewr，string2 = 23aefasdfwer，求string3 = string1 + string2
    
-   二叉树节点的公共祖先
    
-   二叉树的最大深度
    
-   二叉树的中序遍历和层次遍历
    
-   寻找两个升序数组的第K大值
    
-   最长回文子串长度
    
-   最短回文串
    
-   合并两个有序链表
    
-   全排列
    
-   接雨水
    
-   盛最多水的容器
    
-   Pow(x, n)
    
-   合并两个有序的单链表
    
-   数组中值出现了一次的数字
    
-   判断二叉树是否为平衡二叉树
    
-   找到字符串的最长无重复字符子串
    

## [](https://github.com/go-share-team/go_interview/blob/master/%E9%9D%A2%E7%BB%8F/%E8%85%BE%E8%AE%AF/%E8%85%BE%E8%AE%AF-%E6%BB%B4%E6%BB%B4-%E5%A4%B4%E6%9D%A1-%E9%98%BF%E9%87%8C-%E5%A5%BD%E6%9C%AA%E6%9D%A5-%E8%B7%9F%E8%B0%81%E5%AD%A6-%E8%B4%A7%E6%8B%89%E6%8B%89-%E6%9C%80%E5%8F%B3-%E7%99%BE%E5%BA%A6-%E5%BA%A6%E5%B0%8F%E6%BB%A1-%E5%B0%8F%E7%B1%B3--%E5%90%88%E9%9B%86.md#%E6%B5%B7%E9%87%8F%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86%E9%97%AE%E9%A2%98%E9%9D%A2%E8%AF%95%E5%AE%98%E5%BE%88%E5%96%9C%E6%AC%A2%E9%97%AE)海量数据处理问题（xx官很喜欢问）

-   hash
-   字典树
-   bitmap
-   布隆过滤器
-   MapReduce
-   桶

算法题
-   熟悉的排序算法有哪些？归并排序和快速排序还记得原理吗？
-   归并排序时间复杂度是什么？有什么样的优缺点呢？
写代码

-   给定 n 个不同元素的数组，设计算法等概率取 m 个不同的元素

智力题

-   掷骰子，游戏规则：希望结果尽可能大，如果对第一次的结果不满意可以掷第二次，但是第一次结果就作废了，以第二次的结果为准。这个掷骰子结果的数学期望是多少呢？

写代码

-   输入两个整数 a 和 b，输出他们相除的结果，如果有循环小数用括号表示。如：
    -   a=-1，b=2，输出 “-0.5”
    -   a=1，b=3，输出 “0.(3)”
    -   a=10，b=80，输出 “0.125”
    -   a=-100，b=10，输出 “-10”


-   操作系统有任务管理器，管理任务的执行，任务之间存在依赖关系，写一个算法判断是否存在任务的循环依赖？
-   是否了解二叉排序树？左子树比根小，右子树比根大，如何移除一个二叉排序树的节点？
    -   这个题目卡了挺久的，因为数据结构和算法比较薄弱，xx官给了很多提示，包括指引我从最简单的情况（所有节点只有左节点）开始考虑；


-   用 RAND(5) 实现 RAND(7)？
    -   眼熟，但是不会；
-   A 和 B 从一堆棋子中轮流取，每次需要取走 1-5 个棋子，A 先取，如何必胜？
    -   眼熟，讨论了一会；
-   翻转链表中的第 m 至 n 的节点
    -   不难；


写代码

-   给定一个正整数数组 arr，求 arr[i] / arr[j] 的最大值，其中 i < j。
-   掷骰子，游戏规则：希望结果尽可能大，如果对第一次的结果不满意可以掷第二次，但是第一次结果就作废了，以第二次的结果为准。这个掷骰子结果的数学期望是多少呢？
-   Hint 1：如果第一次扔到 6，还考虑扔第二次吗？如果第一次扔到 1 考虑吗？
-   Hint 2：那什么情况考虑扔第二次，什么情况不考虑？
-   输入两个整数 a 和 b，输出他们相除的结果，如果有循环小数用括号表示。如：
-   a=-1，b=2，输出 "-0.5"
-   a=1，b=3，输出 "0.(3)"
-   a=10，b=80，输出 "0.125"
-   a=-100，b=10，输出 "-10"

-   一副扑克牌中随机取 5 张，取到顺子的概率是多少？
-   Hint 1：一种花色有多少种顺子？9 种
-   Hint 2：一个顺子有 5 张牌，有多少种组合可能？4 ^ 5 种
-   Hint 3：分子已经知道了，分母怎么表示，n 张取 m 张怎么表示？
//todo 
-   二叉树查找最小深度
2.算法题 给定正整数数组nums【】，返回sum为target的两个数的下标
使用数组实现一个队列
一个二分树的中序遍历，我写了前序遍历，中序没有
算法：最长有序括号，常见题，秒
代码题翻转一个链表  
2.写一题算法，层次遍历树并输出每层的层级  
3.写一道题，二叉树的后序遍历，非递归算法。
1.  给出人数，任意的人数可以组成一个队伍，只要队长不同就算是不同的队伍一共有几种： 比如： 共有2个人 ( a 和 b ) 那么可以： i. a 独自一人 ii. b 独自一人 iii. ab 两人，a为队长 iv. ab 两人，b为队长 所以一共有4种组合，写一个函数实现这样的效果
    
2.  在一个迷宫中，你可以上下左右个走一步这要消耗一个单位的时间，或者飞到对称点，这也要消耗一个单位的时间，s为开始，e为结束，0 为可以行走的路，#为路障不可以行走，如： S # 0 # E 0 0 0 0
    

我们可以从S对称飞到右下角，然后往上再往左，一共消耗 3 个时间 写一个函数求到终点消耗时间最少的走法


10、代码考核：链表求和。给定一个链表 L1、 L2，每个元素是为 10 以内的正整数，链表表  
示一个数字，表头为高位。求两个链表之和，以链表形式返回。如： L1:5->6->2,L2:1->7->0，  
和为 562+170=732  

5.如何判断二叉树是否是搜索二叉树

## 排查过的记忆最深的问题
//todo

## cpu 正常时，内存持续增长，排查问题的思路  
//todo

## 8.分布式锁怎么实现  
//todo
redis如何实现的，zk如何实现的

## 手写一个布隆过滤器，环形量表，lru，lfu
//todo
## 6.1亿的数据怎么取前1000条数据
//todo

## 2. 线上某个程序CPU 100%，⼀般是什么原因导致的？怎么排查？讲讲排查步骤？  
todo

## 做个智力题：8个球7个一样重的，有一个偏重，一个天平，如何两次找出偏重的小球
todo


## 大数据专题
10万长度的数组，代表10w个任务，如何用多线程处理完，并且按排好序的序号输出结果
-   给个数组 让你找到最大值
-   一个数组判断某个值在不在
1T的数据如何排序
1T的数据如何取中位数

-   说一下（Redis）分布式锁的实现？
-   setnx / 唯一 value / ttl

-   基于 Redis 的分布式锁会有什么问题？
-   主从模型下同步不保证一致会导致锁失效

-   Redis 分布式锁超时可以超时时间设长一点可以吗？不可以的话需要怎么解决？
-   不根本解决问题，可以考虑旁路的 goroutine 不断自旋续期

-   对 Redis 锁续期这个怎么实现呢？
-   日常在用的 Redis 集群都是什么架构？在主从模式和 Redis Cluster 中分布式锁会有什么问题？

# 如何设计秒杀
食古不化者以为秒杀时的并发：每秒50~100W事务，一个个都要落实到磁盘……
实际上秒杀时的并发：绝大多数事务会被挡回去，返回个“您没有资格参与”完事（当然会说的委婉点）。
秒杀是和正常的购买/浏览完全不同的场景，只要确保有足够的算力把大多数请求挡回去就够了，根本就不应该把这些无效请求漏到后方、对后方服务器造成冲击——既然仅仅是挡回去而已，哪还需要什么事务支持需要什么数据一致。
换句话说，秒杀并不是“雇一个亿的前台业务员、配备一亿台电脑，以便服务所有人”，而是“雇几十个业务员搬出来几十个抽奖箱，让所有人抽奖——拿到奖卷的才有资格继续购买流程”。
于是，通过一步抽奖，就把50w~100w并发这种变态的业务场景变成了“抽到奖券的10个人找柜员交易”这样压根不存在压力的场景了。
你要真堆起50~100w并发，结果就接了10个单……你猜老板会不会当场赶你滚蛋？

用redis 队列或者其他手段，达到多少请求后其他的全部关闭。
  
  
库存再分多一百个库，每个ip都hash到一个库
秒杀要单独的库
秒杀url要当时随机生成或者动态化
### **资源静态化：**

秒杀一般都是特定的商品还有页面模板，现在一般都是前后端分离的，所以页面一般都是不会经过后端的，但是前端也要自己的服务器啊，那就把能提前放入**cdn服务器**的东西都放进去，反正把所有能提升效率的步骤都做一下，减少真正秒杀时候服务器的压力。

我们都知道数据库顶不住但是他的兄弟非关系型的数据库**Redis**能顶啊！

那不简单了，我们要开始秒杀前你通过定时任务或者运维同学**提前把商品的库存加载到Redis中**去，让整个流程都在Redis里面去做，然后等秒杀介绍了，再异步的去修改库存就好了。

但是用了Redis就有一个问题了，我们上面说了我们采用**主从**，就是我们会去读取库存然后再判断然后有库存才去减库存，正常情况没问题，但是高并发的情况问题就很大了。

这里我就不画图了，我本来想画图的，想了半天我觉得语言可能更好表达一点。

**多品几遍！！！**就比如现在库存只剩下1个了，我们高并发嘛，4个服务器一起查询了发现都是还有1个，那大家都觉得是自己抢到了，就都去扣库存，那结果就变成了-3，是的只有一个是真的抢到了，别的都是超卖的。咋办？
用lru

![[Pasted image 20230624204448.png]]

# es的controller选举流程，分片leader选举流程
sync.pool原理

# undo刷盘时机
# redo刷盘时机
# kafka如何优化broker参数
如何基于kafka实现延时策略
消息堆积怎么解决
kafka生产和消费为什么快
如何确保消息有序
既要高并发又要有序消费，应该如何做
为什么会丢数据？
生产者如何把数据发到broker的
重复消费的场景
kafka如何利用pageCache来生产消费快速读写的
如何理解kafka的端道端延迟，如何优化
kafka如何使用零拷贝技术

raft leader选举，脑裂后选举，日志复制，修复不一致日志和数据安全
kafka如何使用mmap
kafka生产者发送消息也会阻塞吗
mysql加索引的时候会不会锁表
带排序的分页查询如何优化
cpu高系统反应慢如何排查
网络四元组的理解
数据达到多少才需要分库分表
为什么es不用b+树做底层结构
什么是自适应哈希索引
like为什么%开头就失效
自增还是uuid，数据库主键类型如何选择
b树和b+区别
explain字段有哪些
innodb io日志相关参数优化
innodb io线程相关参数优化
什么是写失效？
什么是行溢出
如何进行join优化
索引失效的场景







