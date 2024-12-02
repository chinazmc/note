
## **01/游族 OLAP 系统历史背景**

  

### **1. 历史背景与痛点**

  

![](https://pic1.zhimg.com/v2-d2bacb363625f69148d850ae09671148_1440w.jpg)

  

首先分享一下我们的历史背景，上图是我们之前做实时和[离线指标](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E7%A6%BB%E7%BA%BF%E6%8C%87%E6%A0%87&zhida_source=entity)计算所使用的一些组件：

- 分钟级别调度的指标计算：用 Presto 或者是 Clickhouse。
- Kafka数据流的计算：用 SparkStreaming 或者 Flink 去读取并计算。
- [标签表](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%A0%87%E7%AD%BE%E8%A1%A8&zhida_source=entity)的计算：会导入一些标签表到 HBase 里面，然后通过 Data API 的方式去提供给其他的系统使用（比如我们公司是做游戏的，会有一些玩家的标签表在对接客服之类的系统，他们会实时去查看每一个玩家的信息，进行一些问题的解答，我们会提供这样的数据）。
- 报表展示：报表的实时指标的结果会落到 MySQL 库里面，[报表系统](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%8A%A5%E8%A1%A8%E7%B3%BB%E7%BB%9F&zhida_source=entity)会直接读取 MySQL 作为指标的展示。

**这些组件其实各有优势**：比如 Presto [直联](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E7%9B%B4%E8%81%94&zhida_source=entity) Hive，不需要做其他的操作，就可以做一些自主分析；ClickHouse 的一些单表，查询性能也特别好。

**但是慢慢的演变之后，我们就会产生特别多的组件，给我们带来了一些痛点**：

- 首先组件太多，维护多套组件的运维成本是比较高的。
- 其次，各组件的 SQL 语法存在差异，特别是 ClickHouse 不支持标准 SQL，所以开发维护任务的成本也会比较高。
- 第三，同指标数据因为有多套系统在计算，需要去保证不同组件计算的结果和口径都一样，成本也是比较高。
- 最后，我们的结果数据是落在 MySQL 里面的，有一些维度比较多的数据结果数据量是比较大的，需要对 MySQL 通过分表去支持数据的存储和查询，然而数据量大的时候，即使分表，其查询性能也比较差，所以我们的报表系统时间上响应会比较慢。

  

### **2. 诉求**

为了解决以上痛点，我们需要选择统一的 OLAP 引擎，该引擎至少要满足以下要求：

- **数据秒级写入，低延迟毫秒级响应**
- **复杂场景多表关联查询性能好**
- **运维简单，方便扩展**
- **支持[高并发](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E9%AB%98%E5%B9%B6%E5%8F%91&zhida_source=entity)**
- **易用性强，开发简单方便**

我们对比了市面上一些组件，希望用一款存算一体的组件去优化我们的整个架构。首先，ClickHouse 的使用和运维比较困难，并且多表关联的性能比较差，所以我们没有选择 ClickHouse。我们又对比了 StarRocks 和 Doris，因为 StarRocks 在性能上会更好，所以我们最终选择了 StarRocks 作为统一的 OLAP 引擎。

--

## **02/StarRocks 的特点和优势**

  

### **1. 极致的[查询性能](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=4&q=%E6%9F%A5%E8%AF%A2%E6%80%A7%E8%83%BD&zhida_source=entity)**

StarRocks 是有着极致的查询性能的，主要得益于以下的这几点：

- **分布式执行 MPP**：一条数据/一条查询请求会被拆分成多个物理的执行单元，可以充分利用所有节点的资源，这样对于查询性能是一个很好的提升。
- **[列式存储引擎](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%88%97%E5%BC%8F%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E&zhida_source=entity)**：对于大多数的 OLAP 引擎来说的话，基本会选择列式存储，因为很多的 OLAP 场景当中，计算基本上只会涉及到部分列的一些提取，所以相对于“行存”来说，[列存](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%88%97%E5%AD%98&zhida_source=entity)只读取部分列的数据，可以极大的降低磁盘 IO。
- **全面向量化引擎**：StarRocks 所有的算子都实现了[向量化](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=2&q=%E5%90%91%E9%87%8F%E5%8C%96&zhida_source=entity)，向量化简单的理解就是它可以消除[程序循环](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E7%A8%8B%E5%BA%8F%E5%BE%AA%E7%8E%AF&zhida_source=entity)的优化，是实现了 Smid 的一个特性，也就是当对一列数据进行相同的操作的时候，可以使用单条指令去操作多条数据，这是一个在 CPU 寄存器层面实行的对数据并行操作的优化。
- **CBO 优化器**：这是 StarRocks 从 0 开始设计的一款全新的优化器，在多表查询或者一些复杂查询的情况下，光有好的一些执行性能是不够的，因为同样的一条 SQL，会有不同的执行计划，不同计划之间的执行性能的差异可能会差几个量级，所以需要一款更好的优化器，才能够选择出相对更优的一个执行计划，从而提升查询效率。

  

![](https://pica.zhimg.com/v2-e55b828f8da9ed7d580ae9452efafe28_1440w.jpg)

  

上面的图表是一个 SSB 的基准测试，把一个[星型模型](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%98%9F%E5%9E%8B%E6%A8%A1%E5%9E%8B&zhida_source=entity)转变成了一个单表，然后用一些 SQL 查询语句去测试。在这种情况下可以看到 StarRocks 的性能是好于 ClickHouse 和 Druid 的。下面的图表是一些低基准数据[聚合查询](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E8%81%9A%E5%90%88%E6%9F%A5%E8%AF%A2&zhida_source=entity)的对比，同样也是 StarRocks 的性能会显得更好一些。

  

### **2. 丰富的导入方式**

  

![](https://pica.zhimg.com/v2-bc1ee8c36ece9036c5c04e6fdd1f95e6_1440w.jpg)

  

StarRocks 还拥有着丰富的导入方式，用户可以根据自己的实际场景选择合适的导入方式。以前使用其他组件时，在大多数情况下我们都会选择一些其他的导入工具来帮助我们完成数据的导入，比如 Sqoop、 DataX，等等一些工具；但是 StarRocks 有丰富的导入方式，所以无需借助其他工具，对接大多数组件都可以通过 StarRocks 提供的这些导入方式去直接完成数据的导入，可以极大节省开发时间。

  

### **3. StarRocks 的优势特点**

StarRocks 还有一些其他优势：

  

![](https://pica.zhimg.com/v2-d5949bde01fff5ff24fd7e8f61166394_1440w.jpg)

  

- **运维简单**：右侧这个图是 StarRocks 一个简单的架构图，只有 FE 和 BE 两种组件，不依赖于外部组件，运维简单，并且也方便扩缩容。
- **丰富的[数据模型](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B&zhida_source=entity)**：StarRocks 支持明细、聚合、更新、主键4种数据模型，同时它还支持[物化视图](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E7%89%A9%E5%8C%96%E8%A7%86%E5%9B%BE&zhida_source=entity)，方便我们针对不同的场景去选择合适的数据模型。
- **简单易用**：StarRocks 兼容 MySQL 协议，支持标准的 SQL 语法，不需要太多的学习成本就可以去直接使用它。
- **支持多种[外部表](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%A4%96%E9%83%A8%E8%A1%A8&zhida_source=entity)**：StarRocks 支持多种外部表，比如 MySQL、ElasticSearch、Hive、StarRocks（这里指另一个集群的 StarRocks）等等，做跨集群的、跨组件的[关联查询](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=2&q=%E5%85%B3%E8%81%94%E6%9F%A5%E8%AF%A2&zhida_source=entity)也无需数据的导入，可以直接建立外部表，基于多个数据源去做关联查询，在一些分析当中是比较方便的。

--

## **03/StarRocks 在游族 OLAP 系统中的应用场景**

  

### **1. 实时计算场景/家长监控中心**

  

![](https://pic3.zhimg.com/v2-0fe0e21cfa42cf0abf03898893d844e2_1440w.jpg)

  

首先分享一个实质性比较高的场景。

- **左侧图看到的是我们的一款小程序**

这款小程序是根据文化部的指导和要求研发的，目的是为了加强家长对未成年参与网络游戏的监护，提供一种家长监护的渠道，可以使得家长能够看到未成年的一些游戏时长的数据或者是充值金额，从而对孩子进行游戏时长的限制和游戏充值的限制。

这需要我们为该游戏提供有各个未成年账号的一些实时的在线数据，或者是充值数据。

- **右侧图是这一需求的数据流转图**

我们会通过读取 Kafka 里面的数据在 Flink 当中进行计算，实时写入StarRocks，再通过 Data API 的方式去提供给小程序使用，因为跨部门协作，所以用 Data API 的方式去提供数据比较安全；我们也有一条离线覆盖的线路，因为在 Flink 计算，难免会有一些上报的数据存在网络延迟，在这个场景中数据都会实时更新到 StarRocks，部分数据的计算可能会有一些差异，所以我们最终要用离线数据去覆盖。

这里因为我们会实时去更新账号信息，StarRocks 可更新的特性也为我们提供了很大的方便。

  

### **2. 实时更新[模型选择](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%A8%A1%E5%9E%8B%E9%80%89%E6%8B%A9&zhida_source=entity)**

  

![](https://pic4.zhimg.com/v2-21c7219cc2ca5ffd304f9f3186912bc1_1440w.jpg)

  

StarRocks 中提供了两种模型可以用于数据的更新，这两种组件的内部机制是有所区别的，所以使用场景也不太一样。

- **更新模型**

内部是使用 Merge on Read 的方式去实现数据的更新的，也就是说 StarRocks 在底层操作的时候不会去更新数据，但是会在查询的时候实时去合并版本，所以同一主键的数据会存储多个版本；这样的好处是在写入的时候会非常流畅，但是也有坏处，在频繁导入数据的时候，主键会存在多个版本的数据，这对于查询性能会有所影响。

- **[主键模型](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E4%B8%BB%E9%94%AE%E6%A8%A1%E5%9E%8B&zhida_source=entity)**

内部是使用 Delete and Insert（删除并更新）的方式，StarRocks 会将主键存于内存中，在数据写入的时候，会去内存中找到这条数据，然后执行一个标记删除的操作，之后会把新的数据插入进去，最后合并的时候只需要过滤掉那些标记删除的数据就可以了，这样它的查询性能会比更新模型更高；因为我们的这一需求实时性要求是比较高的，所以更新特别频繁，最终的使用也是给小程序提供实时的服务，所以我们最终还是会优先考虑查询性能。我们更倾向于去选择主键模型，去作为表的数据模型。

  

### **3. 主键模型不能使用 Delete 方式删除数据**

前文提到离线覆盖实时的一个操作，场景是当我们在数据有一些差异的时候，需要用离线数据覆盖实时数据；使用 StarRocks 的主键模型进行这种操作时，其实并不是用 Delete 的方式去删除数据的，只能够通过 Stream Load、Broker Load、Routine Load 等这三种导入的方式去删除数据，这是非常不方便的，导入时需要先提供一个[标志位](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%A0%87%E5%BF%97%E4%BD%8D&zhida_source=entity)，去标明这是 Upsert 还是 Delete；对于直接写 SQL 语句去删除是非常不友好不方便的，下图中是使用主键模型时删除数据的代码示例。

  

![](https://picx.zhimg.com/v2-84e0f385d523198efd0efe5cb607d531_1440w.jpg)

  

### **4. 软删除**

  

![](https://pic1.zhimg.com/v2-9cadd37675b1421a43bc0e27512c7e7e_1440w.jpg)

  

在这种情形下，我们会选择使用软删除的方式去标记删除，因为 StarRocks主键模型能够更新数据的特性，可以使我们把这些需要删除的数据先查询出来，再变更它的一个删除标志位，这样就可以达到了删除的目的了。

数据在 StarRocks 的更新模型是可以支持删除操作的，但是在这种情况下，**我们为什么还是选择主键模型，而不是去选择更新模型呢？**主要是由以下的 3 点情况：

- 第一，我们这个场景的数据更新，频率是非常高的，所以用更新模型的查询性势必会有所下降。
- 第二，更新模型的删除也是有一些限制的，在删除条件比较复杂的情况下也是无法删除的；比如说只能够根据“[排序列](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%8E%92%E5%BA%8F%E5%88%97&zhida_source=entity)”去删除，或者是删除条件只能用与 AND 不能用或 OR，或者是只能使用一部分的操作符，并不是所有的删除情况、所有的条件下都可以去删除；在我们的场景和删除条件下，在更新模型里面无法满足。
- 第三，我们会用离线数据去覆盖实时数据，这两份数据其实是非常相近的，只会有很少的不一致，所以我们删除的冗余也是很少的；此外这个需求只会保留 30 天的数据，我们也会对表做一个动态分区，让 StarRocks 自己去对这些表的数据做一个生命周期的管理；总结下来，我们删除的无效数据是非常少的，也就是冗余会比较少，因此并不会影响到查询性能。

所以基于这三点原因，我们在这种场景下会更倾向于去选择主键模型。

  

### **5. 报表实时指标计算/架构介绍**

接下来介绍一下报表的一些实时指标的计算，首先报表是固定维度的，我们会有各种时效性的指标，所以**在引进 StarRocks 之后，我们的架构也做了一些变化**。

  

![](https://pica.zhimg.com/v2-a697160b8c2776959060b42fd7bc8092_1440w.jpg)

  

- 首先，Flink 会读取 Kafka 的数据，但是只做一些简单的ETL操作；
- 其次，我们也会去跟 HBase 交互，实时生成比如账号首次登录表，或者是角色首登表这种信息，并且把这些信息关联到日志上面。
- 做完上述操作之后，数据会写入到下级的 Kafka，最终同时写入 Hive 和 StarRocks 里面。
- 最终，指标计算会统一在 StarRocks 里面做分钟级别的调度，完成实时指标的计算，一些数据的逻辑分层也是在这里面做。

**最终我们的报表系统也是以 StarRocks 为核心，直接读取 StarRocks 的结果数据**，不需要再像之前一样，在一个组件里面计算完的数据还导到 MySQL 里面进行展示；此外，我们对外提供的数据存在 StarRocks 里面，也能够**通过一个统一的 Data API 的方式去提供**，因为它是支持多并发的。

  

### **6. [数据关系模型](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E5%85%B3%E7%B3%BB%E6%A8%A1%E5%9E%8B&zhida_source=entity)转变**

  

![](https://pica.zhimg.com/v2-3952548f8e4eee33c5420936425c0ac6_1440w.jpg)

  

在数据建模方面，我们也有一些转变。以前在使用 ClickHouse 时，因为多表 Join 的能力不理想，我们的数据关系模型基本会使用一些大宽表，尽量使用[单表查询](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%8D%95%E8%A1%A8%E6%9F%A5%E8%AF%A2&zhida_source=entity)，以保证查询性能；但[宽表模型](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%AE%BD%E8%A1%A8%E6%A8%A1%E5%9E%8B&zhida_source=entity)的问题是，一旦维度有所变化，回溯的成本是很高的。

在引入 StarRocks 之后，因为它有很好的多表 Join 的能力，所以我们尽量会去考虑星型模型或者[雪花模型](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E9%9B%AA%E8%8A%B1%E6%A8%A1%E5%9E%8B&zhida_source=entity)，当维度不变化的时候，我们不需要回溯的成本，可以直接用多表 Join 的方式去查询数据，同时也把事实表和[维度表](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E7%BB%B4%E5%BA%A6%E8%A1%A8&zhida_source=entity)去解耦，以便去应对更多的灵活分析、多维分析的场景。

  

### **7. 精确一次性保证**

  

![](https://picx.zhimg.com/v2-065e7fe3c4ca8c47cdad74cf312187cd_1440w.jpg)

  

在精准一次性保证的方面，以前我们使用 Flink 写入 ClickHouse 的时候，是无法保证数据的精确一次性的，这样我们在下游做计算时，会去做各种去重的操作，比如说日志 ID 的去重，账号的去重、订单的去重等等。

在引入 StarRocks 之后，我们使用官方的插件 Flink-Connector 去写入，是可以保证数据的精确一次性的。虽然说我们计算原始日志，也会对日志做去重，因为我们不能够保证日志从手机端上报到我们最终落入StarRocks [全链路](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%85%A8%E9%93%BE%E8%B7%AF&zhida_source=entity)的精确一致性；但是对于Flink处理过的数据，能够保证精确一次性，这对我们来说也是非常有意义的，能够减少一些后续的操作。

  

### **8. 指标存储转变**

以前实时计算的结果都会写入 MySQL，需要借助其他工具，比如 Sqoop、DataX，或者是程序写入等等；对于 ClickHouse 可能会用库引擎的方式或者是程序写入。

  

![](https://pic1.zhimg.com/v2-7f0d9bff378b0ba9c01b58c40fe6f15c_1440w.jpg)

  

在引入 StarRocks 之后，实现了算存一体，不需要其他导入；在做[查询分析](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%9F%A5%E8%AF%A2%E5%88%86%E6%9E%90&zhida_source=entity)需要关联其他组件的时候，我们也可以根据其他数据源建立外表，然后直接做查询分析；这对于数据的流通性来说是非常友好的，也更能方便我们的开发工作，比如一些 [adhoc](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=adhoc&zhida_source=entity) 临时的分析也不用导数，直接做一些[多数据源](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%BA%90&zhida_source=entity)的查询就可以了。

  

### **9. 常用数据导入方式**

**实时数据**：使用Flink-connector-StarRocks，其内部实现是通过缓存并批量由 Stream Load 导入 。

**离线数据**：创建 Hive 外表，用 Insert Into Select 方式直接写入结果表。

  

![](https://pic1.zhimg.com/v2-1e2f430d78bf4c6dfdfcb20f2a4dc28a_1440w.jpg)

  

**①[实时数据](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=4&q=%E5%AE%9E%E6%97%B6%E6%95%B0%E6%8D%AE&zhida_source=entity)**

我们使用官方的 Flink-Connector 插件导入数据，它的内部是通过缓存并批量由 Stream Load 去导入的，而 Stream Load 是通过 HTTP 协议提交和传输数据。

Flink Sync 数据并要支持精确一次性的话，需要外部系统支持“两阶段提交”的机制；但是 StarRocks 没有这个机制，所以精确一次性的导入依赖于 StarRocks 的 Label 机制，就是说在 StarRocks 当中，所有的导入数据都会有一个 Label 用于标识每一个导入的作业；Label 相同时 StarRocks 会保证数据只导入一次，这样保证 Flink 到 StarRocks 的精确一次性。

Label 默认保存三天，也可以通过参数去调节，但因为该机制下系统要查看它的 Label 是否存在，如果 Label 保存的时间过长，肯定影响导入性能，所以也不可能无限期的保存。只要我们做好任务的监控，在通常情况下，我们是可以保证数据精确一次性写入的。

**②离线数据**

我们目前主要是把以前 Hive 的一些结果[数据迁移](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E8%BF%81%E7%A7%BB&zhida_source=entity)到StarRocks，以及一些离线计算的结果，也会刷到 StarRocks 里面去，所以我们使用Hive外表这种方式，虽然该方式性能不如 Broker Load，但更方便。这种导入方式也有一些需要注意的点，因为 Hive 的源数据信息，包括分区信息以及分区下的一些文件信息，都是存储在 StarRocks FE 中的，所以在更新 Hive 数据的时候，需要及时更新FE的缓存。

在 StarRocks 里面提供了三种缓存的更新方式，首先是自动刷新缓存，默认是两个小时会自动刷新一次；但是我们在导入离线数据的任务中，倾向于用第二种方式，就是手动的去刷新缓存，能保证下一个导入任务执行的时候，缓存就已经更新了；最后一种是自动[增量更新](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E5%A2%9E%E9%87%8F%E6%9B%B4%E6%96%B0&zhida_source=entity)，跟第一种的区别是能够自动的增量去更新，效率更高，所以也能够提升更新频率。

  

### **10. 分区分桶选择**

  

![](https://pic2.zhimg.com/v2-6191baebeda7c4979e128aa5d9a202b1_1440w.jpg)

  

下面分享一下我们在 StarRocks 建表的时候会涉及到的分区分桶的选择。

**①先分区后分桶**：如果不分区，StarRocks 会把整张表作为一个分区；分区是使用 Range 方式，分桶是使用 Hash 方式。

**②分区选择**：通常会从数据管理的角度来选择分区，第一要方便过滤数据；第二大多数情况下会选择时间分区，这样可以使用动态分区，能够自动的帮我们删减分区。

**③分桶选择**：因为分桶是用哈希的方式落到各个文件上面，所以通常会选择一个高基数的列来作为分桶键，这样可以尽量保证数据在各个桶中是均衡的；分桶的个数我们会先去计算它的存储量，然后尽量将每个 Tablet 设置成 500 兆左右。

在使用初期也曾遇到过问题，我们按照官方指南分成8个桶或32个桶，后来发现按天分区的话，一天的数据是在一天这个分区里面去分桶，所以有些表就会小文件特别多，然后在查询的时候，扫描时间会特别长，这样也会降低查询性能，因为分桶是影响查询的并行度的，分桶如果太少，并行度会比较低，太多的话又会导致小文件比较多。

**所以这需要我们在建表设置分桶的时候，就对这个表的数据量有一个预估，然后去选择合适的分桶个数。**

  

### **11. [慢查询](https://zhida.zhihu.com/search?content_id=221644537&content_type=Article&match_order=1&q=%E6%85%A2%E6%9F%A5%E8%AF%A2&zhida_source=entity)分析**

  

![](https://pic3.zhimg.com/v2-fd02fe85f97166c926b309241c7b4278_1440w.jpg)

  

最后介绍下慢查询分析。

StarRocks 也提供了一些慢查询分析工具，比如可以到日志里面去查看慢查询的信息，或者是到页面上去看它的 Profile（如图所示）。

Profile 是 BE 执行后的一个结果，包含了每一步的耗时和处理量的结果。当遇到慢查询时，你可以去具体分析这个 SQL 的执行情况，然后去通过对应的一些信息达到优化。

--

## **04/游族 StarRocks 应用的未来规划**

  

最后分享一下游族对于 StarRocks 使用的一些未来规划。

①首先我们会将剩余的实时场景全部迁入到 StarRocks 里面，建立以StarRocks 为核心的统一的查询分析平台。

②我们之前的 Data API 服务其实是临时的，所以我们也会去完善，建成一个全面的集成 StarRocks 的 Data API

③最后，我们会完善 StarRocks 的一些监控，比如慢查询的监控、任务的监控、性能的监控等等。

--

## **05/问答环节环节**

  

**Q1：谢谢成彬老师的分享，下面是问答环节，欢迎直播间的小伙伴们留言提问。祖玛提了第一个问题，他问家长中心的场景中延迟数据为什么不适用 Flink 在计算，而是用离线数据的方式去覆盖**

A1：首先因为我们每条数据的计算，涉及到 Flink 状态的变更，还有我们 StarRocks 里面的数据也会变更，所以说如果我们要再把延迟的数据加入，会改变掉它原来的一个计算结果，因为我们的计算是有持续性的。

当逻辑复杂的时候，我们还要这样操作的话，首先是我们这样操作会特别的复杂，第二我们这样去回溯也是更浪费资源的，会把之前的很多数据再计算一遍，再就是我们在实现需求的时候，会先去观察我们整体链路的数据延迟情况，然后去设置一个合理的水位线去计算。

所以基于这些原因，我们最终选择了用离线覆盖的方式，去纠正我们的延迟数据。

**Q2：谢谢成彬老师。第二个问题是威克特提的，他的问题是 StarRocks可以确保数据的一致性，为什么还需要 Hive 进行一次数据覆盖？是出于什么考虑？**

A2：因为 StarRocks 它是可以确保数据的一致性；在使用 Flink 实时计算的时候，它的时效性和准确性是有取舍的；因为你不知道可能是用户的网络原因，或者是数据采集的一些过程，都有可能导致数据的延迟。

比如你设置的一个就是实时计算的延迟是一分钟，只要有一条数据它在一分钟甚至是十分钟他都没有来的话，Flink 计算的结果始终是不准确的（因为数据的延迟，在计算的时刻拿不到完整的数据去做计算）。

所以我们用离线计算覆盖，因为离线计算能够保证前一天的所有数据，所有能采集的都已经采集到了，这样的话我们去进行一个离线的覆盖。

# Reference
https://zhuanlan.zhihu.com/p/600804189