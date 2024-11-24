#nsq #mq 

![](https://tech.youzan.com/building-nsq-client-cont/)

## overview

有赞的自研版 NSQ 在高可用性以及负载均衡方面进行了改造，自研版的 nsqd 中引入了数据分区以及副本，副本保存在不同的 nsqd 上，达到容灾目的。此外，自研版 NSQ 在原有 Protocol Spec 基础上进行了拓展，支持基于分区的消息生产、消费，以及基于消息分区的有序消费，以及消息追踪功能。

为了充分支持自研版 NSQ 新功能，在要构建 NSQ client 时，需要在兼容原版 NSQ 的基础上，实现额外的设计。本文作为[《Building Client Libraries》](http://nsq.io/clients/building_client_libraries.html)的拓展，为构建有赞自研版 NSQ client 提供指引。

参考 NSQ 官方的构造 client 指南的结构，接下来的文章分为如下部分：

1 workflow 及配置  
2 nsqd 发现服务  
3 nsqd 建连  
4 发送/接收消息  
5 顺序消费

> 本文根据有赞自研版 nsq 的新特性，对 nsq 文档[1](https://tech.youzan.com/building-nsq-client-cont/#fn:1)中构建 nsq client 的专题进行补充。在阅读[《Building Client Libraries》](http://nsq.io/clients/building_client_libraries.html)的基础上阅读本文，更有助于理解。

## workflow 及配置

通过一张图，浏览一下 nsq client 的工作流程。

![[Pasted image 20221102102908.png]]client 作为消息 producer 或者 consumer 启动后，负责 lookup 的功能通过 nsqlookupd 进行 nsqd 服务发现。对于服务发现返回的 nsqd 节点，client 进行建连操作以及异常处理。nsq 连接建立后，producer 进行消息发送，consumer 则监听端口接收消息。同时，client 负责响应来自 nsqd 的心跳，以保持连接不被断开。在和 nsqd 消息通信过程中，client 通过 lookup 发现，持续更新 nsq 集群中 topic 以及节点最新信息，并对连接做相应更新操作。当消息发送/消费结束时，client 负责关闭相应 nsqd 连接。文章在接下来讨论这一流程中的关键步骤，对相应功能的实现做更详细的说明。

### 配置 client

自研版 NSQ 改造自开源版 NSQ，继承了开源版 NSQ 中的配置。[1](https://tech.youzan.com/building-nsq-client-cont/#fn:1)中 Configuration 段落的内容适用于有赞自研版。唯一需要指出的地方是，开源版 nsq 将使用 nsqlookupd 作为 nsqd 服务发现的一个可选项，基于配置灵活性的考量，开源版 NSQ 允许 client 通过 nsqd 的地址直接建立连接，自研版 NSQ 由于支持动态负载，nsqd 之间的主从关系在集群中发生切换的时候，需要依赖自研版的 nsqlookupd 将变更信息反馈给 nsq client。基于此，使用 nsqlookupd 进行服务发现，在自研版 NSQ 中是一个“标配”。我们也将在下一节中对服务发现过程做详细的说明。

## nsqd 发现服务

开源版中，提供 nsqd 发现服务作为 nsqlookupd 的重要功能，用于向消息的 consumer/producer 提供可消费/生产的 nsqd 节点。上文提到，区别于开源版本，自研版的 nsqlookupd 将作为服务发现的唯一入口。nsq client 负责调用 nsqlookupd 的 lookup 服务，并通过 poll，定时更新消息在 nsq 集群上的读写分布信息。根据 lookup 返回的结果，nsq client 对 nsqd 进行建连。

在自研版中访问 lookup 服务的方式和开源版一样简单直接，向 nsqlookupd 的 http 端口 GET：/lookup?topic={topic_name} 即可完成。不同之处在于，自研版本 NSQ 的 lookup 服务中支持两种新的查询参数：

> GET：/lookup?topic={topic_name}&access={r or w}&metainto=true

其中：access 用于区分 nsq client 的生产/消费请求。代表 producer 的 lookup 查询中的参数为 access=w，consumer 的 lookup 查询为 access=r。metainfo 参数用于提示 nsqlookup 是否返回查询 topic 的分区元数据，包括查询 topic 的分区总数，以及副本个数。顺序消费时，prodcuer 通过返回的分区元数据来判断 lookup 响应中返回的分区数是否完整，顺序消费和生产的详细部分我们将在发送消息章节中讨论。client 在访问 lookup 服务时，根据 prodcuer&consumer 角色的差别可以使用两类查询参数

-   Producer GET:/lookup?topic=test&access=w&metainfo=true
-   Consumer GET:/lookup?topic=test&access=r

在设计上，由于 metainfo 提供的信息是为 producer 在顺序消费场景下的生产，为了减少 nsqlookupd 服务压力，代表 consumer 的 lookup 查询无需携带 metainfo 参数。自研版 lookup 的响应和开原版本兼容：
```json

{
    "status_code":200,
    "status_txt":"OK",
    "data":{
        "channels":[
            "BaseConsumer"
        ],
        "meta":{
            "partition_num":1,
            "replica":1
        },
        "partitions":{
            "0":{
                "id":"XX.XX.XX.XX:33122",
                "remote_address":"X.X.X.X:33122",
                "hostname":"host-name",
                "broadcast_address":"XX.XX.XX.XX",
                "tcp_port":4150,
                "http_port":4151,
                "version":"0.3.7-HA.1.5.4.1",
                "distributed_id":"XX.XX.XX.XX:4250:4150:338437"
            }
        },
        "producers":[
            {
                "id":"XX.XX.XX.XX:33122",
                "remote_address":"XX.XX.XX.XX:33122",
                "hostname":"host-name",
                "broadcast_address":"XX.XX.XX.XX",
                "tcp_port":4150,
                "http_port":4151,
                "version":"0.3.7-HA.1.5.4.1",
                "distributed_id":"XX.XX.XX.XX:4250:4150:338437"
            }
        ]
    }
}

```
### lookup 流程

client 的 lookup 流程如下：  
![[Pasted image 20221102103653.png]]

自研版 nsqlookupd 添加了 listlookup 服务，用于发现接入集群中的 nsqlookupd。nsq client 通过访问一个已配置的 nsqlookupd 地址，发现所有 nsqlookupd。获得 nsqlookupd 后，nsq client 遍历得到的 nsqlookupd 地址，进行 lookup 节点发现。nsq client 合并遍历获得的节点信息并返回。nsq client 在访问 listlookup 以及 lookup 服务失败的场景下（如，访问超时），nsq client 可以尝试重试。lookup 超过最大重试次数后依然失败的情况，nsq client 可以降低访问该 nsqlookupd 的优先级。client 定期查询 lookup，保证 client 更新连接到有效的 nsqd。

## nsqd 建连

自研版 nsqd 在建连时遵照[1](https://tech.youzan.com/building-nsq-client-cont/#fn:1)中描述的建连步骤，通过 lookup 返回结果中 partitions 字段中的{broadcast_address}:{tcp_port}建立 TCP 连接。自研版中，一个独立的 TCP 连接对应一个 topic 的一个分区。consumer 在建连的时候需要建立与分区数量对应的 TCP 连接，以接收到所有的分区中的消息。client 的基本建连过程依然遵守[1](https://tech.youzan.com/building-nsq-client-cont/#fn:1)中的 4 步:

-   client 发送 magic 标志
-   client 发送 an IDENTIFY command 并处理返回结果
-   client 发送 SUB command (指定目标的 topic 以及分区)， 并处理返回结果
-   client 发送 RDY 命令

client 通过自研版 NSQ 中拓展的SUB命令，连接到指定 topic 的指定分区。
```bash
SUB <topic> <channel_name> <topic_partition> \n
topic_name -- 消费的 topic 名称
topic_partition -- topic 的合法分区名
channel_name -- channel
```

Response:

```bash
OK
```

Error response:
```bash
E_INVALID
E_BAD_TOPIC
E_BAD_CHANNEL
E_SUB_ORDER_IS_MUST  当前 topic 只支持顺序消费
```

client 在建连过程中，向 lookup 返回的每一个 nsqd partition 发送该命令。SUB 命令的出错响应中，自研版本 NSQ 中加入了最后一个错误代码，当 client SUB 一个配置为顺序消费的 topic 时，client 会收到该错误。相应的携带分区号的 PUB 命令格式为：
```bash
PUB <topic_name> <topic_partition>\n
[ 4-byte size in bytes ][ N-byte binary data ]

topic -- 合法 topic 名称
partitionId -- 合法分区名
```

Response:
```bash
OK
```

Error response:
```bash
E_INVALID
E_BAD_TOPIC
E_BAD_MESSAGE
E_PUB_FAILED
E_FAILED_ON_NOT_LEADER    当前尝试写入的 nsqd 节点在副本中不是 leader
E_FAILED_ON_NOT_WRITABLE  当前的 nqd 节点禁止写入消息
E_TOPIC_NOT_EXIST         向当前连接中写入不存在的 topic 消息
```

自研版本 NSQ 中加入了最后三个错误代码，分别用于提示当前尝试写入的 nsqd 节点在副本中不是 leader，以及当前的 nqd 节点禁止写入。client 在接收到错误的时候，应该直接关闭 TCP 连接，等待 lookup 定时查询更新 nsqd 节点信息，或者立刻发起 lookup 查询。如果没有传入 partition id, 服务端会选择默认的 partition. 客户端可以选择 topic 的 partition 发送算法，可以根据负载情况选择一个 partition 发送，也可以固定的 Key 发送到固定的 partition。client 在消费时，可以指定只消费一个 partition 还是消费所有 partition。每个 partition 会建立独立的 socket 连接接收消息。client 需要处理多个 partition 的 channel 消费问题。

## 发送/接收消息

主要讨论生产者和消费者对消息的处理

### 生产者发送消息

client的消息生产流程如下：  
![[Pasted image 20221102104611.png]]

建连过程中，对于消息生产者，client 在接收到对于 IDENTITY 的响应之后，使用 PUB 命令向连接发送消息。client 在 PUB 命令后需要解析收到的响应，出现 Error response 的情况下，client 应当关闭当前连接，并向 lookup 服务重新获取 nsqd 节点信息。client 可以将 nsqd 连接通过池化，在生产时进行复用，连接池中指定 topic 的连接为空时，client 将初始化该连接，因失败而关闭的连接将不返回连接池。

### 消费者接收消息

client 的消费流程如下：  
![[Pasted image 20221102104825.png]]

client 在和 nsqd 建立连接后，使用异步方式消费消息。nsqd 定时向连接中发送心跳相应，用于检查 client 的活性。client 在收到心跳帧后，向 nsqd 回应 NOP。当 nsqd 连续两次发送心跳未能收到回应后，nsqd 连接将在服务端关闭。参考[1](https://tech.youzan.com/building-nsq-client-cont/#fn:1)中消息处理章节的相关内容，client 在消费消息的时候有如下情景：

1.  消息被顺利消费的情况下，FIN 通过 nsqd 连接发送；
2.  消息的消费失败的情况下，client 需要通知 nsqd 该条消息消费失败。client 通过 REQ 命令携带延时参数，通知 nsqd 将消息重发。如果消费者没有指定延时参数，client 可以根据消息的 attempt 参数，计算延时参数。client 可以允许消费者指定当某条消息的消费次数超过指定的次数，client 是否可以将该条消息丢弃，或者重新发送至 nsqd 消息队列尾部去后，再 FIN；
3.  消息的消费需要更多的时间，client 发送 TOUCH 命令，重置 nsqd 的超时时间。TOUCH 命令可以重复发送，直到消息超时，或者 client 发送 FIN，REQ 命令。是否发送 TOUCH 命令，决定权应当由消费者决定，client 本身不应当使用 TOUCH；
4.  下发至 client 的消息超时，此时 nsqd 将重发该消息。此种情况下，client 可能重复消费到消息。消费者在保证消息消费的幂等性的同时，对于重发消息 client 应当能够正常消费；

### 消息 ACK

自研版本 NSQ 中，对原有的消息 ID 进行了改造，自研版本中的消息 ID 长度依然为 16 字节：

[8-byte internal id][8-byte trace id]

高位开始的 16 字节，是自研版 NSQ 的内部 ID，后 16 字节是该条消息的 TraceID，用于 client 实现消息追踪功能。

## 顺序消费

基于 topic 分区的消息顺序生产消费，是自研版 NSQ 中的新功能。自研版 NSQ 允许生产者通过 shardingId 映射将消息发送到固定 topic 分区。建立连接时，消费者在发送 IDENTIFY 后，通过新的 SubOrder 命令连接到顺序消费 topic。顺序消费时，nsqd 的分区在接收到来自 client 的 FIN 确认之前，将不会推送下一条消息。

在 nsqd 配置为顺序消费的 topic 需要 nsq client 通过 SubOrder 进行消费。向顺序消费 topic 发送 Sub 命令将会收到错误信息，同时连接将被关闭。

E_SUB_ORDER_IS_MUST

client 在启动消费者前，可以通过配置指导 client 在 SUB 以及 SUB_ORDER 命令之间切换，或者基于 topic 进行切换。SUB_ORDER 在 TCP 协议的格式如下：
```bash
SUB_ORDER  <topic_name> <channel_name> <topic_partition>\n

topic_name -- 进行顺序消息的 topic 名称
channel_name -- channel 名称
topic_partition -- topic 分区名称
```

NSQ 新集群中，消息的顺序生产／消费基于 topic 分区。消息生产者通过指定 shardingID，向目标 partition 发送消息；消息消费者通过指定分区 ID，从指定分区接收消息。Client 进行顺序消费时时，TCP 连接的 RDY 值相当于将被 NSQ 服务端指定为 1，在当前消息 Finish 之前不会推送下一条。NSQ 服务器端 topic 进行强制消费配置，当消费场景中日志出现

E_SUB_ORDER_IS_MUST

错误消息时，说明该 topic 必须进行顺序消费。

顺序消费的场景由消息生产这个以及消息消费者两方的操作完成：

-   消息生产者通过 SUB_ORDER 命令，连接到 Topic 所在的所有 NSQd 分区；
-   消息消费者通过设置 shardingID 映射，将消息发送到指定 NSQd 的指定 partition，进行生产；

### 顺序消费场景下的消息生产

client 在进行消息生产时，将携带有相同 shardingID 的消息投递到同一分区中，分区的消息则通过 lookup 服务发现。作为生产者，client 在 lookup 请求中包含 metainfo 参数，用于获得 topic 的分区总数。client 负责将 shardingID 映射到 topic 分区，同时保证映射的一致性：具有相同的 shardindID 的消息始终被投递到固定的分区连接中。当 shardingID 映射到的 topic 分区对于 client 不可达时，client 结束发送，告知生产者返回错误信息，并立即更新 lookup。

### 顺序消费场景下的消息消费

client 在进行消息消费时，通过 SUB_ORDER 命令连接到 topic 所有分区上。顺序消费的场景中，当某条消费的消息超时或 REQUEUE 后，nsq 将会立即将该条消息下发。消息超时或者超过指定重试次数后的策略由消费者指定，client 可以对于重复消费的消息打印日志或者告警。

## 消息追踪功能的实现

新版 NSQ 通过在消息 ID 中增加 TraceID，对消息在 NSQ 中的生命周期进行追踪，client 通过 PUBTRACE 新命令将需要追踪的消息发送到 NSQ，PUB_TRACE 命令将包含 traceID 和消息字节码，格式如下：

PUB_TRACE <topic_name> <topic_partition>\n
[ 4-byte size in bytes ][8-byte size trace id][ N-byte binary data ]

相较于 PUB 命令，PUB_TRACE 在消息体中增加了 traceID 字段，client 在实现时，传递 64 位整数。NSQ 对 PUB_TRACE 的响应格式如下：

OK(2-bytes)+[8-byte internal id]+[8-byte trace id from client]+[8-byte internal disk queue offset]+[4 bytes internal disk queue data size]

client 可通过配置或者动态开关，开启或关闭消息追踪功能，让生产者在 PUB 和 PUB_TRACE 命令之间进行切换。为了得到完整的 Trace 信息，建议 client 在生产者端打印 PUB_TRACE 响应返回的信息，在消费者端打印收到消息的 TraceID 和时间。

## 总结

本文结合自研版 NSQ 新特性，讨论了构建支持服务发现、顺序消费、消息追踪等新特性的 client 过程中的一些实践。

# Reference
https://tech.youzan.com/building-nsq-client-cont/