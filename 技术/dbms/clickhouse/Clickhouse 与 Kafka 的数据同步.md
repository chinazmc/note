
## 介绍

Clickhouse 本身为一个分析型数据库，提供很多跟其他组件的同步方案，本文将以 Kafka 作为数据来源介绍如何将 Kafka 的数据同步到 Clickhouse 中。

### 流程图

话不多说，先上一张数据同步的流程图

![同步流程图.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9f526d14e4441f59db73c67fb7405ae~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 建表

在数据同步之前，我们需要建对应的 clickhouse 表，根据上面的流程图，我们需要建立三个表：

1.数据表

2.kafka 引擎表

3.物化视图

#### 数据表

```sql
# 创建数据表
CREATE DATABASE IF NOT EXISTS data_sync;
CREATE TABLE IF NOT EXISTS data_sync.test
(
    name String DEFAULT 'lemonNan' COMMENT '姓名',
    age int DEFAULT 18 COMMENT '年龄',
    gongzhonghao String DEFAULT 'lemonCode' COMMENT '公众号',
    my_time DateTime64(3, 'UTC') COMMENT '时间'
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(my_time)
ORDER BY my_time

```

#### 引擎表
```sql
# 创建 kafka 引擎表, 地址: 172.16.16.4, topic: lemonCode
CREATE TABLE IF NOT EXISTS data_sync.test_queue(
    name String,
    age int,
    gongzhonghao String, 
    my_time DateTime64(3, 'UTC')
) ENGINE = Kafka
SETTINGS
  kafka_broker_list = '172.16.16.4:9092',
  kafka_topic_list = 'lemonCode',
  kafka_group_name = 'lemonNan',
  kafka_format = 'JSONEachRow',
  kafka_row_delimiter = '\n',
  kafka_schema = '',
  kafka_num_consumers = 1

```

#### 物化视图
```sql
# 创建物化视图
CREATE MATERIALIZED VIEW IF NOT EXISTS test_mv TO test AS SELECT name, age, gongzhonghao, my_time FROM test_queue;

```

### 数据模拟

下面是开始模拟流程图的数据走向，已安装 Kafka 的可以跳过安装步骤。

#### 安装 kafka

kafka 这里为了演示安装的是单机
```bash
# 启动 zookeeper
docker run -d --name zookeeper -p 2181:2181  wurstmeister/zookeeper
# 启动 kafka, KAFKA_ADVERTISED_LISTENERS 后的 ip地址为机器ip
docker run -d --name kafka -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 --link zookeeper -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.16.16.4:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka

```

#### 使用kafka命令发送数据

bash

代码解读

复制代码

`# 启动生产者，向 topic lemonCode 发送消息 kafka-console-producer.sh --bootstrap-server 172.16.16.4:9092 --topic lemonCode # 发送以下消息 {"name":"lemonNan","age":20,"gongzhonghao":"lemonCode","my_time":"2022-03-06 18:00:00.001"} {"name":"lemonNan","age":20,"gongzhonghao":"lemonCode","my_time":"2022-03-06 18:00:00.001"} {"name":"lemonNan","age":20,"gongzhonghao":"lemonCode","my_time":"2022-03-06 18:00:00.002"} {"name":"lemonNan","age":20,"gongzhonghao":"lemonCode","my_time":"2022-03-06 23;59:59.002"}`

#### 查看 Clickhouse 的数据表

csharp

代码解读

复制代码

`select * from test;`

![clickhouse消费结果图.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7696610ee66542828ecc8260640b3009~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 到这一步，数据已经从 Kafka 同步到了 Clickhouse 中了，怎么说呢，还是比较方便的。

### 关于数据副本

这里使用的表引擎是 `ReplicateMergeTree`, 用 ReplicateMergeTree 的一个原因是生成多个数据副本，减少数据丢失风险，使用 ReplicateMergeTree 引擎的话，数据会自动同步到相同分片的其他节点上。

在实际情况里，还有一种方式也可以进行数据的同步，通过使用不同的 `kafka consumer group` 进行数据消费。

具体见下图:

#### 副本方案1

通过 ReplicateMergeTree 的同步机制将数据同步到同分片下其他节点，同步时占用消费节点资源。

![副本方案1.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/624e8ac0251249189abb154d3ca58c97~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 副本方案2

通过 Kafka 本身的消费机制，将消息广播至多个 Clickhouse 节点，数据同步不占用 Clickhouse 额外资源。

![副本方案2.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf740961c89f4a788377549f6deeec75~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 注意的地方

搭建过程可能需要注意的地方

- 本文出现的 `172.16.16.4` 为机器内网ip
- 一般引擎表以 queue 为结尾，物化视图以 mv 为结尾，辨识度会高一点

### 总结

本文介绍了数据从 Kafka 同步至 Clickhouse以及多副本的方案，Clickhouse 还提供了很多其它的集成方案，包括 Hive、MongoDB、S3、SQLite、Kafka 等等等等，具体可以看下方链接。

> 集成的表引擎：[clickhouse.com/docs/zh/eng…](https://link.juejin.cn/?target=https%3A%2F%2Fclickhouse.com%2Fdocs%2Fzh%2Fengines%2Ftable-engines%2Fintegrations%2F "https://clickhouse.com/docs/zh/engines/table-engines/integrations/")

