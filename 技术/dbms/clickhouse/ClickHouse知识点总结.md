
# ClickHouse简介

ClickHouse 是列式存储数据库，使用 C++ 语言编写，主要用于在线分析处理查询（OLAP），能够使用 SQL 查询实时生成分析数据报告。

# ClickHouse 的特点

## 列式存储与压缩

列式储存的好处：

➢ 对于列的聚合，计数，求和等统计操作原因优于行式存储。

➢ 由于某一列的数据类型都是相同的，针对于数据存储更容易进行数据压缩，每一列 选择更优的数据压缩算法，大大提高了数据的压缩比重。

➢ 由于数据压缩比更好，一方面节省了磁盘空间，另一方面对于 cache 也有了更大的 发挥空间。

ClickHouse服务为了节省磁盘空间，会使用高性能压缩算法对存储的数据进行压缩。默认启用的是lz4（lz4 fast compression）压缩算法，在MergeTree家族引擎下可以通过ClickHouse服务端配置中的compression节点选项配置来改变默认的压缩算法。

## 合理利用磁盘

当使用内存到达到阈值，还可以通过配置参数进行磁盘groupby、orderby

## 数据分区与线程级并行

ClickHouse 会利用服务器的一切必要资源，从而以最自然的方式并行化处理大规模查询。

ClickHouse 将数据划分为多个 partition，每个 partition 再进一步划分为多个 index granularity(索引粒度)，然后通过多个 CPU核心分别处理其中的一部分来实现并行数据处理。 在这种设计下，单条 Query 就能利用整机所有 CPU。极致的并行处理能力，极大的降低了查 询延时。 所以，ClickHouse 即使对于大量数据的查询也能够化整为零平行处理。但是有一个弊端 就是对于单条查询使用多 cpu，就不利于同时并发多条查询。所以对于高 qps 的查询业务， ClickHouse 并不是强项。

## 分布式查询

在ClickHouse中，数据可以保存在不同的shard上，每一个shard都由一组用于容错的replica组成，查询可以并行地在所有shard上进行处理。这些对用户来说是透明的

## 支持 SQL

ClickHouse几乎覆盖了标准 SQL 的大部分语法，包括 DDL 和 DML，以及配套的各种函数，用户管 理及权限管理，数据的备份与恢复。

## 多样化向量引擎

ClickHouse 和 MySQL 类似，把表级的存储引擎插件化，根据表的不同需求可以设定不同 的存储引擎。目前包括合并树、日志、接口和其他四大类 20 多种引擎。

ClickHouse 采用了向量引擎技术，可以更为高效地使用 CPU 资源。

向量执行引擎其实就是利用了CPU的SIMD指令来处理计算，从而达到一条指令运行多个操作（即并行）的效果。

## 高吞吐写入能力

ClickHouse 采用类 LSM Tree的结构，数据写入后定期在后台 Compaction。通过类 LSM tree 的结构，ClickHouse 在数据导入时全部是顺序 append 写，写入后数据段不可更改，在后台 compaction 时也是多个段 merge sort 后顺序写回磁盘。顺序写的特性，充分利用了磁盘的吞吐能力，即便在 HDD 上也有着优异的写入性能。

## 支持索引

ClickHouse 按照排序键对数据进行排序并支持主键索引，可以使其在几十毫秒内完成对特定值或特定范围的查找。

## 支持近似计算

ClickHouse 提供了许多在允许牺牲数据精度的情况下对查询进行加速的方法。

## 语法优化

ClickHouse 提供了基于RBO(Rule Based Optimization)的SQL 优化规则

# ClickHouse 的缺点

1）不支持事务。

2）不擅长根据主键按照行粒度进行查询，故不能把ClickHouse当作Key-Value数据库来使用。

3）不擅长按行删除、修改数据。

4）不擅长高 qps 的查询业务，单条查询使用多 cpu

5）不擅长join

# 表引擎

## TinyLog

以列文件的形式保存在磁盘上，不支持索引，没有并发控制。一般保存少量数据的小表， 生产环境上作用有限。

## Memory

内存引擎，数据以未压缩的原始形式直接保存在内存当中，服务器重启数据就会消失。 读写操作不会相互阻塞，不支持索引。简单查询下有非常非常高的性能表现。

## MergeTree

ClickHouse 中最强大的表引擎当属 MergeTree（合并树）引擎及该系列（*MergeTree） 中的其他引擎，支持索引和分区。

```sql
create table t_order_mt(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time Datetime
) engine =MergeTree
    partition by toYYYYMMDD(create_time)
    primary key (id)
    order by (id,sku_id);
```
### **partition by 分区（可选）**

分区的目的主要是降低扫描的范围，优化查询速度

MergeTree 是以列文件+索引文件+表定义文件组成的，但是如果设定了分区那么这些文 件就会保存到不同的分区目录中。

分区后，面对涉及跨分区的查询统计，ClickHouse 会以分区为单位并行处理。

任何一个批次的数据写入都会产生一个临时分区，不会纳入任何一个已有的分区。写入 后的某个时刻（大概 10-15 分钟后），ClickHouse 会自动执行合并操作（等不及也可以手动 通过 optimize 执行），把临时分区的数据，合并到已有分区中。

### **primary key 主键（可选）**

提供了数据的一级索引，但是却不是唯一约束。这就意味着是可以存在相同 primary key 的数据的。

默认情况下主键与排序键相同，所以通常直接使用order by代为指定主键，省略primary key

根据条件通过对主键进行某种形式的二分查找，能够定位到对应的 index granularity,避 免了全表扫描。

index granularity： 直接翻译的话就是索引粒度，指在稀疏索引中两个相邻索引对应数 据的间隔。ClickHouse 中的 MergeTree 默认是 8192。

![](https://img2022.cnblogs.com/blog/2363466/202204/2363466-20220429104732258-217445703.png)

稀疏索引的好处就是可以用很少的索引数据，定位更多的数据，代价就是只能定位到索 引粒度的第一行，然后再进行进行一点扫描。

### **order by（必选）**

order by 设定了分区内的数据按照哪些字段顺序进行有序保存。

order by 是 MergeTree 中唯一一个必填项，甚至比 primary key 还重要，因为当用户不 设置主键的情况，很多处理会依照 order by 的字段进行处理（比如后面会讲的去重和汇总）。

要求：主键必须是 order by 字段的前缀字段。 比如 order by 字段是 (id,sku_id) 那么主键必须是 id 或者(id,sku_id)

### **二级索引**

其中 GRANULARITY N 是设定二级索引对于一级索引粒度的粒度。
```sql
create table t_order_mt2(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time Datetime,
    INDEX a total_amount TYPE minmax GRANULARITY 5
) engine =MergeTree
    partition by toYYYYMMDD(create_time)
    primary key (id)
    order by (id, sku_id);
```
### 数据TTL

TTL 即 Time To Live，MergeTree 提供了可以管理数据表或者列的生命周期的功能。

列级 TTL
```sql
create table t_order_mt3(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2) TTL create_time+interval 10 SECOND,
    create_time Datetime 
) engine =MergeTree
    partition by toYYYYMMDD(create_time)
    primary key (id)
    order by (id, sku_id);
```
表级 TTL
```sql
alter table t_order_mt3 MODIFY TTL create_time + INTERVAL 10 SECOND;
```

## ReplacingMergeTree

ReplacingMergeTree 是 MergeTree 的一个变种，它存储特性完全继承 MergeTree，只是 多了一个去重的功能。 尽管 MergeTree 可以设置主键，但是 primary key 其实没有唯一约束 的功能。如果你想处理掉重复的数据，可以借助这个 ReplacingMergeTree。

ReplacingMergeTree() 填入的参数为版本字段，重复数据保留版本字段值最大的。 如果不填版本字段，默认按照插入顺序保留最后一条。

➢ 实际上是使用 order by 字段作为唯一键

➢ 去重不能跨分区

➢ 只有同一批插入（新版本）或合并分区时才会进行去重

➢ 认定重复的数据保留，版本字段值最大的

➢ 如果版本字段相同则按插入顺序保留最后一笔

## SummingMergeTree

对于不查询明细，只关心以维度进行汇总聚合结果的场景。如果只使用普通的MergeTree 的话，无论是存储空间的开销，还是查询时临时聚合的开销都比较大。 ClickHouse 为了这种场景，提供了一种能够“预聚合”的引擎 SummingMergeTree

➢ 以 SummingMergeTree（）中指定的列作为汇总数据列

➢ 可以填写多列必须数字列，如果不填，以所有非维度列且为数字列的字段为汇总数据列

➢ 以 order by 的列为准，作为维度列

➢ 其他的列按插入顺序保留第一行

➢ 不在一个分区的数据不会被聚合

➢ 只有在同一批次插入(新版本)或分片合并时才会进行聚合

# SQL操作

## Insert

基本与标准 SQL（MySQL）基本一致

（1）标准 insert into [table_name] values(…),(….)

（2）从表到表的插入 insert into [table_name] select a,b,c from [table_name_2]

## Update 和 Delete

ClickHouse 提供了 Delete 和 Update 的能力，这类操作被称为 Mutation 查询，它可以看 做 Alter 的一种。

虽然可以实现修改和删除，但是和一般的 OLTP 数据库不一样，Mutation 语句是一种很 “重”的操作，而且不支持事务。

“重”的原因主要是每次修改或者删除都会导致放弃目标数据的原有分区，重建新分区。 所以尽量做批量的变更，不要进行频繁小数据的操作。

（1）删除操作

alter table t_order_smt delete where sku_id ='sku_001';

（2）修改操作

alter table t_order_smt update total_amount=toDecimal32(2000.00,2) where id =102;

由于操作比较“重”，所以 Mutation 语句分两步执行，同步执行的部分其实只是进行 新增数据新增分区和并把旧分区打上逻辑上的失效标记。直到触发分区合并的时候，才会删 除旧数据释放磁盘空间，一般不会开放这样的功能给用户，由管理员完成

## 查询操作

ClickHouse 基本上与标准 SQL 差别不大

➢ 支持子查询

➢ 支持 CTE(Common Table Expression 公用表表达式 with 子句)

➢ 支持各种 JOIN，但是 JOIN 操作无法使用缓存，所以即使是两次相同的 JOIN 语句， ClickHouse 也会视为两条新 SQL

➢ 窗口函数(官方正在测试中...)

➢ 不支持自定义函数

➢ GROUP BY 操作增加了 with rollup\with cube\with total 用来计算小计和总计。

with rollup：从右至左去掉维度进行小计

with cube ：从右至左去掉维度进行小计，再从左至右去掉维度进行小计

with totals：只计算合计

## alter 操作

1）新增字段

alter table tableName add column newcolname String after col1; 

2）修改字段类型

alter table tableName modify column newcolname String; 

3）删除字段

alter table tableName drop column newcolname;

## 导出数据

clickhouse-client --query "select * from t_order_mt where create_time='2020-06-01 12:00:00'" --format CSVWithNames> /opt/module/data/rs1.csv

# 副本

副本的目的主要是保障数据的高可用性，即使一台 ClickHouse 节点宕机，那么也可以从 其他服务器获得相同的数据。

![](https://img2022.cnblogs.com/blog/2363466/202204/2363466-20220429161939488-961666904.png)

# 分片集群

副本虽然能够提高数据的可用性，降低丢失风险，但是每台服务器实际上必须容纳全量 数据，对数据的横向扩容没有解决。 要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切 分，不同的分片分布到不同的节点上，再通过 Distributed 表引擎把数据拼接起来一同使用。 Distributed 表引擎本身不存储数据，有点类似于 MyCat 之于 MySql，成为一种中间件， 通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据

注意：ClickHouse 的集群是表级别的，实际企业中，大部分做了高可用，但是没有用分 片，避免降低查询性能以及操作集群的复杂性。

集群写入流程（3 分片 2 副本共 6 个节点）

![](https://img2022.cnblogs.com/blog/2363466/202204/2363466-20220429163807255-1139956870.png)

 集群读取流程（3 分片 2 副本共 6 个节点）

![](https://img2022.cnblogs.com/blog/2363466/202204/2363466-20220429163839265-1795192325.png)

# Explain 查看执行计划

EXPLAIN [AST | SYNTAX | PLAN | PIPELINE] [setting = value, ...] SELECT ... [FORMAT ...]

➢ PLAN：用于查看执行计划，默认值。

◼ header 打印计划中各个步骤的 head 说明，默认关闭，默认值 0;

◼ description 打印计划中各个步骤的描述，默认开启，默认值 1；

◼ actions 打印计划中各个步骤的详细信息，默认关闭，默认值 0。

➢ AST ：用于查看语法树;

➢ SYNTAX：用于优化语法;

➢ PIPELINE：用于查看 PIPELINE 计划。

◼ header 打印计划中各个步骤的 head 说明，默认关闭;

◼ graph 用 DOT 图形语言描述管道图，默认关闭，需要查看相关的图形需要配合 graphviz 查看；

◼ actions 如果开启了 graph，紧凑打印打，默认开启。

老版本查看执行计划

clickhouse-client -h 主机名 --send_logs_level=trace <<< "sql" > /dev/null

# 建表优化

## 时间字段的类型

建表时能用数值型或日期时间型表示的字段就不要用字符串

虽然 ClickHouse 底层将 DateTime 存储为时间戳 Long 类型，但不建议存储 Long 类型， 因为 DateTime 不需要经过函数转换处理，执行效率高、可读性好。

## 空值存储类型

官方已经指出 Nullable 类型几乎总是会拖累性能，因为存储 Nullable 列时需要创建一个 额外的文件来存储 NULL 的标记，并且 Nullable 列无法被索引。因此除非极特殊情况，应直 接使用字段默认值表示空，或者自行指定一个在业务中无意义的值

## 分区和索引

分区粒度根据业务特点决定，不宜过粗或过细。一般选择按天分区，也可以指定为 Tuple()， 以单表一亿数据为例，分区大小控制在 10-30 个为最佳。

必须指定索引列，ClickHouse 中的索引列即主键列，而主键必须是 order by 字段的前缀字段。一般在查询条件中经常被用来充当筛选条件的属性被纳入进来；可以是单一维度，也可以是组合维度的索引；通常需要满足高级列在前、查询频率大的在前原则；还有基数特别大的不适合做索引列， 如用户表的 userid 字段；通常筛选后的数据满足在百万以内为最佳。

## 表参数

Index_granularity 是用来控制索引粒度的，默认是 8192，如非必须不建议调整。

如果表中不是必须保留全量历史数据，建议指定 TTL（生存时间值），可以免去手动过期 历史数据的麻烦，TTL 也可以通过 alter table 语句随时修改。

## 写入和删除优化

（1）尽量不要执行单条或小批量删除和插入操作，这样会产生小分区文件，给后台 Merge 任务带来巨大压力

（2）不要一次写入太多分区，或数据写入太快，数据写入太快会导致 Merge 速度跟不 上而报错，一般建议每秒钟发起 2-3 次写入操作，每次操作写入 2w~5w 条数据（依服务器 性能而定）

处理方式： “ Too many parts 处理 ” ：使用 WAL 预写日志，提高写入性能。

in_memory_parts_enable_wal 默认为 true

在服务器内存充裕的情况下增加内存配额，一般通过 max_memory_usage 来实现 在服务器内存不充裕的情况下，建议将超出部分内容分配到系统硬盘上，但会降低执行 速度，一般通过 max_bytes_before_external_group_by、max_bytes_before_external_sort 参数 来实现。

## 常见配置

配置项主要在 config.xml 或 users.xml 中， 基本上都在 users.xml 里

CPU 资源

![](https://img2022.cnblogs.com/blog/2363466/202204/2363466-20220430103249905-2135413658.png)

内存

![](https://img2022.cnblogs.com/blog/2363466/202204/2363466-20220430103317987-78443480.png)

存储

ClickHouse 不支持设置多数据目录，为了提升数据 io 性能，可以挂载虚拟券组，一个券 组绑定多块物理磁盘提升读写性能，多数据查询场景 SSD 会比普通机械硬盘快 2-3 倍

# 语法优化规则

ClickHouse 的 SQL 优化规则是基于 RBO(Rule Based Optimization)，下面是一些优化规则

## COUNT 优化

在调用 count 函数时，如果使用的是 count() 或者 count(*)，且没有 where 条件，则 会直接使用 system.tables 的 total_rows

如果 count 具体的列字段，则不会使用此项优化：

## 谓词下推

当 group by 有 having 子句，但是没有 with cube、with rollup 或者 with totals 修饰的时 候，having 过滤会下推到 where 提前过滤。

子查询也支持谓词下推

## 聚合函数消除

如果对聚合键，也就是 group by key 使用 min、max、any 聚合函数，则将函数消除

## 标量替换

如果子查询只返回一行数据，在被引用的时候用标量替换

## 三元运算优化

如果开启了 optimize_if_chain_to_multiif 参数，三元运算符会被替换成 multiIf 函数

## 聚合计算外推

## 删除重复的 order by key

## 删除重复的 limit by key

## 删除重复的 USING Key

## 消除子查询重复字段

# 查询优化

## 单表查询

### Prewhere 替代 where

prewhere 只支持 *MergeTree 族系列引擎的表，首先会读取指定的列数据，来判断数据过滤，等待数据过滤 之后再读取 select 声明的列字段来补全其余属性。

当查询列明显多于筛选列时使用 Prewhere 可十倍提升查询性能，Prewhere 会自动优化 执行过滤阶段的数据读取方式，降低 io 操作。 在某些场合下，prewhere 语句比 where 语句处理的数据量更少性能更高。

默认情况下， where 条件会自动优化成 prewhere，但是某些场景即使开启优 化，也不会自动转换成 prewhere，需要手动指定 prewhere：

⚫ 使用常量表达式

⚫ 使用默认值为 alias 类型的字段

⚫ 包含了 arrayJOIN，globalIn，globalNotIn 或者 indexHint 的查询

⚫ select 查询的列字段和 where 的谓词相同

⚫ 使用了主键字段

### 数据采样

通过采样运算可极大提升数据分析的性能

采样修饰符只有在 MergeTree engine 表中才有效，且在创建表时需要指定采样策略。
```sql
SELECT Title,count(*) AS PageViews 
FROM hits_v1
SAMPLE 0.1 #代表采样 10%的数据,也可以是具体的条数
WHERE CounterID =57
GROUP BY Title
ORDER BY PageViews DESC LIMIT 1000
```
### 列裁剪与分区裁剪

数据量太大时应避免使用 select * 操作，查询的性能会与查询的字段大小和数量成线性 表换，字段越少，消耗的 io 资源越少，性能就会越高。

分区裁剪就是只读取需要的分区，在过滤条件中指定。

### orderby 结合 where、limit

千万以上数据集进行 order by 查询时需要搭配 where 条件和 limit 语句一起使用。

### 避免构建虚拟列

如非必须，不要在结果集上构建虚拟列，虚拟列非常消耗资源浪费性能，可以考虑在前 端进行处理，或者在表中构造实际字段进行额外存储。

### uniqCombined 替代 distinct

性能可提升 10 倍以上，uniqCombined 底层采用类似 HyperLogLog 算法实现，能接收 2% 左右的数据误差，可直接使用这种去重方式提升查询性能。Count(distinct )会使用 uniqExact 精确去重。

不建议在千万级不同数据上执行 distinct 去重查询，改为近似去重 uniqCombined

### 使用物化视图

### 其他注意事项

（1）查询熔断

为了避免因个别慢查询引起的服务雪崩的问题，除了可以为单个查询设置超时以外，还 可以配置周期熔断，在一个查询周期内，如果用户频繁进行慢查询操作超出规定阈值后将无 法继续进行查询操作。

（2）关闭虚拟内存

物理内存和虚拟内存的数据交换，会导致查询变慢，资源允许的情况下关闭虚拟内存。

（3）配置 join_use_nulls

为每一个账户添加 join_use_nulls 配置，左表中的一条记录在右表中不存在，右表的相 应字段会返回该字段相应数据类型的默认值，而不是标准 SQL 中的 Null 值。

（4）批量写入时先排序

批量写入数据时，必须控制每个批次的数据中涉及到的分区的数量，在写入之前最好对 需要导入的数据进行排序。无序的数据或者涉及的分区太多，会导致 ClickHouse 无法及时对 新导入的数据进行合并，从而影响查询性能。

（5）关注 CPU

cpu 一般在 50%左右会出现查询波动，达到 70%会出现大范围的查询超时，cpu 是最关 键的指标，要非常关注。

## 多表关联

### 用 IN 代替 JOIN

当多表联查时，查询的数据仅从其中一张表出时，可考虑用 IN 操作而不是 JOIN

select a.* from hits_v1 a where a. CounterID in (select CounterID from visits_v1);
#反例：使用 join
select a.* from hits_v1 a left join visits_v1 b on a. CounterID=b.CounterID;

### 大小表 JOIN

多表 join 时要满足小表在右的原则，右表关联时被加载到内存中与左表进行比较， ClickHouse 中无论是 Left join 、Right join 还是 Inner join 永远都是拿着右表中的每一条记录 到左表中查找该记录是否存在，所以右表必须是小表。

### 注意谓词下推（版本差异）

ClickHouse 在 join 查询时不会主动发起谓词下推的操作，需要每个子查询提前完成过滤操作，需要注意的是，是否执行谓词下推，对性能影响差别很大（新版本中已经不存在此问 题，但是需要注意谓词的位置的不同依然有性能的差异）

### 分布式表使用 GLOBAL

两张分布式表上的 IN 和 JOIN 之前必须加上 GLOBAL 关键字，右表只会在接收查询请求 的那个节点查询一次，并将其分发到其他节点上。如果不加 GLOBAL 关键字的话，每个节点 都会单独发起一次对右表的查询，而右表又是分布式表，就导致右表一共会被查询 N²次（N 是该分布式表的分片数量），这就是查询放大，会带来很大开销。

对于一个例如N台集群的集群，对于集群内部，如果使用IN的话需要进行N²次查询；而GLOABL IN的就只需要进行2N次。

### 使用字典表

将一些需要关联分析的业务创建成字典表进行 join 操作，前提是字典表不宜太大，因为 字典表会常驻内存

### 提前过滤

通过增加逻辑过滤可以减少数据扫描，达到提高执行速度及降低内存消耗的目的

# 数据一致性

查询 CK 手册发现，即便对数据一致性支持最好的 Mergetree，也只是保证最终一致性

## 手动 OPTIMIZE

在写入数据后，立刻执行 OPTIMIZE 强制触发新写入分区的合并动作。

## 通过 Group by 去重

（1）执行去重的查询
```sql
SELECT
    user_id ,
    argMax(score, create_time) AS score, 
    argMax(deleted, create_time) AS deleted,
    max(create_time) AS ctime 
FROM test_a 
GROUP BY user_id
HAVING deleted = 0;
```
argMax(field1，field2):按照 field2 的最大值取 field1 的值。

（2）创建视图
```sql
CREATE VIEW view_test_a AS
SELECT
    user_id ,
    argMax(score, create_time) AS score, 
    argMax(deleted, create_time) AS deleted,
    max(create_time) AS ctime 
FROM test_a 
GROUP BY user_id
HAVING deleted = 0;
```
（3）插入重复数据，再次查询
```sql
#再次插入一条数据
INSERT INTO TABLE test_a(user_id,score,create_time) VALUES(0,'AAAA',now())
#再次查询
SELECT *
FROM view_test_a
WHERE user_id = 0;
```
（4）删除数据测试
```sql
#再次插入一条标记为删除的数据
INSERT INTO TABLE test_a(user_id,score,deleted,create_time) 
VALUES(0,'AAAA',1,now());
#再次查询，刚才那条数据看不到了
SELECT *
FROM view_test_a
WHERE user_id = 0;

```
## 通过 FINAL 查询

在查询语句后增加 FINAL 修饰符，这样在查询的过程中将会执行 Merge 的特殊逻辑（例 如数据去重，预聚合等）。

但是这种方法在早期版本基本没有人使用，因为在增加 FINAL 之后，我们的查询将会变 成一个单线程的执行过程，查询速度非常慢。

在 v20.5.2.7-stable 版本中，FINAL 查询支持多线程执行，并且可以通过 max_final_threads 参数控制单个查询的线程数。但是目前读取 part 部分的动作依然是串行的。

FINAL 查询最终的性能和很多因素相关，列字段的大小、分区的数量等等都会影响到最 终的查询时间，所以还要结合实际场景取舍。

select * from visits_v1 final WHERE StartDate = '2014-03-17' limit 100 settings max_final_threads = 2;

# 物化视图

ClickHouse 的物化视图是一种查询结果的持久化，它确实是给我们带来了查询效率的提 升。用户查起来跟表没有区别，它就是一张表，它也像是一张时刻在预计算的表，创建的过 程它是用了一个特殊引擎，加上后来 as select，就是 create 一个 table as select 的写法。

“查询结果集”的范围很宽泛，可以是基础表中部分数据的一份简单拷贝，也可以是多 表 join 之后产生的结果或其子集，或者原始数据的聚合指标等等。所以，物化视图不会随着基础表的变化而变化，所以它也称为快照（snapshot）

## 物化视图与普通视图的区别

普通视图不保存数据，保存的仅仅是查询语句，查询的时候还是从原表读取数据，可以 将普通视图理解为是个子查询。物化视图则是把查询的结果根据相应的引擎存入到了磁盘 或内存中，对数据重新进行了组织，你可以理解物化视图是完全的一张新表。

## 优缺点

优点：查询速度快，要是把物化视图这些规则全部写好，它比原数据查询快了很多，总 的行数少了，因为都预计算好了。

缺点：它的本质是一个流式数据的使用场景，是累加式的技术，所以要用历史数据做去重、去核这样的分析，在物化视图里面是不太好用的。在某些场景的使用也是有限的。而且 如果一张表加了好多物化视图，在写这张表的时候，就会消耗很多机器的资源，比如数据带 宽占满、存储一下子增加了很多。

## 基本语法

CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ...

1）创建物化视图的限制

1.必须指定物化视图的 engine 用于数据存储

2.TO [db].[table]语法的时候，不得使用 POPULATE。

3.查询语句(select）可以包含下面的子句： DISTINCT, GROUP BY, ORDER BY, LIMIT…

4.物化视图的 alter 操作有些限制，操作起来不大方便。

5.若物化视图的定义使用了 TO [db.]name 子语句，则可以将目标表的视图 卸载 DETACH 再装载 ATTACH（分区数据的迁移或备份）

2）物化视图的数据更新

（1）物化视图创建好之后，若源表被写入新数据则物化视图也会同步更新

（2）POPULATE 关键字决定了物化视图的更新策略：

◼ 若有 POPULATE 则在创建视图的过程会将源表已经存在的数据一并导入，类似于 create table ... as

◼ 若无 POPULATE 则物化视图在创建之后没有数据，只会在创建只有同步之后写入源表的数据

◼ clickhouse 官方并不推荐使用 POPULATE，因为在创建物化视图的过程中同时写入的数据不能被插入物化视图。

（3）物化视图不支持同步删除，若源表的数据不存在（删除了）则物化视图的数据仍然保留

（4）物化视图是一种特殊的数据表，可以用 show tables 查看

# MaterializeMySQL

ClickHouse 20.8.2.3 版本新增加了 MaterializeMySQL 的 database 引擎，该 database 能 映 射 到 MySQL 中 的 某 个 database ， 并 自 动 在 ClickHouse 中 创 建 对 应 的 ReplacingMergeTree。ClickHouse 服务做为 MySQL 副本，读取 Binlog 并执行 DDL 和 DML 请 求，实现了基于 MySQL Binlog 机制的业务数据库实时同步功能。

## 特点

（1）MaterializeMySQL 同时支持全量和增量同步，在 database 创建之初会全量同步 MySQL 中的表和数据，之后则会通过 binlog 进行增量同步。

（2）MaterializeMySQL database 为其所创建的每张 ReplacingMergeTree 自动增加了 _sign 和 _version 字段。

其中，_version 用作 ReplacingMergeTree 的 ver 版本参数，每当监听到 insert、update 和 delete 事件时，在 databse 内全局自增。而 _sign 则用于标记是否被删除，取值 1 或 者 -1。

目前 MaterializeMySQL 支持如下几种 binlog 事件:

➢ MYSQL_WRITE_ROWS_EVENT: _sign = 1，_version ++

➢ MYSQL_DELETE_ROWS_EVENT: _sign = -1，_version ++

➢ MYSQL_UPDATE_ROWS_EVENT: 新数据 _sign = 1

➢ MYSQL_QUERY_EVENT: 支持 CREATE TABLE 、DROP TABLE 、RENAME TABLE 等。

## 使用细则

（1）DDL 查询

MySQL DDL 查询被转换成相应的 ClickHouse DDL 查询（ALTER, CREATE, DROP, RENAME）。 如果 ClickHouse 不能解析某些 DDL 查询，该查询将被忽略。

（2）数据复制

MaterializeMySQL 不支持直接插入、删除和更新查询，而是将 DDL 语句进行相应转换：

MySQL INSERT 查询被转换为 INSERT with _sign=1。

MySQL DELETE 查询被转换为 INSERT with _sign=-1。

MySQL UPDATE 查询被转换成 INSERT with _sign=1 和 INSERT with _sign=-1。

（3）SELECT 查询

如果在 SELECT 查询中没有指定_version，则使用 FINAL 修饰符，返回_version 的最大值 对应的数据，即最新版本的数据。

如果在 SELECT 查询中没有指定_sign，则默认使用 WHERE _sign=1，即返回未删除状态 （_sign=1)的数据。

（4）索引转换

ClickHouse数据库表会自动将 MySQL 主键和索引子句转换为 ORDER BY 元组。

ClickHouse只有一个物理顺序，由 ORDER BY 子句决定。如果需要创建新的物理顺序， 请使用物化视图。

## 实现

### MySQL 开启 binlog 和 GTID 模式

（1）打开/etc/my.cnf,在[mysqld]下添加：

server-id=1

log-bin=mysql-bin

binlog_format=ROW

（2）开启 GTID 模式

如果如果 clickhouse 使用的是 20.8 prestable 之后发布的版本，那么 MySQL 还需要配置 开启 GTID 模式, 这种方式在 mysql 主从模式下可以确保数据同步的一致性(主从切换时)。

gtid-mode=on

enforce-gtid-consistency=1 # 设置为主从强一致性

log-slave-updates=1 # 记录日志

GTID 是 MySQL 复制增强版，从 MySQL 5.6 版本开始支持，目前已经是 MySQL 主流 复制模式。它为每个 event 分配一个全局唯一 ID 和序号，我们可以不用关心 MySQL 集群 主从拓扑结构，直接告知 MySQL 这个 GTID 即可。

（3）重启 MySQL

### 开启 ClickHouse 物化引擎

set allow_experimental_database_materialize_mysql=1;

### 创建复制管道

ClickHouse 中创建 MaterializeMySQL 数据库

CREATE DATABASE test_binlog ENGINE = MaterializeMySQL('hadoop1:3306','testck','root','000000');

其中 4 个参数分别是 MySQL 地址、databse、username 和 password。

# Reference
https://www.cnblogs.com/jpppp/p/16204697.html#ClickHouse%E7%AE%80%E4%BB%8B