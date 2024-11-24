本次分享的内容主要分为以下五点：

- HBase基本知识；
- HBase读写流程；
- RowKey设计要点；
- HBase生态介绍；
- HBase典型案例分析。

首先我们简单介绍一下 HBase 是什么。

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/87827D84FA4D94AA808FCF4A9F9639D0.jpg)

HBase 开始是受 Google 的 BigTable 启发而开发的分布式、多版本、面向列的开源数据库。其主要特点是支持上亿行、百万列，支持强一致性、并且具有高扩展、高可用等特点。

既然 HBase 是一种分布式的数据库，那么其和传统的 RMDB 有什么区别的呢？我们先来看看HBase表核心概念，理解这些基本的核心概念对后面我理解 HBase 的读写以及如何设计 HBase 表有着重要的联系。

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/5DE38A96671AAB1DCCD42F6B684AE990.jpg)

HBase 表主要由以下几个元素组成：  
- RowKey：表中每条记录的主键；
- Column Family：列族，将表进行横向切割，后面简称CF；
- Column：属于某一个列族，可动态添加列；
- Version Number：类型为Long，默认值是系统时间戳，可由用户自定义；
- Value：真实的数据。

大家可以从上面的图看出：一行（Row）数据是可以包含一个或多个 Column Family，但是我们并不推荐一张 HBase 表的 Column Family 超过三个。Column 是属于 Column Family 的，一个 Column Family 包含一个或多个 Column。  
在物理层面上，所有的数据其实是存放在 Region 里面的，而 Region 又由 RegionServer 管理，其对于的关系如下：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/88163101AE88382509B6BCCC632C2F75.jpg)
- Region：一段数据的集合；
- RegionServer：用于存放Region的服务。

从上面的图也可以清晰看到，一个 RegionServer 管理多个 Region；而一个 Region 管理一个或多个 Column Family。  
到这里我们已经了解了 HBase 表的组成，但是 HBase 表里面的数据到底是怎么存储的呢？  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/6E2C7D1F01834607D312484BBCC4186E.jpg)

  
上面是一张从逻辑上看 HBase 表形式，这个和关系型数据库很类似。那么如果我们再深入看，可以看出，这张表的划分可以如下图表示。  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/DC3D3A21B634E83C243BE62860B68600.jpg)

  
从上图大家可以明显看出，这张表有两个 Column Family ，分别为 personal 和 office。而 personal 又有三列name、city 以及 phone；office 有两列 tel 以及 address。由于存储在 HBase 里面的表一般有上亿行，所以 HBase 表会对整个数据按照 RowKey 进行字典排序，然后再对这张表进行横向切割。切割出来的数据是存储在 Region 里面，而不同的 Column Family 虽然属于一行，但是其在底层存储是放在不同的 Region 里。所以这张表我用了六种颜色表示，也就是说，这张表的数据会被放在六个 Region 里面的，这就可以把数据尽可能的分散到整个集群。  
在前面我们介绍了 HBase 其实是面向列的数据库，所以说一行 HBase 的数据其实是分了好几行存储，一个列对应一行，HBase 的 KV 结构如下：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/A878E083F61A42E3B518B780E9B2986F.jpg)

为了简便期间，在后面的表示我们删除了类似于 Key Length 的属性，只保留 Row Key、Column Family、Column Qualifier等信息。所以 RowKey 为 Row1 的数据列表示为上图后一行的形式。以此类推，整个表的存储就可以如下表示：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/29FBC1816B31CDB758026A74A79FAB9A.jpg)  
大家可以从上面的 kv 表现形式看出，Row11 的 phone 这列其实是没有数据的，在 HBase 的底层存储里面也就没有存储这列了，这点和我们传统的关系型数据库有很大的区别，有了这个特点， HBase 特别适合存储稀疏表。  
我们前面也将了 HBase 其实是多版本的，那如果我们修改了 HBase 表的一列，HBase 又是如何存储的呢？  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/7C5729ACE2AC5A61DAD707965203E693.jpg)

比如上如中我们将 Row1 的 city 列从北京修改为上海了，如果使用 KV 表示的话，我们可以看出其实底层存储了两条数据，这两条数据的版本是不一样的，新的一条数据版本比之前的新。总结起来就是：  

- HBase支持数据多版本特性，通过带有不同时间戳的多个KeyValue版本来实现的；
- 每次put，delete都会产生一个新的Cell，都拥有一个版本；
- 默认只存放数据的三个版本，可以配置；
- 查询默认返回新版本的数据，可以通过制定版本号或版本数获取旧数据。

到这里我们已经了解了 HBase 表及其底层的 KV 存储了，现在让我们来了解一下 HBase 是如何读写数据的。首先我们来看看 HBase 的架构设计，这种图来自于社区：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/81D0A67BD985C7205A44B01087015325.jpg)

  
HBase 的写过程如下：  
- 先将数据写到WAL中；
- WAL 存放在HDFS之上；
- 每次Put、Delete操作的数据均追加到WAL末端；
- 持久化到WAL之后，再写到MemStore中；
- 两者写完返回ACK到客户端。

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/853B3E02384DF6A4C7D54980EB57A717.jpg)

  
MemStore 其实是一种内存结构，一个Column Family 对应一个MemStore，MemStore 里面的数据也是对 Rowkey 进行字典排序的，如下：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/5B60D3C24EE28178CFC2784190BC01F1.jpg)

  
既然我们写数都是先写 WAL，再写 MemStore ，而 MemStore 是内存结构，所以 MemStore 总会写满的，将 MemStore 的数据从内存刷写到磁盘的操作成为 flush：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/E688A985824420DEEF02BF0BDC5DBB2A.jpg)

  
以下几种行为会导致 flush 操作  

- 全局内存控制；
- MemStore使用达到上限；
- RegionServer的Hlog数量达到上限；
- 手动触发；
- 关闭RegionServer触发。

每次 flush 操作都是将一个 MemStore 的数据写到一个 HFile 里面的，所以上图中 HDFS 上有许多个 HFile 文件。文件多了会对后面的读操作有影响，所以 HBase 会隔一定的时间将 HFile 进行合并。根据合并的范围不同分为 Minor Compaction 和 Major Compaction：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/CA98CB79F4DED4336424CC648874DF68.jpg)

  
**Minor Compaction**: 指选取一些小的、相邻的HFile将他们合并成一个更大的Hfile。  
**Major Compaction**：  

- 将一个column family下所有的 Hfiles 合并成更大的；
- 删除那些被标记为删除的数据、超过TTL（time-to-live）时限的数据，以及超过了版本数量限制的数据。

HBase 读操作相对于写操作更为复杂，其需要读取 BlockCache、MemStore 以及 HFile。  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/A2DBFE18798EB8307E3FE3B7D4BE9093.jpg)

  
上图只是简单的表示 HBase 读的操作，实际上读的操作比这个还要复杂，我这里就不深入介绍了。  
到这里，有些人可能就想到了，前面我们说 HBase 表按照 Rowkey 分布到集群的不同机器上，那么我们如何去确定我们该读写哪些 RegionServer 呢？这就是 HBase Region 查找的问题，  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/B20B0641B4D7D401A0CC3DDC3475C372.jpg)

  
客户端按照上面的流程查找需要读写的 RegionServer 。这个过程一般是次读写的时候进行的，在次读取到元数据之后客户端一般会把这些信息缓存到自己内存中，后面操作直接从内存拿就行。当然，后面元数据信息可能还会变动，这时候客户端会再次按照上面流程获取元数据。  
到这里整个读写流程得基本知识就讲完了。现在我们来看看 HBase RowKey 的设计要点。我们一般都会说，看 HBase 设计的好不好，就看其 RowKey 设计的好不好，所以RowKey 的设计在后面的写操作至关重要。我们先来看看 Rowkey 的作用  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/D765EF08608CF7FBE88D927966D3DE6A.jpg)

  
HBase 中的 Rowkey 主要有以下的作用：  

- 读写数据时通过Row Key找到对应的Region
- MemStore 中的数据按RowKey字典顺序排序
- HFile中的数据按RowKey字典顺序排序

从下图可以看到，底层的 HFile 终是按照 Rowkey 进行切分的，所以我们的设计原则是**结合业务的特点，并考虑高频查询，尽可能的将数据打散到整个集群。**  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/B430C82FEB23B08859FF4752378421D3.jpg)

  
一定要充分分析清楚后面我们的表需要怎么查询。下面我们来看看三种比较场景的 Rowkey 设计方案。  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/99E307384A8FE7E7A6A9B95868B1A0CE.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/891428ED2B5BE43DA9C3AE7D462CFDBC.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/E7F053D362C95DE20802707C1ACF8576.jpg)

  
这三种 Rowkey 的设计非常常见，具体的内容图片上也有了，我就不打文字了。  
数据如果只是存储在哪里其实并没有什么用，我们还需要有办法能够使用到里面的数据。幸好的是，当前 HBase 有许多的组件可以满足我们各种需求。如下图是 HBase 比较常用的组件：  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/59DAF735471D3D9F623B8426767A02CF.jpg)

  
HBase 的生态主要有：  

- Phoenix：主要提供使用 SQL 的方式来查询 HBase 里面的数据。一般能够在毫秒级别返回，比较适合 OLTP 场景。
- Spark：我们可以使用 Spark 进行 OLAP 分析；也可以使用 Spark SQL 来满足比较复杂的 SQL 查询场景；使用 Spark Streaming 来进行实时流分析。
- Solr：原生的 HBase 只提供了 Rowkey 单主键，如果我们需要对 Rowkey 之外的列进行查找，这时候就会有问题。幸好我们可以使用 Solr 来建立二级索引/全文索引充分满足我们的查询需求。
- HGraphDB：HGraphDB是分布式图数据库。依托图关联技术，帮助金融机构有效识别隐藏在网络中的黑色信息，在团伙欺诈、黑中介识别等。
- GeoMesa：目前基于NoSQL数据库的时空数据引擎中功能丰富、社区贡献人数多的开源系统。
- OpenTSDB：基于HBase的分布式的，可伸缩的时间序列数据库。适合做监控系统；譬如收集大规模集群（包括网络设备、操作系统、应用程序）的监控数据并进行存储，查询。

下面简单介绍一下这些组件。  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/63318DA7CF75C0A8B77F9C180D374620.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/6CD5F6BA748732C7C103FCE58C440A40.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/D57635CC616043750B84AEE5FCFB045A.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/05D7FEB7CE8983C173D214BFCD3D3F1B.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/1674016ABB08A79BD7DDB83B8AD8248D.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/2266AAB0068B81069F4F720E0D5556EF.jpg)

  
有了这么多组件，我们都可以干什么呢？来看看 HBase 的典型案例。  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/BE524CDBDE38BB2A40336156DDC5A386.jpg)

  
HBase 在风控场景、车联网/物联网、广告推荐、电子商务等行业有这广泛的使用。下面是四个典型案例的架构，由于图片里有详细的文字，我就不再打出来了。  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/128F70D39DDEE9D430C4A3A06B18612A.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/752CD7C4EE7836EAB6B0613EEB1166FB.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/ABB685A3F66BEF2517250EFC8568FFD1.jpg)

  

![](https://image.z.itpub.net/zitpub.net/JPG/2020-05-27/360E157625BAFC44FBBAEEC29E5F8A3F.jpg)

## 一、介绍HBase

> [Apache](https://link.zhihu.com/?target=https%3A//www.apache.org/) HBase™ is the [Hadoop](https://link.zhihu.com/?target=https%3A//hadoop.apache.org/) database, a distributed, scalable, big data store.  
> HBase is a type of "NoSQL" database.  

Apache HBase 是 Hadoop 数据库，一个分布式、可伸缩的**大数据存储**。

HBase是依赖Hadoop的。为什么HBase能存储海量的数据？**因为HBase是在HDFS的基础之上构建的，HDFS是分布式文件系统**。

## 二、为什么要用HBase

截至到现在，三歪已经学了不少的组件了，比如说分布式搜索引擎「Elasticsearch」、分布式文件系统「HDFS」、分布式消息队列「Kafka」、缓存数据库「Redis」等等...

**能够处理数据的中间件(系统)，这些中间件基本都会有持久化的功能**。为什么？如果某一个时刻挂了，那**还在内存但还没处理完的数据**不就凉了？

Redis有AOF和RDB、Elasticsearch会把数据写到translog然后结合FileSystemCache将数据刷到磁盘中、Kafka本身就是将数据顺序写到磁盘....

这些中间件会实现持久化（像HDFS和MySQL我们本身就用来存储数据的），为什么我们还要用HBase呢？

**虽然没有什么可比性**，但是在学习的时候总会有一个疑问：「既然已学过的系统都有类似的功能了，那为啥我还要去学这个玩意？」

三歪是这样理解的：

- MySQL？MySQL数据库我们是算用得最多了的吧？但众所周知，MySQL是**单机**的。MySQL能存储多少数据，取决于那台服务器的硬盘大小。以现在互联网的数据量，很多时候MySQL是没法存储那么多数据的。
- 比如我这边有个系统，一天就能产生1TB的数据，这数据是不可能存MySQL的。（如此大的量数据，我们现在的做法是先写到Kafka，然后落到Hive中）
- Kafka？Kafka我们主要用来处理消息的（解耦异步削峰）。数据到Kafka，Kafka会将数据持久化到硬盘中，并且Kafka是分布式的（很方便的扩展），理论上Kafka可以存储很大的数据。但是Kafka的数据我们**不会「单独」取出来**。持久化了的数据，最常见的用法就是重新设置offset，做「回溯」操作
- Redis？Redis是缓存数据库，所有的读写都在内存中，速度贼快。AOF/RDB存储的数据都会加载到内存中，Redis不适合存大量的数据（因为内存太贵了！）。
- Elasticsearch？Elasticsearch是一个分布式的搜索引擎，主要用于检索。理论上Elasticsearch也是可以存储海量的数据（毕竟分布式），我们也可以将数据用『索引』来取出来，似乎已经是非常完美的中间件了。
- 但是如果我们的数据**没有经常「检索」的需求**，其实不必放到Elasticsearch，数据写入Elasticsearch需要分词，无疑会浪费资源。
- HDFS？显然HDFS是可以存储海量的数据的，它就是为海量数据而生的。它也有明显的缺点：不支持随机修改，查询效率低，对小文件支持不友好。

> 上面这些技术栈三歪都已经写过文章了。学多了你会发现它们的持久化机制都差不太多，有空再来总结一下。  

文中的开头已经说了，HBase是基于HDFS分布式文件系统去构建的。换句话说，HBase的数据其实也是存储在HDFS上的。那肯定有好奇宝宝就会问：**HDFS和HBase有啥区别阿**？

HDFS是文件系统，而HBase是数据库，其实也没啥可比性。「**你可以把HBase当做是MySQL，把HDFS当做是硬盘。HBase只是一个NoSQL数据库，把数据存在HDFS上**」。

> 数据库是一个以某种**有组织的方式存储的数据集合**。  

扯了这么多，那我们为啥要用HBase呢？HBase在HDFS之上提供了**高并发的随机写和支持实时查询**，这是HDFS不具备的。

我一直都说在学习某一项技术之前首先要了解它能干什么。如果仅仅看上面的”对比“，我们可以发现HBase可以以**低成本**来**存储海量**的数据并且支持高并发随机写和实时查询。

但HBase还有一个特点就是：**存储数据的”结构“可以地非常灵活**（这个下面会讲到，这里如果没接触过HBase的同学可能不知道什么意思）。

## 三、入门HBase

听过HBase的同学可能都听过「列式存储」这个词。我最开始的时候觉得HBase很难理解，就因为它这个「列式存储」我一直理解不了它为什么是「列式」的。

在网上也有很多的博客去讲解什么是「列式」存储，它们会举我们现有的数据库，比如MySQL。存储的结构我们很容易看懂，就是一行一行数据嘛。
![](https://pic4.zhimg.com/80/v2-58697c09af0a80f04af4b2231e6a0813_720w.webp)
转换成所谓的列式存储是什么样的呢？
![](https://pic3.zhimg.com/80/v2-8b4c896c9decec36b89e6b2fee968c6a_720w.webp)

可以很简单的发现，无非就是把**每列抽出来，然后关联上Id**。这个叫列式存储吗？**我在这打个问号**。
转换后的数据**从我的角度来看**，数据还是一行一行的。
这样做有什么好处吗？很明显以前我们一行记录多个属性(列)，有部分的列是空缺的，但是我们还是需要空间去存储。现在把这些列全部拆开，有什么我们就存什么，这样**空间就能被我们充分利用**。
这种形式的数据更像什么？明显是`Key-Value`嘛。那我们该怎么理解HBase所谓的列式存储和`Key-Value`结构呢？走进三歪的小脑袋，一探究竟。
### 3.1 HBase的数据模型

在看HBase数据模型的时候，其实最好还是不要用「关系型数据库」的知识去理解它。

> In HBase, data is stored in tables, which have rows and columns. This is a terminology overlap withrelational databases (RDBMSs), **but this is not a helpful analogy**.  

HBase里边也有表、行和列的概念。

- 表没什么好说的，就是一张表
- 一行数据由**一个行键**和**一个或多个相关的列以及它的值**所组成

好了，现在比较抽象了。在HBase里边，定位一行数据会有一个唯一的值，这个叫做行键(**RowKey**)。而在HBase的列不是我们在关系型数据库所想象中的列。

HBase的列（**Column**）都得归属到列族（**Column Family**）中。在HBase中用列修饰符（**Column Qualifier**）来标识每个列。

**在HBase里边，先有列族，后有列**。

什么是列族？可以简单理解为：列的属性**类别**

什么是列修饰符？**先有列族后有列，在列族下用列修饰符来标识一列**。

还很抽象是不是？三歪来画个图：

![](https://pic3.zhimg.com/80/v2-22e7f861a6a079a558a1fb173dbd8b9e_720w.webp)

我们再放点具体的值去看看，就更加容易看懂了：

![](https://pic1.zhimg.com/80/v2-1f69aa3a03c9666d5b613e5a0a5e70c4_720w.webp)

这张表我们有两个列族，分别是`UserInfo`和`OrderInfo`。在UserInfo下有两个列，分别是`UserInfo:name`和`UserInfo:age`，在`OrderInfo`下有两个列，分别是`OrderInfo:orderId`和`OrderInfo:money`。

`UserInfo:name`的值为：三歪。`UserInfo:age`的值为24。`OrderInfo:orderId`的值为23333。`OrderInfo:money`的值为30。这些数据的主键(RowKey)为1

上面的那个图看起来可能不太好懂，我们再画一个我们比较熟悉的：
![](https://pic3.zhimg.com/80/v2-8f3a2b847d94b936e4c2f251f89ba5c2_720w.webp)

HBase表的每一行中，列的组成都是**灵活**的，**行与行之间的列不需要相同**。如图下：
![](https://pic1.zhimg.com/80/v2-fac481c6ec3efacce0ec3f3c7d127db0_720w.webp)

![](https://pic2.zhimg.com/80/v2-de19abe0e27b1660e3474bd11aaf8d61_720w.webp)

换句话说：**一个列族下可以任意添加列，不受任何限制**

数据写到HBase的时候都会被记录一个**时间戳**，这个时间戳被我们当做一个**版本**。比如说，我们**修改或者删除**某一条的时候，本质上是往里边**新增**一条数据，记录的版本加一了而已。

比如现在我们有一条记录：
![](https://pic3.zhimg.com/80/v2-433053cc5a4dfe5a9d26f9231cdaca5e_720w.webp)

现在要把这条记录的值改为40，实际上就是多添加一条记录，在读的时候按照时间戳**读最新**的记录。在外界「看起来」就是把这条记录改了。
![](https://pic2.zhimg.com/80/v2-a08e56a695e366a580c12514ffe09a21_720w.webp)
### 3.2 HBase 的Key-Value

HBase本质上其实就是`Key-Value`的数据库，上一次我们学`Key-Value`数据库还是Redis呢。那在HBase里边，Key是什么？Value是什么？

我们看一下下面的HBase`Key-Value`结构图：
![](https://pic4.zhimg.com/80/v2-5c2c05489dc0c4af823d26641dc0e477_720w.webp)

Key由RowKey(行键)+ColumnFamily（列族）+Column Qualifier（列修饰符）+TimeStamp（时间戳--版本）+KeyType（类型）组成，而Value就是实际上的值。

对比上面的例子，其实很好理解，因为我们修改一条数据其实上是在原来的基础上增加一个版本的，那我们要**准确定位**一条数据，那就得（RowKey+Column+时间戳）。

KeyType是什么？我们上面只说了「修改」的情况，你们有没有想过，如果要删除一条数据怎么做？实际上也是增加一条记录，只不过我们在KeyType里边设置为“Delete”就可以了。

### 3.3 HBase架构

扯了这么一大堆，已经说了HBase的数据模型和Key-Value了，我们还有一个问题：「为什么经常会有人说HBase是列式存储呢？」

其实HBase更多的是「**列族存储**」，要谈列族存储，就得先了解了解HBase的架构是怎么样的。

我们先来看看HBase的架构图：
![](https://pic2.zhimg.com/80/v2-f9029a2beaf2b07d9ae949013ddca351_720w.webp)

1、**Client**客户端，它提供了访问HBase的接口，并且维护了对应的cache来加速HBase的访问。
2、**Zookeeper**存储HBase的元数据（meta表），无论是读还是写数据，都是去Zookeeper里边拿到meta元数据**告诉给客户端去哪台机器读写数据**
3、**HRegionServer**它是处理客户端的读写请求，负责与HDFS底层交互，是真正干活的节点。
总结大致的流程就是：client请求到Zookeeper，然后Zookeeper返回HRegionServer地址给client，client得到Zookeeper返回的地址去请求HRegionServer，HRegionServer读写数据后返回给client。

![](https://pic1.zhimg.com/80/v2-ef188e98c916eb1b3688261b77773e58_720w.webp)
### 3.4 HRegionServer内部

我们来看下面的图：
![](https://pic3.zhimg.com/80/v2-6f292c56780bad2428f925c473ff9b82_720w.webp)

前面也提到了，HBase可以存储海量的数据，HBase是分布式的。所以我们可以断定：**HBase一张表的数据会分到多台机器上的**。那HBase是怎么切割一张表的数据的呢？用的就是**RowKey**来切分，其实就是表的**横向**切割。

![](https://pic4.zhimg.com/80/v2-33dd3cf742d701f91f9e3f2ccd74230b_720w.webp)
说白了就是一个**HRegion**上，存储HBase表的一部分数据。
![](https://pic4.zhimg.com/80/v2-623be466ec8ea2ad9c48b79f1d842023_720w.webp)

HRegion下面有**Store**，那Store是什么呢？我们前面也说过，一个HBase表首先要定义**列族**，然后列是在列族之下的，列可以随意添加。

**一个列族的数据是存储**在一起的，所以一个列族的数据是存储在一个Store里边的。

看到这里，其实我们可以认为HBase是基于列族存储的（毕竟物理存储，一个列族是存储到同一个Store里的）
![](https://pic3.zhimg.com/80/v2-acbb43541256b6076cd9939035a718f6_720w.webp)

Store里边有啥？有`Mem Store、Store File、HFile`，我们再来看看里边都代表啥含义。
![](https://pic3.zhimg.com/80/v2-bb0152a17990f0572c97b7076d4a1a62_720w.webp)

HBase在写数据的时候，会先写到`Mem Store`，当`MemStore`超过一定阈值，就会将内存中的数据刷写到硬盘上，形成**StoreFile**，而`StoreFile`底层是以`HFile`的格式保存，`HFile`是HBase中`KeyValue`数据的存储格式。

所以说：`Mem Store`我们可以理解为内存 buffer，`HFile`是HBase实际存储的数据格式，而`StoreFile`只是HBase里的一个名字。

回到HRegionServer上，我们还漏了一块，就是`HLog`。

![](https://pic4.zhimg.com/80/v2-a2bb5cd2216cb2b98d181d5eac1ca87f_720w.webp)
这里其实特别好理解了，我们写数据的时候是先写到内存的，为了防止机器宕机，内存的数据没刷到磁盘中就挂了。我们在写`Mem store`的时候还会写一份`HLog`。

这个`HLog`是顺序写到磁盘的，所以速度还是挺快的（是不是有似曾相似的感觉）...

稍微总结一把：

- HRegionServer是真正干活的机器（用于与hdfs交互），我们HBase表用RowKey来横向切分表
- HRegion里边会有多个Store，每个Store其实就是一个列族的数据（所以我们可以说HBase是基于列族存储的）
- Store里边有Men Store和StoreFile(HFile)，其实就是先走一层内存，然后再刷到磁盘的结构

### 3.5 被遗忘的HMaster

我们在上面的图会看到有个**Hmaster**，它在HBase的架构中承担一种什么样的角色呢？**读写请求都没经过Hmaster呀**。

![](https://pic2.zhimg.com/80/v2-b47bf4ee838b0356d29415322eff9061_720w.webp)

那HMaster在HBase里承担什么样的角色呢？？

> `HMaster` is the implementation of the Master Server. The Master server is responsible for monitoring all RegionServer instances in the cluster, and is the interface for all metadata changes.  

HMaster会处理 HRegion 的分配或转移。如果我们HRegion的数据量太大的话，HMaster会对拆分后的Region**重新分配RegionServer**。（如果发现失效的HRegion，也会将失效的HRegion分配到正常的HRegionServer中）

HMaster会处理元数据的变更和监控RegionServer的状态。
## 四、RowKey的设计

到这里，我们已经知道RowKey是什么了。不难理解的是，我们肯定是要保证RowKey是**唯一**的，毕竟它是行键，有了它我们才可以唯一标识一条数据的。

在HBase里边提供了三种的查询方式：

1. 全局扫描
2. 根据一个RowKey进行查询
3. 根据RowKey过滤的**范围**查询

### 4.1 根据一个RowKey查询

首先我们要知道的是RowKey是会按**字典序**排序的，我们HBase表会用RowKey来横向切分表。

无论是读和写我们都是用RowKey去定位到HRegion，然后找到HRegionServer。这里有一个很关键的问题：**那我怎么知道这个RowKey是在这个HRegion上的**？

HRegion上有两个很重要的属性：`start-key`和`end-key`。

我们在定位HRegionServer的时候，实际上就是定位我们这个RowKey在不在这个HRegion的`start-key`和`end-key`范围之内，如果在，说明我们就找到了。

这个时候会带来一个问题：由于我们的RowKey是以字典序排序的，如果我们对RowKey没有做任何处理，那就有可能存在**热点数据**的问题。

举个例子，现在我们的RowKey如下：

```java
java3y111
java3y222
java3y333
java3y444
java3y555
aaa
bbb
java3y777
java3y666
java3y...
```

`Java3yxxx`开头的RowKey很多，而其他的RowKey很少。如果我们有多个`HRegion`的话，那么存储`Java3yxxx`的HRegion的数据量是最大的，而分配给其他的HRegion数量是很少的。

关键是我们的查询也几乎都是以`java3yxxx`的数据去查，这会导致某部分数据会集中在某台HRegionServer上存储以及查询，而其他的HRegionServer却很空闲。

如果是这种情况，我们要做的是什么？对**RowKey散列**就好了，那分配到HRegion的时候就比较均匀，少了热点的问题。

> HBase优化手册：  
> 建表申请时的预分区设置，对于经常使用HBase的小伙伴来说,HBase管理平台里申请HBase表流程必然不陌生了。  
> '**给定split的RowKey组例如:aaaaa,bbbbb,ccccc;或给定例如:startKey=00000000,endKey=xxxxxxxx,regionsNum=x**'  
> **第一种方式**:  
> 是自己指定RowKey的分割点来划分region个数.比如有一组数据RowKey为\[1,2,3,4,5,6,7],此时给定split RowKey是3,6,那么就会划分为\[1,3),\[3,6),\[6,7)的三个初始region了.如果对于RowKey的组成及数据分布非常清楚的话,可以使用这种方式精确预分区.  
>   
> **第二种方式** :  
> 如果只是知道RowKey的组成大致的范围,可以选用这种方式让集群来均衡预分区,设定始末的RowKey,以及根据数据量给定大致的region数,一般建议region数最多不要超过集群的rs节点数,过多region数不但不能增加表访问性能,反而会增加master节点压力.如果给定始末RowKey范围与实际偏差较大的话,还是比较容易产生数据热点问题.  
>   
> **最后:生成RowKey时,尽量进行加盐或者哈希的处理,这样很大程度上可以缓解数据热点问题.**  

### 4.2根据RowKey范围查询

上面的情况是针对通过RowKey单个查询的业务的，如果我们是根据RowKey范围查询的，那没必要上面那样做。

HBase将RowKey设计为字典序排序，如果不做限制，那很可能类似的RowKey存储在同一个HRegion中。那我正好有这个场景上的业务，那我查询的时候不是快多了吗？**在同一个HRegion就可以拿到我想要的数据了**。

举个例子：我们会间隔几秒就采集直播间热度，将这份数据写到HBase中，然后业务方经常要把主播的一段时间内的热度给查询出来。

我设计好的RowKey，将该主播的一段时间内的热度都写到同一个HRegion上，拉取的时候只要访问一个HRegionServer就可以得到全部我想要的数据了，那查询的速度就快很多。

## 最后

最后三歪再来带着大家回顾一下这篇文章写了什么：

1. HBase是一个NoSQL数据库，一般我们用它来存储海量的数据（因为它基于HDFS分布式文件系统上构建的）
2. HBase的一行记录由一个RowKey和一个或多个的列以及它的值所组成。先有列族后有列，列可以随意添加。
3. HBase的增删改记录都有「版本」，默认以时间戳的方式实现。
4. RowKey的设计如果没有特殊的业务性，最好设计为散列的，这样避免热点数据分布在同一个HRegionServer中。
5. HBase的读写都经过Zookeeper去拉取meta数据，定位到对应的HRegion，然后找到HRegionServer

# Reference
https://z.itpub.net/article/detail/41579557F505CBBF635206E8B3F3E910
https://zhuanlan.zhihu.com/p/145551967