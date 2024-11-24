

越来越多的消息平台开始采用实时流技术，这大大促进了 Pulsar 的发展。2020 年，Pulsar 受到持续关注，多家媒体争相报道；从《财富》百强企业到有前瞻性的初创团队，凡是开发消息平台和事件流应用程序的公司都在持续关注 Pulsar，这些关注激励着 Pulsar 项目和社区的迅猛发展。

  

同时，Pulsar 的应用场景也越来越广泛。近期的新闻和文章都在推送 Pulsar 的用户案例和功能特性，帮助大家进一步了解 Pulsar。

  

-   Verizon Media  
    https://www.youtube.com/watch?v=FXQvsHz_S1A
    
-   Iterable  
    https://www.youtube.com/watch?v=NrDvSNewNT0
    
-   Nutanix  
    https://www.youtube.com/watch?v=zAHxgG_U67Q
    
-   Overstock.com  
    https://www.youtube.com/watch?v=pmaCG1SHAW8
    

  

但并非所有的媒体报道都真实准确。前段时间 Confluent 发表了《 Kafka、Pulsar 和 RabbitMQ 对比》的技术文章，Pulsar 社区的小伙伴纷纷来信，让我们对此文做出回应。Pulsar 成为一项革新技术，社区发展迅猛，我们感谢社区小伙伴们的大力支持，并借此机会和大家深入聊聊 Pulsar。  

  

本系列文章分为上下篇，上篇从性能、架构和功能方面比较 Pulsar 和 Kafka 的区别。下篇主要介绍 Pulsar 的用例、支持与社区等。

> Kafka 的文档比较全面，普及范围广泛；我们会加大力度，普及 Pulsar 知识，让更多人了解 Pulsar。

  

  

  

**概 况**

  

  

  

**>****>****>** **组件**

Pulsar 有 3 个重要组件：**broker、Apache BookKeeper 和 Apache ZooKeeper**。Broker 是无状态服务，客户端需要连接到 broker 进行核心消息传递。而 BookKeeper 和 ZooKeeper 是有状态服务。

  

BookKeeper 节点（bookie）存储消息和游标，ZooKeeper 则只用于为 broker 和 bookie 存储元数据。另外，BookKeeper 使用 RocksDB 作为内嵌数据库，用于存储内部索引，但不能独立于 BookKeeper 单独管理 RocksDB。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/0EIbbCE3P2ZgNlKbfQ83tFzJSVG1CgfZHNQ7hSZHPN10X6NGfaic39hWPwKWZQ0UfS2llzrF7XkMEf3r4tGJ8og/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  

Kafka 采用单片架构模型，将服务与存储紧密结合，而 Pulsar 采用了**多层架构**，各层可以单独管理。Pulsar 在 broker 计算层进行计算，在 bookie 存储层管理有状态存储。

  

表面上来看，Pulsar 的架构似乎比 Kafka 的架构更为复杂，但实际情况并非如此。架构设计需要权衡利弊，Pulsar 采用了 BookKeeper，因此伸缩性更灵活，速度更快，性能更一致，运维开销更小。后文，我们会详细讨论这几个方面。

  

**>>>** **存储架构**

Pulsar 的多层架构影响了存储数据的方式。Pulsar 将 topic 分区划分为分片（segment），然后将这些分片存储在 Apache BookKeeper 的存储节点上，以提高性能、可伸缩性和可用性。

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/0EIbbCE3P2ZgNlKbfQ83tFzJSVG1CgfZvLV7oYEMYvU89HFtyQOrkJCyYxvHn4adJsCVXZfQx6S8QuU48Vjfvw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  

Pulsar 的无限分布式日志以分片为中心，借助扩展日志存储（通过 Apache BookKeeper）实现，内置分层存储支持，因此分片可以均匀地分布在存储节点上。由于与任一给定 topic 相关的数据都不会与特定存储节点进行捆绑，因此很容易替换存储节点或缩扩容。另外，集群中最小或最慢的节点也不会成为存储或带宽的短板。

  

Pulsar 架构能实现**分区管理，负载均衡**，因此使用 Pulsar 能够快速扩展并达到高可用。这两点至关重要，所以 Pulsar 非常适合用来构建关键任务服务，如金融应用场景的计费平台，电子商务和零售商的交易处理系统，金融机构的实时风险控制系统等。

  

通过性能强大的 Netty 架构，数据从 producers 到 broker，再到 bookie 的转移都是零拷贝，不会生成副本。这一特性对所有流应用场景都非常友好，因为数据直接通过网络或磁盘进行传输，没有任何性能损失。

  

**>****>****>** **消息消费**

Pulsar 的消费模型采用了**流拉取**的方式。流拉取是长轮询的改进版，不仅实现了单个调用和请求之间的零等待，还可以提供双向消息流。通过流拉取模型，Pulsar 实现了端到端的低延迟，这种低延迟比所有现有的长轮询消息系统（如 Kafka）都低。  

  

  

  

  

**使用简单**

  

  

  

**>****>****>** **运维简单**

在评估特定技术的操作简便性时，不仅要考虑初始设置，还要考虑长期维护和可伸缩性。需要考虑以下几项：

  

-   要跟上业务增长的速度，扩展集群是否快速、方便？
    
-   集群是否对多租户（对应于多团队、多用户）开箱可用？
    
-   运维（如替换硬件）是否会影响业务的可用性与可靠性？
    
-   是否可以轻松复制数据以实现数据的地理冗余或不同的访问模式？
    

  

长期使用 Kafka 的用户发现维护 Kafka 时，以上这些都很难做到。多数任务需要借助 Kafka 之外的工具，如用于管理集群再平衡的 cruise control，以及用于复制需求的 Kafka mirror-maker。

  

由于 Kafka 很难在不同的团队间共享，很多机构开发了用于支持和管理多个不同集群的工具。这些工具对大规模应用 Kafka 至关重要，但同时也增加了 Kafka 的复杂性。最适合管理 Kafka 集群的工具都是商业软件，不开源。那这就不意外了，囿于 Kafka 复杂的管理和运维，许多企业转而购买 Confluent 的商业服务。

  

相比之下，Pulsar 的目标是简化运维和可扩展。根据 Pulsar 的性能，对以上问题，我们回复如下：

  

Q：要跟上业务增长的速度，扩展集群的操作是否迅速便捷？  
A：Pulsar 有自动负载均衡的能力，集群中新增了计算和存储节点，可以自动、立即投入使用。这样 broker 之间可以迁移 topic 来平衡负载，新 bookie 节点可以立即接受新数据分片的写入流量，无需手动重新平衡或管理 broker。

  

Q：集群是否对多租户（对应于多团队、多用户）开箱可用？    

A：Pulsar 采用分层架构，租户和命名空间能够与机构或团队形成良好的逻辑映射，Pulsar 通过这种相同的机构支持简易 ACL、配额、自主服务控制，同时也支持资源隔离，因此集群使用者可以轻松管理、共享集群。

  

Q：运维任务（如替换硬件）是否会影响业务的可用性与可靠性？    

A：Pulsar 的 broker 是无状态的，替换操作简单，无需担心数据丢失。Bookie 节点会自动复制全部未复制的数据分片，而且用于解除和替换节点的工具为内置工具，很容易实现自动化。

  

Q：是否可以轻松复制数据以实现数据的地理冗余或不同的访问模式？    

A：Pulsar 具有内置的复制功能，可用于无缝跨地域同步数据或复制数据到其他集群，以实现其他功能（如灾备、分析等）。

  

和 Kafka 相比，Pulsar 为流数据的现实问题提供了更完善的解决方案。从这个角度看，Pulsar 拥有**更完善的核心功能集**，使用简单，因而允许使用者和开发者专注于业务的核心需求。

  

**>****>****>** **文档与学习**

Pulsar 比 Kafka 发展稍晚，Pulsar 的生态系统还不够完善，文档和培训资源也在陆续地补充。在过去的一年半时间里，Pulsar 正在逐步完善这些资料。以下为一些主要成果：

  

**👍 Pulsar Summit Virtual Conference 2020** 

Pulsar 的首次全球峰会，汇集了 36 个主题演讲，覆盖了 25+ 机构，注册参会者超过 600 人。

  

**👍 2020 年原创 50+ 视频及培训版块  
**https://streamnative.io/resource#pulsar-summit

  

**👍 Pulsar 每周直播及互动教程  
**https://www.bilibili.com/video/BV1T741147B6

  

**👍 业内领先讲师进行专业培训  
**https://bit.ly/31NLaCD

  

**👍 与战略商业伙伴每月举办一次网络研讨会  
**https://www.youtube.com/playlist?list=PLqRma1oIkcWhfmUuJrMM5YIG8hjju62Ev

  

**👍 发布来自腾讯、Yahoo!Japan**、OVHCloud、涂鸦**智能等多种应用场景的白皮书  
**https://streamnative.io/resource#white-paper

  

更多关于 Pulsar 文档和培训的内容，参阅 StreamNative 的 Resources 网站。  
https://streamnative.io/resource

  

**>****>****>** **企业支持**

Kafka 和 Pulsar 都提供企业级支持。多个大型供应商（包括 Confluent）为 Kafka 提供企业级支持。**StreamNative** 为 Pulsar 提供企业级支持，目前处于起步发展阶段。StreamNative 为企业提供全面托管的 Pulsar 云服务和 Pulsar 企业级支持服务。

  

StreamNative 团队在消息和事件流方面经验丰富，成长迅速。StreamNative 由 Pulsar 和 BookKeeper 核心成员创建。在 StreamNative 团队的帮助下，Pulsar 生态系统在短短几年时间里突飞猛进，得到了战略合作伙伴的支持，这种支持将会进一步促进 Pulsar 的发展，满足大量应用场景的需求。在下一篇，我们会详细介绍这部分内容。

  

Pulsar 的发展最近有了重大突破。

  

**👍 2020 年 3 月，OVHCloud 和 StreamNative 联合推出了 KoP（Kafka-on-Pulsar）**

通过向现有 Pulsar 集群添加 KoP 协议处理程序，用户不需要修改代码，就可以把现有的 Kafka 应用程序和服务迁移到 Pulsar。

  

**👍 2020 年 6 月，中国移动与 StreamNative 宣布推出另一重要产品 —— AoP（AMQP on Pulsar）**

使用 AoP，RabbitMQ 应用程序可以充分利用 Pulsar 的重要功能，如使用 Apache BookKeeper 和分层存储支持无限事件流存储等。

  

**👍 2020 年 9 月，StreamNative 开源了 MoP（MQTT on Pulsar）**

MoP 将 MQTT 协议处理插件引入 Pulsar broker。把 MoP 协议处理插件添加到现有 Pulsar 集群后，用户不用修改代码就可以将现有的 MQTT 应用程序和服务迁移到 Pulsar。

  

这样 MQTT 应用程序就可以利用 Pulsar 的特性，例如 Apache Pulsar 计算和存储分离的架构以及 Apache BookKeeper 保存事件流和 Pulsar 分层存储等特性。

  

**>****>****>** **生态集成**

随着 Pulsar 应用场景的迅速增加，Pulsar 社区发展壮大，全球用户高度参与。Pulsar 社区活跃，积极推动 Pulsar 生态系统的集成应用。过去的六个月，Pulsar 生态系统中官方支持的 connector 数量急剧增长。

  

为了进一步支持 Pulsar 社区的发展，StreamNative 推出了 StreamNative Hub。StreamNative Hub 支持用户搜索、下载集成应用，会进一步加速 Pulsar connector 和插件生态系统的发展。

https://hub.streamnative.io/

  

Pulsar 社区一直与其他社区密切合作，共同开发一系列集成项目，目前多个项目仍在进行中。已经完成的项目如：  
  
Pulsar 社区与 Flink 社区共同开发的 **Pulsar-Flink Connector**（FLIP-72 的一部分）。  
https://github.com/streamnative/pulsar-flink

  

通过 **Pulsar-Spark Connector**，用户可以使用 Apache Spark 处理 Apache Pulsar 中的事件。

https://github.com/streamnative/pulsar-spark

  

**SkyWalking Pulsar 插件**集成了 Apache SkyWalking 和 Apache Pulsar，用户可以通过 SkyWalking 追踪 Pulsar 消息。

https://github.com/apache/skywalking/tree/master/apm-sniffer/apm-sdk-plugin/pulsar-plugin

  

**>****>****>** **多元客户端库**

目前，Pulsar 官方客户端支持 **7** 种语言，而 Kafka 只支持 **1** 种语言。Confluent 发布博客声称 Kafka 目前支持 22 种语言，然而其官方客户端并不支持这么多种语言，而且有些语言已经不再维护。

  

根据最新统计，**Apache Kafka 官方客户端只支持 1 种语言**。  

https://github.com/apache/kafka/tree/trunk/clients/src/main/java/org/apache/kafka/clients

  

而 Apache Pulsar 官方客户端支持 7 种语言。  
http://pulsar.apache.org/docs/en/client-libraries/

  

-   Java
    
-   C
    
-   C++
    
-   Python
    
-   Go
    
-   .NET
    
-   Node
    

  

Pulsar 还支持由 Pulsar 社区开发的诸多客户端，如：

-   Rust
    
-   Scala
    
-   Ruby
    
-   Erlang
    

  

  

  

  

**性能与可用性**

  

  

  

**>****>****>** **吞吐量、延迟与容量**

Pulsar 和 Kafka 都被广泛用于各个企业，也各有优势，都能通过数量基本相同的硬件处理大流量。部分用户误以为 Pulsar 使用了很多组件，因此需要很多服务器来实现与 Kafka 相匹敌的性能。这种想法适用于一些特定硬件配置，但在多数资源配置相同的情况中，Pulsar 的优势更加明显，可以用相同的资源实现更好的性能。  

  

举例来说，Splunk 最近分享了他们选择 Pulsar 放弃 Kafka 的原因，其中提到“由于分层架构，Pulsar 帮助他们将成本降低了 **30% - 50%**，延迟降低了 **80% - 98%**，运营成本降低了 **33% - 50%**”。

https://www.slideshare.net/streamnative/why-splunk-chose-pulsarkarthik-ramasamy（参考幻灯片第 34 页）

  

Splunk 团队发现 Pulsar 可以更好地利用磁盘 IO，降低 CPU 利用率，同时更好地控制内存。

  

腾讯等公司选择 Pulsar 在很大程度上是因为 Pulsar 的性能。在腾讯计费平台白皮书中提到，腾讯计费平台拥有百万级用户，管理约 300 亿第三方托管账户，目前正在使用 Pulsar 处理日均数亿美元的交易。

https://streamnative.io/whitepaper/case-study-apache-pulsar-tencent-billing

  

考虑到 Pulsar 可预测的低延迟、更强的一致性和持久性保证，腾讯选择了 Pulsar。

  

**>****>****>** **有序性保证**

Apache Pulsar 支持四种不同订阅模式。单个应用程序的订阅模式由排序和消费可扩展性需求决定。以下为这四种订阅模式及相关的排序保证。

  

-   **独占（Exclusive）和灾备（Failover）**订阅模式都在分区级别支持强序列保证，支持跨 consumer 并行消费同一 topic 上的消息。
    
-   **共享（Shared）订阅**模式支持将 consumer 的数量扩展至超过分区的数量，因此这种模式非常适合 worker 队列应用场景。
    
-   **键共享（Key_Shared）订阅**模式结合了其他订阅模式的优点，支持将 consumer 的数量扩展至超过分区的数量，也支持键级别的强序列保证。
    

  

更多关于 Pulsar 订阅模式和相关排序保证的信息，可以参阅：  
http://pulsar.apache.org/docs/en/concepts-messaging/#subscriptions

  

  

  

  

**特 性**

  

  

  

**>****>****>** **内置流处理**

Pulsar 和 Kafka 对于内置流处理的目标不尽相同。针对较为复杂的流处理，Pulsar 集成了 Flink 和 Spark 这两套成熟的流处理框架，并开发了 Pulsar Functions 来处理轻量级计算。Kafka 开发并使用 Kafka Streams 作为流处理引擎。  

  

Kafka Streams 异常复杂，用户要将其作为流处理引擎，需要先弄清楚使用 KStreams 应用程序的场景及方法。对大多数轻量级计算应用场景来说，KStreams 过于复杂。

  

而 Pulsar Functions 轻松实现了轻量级计算，并允许用户创建复杂的处理逻辑，无需单独部署其他系统。Pulsar Functions 还支持原生语言和易于使用的 API。用户不必学习复杂的 API 就可以编写事件流应用程序。

  

最近，Pulsar 改进方案（Pulsar Improvement Proposal，PIP）中介绍了 Function Mesh。Function Mesh 是无服务器架构的事件流框架，结合使用多个 Pulsar Functions 以便构建复杂的事件流应用程序。

  

**>****>****>** **Exactly-Once 处理**

目前，Pulsar 通过 broker 端去重支持 exactly-once producer。这个重大项目正在开发中，敬请期待！

https://github.com/apache/pulsar/wiki/PIP-6:-Guaranteed-Message-Deduplication

  

**PIP-31** 提议 Pulsar 支持事务型消息流，目前正在开发中。这一特性提高了 Pulsar 的消息传递语义和处理保证。

https://github.com/apache/pulsar/wiki/PIP-31:-Transaction-Support

  

在交易型消息流中，每条消息只会写入一次、处理一次，即便 broker 或 Function 实例出现故障，也不会出现数据重复或数据丢失。交易型消息不仅简化了使用 Pulsar 或 Pulsar Functions 向应用程序写入的操作，还扩展了 Pulsar 支持的应用场景。

  

如果开发顺利，**Pulsar 2.7.0** 版本会支持事务型消息流，预计 2020 年 11 月发布。

  

**>****>****>** **Topic（日志）压缩**

Pulsar 支持用户根据需要选择数据格式来消费数据。应用程序可以根据需要选择使用**原始**数据或**压缩**数据。通过按需选择的方式，Pulsar 允许未压缩数据通过保留策略，控制数据无限增长，同时通过周期性压缩生成最新的实物化视图。内置的分层存储特性支持 Pulsar 从 BookKeeper 卸载未压缩数据到云存储中，从而降低长期存储的成本。

  

而 Kafka 不支持用户使用原始数据。并且，在数据压缩后，Kafka 会立即删除原始数据。

  

  

  

  

**用 例**

  

  

  

**>****>****>** **事件流**

雅虎最初开发 Pulsar 将其用作统一的发布/订阅消息平台（又称云消息）。现在，Pulsar 不仅是消息平台，还是**消息和事件流的统一平台**。Pulsar 引入了一系列工具，作为平台的一部分，为构建事件流应用程序提供必要的基础。Pulsar 支持以下事件流功能：

  

-   **无限事件流存储支持**通过向外扩展日志存储（通过 Apache BookKeeper）大规模存储事件，并且 Pulsar 内置的分层存储支持高质量、低成本的系统，如 S3、HDFS 等。
    
      
    
-   **统一的发布/订阅消息模型**方便用户向应用程序中添加消息。这一模型可以根据流量和用户需求进行伸缩。
    
      
    
-   **协议处理**框架、Pulsar 与 Kafka 的协议兼容性（KoP），以及 AMQP （AMQP-on-Pulsar）支持应用程序使用任何现有协议在任一位置生产和消费事件。
    
      
    
-   **Pulsar IO** 提供了一组与大型生态系统集成的 connector，用户不需要编写代码，即可从外部系统获取数据。
    
      
    
-   **Pulsar 与 Flink 的集成**可以全面处理复杂的事件。
    
      
    
-   **Pulsar Functions** 是一个轻量级无服务器框架，能够随时处理事件。
    

  

-   **Pulsar 与 Presto 的集成**（Pulsar SQL），数据专家和开发者能够使用 ANSI 兼容的 SQL 来分析数据和处理业务。
    

  

**>****>****>** **消息路由**

通过 Pulsar IO、Pulsar Functions、Pulsar Protocol Handler，Pulsar 具有完善的路由功能。Pulsar 的路由功能包括基于内容的路由、消息转换和消息扩充。

  

和 Kafka 相比，Pulsar 的路由能力更稳健。Pulsar 为 connector 和 Functions 提供了更灵活的部署模型。可以在 broker 中简单部署，也可以在专用的节点池中部署（类似于 Kafka Streams），节点池支持大规模扩展。Pulsar 还与 Kubernetes 原生集成。另外，可以将 Pulsar 配置为以 pod 的形式来调度 Functions 和 connector 的工作负载，充分利用 Kubernetes 的弹性。

  

**>****>****>** **消息队列**

如前文所述，Pulsar 最初的开发目的是作为统一的消息发布/订阅平台。Pulsar 团队深入研究了现有开源消息系统的优缺点，凭借丰富的经验，设计了统一的消息模型。

  

Pulsar 消息 API 结合**队列**和**流**的能力，不仅实现了 worker 队列以轮询的方式将消息发送给相互竞争的 consumer（通过共享订阅），还支持事件流：一是基于分区（通过灾备订阅）中消息的顺序；二是基于键范围（通过键共享订阅）中消息的顺序。用户可以在同一组数据上构建消息应用程序和事件流应用程序，而无需复制数据到不同的数据系统。

  

另外，Pulsar 社区还在尝试使 Apache Pulsar 原生支持不同的消息协议（如 AoP、KoP、MoP），以扩展 Pulsar 处理消息的能力。

  

  

  

  

**结语**

  

  

  

Pulsar 社区发展迅猛，随着 Pulsar 技术的发展和应用场景的增加，Pulsar 生态也在日益壮大。

  

Pulsar 具有许多优势，在统一的消息和事件流平台脱颖而出，成为大众选择。和 Kafka 相比，Pulsar 弹性更灵活，在运维和扩展上更为简单。

  

新技术的推出和采用都需要一些时间，Pulsar 不仅提供了全套解决方案，安装后可立即投入生产环境，维护成本低。Pulsar 涵盖了构建事件流应用程序所需的基础，集成了丰富的内置功能（包括各种工具）。Pulsar 工具不需要单独安装，即可立即使用。

  

StreamNative 一直致力于开发 Pulsar 新功能，加强现有功能，同时促进社区发展。2020 年我们定下了宏伟目标，目前一切进展顺利，**期待 11 月发布 Pulsar 2.7.0。**


在补充之前，我们先来快速回顾一下上一篇文章中提到的 7 个原因：

  

-   流式处理和队列的合体。Kafka 或 RabbitMQ 在单个平台上都只能处理其中一种方式。而 Pulsar 就像一个合二为一的产品，可以同时处理实时流和消息队列。
    
-   支持分区，但不是必需的。如果只需要一个主题，可以使用一个主题而无需使用分区。如果需要保持多个消费者实例的处理速率，也不需要使用分区。
    
-   分布式的日志。Pulsar 日志是分布式的，可以水平扩展。因此不会保存在单台服务器上，任何一台服务器都不会成为整个系统的瓶颈。
    
-   无状态的 broker。云原生应用程序开发的理想场景，数据的分发和保存相互独立。
    
-   原生支持跨地域复制，而且配置简单。无论是全局分布式应用程序还是灾备方案，任何人都可以通过 Pulsar 搞定。
    
-   稳定的表现。基准测试表明，Pulsar 可以在提供较高吞吐量的同时保持较低的延迟。
    
-   完全开源。不论是与 Kafka 相似的特性，还是 Kafka 没有的特性都是开源的。
    

  

以上便是之前提到的 7 个原因。如果你想了解关于以上几点的详细内容，可以查看开头提到的文章。这些原因似乎已经很多了，但是我发现了更多。

  

  

**>>> 多租户处理 <<<**

上一篇文章中，我应该多谈谈多租户。Pulsar 对多租户的支持是一个很重要的特性。在企业中，消息基础架构会被多团队、多项目使用。为每个团队（项目）分别维护一个独立的消息系统集群代价过高且难以维护。

  

Pulsar 可以有多个租户，这些租户可以有多个命名空间，以保持内容顺序。再加上每个命名空间的访问控制、配额、速率限制等，可以想象，将来我们可以只使用一个集群就能处理多租户问题。其实不仅我们考虑到这个问题，Kafka 也会考虑。在 Kafka 改进建议（KIP）KIP-37 中可以看到与此相关的内容。这个问题已经讨论了一段时间了。

  

  

**>>> Quorum 复制 <<<**

接下来，我会讲到很多细节。要想确保消息不丢失，消息传递系统会配置每条消息生成 2 或 3 个副本，以防出错。Kafka 使用 follow-the-leader 模型来实现这一点。Leader 存储消息，而 follower 复制 leader 上的消息。  

  

一旦有足够多的 follower 确认已经完成了复制，Kafka 就“高兴”了。Pulsar 使用 Quorum 模型。它把消息发送给一堆节点，一旦有足够多的节点确认它们已经接收到消息，Pulsar 就很“高兴”。

  

Quorum 复制更加民主，没有这种 leader-follower 层次结构。所有选票均等时，多数胜利。但这与技术无关。重要的是，随着时间的推移，Quorum 复制倾向于提供更一致的行为。

  

这或许可以解释为什么 Pulsar 具有更一致的延迟性能。如果你想了解关于 Kafka 和 Pulsar 延迟的详细信息，请查看我的这篇博客（文章很长，别说我没提醒你哦）。  
  
事实上，Kafka 也一直在考虑使用 Quorum 复制来改善延迟一致性。关于此，可以查看 KIP-250 中的讨论。

  

  

**>>> 分层存储 <<<**

像 Kafka 这样的流处理系统，其一大优点在于它能够重放已经被消费的消息。如果你第一次见到就很喜欢这些消息，则可以进行重播以更正某些内容，或围绕它们构建新的应用程序，这也是很有趣的。

  

如果你非常喜欢这些消息，想把它们永远保存下来，那该怎么办？比如，如果你在做事件溯源。这听起来很不错，但永久可是一段很长的时间，而且永久存储消息也可能很贵，特别是存储在高性能固态硬盘上。这些硬盘需要维持消息系统保持良好的运行状态。

  

如果能把那些旧消息（那些以后可能会再用到的消息）转移到相对便宜的存储解决方案中，是否可行呢？如果可以使用像 Amazon S3 存储桶这样廉价的云存储，那岂不是很棒吗？你可能已经猜到我要说什么了。

  

借助 Pulsar 分层存储，你可以把那些旧消息自动推送到几乎无限的、廉价的云存储中，然后像检索新消息一样执行相关操作。我敢打赌，Kafka 希望拥有该功能。没错，他们会的。在 KIP-405 中可以看到相关讨论。

  

  

**>>> 端到端加密 <<<**

显然，安全是很重要的，谁都不希望信息安全被偷窥。当然，你会在客户端和消息传递系统之间使用 TLS（在传输过程中加密）。这样做时，消息传递系统必须解密该连接，以便了解客户端想要表达的内容。

  

然后，它把未加密的消息保存在磁盘上。当然，你会坚持对磁盘进行加密，这样即使磁盘被偷，消息仍然是安全的（静态加密）。但在这两种情况下，消息传递系统都需要有数据的密钥。如果不是这样，那就是在处理一大堆莫名其妙的天书。

  

许多情况下，这种级别的加密已经足够好了，但是如果你想要确保没有人可以偷窥你的消息，则需要进行端到端加密。生产者在发送消息之前使用与接收消息的使用者共享的密钥对消息进行加密。当消息保存在消息系统的磁盘上时，就会被加密，而消息系统没有密钥。消息传递系统可以完成它的工作，但是你的消息对于消息传递系统来说是就像天书一样，所以是十分安全的。

  

Pulsar 可以在其 Java 客户端中进行端到端加密。Kafka 一直在 KIP-317 中讨论这一操作。

  

**>>> Broker 平衡 <<<**

Pulsar broker 是无状态的。组件无状态是件非常棒的事情，当一个组件过载时，你可以添加另一个组件来处理负载。当新客户端连接时，可以将它们定向到新实例。但这并不能帮助到第一次被重载的实例。你需要将一些工作从重载实例转移到新的实例上。换句话说，需要重新平衡负载。

  

Pulsar 会自动进行 broker 负载平衡。它监视 broker 的 CPU、内存、网络（不是磁盘，我提到的代理是无状态的）的使用情况，并调整负载以保持平衡。这意味着你不需要在单个 broker 热点时扩容 broker 集群，除非 broker 集群服务能力到达上限。

  

你也可以使用 Kafka 进行代理负载平衡。但是，你必须多安装一个程序，例如 LinkedIn 的 Cruise Control。或者也可以使用 Confluent 的负载平衡器工具（这款工具是需要付费的）。

  

  

**>>> 社区和生态系统 <<<**

有人批评我没有提到 Kafka 社区和生态系统的规模和丰富程度。这个批评很中肯。在社区和生态系统类别中，Kafka 确实比 Pulsar 做得好。作为一个开源项目，Kafka 已经开创了 5 年，所以它有一个更大的社区，有更多相关的项目，并且在 Stack Overflow 上有更多相关的问题与答案。

  

随着大家不断贡献新的组件和周边集成，Pulsar 社区日益发展壮大，Slack 社区的小伙伴们都很友好，也乐于助人。我还想补充一下：Pulsar 在许多方面受到 Kafka 的启发，并且站在了巨人 Kafka 的肩膀上。Kafka 项目和社区值得称赞与尊重。有时候听起来像是我不尊重 Kafka，其实我很尊重 Kafka，我只是对 Pulsar 更有好感罢了。

  

  

**>>> KAFKA 的合理替代品 <<<**

在这篇文章和上一篇文章中，我列举了12 条选择 Pulsar 的理由。更酷的是，我发现随着对 Pulsar 了解的加深，我总能找到新的理由。因此，我可能还会写第三篇和这个主题相关博客。敬请关注。

  

我认为 Pulsar 是 Kafka 的合理替代品。Pulsar 支持 Kafka 的大部分功能，而且 Pulsar 有更多优势（目前我列举了 12 个）。了解 Pulsar 的人越多，它的发展势头就越大。如果你正在评估流和/或队列系统，考虑一下 Apache Pulsar 吧。

  

想要随时掌握 Pulsar 的研发进展、用户案例和热点话题吗？快来关注 Apache Pulsar 和 StreamNative 微信公众号，我们会在第一时间和您分享 Pulsar 的一切。

  

# Reference
https://mp.weixin.qq.com/s?__biz=MzUyMjkzMjA1Ng==&mid=2247485681&idx=1&sn=f6caa8b891153844d4625e0a7cdc4e4d&chksm=f9c512c6ceb29bd0c78043e8642046eecb5c07898557b4541b5ab3c41cb6d98f652704da316e&scene=21#wechat_redirect


https://mp.weixin.qq.com/s?__biz=MzUyMjkzMjA1Ng==&mid=2247484614&idx=1&sn=8701fb8a322b1018634d3ea5971d5194&chksm=f9c51ef1ceb297e73b2303c76f983514e055db537d799cbebb916dcaf12eb8c1509c4ece952f&scene=21#wechat_redirect