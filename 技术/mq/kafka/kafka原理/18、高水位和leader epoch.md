你好，我是胡夕。今天我要和你分享的主题是：Kafka 中的高水位和 Leader Epoch 机制。

你可能听说过高水位（High Watermark），但不一定耳闻过 Leader Epoch。前者是 Kafka 中非常重要的概念，而后者是社区在 0.11 版本中新推出的，主要是为了弥补高水位机制的一些缺陷。鉴于高水位机制在 Kafka 中举足轻重，而且深受各路面试官的喜爱，今天我们就来重点说说高水位。当然，我们也会花一部分时间来讨论 Leader Epoch 以及它的角色定位。

## 什么是高水位？

首先，我们要明确一下基本的定义：什么是高水位？或者说什么是水位？水位一词多用于流式处理领域，比如，Spark Streaming 或 Flink 框架中都有水位的概念。教科书中关于水位的经典定义通常是这样的：

在时刻 T，任意创建时间（Event Time）为 T’，且 T’≤T 的所有事件都已经到达或被观测到，那么 T 就被定义为水位。

“Streaming System”一书则是这样表述水位的：

水位是一个单调增加且表征最早未完成工作（oldest work not yet completed）的时间戳。

为了帮助你更好地理解水位，我借助这本书里的一张图来说明一下。

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430802694-3a358f7a-156c-43c8-bc6e-6aa8c8ed36f0.png)

图中标注“Completed”的蓝色部分代表已完成的工作，标注“In-Flight”的红色部分代表正在进行中的工作，两者的边界就是水位线。

在 Kafka 的世界中，水位的概念有一点不同。Kafka 的水位不是时间戳，更与时间无关。它是和位置信息绑定的，具体来说，它是用消息位移来表征的。另外，Kafka 源码使用的表述是高水位，因此，今天我也会统一使用“高水位”或它的缩写 HW 来进行讨论。值得注意的是，Kafka 中也有低水位（Low Watermark），它是与 Kafka 删除消息相关联的概念，与今天我们要讨论的内容没有太多联系，我就不展开讲了。

## 高水位的作用

在 Kafka 中，高水位的作用主要有 2 个。

1. 定义消息可见性，即用来标识分区下的哪些消息是可以被消费者消费的。
2. 帮助 Kafka 完成副本同步。

下面这张图展示了多个与高水位相关的 Kafka 术语。我来详细解释一下图中的内容，同时澄清一些常见的误区。

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430802554-94c522a0-f03d-4ead-a848-e71fed31a12d.png)

我们假设这是某个分区 Leader 副本的高水位图。首先，请你注意图中的“已提交消息”和“未提交消息”。我们之前在专栏[第 11 讲](https://time.geekbang.org/column/article/102931)谈到 Kafka 持久性保障的时候，特意对两者进行了区分。现在，我借用高水位再次强调一下。在分区高水位以下的消息被认为是已提交消息，反之就是未提交消息。消费者只能消费已提交消息，即图中位移小于 8 的所有消息。注意，这里我们不讨论 Kafka 事务，因为事务机制会影响消费者所能看到的消息的范围，它不只是简单依赖高水位来判断。它依靠一个名为 LSO（Log Stable Offset）的位移值来判断事务型消费者的可见性。

另外，需要关注的是，**位移值等于高水位的消息也属于未提交消息。也就是说，高水位上的消息是不能被消费者消费的**。

图中还有一个日志末端位移的概念，即 Log End Offset，简写是 LEO。它表示副本写入下一条消息的位移值。注意，数字 15 所在的方框是虚线，这就说明，这个副本当前只有 15 条消息，位移值是从 0 到 14，下一条新消息的位移是 15。显然，介于高水位和 LEO 之间的消息就属于未提交消息。这也从侧面告诉了我们一个重要的事实，那就是：**同一个副本对象，其高水位值不会大于 LEO 值**。

**高水位和 LEO 是副本对象的两个重要属性**。Kafka 所有副本都有对应的高水位和 LEO 值，而不仅仅是 Leader 副本。只不过 Leader 副本比较特殊，Kafka 使用 Leader 副本的高水位来定义所在分区的高水位。换句话说，**分区的高水位就是其 Leader 副本的高水位**。

## 高水位更新机制

现在，我们知道了每个副本对象都保存了一组高水位值和 LEO 值，但实际上，在 Leader 副本所在的 Broker 上，还保存了其他 Follower 副本的 LEO 值。我们一起来看看下面这张图。

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430802683-a8b90dc3-c198-440b-a51f-4502566cde0e.png)

在这张图中，我们可以看到，Broker 0 上保存了某分区的 Leader 副本和所有 Follower 副本的 LEO 值，而 Broker 1 上仅仅保存了该分区的某个 Follower 副本。Kafka 把 Broker 0 上保存的这些 Follower 副本又称为**远程副本**（Remote Replica）。Kafka 副本机制在运行过程中，会更新 Broker 1 上 Follower 副本的高水位和 LEO 值，同时也会更新 Broker 0 上 Leader 副本的高水位和 LEO 以及所有远程副本的 LEO，但它不会更新远程副本的高水位值，也就是我在图中标记为灰色的部分。

为什么要在 Broker 0 上保存这些远程副本呢？其实，它们的主要作用是，**帮助 Leader 副本确定其高水位，也就是分区高水位**。

为了帮助你更好地记忆这些值被更新的时机，我做了一张表格。只有搞清楚了更新机制，我们才能开始讨论 Kafka 副本机制的原理，以及它是如何使用高水位来执行副本消息同步的。

![](https://cdn.nlark.com/yuque/0/2021/jpeg/2725209/1633430802547-c8cdaaa3-69d8-414c-ae4c-c3b4bf0c5da8.jpeg)

在这里，我稍微解释一下，什么叫与 Leader 副本保持同步。判断的条件有两个。

1. **该远程 Follower 副本在 ISR 中。**
2. **该远程 Follower 副本 LEO 值落后于 Leader 副本 LEO 值的时间，不超过 Broker 端参数 replica.lag.time.max.ms 的值。如果使用默认值的话，就是不超过 10 秒。**

乍一看，这两个条件好像是一回事，因为目前某个副本能否进入 ISR 就是靠第 2 个条件判断的。但有些时候，会发生这样的情况：即 Follower 副本已经“追上”了 Leader 的进度，却不在 ISR 中，比如某个刚刚重启回来的副本。如果 Kafka 只判断第 1 个条件的话，就可能出现某些副本具备了“进入 ISR”的资格，但却尚未进入到 ISR 中的情况。此时，分区高水位值就可能超过 ISR 中副本 LEO，而高水位 > LEO 的情形是不被允许的。

下面，我们分别从 Leader 副本和 Follower 副本两个维度，来总结一下高水位和 LEO 的更新机制。

**Leader 副本**

处理生产者请求的逻辑如下：

1. 写入消息到本地磁盘。
2. 更新分区高水位值。  
    i. 获取 Leader 副本所在 Broker 端保存的所有远程副本 LEO 值{LEO-1，LEO-2，……，LEO-n}。  
    iii. 更新 currentHW = min( LEO-1，LEO-2，……，LEO-n)。

处理 Follower 副本拉取消息的逻辑如下：

1. 读取磁盘（或页缓存）中的消息数据。
2. 使用 Follower 副本发送请求中的位移值更新远程副本 LEO 值。
3. 更新分区高水位值（具体步骤与处理生产者请求的步骤相同）。

**Follower 副本**

从 Leader 拉取消息的处理逻辑如下：

1. 写入消息到本地磁盘。
2. 更新 LEO 值。
3. 更新高水位值。  
    i. 获取 Leader 发送的高水位值：currentHW。  
    ii. 获取步骤 2 中更新过的 LEO 值：currentLEO。  
    iii. 更新高水位为 min(currentHW, currentLEO)。

## 副本同步机制解析

搞清楚了这些值的更新机制之后，我来举一个实际的例子，说明一下 Kafka 副本同步的全流程。该例子使用一个单分区且有两个副本的主题。

当生产者发送一条消息时，Leader 和 Follower 副本对应的高水位是怎么被更新的呢？我给出了一些图片，我们一一来看。

首先是初始状态。下面这张图中的 remote LEO 就是刚才的远程副本的 LEO 值。在初始状态时，所有值都是 0。

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430802501-cc6cca14-615a-4da2-b64c-e229a955680b.png)

当生产者给主题分区发送一条消息后，状态变更为：

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430803160-8a0e9654-1af2-480f-a4ea-1a5354f3e917.png)

此时，Leader 副本成功将消息写入了本地磁盘，故 LEO 值被更新为 1。

Follower 再次尝试从 Leader 拉取消息。和之前不同的是，这次有消息可以拉取了，因此状态进一步变更为：

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430803260-efa39c3c-fabf-4da5-b179-c95005d09b31.png)

这时，Follower 副本也成功地更新 LEO 为 1。此时，Leader 和 Follower 副本的 LEO 都是 1，但各自的高水位依然是 0，还没有被更新。**它们需要在下一轮的拉取中被更新**，如下图所示：

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430803583-10810c51-3db1-419b-a9b5-2fb8e1a6b021.png)

在新一轮的拉取请求中，由于位移值是 0 的消息已经拉取成功，因此 Follower 副本这次请求拉取的是位移值 =1 的消息。Leader 副本接收到此请求后，更新远程副本 LEO 为 1，然后更新 Leader 高水位为 1。做完这些之后，它会将当前已更新过的高水位值 1 发送给 Follower 副本。Follower 副本接收到以后，也将自己的高水位值更新成 1。至此，一次完整的消息同步周期就结束了。事实上，Kafka 就是利用这样的机制，实现了 Leader 和 Follower 副本之间的同步。

## 问题分析

KIP-101中列举了两个场景，描述了在没有提出leader epoch的情况下，会出现两种数据不一致的问题。下面逐一分析这两个场景。

**场景一：日志丢失问题**

[![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1634450421554-1dedd1c8-e0cc-409d-a94b-dc109d7bad43.png)](https://t1mek1ller.github.io/images/kafka_leader_epoch_error_1.jpg)

首先是考虑第1步中的状态可能发生吗？是可能的:

1. 副本B作为leader收到producer的m2消息并写入本地文件，等待副本A拉取。
2. 副本A发起消息拉取请求，请求中携带自己的最新的日志offset（LEO=1），B收到后更新自己的HW为1，并将HW=1的信息以及消息m2返回给A。
3. A收到拉取结果后更新本地的HW为1，并将m2写入本地文件。发起新一轮拉取请求（LEO=2），B收到A拉取请求后更新自己的HW为2，没有新数据只将HW=2的信息返回给A，并且回复给producer写入成功。此处的状态就是图中第一步的状态。

此时，如果没有异常，A会收到B的回复，得知目前的HW为2，然后更新自身的HW为2。但在第2步A重启了，没有来得及收到B的回复，此时B仍然是leader。**A重启之后会以HW为标准截断自己的日志，因为A作为follower不知道多出的日志是否是被提交过的，防止数据不一致从而截断多余的数据并尝试从leader那里重新同步**

第3步，B崩溃了，min.isr设置的是1，所以zookeeper会从ISRs中再选择一个作为leader，也就是A，但是A的数据不是完整的，从而出现了数据丢失现象。

问题在哪里？在于A重启之后以HW为标准截断了多余的日志。不截断行不行？不行，因为这个日志可能没被提交过（也就是没有被ISRs中的所有节点写入过），如果保留会导致日志错乱。根本原因KIP-101也提到了：

So the essence of this problem is the follower takes an extra round of RPC to update its high watermark.

**原因是follower需要额外一次RPC请求才能得到leader的HW。所以不能直接根据自身的HW截断日志。**

**场景二：日志错乱问题**

[![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1634450421535-308f7a8e-f4f5-4df9-836d-2a65a49719f6.png)](https://t1mek1ller.github.io/images/kafka_leader_epoch_error_2.jpg)

在分析日志错乱的问题之前，我们需要了解到kafka的副本可靠性保证有一个前提：在ISRs中至少有一个节点。如果节点均宕机的情况下，是不保证可靠性的，在这种情况会出现数据丢失，数据丢失是可接受的。这里我们分析的问题比数据丢失更加槽糕，会引发日志错乱甚至导致整个系统异常，而这是不可接受的。

首先考虑第1步中的状态可能发生吗？是可能的：

1. A和B均为ISRs中的节点。副本A作为leader，收到producer的消息m2的请求后写入PageCache并在某个时刻刷新到本地磁盘。
2. **副本B拉取到m2后写入PageCage后（尚未刷盘）**再次去A中拉取新消息并告知A自己的LEO=2，A收到更新自己的HW为1并回复给producer成功。
3. 此时A和B同时宕机，**B的m2由于尚未刷盘，所以m2消息丢失**。此时的状态就是第1步的状态。

第2步，由于A和B均宕机，而min.isr=1并且unclean.leader.election.enable=true（关闭unclean选择策略），所以Kafka会等到第一个ISRs中的节点恢复并选为leader，这里不幸的是B被选为leader，而且还接收到producer发来的新消息m3。**注意，这里丢失m2消息是可接受的，毕竟所有节点都宕机了。**

第3步，A恢复重启后发现自己是follower，而且HW为2，并没有多余的数据需要截断，所以开始和B进行新一轮的同步。但此时A和B均没有意识到，offset为1的消息不一致了。

问题在哪里？在于日志的写入是异步的，上面也提到Kafka的副本策略的一个设计是消息的持久化是异步的，这就会导致在场景二的情况下被选出的leader不一定包含所有数据，从而引发日志错乱的问题。

这里额外引出一个问题，ISRs中选出的leader一定是安全（包含所有已提交数据）的吗？是的，除非ISRs中的节点全部宕机，全部宕机的情况下会出现数据丢失。

## Leader Epoch的引入

为了解决上面提出的两个场景存在的问题，我们可以分析下产生这两个场景的原因是否有什么共性。

场景一提到因为follower的HW更新有延时，所以错误的截断了已经提交了的日志。场景二提到因为异步刷盘的策略，全崩溃的情况下选出的leader并不一定包含所有已提交的日志，而follower还是以HW为准，错误的判断了自身日志的合法性。所以，不论是场景一还是场景二，**根本原因是follower的HW是不可靠的**。

其实，如果熟悉raft的话，应该已经发现上面分析的场景和raft中的日志恢复很类似，raft中的follower是可能和leader的日志不一致的，这个时候会以leader的日志为准进行日志恢复。而raft中的日志恢复很重要的一点是follower根据leader任期号进行日志比对，快速进行日志恢复，follower需要判断新旧leader的日志，以最新leader的数据为准。

这里的leader epoch和raft中的任期号的概念很类似，每次重新选择leader的时候，用一个严格单调递增的id来标志，可以让所有follower意识到leader的变化。而follower也不再以HW为准，每次奔溃重启后都需要去leader那边确认下当前leader的日志是从哪个offset开始的。

KIP-101引入如下概念：

- Leader Epoch: leader纪元，单调递增的int值，每条消息都需要存储所属哪个纪元。
- Leader Epoch Start Offset: 新leader的第一个日志偏移，同时也标志了旧leader的最后一个日志偏移。
- Leader Epoch Sequence File: Leader Epoch以及Leader Epoch Start Offset的变化记录，每个节点都需要存储。
- Leader Epoch Request: 由follower请求leader，得到请求纪元的最大的偏移量。如果请求的纪元就是当前leader的纪元的话，leader会返回自己的LEO，否则返回下一个纪元的Leader Epoch Start Offset。follow会用此请求返回的偏移量来截断日志。

我们再看下引入Leader Epoch之后是如何解决上面的两个场景的。

**场景一**：

[![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1634450421508-62ea77bd-cd5a-4a36-9120-cde1e59e0a38.png)](https://t1mek1ller.github.io/images/kafka_leader_epoch_solve_1.jpg)

这里的关键点在于副本A重启后作为follower，不是忙着以HW为准截断自己的日志，而是先发起LeaderEpochRequest询问副本B第0代的最新的偏移量是多少，副本B会返回自己的LEO为2给副本A，A此时就知道消息m2不能被截断，所以m2得到了保留。当A选为leader的时候就保留了所有已提交的日志，日志丢失的问题得到解决。

有人可能会有疑问，如果发起LeaderEpochRequest的时候就已经挂了怎么办？这种场景下，不会出现日志丢失，因为副本A被选为leader后不会截断自己的日志，日志截断只会发生在follower身上。

**场景二**：

[![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1634450421504-a3aa9ef5-ed54-43c1-8dd9-a47f15c8d61d.png)](https://t1mek1ller.github.io/images/kafka_leader_epoch_solve_2.jpg)

这里的关键点还是在第3步，副本A重启作为follower的第一步还是需要发起LeaderEpochRequest询问leader当前第0代最新的偏移量是多少，由于副本B已经经过换代，所以会返回给A第1代的起始偏移（也就是1），A发现冲突后会截断自己偏移量为1的日志，并重新开始和leader同步。副本A和副本B的日志达到了一致，解决了日志错乱。

## 总结

通过以上分析，我们终于可以回答本文最开始提出的问题：

为什么Kafka需要Leader Epoch？

**Kafka通过Leader Epoch来解决follower日志恢复过程中潜在的数据不一致问题。**

另外，我们从分析中还可以有一些额外的收获，那就是如何保证kafka高可用？

- 关闭unclean策略，必须等到ISRs中的节点恢复
- 设置ISRs的最小大小，设置的越多越可靠（当然性能也会相应损失）

总之，由于极其复杂的交互和部分失败的存在，设计一个具有容错性、一致性、扩展性分布式系统是非常不容易的。

## 小结

今天，我向你详细地介绍了 Kafka 的高水位机制以及 Leader Epoch 机制。高水位在界定 Kafka 消息对外可见性以及实现副本机制等方面起到了非常重要的作用，但其设计上的缺陷给 Kafka 留下了很多数据丢失或数据不一致的潜在风险。为此，社区引入了 Leader Epoch 机制，尝试规避掉这类风险。事实证明，它的效果不错，在 0.11 版本之后，关于副本数据不一致性方面的 Bug 的确减少了很多。如果你想深入学习 Kafka 的内部原理，今天的这些内容是非常值得你好好琢磨并熟练掌握的。

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1633430804127-e3789a2f-f77d-4b45-b223-ba572b79ba1a.png)