起因是公司的主从数据库中，查询时主库不走索引，从库走索引，经过大佬排查发现应该是 Cardinality出现了错误。

1) 什么是Cardinality【卡丁 饿了提】

不是所有的查询条件出现的列都需要添加索引。对于什么时候添加B+树索引。一般的经验是，在访问表中很少一部分时使用B+树索引才有意义。对于性别字段、地区字段、类型字段，他们可取值范围很小，称为低选择性。如

SELECT * FROM student WHERE sex='M'

按性别进行查询时，可取值一般只有M、F。因此SQL语句得到的结果可能是该表50%的数据(加入男女比例1:1)这时添加B+树索引是完全没有必要的。相反，如果某个字段的取值范围很广，几乎没有重复，属于高选择性。则此时使用B+树的索引是最合适的。例如对于姓名字段，基本上在一个应用中不允许重名的出现

怎样查看索引是否有高选择性？通过SHOW INDEX结果中的列Cardinality来观察。非常关键，表示所以中不重复记录的预估值，需要注意的是Cardinality是一个预估值，而不是一个准确值基本上用户也不可能得到一个准确的值，**在实际应用中，Cardinality/n_row_in_table应尽可能的接近1，如果非常小,那用户需要考虑是否还有必要创建这个索引。故在访问高选择性属性的字段并从表中取出很少一部分数据时，对于字段添加B+树索引是非常有必要的。如**

SELECT * FROM member WHERE usernick='David';

表member大约有500W行数据,usernick字段上有一个唯一索引。这是如果查找用户名为David的用户，将得到如下执行计划

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633501680469-6b6fc80f-0ea5-4d97-98ec-788ee577795e.png)

可以看到使用了usernick这个索引。这也符合之前提到的高可选择性，即SQL语句取表中较少行的原则

2) InnoDB存储引擎的Cardinality统计

建立索引的前提是高选择性。这对数据库来说才具有实际意义，那么数据库是怎样统计Cardinality的信息呢?因为MySQL数据库中有各种不同的存储引擎，而每种存储引擎对于B+树索引的实现又各不相同。**所以对Cardinality统计时放在存储引擎层进行的**

在生成环境中，索引的更新操作可能非常频繁。如果每次索引在发生操作时就对其进行Cardinality统计，那么将会对数据库带来很大的负担。另外需要考虑的是，如果一张表的数据非常大，如一张表有50G的数据，那么统计一次Cardinality信息所需要的时间可能非常长。这样的环境下，是不能接受的。因此，数据库对于Cardinality信息的统计都是通过采样的方法完成

在InnoDB存储引擎中，Cardinality统计信息的更新发生在两个操作中：insert和update。InnoDB存储引擎内部对更新Cardinality信息的策略为:

**表中1/16的数据已发生了改变**

stat_modified_counter>2000 000 000

**第一种策略为自从上次统计Cardinality信息后，表中的1/16的数据已经发生过变化，这是需要更新Cardinality信息**

**第二种情况考虑的是，如果对表中某一行数据频繁地进行更新操作，这时表中的数据实际并没有增加，实际发生变化的还是这一行数据，则第一种更新策略就无法适用这种情况，故在InnoDB存储引擎内部有一个计数器start_modified_counter,用来表示发生变化的次数,当start_modified_counter>2 000 000 000 时，则同样更新Cardinality信息**

接着考虑InnoDB存储引擎内部是怎样进行Cardinality信息统计和更新操作呢？同样是通过采样的方法。**默认的InnoDB存储引擎对8个叶子节点Leaf Page进行采用。**采用过程如下

取得B+树索引中叶子节点的数量，记为A

随机取得B+树索引中的8个叶子节点，统计每个页不同记录的个数，即为P1，P2....P8

通过采样信息给出Cardinality的预估值:**Cardinality=(P1+P2+...+P8)\*A/8**

**根据上述的说明可以发现，在InnoDB存储引擎中，Cardinality值通过对8个叶子节点预估而得的。而不是一个实际精确的值。再者，每次对Cardinality值的统计，都是通过随机取8个叶子节点得到的，这同时有暗示了另外一个Cardinality现象，即每次得到的Cardinality值可能不同的，如**

**SHOW INDEX FROM OrderDetails**

上述SQL语句会触发MySQL数据库对于Cardinality值的统计，第一次运行得到的结果如图5-20

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633501680482-ac10f602-75fa-4765-b037-592fcb829eb5.png)

在上述测试过程中，并没有通过INSERT、UPDATE这类的操作来改变OrderDetails中的内容，但是当第二次运行SHOW INDEX FROM OrderDetails语句是，发生了变化，如图5-21

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633501680516-b9cc9c2f-e54f-4c67-bd8c-5c2c7e760d74.png)

可以看到，当第二次运行SHOW INDEX FROM OrderDetails语句时，表OrderDetails索引中的Cardinality值发生了变化，虽然表OrderDetails本身并没有发生任何变化，但是由于Cardinality是随机取8个叶子节点进行分析，所以即使表没有发生变化，用户观察到索引Cardinality值还是会发生变化，这本身不是Bug,而是随机采样而导致的结果

**当然，有一种情况可以使得用户每次观察到的索引Cardinality值是一样的。那就是表足够小，表的叶子节点树小于或者等于8个。这时即使随机采样，也总是会采取倒这些页，因此每次得到的Cardinality值是相同的**

**在InnoDB1.2版本之前，可以通过innodb_stats_sample_pages用来设置统计Cardinality时每次采样页的数量，默认为8.同时，参数innodb_stats_method用来判断如何对待索引中出现NULL值记录。该参数默认值为nulls_equal,表示将NULL值记录为相等的记录。其有效值还nulls_unequal,nulls_ignored,分别表示将NULL值记录视为不同的记录和忽略NULL值记录。**例如某夜中索引记录为NULL、NULL、1、2、2、3、3、3,在参数innodb_stats_method默认设置下，该页的Cardinality为4；若参数innodb_stats_method为nulls_unequal,则该页的Cardinality为5，若参数innodb_stats_method为nulls_ignored，则Cardinality值为3

**当执行ANALYZE TABLE、SHOW TABLE STATUS、SHOW INDEX 以及访问INFORMATION_SCHEMA架构下的表TABLES和STATISTICS时会导致InnoDB存储引擎会重新计算索引Cardinality值，若表中的数据量非常大，并且表中存在多个辅助索引时，执行上述操作可能会非常慢，虽然用户可能并不希望去更新Cardinality值**

InnoDB1.2版本提供了更多参数对Cardinality进行设置。如表

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633501680597-5467e0ef-dda9-4390-a7e5-e13baa47f094.png)

Analyze Table

**MySQL 的Optimizer（优化元件）在优化SQL语句时，首先需要收集一些相关信息，其中就包括表的cardinality（可以翻译为“散列程度”），它表示某个索引对应的列包含多少个不同的值——如果cardinality大大少于数据的实际散列程度，那么索引就基本失效了。**

我们可以使用SHOW INDEX语句来查看索引的散列程度：

SHOW INDEX FROM PLAYERS;

TABLE KEY_NAME COLUMN_NAME CARDINALITY

------- -------- ----------- -----------

PLAYERS PRIMARY PLAYERNO 14

因为此时PLAYER表中不同的PLAYERNO数量远远多于14，索引基本失效。

下面我们通过Analyze Table语句来修复索引：

ANALYZE TABLE PLAYERS;

SHOW INDEX FROM PLAYERS;

结果是：

TABLE KEY_NAME COLUMN_NAME CARDINALITY

------- -------- ----------- -----------

PLAYERS PRIMARY PLAYERNO 1000

此时索引已经修复，查询效率大大提高。

**需要注意的是，如果开启了binlog，那么Analyze Table的结果也会写入binlog，我们可以在analyze和table之间添加关键字local取消写入。**

Checksum Table

数据在传输时，可能会发生变化，也有可能因为其它原因损坏，为了保证数据的一致，我们可以计算checksum（校验值）。

使用MyISAM引擎的表会把checksum存储起来，称为live checksum，当数据发生变化时，checksum会相应变化。

在执行Checksum Table时，可以在最后指定选项qiuck或是extended；quick表示返回存储的checksum值，而extended会重新计算checksum，如果没有指定选项，则默认使用extended。

Optimize Table

经常更新数据的磁盘需要整理碎片，数据库也是这样，Optimize Table语句对MyISAM和InnoDB类型的表都有效。

如果表经常更新，就应当定期运行Optimize Table语句，保证效率。

与Analyze Table一样，Optimize Table也可以使用local来取消写入binlog。

Check Table

数据库经常可能遇到错误，譬如数据写入磁盘时发生错误，或是索引没有同步更新，或是数据库未关闭MySQL就停止了。

遇到这些情况，数据就可能发生错误：

Incorrect key file for table: ' '. Try to repair it.

此时，我们可以使用Check Table语句来检查表及其对应的索引。

譬如我们运行

CHECK TABLE PLAYERS;

结果是

TABLE OP MSG_TYPE MSG_TEXT

-------------- ----- -------- --------

TENNIS.PLAYERS check status OK

MySQL会保存表最近一次检查的时间，每次运行check table都会存储这些信息：

执行

SELECT TABLE_NAME, CHECK_TIME

FROM INFORMATION_SCHEMA.TABLES

WHERE TABLE_NAME = 'PLAYERS'

AND TABLE_SCHEMA = 'TENNIS'; /*TENNIS是数据库名*/

结果是

TABLE_NAME CHECK_TIME

---------- -------------------

PLAYERS 2006-08-21 16:44:25

Check Table还可以指定其它选项：

UPGRADE：用来测试在更早版本的MySQL中建立的表是否与当前版本兼容。

QUICK：速度最快的选项，在检查各列的数据时，不会检查链接（link）的正确与否，如果没有遇到什么问题，可以使用这个选项。

FAST：只检查表是否正常关闭，如果在系统掉电之后没有遇到严重问题，可以使用这个选项。

CHANGED：只检查上次检查时间之后更新的数据。

MEDIUM：默认的选项，会检查索引文件和数据文件之间的链接正确性。

EXTENDED：最慢的选项，会进行全面的检查。

Repair Table

用于修复表，只对MyISAM和ARCHIVE类型的表有效。

这条语句同样可以指定选项：

QUICK：最快的选项，只修复索引树。

EXTENDED：最慢的选项，需要逐行重建索引。

USE_FRM：只有当MYI文件丢失时才使用这个选项，全面重建整个索引。

与Analyze Table一样，Repair Table也可以使用local来取消写入binlog。