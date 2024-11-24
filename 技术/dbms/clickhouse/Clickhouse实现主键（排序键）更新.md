
## 背景

随着业务的发展，报表类业务数据量越来越大、属性也越来越多，数据来源多样化，然后出现了如下几个问题：

1. 数据放在mysql中查询越来越慢。
2. 添加字段变得越来越困难。
3. 各个数据源产出时间不一致，等数据源都准备好再生成报表，则报表生成时间延后；每个数据源单独更新，则会产生大量更新操作，效率很慢。

为了解决以上问题，我们开始探索新的方案，首先想到的就是clickhouse（之前的业务中已有大量使用），作为一款优秀的OLTB数据库大数据量从来都不是问题；采用列式存储，加字段也没有问题；唯一需要解决的就是如何更新数据。

## 核心思想

我们找到了两种方案可以实现数据的更新效果

1. AggregatingMergeTree+AggregateFunction+物化视图+Nullable
2. AggregatingMergeTree+SimpleAggregateFunction+Nullable

虽然标题是update，但其本质还是insert，insert后借助引擎特性达到update效果。两种方式各有长短，目前我们采用的是第二种，下面给大家简单介绍下。

## AggregateFunction方案

### 优点

1. 数据实时更新，无延迟

### 缺点

1. 主键以外数据以二进制格式存储，需要借助Merge函数进行查询
2. 不能对主键以外字段直接进行聚合操作
3. 逻辑复杂
4. 查询sql复杂

### 适用场景

1. 数据更新时间不固定，且频繁
2. 只根据主键进行查询

### 实现方案

借助AggregateFunction、物化视图、Nullable、argMaxIf、isNotNull实现

```
CREATE TABLE channel_data_agg (
 `data_date` String,
 `channel_name` String,
 `active_num` AggregateFunction(argMaxIf, Int32, DateTime64(3), UInt8),
 `retention_day1` AggregateFunction(argMaxIf, Int32, DateTime64(3), UInt8),
 `retention_day7` AggregateFunction(argMaxIf, Int32, DateTime64(3), UInt8),
 `version` AggregateFunction(max, DateTime64(3))
) ENGINE = AggregatingMergeTree() PARTITION BY data_date ORDER BY channel_name SETTINGS index_granularity = 8192;
```

我们的active_num、retention_day1、retention_day7类型为AggregateFunction，写入数据只能以insert select的方式，所以我们需要建立一个数据源表（channel_data_origin）用来接收数据，并且需要创建一个物化视图（channel_data_mv）将聚合结果写入channel_data_agg.

channel_data_origin支持Nullable且物化视图根据isNotNull过滤数据，所以我们只需要写入需要更新的字段即可。

```
CREATE TABLE channel_data_origin (
 `data_date` String,
 `channel_name` String,
 `active_num` Nullable(Int32),
 `retention_day1` Nullable(Int32),
 `retention_day7` Nullable(Int32),
 `ver` DateTime64(3)
) ENGINE = MergeTree() PARTITION BY data_date ORDER BY channel_name SETTINGS index_granularity = 8192;

CREATE MATERIALIZED VIEW IF NOT EXISTS channel_data_mv TO channel_data_agg
AS
SELECT data_date,
       channel_name,
       argMaxIfState(active_num, ver, isNotNull(active_num))  AS active_num,
       argMaxIfState(retention_day1, ver, isNotNull(retention_day1)) AS retention_day1,
       argMaxIfState(retention_day7, ver, isNotNull(retention_day7))  AS retention_day7,
       maxState(ver) as version
FROM channel_data_origin
GROUP BY data_date, channel_name;
```

上面就是所有的建表语句了，现在我们将数据写入channel_data_origin，让channel_data_mv帮我们将数据同步到channel_data_agg就可以了。

我们写入数据，并且查询结果

```SQL
insert into channel_data_origin (data_date, channel_name, active_num, ver) 
values('2022-01-01', 'qimao',1, '2022-01-01 00:00:03'), ('2022-01-01', 'zongheng',2, '2022-01-01 00:00:03');
insert into channel_data_origin (data_date, channel_name, active_num, ver) 
values('2022-01-01', 'qimao',2, '2022-01-01 00:00:04'), ('2022-01-01', 'zongheng',3, '2022-01-01 00:00:04');
insert into channel_data_origin (data_date, channel_name, retention_day1, ver) 
values('2022-01-01', 'qimao',2, '2022-01-01 00:00:04'), ('2022-01-01', 'zongheng',3, '2022-01-01 00:00:04');

select * from channel_data_agg
```

![](https://tech.qimao.com/content/images/2022/09/image-12.png)

如图我们看到的数据指标都是乱码，这是因为AggregateFunction存储的二进制的中间状态，所以查询的时候需要用相对应的aggregate functions 并且添加后缀Merge（argMaxIfMerge）并且配合group进行查询。

```
select 
data_date, 
channel_name, 
argMaxIfMerge(active_num) as active_num, 
argMaxIfMerge(retention_day1) as retention_day1, argMaxIfMerge(retention_day7) as retention_day7,  
maxMerge(version) as version 
from channel_data_agg group by data_date, channel_name;
```

![](https://tech.qimao.com/content/images/2022/09/image-5.png)

AggregateFunction属于聚合函数，不可在嵌套其他聚合函数，例如：argMaxIfMerge(sum(active_num))

## SimpleAggregateFunction方案

### 优点

1. 数据明文保存
2. 逻辑简单
3. 查询方式与普通查询一致，可随意进行聚合操作

### 缺点

1. 需手动optimize，比较耗资源

### 适用场景

1. 数据更新频次低
2. 数据更新时间固定
3. 需要对各数据指标进行汇总

### 实现方案

借助SimpleAggregateFunction、argLast、Nullable实现。

```
-- SimpleAggregateFunction(anyLast, Nullable(Int32))
-- 保留最后一条不为空的数据
CREATE TABLE channel_data_agg_s (
 `data_date` String,
 `channel_name` String,
 `active_num` SimpleAggregateFunction(anyLast, Nullable(Int32)),
 `retention_day1` SimpleAggregateFunction(anyLast, Nullable(Int32)),
 `retention_day7` SimpleAggregateFunction(anyLast, Nullable(Int32)),
 `version` SimpleAggregateFunction(max, DateTime64(3))
) ENGINE = AggregatingMergeTree() PARTITION BY data_date ORDER BY channel_name SETTINGS index_granularity = 8192;

```

写入数据，并且查询结果

```
insert into channel_data_agg_s (data_date, channel_name, active_num, version) 
values('2022-01-01', 'qimao',1, '2022-01-01 00:00:03'), ('2022-01-01', 'zongheng',2, '2022-01-01 00:00:03');
insert into channel_data_agg_s (data_date, channel_name, active_num, version) 
values('2022-01-01', 'qimao',2, '2022-01-01 00:00:04'), ('2022-01-01', 'zongheng',3, '2022-01-01 00:00:04');
insert into channel_data_agg_s (data_date, channel_name, retention_day1, version) 
values('2022-01-01', 'qimao',2, '2022-01-01 00:00:04'), ('2022-01-01', 'zongheng',3, '2022-01-01 00:00:04');

select * from channel_data_agg_s
```

![](https://tech.qimao.com/content/images/2022/09/image-8.png)

可以看到数据并没有被聚合，怎么拿到最终结果呢？

方案一：select语句添加final主动触发数据合并（不建议使用，会耗费大量资源，不可控。）

```
select * from channel_data_agg_s final
```

![](https://tech.qimao.com/content/images/2022/09/image-5.png)

方案二：写入数据后执行optimize触发数据合并（如果写入数据未optimize时，用户在查询数据，会导致数据重复，可以查询条件带入最后optimize时间解决此问题。）

```
-- optimize一定要加分区，用于提升效率
optimize table channel_data_agg_s partition '2022-01-01' final
```
