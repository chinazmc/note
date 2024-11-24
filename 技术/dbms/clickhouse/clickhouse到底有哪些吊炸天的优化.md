### **概述**

查询的本质是什么？

所谓查询，说白了，就是在一堆有序或者无序的数据中，按照一定的条件，筛选出我们期望的[数据集](https://so.csdn.net/so/search?q=%E6%95%B0%E6%8D%AE%E9%9B%86&spm=1001.2101.3001.7020)。

[ClickHouse](https://so.csdn.net/so/search?q=ClickHouse&spm=1001.2101.3001.7020)以快著称。它到底有多快？它又为什么快？说到底，还是绕不开它的底层存储逻辑。本文以CllickHouse使用最为广泛的MergeTree引擎为例，来揭开[ClickHouse性能](https://www.zhihu.com/search?q=ClickHouse%E6%80%A7%E8%83%BD&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3290601554%7D "ClickHouse性能")的神秘面纱。

我们知道，查有序的数据一定比查无序的数据速度要快。不论是从磁盘里查，还是直接从内存里查，均是同理。

写数据则相反，无序的写比有序的写，速度要更快。但是为了查询有序，我们就不得不在插入的时候对数据进行排序。那么怎么排序才合理？特别是海量数据的时候，如何才能保证这种排序性能是[幂等](https://so.csdn.net/so/search?q=%E5%B9%82%E7%AD%89&spm=1001.2101.3001.7020)的？尤其是当数据一批一批的插入的时候，有没有一种比较稳定的排序算法来完成这一工作？其实我们很快就能想到类似归并排序的思想。我们先对插入的每一批数据进行排序，然后再将不同批次插入的有序数据进行merge，并保证merge之后的数据有序就行了。那么一下子我们就联系到了MergeTree的底层核心思想。

### **MergeTree存储原理**

为什么在聊性能优化之前，我们会先讨论MergeTree的存储原理。因为一切的所谓调优，都是建立在存储原理之上的。只有搞明白了数据存储的原理，它在内存里是如何排布的，才能有的放驶，做出针对性的优化。而不是在网上随便找到一篇文章就奉为圭臬，无脑使用，如果运用场景不对，不仅没有任何优化效果，相反会带来可怕的负优化。

#### **MergeTree是在merge什么**

MergeTree的核心是Merge，合并什么？它是将同一个partition内的多个part进行合并。很多初学者可能弄不清part和partition之间的区别。

part是ClickHouse为了加速插入，引入的一个概念。在ClickHouse中，每个线程的每次插入操作，都会产生一个part，这样如果我们开启了多线程来插入（可以通过[max_insert_threads](https://www.zhihu.com/search?q=max_insert_threads&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3290601554%7D "max_insert_threads")来设置），正常情况下，每次insert动作，肯定是处于不同的线程中的，因此每次插入，至少都会产生一个part。这样并发写入，且各个part之间的数据互不影响，所以可以做到快速的插入。

那么这样的快速插入 必然会带来一个问题：查询怎么办？

有可能我第一次插入的和第二次插入的数据有相近的，甚至数据完全相同都是有可能的。这时候如果直接去查part，那么必然需要扫描所有的part，这时候必然会成为性能的拖累（即使part已经是有序的）。我们可以简单举一个例子如下：

![](https://i-blog.csdnimg.cn/blog_migrate/2ada45b2d27b3b6d72acf13d9e68b6d6.png)

上面有三个part，现在我的诉求是查询出其中>=3且<=5的所有数据，虽然在各个part内部，数据本身是有序的，但是这三个part很不凑巧地都包含了3~5的范围，因此必须每个part都扫描一遍，而实际上，part3是没有任何符合条件的数据的，所以这个扫描其实是无用的操作。如果part数量特别多呢？如果查询的条件再苛刻些呢？那么对整个查询的性能影响已经到了无法忽略的地步。

这时候就体现出merge的作用了。所谓merge，他就是将各个满足同一分区范围的数据合并成一个更大的part。假设上述三个part都处于同一个partition内，那么最终效果如下：

![](https://i-blog.csdnimg.cn/blog_migrate/e7e5e597450c87765af135898c893040.png)

合并完之后，数据成了上面的样子 ，这时候查询就非常简单了。直接二分查找就可以。

那么接下来人们肯定关心的一个话题是，什么时候去merge？怎么去merge？

首先，merge肯定是在后台进行的，而不是数据一插入就立即merge。一种比较普遍的说法是数据插入后8-10分钟后会自行merge。这种说法大抵正确，但不全面。

merge的时机的确是不确定的。它既不能每插一批数据数据就进行一次merge，那样势必会拖慢插入的速度。也不能一直不merge，那样就是前面说的问题，查询开销太大。为了寻找这个平衡点，在ClickHouse里，Merge的算法有一个核心参数，叫做base，它决定了part要不要参与merge。为了讲清楚这个base参数，我们同样来举一个例子。

下面我们以base = 3来举例。

![](https://i-blog.csdnimg.cn/blog_migrate/4a0cca6e663ec96227f15a9a09721295.png)

如上图所示， base = 3，代表每3个part 合并成更高一级的part， 我们叫它Level1， 然后3个Level1进一步合并成Level2， 从逻辑层面看上去，就像一棵树。这也就是MergeTree的Tree的由来。

那么ClickHouse里base是如何取值的呢？在ClickHouse里，base初始取值为5，但并不总是一成不变的，如果最小的part长时间得不到合并，那就说明base取值过大，它会适当调小；如果参与merge的part的总大小很小，那么base也会适当减小；当然，part总数本来就比较少，base也会适当减小。

为什么base取值和part的大小还有关系呢？因为并不是part数达到了这个base就会合并，它只是一个准入条件，而不是一个充分条件。他真正merge的时机是要判断 `总part大小 / 最大part大小 >= base` 才会去合并。

所以在某些情况下，可能存在有些part永远得不到merge的情况。（思考下，什么场景会出现？又该怎么解决？）

#### **partition**

上面讲了半天的merge，主要还是针对part而言的。那么parttition又是什么呢？它又是由什么决定的？

partition是一个逻辑上的概念，不像part，是有实物可以看得见摸得着的。我们在建表的时候，可以指定partition by的字段的。由于是使用者自己指定，那么，就有很大的可操作空间。可以说，partition是否合理，很可能会影响到查询和写入的性能。

对于ClickHouse来说，每个part，实际上都是一个文件夹，该文件夹下存储了具体的列存数据，列越多，那么文件也就越多。

![](https://i-blog.csdnimg.cn/blog_migrate/22f0a271a9609cfb36259560f6d9714e.png)

![](https://i-blog.csdnimg.cn/blog_migrate/8f87ce8d1d5dddbf0ba8d83e99a0221f.png)

part越多，意味着文件越多，也就意味着merge需要更多次才能完成。从上图可以看到，有些合并甚至达到了571次。合并本身是需要消耗CPU和内存的，资源被merge占掉了一部分，那么查询和写入的性能就会受到影响。而且文件数目过多，对文件系统本身来说也是沉重的负担。最重要的，merge不是时时刻刻都在发生的，所以当一个查询语句过来，虽然有部分数据可以在合并后的part中找，但未来得及合并部分的数据，还是要在小part里找。那么part数太多，查询自然就会越慢。

以上是part数太多的坏处。

那么怎样会导致part数太多呢？一个最显而易见的场景就是小批量频繁插入，每次插入都会产生一个part。所以clickhouse的写入，是提倡大批次插入的。测试表明，单批次插入数据上万甚至上百万都是没有问题的。

但是单批次插入数据过多的话，也需要注意一些问题。

首先就是实时性，如果一定要等数据量达到某一个数值才插入，那么当数据源生产较慢时，肯定会带来延迟。但是这个有解决方案，那就是设定一个超时时间，如果超时时间到了，即使没有积攒够一个批次，也插入到数据库中。

第二个问题是大批量插入必然会带来大内存消耗。

第三个问题是如果数据本身不规范，那么大批次数据插入很容易带来"TOO MANY PARTITIONS"的报错。它的意思是说单批次插入的数据中包含了过多的partition（一般是100个）。这个限制也是出于性能考虑，因为即使是在一次插入，它的数据处于不同的partition，那么仍然是落在不同的文件夹里的，如果同时写入的文件夹数过多，那么必然会带来性能的损耗。也就失去了先写part后merge的意义了。

前面提到过，partition中的数据实际上是有序的，那么也就是说，如果查询的语句中带有partition by的字段，可以非常快捷地根据排序字段做二分查找。

如下的例子，现在有三个partition，分别为20231101，20231102，和[20231103](https://www.zhihu.com/search?q=20231103&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3290601554%7D "20231103")， 我现在需要查2023-11-03 12：00：00 到 2023-11-03 13：00：00这一个小时的数据，那么前面两个partition其实是可以看都不看，直接跳过的，然后在20231103的分区内做二分查找，就能很快定位到要查询的数据的分布。这样就不需要进行全表扫描了。

![](https://i-blog.csdnimg.cn/blog_migrate/adb1d2542c037d9da9a6cd44e24ea78f.png)

#### **sorting key 和 primary key**

很多人可能都不知道，MergeTree是有主键的。不过它所谓的主键并不是代表数据唯一，而是作为一个排序的手段。

primary key可以显式指定，也可以隐式指定。

隐式指定就是不声明primary key的字段，那么clickhouse会默认拿order by的字段作为primary key。

但是！

primary key和[sorting key](https://www.zhihu.com/search?q=sorting%20key&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3290601554%7D "sorting key")还是有区别的。

从概念上来说，primary key是作为索引的，sorting key是用来排序的。

primary key一定是sorting key，但sorting key不一定是primary key。

在排序的时候，primary key一定要放在靠前的位置。

![](https://i-blog.csdnimg.cn/blog_migrate/2f0b6ca09eb7ef888656f0974c94f5fa.png)

那么，这二者在使用上有什么区别呢？

正常情况下我们是不需要考虑这二者的区别的。但是对于聚合引擎，比如SummingMergeTree和AggregateingMergetree， 分开设置primary key和order by将会有助于优化。为了比较具象的说清楚这二者的关系，下面举个例子：

> 查询条件： WHERE A  
> 聚合条件： GROUP BY (A, B, C)

此时，这样指定主键和排序字段会比较高效：

> PRIMARY KEY(A)  
> ORDER BY (A, B, C)

排序键是为了上下文一致性，和最大化的压缩比例。而且group by的维度字段在物理上靠的更近的话，查询起来也更高效，这是肯定的。

主键是需要占磁盘空间的。而且基于主键的索引在查询时会被加载到内存当中，如果主键过多，那么内存效率相对来说会非常低下。因此，我们只需要把过滤的列放到主键索引里就可以了，它主要用来快速筛选，缩小数据范围。

还有一点比较重要。现在我们order by (A, B, C )三个维度，假设现在需要新插入一个维度D，那么这个时候，我们想要修改表，变成 order by (A, B, C, D)是非常轻量级的。因为A,B,C本身就是排好序的，所以D一开始肯定是没有数据的，那么更新的成本其实是0。如果我们把主键索引和排序键设置成一样的会怎么样？我们需要修改主键索引的文件，相当于重新构建整个索引文件了，这个成本之大，其实是不可想象的。

所以将这二者分开的最大意义，是为了给修改order by腾出一定的空间。

> 思考： ReplacingMergeTree去重，根据的是order by，还是primary key？

注意： ReplacingMergeTree去重是根据order by，而不是primary key。

![](https://i-blog.csdnimg.cn/blog_migrate/c2fa5c183ee76eee284c4480bdce454e.png)

#### **稀疏索引**

我们前面提到的primary key， 和我们通常关系型数据库中见到的主键索引有所不同，它是属于稀疏索引。稀疏索引是ClickHouse中一个非常重要的概念。它有别于MySQL这些OLTP数据库的行级索引。行级索引就是一行数据一条索引，每个索引都可以精确定位到一条数据。所以行级索引的跳数通常等于该索引字段去重后的个数。

但是稀疏索引不同，它的跳数由索引粒度决定，即多条数据才会去创建一个索引。MergeTree引擎在建表的时候，默认的索引粒度是8192，也就是说8192条数据才会创建一个索引（当然不是绝对的，它这个粒度大小是自适应的，可以动态调整大小）。

![](https://i-blog.csdnimg.cn/blog_migrate/f2a9968e755c2e3818108a5f724cf6a4.png)

上图描述了根据主键稀疏索引去查找数据的过程，在每个part目录下，会有一个primary.idx文件，存储的是索引信息，根据稀疏索引，可以在col.mrk2文件中找到实际数据在bin文件中的偏移量，然后根据偏移量就可以定位到数据处于哪个颗粒中。当然这个地方画得比较简略，实际逻辑比这个要复杂。

颗粒，是clickhouse查询的最小单位。

稀疏索引的前提是数据必须是有序的。稀疏索引的存在，使得索引的存储成本极小，8192万行数据，只需要1万个索引，这对存储海量的数据是一个巨大的优化。又由于MergeTree引擎的特性，数据本身就是有序存储的，所以查询的效率自然而然就非常快了。

正常情况下，主键索引的查找逻辑是使用二分查找。但也不能一概而论，对于联合主键索引来说，需要看联合索引的顺序。

通过trace日志查看查询执行的逻辑，可以知道，第一关键字列使用的是二分查找算法，而非第一关键字列，使用的则是通用排除搜索算法。通用排除算法的性能要比二分查找低很多，因为它进行的是全表扫描。也就是说，非第一关键字列放在主键索引里，对该字段并不能起到优化作用。

所以，联合主键索引的创建顺序，是和查询场景息息相关的。

那么这种情况应该怎么解决呢？这里先留个悬念，大家也可以先思考一下。

### **建表优化**

#### **合理分区**

通过测试发现，单表的分区数量最好不要超过1000个，否则会对写入性能有很大影响。

分区的选择可参考性有很多。假如数据存储的时间较长，比如存三年甚至更久，那么按天分区或者按星期分区或许比较合理。但是如果查询的场景大部分是根据天去查询，那么按天分区也许会比按星期会更好一些。

当然，还要考虑到数据规模因素，如果单日数据增量太大，比如APM的span数据，有些规模比较大的企业，可以达到秒增百万，此时如果选择按天分区的话，那么单表的单个分区可能就会达到数十TB。而实际上的查询场景可能比较频繁的还是查询15分钟，或者1个 小时的数据，那么，这时候就可以考虑按小时进行分区了。

#### **合理选择shardingkey**

单节点的clickhouse服务是没有shardingkey的概念的。我们谈论shardingkey的时候，其对象一定是clickhouse集群。

起初，设计shardingkey的意图，仅仅是为了负载均衡，尽量使得数据均匀地分布在把不同的shard内。通过这种方式，充分利用服务器的资源，从而使查询效率最大化。

![](https://i-blog.csdnimg.cn/blog_migrate/1688fb481feec64b223184efeaa0de64.png)

上例中，分片1有10TB数据，分片2有100G数据，分片3则是800G数据，整体分布是不均匀的。

假设此时一个分布式查询语句过来，真正落到各个节点上执行的，查询的还是本地表，那么由于数据规模的不同，需要消耗的内存和CPU也是不同的。这在服务器资源充裕的情况下可能影响不大，可一旦系统资源本身就吃紧，那么MPP架构的短板效应就暴露无疑。性能最差的那个节点的性能，势必会拖累整个查询的性能。因此，我们需要通过shardingkey来尽可能地使数据均匀地落在不同的分片。

为什么需要shardingkey来做均衡，而不是通过round-robin的方式来写？

我们知道，clickhouse数据的插入是以batch为单位的，只要每个batch相对平均，那么round-robin之后最终得到的每个分片上的数据，始终是相对均衡的。

但是这种方式无法保证业务上的逻辑，它很可能会把有业务关联的数据分拆到了不同的shard里，这对于查询，往往是非常致命的。

接下来我们来考虑下面这一种场景：

![](https://i-blog.csdnimg.cn/blog_migrate/dcf60fcf724d7ac9f9939ffe1723ed8b.png)

我们现在有两张表，并要对这两张表进行关联查询，虽然从数据量上来看，这两张表在shard1和shard2上面分布是均匀的，但是要做join查询，如果本地join，那么得不到我们想要的数据结果，所以只能用global join去查分布式表。这也就意味着，需要先将所有数据先查出来，然后在汇总节点上去做join，这对于查询来说，开销是巨大的。

而如果这时候，我们使用id作为shardingkey， 算法如下：

```cobol
shardNum = id % clickhouse_shards + 1
```

当计算出来的shardNum和分片编号相等时，将数据存入对应的分片，那么上述数据就可以重新排布如下：

![](https://i-blog.csdnimg.cn/blog_migrate/13de8eb7728540ab98fd3183eb9d3cb1.png)

那么这时候去查询，实际上使用本地join得到结果，再汇总出去，性能提升将会是巨大的。

当然，这种选择shardingkey的方式，也是有些需要去考虑的点的：

- 如果集群内的某一个分片挂了怎么办？按照这种shardingkey的逻辑，这部分数据不可能写到其他的分片，那么会导致本来该写到该分片的数据永远block住，无法写入，除非集群被修复。
- 如果集群扩容或者缩容了，那么由于clickhouse_shards发生了变化，计算出来的shardNum和之前的算法算出来是不一致的，那么就需要一个rehash的过程，否则数据就会乱掉。
- 如果shardingkey选取不合理，也可能造成数据分布不均匀的情况。

- [clickhouse_sinker](https://link.zhihu.com/?target=https%3A//github.com/housepower/clickhouse_sinker "clickhouse_sinker")提供了shardingkey的选择方案，如果指定的shardingkey本身就是一个数值类型，那么直接取余计算。如果指定的shardingkey是一个字符串类型，那么会先通过xxhash64得到一个值，再拿该值去取余计算。
- [ckman](https://link.zhihu.com/?target=https%3A//github.com/housepower/ckman "ckman")提供了按照shardingkey做rebalance的功能，它可以在集群扩缩容之后对已有数据进行rehash，从而保证与后写入的数据保持hash算法一致。

![](https://i-blog.csdnimg.cn/blog_migrate/2ab0396c91aa92a68e0c583bce87185c.png)

#### **合理选择 order by字段**

order by的字段怎么选取，是由具体的查询SQL来决定的。

不同的查询场景，使用不同的order by字段，得到的效果可能差别非常大。

这就要求我们在建表之初，就要考虑好具体的查询场景，从而谨慎决定order by的字段（包括顺序）。

前面提到过，如果不显式指定主键字段，那么order by的字段就会被当成主键字段。

前面还提到过，对于联合主键，第一关键列会用到二分查找，而非第一关键列使用的则是全表扫描的通用排除搜索算法。这也就意味着，有些字段即使放在order by里，其实也是不适合作为过滤条件的。

那通用排除算法就毫无用处吗？

其实也不是的，它对于低基数的排序列，还是有很大的加速作用的。

![](https://i-blog.csdnimg.cn/blog_migrate/07c5f5286f082260adf312e923abb333.png)

上例中，cardinality_IsRobot就是一个低基数的列，它在800w数据中一共只有4个值。

这种如何理解呢？

![](https://i-blog.csdnimg.cn/blog_migrate/b7abf03511a55c5cd06bcf5b1d685021.png)

在这里例子中，col1具有低基数。虚线框表示的是granule，那么相同的col1可能分布在不同的颗粒上。这时候，如果我们按照 col1 = 'v1' and col2 = 'w3'去做查询。

假设order by 顺序为：

```cobol
ORDER BY (col1, col2);
```

此时col1是第一关键列，那么按照col1做二分查找，反而一个颗粒都过滤不掉，到了col2, 需要做全表扫描。我们把整个过程分成两个阶段来看：

> 第一阶段： col1做二分查找， 4个granule，过滤完，还是4个granule  
> 第二阶段： col2做通用排除搜索，至少得扫描到第三个granule的时候，才能停止（因为col2也是排过序的，所以看到有w4，说明后面肯定不会有w3了）  
> 扫描的granule一共为 4 + 3 = 7个轮次。

如果我们改一下order by的顺序：

```cobol
ORDER BY (col2, col1);
```

我们仍然当成两个阶段来分析：

> 第一阶段，根据col2做二分查找， 4个granule立即缩减到2个（索引2和索引3）  
> 第二阶段，根据col1做通用排除搜索，仍然需要扫描2个granule  
> 扫描的granule一共为 2 + 2 = 4个轮次。

上面通过举例的方式讲解了order by的顺序对查询效率的影响。但实际使用时场景肯定要比这复杂得多，因为不太可能所有设计者一开始就能预料到所有的查询场景，随着业务的不断进化，SQL的查询场景也是越来越复杂，那么这个时候，为了提升性能，势必要修改order by的字段（或者顺序），这对于MergeTree来说，是一个非常重甚至是危险的操作，它可能导致数据在一定的时间内无法对外提供服务。

此时我们有几种解决方案：

- 额外建一张表，使用新的order by字段
    - 这种方案的弊端在于需要冗余一倍以上的存储空间，而且表的数据需要双写，一旦有哪一个环节出问题，那么得到的结果就不是我们期望的。而且要修改表，需要同步修改两张表
    - 对于用户不透明，在场景A需要查A表，在场景B需要查B表，对于使用者来说，十分不友好

- 创建物化视图，在物化视图里使用新的排序逻辑
    - 同样需要占用一定的额外空间，但相比于重新建一张表，会好很多，表的schema修改也需要同步进行，好处是数据无需双写，只需要写一份数据就行了
    - 对用户同样不透明，场景A需要查A表，场景B需要查物化视图

- 创建projection
    - 最轻量级，所占的存储空间也是最少的，projection本身就是属于表的一部分，所以无需双写，也不用考虑同步修改表的问题
    - 对用户透明，projection内部会自动选择最优的版本进行查询。
    - projection的最大问题是，不那么容易被命中，如果WHERE条件中加入了projection中未声明的字段，那么projection并不会走到。

#### **二级索引**

所谓二级索引，是MergeTree在主键索引之外提供的另一类索引类型。它是作用在每个granule之上的，所以是先有主键索引，然后才能谈论二级索引的使用。

官方提供了很多中二级索引类型，通过这些索引，我们可以更快速地对整个颗粒做统计、过滤、去重等操作。

下图给出了常见的二级索引在SQL查询加速优化方面的范围列表。

![](https://i-blog.csdnimg.cn/blog_migrate/29cd8af572cd46a6abfec7aada8317c4.png)

二级索引虽然比较多，但其中使用比较广泛的场景是全文搜索。

首先给出结论，clickhouse的全文搜索能力，与ES相比，还是有很大差距的。

clickhouse使用二级索引做全文检索的核心思想就是：利用[布隆过滤器](https://www.zhihu.com/search?q=%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3290601554%7D "布隆过滤器")，快速筛选掉大部分不符合条件的颗粒，然后在符合条件的少部分颗粒中进行查询。

布隆过滤器的原理就不多做解释了。它提供了一个长度，hash函数的个数，以及一个种子值。通过布隆过滤器可以判断一个字段值一定不存在，而不能判断一个值一定存在。

![](https://i-blog.csdnimg.cn/blog_migrate/e586bae546b8fdc4356c2a9daeedc81d.png)

因此，对于IN, LIKE的查询有优化效果，但对于 NOT IN， NOT LIKE则无优化效果。

要想减小假阳性产生的概率，那么bitmap的长度要足够大，hash函数的个数决定了计算出来的hash值重复概率，hash函数越多，计算出来的hash值重复的可能性就越小，但是占用的bitmap空间也会越多，当bitmap长度一定，hash函数太多，那么极可能最终bitmap里全是1，那么对于过滤就毫无益处了。

bitmap足够大，hash函数足够多，理论上是可以保证足够小的假阳率的。但是二级索引也是需要占用存储空间的，而且hash函数越多，计算也就越复杂，耗费的CPU也会越多。因此，二级索引说白了，其实就是拿空间换时间。

我们说回全文检索。比较适合做全文搜索的二级索引有三个，分别是tokenbf_v1, ngrambf_v1， 以及inverted。我在[ClickHouse利用跳数索引加速模糊查询](https://zhuanlan.zhihu.com/p/664944219 "ClickHouse利用跳数索引加速模糊查询")一文中详细介绍了三者的区别。概括起来就是：

||tokenbf_v1|ngrambf_v1|inverted|
|---|---|---|---|
|分词方式|按非字母数字的特殊字符做分词|按长度做分词|按非字母数字的特殊字符做分词|
|索引原理|布隆过滤器|布隆过滤器|倒排索引|
|特殊字符过滤|不支持|支持|不支持|
|汉字过滤|不支持|支持|不支持|
|生产可用|是|是|实验性质，不推荐|
|占用存储空间|适中|和n长度有关，n越大，占用空间越少，n越小，占用空间越大|小|

tokenbf_v1：

![](https://i-blog.csdnimg.cn/blog_migrate/e39357e57acda60a344f460b2de35a51.png)

ngrambf_v1：

![](https://i-blog.csdnimg.cn/blog_migrate/1a43a0c34548ab72c465fdbbc71bebb0.png)

inverted:

![](https://i-blog.csdnimg.cn/blog_migrate/a0f738941c4f4e202789bcafbc6654fc.png)

个人比较推荐使用ngrambf_v1去创建二级索引。但是ngrambf_v1的长度n的选取非常讲究。

如果搜索的关键字的长度小于n，那么这个索引是不会被命中的。比如n = 10, 但是条件为where value like '%test%'，那么这个索引边并不会生效。所以这就决定了n的值不是越大越好。测试表明，n的值也不是越小越好。因为当n足够小时，那么拆分出来的token就会越多，填充到bitmap的hash值也就会越多，那么假阳率也就越高，这时候反而什么都过滤不出来。

另有一点需要注意的是，二级索引能够提升查询效率的前提是，主键索引要创建得合理。

假设有如下场景：我们需要根据visitor_id = '1001'来做过滤。

![](https://i-blog.csdnimg.cn/blog_migrate/b99159609d7e2ccad2805977cc101cbb.png)

布隆过滤器设置得再好，它也是针对颗粒而言的，在上例中，1001这个数，本身在所有的颗粒中都存在，所以即便是布隆过滤器的假阳率接近0，也是无法过滤掉任何一个颗粒的。但实际上呢，可能1001只是颗粒中8192行里的某一条或几条数据。那么这种场景，二级索引也是没有任何优化作用的。

#### **利用物化视图和投影**

前面我们已经提到过物化视图和投影在修改order by上的场景应用。

但物化视图和投影的更大用处是用于聚合查询的预聚合。projection相比物化视图来说更加轻量级，但从效果上来说，毕竟还是要比物化视图稍微差一点。这个很好理解，projection操作的毕竟还是原表，而物化视图把数据完全独立出来了，查询的时候是不需要在原表进行搜索的。

关于物化视图和projection的一些使用技巧。

举个交易串联的场景的例子来说明一下。交易串联场景，就是按照交易ID进行预聚合。我们这里使用物化视图来演示。

先创建本地表和分布式表：
```sql
CREATE TABLE IF NOT EXISTS tran_log ON CLUSTER abc (
    ts DateTime64(3),
    tranId String,
    serviceCode String,
    message String
) ENGINE=ReplicatedMergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (ts, tranId, serviceCode);
​
CREATE TABLE dist_tran_log ON CLUSTER abc AS tran_log ENGINE = Distributed(abc, default, tran_log, rand());
```

这里提供两种创建物化视图的风格。

风格1：物化视图直接与本地表关联：
```sql
CREATE MATERIALIZED VIEW mv_tran ON CLUSTER abc
ENGINE = ReplicatedAggregatingMergeTree() ORDER BY (tranId)
POPULATE AS SELECT tranId, uniqExactState(ts) as logCount FROM tran_log GROUP BY tranId;
​
CREATE TABLE dist_mv_tran ON CLUSTER abc AS mv_tran ENGINE = Distributed(abc, default, mv_tran, rand());
```

风格2：创建一张聚合表，物化视图作为聚合表和本地表的桥接：
```sql
CREATE TABLE agg_tran ON CLUSTER abc (
 tranId String,
 logCount uniqExactState(DateTime64(3))
) ENGINE=ReplicatedReplacingMergeTree()
ORDER BY transId;
​
CREATE TABLE dist_agg_tran ON CLUSTER abc AS agg_tran ENGINE = Distributed(abc, default, agg_tran, rand());
​
CREATE MATERIALIZED VIEW mv_tran ON CLUSTER abc
TO agg_tran AS
SELECT tranId, uniqExactState(ts) as logCount FROM tran_log GROUP BY tranId;
​
CREATE TABLE dist_mv_tran ON CLUSTER abc AS mv_tran ENGINE = Distributed(abc, default, mv_tran, rand());
```

风格2相比于风格1，优势在于可以在运行期暂停和重启与原始表的连接关系。

华为云ClickHouse提供的[Adaptive MV](https://link.zhihu.com/?target=https%3A//support.huaweicloud.com/intl/en-us/eu-west-0-cmpntguide-lts-mrs/mrs_01_24287.html "Adaptive MV") 提供了类似Projection的透明物化视图。

我们准备一些数据，在各个shard上执行（故意引入一些重复数据）：
```sql
INSERT INTO tran_log VALUES 
('2000-01-01 00:00:00.000', '001', 'open', 'open browser') 
('2000-01-01 00:00:01.000', '001', 'buy', 'buy a ticket') 
('2000-01-01 00:00:02.000', '001', 'close', 'close browser') 
('2000-01-01 00:00:00.000', '002', 'open', 'open browser')
('2000-01-01 00:00:01.000', '002', 'buy', 'buy a ticket')
('2000-01-01 00:00:02.000', '002', 'close', 'close browser')
('2000-01-01 00:00:00.000', '003', 'open', 'open browser') 
('2000-01-01 00:00:01.000', '003', 'buy', 'buy a ticket') 
('2000-01-01 00:00:02.000', '003', 'close', 'close browser');
```

我们使用分布式表聚合查询，和直接查询物化视图，得到的结果均一致：
```sql
SELECT tranId, uniqExact(ts) AS cnt FROM dist_tran_log GROUP BY tranId;
SELECT tranId, uniqExactMerge(logCount) AS cnt FROM dist_mv_tran GROUP BY tranId;
┌─tranId─┬─cnt─┐
│ 003    │   3 │
│ 002    │   3 │
│ 001    │   3 │
└────────┴─────┘
```

注意上述物化视图的使用，创建物化视图时，使用了uniqExactState, 而查询时，使用了uniqueExactMerge， 这种用法并不特殊，ClickHouse所有的聚合函数几乎都可以这样使用。

首先我们要知道，不管是物化视图，还是projection，都是针对单个part而言的。

这也就意味着，如果需要聚合折叠的数据位于不同的part，那么查询的结果很有可能是不准的。

![](https://i-blog.csdnimg.cn/blog_migrate/06672881f61fc86d6146caeea5db7a86.png)

如上图所示，由于uniqExact只能在part内去重，所以得到的最终结果，仍然包含有重复的数据，并不是我们所期望的结果。

而uniqExactState存储的并不是去重后的数据，而是一个去重后的状态，这个状态是不能明文读取的。只有当使用uniqExactMerge去查询时，它得到的才是一个part合并之后的数据。

![](https://i-blog.csdnimg.cn/blog_migrate/6921721dbfd8074a98be5b4b9b48b10d.png)

### **查询SQL优化**

#### **常规查询优化**

#### **prewhere**

prewhere就是先只读取执行prewhere表达式所需要的列，然后再补全读取select所需要的其他列。它的作用就是在查询之前，提前过滤掉一部分的数据。

现在prewhere已经不需要显式指定了。clickhouse优化器会默认将where条件移到prewhere中去执行，所以写where和prewhere效果是一样的。

#### **谓词下推**

所谓的谓词下推，就是尽可能地将过滤条件对数据源操作，使参与查询的数据越少越好。

用在SQL上，就是先过滤，后聚合。

举一个例子：

```cobol
SELLECT count() FROM A JOIN B ON A.id = B.id WHERE A.a > 10 AND B.b < 100;
```

上面这个查询，它会先join，然后从结果集里去执行WHERE条件的过滤。

这时候如果我们将WHERE条件放到子查询中，先过滤，再JOIN：

```cobol
SELECT count() FROM (SELECT * FROM A WHERE A.a > 10) AS A1JOIN(SELECT * FROM B WHERE B.b < 100) AS B1ON A1.id = B1.id
```

这样参与join的数据就会一下子少很多。

#### **聚合外推**

这个比较好理解，就是先聚合，再统一计算。

如 sum(money * 2), 那么需要计算N次，再进行N次sum； 而sum(money) * 2, 只需要N次sum后计算1次。

#### **count优化**

使用count()代替具体的count(col_name)， 原因是count(col_name)需要实际去计算一遍，但是count()的话，底层文件是有一个count.txt文件的，直接读取文件数据即可。

![](https://i-blog.csdnimg.cn/blog_migrate/15039c7858b503451ba6a7c369daa9eb.png)

#### **大表join**

clickhouse的join查询在OLAP场景似乎是不可避免的。尤其是clickhouse这种MPP去中心化架构，分布式join也越来越成为很多企业绕不过去的一个话题。那么如何才能高效的做到大表join？

首先，我们要知道clickhouse里表join的原理。

#### **单节点join**

clickhouse单机join默认采用的是hash join算法，也就是说，先将右表全量读取到内存，然后构建hashmap，然后从左表分批读取数据，到hashmap中去匹配，如果命中，那么就作为join之后的结果输出。

![](https://i-blog.csdnimg.cn/blog_migrate/eb7dbb0942c7ce7909c4712df12dac8e.png)

因为右表是要全量加载到内存的，这就要求要足够小。但是实际上右表大于TB级是很常见的事情，这时候就很容易出现OOM。

为了解决这个问题，有一种解决思路是将右表构建成大宽表，将维度拍平，使行尽量少，这样右表只需要加载少量的列进内存，从而缓解这一情况。

一个典型的落地实施案例就是clickhouse_sinker存储Prometheus指标的方案。它将指标分拆成两张表，一张metric表，用来存储时序指标值，一张metric_series表，用来存储具体的指标。两张表通过series_id进行关联。

在metric表中，每个时间点的每个值，都对应着一个series_id，我们通过这个series_id，反查metric_series表，就可以找到这个series_id对应的指标以及label。

#### **分布式join**

clickhouse的分布式join有两种玩法，一种是带global，一种是不带global。

#### **普通join**

- 汇总节点将左表替换为本地表
- 将左表的本地表分发到每个节点
- 在每个节点执行本地join
- 将结果汇总回汇总节点

这种做法有一个非常严重的问题。在第三步，每个节点执行分布式join的时候，如果右表也是分布式表，那么集群中的每个节点都要去执行分布式查询，那也就是说，如果集群有N个节点，右表查询就会在集群中执行N\*N次，这就是读放大现象。

正是因为这种问题的存在，在比较新的clickhouse版本中，分布式join如果不加global，已经会从语法层面报错了。这说明官方已经禁止了这种写法的存在。

#### **global join**

- 汇总节点将右表改成子查询，先在汇总节点将右表的数据结果集查询出来
- 将右表的结果集广播给各个节点，与各个节点的左表本地表进行join查询
- 各个节点将查询结果发送给汇总节点

由于右表的结果已经在汇总节点计算出来了，那么也就不需要在其他节点重复计算，从而避免了读放大的问题。

但global join 的问题是它需要将整个子查询的结果集发送给各个节点，如果右表的结果集特别大，那么整个过程耗费的网络带宽也将是非常恐怖的。

#### **正确姿势**

- 大表在左，小表在右
- 数据预分布，实现colocate join。也就是将涉及到join的表按照相同的join key分片，使需要join的数据尽量都能在本地完成。也就是前文提到的shardingkey的用处。
- 改大宽表

#### **并发查询**

并发查询也是clickhouse的一个比较薄弱的领域。

因为对于clickhouse的查询来说，一个大的查询SQL往往会把整个服务器的资源吃满，如果并发查询比较多的话，那么不可避免地造成资源竞争，最终的结果就是谁也快不了，甚至还会出现OOM的情况。

官方默认的最大并发数是100， 这个100是每个节点的最大并发数，而不是整个集群的。它包含了查询的并发和写入的并发。

我们很容易碰到这样的场景：在界面上点击一个查询耗时比较久，等得不耐烦，就多点了几下。事实上，clickhouse并不会因为你在界面上多点了一下鼠标，就取消之前的SQL运行，反而会产生多个SQL在并发执行，如果这种耗时比较久的SQL越积CPU打满，造成的结果就是恶性循环，其他SQL也会越来越慢，最终导致并发的SQL超过了设置的最大并发数，再也无法执行任何查询语句，甚至写入都会受到影响。

我们一般建议将最大查询并发数，设置为最大并发数的90%，预留出10%的并发数量供数据写入使用。这样即使查询并发数打满了，仍然不会影响到数据的写入。

我们可以通过配置来实现这一策略：

```cobol
<clickhouse>    <max_concurrent_queries>100</max_concurrent_queries>    <max_concurrent_select_queries>90</max_concurrent_select_queries></clickhouse>
```

#### **读写分离**

读写分离，其实也是资源最大化利用的一个实现方案。

写的节点和读的节点互不干扰，彼此都能完整利用服务器的资源。比如插入的时候插入1，3，5节点，查询的时候查询2，4，6节点。

![](https://i-blog.csdnimg.cn/blog_migrate/52adb940638784a0b2f59693ece07d6e.png)

### **硬件优化**

#### **磁盘选择**

推荐做RAID5磁盘阵列。

为什么推荐raid5， 首先是快，其次是有容错能力。raid0虽然快，但无容错能力，一块盘坏，数据全无。raid1虽然有冗余能力了，但是性能下降了。而raid5兼顾了二者的优点。

当然有条件的推荐直接上SSD，那就是快上加快。现在国产存储逐渐展露头角，固态硬盘也不贵，一块1TB的固态硬盘，成本也就三四百块钱。

有人说，我有多块磁盘，不做raid行不行？从技术实现上来说，肯定是可行的。但带来的就是性能的牺牲，做成了raid，可以统一索引，而配置多块盘到default卷，如果要查询的数据跨盘了，那么需要在不同的磁盘上都构建一遍索引。这个性能差异在数据量不大的情况下，可能肉眼感受不出来，如果数据量达到了一定的量级，差异还是有的。而且，冗余能力肯定是没有的，如果其中一块盘坏了，那么数据能不能恢复，就要看人品了。

我个人比较推荐固态硬盘和机械盘搭配使用。固态磁盘做default磁盘，机械磁盘可以做raid5。热数据，经常会被查询的数据（比如近两三天的数据），存储到固态磁盘，这样不论是写入，还是查询都非常快，超过一定时间的数据，可以通过存储策略转移到机械磁盘上，因为查询频率并不怎么高，那么受到的影响就会有限。

磁盘的io决定了数据的读写速度。在大数据量场景，机械磁盘和固态磁盘的查询和写入性能差异还是非常大的。测试环境、POC环境，如果条件有限，可以用机械磁盘，生产环境，特别是对响应要求比较高的生产系统，一定要上固态，最好是一步到位直接使用NVMe。

#### **生命周期管理**

前面提到磁盘，那么就不得不提到存储策略。而存储策略和表的生命周期往往是相伴相生的。

生命周期对表的优化体现在两个方面。

第一个是实现数据的冷热分离。就如前面提到的，把时间比较近的，查询比较频繁的数据存在固态磁盘，这样使查询和写入都有一个最佳性能。但固态硬盘的成本毕竟较贵，不可能无限扩容，因此，可以将一些比较查询不那么频繁的温数据存放在机械磁盘中。而那些一个月以上的数据，可能一年也查询不了几次，主要用来做归档用的，就可以存储在HDFS或者S3这类对象存储中，其特点是不利于查询，但是存储空间够大，成本便宜。

![](https://i-blog.csdnimg.cn/blog_migrate/7844bb8c56376eb9aaa98618a941eee9.png)

但是有些使用场景可能并不需要存储那么长时间的数据，也没有归档的硬性要求，这时，就可以利用生命周期来做删除策略，将某短时间时间之前的数据直接删除掉，这样保持表里永远只存储一定时间段的数据，那么数据规模也将维持在一个固定的范围，不会无休止增长，查询性能相对来说也会比较稳定，不会有太大的波动。

> 需要注意的是，生命周期的前提是数据的插入是有序的。比如实时的时序数据。  
> 如果插入的数据无序，又设置了生命周期，那么可能出现的一种情况就是数据刚刚插入，生命周期就生效，立即就给删除了，除了徒然增加资源消耗，这部分数据永远无法查询到。  
> 而且这种情况一旦出现，其时间跨度一定很大，必然落在不同的partition里，那么数据的频繁写入、删除，必然给zookeeper带来非常大的压力，很容易造成表出现READ_ONLY的情况。

#### **集群规模**

ClickHouse是MPP架构。MPP全称叫海量并行处理架构。这种架构的集群，每个节点的地位是相等的，也就是说，每个分片内，各个副本都是主，既可以读，也可以写。它最大的特点就是share nothing，也就是每个节点只访问本地的资源。

也就是说，一个查询语句落到集群里，它是先在各个节点进行查询、聚合、汇总，最后汇聚到汇总节点统一返回。

这就很容易得出结论，当CPU或者内存成为瓶颈时，扩容集群的分片数，是可以有效加速查询效率的。

我曾经给很多客户做过集群资源评估，推荐过集群硬件配置。需要考虑很多因素，比如数据日增量，存储时间，查询场景（如查询多长时间的数据）等。数据日增量决定了建表的分区策略，存储时间和日增量就能计算出数据总规模（当然要考虑压缩），这决定了硬盘容量需要多大。正常情况下需要冗余1.5倍左右的磁盘空间， 因为数据被merge之后，原始part的数据并不是立即被删除的，所以大多时候经常是两份数据共存，那么如果实际数据是1TB（压缩后），那么实际占用的磁盘肯定是不止1TB的。

查询场景决定了CPU和内存资源的占用，如果查询的数据量大，那么对应需要占用的CPU和内存自然就要更多。此时，要么优化查询语句以及建表语句，要么就只能使用更多的资源。

### **读写性能瓶颈**

此处讨论的瓶颈，是指表和SQL都已经做到了最大化的优化，硬件以及第三方原因带来的性能瓶颈。

#### **写性能瓶颈**

#### **zookeeper**

写数据的瓶颈，绝大部分都是zookeeper引起的。特别是insert分布式表的时候，它的数据会先全部insert到某一个节点上的一个临时目录，然后通过zookeeper按照sharding policy进行分发到各个分片节点，同时副本节点需要从主节点上去拉取数据，走的都是zookeeper。

因此，我们不推荐写分布式表，一来是给单节点带来非常大的负担，二来给zookeeper带来很大的压力。第三就是无法保证数据在物理上和业务逻辑上的均衡。

zookeeper的瓶颈主要在于znode的数量，如果znode的数量过多，会非常影响性能。经测试，当znode数量达到M级别（百万）时，插入性能就已经非常缓慢了，严重的甚至可能造成zookeeper失联，从而使表进入read_only状态。

znode数量与什么东西有关呢？

首先是part的数量。每个part在zookeeper上都有副本数 * part数个znode。通俗点计算，N个节点，M个part，那么znode的数量就会无限接近于M* N。因此，如果插入的batch太小，数据来不及merge，那么就会产生非常多的part。

还有一种可能就是replica_queue过多。这种情况一般是merge任务已经分发到了各副本节点，但是副本节点来不及合并，那么这个任务都阻塞在队列里，队列越来越长，就会导致znode越来越多。

之前笔者就曾碰见过，zookeeper上log_pointer数据意外丢失，造成数据一直没有同步，但是数据又一直在写，导致replica_queue越积越多，znode达到了150w之巨，最终导致了zookeeper的不可用。

#### **磁盘IO**

磁盘IO很好理解，固态硬盘的写入速度肯定是远远大于机械硬盘的。如果发现CPU一直没吃满的情况下，写入速度怎么都上不去，那么很有可能就是磁盘IO限制了写入的性能了。这时，就需要考虑更换性能更高的固态硬盘了。

#### **网络带宽**

网络带宽的瓶颈一般出现在副本同步数据上。因为数据插入，肯定是插入到分片的某一个副本上的，那么其他副本要同步数据，必然要通过网络传输的方式来同步，那么当写入速度足够快，网络带宽必然会成为数据同步的瓶颈。一般生产环境，推荐使用万兆网卡。

#### **读性能瓶颈**

#### **磁盘IO**

查询，说白了就是扫描磁盘 。因此读磁盘的快慢就决定了查询的快慢，这点是显而易见的。

#### **网络带宽**

分布式查询，需要将数据结果从各个节点汇总到初始节点，这个过程是需要占用非常大的带宽的。如果涉及到分布式join，那么带宽将会占用更多。

#### **内存和CPU**

MPP架构决定的，在本地查询，充分利用本地资源。当我们一个查询SQL运行比较慢，通过增加节点能有明显改善的时候，那说明瓶颈就在CPU上。如果内存不足，直接就OOM了，就不会有执行完成的机会。

### **优化推荐**

#### **硬件层面**

- 优先使用固态硬盘
- 使用raid5磁盘阵列
- zookeeper单独部署，不要与其他业务混用
- 为保证单节点资源利用最大化，不建议集群混合部署，一个节点仅部署一个服务

#### **使用层面**

- 建表时合理分区
- 合理指定主键字段，排序字段
- 合理创建二级索引（全文检索场景）
- 合理使用物化视图和projection（预聚合场景）
- 选择好shardingkey（方便Colocate join）
- 尽量使用子查询代替join
- 配置读写分离
- 配置生命周期，冷热数据分离或清理策略