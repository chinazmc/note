#es 

Cluster模块封装了在集群层面要执行的任务。例如，把分片分配给节点属于集群层面的工作，在节点间迁移分片以保持数据均衡，集群健康、集群级元信息管理，以及节点管理都属于集群层面工作。本章重点论述集群任务的执行，以及集群状态的下发过程。分片分配和节点管理等单独讨论更合适一些。在_cluster/health API中看到的number_of_pending_tasks（任务数量）就是等待执行的“集群任务”的任务数量，通过_cat/pending_tasks API可以列出具体的任务列表。本章介绍主节点都会执行哪些任务，以及任务的执行细节。这些任务由主节点执行，如果其他节点产生某些事件涉及集群层面的变更，则它需要向主节点发送一个 RPC 请求，然后由主节点执行集群任务。例如，在数据写入过程中，主分片写副分片失败，它会向主节点发送一个RPC请求，将副分片从同步分片列表中移除。集群任务执行完毕，可能会产生新的集群状态。如果产生新的集群状态，则主节点会把它广播到其他节点。主节点和其他节点的通信使用最广泛的方式，就是通过下发集群状态让从节点执行相应的处理。控制信息、变更信息都存储在集群状态中。

# 集群状态
集群状态在ES中封装为ClusterState类。可以通过_cluster/state API来获取集群状态。
curl -X GET ＂localhost:9200/_cluster/state＂
响应信息中提供了集群名称、集群状态的总压缩大小（下发到数据节点时是被压缩的）和集群状态本身，请求时可以通过设置过滤器来获取特定内容。

默认情况下，协调节点在收到这个请求后会把请求路由到主节点执行，确保获取最新的集群状态。可以通过在请求中添加local=true参数，让接收请求的节点返回本地的集群状态。例如，在排查问题时如果怀疑节点的集群状态是否最新，则可以用这种方式来验证。

集群状态返回的信息比较多，为了节省篇幅，摘取关键信息如下。
由于集群状态需要频繁下发，而且内容较多，从ES 2.0版本开始，主节点发布集群信息时支持在相邻的两个版本号之间只发送增量内容。

# 以下 有空在看
