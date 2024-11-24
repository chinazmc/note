
### 使用历程

我们在早期业务中处理数据统计汇总的时候，采用的都是MySQL，随着业务的不断发展，我们中途还结合使用了TiDB。但是它们都有着共同的缺点，那就是随着业务的发展和数据量的不断增加，二者汇总的效率也是越来越低。最后已经无法在正常时间内完成其汇总工作，于是我们将目光转向了ClickHouse。  
ClickHouse是一款非常优秀的OLAP数据库，特别擅长处理数据分析和汇总等。在一张3000W+的数据表里面，进行`GROUP BY`，`DISTINCT`，`COUNT`等相关汇总操作的时候，对比TiDB，处理速度上达到了15~30倍的提升。于是我们便将离线数据的汇总分析切换到了ClickHouse处理，将原本12小时都无法汇总分析完的数据，在3个小时左右就可以处理完毕。ClickHouse强悍的处理效率，保证了我们关键数据的及时输出，也避免了我们重跑数据时迟迟无法出结果的尴尬。  
现如今我们已经将数据汇总分析切换到了Hadoop，但是ClickHouse依然活跃在各大业务中，比如错误收集系统。该系统一开始也是MySQL的处理方案，但是随着收集的错误内容增多以及查询的复杂性，页面数据响应的速度也是越来越慢。于是在一次大版本迭代中将存储切换成了ClickHouse，也顺势解决了数据返回响应慢的问题，保证了后续业务的拓展和稳定性。

### 引擎介绍

##### 初识ClickHouse

ClickHouse简单来说就是一个OLAP类型的列式数据库。如果你日常使用的是MySQL，那么在使用ClickHouse时，是没有太大门槛的。但是二者也有不少区别，我们就日常的使用，列举几个稍微不一样的地方：

```sql
-- 对int类型的字段进行模糊搜索：
SELECT `version`, `version_code`
FROM `test_table`
WHERE `project` = 'reader_free'
	AND toString(`version_code`) LIKE '%5%'
GROUP BY `version`, `version_code`
ORDER BY `version_code` DESC
LIMIT 100;

-- 删除数据
ALTER TABLE [db.]table DELETE WHERE filter_expr;

-- 更新数据；注意不能对索引列进行数据更新
ALTER TABLE [db.]table UPDATE column1 = expr1 [, ...] WHERE filter_expr;
```

不过需要注意的是，ClickHouse作为OLAP数据库，也是有不少缺点的，比如不支持事务，不支持高并发，对更新操作不友好等。所以在选择使用的时候，一定要结合自己的业务情况。

##### 四大类表引擎

ClickHouse的表引擎主要有四大类，分别是Log、MergeTree、Integration和Special。简单来说，Log系列用来做小表数据分析，MergeTree系列用来做大数据量分析，Integration系列用于外部数据集成，Special则如其名，比较特殊，一般主要是Replicated复制表和Distributed分布式表结合其它引擎配合使用。  
Log、Special和Integration这三类引擎主要用于特殊用途，场景相对有限。MergeTree引擎才是官方主推的存储引擎，支持ClickHouse几乎所有的核心功能。在我们的日常业务中，基本上使用的也都是该类引擎。  
MergeTree系列引擎主要有以下几个细分：

- [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/#mergetree)
    
    > MergeTree是ClickHouse中最强大的表引擎。在大量数据写入时数据，数据高效的以批次的形式写入，写入完成后在后台会按照一定的规则进行数据合并。注意，Merge引擎不属于*MergeTree系列。
    
- [ReplacingMergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/replacingmergetree/#replacingmergetree)
    
    > 该引擎和 [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/) 的不同之处在于它会删除排序键值相同的重复项。
    
- [SummingMergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/summingmergetree/#summingmergetree)
    
    > 该引擎继承自 [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/)。区别在于，当合并 `SummingMergeTree` 表的数据片段时，ClickHouse 会把所有具有相同主键的行合并为一行，该行包含了被合并的行中具有数值数据类型的列的汇总值。如果主键的组合方式使得单个键值对应于大量的行，则可以显著的减少存储空间并加快数据查询的速度。
    > 
    > 我们推荐将该引擎和 `MergeTree` 一起使用。例如，在准备做报告的时候，将完整的数据存储在 `MergeTree` 表中，并且使用 `SummingMergeTree` 来存储聚合数据。这种方法可以使你避免因为使用不正确的主键组合方式而丢失有价值的数据。
    
- [AggregatingMergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/aggregatingmergetree/#aggregatingmergetree)
    
    > 该引擎继承自 [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/)，并改变了数据片段的合并逻辑。 ClickHouse 会将一个数据片段内所有具有相同主键（准确的说是 [排序键](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/)）的行替换成一行，这一行会存储一系列聚合函数的状态。
    > 
    > 可以使用 `AggregatingMergeTree` 表来做增量数据的聚合统计，包括物化视图的数据聚合。
    
- [CollapsingMergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/collapsingmergetree/#table_engine-collapsingmergetree)
    
    > 该引擎继承于 [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/)，并在数据块合并算法中添加了折叠行的逻辑。
    > 
    > `CollapsingMergeTree` 会异步的删除（折叠）这些除了特定列 `Sign` 有 `1` 和 `-1` 的值以外，其余所有字段的值都相等的成对的行。没有成对的行会被保留。
    
- [VersionedCollapsingMergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/versionedcollapsingmergetree/#versionedcollapsingmergetree)
    
    > 引擎继承自 [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/#table_engines-mergetree) 并将折叠行的逻辑添加到合并数据部分的算法中。 `VersionedCollapsingMergeTree` 用于相同的目的 [折叠树](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/collapsingmergetree/) 但使用不同的折叠算法，允许以多个线程的任何顺序插入数据。 特别是， `Version` 列有助于正确折叠行，即使它们以错误的顺序插入。 相比之下, `CollapsingMergeTree` 只允许严格连续插入。
    
- [GraphiteMergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/graphitemergetree/#graphitemergetree)
    
    > 该引擎用来对 [Graphite](https://graphite.readthedocs.io/en/latest/index.html)数据进行瘦身及汇总。对于想使用CH来存储Graphite数据的开发者来说可能有用。
    

关于MergeTree更多的细节我们可以参考文档： [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/)，下面就`ReplacingMergeTree`引擎进行举例说明。

##### `ReplacingMergeTree`引擎的使用

我们在MySQL建表的时候，如果希望某些字段唯一，通常都会通过唯一索引去实现。不过在ClickHouse里面，我们可以采用`ReplacingMergeTree`引擎。下面摘抄一段该引擎的简介：

> 该引擎和 [MergeTree](https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/mergetree/) 的不同之处在于它会删除排序键值相同的重复项。
> 
> 数据的去重只会在数据合并期间进行。合并会在后台一个不确定的时间进行，因此你无法预先作出计划。有一些数据可能仍未被处理。尽管你可以调用 `OPTIMIZE` 语句发起计划外的合并，但请不要依靠它，因为 `OPTIMIZE` 语句会引发对数据的大量读写。
> 
> 因此，`ReplacingMergeTree` 适用于在后台清除重复的数据以节省空间，但是它不保证没有重复的数据出现。

举例：我们希望`project`和`tag_name`数据没有重复，则可以按照如下建表：

```sql
CREATE TABLE `test_table` (
    `id` String COMMENT '唯一标识码',
    `project` String COMMENT '项目标识',
    `tag_name` String COMMENT '标签名称'
) ENGINE = ReplacingMergeTree() PARTITION BY project ORDER BY (project, tag_name) SETTINGS index_granularity = 8192;
```

这样子建表后，在一些数据更新的时候可以直接采用`INSERT`的方案处理，但是在查询的时候，务必记得使用`FINAL`修饰符，SQL如下：

```sql
SELECT * FROM `test_table` FINAL WHERE `project` = 'reader_free' LIMIT 20;
```

如果不使用该修饰符，由于数据的去重合并在后台不确定什么时候完成(实测可能很久)，那么查询到的可能会有重复数据，如果想第一时间处理完成合并，可以使用下述SQL：

```sql
OPTIMIZE TABLE `test_table` FINAL;
```

### 配置参数优化

##### `max_memory_usage` 系列参数

我们早期使用的ClickHouse都是自建的，所以在很多参数上使用的都是默认值。当我们进行大数据量或者复杂查询的时候，经常一不小心就把服务给查挂了。后来查询资料才知道，是`max_memory_usage` 系列参数没有优化导致的。  
我们去官网查询这些参数，对应的含义如下：

|name|value|description|
|---|---|---|
|max_memory_usage|12025908428|Maximum memory usage for processing of single query. Zero means unlimited.|
|max_memory_usage_for_user|12025908428|Maximum memory usage for processing all concurrently running queries for the user. Zero means unlimited.|
|max_memory_usage_for_all_queries|13743895347|Maximum memory usage for processing all concurrently running queries on the server. Zero means unlimited.|
|secondary_index_build_max_memory_usage|268435456|Maximum memory (256MB) usage for secondary index build.|

根据上述含义得知，默认值都是0，代表的是查询的时候对内存不做限制，这就导致大数据量查询的时候，很容易把内存打满而导致服务挂掉。所以这些参数，在自建ClickHouse的时候，务必记得优化。  
现如今我们的业务均已上云，使用的都是阿里云ClickHouse服务。阿里云的ClickHouse服务对这些参数都已经做了很好的优化。比如上述表格中的value列，便是其中一个实例下阿里云优化后的参数。

##### 常用关键参数

ClickHouse的配置参数有很多，不过如果使用云的话，大部分的参数已经被优化好了。下面列举几个参数，做个简单的了解：

|配置项名称|描述|
|---|---|
|background_pool_size|后台用来merge进程的大小，默认是16，建议改成cpu核数的2倍|
|log_queries|默认值为0，修改为1，系统会自动创建system_query_log表，并记录每次查询的query信息|
|max_concurrent_queries|最大并发处理的请求数(包含select,insert等)，默认值100，推荐150(不够再加)。|
|max_connections|最大连接数，根据实际情况配置，默认4096|
|max_threads|设置单个查询所能使用的最大cpu个数；默认是CPU核数|

更多的参数介绍，可以参考文档：[服务器配置](https://clickhouse.com/docs/zh/operations/server-configuration-parameters/settings/)。

### 经验总结

##### `ORDER BY`的使用

`ORDER BY`是在建表时候定义好的，名称叫做排序键，用于指定在一个数据片段内，数据以何种标准排序。在没有指定主键的情况下，主键默认是和排序键相同的。  
排序键可以为多个列字段，需要注意的是排序键的顺序会对查询效率产生影响，排序键的列是不允许进行更新操作的，同时也推荐参与排序的列基数不要太高。  
举例：在一张数据量6000W+的表里面，需要分批处理数据，原始的表结构和查询SQL语句如下：

```sql
-- 初始的时候排序键只有id
CREATE TABLE `test_table` (
    `id` UInt32,
    `uid` String,
    `oaid` String,
    `date` Date,
    `created_time` Nullable(DateTime),
    `updated_time` Nullable(DateTime)
) ENGINE = MergeTree() PARTITION BY date ORDER BY id SETTINGS index_granularity = 8192;

-- 查询的时候排序却使用了uid
SELECT `uid`, `oaid`, `created_time`
FROM `test_table`
ORDER BY `uid`, `id`
LIMIT 20000000, 10000;
```

上述查询，在`LIMIT`值不够大的时候差距不明显，但是随着`LIMIT`的值变大，查询速度会慢到无法忍受。  
优化后的方案如下：

```sql
-- 此处排序键已经把uid给加上了
CREATE TABLE `test_table` (
    `id` UInt32,
    `uid` String,
    `oaid` String,
    `date` Date,
    `created_time` Nullable(DateTime),
    `updated_time` Nullable(DateTime)
) ENGINE = MergeTree() PARTITION BY date ORDER BY (uid, id) SETTINGS index_granularity = 8192;
```

同样的数据和`LIMIT`条件下，没有排序键耗时50s+，但是加了排序键后只需要1s就够了。

##### 尝鲜`Array`特性

在我们的业务中，如果要求一个字段支持多个值，那么我们在设计表的时候，都会设计为字符串，然后采用逗号分隔。当需要使用的时候，可以借助`FIND_IN_SET`函数，完成对应的查询。  
不过在ClickHouse里面，我们可以借用`Array`的特性，实现上述的效果，具体SQL如下：

```sql
-- 创建表，注意tag_ids便是array字段
CREATE TABLE `test_array` (
    `date` String COMMENT '日期',
    `imei` String COMMENT 'imei值',
    `tag_ids` Array(Int64) COMMENT '标签id的数据'
) ENGINE = MergeTree() PARTITION BY date ORDER BY (date, imei) SETTINGS index_granularity = 8192;

-- 插入数据
INSERT INTO `test_array` VALUES
    ('2021-03-26', 'sddhdsdjgshjdg1', [1, 22, 33]),
    ('2021-03-26', '343rfsf', [11, 2, 33]),
    ('2021-03-26', 'cwesfef', [123, 22, 3]),
    ('2021-03-26', 'jyujytbsv', [1, 2, 13]);

-- 查询数据
SELECT * FROM `test_array` WHERE date = '2021-03-26' AND has(tag_ids, 3);
SELECT * FROM `test_array` WHERE date = '2021-03-26' AND has(tag_ids, 1234);
```

`Array`的功能特性有很多，上述案例只是最简单的使用，一些复杂的使用需要根据对应的场景来分析和实现。

##### 数据去重时的选择

在我们日常查询数据汇总去重的时候，常用的方案都是`COUNT(DISTINCT field)`，不过在ClickHouse里面，我们可以使用`uniqExact`函数来替代该方案。如果对数据精确到没有特别高的要求，我们还可以去使用`uniq`和`uniqCombined`函数，不过这2个函数大概有2%左右的误差。  
在一张有21亿数据的表下我们对这4个函数的进行对比，结果如下：

|函数|结果|耗时|误差绝对值|准确率|
|---|---|---|---|---|
|count(distinct)|2394942|14.450 sec|0|100.00%|
|uniqExact|2394942|13.729 sec|0|100.00%|
|uniqCombined|2396115|4.188 sec|1173|99.95%|
|uniq|2390192|3.414 sec|4750|99.80%|

根据上述结果我们可以明显发现`uniq`函数的优越性。所以我们在日常查询中，可以尽可能的去使用`uniq`函数，而且官方也建议在几乎所有的功能下均应该使用`uniq`函数。

##### 尽可能少的使用`JOIN`

ClickHouse的`JOIN`操作会对其查询效率产生很大的负面影响，所以我们应该尽量避免使用`JOIN`操作，如果需要的话，也尽量控制在3张表内的`JOIN`操作。  
我们可以使用一些技巧来规避`JOIN`操作，其一就是在表设计的时候，把表设计为大宽表，通过字段冗余来减少`JOIN`的处理。  
另外一个就是如果查询的数据从另外一张表取出时，可以使用`IN`来实现，示例如下：

```sql
-- IN的方案，执行耗时：0.027 sec
SELECT *
FROM `table_a`
WHERE logan_file_id IN (
	SELECT id
	FROM `table_b`
	WHERE package_channel = 'qm-oppo_lf'
)
ORDER BY create_time
LIMIT 10;

-- JOIN的方案，执行耗时：1.587 sec
SELECT a.*
FROM `table_a` a
	LEFT JOIN `table_b` b ON a.logan_file_id = b.id
WHERE b.package_channel = 'qm-oppo_lf'
ORDER BY a.create_time
LIMIT 10;
```

根据查询耗时结果可以看到，`IN`的方案查询效率明显优于`JOIN`操作。

##### `PREWHERE`代替`WHERE`

在我们日常使用最多的 MergeTree引擎中，可以使用`PREWHERE`来代替`WHERE`，尤其是查询列较多的时候，可以显著的提升查询性能。因为`PREWHERE`会自动优化执行过滤阶段的数据读取方式，降低IO操作。不过现在默认情况下ClickHouse会将`WHERE`中的部分转为`PREWHERE`。

```sql
-- 常规使用
SELECT * FROM `test_table` WHERE channel LIKE '%de%' AND dt BETWEEN '20210301' AND '20211101' LIMIT 10;
-- 替换WHERE
SELECT * FROM `test_table` PREWHERE channel LIKE '%de%' AND dt BETWEEN '20210301' AND '20211101' LIMIT 10;
```

##### 以分区删除并重建代替更新

ClickHouse在处理数据更新的时候，采用的是异步处理的方案，虽然马上返回了结果，但是其还需要时间去后台完成真正的更新操作，支持的不是很友好。所以在一些数据汇总的处理上，我们可以采用分区删除重建的方案来代替更新。  
比如在一个每小时汇总的结果表`table_b`里面，由于原始表`table_a`每时每刻都在产生数据，所以如果想看到当前的实时数据，那么就意味着`table_b`要不断进行更新操作。我们可以采用的方案是：在`table_b`以小时作为分区，在每分钟汇总的时候先删除该分区，然后重新生成并插入数据。  
该方案使用到的相关SQL如下：

```sql
-- 删除分区
ALTER TABLE `table_b` DROP PARTITION '2021-11-01 10:00:00';

-- 重新汇总并插入分区数据
INSERT INTO `table_b` (`t_datehour`, `action_id`, `pv`, `uv`)
SELECT toStartOfHour(`t_datetime`), `action_id`, SUM(`t_count`)
	, uniq(`uid`)
FROM `table_a`
WHERE `os_type` = 1
	AND toDate(`t_datetime`) = '2021-11-01'
	AND toHour(`t_datetime`) = 10
GROUP BY toStartOfHour(`t_datetime`), `action_id`;
```
