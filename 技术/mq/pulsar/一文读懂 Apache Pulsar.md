
## Pulsar 介绍

Apache Pulsar 作为 Apache 软件基金会顶级项目，是下一代云原生分布式消息流平台，集消息、存储、轻量化函数计算为一体，采用计算与存储分离架构设计，支持多租户、持久化存储、跨区域复制、具有强一致性、高吞吐、低延迟及高可扩展性等流数据存储特性。

Pulsar 诞生于 2012 年，最初的目的是为在 Yahoo 内部，整合其他消息系统，构建统一逻辑、支撑大集群和跨区域的消息平台。当时的其他消息系统（包括 Kafka），都不能满足 Yahoo 的需求，比如大集群多租户、稳定可靠的 IO 服务质量、百万级 Topic、跨地域复制等，因此 Pulsar 应运而生。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7o5bo58oj31vw0u0aci.jpg](https://segmentfault.com/img/remote/1460000041096452 "https://tva1.sinaimg.cn/large/008i3skNly1gx7o5bo58oj31vw0u0aci.jpg")

Pulsar 的关键特性如下：

-   Pulsar 的单个实例原生支持多个集群，可跨机房在集群间无缝地完成消息复制。
-   极低的发布延迟和端到端的延迟。
-   可无缝扩展到超过一百万个 Topic。
-   简单的客户端 API，支持 Java、Go、Python 和 C++.
-   支持多种 Topic 订阅模式 （独占订阅、共享订阅、故障转移订阅）。
-   通过 Apache BookKeeper 提供的持久化消息存储机制保证消息传递。
-   由轻量级的 Serverless 计算框架 Pulsar Functions 实现流原生的数据处理。
-   基于 Pulsar Functions 的 serverless connector 框架 Pulsar IO 使得数据更容易移入、移出 Apache Pulsar。
-   分层存储可在数据陈旧时，将数据从热存储卸载到冷/长期存储（如S3、GCS）中。

![https://tva1.sinaimg.cn/large/008i3skNly1gx6tsnsw8qj31bk0u0gx8.jpg](https://segmentfault.com/img/remote/1460000041096453 "https://tva1.sinaimg.cn/large/008i3skNly1gx6tsnsw8qj31bk0u0gx8.jpg")

**社区：**

目前 Apache Pulsar 在 Github 的 star 数量是10K+，共有470+个 contributor。并且正在持续更新，社区的活跃度比较好。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7n3h8huij317n0u0agi.jpg](https://segmentfault.com/img/remote/1460000041096454 "https://tva1.sinaimg.cn/large/008i3skNly1gx7n3h8huij317n0u0agi.jpg")

## 概念

### Producer

消息的源头，也是消息的发布者，负责将消息发送到 topic。

### Consumer

消息的消费者，负责从 topic 订阅并消费消息。

### Topic

消息数据的载体，在 Pulsar 中 Topic 可以指定分为多个 partition，如果不设置默认只有一个 partition。

### Broker

Broker 是一个无状态组件，主要负责接收 Producer 发送过来的消息，并交付给 Consumer。

### BookKeeper

分布式的预写日志系统，为消息系统比 Pulsar 提供存储服务，为多个数据中心提供跨机器复制。

### Bookie

Bookie 是为消息提供持久化的 Apache BookKeeper 的服务端。

### Cluster

Apache Pulsar 实例集群，由一个或多个实例组成。

## 云原生架构

![https://tva1.sinaimg.cn/large/008i3skNly1gx6vxwcoqwj31ar0u0djm.jpg](https://segmentfault.com/img/remote/1460000041096455 "https://tva1.sinaimg.cn/large/008i3skNly1gx6vxwcoqwj31ar0u0djm.jpg")

Apache Pulsar 采用计算存储分离的一个架构，不与计算逻辑耦合在一起，可以做到数据独立扩展和快速恢复。计算存储分离式的架构随着云原生的发展，在各个系统中出现的频次也是越来越多。Pulsar 的 Broker 层就是一层无状态的计算逻辑层，主要负责接收和分发消息，而存储层由 Bookie 节点组成，负责消息的存储和读取。

Pulsar 的这种计算存储分离式的架构，可以做到水平扩容不受限制，如果系统的 Producer、Consumer 比较多，那么就可以直接扩容计算逻辑层 Broker，不受数据一致性的影响。如果不是这种架构，我们在扩容的时候，计算逻辑和存储都是实时变化的，就很容易受到数据一致性的限制。同时计算层的逻辑本身就很复杂容易出错，而存储层的逻辑相对简单，出错的概率也比较小，在这种架构下，如果计算层出现错误，可以进行单方面恢复而不影响存储层。

Pulsar 还支持数据分层存储，可以将旧消息移到便宜的存储方案中，而最新的消息可以存到 SSD 中。这样可以节约成本、最大化利用资源。

## 集群架构

![https://tva1.sinaimg.cn/large/008i3skNly1gx4b6s164ej30wt0lptbv.jpg](https://segmentfault.com/img/remote/1460000041096456 "https://tva1.sinaimg.cn/large/008i3skNly1gx4b6s164ej30wt0lptbv.jpg")

Pulsar的集群由多个 Pulsar 实例组成的，其中包括

-   多个 Broker 实例，负责接收和分发消息
-   一个 ZooKeeper 服务，用来协调集群配置
-   BookKeeper 服务端集群 Bookie，用来做消息的持久化
-   集群之间通过跨地域复制进行消息同步

## 设计原理

pulsar 采用发布-订阅的设计模式（pub-sub）,在该设计模式中 producer 发布消息到 topic ，consumer 订阅 topic 中的消息并在处理完成之后发送 ack 确认。

![https://tva1.sinaimg.cn/large/008i3skNly1gx4b4elwu8j30jm032wem.jpg](https://segmentfault.com/img/remote/1460000041096457 "https://tva1.sinaimg.cn/large/008i3skNly1gx4b4elwu8j30jm032wem.jpg")

### Producer

#### 发送模式

Producer 发送消息有两种模式，可以同步（sync）或异步（async）的方式发布消息到 broker。

-   同步发送消息是 Producer 发送消息以后要等到 broker 的确认以后才会认为消息发送成功，如果没有收到确认就认为消息发送失败。
    
    MessageId messageId = producer.send("同步发送的消息".getBytes(StandardCharsets.UTF_8));
    
-   异步发送消息是 Producer 发送消息，将消息放到阻塞队列中并立即返回。不需要等待 broker 的确认。
    
    CompletableFuture<MessageId> messageIdCompletableFuture = producer.sendAsync(
                    "异步发送的消息".getBytes(StandardCharsets.UTF_8));
    

#### 访问方式

Pulsar 为 Producer 提供多种不同类型的 Topic 访问模式：

-   Shared
    
    默认情况下多个生产者可以发布消息到同一个 Topic 。
    
-   Exclusive
    
    要求生产者以独占模式访问 Topic，在此模式下 如果 Topic 已经有了生产者，那么其他生产者在连接就会失败报错。
    
    "**Topic has an existing exclusive producer: standalone-0-12**"
    
-   WaitForExclusive
    
    如果主题已经连接了生产者，则将当前生产者挂起，直到生产者获得了 Exclusive 访问权限。
    

可以通过以下方式来设置访问模式：

Producer<byte[]> producer = pulsarClient.newProducer().accessMode(ProducerAccessMode.Shared).topic("test-topic-1").create();

#### 压缩

Pulsar 支持对 Producer 发送的消息进行压缩，Pulsar 支持以下类型的压缩：

-   LZ4
    
    LZ4 是无损压缩算法，提供每核 > 500 MB/s 的压缩速度，可通过多核 CPU 进行扩展。它具有极快的解码器，每个内核的速度达数 GB/s，通常达到多核系统上的 RAM 速度限制。
    
-   ZLIB
    
    zlib旨在成为一个免费的、通用的、不受法律约束的——也就是说，不受任何专利保护——无损数据压缩库，几乎可以在任何计算机硬件和操作系统上使用。zlib 数据格式本身可以跨平台移植。
    
-   ZSTD
    
    Zstandard 是一种快速压缩算法，提供高压缩率。它还为小数据提供了一种特殊模式，称为字典压缩。参考库提供了非常广泛的速度/压缩权衡，并由极快的解码器提供支持。
    
-   snappy
    
    Snappy是一个压缩/解压库。它不以最大压缩为目标，也不与任何其他压缩库兼容;相反，它的目标是非常高的速度和合理的压缩。
    

#### 批处理

Producer 支持在单次请求发送批量消息，可以通过( acknowledgmentAtBatchIndexLevelEnabled = true ) 来开启批处理。当一个批次的所有消息都被 Consumer 消费确认后，这个批次的消息才会被确认发送成功。如果出现了意外故障，可能会导致这个批次的所有消息重新投递，包括已经被确认消费过得消息。

为了避免这个问题，Pulsar 2.6.0 以后开始引入了批量索引确认，由 broker 来维护每个索引的确认状态，避免将已确认的消息发送给 Consumer。当这个批次的所有消息索引都被确认后，将会删除批消息。

#### 消息分块

Producer 支持将消息分块，可以使用 chunkingEnabled=true 启动分块，启用分块时，要注意一下几点：

-   不能同时启动批处理和分块，要启动分块，必须提前禁用批处理。
-   仅持久化主题支持分块。
-   仅对独占和故障转移订阅类型支持分块。

当分块被启用的时候，如果 Producer 发送的消息超过最大负载大小，则 Producer 会将原始消息拆分成多个块消息，然后将每个块发送给 broker，分块消息在broker内的存储和正常的消息是一样的。只是在 Consumer 消费的时候，发现这是一个分块消息，就需要在缓存分块消息，当收集完一个消息的所有分块组合成原始放到接收者队列里面给客户端消费。如果 Producer 未能发送所有的分块消息， Consumer 有过期处理机制，默认的过期时间为一个小时。

**一个生产者和一个有序的消费者的分块消息模型：**

![https://tva1.sinaimg.cn/large/008i3skNly1gx4d5zvt7bj30jh051q2y.jpg](https://segmentfault.com/img/remote/1460000041096458 "https://tva1.sinaimg.cn/large/008i3skNly1gx4d5zvt7bj30jh051q2y.jpg")

**多个生产者和一个有序消费者的分块消息模型：**

![https://tva1.sinaimg.cn/large/008i3skNly1gx4dch0rb1j30j708dq3b.jpg](https://segmentfault.com/img/remote/1460000041096459 "https://tva1.sinaimg.cn/large/008i3skNly1gx4dch0rb1j30j708dq3b.jpg")

### Consumer

Consumer 是消息的消费者，通过订阅指定的 Topic 从 broker中获取消息。

Consumer 向 broker 发送流请求以获取消息。在 Consumer 端有一个队列来接收从 broker 推送的消息。您可以使用receiverQueueSize参数配置队列大小。默认大小为1000)。每次consumer.receive()调用时，都会从缓冲区中取出一条消息。

#### 接收方式

消息可以同步（sync）或异步（async）的方式从 broker 接收，还可以通过MessageListener返回消息：接收消息后回调用户的MessageListener。

-   同步接收消息将被阻塞，直到有消息可用。
    
    Message<byte[]> message = consumer.receive();
    System.out.println("接收消息内容: " + new String(message.getData()));
    consumer.acknowledge(message);  // 确认消费消息
    
-   异步接收消息将会立即返回一个未来值。使用CompletableFuture。如果CompletableFuture完成消息接收，应该随后调用receiveAsync()，否则，它会在应用程序中创建接收请求的积压。
    
    可以通过调用.cancel(false) ( CompletableFuture.cancel(boolean) ）在完成之前取消返回的`future`，以将其从接收请求的积压中删除。
    
    CompletableFuture<Message<byte[]>> messageCompletableFuture = consumer.receiveAsync();
    Message<byte[]> message = messageCompletableFuture.get();
    System.out.println("接收消息内容: " + new String(message.getData()));
    consumer.acknowledge(message); // 确认消费消息
    
-   客户端库为消费者提供侦听器实现。例如，Java 客户端提供了一个MesssageListener 接口，每当收到新消息时都会调用received方法。
    
    pulsarClient.newConsumer().topic("test-topic-1").messageListener((MessageListener<byte[]>) (consumer, msg) -> {
                System.out.println("接受消息内容: " + new String(msg.getData()));
                try {
                    consumer.acknowledge(msg);  //确认消费消息
                } catch (PulsarClientException e) {
                    consumer.negativeAcknowledge(msg);  // 消息消费失败
                }
            }).subscriptionName("test-subscription-1").subscribe();
    

#### 消费确认

**成功确认**：

当 Consumer 成功消费一条消息以后，需要向 broker 发送消息确认。只有在所有的订阅都确认后 ，消息才会被删除。如果需要存储已经确认消费成功的消息，需要设置消息保存策略。否则 Pulsar 会立即删除所有确认消费成功的消息。

对于批处理的消息，可以通过以下两种方式确认消息：

-   消息被单独确认。通过单独确认，消费者需要确认每条消息并向代理发送确认请求。
-   消息被累积确认。通过累积确认，消费者只需要确认它收到的最后一条消息。流中直到提供的消息的所有消息都不会重新传递给该消费者。

**失败确认：**

当 Consumer 消费失败一条消息，想要再次消费该消息时，需要向 broker 发送否定确认。说明消息没有消费成功，这样 broker 会重新投递这个消息。消息会被单独或累积地否定确认，具体取决于消费订阅类型：

-   在独占和故障转移订阅类型中，消费者仅否定他们收到的最后一条消息。
-   在 shared 和 Key_Shared 订阅类型中，您可以单独否定确认消息。

**确认超时：**

如果一条消息没有消费成功，想要触发 broker 自动重发消息，可以采用未确认消息自动重发机制。建议优先使用失败确认，可以更加精准的控制单个消息的重新投递。

#### 死信队列

Apache Pulsar 内置死信队列特性，当消息处理失败收到否认 Ack 时，Apache Pulsar 可以自动重试。超过重试次数可以将消息存放至死信队列，以确保新消息可以得到处理。

Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
              .topic(topic)
              .subscriptionName("my-subscription")
              .subscriptionType(SubscriptionType.Shared)
              .deadLetterPolicy(DeadLetterPolicy.builder()
                    .maxRedeliverCount(maxRedeliveryCount)
                    .build())
              .subscribe();

#### 消费模型

Apache Pulsar 提供了队列模型和流模型的统一，在 Topic 级别只需要保存一份数据，同一份数据可多次消费，以流、队列等方式计算不同的订阅模型，大大提升了灵活度。Apache Pulsar 中有四种订阅类型：exclusive、shared、failover和key_shared。这些类型如下图所示。

![https://tva1.sinaimg.cn/large/008i3skNly1gx5905r8klj314w0u0wha.jpg](https://segmentfault.com/img/remote/1460000041096460 "https://tva1.sinaimg.cn/large/008i3skNly1gx5905r8klj314w0u0wha.jpg")

### Topic

#### Topic 命名

在 Pulsar 中 Topic 负责将消息从生产者传递到消费者，主题名称是具有明确定义结构的 URL：

{persistent|non-persistent}://tenant/namespace/topic

-   persistent / non-persistent
    
    表示主题的类型，主题分为持久化和非持久化主题，默认是持久化的类型。持久化的主题会将消息保存到磁盘上，而非持久化的主题就不会将消息保存到磁盘。
    
-   tenant
    
    Pulsar 实例中主题的租户，租户对于 Pulsar 中的多租户至关重要，并且分布在集群中。
    
-   namespace
    
    将相关联的 Topic 作为一个组来管理，是管理 Topic 的基本单元。每个租户可以有一个或多个命名空间。
    
-   topic
    
    Pulsar 中的主题被命名为通道，用于将消息从生产者传输到消费者。
    

> 在 Pulsar 中不需要显示的创建 Topic，如果尝试向不存在的 Topic 发送或接收消息，会在默认租户和命名空间中创建 Topic。

#### Topic 分区

普通的 Topic 被保存在单个 broker 中，而 Topic 可以分为多个 partition，分别存储到不同的 broker 中，由多个 broker 来处理，这大大的提升了 Topic 的吞吐量。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7m7xcu7oj30rz133420.jpg](https://segmentfault.com/img/remote/1460000041096461 "https://tva1.sinaimg.cn/large/008i3skNly1gx7m7xcu7oj30rz133420.jpg")

如上图，Topic1 被分为了5个分区，分别是 P0、P1、P2、P3、P4。这5个分区被分到了3个 Broker（Broker1、Broker2、Broker3） 中，由于分区的数量多于代理的数量，前两个代理分别处理两个分区，第三个代理只处理一个，Pulsar 会自动处理这种分区分布。

发布到分区主题时，必须指定_路由模式_。路由模式决定了每条消息应该发布到哪个分区。

共有三种[MessageRoutingMode](https://link.segmentfault.com/?enc=%2BaeVDTcc3x379JGgO30mJw%3D%3D.gX%2Fb2WPGJsn%2FK%2B956TyRIR0YGXkA6X6wPbP%2BTy7itfJGsNi1G8a7Y1G%2BHD2rnv17d64agHKfG%2FB%2Bj297AuK2XPLoUCUsQ42o%2FTx5FFWufmMnAU27V9RDU6fStnBZIh0CkHulx9S%2Fj%2FagPdOvdIyrRw%3D%3D) 可用：

-   RoundRobinPartition
    
    如果没有提供密钥，生产者将以循环方式跨所有分区发布消息，以实现最大吞吐量。 请注意，循环不是针对单个消息进行的，而是设置为与批处理延迟相同的边界，以确保批处理有效。而如果在消息上指定了一个键，分区的生产者将散列键并将消息分配给特定的分区。
    
-   SinglePartition
    
    如果没有提供密钥，分区生产者将随机选择一个分区并将所有消息发布到该分区中。 如果在消息上提供了密钥，则分区生产者将散列密钥并将消息分配给特定分区。
    
-   CustomPartition
    
    使用将被调用的自定义消息路由器实现来确定特定消息的分区。
    

### 多租户

Apache Pulsar 的多租户特性可以满足企业的管理需求，租户、命名空间是 Apache Pulsar 支持多租户的 2 个核心概念。

-   在租户级别，Pulsar 为特定的租户预留合适的存储空间、应用授权与认证机制。
-   在命名空间级别，Pulsar 提供一些列的配置策略，包括存储配额、流控、消息过期策略和命名空间之间的隔离策略。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7m1ttzyuj30dk06y74k.jpg](https://segmentfault.com/img/remote/1460000041096462 "https://tva1.sinaimg.cn/large/008i3skNly1gx7m1ttzyuj30dk06y74k.jpg")

### 跨地域复制

跨地域复制机制为大型分布式系统多数据中心提供了冗余，防止服务无法正常运行。也为跨地域生产、跨地域消费提供了基础。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7mqbmrprj30jp09p74u.jpg](https://segmentfault.com/img/remote/1460000041096463 "https://tva1.sinaimg.cn/large/008i3skNly1gx7mqbmrprj30jp09p74u.jpg")

### 分层存储

Pulsar 的**分层存储**功能允许将较旧的积压数据从 BookKeeper 移动到长期且更便宜的存储，降低存储成本，同时仍然允许客户端访问积压，就好像没有任何变化一样。管理员可配置命名空间大小阈值策略，实现数据自动迁移到长期存储。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7n6pk1lxj30uu0jdab0.jpg](https://segmentfault.com/img/remote/1460000041096464 "https://tva1.sinaimg.cn/large/008i3skNly1gx7n6pk1lxj30uu0jdab0.jpg")

## 组件

### Pulsar Schema Registry

Schema registry 使 Producer 和 Consumer 通过 broker 即可沟通 Topic 的数据结构，而不需要借助外部协调机制，从而避免各种潜在的问题如序列化、反序列化问题。

### Pulsar Functions

Pulsar Functions 是一个轻量化计算框架，它给用户提供一个部署简单、运维简单、API 简单的 FAAS（Function as a service）平台，目标是帮助用户轻松创建各种级别的复杂处理逻辑，而无需部署单独的计算系统。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7nka8twdj31bk0t50um.jpg](https://segmentfault.com/img/remote/1460000041096465 "https://tva1.sinaimg.cn/large/008i3skNly1gx7nka8twdj31bk0t50um.jpg")

### Pulsar IO

Pulsar IO 支持 Apache Pulsar 与外部系统如数据库、其他消息系统进行交互，如 Apache Cassandra 等系统，用户无需关注实现细节，用一个命令就可以快速运行。

Source 将数据从外部系统导入 Apache Pulsar，Sink 将数据从 Apache Pulsar 导出到外部系统。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7nsujib5j31cu09s3z1.jpg](https://segmentfault.com/img/remote/1460000041096466 "https://tva1.sinaimg.cn/large/008i3skNly1gx7nsujib5j31cu09s3z1.jpg")

### Pulsar SQL

Pulsar SQL 是构建在 Apache Pulsar 之上的查询层，支持用户动态查询存储在 Pulsar 内部的所有旧数据流。用户可以在同一系统上注入数据的同时进行数据流的清理、转化和查询，极大简化了数据管道。

![https://tva1.sinaimg.cn/large/008i3skNly1gx7o3112oaj31270u0q5u.jpg](https://segmentfault.com/img/remote/1460000041096467 "https://tva1.sinaimg.cn/large/008i3skNly1gx7o3112oaj31270u0q5u.jpg")

## 快速上手

### 二进制安装

这里就简单介绍一个 Pulsar Demo 的案例，只安装一个 Pulsar 的服务端，首先通过以下命令下载 Pulsar：

wget https://archive.apache.org/dist/pulsar/pulsar-2.8.1/apache-pulsar-2.8.1-bin.tar.gz

下载到本地以后，通过以下命令解压 apache-pulsar-2.8.1-bin.tar.gz 压缩包：

tar xvfz apache-pulsar-2.8.1-bin.tar.gz

然后 cd 到 apache-pulsar-2.8.1 文件夹下，包含以下目录 ：

-   bin ：Pulsar 的命令行工具，例如`[pulsar](https://pulsar.apache.org/docs/en/reference-cli-tools#pulsar)`和`[pulsar-admin](https://pulsar.apache.org/tools/pulsar-admin/)`。
-   conf : Pulsar 的配置文件，包括[broker 配置](https://link.segmentfault.com/?enc=KQ1aKBCvOmq3jcpHC%2FFvdA%3D%3D.p%2Fqg61XnEF9l72TjIg3KF0HeuDt9Iq3xfbFBVjINcqHcB%2FonoboUx6hiw%2FijnwTfqyuGCgRPc4FCvBylXC73zNBvcd47tAaRzxSuRAyVLdk%3D)、[ZooKeeper 配置](https://link.segmentfault.com/?enc=EaT2Gzv9tn%2Bf4yvSFS4S0Q%3D%3D.24T7DPPhUzYqqnkCZS46VL8g02i2btA74GRrHHoXaSMo95fdVvkVAwJyjUgmz56zqjxwz7bLdmKMW2UMHY8FIQ3OoQMB3WUZjZYGx%2FbWgZk%3D)等。
-   examples : 包含[Pulsar 函数](https://link.segmentfault.com/?enc=X7sYfMXuWDaRB%2BZERAJsrw%3D%3D.rGpVGMc8pWGzvupAURNalGulDwCIsw6jCRh2BknOsleWzDpYv3UCbfDEz9ckpHZYzf7ZYzvJDpRPvrcG9ofshQ%3D%3D)示例的Java JAR 文件。
-   lib : Pulsar 使用的[JAR](https://link.segmentfault.com/?enc=iS20E0h%2FMUyVZYwRiIAtRA%3D%3D.cpssYiR894iE8p8EA5eDfun4D61ZT9wDGIEXvqQF0eHjDX5%2FaRoqxxvhvlp0hu6J))文件。
-   licenses ：`.txt`以 Pulsar[代码库的](https://link.segmentfault.com/?enc=0Srb1Gz5jXF72atTE%2Fz2Fw%3D%3D.8K6gYLgh3isUNIVwrdz44nLs6UvNQgHMH%2BCC7oLg8J%2BO%2BrUrp%2BYoq7v9DMdgrVwQ)各种组件的形式存在的许可证文件。

### **独立启动 Pulsar**

当您在本地安装好 Pulsar 以后，您就可以使用`[pulsar](https://pulsar.apache.org/docs/en/reference-cli-tools#pulsar)`存储在`bin`目录中的命令启动本地集群，并指定您要以独立模式启动 Pulsar。

$ bin/pulsar standalone

如果你已经成功启动了 Pulsar，你会看到这样的`INFO`level 日志消息：

2017-06-01 14:46:29,192 - INFO  - [main:WebSocketService@95] - Configuration Store cache started
2017-06-01 14:46:29,192 - INFO  - [main:AuthenticationService@61] - Authentication is disabled
2017-06-01 14:46:29,192 - INFO  - [main:WebSocketService@108] - Pulsar WebSocket Service started

### 一个发送/接收消息案例

public class PulsarDemo {

    private static PulsarClient PULSAR_CLIENT = null;

    static {
        try {
            // 创建pulsar客户端
            PULSAR_CLIENT = PulsarClient.builder().serviceUrl("pulsar://127.0.0.1:6650").build();
        } catch (PulsarClientException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws PulsarClientException {

        // 创建生产者
        Producer<byte[]> producer = PULSAR_CLIENT.newProducer().topic("test-topic-1").create();
        // 同步发送消息
        MessageId messageId = producer.send("同步发送的消息".getBytes(StandardCharsets.UTF_8));
        System.out.println("消息发送成功，消息id: " + messageId);

        // 创建消费者
        Consumer<byte[]> consumer = PULSAR_CLIENT.newConsumer().topic("test-topic-1")
                .subscriptionName("test-subscription-1").subscribe();
        //获取一个消息内容
        Message<byte[]> message = consumer.receive();
        System.out.println("接收的消息内容: " + new String(message.getData()));
        // 确认消费成功，以便pulsar删除消费成功的消息
        consumer.acknowledge(message);

        //关闭客户端
        producer.close();
        consumer.close();
        PULSAR_CLIENT.close();
    }
}

输出：

消息发送成功，消息id: 66655:0:-1:0
接收的消息内容: 同步发送的消息

# Reference
https://segmentfault.com/a/1190000041096450