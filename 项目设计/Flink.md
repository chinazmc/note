### 为什么选择 Flink CDC

Flink CDC 基于数据库日志的 Change Data Capture 技术，实现了全量和增量的一体化读取能力，并借助 Flink 优秀的管道能力和丰富的上下游生态，支持捕获多种数据库的变更，并将这些变更实时同步到下游存储。

目前，**Flink CDC 的上游**已经支持了 MySQL、MariaDB、PG、Oracle、MongoDB 、Oceanbase、TiDB、SQLServer 等数据库。

**Flink CDC 的下游**则更加丰富，支持写入 Kafka、Pulsar 消息队列，也支持写入 Hudi、Iceberg 、Doris 等，支持写入各种数据仓库及数据湖中。

同时，通过 Flink SQL 原生支持的 Changelog 机制，可以让 CDC 数据的加工变得非常简单。用户通过 SQL 便能实现数据库全量和增量数据的清洗、打宽、聚合等操作，极大地降低了用户门槛。此外， Flink DataStream API 支持用户编写代码实现自定义逻辑，给用户提供了深度定制业务的自由度。

**Flink CDC 技术的核心是支持将表中的全量数据和增量数据做实时一致性的同步与加工，让用户可以方便地获每张表的实时一致性快照。**比如一张表中有历史的全量业务数据，也有增量的业务数据在源源不断写入、更新。Flink CDC 会实时抓取增量的更新记录，实时提供与数据库中一致性的快照，如果是更新记录，会更新已有数据；如果是插入记录，则会追加到已有数据，整个过程中，Flink CDC 提供了一致性保障，即不重不丢。

**Flink CDC 具有如下优势：**

- Flink 的算子和 SQL 模块更为成熟和易用；
- Flink 作业可以通过调整算子并行度的方式轻松扩展处理能力；
- Flink 支持高级的状态后端（State Backends），允许存取海量的状态数据；
- Flink 提供更多的 Source 和 Sink 等生态支持；
- Flink 有更大的用户基数和活跃的支持社群，问题更容易解决。

而且 Flink Table / SQL 模块将数据库表和变动记录流（例如 CDC 的数据流）看做是同一事物的两面，因此内部提供的 Upsert 消息结构（`+I`表示新增、`-U`表示记录更新前的值、`+U`表示记录更新后的值，`-D`表示删除）可以与 Debezium 等生成的变动记录一一对应。