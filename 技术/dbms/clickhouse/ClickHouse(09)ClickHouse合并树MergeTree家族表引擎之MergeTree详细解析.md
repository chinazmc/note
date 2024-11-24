                

> Clickhouse中最强大的表引擎当属MergeTree（合并树）引擎及该系列（MergeTree）中的其他引擎。MergeTree系列的引擎被设计用于插入极大量的数据到一张表当中。数据可以以数据片段的形式一个接着一个的快速写入，数据片段在后台按照一定的规则进行合并。相比在插入时不断修改（重写）已存储的数据，这种策略会高效很多。

**主要特点**

- 存储的数据按主键排序。这使得您能够创建一个小型的稀疏索引来加快数据检索。
- 如果指定了分区键的话，可以使用分区。在相同数据集和相同结果集的情况下ClickHouse中某些带分区的操作会比普通操作更快。查询中指定了分区键时ClickHouse会自动截取分区数据。这也有效增加了查询性能。
- 支持数据副本。ReplicatedMergeTree系列的表提供了数据副本功能。
- 支持数据采样。需要的话，您可以给表设置一个采样方法。

# 建表

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
[SETTINGS name=value, ...]
```

对于建表语句中这个部分的解释.

- **ENGINE**:引擎名和参数。ENGINE=MergeTree().MergeTree引擎没有参数。
- **ORDERBY**:排序键。可以是一组列的元组或任意的表达式。例如:ORDER BY (CounterID,EventDate)。如果没有使用PRIMARY KEY显式指定的主键，ClickHouse会使用排序键作为主键。如果不需要排序，可以使用 ORDER BY tuple().
- **PARTITION BY**:分区键，可选项。大多数情况下，不需要分使用区键。即使需要使用，也不需要使用比月更细粒度的分区键。分区不会加快查询（这与ORDER BY表达式不同）。永远也别使用过细粒度的分区键。不要使用客户端指定分区标识符或分区字段名称来对数据进行分区（而是将分区字段标识或名称作为ORDER BY表达式的第一列来指定分区）。要按月分区，可以使用表达式toYYYYMM(date_column)，这里的date_column是一个 Date类型的列。分区名的格式会是"YYYYMM"。
- **PRIMARY KEY**:如果要选择与排序键不同的主键，在这里指定，可选项。默认情况下主键跟排序键（由ORDER BY 子句指定）相同。因此，大部分情况下不需要再专门指定一个PRIMARY KEY子句。
- **SAMPLE BY**:用于抽样的表达式，可选项。如果要用抽样表达式，主键中必须包含这个表达式。例如：SAMPLE BY intHash32(UserID) ORDER BY (CounterID, EventDate, intHash32(UserID))。
- **TTL**:指定行存储的持续时间并定义数据片段在硬盘和卷上的移动逻辑的规则列表，可选项。表达式中必须存在至少一个 Date或DateTime类型的列，比如：TTL date + INTERVAl 1 DAY。规则的类型 DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'指定了当满足条件（到达指定时间）时所要执行的动作：移除过期的行，还是将数据片段（如果数据片段中的所有行都满足表达式的话）移动到指定的磁盘（TO DISK 'xxx') 或 卷（TO VOLUME 'xxx'）。默认的规则是移除（DELETE）。可以在列表中指定多个规则，但最多只能有一个DELETE的规则。
- **SETTINGS**:控制 MergeTree 行为的额外参数，可选项：
- **index_granularity**:索引粒度。索引中相邻的『标记』间的数据行数。默认值8192 。
- **index_granularity_bytes**:索引粒度，以字节为单位，默认值:10Mb。如果想要仅按数据行数限制索引粒度, 可以设置为0,但是不建议。
- **min_index_granularity_bytes**:允许的最小数据粒度，默认值：1024b。该选项用于防止误操作，添加了一个非常低索引粒度的表。参考数据存储
- **enable_mixed_granularity_parts**:是否启用通过index_granularity_bytes控制索引粒度的大小。在19.11版本之前, 只有index_granularity配置能够用于限制索引粒度的大小。当从具有很大的行（几十上百兆字节）的表中查询数据时候，index_granularity_bytes配置能够提升ClickHouse的性能。如果您的表里有很大的行，可以开启这项配置来提升SELECT 查询的性能。
- **use_minimalistic_part_header_in_zookeeper**:ZooKeeper中数据片段存储方式。如果use_minimalistic_part_header_in_zookeeper=1，ZooKeeper会存储更少的数据。
- **min_merge_bytes_to_use_direct_io**:使用直接I/O来操作磁盘的合并操作时要求的最小数据量。合并数据片段时，ClickHouse会计算要被合并的所有数据的总存储空间。如果大小超过了min_merge_bytes_to_use_direct_io设置的字节数，则ClickHouse将使用直接I/O接口对磁盘读写。如果设置min_merge_bytes_to_use_direct_io = 0，则会禁用直接 I/O。默认值：10 _1024_ 1024 * 1024 字节。
- **merge_with_ttl_timeout**:TTL合并频率的最小间隔时间，单位：秒。默认值:86400(1 天)。
- **write_final_mark**:是否启用在数据片段尾部写入最终索引标记。默认值:1（不要关闭）。
- **merge_max_block_size**:在块中进行合并操作时的最大行数限制。默认值：8192
- **storage_policy**:存储策略。 参见 使用具有多个块的设备进行数据存储.
- **min_bytes_for_wide_part,min_rows_for_wide_part**:在数据片段中可以使用Wide格式进行存储的最小字节数/行数。您可以不设置、只设置一个，或全都设置。
- **max_parts_in_total**:所有分区中最大块的数量
- **max_compress_block_size**:在数据压缩写入表前，未压缩数据块的最大大小。可以在全局设置中设置该值。建表时指定该值会覆盖全局设置。
- **min_compress_block_size**:在数据压缩写入表前，未压缩数据块的最小大小。可以在全局设置中设置该值。建表时指定该值会覆盖全局设置。
- **max_partitions_to_read**:一次查询中可访问的分区最大数。您可以在全局设置中设置该值。

# 数据存储

表由按主键排序的数据片段（DATAPART）组成。

当数据被插入到表中时，会创建多个数据片段并按主键的字典序排序。例如，主键是(CounterID,Date)时，片段中数据首先按CounterID排序，具有相同CounterID的部分按Date排序。

不同分区的数据会被分成不同的片段，ClickHouse在后台合并数据片段以便更高效存储。不同分区的数据片段不会进行合并。合并机制并不保证具有相同主键的行全都合并到同一个数据片段中。

数据片段可以以Wide或Compact格式存储。在Wide格式下，每一列都会在文件系统中存储为单独的文件，在Compact格式下所有列都存储在一个文件中。Compact格式可以提高插入量少插入频率频繁时的性能。

数据存储格式由min_bytes_for_wide_part和min_rows_for_wide_part表引擎参数控制。如果数据片段中的字节数或行数少于相应的设置值，数据片段会以Compact格式存储，否则会以Wide格式存储。

每个数据片段被逻辑的分割成颗粒（granules）。颗粒是ClickHouse中进行数据查询时的最小不可分割数据集。ClickHouse不会对行或值进行拆分，所以每个颗粒总是包含整数个行。每个颗粒的第一行通过该行的主键值进行标记，ClickHouse会为每个数据片段创建一个索引文件来存储这些标记。对于每列，无论它是否包含在主键当中，ClickHouse都会存储类似标记。这些标记让您可以在列文件中直接找到数据。

颗粒的大小通过表引擎参数index_granularity和index_granularity_bytes控制。颗粒的行数的在[1,index_granularity]范围中，这取决于行的大小。如果单行的大小超过了index_granularity_bytes设置的值，那么一个颗粒的大小会超过index_granularity_bytes。在这种情况下，颗粒的大小等于该行的大小。

# 主键和索引在查询中的表现

我们以 (CounterID, Date) 以主键。排序好的索引的图示会是下面这样：

```sql
    全部数据  :     [-------------------------------------------------------------------------]
    CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
    Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
    标记:            |      |      |      |      |      |      |      |      |      |      |
                    a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
    标记号:          0      1      2      3      4      5      6      7      8      9      10
```

如果指定查询如下：

CounterID in ('a','h')，服务器会读取标记号在 [0, 3) 和 [6, 8) 区间中的数据。  
CounterID IN ('a','h') AND Date = 3，服务器会读取标记号在 [1, 3) 和 [7, 8) 区间中的数据。  
Date = 3，服务器会读取标记号在[1, 10]区间中的数据。  
上面例子可以看出使用索引通常会比全表描述要高效。

稀疏索引会引起额外的数据读取。当读取主键单个区间范围的数据时，每个数据块中最多会多读 index_granularity * 2 行额外的数据。

稀疏索引使得您可以处理极大量的行，因为大多数情况下，这些索引常驻于内存。

ClickHouse 不要求主键唯一，所以可以插入多条具有相同主键的行。

可以在PRIMARY KEY与ORDER BY条件中使用可为空的类型的表达式，但强烈建议不要这么做。为了启用这项功能，需要打开allow_nullable_key，NULLS_LAST规则也适用于ORDER BY条件中有NULL值的情况下。

## 主键的选择

主键中列的数量并没有明确的限制。依据数据结构，可以在主键包含多些或少些列。一般主键的选择可以按照下面的规则：

- 改善索引的性能。
- 如果当前主键是 (a, b) ，在下列情况下添加另一个 c 列会提升性能：
- 查询会使用 c 列作为条件
- 很长的数据范围（index_granularity的数倍）里(a, b)都是相同的值，并且这样的情况很普遍。换言之，就是加入另一列后，可以让您的查询略过很长的数据范围。
- 改善数据压缩。ClickHouse以主键排序片段数据，所以，数据的一致性越高，压缩越好。
- 在CollapsingMergeTree和SummingMergeTree引擎里进行数据合并时会提供额外的处理逻辑。在这种情况下，指定与主键不同的 排序键也是有意义的。

长的主键会对插入性能和内存消耗有负面影响，但主键中额外的列并不影响SELECT查询的性能。

可以使 ORDER BY tuple()语法创建没有主键的表。在这种情况下ClickHouse根据数据插入的顺序存储。如果在使用INSERT...SELECT时希望保持数据的排序，需要设置 max_insert_threads = 1。

## 选择与排序键不同的主键

Clickhouse可以做到指定一个跟排序键不一样的主键，此时排序键用于在数据片段中进行排序，主键用于在索引文件中进行标记的写入。这种情况下，主键表达式元组必须是排序键表达式元组的前缀(即主键为(a,b)，排序列必须为(a,b,**))。

当使用SummingMergeTree和AggregatingMergeTree引擎时，这个特性非常有用。通常在使用这类引擎时，表里的列分两种：维度和度量。典型的查询会通过任意的GROUP BY对度量列进行聚合并通过维度列进行过滤。由于SummingMergeTree和AggregatingMergeTree会对排序键相同的行进行聚合，所以把所有的维度放进排序键是很自然的做法。但这将导致排序键中包含大量的列，并且排序键会伴随着新添加的维度不断的更新。

在这种情况下合理的做法是，只保留少量的列在主键当中用于提升扫描效率，将维度列添加到排序键中。

对排序键进行ALTER是轻量级的操作，因为当一个新列同时被加入到表里和排序键里时，已存在的数据片段并不需要修改。由于旧的排序键是新排序键的前缀，并且新添加的列中没有数据，因此在表修改时的数据对于新旧的排序键来说都是有序的。

## 索引和分区在查询中的应用

对于SELECT查询，ClickHouse分析是否可以使用索引。如果WHERE/PREWHERE子句具有下面这些表达式（作为完整WHERE条件的一部分或全部）则可以使用索引：进行相等/不相等的比较；对主键列或分区列进行IN运算、有固定前缀的LIKE运算(如name like 'test%')、函数运算(部分函数适用)，还有对上述表达式进行逻辑运算。

因此，在索引键的一个或多个区间上快速地执行查询是可能的。下面例子中，指定标签；指定标签和日期范围；指定标签和日期；指定多个标签和日期范围等执行查询，都会非常快。

```sql
--表引擎配置
--ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate) SETTINGS index_granularity=8192;

--
SELECT count() FROM table WHERE EventDate = toDate(now()) AND CounterID = 34
SELECT count() FROM table WHERE EventDate = toDate(now()) AND (CounterID = 34 OR CounterID = 42)
SELECT count() FROM table WHERE ((EventDate >= toDate('2014-01-01') AND EventDate <= toDate('2014-01-31')) OR EventDate = toDate('2014-05-01')) AND CounterID IN (101500, 731962, 160656) AND (CounterID = 101500 OR EventDate != toDate('2014-05-01'))
```

ClickHouse 会依据主键索引剪掉不符合的数据，依据按月分区的分区键剪掉那些不包含符合数据的分区。

上面的查询显示，即使索引用于复杂表达式，因为读表操作经过优化，所以使用索引不会比完整扫描慢。

但是下面这个就不会走索引。

```sql
SELECT count() FROM table WHERE CounterID = 34 OR URL LIKE '%upyachka%'
```

要检查 ClickHouse 执行一个查询时能否使用索引，可设置 force_index_by_date 和 force_primary_key 。

使用按月分区的分区列允许只读取包含适当日期区间的数据块，这种情况下，数据块会包含很多天（最多整月）的数据。在块中，数据按主键排序，主键第一列可能不包含日期。因此，仅使用日期而没有用主键字段作为条件的查询将会导致需要读取超过这个指定日期以外的数据。

## 部分单调主键的使用

考虑这样的场景，比如一个月中的天数。它们在一个月的范围内形成一个单调序列 ，但如果扩展到更大的时间范围它们就不再单调了。这就是一个部分单调序列。如果用户使用部分单调的主键创建表，ClickHouse同样会创建一个稀疏索引。当用户从这类表中查询数据时，ClickHouse 会对查询条件进行分析。如果用户希望获取两个索引标记之间的数据并且这两个标记在一个月以内，ClickHouse 可以在这种特殊情况下使用到索引，因为它可以计算出查询参数与索引标记之间的距离。

如果查询参数范围内的主键不是单调序列，那么 ClickHouse 无法使用索引。在这种情况下，ClickHouse 会进行全表扫描。

ClickHouse 在任何主键代表一个部分单调序列的情况下都会使用这个逻辑。

## 跳数索引

此索引在 CREATE 语句的列部分里定义。

```sql
INDEX index_name expr TYPE type(...) GRANULARITY granularity_value
```

MergeTree系列的表可以指定跳数索引。跳数索引是指数据片段按照粒度(建表时指定的index_granularity)分割成小块后，将上述SQL的granularity_value数量的小块组合成一个大的块，对这些大块写入索引信息，这样有助于使用where筛选时跳过大量不必要的数据，减少SELECT需要读取的数据量。具体可以看下面这个例子。

```sql
CREATE TABLE table_name
(
    u64 UInt64,
    i32 Int32,
    s String,
    ...
    INDEX a (u64 * i32, s) TYPE minmax GRANULARITY 3,
    INDEX b (u64 * length(s)) TYPE set(1000) GRANULARITY 4
) ENGINE = MergeTree()
...
```

上例中的索引能让 ClickHouse 执行下面这些查询时减少读取数据量。

```sql
SELECT count() FROM table WHERE s < 'z'
SELECT count() FROM table WHERE u64 * i32 == 10 AND u64 * length(s) >= 1234
```

### 可用的索引类型

- minmax:存储指定表达式的极值（如果表达式是 tuple ，则存储 tuple 中每个元素的极值），这些信息用于跳过数据块，类似主键。
    
- set(max_rows):存储指定表达式的不重复值（不超过 max_rows 个，max_rows=0 则表示『无限制』）。这些信息可用于检查数据块是否满足 WHERE 条件。
    
- ngrambf_v1(n, size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed):存储一个包含数据块中所有 n元短语（ngram） 的 布隆过滤器 。只可用在字符串上。 可用于优化 equals ， like 和 in 表达式的性能。
    
    1. n – 短语长度。
    2. size_of_bloom_filter_in_bytes – 布隆过滤器大小，字节为单位。（因为压缩得好，可以指定比较大的值，如 256 或 512）。
    3. number_of_hash_functions – 布隆过滤器中使用的哈希函数的个数。
    4. random_seed – 哈希函数的随机种子。
- tokenbf_v1(size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed):跟ngrambf_v1类似，但是存储的是token而不是ngrams。Token是由非字母数字的符号分割的序列。
    
- bloom_filter(bloom_filter([false_positive]):为指定的列存储布隆过滤器
    
    可选参数false_positive用来指定从布隆过滤器收到错误响应的几率。取值范围是 (0,1)，默认值：0.025
    
    支持的数据类型：Int_, UInt_, Float*, Enum, Date, DateTime, String, FixedString, Array, LowCardinality, Nullable。
    

## 并发数据访问

对于表的并发访问，我们使用多版本机制。换言之，当一张表同时被读和更新时，数据从当前查询到的一组片段中读取。没有冗长的的锁。插入不会阻碍读取。

对表的读操作是自动并行的。

## 列和表的 TTL

TTL用于设置值的生命周期，它既可以为整张表设置，也可以为每个列字段单独设置。表级别的TTL还会指定数据在磁盘和卷上自动转移的逻辑。

TTL表达式的计算结果必须是日期或日期时间类型的字段。

例如:

```sql
-- TTL time_column
-- TTL time_column + interval

TTL date_time + INTERVAL 1 MONTH
TTL date_time + INTERVAL 15 HOUR
```

### 列TTL

创建列TTL:

```sql
CREATE TABLE example_table
(
    d DateTime,
    a Int TTL d + INTERVAL 1 MONTH,
    b Int TTL d + INTERVAL 1 MONTH,
    c String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d;
```

在已有的列创建ttl

```sql
ALTER TABLE example_table
    MODIFY COLUMN
    c String TTL d + INTERVAL 1 DAY;
```

修改列字段的 TTL

```sql
ALTER TABLE example_table
    MODIFY COLUMN
    c String TTL d + INTERVAL 1 MONTH;
```

### 表TTL

表可以设置一个用于移除过期行的表达式，以及多个用于在磁盘或卷上自动转移数据片段的表达式。当表中的行过期时，ClickHouse 会删除所有对应的行。对于数据片段的转移特性，必须所有的行都满足转移条件。

```sql
TTL expr
    [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'][, DELETE|TO DISK 'aaa'|TO VOLUME 'bbb'] ...
    [WHERE conditions]
    [GROUP BY key_expr [SET v1 = aggr_func(v1) [, v2 = aggr_func(v2) ...]] ]
```

TTL 规则的类型紧跟在每个 TTL 表达式后面，它会影响满足表达式时（到达指定时间时）应当执行的操作：

- DELETE:删除过期的行（默认操作）;
- TO DISK 'aaa':将数据片段移动到磁盘 aaa;
- TO VOLUME 'bbb':将数据片段移动到卷 bbb.
- GROUP BY:聚合过期的行

使用WHERE从句，您可以指定哪些过期的行会被删除或聚合(不适用于移动)。GROUP BY表达式必须是表主键的前缀。如果某列不是GROUP BY表达式的一部分，也没有在SET从句显示引用，结果行中相应列的值是随机的(就好像使用了any函数)。

例子:

```sql
-- 创建时指定 TTL
CREATE TABLE example_table
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH [DELETE],
    d + INTERVAL 1 WEEK TO VOLUME 'aaa',
    d + INTERVAL 2 WEEK TO DISK 'bbb';

--修改表的 TTL
ALTER TABLE example_table
    MODIFY TTL d + INTERVAL 1 DAY;

-- 创建一张表，设置一个月后数据过期，这些过期的行中日期为星期一的删除

CREATE TABLE table_with_where
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH DELETE WHERE toDayOfWeek(d) = 1;

-- 创建一张表，设置过期的列会被聚合。列x包含每组行中的最大值，y为最小值，d为可能任意值。

CREATE TABLE table_for_aggregation
(
    d DateTime,
    k1 Int,
    k2 Int,
    x Int,
    y Int
)
ENGINE = MergeTree
ORDER BY (k1, k2)
TTL d + INTERVAL 1 MONTH GROUP BY k1, k2 SET x = max(x), y = min(y);
```

### 删除数据

ClickHouse 在数据片段合并时会删除掉过期的数据。

当ClickHouse发现数据过期时,它将会执行一个计划外的合并。要控制这类合并的频率,您可以设置merge_with_ttl_timeout。如果该值被设置的太低,它将引发大量计划外的合并，这可能会消耗大量资源。

如果在两次合并的时间间隔中执行SELECT查询,则可能会得到过期的数据。为了避免这种情况，可以在SELECT之前使用OPTIMIZE。

# 使用多个块设备进行数据存储

MergeTree 系列表引擎可以将数据存储在多个块设备上。这对某些可以潜在被划分为“冷”“热”的表来说是很有用的。最新数据被定期的查询但只需要很小的空间。相反，详尽的历史数据很少被用到。如果有多块磁盘可用，那么“热”的数据可以放置在快速的磁盘上（比如NVMe固态硬盘或内存），“冷”的数据可以放在相对较慢的磁盘上（比如机械硬盘）。

数据片段是MergeTree引擎表的最小可移动单元。属于同一个数据片段的数据被存储在同一块磁盘上。数据片段会在后台自动的在磁盘间移动，也可以通过ALTER查询来移动。

### 配置

磁盘、卷和存储策略应当在主配置文件 config.xml 或 config.d 目录中的独立文件中的 \<storage_configuration> 标签内定义。

磁盘、卷配置如下，disk_name_N> — 磁盘名，名称必须与其他磁盘不同。path — 服务器将用来存储数据 (data 和 shadow 目录) 的路径, 应当以 ‘/’ 结尾。keep_free_space_bytes — 需要保留的剩余磁盘空间。

```xml
<storage_configuration>
    <disks>
        <disk_name_1> <!-- disk name -->
            <path>/mnt/fast_ssd/clickhouse/</path>
        </disk_name_1>
        <disk_name_2>
            <path>/mnt/hdd1/clickhouse/</path>
            <keep_free_space_bytes>10485760</keep_free_space_bytes>
        </disk_name_2>
        <disk_name_3>
            <path>/mnt/hdd2/clickhouse/</path>
            <keep_free_space_bytes>10485760</keep_free_space_bytes>
        </disk_name_3>

        ...
    </disks>

    ...
</storage_configuration>
```

存储策略配置如下，policy_name_N — 策略名称，不能重复。volume_name_N — 卷名称，不能重复。  
disk — 卷中的磁盘。max_data_part_size_bytes — 卷中的磁盘可以存储的数据片段的最大大小。move_factor — 当可用空间少于这个因子时，数据将自动的向下一个卷（如果有的话）移动 (默认值为 0.1)。prefer_not_to_merge - 禁止在这个卷中进行数据合并。该选项启用时，对该卷的数据不能进行合并。这个选项主要用于慢速磁盘。

```xml
<storage_configuration>
    ...
    <policies>
        <policy_name_1>
            <volumes>
                <volume_name_1>
                    <disk>disk_name_from_disks_configuration</disk>
                    <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
                </volume_name_1>
                <volume_name_2>
                    <!-- configuration -->
                </volume_name_2>
                <!-- more volumes -->
            </volumes>
            <move_factor>0.2</move_factor>
        </policy_name_1>
        <policy_name_2>
            <!-- configuration -->
        </policy_name_2>

        <!-- more policies -->
    </policies>
    ...
</storage_configuration>
```

# 虚拟列

这个介绍mergeTree表的一些虚拟列。

- _part - 分区名称。
- _part_index - 作为请求的结果，按顺序排列的分区数。
- _partition_id — 分区名称。
- _part_uuid - 唯一部分标识符（如果 MergeTree 设置assign_part_uuids 已启用）。
- _partition_value — partition by 表达式的值（元组）。
- _sample_factor - 采样因子（来自请求）。

# Reference
https://zhangfeidezhu.com/?p=246