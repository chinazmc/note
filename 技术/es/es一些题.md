
### 1. 什么是Elasticsearch？

Elasticsearch 是一个基于 Lucene 的搜索引擎。它提供了具有 HTTP Web 界面和无架构 JSON 文档的分布式，多租户能力的全文搜索引擎。  
Elasticsearch 是用 Java 开发的，根据 Apache 许可条款作为开源发布。

### 2.为什么要使用Elasticsearch?

​ 　　因为在我们商城中的数据，将来会非常多，所以采用以往的模糊查询，模糊查询前缀匹配，索引失效，会放弃索引，导致商品查询是全表扫面，在百万级别的数据库中，效率非常低下，而我们使用ES做一个全文索引，我们将经常查询的商品的某些字段，比如说商品名，描述、价格还有id这些字段我们放入我们索引库里，可以提高查询速度。

### 2. ES中的倒排索引是什么？

传统的检索方式是通过文章，逐个遍历找到对应关键词的位置。  
倒排索引，是通过分词策略，形成了词和文章的映射关系表，也称倒排表，这种词典 + 映射表即为**倒排索引**。

其中词典中存储词元，倒排表中存储该词元在哪些文中出现的位置。  
有了倒排索引，就能实现 O(1) 时间复杂度的效率检索文章了，极大的提高了检索效率。

加分项：  
倒排索引的底层实现是基于：FST（Finite State Transducer有限状态转换器）数据结构。

Lucene 从 4+ 版本后开始大量使用的数据结构是 FST。FST 有两个优点：  
1）空间占用小。通过对词典中单词前缀和后缀的重复利用，压缩了存储空间；  
2）查询速度快。O(len(str)) 的查询时间复杂度。

### 3. ES是如何实现master选举的？

前置条件：  
1）只有是候选主节点（master：true）的节点才能成为主节点。  
2）最小主节点数（min_master_nodes）的目的是防止脑裂。

Elasticsearch 的选主是 ZenDiscovery 模块负责的，主要包含 Ping（节点之间通过这个RPC来发现彼此）和 Unicast（单播模块包含一个主机列表以控制哪些节点需要 ping 通）这两部分；  
获取主节点的核心入口为 findMaster，选择主节点成功返回对应 Master，否则返回 null。

选举流程大致描述如下：  
第一步：确认候选主节点数达标，elasticsearch.yml 设置的值 discovery.zen.minimum_master_nodes;  
第二步：对所有候选主节点根据nodeId字典排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。  
第三步：如果对某个节点的投票数达到一定的值（候选主节点数n/2+1）并且该节点自己也选举自己，那这个节点就是master。否则重新选举一直到满足上述条件。

-   补充：
    -   这里的 id 为 string 类型。
    -   master 节点的职责主要包括集群、节点和索引的管理，不负责文档级别的管理；data 节点可以关闭 http 功能。

### 4. 如何解决ES集群的脑裂问题

所谓集群脑裂，是指 Elasticsearch 集群中的节点（比如共 20 个），其中的 10 个选了一个 master，另外 10 个选了另一个 master 的情况。

当集群 master 候选数量不小于 3 个时，可以通过设置最少投票通过数量（discovery.zen.minimum_master_nodes）超过所有候选节点一半以上来解决脑裂问题；  
当候选数量为两个时，只能修改为唯一的一个 master 候选，其他作为 data 节点，避免脑裂问题。

### 5. 详细描述一下ES索引文档的过程？

这里的索引文档应该理解为文档写入 ES，创建索引的过程。

第一步：客户端向集群某节点写入数据，发送请求。（如果没有指定路由/协调节点，请求的节点扮演协调节点的角色。）  
第二步：协调节点接受到请求后，默认使用文档 ID 参与计算（也支持通过 routing），得到该文档属于哪个分片。随后请求会被转到另外的节点。

bash

```java
# 路由算法：根据文档id或路由计算目标的分片id
shard = hash(document_id) % (num_of_primary_shards)
```

第三步：当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 Memory Buffer，然后定时（默认是每隔 1 秒）写入到F ilesystem Cache，这个从 Momery Buffer 到 Filesystem Cache 的过程就叫做 refresh；  
第四步：当然在某些情况下，存在 Memery Buffer 和 Filesystem Cache 的数据可能会丢失，ES 是通过 translog 的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中，当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flush；  
第五步：在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync 将创建一个新的提交点，并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。  
第六步：flush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512 M）时。

![elasticsearch_index_process.jpg](https://www.wenyuanblog.com/medias/blogimages/elasticsearch_index_process.jpg)

-   补充：关于 Lucene 的 Segement
    -   Lucene 索引是由多个段组成，段本身是一个功能齐全的倒排索引。
    -   段是不可变的，允许 Lucene 将新的文档增量地添加到索引中，而不用从头重建索引。
    -   对于每一个搜索请求而言，索引中的所有段都会被搜索，并且每个段会消耗 CPU 的时钟周、文件句柄和内存。这意味着段的数量越多，搜索性能会越低。
    -   为了解决这个问题，Elasticsearch 会合并小段到一个较大的段，提交新的合并段到磁盘，并删除那些旧的小段。（段合并）

### 6. 详细描述一下ES更新和删除文档的过程？

删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更。

磁盘上的每个段都有一个相应的 .del 文件。当删除请求发送后，文档并没有真的被删除，而是在 .del 文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在 .del 文件中被标记为删除的文档将不会被写入新段。

在新的文档被创建时，Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在 .del 文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉。

### 7. 详细描述一下ES搜索的过程？

搜索被执行成一个两阶段过程，即 Query Then Fetch；  
Query阶段：  
查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列。PS：在搜索的时候是会查询Filesystem Cache的，但是有部分数据还在Memory Buffer，所以搜索是近实时的。  
每个分片返回各自优先队列中 **所有文档的 ID 和排序值** 给协调节点，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。  
Fetch阶段：  
协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每个分片加载并 丰富 文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了，协调节点返回结果给客户端。

![](https://img2020.cnblogs.com/blog/2107925/202103/2107925-20210309175442258-281616628.png)

### 8. 在并发情况下，ES如果保证读写一致？

可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖，由应用层来处理具体的冲突；  
另外对于写操作，一致性级别支持quorum/one/all，默认为quorum，即只有当大多数分片可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，分片将会在一个不同的节点上重建。  
对于读操作，可以设置replication为sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置replication为async时，也可以通过设置搜索请求参数_preference为primary来查询主分片，确保文档是最新版本。

### 9. ES对于大数据量（上亿量级）的聚合如何实现？

Elasticsearch 提供的首个近似聚合是cardinality 度量。它提供一个字段的基数，即该字段的distinct或者unique值的数目。它是基于HLL算法的。HLL 会先对我们的输入作哈希运算，然后根据哈希运算的结果中的 bits 做概率估算从而得到基数。其特点是：可配置的精度，用来控制内存的使用（更精确 ＝ 更多内存）；小的数据集精度是非常高的；我们可以通过配置参数，来设置去重需要的固定内存使用量。无论数千还是数十亿的唯一值，内存使用量只与你配置的精确度相关。

### 10. 对于GC方面，在使用ES时要注意什么？

1）倒排词典的索引需要常驻内存，无法GC，需要监控data node上segment memory增长趋势。  
2）各类缓存，field cache, filter cache, indexing cache, bulk queue等等，要设置合理的大小，并且要应该根据最坏的情况来看heap是否够用，也就是各类缓存全部占满的时候，还有heap空间可以分配给其他任务吗？避免采用clear cache等“自欺欺人”的方式来释放内存。  
3）避免返回大量结果集的搜索与聚合。确实需要大量拉取数据的场景，可以采用scan & scroll api来实现。  
4）cluster stats驻留内存并无法水平扩展，超大规模集群可以考虑分拆成多个集群通过tribe node连接。  
5）想知道heap够不够，必须结合实际应用场景，并对集群的heap使用情况做持续的监控。

### 11. 说说你们公司ES的集群架构，索引数据大小，分片有多少，以及一些调优手段？

根据实际情况回答即可，如果是我的话会这么回答：  
我司有多个ES集群，下面列举其中一个。该集群有20个节点，根据数据类型和日期分库，每个索引根据数据量分片，比如日均1亿+数据的，控制单索引大小在200GB以内。　  
下面重点列举一些调优策略，仅是我做过的，不一定全面，如有其它建议或者补充欢迎留言。  
部署层面：  
1）最好是64GB内存的物理机器，但实际上32GB和16GB机器用的比较多，但绝对不能少于8G，除非数据量特别少，这点需要和客户方面沟通并合理说服对方。  
2）多个内核提供的额外并发远胜过稍微快一点点的时钟频率。  
3）尽量使用SSD，因为查询和索引性能将会得到显著提升。  
4）避免集群跨越大的地理距离，一般一个集群的所有节点位于一个数据中心中。  
5）设置堆内存：节点内存/2，不要超过32GB。一般来说设置export ES_HEAP_SIZE=32g环境变量，比直接写-Xmx32g -Xms32g更好一点。  
6）关闭缓存swap。内存交换到磁盘对服务器性能来说是致命的。如果内存交换到磁盘上，一个100微秒的操作可能变成10毫秒。 再想想那么多10微秒的操作时延累加起来。不难看出swapping对于性能是多么可怕。  
7）增加文件描述符，设置一个很大的值，如65535。Lucene使用了大量的文件，同时，Elasticsearch在节点和HTTP客户端之间进行通信也使用了大量的套接字。所有这一切都需要足够的文件描述符。  
8）不要随意修改垃圾回收器（CMS）和各个线程池的大小。  
9）通过设置gateway.recover_after_nodes、gateway.expected_nodes、gateway.recover_after_time可以在集群重启的时候避免过多的分片交换，这可能会让数据恢复从数个小时缩短为几秒钟。  
索引层面：  
1）使用批量请求并调整其大小：每次批量数据 5–15 MB 大是个不错的起始点。  
2）段合并：Elasticsearch默认值是20MB/s，对机械磁盘应该是个不错的设置。如果你用的是SSD，可以考虑提高到100-200MB/s。如果你在做批量导入，完全不在意搜索，你可以彻底关掉合并限流。另外还可以增加 index.translog.flush_threshold_size 设置，从默认的512MB到更大一些的值，比如1GB，这可以在一次清空触发的时候在事务日志里积累出更大的段。  
3）如果你的搜索结果不需要近实时的准确度，考虑把每个索引的index.refresh_interval 改到30s。  
4）如果你在做大批量导入，考虑通过设置index.number_of_replicas: 0 关闭副本。  
5）需要大量拉取数据的场景，可以采用scan & scroll api来实现，而不是from/size一个大范围。  
存储层面：  
1）基于数据+时间滚动创建索引，每天递增数据。控制单个索引的量，一旦单个索引很大，存储等各种风险也随之而来，所以要提前考虑+及早避免。  
2）冷热数据分离存储，热数据（比如最近3天或者一周的数据），其余为冷数据。对于冷数据不会再写入新数据，可以考虑定期force_merge加shrink压缩操作，节省存储空间和检索效率。


## 1、简要介绍一下Elasticsearch？

严谨期间，如下一段话直接拷贝官方网站：https://www.elastic.co/cn/elasticsearch/

Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

ElasticSearch 是基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。

核心特点如下：

-   分布式的实时文件存储，每个字段都被索引且可用于搜索。
    
-   分布式的实时分析搜索引擎，海量数据下近实时秒级响应。
    
-   简单的restful api，天生的兼容多语言开发。
    
-   易扩展，处理PB级结构化或非结构化数据。
    

## 2、 您能否说明当前可下载的稳定Elasticsearch版本？

Elasticsearch 当前最新版本是7.10（2020年11月21日）。

为什么问这个问题？ES 更新太快了，应聘者如果了解并使用最新版本，基本能说明他关注 ES 更新。甚至从更广维度讲，他关注技术的迭代和更新。

但，不信你可以问问，很多求职者只知道用了 ES，什么版本一概不知。

## 3、安装 Elasticsearch 需要依赖什么组件吗？

ES 早期版本需要JDK，在7.X版本后已经集成了 JDK，已无需第三方依赖。

## 4、您能否分步介绍如何启动 Elasticsearch 服务器？

启动方式有很多种，一般 bin 路径下

`./elasticsearch -d` 

就可以后台启动。

打开浏览器输入 http://ES IP:9200 就能知道集群是否启动成功。

如果启动报错，日志里会有详细信息，逐条核对解决就可以。


## 6、 解释一下Elasticsearch Cluster？

Elasticsearch 集群是一组连接在一起的一个或多个 Elasticsearch 节点实例。

Elasticsearch 集群的功能在于在集群中的所有节点之间分配任务，进行搜索和建立索引。

## 7、解释一下 Elasticsearch Node？

节点是 Elasticsearch 的实例。实际业务中，我们会说：ES集群包含3个节点、7个节点。

这里节点实际就是：一个独立的 Elasticsearch 进程，一般将一个节点部署到一台独立的服务器或者虚拟机、容器中。

不同节点根据角色不同，可以划分为：

-   主节点
    

帮助配置和管理在整个集群中添加和删除节点。

-   数据节点
    

存储数据并执行诸如CRUD（创建/读取/更新/删除）操作，对数据进行搜索和聚合的操作。

-   客户端节点（或者说：协调节点） 将集群请求转发到主节点，将与数据相关的请求转发到数据节点
    
-   摄取节点
    

用于在索引之前对文档进行预处理。

... ...

## 8、解释一下 Elasticsearch集群中的 索引的概念 ？

Elasticsearch 集群可以包含多个索引，与关系数据库相比，它们相当于数据库表

其他类别概念，如下表所示，点到为止。

![](https://filescdn.proginn.com/8bb58ccb10572093059cf83514053fbd/1fe9075acef6787c0873e41318208cfa.webp)

![](https://filescdn.proginn.com/964bbdff5394043e642d6aa0689ba386/40dbfc80e3a3332e3ae47edcb3e240bd.webp)

## 9、解释一下 Elasticsearch 集群中的 Type 的概念 ？

5.X 以及之前的 2.X、1.X 版本 ES支持一个索引多个type的，举例 ES 6.X 中的Join 类型在早期版本实际是多 Type 实现的。

在6.0.0 或 更高版本中创建的索引只能包含一个 Mapping 类型。

Type 将在Elasticsearch 7.0.0中的API中弃用，并在8.0.0中完全删除。

很多人好奇为什么删除？看这里：

https://www.elastic.co/guide/en/elasticsearch/reference/current/removal-of-types.html

## 10、你能否在 Elasticsearch 中定义映射？

映射是定义文档及其包含的字段的存储和索引方式的过程。

例如，使用映射定义：

-   哪些字符串字段应该定义为 text 类型。
    
-   哪些字段应该定义为：数字，日期或地理位置 类型。
    
-   自定义规则来控制动态添加字段的类型。
    

## 11、Elasticsearch的 文档是什么？

文档是存储在 Elasticsearch 中的 JSON 文档。它等效于关系数据库表中的一行记录。

## 12、解释一下 Elasticsearch 的 分片？

当文档数量增加，硬盘容量和处理能力不足时，对客户端请求的响应将延迟。

在这种情况下，将索引数据分成小块的过程称为分片，可改善数据搜索结果的获取。

## 13、定义副本、创建副本的好处是什么？

副本是 分片的对应副本，用在极端负载条件下提高查询吞吐量或实现高可用性。

所谓高可用主要指：如果某主分片1出了问题，对应的副本分片1会提升为主分片，保证集群的高可用。

## 14、请解释在 Elasticsearch 集群中添加或创建索引的过程？

要添加新索引，应使用创建索引 API 选项。创建索引所需的参数是索引的配置Settings，索引中的字段 Mapping 以及索引别名 Alias。

也可以通过模板 Template 创建索引。

## 15、在 Elasticsearch 中删除索引的语法是什么？

可以使用以下语法删除现有索引：

`DELETE` 

支持通配符删除：

`DELETE my_*   `

## 16、在 Elasticsearch 中列出集群的所有索引的语法是什么？

`    GET _cat/indices    `

## 17、在索引中更新 Mapping 的语法？

`       PUT test_001/_mapping   {     "properties": {       "title":{         "type":"keyword"       }     }   }    `

## 18、在Elasticsearch中 按 ID检索文档的语法是什么？

`GET test_001/_doc/1   `

## 19、解释 Elasticsearch 中的相关性和得分？

当你在互联网上搜索有关 Apple 的信息时。它可以显示有关水果或苹果公司名称的搜索结果。

-   你可能要在线购买水果，检查水果中的食谱或食用水果，苹果对健康的好处。
    
-   你也可能要检查Apple.com，以查找该公司提供的最新产品范围，检查评估公司的股价以及最近6个月，1或5年内该公司在纳斯达克的表现。
    

同样，当我们从 Elasticsearch 中搜索文档（记录）时，你会对获取所需的相关信息感兴趣。基于相关性，通过Lucene评分算法计算获得相关信息的概率。

ES 会将相关的内容都返回给你，只是：计算得出的评分高的排在前面，评分低的排在后面。

计算评分相关的两个核心因素是：词频和逆向文档频率（文档的稀缺性）。

大体可以解释为：单篇文档词频越高、得分越高；多篇文档某词越稀缺，得分越高。

## 20、我们可以在 Elasticsearch 中执行搜索的各种可能方式有哪些？

核心方式如下：

方式一：基于 DSL 检索（最常用） Elasticsearch提供基于JSON的完整查询DSL来定义查询。

`GET /shirts/_search   {     "query": {       "bool": {         "filter": [           { "term": { "color": "red"   }},           { "term": { "brand": "gucci" }}         ]       }     }   }   `

方式二：基于 URL 检索

`GET /my_index/_search?q=user:seina   `

方式三：类SQL 检索

`POST /_sql?format=txt   {     "query": "SELECT * FROM uint-2020-08-17 ORDER BY itemid DESC LIMIT 5"   }   `

功能还不完备，不推荐使用。

## 21、Elasticsearch 支持哪些类型的查询？

查询主要分为两种类型：精确匹配、全文检索匹配。

-   精确匹配，例如 term、exists、term set、 range、prefix、 ids、 wildcard、regexp、 fuzzy等。
    
-   全文检索，例如match、match_phrase、multi_match、match_phrase_prefix、query_string 等
    

## 22、精准匹配检索和全文检索匹配检索的不同？

两者的本质区别：

-   精确匹配用于：是否完全一致？
    

举例：邮编、身份证号的匹配往往是精准匹配。

-   全文检索用于：是否相关？
    

举例：类似B站搜索特定关键词如“马保国 视频”往往是模糊匹配，相关的都返回就可以。

## 23、请解释一下 Elasticsearch 中聚合？

聚合有助于从搜索中使用的查询中收集数据，聚合为各种统计指标，便于统计信息或做其他分析。聚合可帮助回答以下问题：

-   我的网站平均加载时间是多少？
    
-   根据交易量，谁是我最有价值的客户？
    
-   什么会被视为我网络上的大文件？
    
-   每个产品类别中有多少个产品？
    

聚合的分三类：

主要查看7.10 的官方文档，早期是4个分类，别大意啊！

-   分桶 Bucket 聚合
    

根据字段值，范围或其他条件将文档分组为桶（也称为箱）。

-   指标 Metric 聚合
    

从字段值计算指标（例如总和或平均值）的指标聚合。

-   管道 Pipeline 聚合
    

子聚合，从其他聚合（而不是文档或字段）获取输入。

## 24、你能告诉我 Elasticsearch 中的数据存储功能吗？

Elasticsearch是一个搜索引擎，输入写入ES的过程就是索引化的过程，数据按照既定的 Mapping 序列化为Json 文档实现存储。

## 25、什么是Elasticsearch Analyzer？

分析器用于文本分析，它可以是内置分析器也可以是自定义分析器。它的核心三部分构成如下图所示：

![](https://filescdn.proginn.com/6978fee104ce1abef8baf0e570531dfd/b3a9790abe771c217205706e9f88f765.webp)

推荐：[Elasticsearch自定义分词，从一个问题说开去](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484394&idx=1&sn=e09fd10df524af1552903baadfbb026d&chksm=eaa82bc2dddfa2d4eab983005ee0c7cf19549fc5b2fa507c238f9e017dfd38f0d045c8d3f871&scene=21#wechat_redirect)

## 26、你可以列出 Elasticsearch 各种类型的分析器吗？

Elasticsearch Analyzer 的类型为内置分析器和自定义分析器。

-   Standard Analyzer
    

标准分析器是默认分词器，如果未指定，则使用该分词器。

它基于Unicode文本分割算法，适用于大多数语言。

-   Whitespace Analyzer
    

基于空格字符切词。

-   Stop Analyzer
    

在simple Analyzer的基础上，移除停用词。

-   Keyword Analyzer
    

不切词，将输入的整个串一起返回。

自定义分词器的模板

自定义分词器的在Mapping的Setting部分设置：

`PUT my_custom_index   {    "settings":{     "analysis":{     "char_filter":{},     "tokenizer":{},     "filter":{},     "analyzer":{}     }    }   }   `

脑海中还是上面的三部分组成的图示。其中：

“char_filter”:{},——对应字符过滤部分；

“tokenizer”:{},——对应文本切分为分词部分；

“filter”:{},——对应分词后再过滤部分；

“analyzer”:{}——对应分词器组成部分，其中会包含：1. 2. 3。

## 27、如何使用 Elasticsearch Tokenizer？

Tokenizer 接收字符流（如果包含了字符过滤，则接收过滤后的字符流；否则，接收原始字符流），将其分词。同时记录分词后的顺序或位置(position)，以及开始值（start_offset）和偏移值(end_offset-start_offset)。

## 28、token filter 过滤器 在 Elasticsearch 中如何工作？

针对 tokenizers 处理后的字符流进行再加工，比如：转小写、删除（删除停用词）、新增（添加同义词）等。

## 29、Elasticsearch中的 Ingest 节点如何工作？

ingest 节点可以看作是数据前置处理转换的节点，支持 pipeline管道 设置，可以使用 ingest 对数据进行过滤、转换等操作，类似于 logstash 中 filter 的作用，功能相当强大。

## 30、Master 节点和 候选 Master节点有什么区别？

主节点负责集群相关的操作，例如创建或删除索引，跟踪哪些节点是集群的一部分，以及决定将哪些分片分配给哪些节点。

拥有稳定的主节点是衡量集群健康的重要标志。

而候选主节点是被选具备候选资格，可以被选为主节点的那些节点。

## 31、Elasticsearch中的属性 enabled, index 和 store 的功能是什么？

-   enabled：false，启用的设置仅可应用于顶级映射定义和 Object 对象字段，导致 Elasticsearch 完全跳过对字段内容的解析。
    

仍然可以从_source字段中检索JSON，但是无法搜索或以其他任何方式存储JSON。

如果对非全局或者 Object 类型，设置 enable : false 会报错如下：

 `"type": "mapper_parsing_exception",    "reason": "Mapping definition for [user_id] has unsupported parameters:  [enabled : false]"`

-   index：false, 索引选项控制是否对字段值建立索引。它接受true或false，默认为true。未索引的字段不可查询。
    

如果非要检索，报错如下：

 `"type": "search_phase_execution_exception",     "reason": "Cannot search on field [user_id] since it is not indexed."`

-   store：
    
    某些特殊场景下，如果你只想检索单个字段或几个字段的值，而不是整个_source的值，则可以使用源过滤来实现；
    
    这个时候， store 就派上用场了。
    

![](https://filescdn.proginn.com/bee9711b23711913475d429161a57f50/7e2e04f29292e0f6aa0277d6bf47c492.webp)

## 32、Elasticsearch Analyzer 中的字符过滤器如何利用？

字符过滤器将原始文本作为字符流接收，并可以通过添加，删除或更改字符来转换字符流。

字符过滤分类如下：

-   HTML Strip Character Filter.
    

用途：删除HTML元素，如**，并解码HTML实体，如＆amp 。**

-   **Mapping Character Filter**
    

**用途：替换指定的字符。**

-   **Pattern Replace Character Filter**
    

**用途：基于正则表达式替换指定的字符。**

## **33、请解释有关 Elasticsearch的 NRT？**

**从文档索引（写入）到可搜索到之间的延迟默认一秒钟，因此Elasticsearch是近实时（NRT）搜索平台。**

**也就是说：文档写入，最快一秒钟被索引到，不能再快了。**

**写入调优的时候，我们通常会动态调整：refresh_interval = 30s 或者更达值，以使得写入数据更晚一点时间被搜索到。**

## **34、REST API在 Elasticsearch 方面有哪些优势？**

**REST API是使用超文本传输协议的系统之间的通信，该协议以 XML 和 JSON格式传输数据请求。**

**REST 协议是无状态的，并且与带有服务器和存储数据的用户界面分开，从而增强了用户界面与任何类型平台的可移植性。它还提高了可伸缩性，允许独立实现组件，因此应用程序变得更加灵活。**

**REST API与平台和语言无关，只是用于数据交换的语言是XML或JSON。**

**借助：REST API 查看集群信息或者排查问题都非常方便。**

## **35、在安装Elasticsearch时，请说明不同的软件包及其重要性？**

**这个貌似没什么好说的，去官方文档下载对应操作系统安装包即可。**

**部分功能是收费的，如机器学习、高级别 kerberos 认证安全等选型要知悉。**

## **36、Elasticsearch 支持哪些配置管理工具？**

-   **Ansible**
    
-   **Chef**
    
-   **Puppet**
    
-   **Salt Stack**
    
    **是 DevOps 团队使用的 Elasticsearch支持的配置工具。**
    

## **37、您能解释一下X-Pack for Elasticsearch的功能和重要性吗？**

**X-Pack 是与Elasticsearch一起安装的扩展程序。**

**X-Pack的各种功能包括安全性（基于角色的访问，特权/权限，角色和用户安全性），监视，报告，警报等。**

## **38、可以列出X-Pack API 吗？**

**付费功能只是试用过（面试时如实回答就可以）。**

**7.1 安全功能免费后，用 X-pack 创建Space、角色、用户，设置SSL加密，并且为不同用户设置不同的密码和分配不同的权限。**

**其他如：机器学习、 Watcher、 Migration 等 API 用的较少。**

## **39、能列举过你使用的 X-Pack 命令吗?**

**7.1 安全功能免费后，使用了：setup-passwords 为账号设置密码，确保集群安全。**

## **40、在Elasticsearch中 cat API的功能是什么？**

**cat API 命令提供了Elasticsearch 集群的分析、概述和运行状况，其中包括与别名，分配，索引，节点属性等有关的信息。**

**这些 cat 命令使用查询字符串作为其参数，并以J SON 文档格式返回结果信息。**

## **41、Elasticsearch 中常用的 cat命令有哪些？**

**面试时说几个核心的就可以，包含但不限于：**

含义

命令

别名

GET _cat/aliases?v

分配相关

GET _cat/allocation

计数

GET _cat/count?v

字段数据

GET _cat/fielddata?v

运行状况

GET_cat/health?

索引相关

GET _cat/indices?v

主节点相关

GET _cat/master?v

节点属性

GET _cat/nodeattrs?v

节点

GET _cat/nodes?v

待处理任务

GET _cat/pending_tasks?v

插件

GET _cat/plugins?v

恢复

GET _cat / recovery?v

存储库

GET _cat /repositories?v

段

GET _cat /segments?v

分片

GET _cat/shards?v

快照

GET _cat/snapshots?v

任务

GET _cat/tasks?v

模板

GET _cat/templates?v

线程池

GET _cat/thread_pool?v

## **42、您能解释一下 Elasticsearch 中的 Explore API 吗？**

**没有用过，这是 Graph （收费功能）相关的API。**

**点到为止即可，类似问题实际开发现用现查，类似问题没有什么意义。**

**https://www.elastic.co/guide/en/elasticsearch/reference/current/graph-explore-api.html**

## **43、迁移 Migration API 如何用作 Elasticsearch？**

**迁移 API简化了X-Pack索引从一个版本到另一个版本的升级。**

**点到为止即可，类似问题实际开发现用现查，类似问题没有什么意义。**

**https://www.elastic.co/guide/en/elasticsearch/reference/current/migration-api.html**

## **44、如何在 Elasticsearch中 搜索数据？**

**Search API 有助于从索引、路由参数引导的特定分片中查找检索数据。**

## **45、你能否列出与 Elasticsearch 有关的主要可用字段数据类型？**

-   **字符串数据类型，包括支持全文检索的 text 类型 和 精准匹配的 keyword 类型。**
    
-   **数值数据类型，例如字节，短整数，长整数，浮点数，双精度数，half_float，scaled_float。**
    
-   **日期类型，日期纳秒Date nanoseconds，布尔值，二进制（Base64编码的字符串）等。**
    
-   **范围（整数范围 integer_range，长范围 long_range，双精度范围 double_range，浮动范围 float_range，日期范围 date_range）。**
    
-   **包含对象的复杂数据类型，nested 、Object。**
    
-   **GEO 地理位置相关类型。**
    
-   **特定类型如：数组（数组中的值应具有相同的数据类型）**
    

## **46、详细说明ELK Stack及其内容？**

**ELK Stack是一系列搜索和分析工具（Elasticsearch），收集和转换工具（Logstash）以及数据管理及可视化工具（Kibana）、解析和收集日志工具（Beats 未来是 Agent）以及监视和报告工具（例如X Pack）的集合。**

**相当于用户基本不再需要第三方技术栈，就能全流程、全环节搞定数据接入、存储、检索、可视化分析等全部功能。**

## **47、Kibana在Elasticsearch的哪些地方以及如何使用？**

**Kibana是ELK Stack –日志分析解决方案的一部分。**

**它是一种开放源代码的可视化工具，可以以拖拽、自定义图表的方式直观分析数据，极大降低的数据分析的门槛。**

**未来会向类似：商业智能和分析软件 - Tableau 发展。**

## **48、logstash 如何与 Elasticsearch 结合使用？**

**logstash 是ELK Stack附带的开源 ETL 服务器端引擎，该引擎可以收集和处理来自各种来源的数据。**

**最典型应用包含：同步日志、邮件数据，同步关系型数据库（Mysql、Oracle）数据，同步非关系型数据库（MongoDB）数据，同步实时数据流 Kafka数据、同步高性能缓存 Redis 数据等。**

## **49、Beats 如何与 Elasticsearch 结合使用？**

**Beats是一种开源工具，可以将数据直接传输到 Elasticsearch 或通过 logstash，在使用Kibana进行查看之前，可以对数据进行处理或过滤。**

**传输的数据类型包含：审核数据，日志文件，云数据，网络流量和窗口事件日志等。**

## **50、如何使用 Elastic Reporting ？**

**收费功能，只是了解，点到为止。**

**Reporting API有助于将检索结果生成 PD F格式，图像 PNG 格式以及电子表格 CSV 格式的数据，并可根据需要进行共享或保存。**

## **51、您能否列出 与 ELK日志分析相关的应用场景？**

-   **电子商务搜索解决方案**
    
-   **欺诈识别**
    
-   **市场情报**
    
-   **风险管理**
    
-   **安全分析 等。**
    

## **小结**

**以上都是非常非常基础的问题，更多大厂笔试、面试真题拆解分析推荐看 Elastic 面试系列专题文章。**

**面试要“以和为贵”、不要搞窝里斗， Elastic 面试官要讲“面德“，点到为止！**

**应聘者也要注意：不要大意！面试官都是”有备而来”，针对较难的问题，要及时“闪”，要做到“全部防出去”。**

**如果遇到应聘者有回答 不上来的，面试官要：“耗子猥汁“，而应聘者要好好反思，以后不要再犯这样的错误。**


## 1、elasticsearch 了解多少，说说你们公司 es 的集群架构，索引数据大小，分片有多少，以及一些调优手段 。

-   面试官：想了解应聘者之前公司接触的 ES 使用场景、规模，有没有做过比较大规模的索引设计、规划、调优。
-   解答：如实结合自己的实践场景回答即可。
-   比如：ES 集群架构 13 个节点，索引根据通道不同共 20+索引，根据日期，每日递增 20+，索引：10分片，每日递增 1 亿+数据，每个通道每天索引大小控制：150GB 之内。
-   仅索引层面调优手段：

### 1.1、设计阶段调优

（1）根据业务增量需求，采取基于日期模板创建索引，通过 roll over API 滚动索引；

（2）使用别名进行索引管理；

（3）每天凌晨定时对索引做 force_merge 操作，以释放空间；

（4）采取冷热分离机制，热数据存储到 SSD，提高检索效率；冷数据定期进行 shrink操作，以缩减存储；

（5）采取 curator 进行索引的生命周期管理；

（6）仅针对需要分词的字段，合理的设置分词器；

（7）Mapping 阶段充分结合各个字段的属性，是否需要检索、是否需要存储等。

### 1.2、写入调优

（1）写入前副本数设置为 0；

（2）写入前关闭 refresh_interval 设置为-1，禁用刷新机制；

（3）写入过程中：采取 bulk 批量写入；

（4）写入后恢复副本数和刷新间隔；

（5）尽量使用自动生成的 id。

### 1.3、查询调优

（1）禁用 wildcard；

（2）禁用批量 terms（成百上千的场景）；

（3）充分利用倒排索引机制，能 keyword 类型尽量 keyword；

（4）数据量大时候，可以先基于时间敲定索引再检索；

（5）设置合理的路由机制。

### 1.4、其他调优

-   部署调优，业务调优等。
-   上面的提及一部分，面试者就基本对你之前的实践或者运维经验有所评估了。

## 2、elasticsearch 的倒排索引是什么

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b118b379fcc45e6ba7768e318a141b1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

lucene 从 4+版本后开始大量使用的数据结构是 FST。FST 有两个优点：

（1）空间占用小。通过对词典中单词前缀和后缀的重复利用，压缩了存储空间；

（2）查询速度快。O(len(str))的查询时间复杂度。

## 3、elasticsearch 索引数据多了怎么办，如何调优，部署

面试官：想了解大数据量的运维能力。

解答：索引数据的规划，应在前期做好规划，正所谓“设计先行，编码在后”，这样才能有效的避免突如其来的数据激增导致集群处理能力不足引发的线上客户检索或者其他业务受到影响。

如何调优，正如问题 1 所说，这里细化一下：

### 3.1 动态索引层面

基于模板+时间+rollover api 滚动创建索引，举例：设计阶段定义：blog 索引的模板格式为： blog_index_时间戳的形式，每天递增数据。这样做的好处：不至于数据量激增导致单个索引数据量非 常大，接近于上线 2 的32 次幂-1，索引存储达到了 TB+甚至更大。

一旦单个索引很大，存储等各种风险也随之而来，所以要提前考虑+及早避免。

### 3.2 存储层面

冷热数据分离存储，热数据（比如最近 3 天或者一周的数据），其余为冷数据。

对于冷数据不会再写入新数据，可以考虑定期 force_merge 加 shrink 压缩操作，节省存储空间和检索效率。

### 3.3 部署层面

-   一旦之前没有规划，这里就属于应急策略。
-   结合 ES 自身的支持动态扩展的特点，动态新增机器的方式可以缓解集群压力，注意：如果之前主节点等规划合理，不需要重启集群也能完成动态新增的。

## 4、elasticsearch 是如何实现 master 选举的

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be891153575f418f9cf6552f4fe2f2d0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

```bash
1GET /_cat/nodes?v&h=ip,port,heapPercent,heapMax,id,name
2ip port heapPercent heapMax id name复制代码
复制代码
```

## 5、详细描述一下 Elasticsearch 索引文档的过程

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a08bec544c2c49e3a8663dd030c6d26b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

## 6、详细描述一下 Elasticsearch 搜索的过程？

面试官：想了解 ES 搜索的底层原理，不再只关注业务层面了。

解答：

搜索拆解为“query then fetch” 两个阶段。

query 阶段的目的：定位到位置，但不取。

步骤拆解如下：

（1）假设一个索引数据有 5 主+1 副本 共 10 分片，一次请求会命中（主或者副本分片中）的一个。

（2）每个分片在本地进行查询，结果返回到本地有序的优先队列中。

（3）第 2）步骤的结果发送到协调节点，协调节点产生一个全局的排序列表。

fetch 阶段的目的：取数据。

路由节点获取所有文档，返回给客户端。

## 7、Elasticsearch 在部署时，对 Linux 的设置有哪些优化方法

面试官：想了解对 ES 集群的运维能力。

解答：

（1）关闭缓存 swap;

（2）堆内存设置为：Min（节点内存/2, 32GB）;

（3）设置最大文件句柄数；

（4）线程池+队列大小根据业务需要做调整；

（5）磁盘存储 raid 方式——存储有条件使用 RAID10，增加单节点性能以及避免单节点存储故障。

## 8、lucence 内部结构是什么？

面试官：想了解你的知识面的广度和深度。

解答：

Lucene 是有索引和搜索的两个过程，包含索引创建，索引，搜索三个要点。可以基于这个脉络展开一些。

## 9、Elasticsearch 是如何实现 Master 选举的？

（1）Elasticsearch 的选主是 ZenDiscovery 模块负责的，主要包含 Ping（节点之间通过这个 RPC 来发 现彼此）和 Unicast（单播模块包含一个主机列表以控制哪些节点需要 ping 通）这两部分；

（2）对所有可以成为 master 的节点（node.master: true）根据 nodeId 字典排序，每次选举每个节 点都把自己所知道节点排一次序，然后选出第一个（第 0 位）节点，暂且认为它是 master 节点。

（3）如果对某个节点的投票数达到一定的值（可以成为 master 节点数 n/2+1）并且该节点自己也选 举自己，那这个节点就是 master。否则重新选举一直到满足上述条件。

（4）补充：master 节点的职责主要包括集群、节点和索引的管理，不负责文档级别的管理；data 节点可以关闭 http 功能*。

## 10、Elasticsearch 中的节点（比如共 20 个），其中的 10 个

选了一个 master，另外 10 个选了另一个 master，怎么办？

（1）当集群 master 候选数量不小于 3 个时，可以通过设置最少投票通过数量（discovery.zen.minimum_master_nodes）超过所有候选节点一半以上来解决脑裂问题；

（3）当候选数量为两个时，只能修改为唯一的一个 master 候选，其他作为 data节点，避免脑裂问题。

## 11、客户端在和集群连接时，如何选择特定的节点执行请求的？

TransportClient 利用 transport 模块远程连接一个 elasticsearch 集群。它并不加入到集群中，只是简单的获得一个或者多个初始化的 transport 地址，并以 轮询 的方式与这些地址进行通信。

## 12、详细描述一下 Elasticsearch 索引文档的过程。

协调节点默认使用文档 ID 参与计算（也支持通过 routing），以便为路由提供合适的分片

```scss
shard = hash(document_id) % (num_of_primary_shards)复制代码
复制代码
```

（1）当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 MemoryBuffffer，然后定时（默认是每隔 1 秒）写入到 Filesystem Cache，这个从 MomeryBuffffer 到 Filesystem Cache 的过程就叫做 refresh；

（2）当然在某些情况下，存在 Momery Buffffer 和 Filesystem Cache 的数据可能会丢失，ES 是通过translog 的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中 ， 当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flflush；

（3）在 flflush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync将创建一个新的提交点，并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。

（4）flflush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512M）时；补充：关于 Lucene 的 Segement：

（1）Lucene 索引是由多个段组成，段本身是一个功能齐全的倒排索引。

（2）段是不可变的，允许 Lucene 将新的文档增量地添加到索引中，而不用从头重建索引。

（3）对于每一个搜索请求而言，索引中的所有段都会被搜索，并且每个段会消耗CPU 的时钟周、文件句柄和内存。这意味着段的数量越多，搜索性能会越低。

（4）为了解决这个问题，Elasticsearch 会合并小段到一个较大的段，提交新的合并段到磁盘，并删除那些旧的小段。

## 13、Elasticsearch 是一个分布式的 RESTful 风格的搜索和数据分析引擎。

（1）查询 ： Elasticsearch 允许执行和合并多种类型的搜索 — 结构化、非结构化、地理位置、度量指标 — 搜索方式随心而变。

（2）分析 ： 找到与查询最匹配的十个文档是一回事。但是如果面对的是十亿行日志，又该如何解读呢？Elasticsearch 聚合让您能够从大处着眼，探索数据的趋势和模式。

（3）速度 ： Elasticsearch 很快。真的，真的很快。

（4）可扩展性 ： 可以在笔记本电脑上运行。 也可以在承载了 PB 级数据的成百上千台服务器上运行。

（5）弹性 ： Elasticsearch 运行在一个分布式的环境中，从设计之初就考虑到了这一点。

（6）灵活性 ： 具备多个案例场景。数字、文本、地理位置、结构化、非结构化。所有的数据类型都欢迎。

（7）HADOOP & SPARK ： Elasticsearch + Hadoop

## 14、Elasticsearch是一个高度可伸缩的开源全文搜索和分析引擎。它允许您快速和接近实时地存储、搜索和分析大量数据。

这里有一些使用Elasticsearch的用例：

（1）你经营一个网上商店，你允许你的顾客搜索你卖的产品。在这种情况下，您可以使用Elasticsearch来存储整个产品目录和库存，并为它们提供搜索和自动完成建议。

（2）你希望收集日志或事务数据，并希望分析和挖掘这些数据，以查找趋势、统计、汇总或异常。在这种情况下，你可以使用loghide (Elasticsearch/ loghide /Kibana堆栈的一部分)来收集、聚合和解析数据，然后让loghide将这些数据输入到Elasticsearch中。一旦数据在Elasticsearch中，你就可以运行搜索和聚合来挖掘你感兴趣的任何信息。

（3）你运行一个价格警报平台，允许精通价格的客户指定如下规则:“我有兴趣购买特定的电子设备，如果下个月任何供应商的产品价格低于X美元，我希望得到通知”。在这种情况下，你可以抓取供应商的价格，将它们推入到Elasticsearch中，并使用其反向搜索(Percolator)功能来匹配价格走势与客户查询，并最终在找到匹配后将警报推送给客户。

（4）你有分析/业务智能需求，并希望快速调查、分析、可视化，并对大量数据提出特别问题(想想数百万或数十亿的记录)。在这种情况下，你可以使用Elasticsearch来存储数据，然后使用Kibana (Elasticsearch/ loghide /Kibana堆栈的一部分)来构建自定义仪表板，以可视化对您来说很重要的数据的各个方面。此外，还可以使用Elasticsearch聚合功能对数据执行复杂的业务智能查询。

## 15、详细描述一下 Elasticsearch 更新和删除文档的过程。

（1）删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更；

（2）磁盘上的每个段都有一个相应的.del 文件。当删除请求发送后，文档并没有真的被删除，而是在.del 文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del 文件中被标记为删除的文档将不会被写入新段。

（3）在新的文档被创建时，Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del 文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉

## 16、详细描述一下 Elasticsearch 搜索的过程。

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b99b31d22d954939a2d1cb8edd4e0cd7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

## 17、在 Elasticsearch 中，是怎么根据一个词找到对应的倒排索引的？

（1）Lucene的索引过程，就是按照全文检索的基本过程，将倒排表写成此文件格式的过程。

（2）Lucene的搜索过程，就是按照此文件格式将索引进去的信息读出来，然后计算每篇文档打分(score)的过程。

## 18、Elasticsearch 在部署时，对 Linux 的设置有哪些优化方法？

（1）64 GB 内存的机器是非常理想的， 但是 32 GB 和 16 GB 机器也是很常见的。少于 8 GB 会适得其反。

（2）如果你要在更快的 CPUs 和更多的核心之间选择，选择更多的核心更好。多个内核提供的额外并发远胜过稍微快一点点的时钟频率。

（3）如果你负担得起 SSD，它将远远超出任何旋转介质。 基于 SSD 的节点，查询和索引性能都有提升。如果你负担得起，SSD 是一个好的选择。

（4）即使数据中心们近在咫尺，也要避免集群跨越多个数据中心。绝对要避免集群跨越大的地理距离。

（5）请确保运行你应用程序的 JVM 和服务器的 JVM 是完全一样的。 在Elasticsearch 的几个地方，使用 Java 的本地序列化。

（6）通过设置 gateway.recover_after_nodes、gateway.expected_nodes、gateway.recover_after_time 可以在集群重启的时候避免过多的分片交换，这可能会让数据恢复从数个

小时缩短为几秒钟。

（7）Elasticsearch 默认被配置为使用单播发现，以防止节点无意中加入集群。只有在同一台机器上运行的节点才会自动组成集群。最好使用单播代替组播。

（8）不要随意修改垃圾回收器（CMS）和各个线程池的大小。

（9）把你的内存的（少于）一半给 Lucene（但不要超过 32 GB！），通过ES_HEAP_SIZE 环境变量设置。

（10）内存交换到磁盘对服务器性能来说是致命的。如果内存交换到磁盘上，一个100 微秒的操作可能变成 10 毫秒。 再想想那么多 10 微秒的操作时延累加起来。 不难看出 swapping 对于性能是多么可怕。

（11）Lucene 使用了大 量 的文件。同时，Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字。 所有这一切都需要足够的文件描述符。你应该增加你的文件描述符，设置一个很大的值，如 64,000。

## 19、对于 GC 方面，在使用 Elasticsearch 时要注意什么？

（1）倒排词典的索引需要常驻内存，无法 GC，需要监控 data node 上 segmentmemory 增长趋势。

（2）各类缓存，fifield cache, fifilter cache, indexing cache, bulk queue 等等，要设置合理的大小，并且要应该根据最坏的情况来看 heap 是否够用，也就是各类缓存全部占满的时候，还有 heap 空间可以分配给其他任务吗？避免采用 clear cache等“自欺欺人”的方式来释放内存。

（3）避免返回大量结果集的搜索与聚合。确实需要大量拉取数据的场景，可以采用scan & scroll api来实现。

（4）cluster stats 驻留内存并无法水平扩展，超大规模集群可以考虑分拆成多个集群通过 tribe node连接。

（5）想知道 heap 够不够，必须结合实际应用场景，并对集群的 heap 使用情况做持续的监控。

（6）根据监控数据理解内存需求，合理配置各类circuit breaker，将内存溢出风险降低到最低

## 20、Elasticsearch 对于大数据量（上亿量级）的聚合如何实现？

Elasticsearch 提供的首个近似聚合是 cardinality 度量。它提供一个字段的基数，即该字段的 distinct或者 unique 值的数目。它是基于 HLL 算法的。HLL 会先对我们的输入作哈希运算，然后根据哈希运算 的结果中的 bits 做概率估算从而得到基数。其特点是：可配置的精度，用来控制内存的使用（更精确 ＝ 更多内存）；小的数据集精度是非常高的；我们可以通过配置参数，来设置去重需要的固定内存使用量。无论数千还是数十亿的唯一值，内存使用量只与你配置的精确度相关。

## 21、在并发情况下，Elasticsearch 如果保证读写一致？

（1）可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖，由应用层来处理具体的冲突；

（2）另外对于写操作，一致性级别支持 quorum/one/all，默认为 quorum，即只有当大多数分片可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，分片将会在一个不同的节点上重建。

（3）对于读操作，可以设置 replication 为 sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置 replication 为 async 时，也可以通过设置搜索请求参数_preference 为 primary 来查

询主分片，确保文档是最新版本。

## 22、如何监控 Elasticsearch 集群状态？

Marvel 让你可以很简单的通过 Kibana 监控 Elasticsearch。你可以实时查看你的集群健康状态和性 能，也可以分析过去的集群、索引和节点指标。

23、介绍下你们电商搜索的整体技术架构。

## 24、介绍一下你们的个性化搜索方案？

基于word2vec和Elasticsearch实现个性化搜索

（1）基于word2vec、Elasticsearch和自定义的脚本插件，我们就实现了一个个性化的搜索服务，相对于原有的实现，新版的点击率和转化率都有大幅的提升；

（2）基于word2vec的商品向量还有一个可用之处，就是可以用来实现相似商品的推荐；

（3）使用word2vec来实现个性化搜索或个性化推荐是有一定局限性的，因为它只能处理用户点击历史这样的时序数据，而无法全面的去考虑用户偏好，这个还是有很大的改进和提升的空间；

## 25、是否了解字典树？

常用字典数据结构如下所示：

Trie 的核心思想是空间换时间，利用字符串的公共前缀来降低查询时间的开销以达到提高效率的目的。

它有 3 个基本性质：

1）根节点不包含字符，除根节点外每一个节点都只包含一个字符。

2）从根节点到某一节点，路径上经过的字符连接起来，为该节点对应的字符串。

3）每个节点的所有子节点包含的字符都不相同。

或者用数组来模拟动态。而空间的花费，不会超过单词数×单词长度。

（2）实现：对每个结点开一个字母集大小的数组，每个结点挂一个链表，使用左儿子右兄弟表示法记

录这棵树；

（3）对于中文的字典树，每个节点的子节点用一个哈希表存储，这样就不用浪费太大的空间，而且查

询速度上可以保留哈希的复杂度 O(1)

## 26、拼写纠错是如何实现的？

（1）拼写纠错是基于编辑距离来实现；编辑距离是一种标准的方法，它用来表示经过插入、删除和替换操作从一个字符串转换到另外一个字符串的最小操作步数；

（2）编辑距离的计算过程：比如要计算 batyu 和 beauty 的编辑距离，先创建一个7×8 的表（batyu长度为 5，coffffee 长度为 6，各加 2），接着，在如下位置填入黑色数字。其他格的计算过程是取以下

三个值的最小值：

如果最上方的字符等于最左方的字符，则为左上方的数字。否则为左上方的数字+1。（对于 3,3 来说为0）

左方数字+1（对于 3,3 格来说为 2）

上方数字+1（对于 3,3 格来说为 2）

最终取右下角的值即为编辑距离的值 3。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4583e9041b6340f885b3c6001ba27ce2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05d3031f047643e582b3c5f8e79b3614~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)


# Reference
https://jishuin.proginn.com/p/763bfbd31fed
https://juejin.cn/post/6958408979235995655/#heading-15