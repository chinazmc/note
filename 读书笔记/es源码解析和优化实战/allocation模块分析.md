#es 

本章主要分析allocation模块的结构和原理，然后以集群启动过程为例分析allocation模块的工作过程。

# 什么是allocation
分片分配就是把一个分片指派到集群中某个节点的过程。分配决策由主节点完成，分配决策包含两方面：
· 哪些分片应该分配给哪些节点；
· 哪个分片作为主分片，哪些作为副分片。

对于新建索引和已有索引，分片分配过程也不尽相同。不过不管哪种场景，ES都通过两个基础组件完成工作：allocators和deciders。allocators尝试寻找最优的节点来分配分片，deciders则负责判断并决定是否要进行这次分配。

· 对于新建索引，allocators负责找出拥有分片数最少的节点列表，并按分片数量升序排序，因此分片较少的节点会被优先选择。所以对于新建索引，allocators的目标就是以更均衡的方式把新索引的分片分配到集群的节点中。然后deciders依次遍历allocators给出的节点，并判断是否把分片分配到该节点。例如，如果分配过滤规则中禁止节点A持有索引idx中的任一分片，那么过滤器也阻止把索引idx分配到节点A中，即便A节点是allocators从集群负载均衡角度选出的最优节点。需要注意的是，allocators只关心每个节点上的分片数，而不管每个分片的具体大小。这恰好是 deciders 工作的一部分，即阻止把分片分配到将超出节点磁盘容量阈值的节点上。

· 对于已有索引，则要区分主分片还是副分片。对于主分片，allocators只允许把主分片指定在已经拥有该分片完整数据的节点上。而对于副分片，allocators则是先判断其他节点上是否已有该分片的数据的副本（即便数据不是最新的）。如果有这样的节点，则allocators优先把分片分配到其中一个节点。因为副分片一旦分配，就需要从主分片中进行数据同步，所以当一个节点只拥分片中的部分数据时，也就意味着那些未拥有的数据必须从主节点中复制得到。这样可以明显地提高副分片的数据恢复速度。

# 触发时机
触发分片分配有以下几种情况：
· index增删；
· node增删；
· 手工reroute；
· replica数量改变；
· 集群重启。

# allocation模块结构概叙
这个复杂的分配过程在reroute函数中实现：
AllocationService.reroute
此函数对外有两种重载，一种是通过接口调用的手工 reroute，另一种是内部模块调用的reroute。本章以内部模块调用的reroute为例，手工reroute过程与此类似。
AllocationService.reroute对一个或多个主分片或副分片执行分配，分配以后产生新的集群状态。Master节点将新的集群状态广播下去，触发后续的流程。对于内部模块调用，返回值为新产生的集群状态，对于手工执行的reroute命令，返回命令执行结果。

# allocators
目前，allocators的类型如下图所示。
![[Pasted image 20220404172213.png]]
allocators负责为某个特定的分片分配目的节点。每个allocator的主要工作是根据某种逻辑得到一个节点列表，然后调用deciders去决策，根据决策结果选择一个目的node。
allocators分为gatewayAllocator和shardsAllocator两种。gatewayAllocator是为了找到现有分片，shardsAllocator是根据权重策略在集群的各节点间均衡分片分布。
其中gatewayAllocator又分主分片和副分片的allocator。下面概述每个allocator的作用。
· primaryShardAllocator：找到那些拥有某分片最新数据的节点；
· replicaShardAllocator：找到磁盘上拥有这个分片数据的节点；
· BalancedShardsAllocator：找到拥有最少分片个数的节点。

# 核心reroute实现
reroute中主要实现两种allocator：
· gatewayAllocator，用于分配现实已存在的分片，从磁盘中找到它们；
· shardsAllocator，用于平衡分片在节点间的分布。
## 从gateway到allocation流程的转换
两者之间没有明显的界限，gateway 的最后一步执行 reroute，等待这个函数返回，然后打印gateway选举结果的日志，集群完全重启时，reroute向各节点发起的询问shard级元数据的操作基本还没执行完，因此一般只有少数主分片被选举完了，gateway流程的结束只是集群级和索引级的元数据已选举完毕，主分片的选举正在进行中。

# 思考
· 请求分片信息的fetchData请求效率低，可以借鉴HDFS的上报块状态流程。
· 不需要等所有主分片都分配完才执行副分片的分配。每个分片有自己的分配流程。
· 不需要等所有分片都分配完才执行recovery流程。
· 主分片不需要等副分片分配成功才进入主分片的recovery，主副分片有自己的recovery流程。

