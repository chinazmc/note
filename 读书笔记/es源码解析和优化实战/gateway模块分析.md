#es 

gateway模块负责集群元信息的存储和集群重启时的恢复。

# 元数据
ES中存储的数据有以下几种：
· state元数据信息；
· index Lucene生成的索引文件；
· translog事务日志。
元数据信息又有以下几种：
· nodes/0/_state/*.st，集群层面元信息；
· nodes/0/indices/{index_uuid}/_state/*.st，索引层面元信息；
· nodes/0/indices/{index_uuid}/0/_state/*.st，分片层面元信息。

分别对应ES中的数据结构：
· MetaData（集群层），主要是clusterUUID、settings、templates等；
· IndexMetaData（索引层），主要是numberOfShards、mappings等；
· ShardStateMetaData（分片层），主要是version、indexUUID、primary等。
上述信息被持久化到磁盘，需要注意的是：持久化的state不包括某个分片存在于哪个节点这种内容路由信息，集群完全重启时，依靠gateway的recovery过程重建RoutingTable。当读取某个文档时，根据路由算法确定目的分片后，从RoutingTable中查找分片位于哪个节点，然后将请求转发到目的节点。

# 元数据的持久化
只有具备Master资格的节点和数据节点可以持久化集群状态。当收到主节点发布的集群状态时，节点判断元信息是否发生变化，如果发生变化，则将其持久化到磁盘中。

# 元数据的恢复
上述的三种元数据信息被持久化存储到集群的每个节点，当集群完全重启（full restart）时，由于分布式系统的复杂性，各个节点保存的元数据信息可能不同。此时需要选择正确的元数据作为权威元数据。
gateway的recovery负责找到正确的元数据，应用到集群。
当集群完全重启，达到 recovery 条件时，进入元数据恢复流程，一般情况下，recovery 条件由以下三个配置控制。
· gateway.expected_nodes，预期的节点数。加入集群的节点数（数据节点或具备Master资格的节点）达到这个数量后立即开始gateway的恢复。默认为0。
· gateway.recover_after_time，如果没有达到预期的节点数量，则恢复过程将等待配置的时间，再尝试恢复。默认为5min。
· gateway.recover_after_nodes，只要配置数量的节点（数据节点或具备Master资格的节点）加入集群就可以开始恢复。
假设取值为10、5min、8，则集群启动时节点达到10个则立即进入recovery；如果一直没有达到10个，5min超时后如果节点达到8个也进入recovery。
还有一些更细致的配置项，原理与上面三个配置类似：
· gateway.expected_master_nodes，预期的具备 Master 资格的节点数，加入集群的具备Master资格的节点数达到这个数量后立即开始gateway的恢复。默认为0。
· gateway.expected_data_nodes，预期的具备数据节点资格的节点数，加入集群的具备数据节点资格的节点数量达到这个数量后立即开始gateway的恢复。默认为0。
· gateway.recover_after_master_nodes，指定数量的具备Master资格的节点加入集群后就可以开始恢复。
· gateway.recover_after_data_nodes，指定数量的数据节点加入集群后就可以开始恢复。
当集群完全启动时，gateway模块负责集群层和索引层的元数据恢复，分片层的元数据恢复在allocation模块实现，但是由gateway模块在执行完上述两个层次恢复工作后触发。
当集群级、索引级元数据选举完毕后，执行submitStateUpdateTask 提交一个 source 为local-gateway-elected-state的任务，触发获取shard级元数据的操作，这个Fetch过程是异步的，根据集群分片数量规模，Fetch过程可能比较长，然后submit任务就结束，gateway流程结束。因此，三个层次的元数据恢复是由gateway模块和allocation模块共同完成的，在Gateway将集群级、索引级元数据选举完毕后，在submitStateUpdateTask提交的任务中会执行allocation模块的reroute继续后面的流程。

# 元数据恢复流程分析
主要实现在 GatewayService 类中，它继承自ClusterStateListener，在集群状态发生变化（clusterChanged）时触发，仅由Master节点执行。
（1）Master选举成功之后，判断其持有的集群状态中是否存在STATE_NOT_RECOVERED_BLOCK，如果不存在，则说明元数据已经恢复，跳过gateway恢复过程，否则等待。
（2）Master从各个节点主动获取元数据信息。
（3）从获取的元数据信息中选择版本号最大的作为最新元数据，包括集群级、索引级。
（4）两者确定之后，调用allocation模块的reroute，对未分配的分片执行分配，主分片分配过程中会异步获取各个shard级别元数据，默认超时为13s。
集群级和索引级元数据信息是根据存储在其中的版本号来选举的，而主分片位于哪个节点却是allocation模块动态计算出来的，先前主分片不一定还被选为新的主分片。关于主分片选举策略我们在allocation一章中介绍。
## 选举集群级和索引级别的元数据
首先向有Master资格的节点发起请求，获取它们存储的元数据：
等待回复时，必须收到所有节点的回复，无论回复成功还是失败（节点通信失败异常会被捕获，作为失败处理），此处没有超时。在收齐的这些回复中，有效元信息的总数必须达到指定数量。异常情况下，例如，某个节点上元信息读取失败，则回复信息中元数据为空。
## 触发allocation
当上述两个层次的元信息选举完毕，调用clusterService.submitStateUpdateTask 提交一个集群任务，该任务在masterService#updateTask线程池中执行，实现位于GatewayRecoveryListener# onSuccess。主要工作是构建集群状态（ClusterState），其中的内容路由表依赖allocation模块协助完成，调用allocationService.reroute 进入下一阶段：异步执行分片层元数据的恢复，以及分片分配。updateTask线程结束。至此，gateway恢复流程结束，集群级和索引级元数据选举完毕，如果存在未分配的主分片，则分片级元数据选举和分片分配正在进行中。

# 思考
· 元数据信息是根据版本号选举出来的，而元数据写入成功的条件是“多数”，因此，保证进入recovery的条件为节点数量为“多数”，可以保证集群级和索引级的一致性。
· 获取各节点存储的元数据，然后根据版本号选举时，仅向具有Master资格的节点获取元数据。