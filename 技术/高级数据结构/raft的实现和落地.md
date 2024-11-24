
# Raft算法原理

# 概述

本文是博客解析raft算法及etcd raft库实现的系列三篇文章之一，之所以详细结合etcd实现解析raft算法原理及实现，因为etcd的raft实现是最接近论文本身的，结合论文原理一起阅读十分酸爽。这个系列文章的索引如下：

-   [Raft算法原理](https://www.codedump.info/post/20180921-raft/)
-   [etcd Raft库解析](https://www.codedump.info/post/20180922-etcd-raft/)
-   [Etcd存储的实现](https://www.codedump.info/post/20181125-etcd-server/)
-   [Etcd Raft库的工程化实现](https://www.codedump.info/post/20210515-raft/)

另外，我个人还针对etcd 3.1.10版本的raft相关代码实现做了一些代码的注释笔记，地址在此：[etcd-3.1.10-codedump](https://github.com/lichuang/etcd-3.1.10-codedump)。

# 简介

关于Raft算法，有两篇经典的论文，一篇是《In search of an Understandable Consensus Algorithm》，这是作者最开始讲述Raft算法原理的论文，但是这篇论文太简单了，很多算法的细节没有涉及到。更详细的论文是《Consensus: Bridging Theory and Practice》，除了包括第一篇论文的内容以外，还加上了很多细节的描述。在我阅读完etcd raft算法库的实现之后，发现这个库的代码基本就是按照后一篇论文来写的，甚至有部分测试用例的注释里也写明了是针对这篇论文的某一个小节的情况做验证。

这篇文章做为我后续分析etcd raft算法的前导文章，将结合后一篇论文加上一些自己的演绎和理解来讲解Raft算法的原理。

# 算法的基本流程

## Raft算法概述

Raft算法由leader节点来处理一致性问题。leader节点接收来自客户端的请求日志数据，然后同步到集群中其它节点进行复制，当日志已经同步到超过半数以上节点的时候，leader节点再通知集群中其它节点哪些日志已经被复制成功，可以提交到raft状态机中执行。

通过以上方式，Raft算法将要解决的一致性问题分为了以下几个子问题。

-   leader选举：集群中必须存在一个leader节点。
-   日志复制：leader节点接收来自客户端的请求然后将这些请求序列化成日志数据再同步到集群中其它节点。
-   安全性：如果某个节点已经将一条提交过的数据输入raft状态机执行了，那么其它节点不可能再将相同索引 的另一条日志数据输入到raft状态机中执行。

Raft算法需要一直保持的几个属性。

-   选举安全性（Election Safety）：在一个任期内只能存在最多一个leader节点。
-   Leader节点上的日志为只添加（Leader Append-Only）：leader节点永远不会删除或者覆盖本节点上面的日志数据，leader节点上写日志的操作只可能是添加操作。
-   日志匹配性（Log Matching）：如果两个节点上的日志，在日志的某个索引上的日志数据其对应的任期号相同，那么在两个节点在这条日志之前的日志数据完全匹配。
-   leader完备性（Leader Completeness）：如果一条日志在某个任期被提交，那么这条日志数据在leader节点上更高任期号的日志数据中都存在。
-   状态机安全性（State Machine Safety）：如果某个节点已经将一条提交过的数据输入raft状态机执行了，那么其它节点不可能再将相同索引的另一条日志数据输入到raft状态机中执行。

![raft-propertities](https://www.codedump.info/media/imgs/20180921-raft/raft-propertities.png "raft-propertities")

## Raft算法基础

在Raft算法中，一个集群里面的所有节点有以下三种状态：

-   Leader：领导者，一个集群里只能存在一个Leader。
-   Follower：跟随者，follower是被动的，一个客户端的修改数据请求如果发送到Follower上面时，会首先由Follower重定向到Leader上，
-   Candidate：参与者，一个节点切换到这个状态时，将开始进行一次新的选举。

每一次开始一次新的选举时，称为一个“任期”。每个任期都有一个对应的整数与之关联，称为“任期号”，任期号用单词“Term”表示，这个值是一个严格递增的整数值。

节点的状态切换状态机如下图所示。

![raft states](https://www.codedump.info/media/imgs/20180921-raft/raft-states.jpg "raft states")

上图中标记了状态切换的6种路径，下面做一个简单介绍，后续都会展开来详细讨论。

1.  start up：起始状态，节点刚启动的时候自动进入的是follower状态。
2.  times out, starts election：follower在启动之后，将开启一个选举超时的定时器，当这个定时器到期时，将切换到candidate状态发起选举。
3.  times out, new election：进入candidate 状态之后就开始进行选举，但是如果在下一次选举超时到来之前，都还没有选出一个新的leade，那么还会保持在candidate状态重新开始一次新的选举。
4.  receives votes from majority of servers：当candidate状态的节点，收到了超过半数的节点选票，那么将切换状态成为新的leader。
5.  discovers current leader or new term：candidate状态的节点，如果收到了来自leader的消息，或者更高任期号的消息，都表示已经有leader了，将切换回到follower状态。
6.  discovers server with higher term：leader状态下如果收到来自更高任期号的消息，将切换到follower状态。这种情况大多数发生在有网络分区的状态下。

如果一个candidate在一次选举中赢得leader，那么这个节点将在这个任期中担任leader的角色。但并不是每个任期号都一定对应有一个leader的，比如上面的情况3中，可能在选举超时到来之前都没有产生一个新的leader，那么此时将递增任期号开始一次新的选举。

从以上的描述可以看出，任期号在raft算法中更像一个“逻辑时钟（logic clock）”的作用，有了这个值，集群可以发现有哪些节点的状态已经过期了。每一个节点状态中都保存一个当前任期号（current term），节点在进行通信时都会带上本节点的当前任期号。如果一个节点的当前任期号小于其他节点的当前任期号，将更新其当前任期号到最新的任期号。如果一个candidate或者leader状态的节点发现自己的当前任期号已经小于其他节点了，那么将切换到follower状态。反之，如果一个节点收到的消息中带上的发送者的任期号已经过期，将拒绝这个请求。

raft节点之间通过RPC请求来互相通信，主要有以下两类RPC请求。RequestVote RPC用于candidate状态的节点进行选举之用，而AppendEntries RPC由leader节点向其他节点复制日志数据以及同步心跳数据的。

## leader选举

现在来讲解leader选举的流程。

raft算法是使用心跳机制来触发leader选举的。

在节点刚开始启动时，初始状态是follower状态。一个follower状态的节点，只要一直收到来自leader或者candidate的正确RPC消息的话，将一直保持在follower状态。leader节点通过周期性的发送心跳请求（一般使用带有空数据的AppendEntries RPC来进行心跳）来维持着leader节点状态。每个follower同时还有一个选举超时（election timeout）定时器，如果在这个定时器超时之前都没有收到来自leader的心跳请求，那么follower将认为当前集群中没有leader了，将发起一次新的选举。

发起选举时，follower将递增它的任期号然后切换到candidate状态。然后通过向集群中其它节点发送RequestVote RPC请求来发起一次新的选举。一个节点将保持在该任期内的candidate状态下，直到以下情况之一发生。

-   该candidate节点赢得选举，即收到超过半数以上集群中其它节点的投票。
-   另一个节点成为了leader。
-   选举超时到来时没有任何一个节点成为leader。

下面来逐个分析以上几种情况。

第一种情况，如果收到了集群中半数以上节点的投票，那么此时candidate节点将成为新的leader。每个节点在一个任期中只能给一个节点投票，而且遵守“先来后到”的原则。这样就保证了，每个任期最多只有一个节点会赢得选举成为leader。但并不是每个进行选举的candidate节点都会给它投票，在后续的“选举安全性”一节中将展开讨论这个问题。当一个candidate节点赢得选举成为leader后，它将发送心跳消息给其他节点来宣告它的权威性以阻止其它节点再发起新的选举。

第二种情况，当candidate节点等待其他节点时，如果收到了来自其它节点的AppendEntries RPC请求，同时做个请求中带上的任期号不比candidate节点的小，那么说明集群中已经存在leader了，此时candidate节点将切换到follower状态；但是，如果该RPC请求的任期号比candidate节点的小，那么将拒绝该RPC请求继续保持在candidate状态。

第三种情况，一个candidate节点在选举超时到来的时候，既没有赢得也没有输掉这次选举。这种情况发生在集群节点数量为偶数个，同时有两个candidate节点进行选举，而两个节点获得的选票数量都是一样时。当选举超时到来时，如果集群中还没有一个leader存在，那么candidate节点将继续递增任期号再次发起一次新的选举。这种情况理论上可以一直无限发生下去。

为了减少第三种情况发生的概率，每个节点的选举超时时间都是随机决定的，一般在150~300毫秒之间，这样两个节点同时超时的情况就很罕见了。

以上过程用伪代码来表示如下。

```markdown
节点刚启动，进入follower状态，同时创建一个超时时间在150-300毫秒之间的选举超时定时器。

follower状态节点主循环：
  如果收到leader节点心跳：
    心跳标志位置1

  如果选举超时到期：
    没有收到leader节点心跳：
      任期号term+1，换到candidate状态。
    如果收到leader节点心跳：
      心跳标志位置空

  如果收到选举消息：
    如果当前没有给任何节点投票过 或者 消息的任期号大于当前任期号：
      投票给该节点
    否则：
      拒绝投票给该节点

candidate状态节点主循环：
  向集群中其他节点发送RequestVote请求，请求中带上当前任期号term

  收到AppendEntries消息：
    如果该消息的任期号 >= 本节点任期号term：
      说明已经有leader，切换到follower状态
    否则：
      拒绝该消息

  收到其他节点应答RequestVote消息：
    如果数量超过集群半数以上，切换到leader状态
    
  如果选举超时到期：
    term+1，进行下一次的选举
```

## 日志复制

日志复制的流程大体如下：

1.  每个客户端的请求都会被重定向发送给leader，这些请求最后都会被输入到raft算法状态机中去执行。
2.  leader在收到这些请求之后，会首先在自己的日志中添加一条新的日志条目。
3.  在本地添加完日志之后，leader将向集群中其他节点发送AppendEntries RPC请求同步这个日志条目，当这个日志条目被成功复制之后（什么是成功复制，下面会谈到），leader节点将会将这条日志输入到raft状态机中，然后应答客户端。

Raft日志的组织形式如下图所示。

![raftlog](https://www.codedump.info/media/imgs/20180921-raft/raftlog.jpg "raft log")

每个日志条目包含以下成员。

1.  index：日志索引号，即图中最上方的数字，是严格递增的。
2.  term：日志任期号，就是在每个日志条目中上方的数字，表示这条日志在哪个任期生成的。
3.  command：日志条目中对数据进行修改的操作。

一条日志如果被leader同步到集群中超过半数的节点，那么被称为“成功复制”，leader节点在收到半数以上的节点应答某条日志之后，就会提交该日志，此时日志这个日志条目就是“已被提交（committed）”。如果一条日志已被提交，那么在这条日志之前的所有日志条目也是被提交的，包括之前其他任期内的leader提交的日志。如上图中索引为7的日志条目之前的所有日志都是已被提交的日志。

以下面的图示来说明日志复制的流程。

![log-replication-1](https://www.codedump.info/media/imgs/20180921-raft/log-replication-1.png "log replication 1")

在上图中，一个请求有以下步骤。

-   1.  客户端发送SET a=1的命令到leader节点上。
-   2.  leader节点在本地添加一条日志，其对应的命令为SET a=1。这里涉及到两个索引值，committedIndex存储的最后一条提交（commit）日志的索引，appliedIndex存储的是最后一条应用到状态机中的日志索引值，一条日志只有被提交了才能应用到状态机中，因此总有 committedIndex >= appliedIndex不等式成立。在这里只是添加一条日志还并没有提交，两个索引值还指向上一条日志。
-   3.  leader节点向集群中其他节点广播AppendEntries消息，带上SET a=1命令。

![log-replication-2](https://www.codedump.info/media/imgs/20180921-raft/log-replication-2.png "log replication 2")

接下来继续看，上图中经历了以下步骤。

-   4.  收到AppendEntries请求的follower节点，同样在本地添加了一条新的日志，也还并没有提交。
-   5.  follower节点向leader节点应答AppendEntries消息。
-   6.  当leader节点收到集群半数以上节点的AppendEntries请求的应答消息时，认为SET a=1命令成功复制，可以进行提交，于是修改了本地committed日志的索引指向最新的存储SET a=1的日志，而appliedIndex还是保持着上一次的值，因为还没有应用该命令到状态机中。

![log-replication-3](https://www.codedump.info/media/imgs/20180921-raft/log-replication-3.png "log replication 3")

当这个命令提交完成了之后，命令就可以提交给应用层了。

-   7.  提交命令完成，给应用层说明这条命令已经提交。此时修改appliedIndex与committedIndex一样了。
-   8.  leader节点在下一次给follower的AppendEntries请求中，会带上当前最新的committedIndex索引值，follower收到之后同样会修改本地日志的committedIndex索引。

需要说明的是，7和8这两个操作并没有严格的先后顺序，谁在前在后都没关系。

leader上保存着已被提交的最大日志索引信息，这个索引值被称为“nextIndex”，在每次向follower节点发送的AppendEntries RPC请求中都会带上这个索引信息，这样follower节点就知道哪个日志已经被提交了，被提交的日志将会输入Raft状态机中执行。

Raft算法保持着以下两个属性，这两个属性共同作用满足前面提到的日志匹配（LogMatch）属性：

-   如果两个日志条目有相同的索引号和任期号，那么这两条日志存储的是同一个指令。
-   如果在两个不同的日志数据中，包含有相同索引和任期号的日志条目，那么在这两个不同的日志中，位于这条日志之前的日志数据是相同的。

## 新leader与follower同步数据

在正常的情况下，follower节点和leader节点的日志一直保持一致，此时AppendEntries RPC请求将不会失败。但是，当leader节点宕机时日志就可能出现不一致的情况，比如在这个leader节点宕机之前同步的数据并没有得到超过半数以上节点都复制成功了。如下图所示就是一种出现前后日志不一致的情况。

![inconsistent logs](https://www.codedump.info/media/imgs/20180921-raft/inconsistentlog.jpg "inconsistent logs")

在上图中，最上面的一排数字是日志的索引，盒子中的数据是该日志对应的任期号，左边的字母表示的是a-f这几个不同的节点。图中演示了好几种节点日志与leader节点日志不一致的情况，下面说明中以二元组<任期号，索引号>来说明各个节点的日志数据情况：

-   leader节点：<6, 10>。
-   a节点：<6,9>，缺少日志。
-   b节点：<4,4>，任期号比leader小，因此缺少日志。
-   c节点：<6,11>，任期号与leader相同，但是有比leader日志索引更大的日志，这部分日志是未提交的日志。
-   d节点：<7,12>，任期号比leader大，这部分日志是未提交的日志。
-   e节点：<4,7>，任期号与索引都比leader小，因此既缺少日志，也有未提交的日志。
-   f节点：<3,11>，任期号比leader小，所以缺少日志，而索引比leader大，这部分日志又是未提交的日志。

在Raft算法中，解决日志数据不一致的方式是Leader节点同步日志数据到follower上，覆盖follower上与leader不一致的数据。

为了解决与follower节点同步日志的问题，leader节点中存储着两个与每个follower节点日志相关的数据。

-   nextIndex存储的是下一次给该节点同步日志时的日志索引。
-   matchIndex存储的是该节点的最大日志索引。

从以上两个索引的定义可知，在follower与leader节点之间日志复制正常的情况下，nextIndex = matchIndex + 1。但是如果出现不一致的情况，则这个等式可能不成立。每个leader节点被选举出来时，将做如下初始化操作：

-   nextIndex为leader节点最后一条日志。
-   matchIndex为0。

这么做的原因在于：leader节点将从后往前探索follower节点当前存储的日志位置，而在不知道follower节点日志位置的情况下只能置空matchIndex了。

leader节点通过AppendEntries消息来与follower之间进行日志同步的，每次给follower带过去的日志就是以nextIndex来决定，其可能有两种结果：

-   如果follower节点的日志与这个值匹配，将返回成功；
-   否则将返回失败，同时带上本节点当前的最大日志ID（假设这个索引为hintIndex），方便leader节点快速定位到follower的日志位置以下一次同步正确的日志数据。

而leader节点在收到返回失败的情况下，将置`nextIndex = min(hintIndex+1,上一次append消息的索引)`，再次发出添加日志请求。

以上图的几个节点为例来说明情况。

-   初始状态下，leader节点将存储每个folower节点的nextIndex为10，matchIndex为0。因此在成为leader节点之后首次向follower节点同步日志数据时，将复制索引位置在10以后的日志数据，同时带上日志二元组<6,10>告知follower节点当前leader保存的follower日志状态。
    
-   a节点：由于节点的最大日志数据二元组是<6,9>，正好与leader发过来的日志<6,10>紧挨着，因此返回复制成功。
    

**（注：在下面解释每个follower与leader同步流程的示意图中，follower节点回应的消息，格式是<append消息的索引,当前follower当前的最大消息索引>，比如<10,4>表示接收到leader发过来的append消息索引是10，而当前该follower最大消息索引是4。）**

![leader-to-a](https://www.codedump.info/media/imgs/20180921-raft/leader-to-a.png "leader-to-a")

-   b节点：由于节点的最大日志数据二元组是<4,4>，与leader发送过来的日志数据<6,10>不匹配，将返回失败同时带上自己最后的日志索引4（即hintIndex=4），leader节点在收到该拒绝消息之后，将修改保存该节点的nextIndex为min(10,4+1)=5，所以下一次leader节点将同步从索引5到10的数据给b节点。

![leader-to-b](https://www.codedump.info/media/imgs/20180921-raft/leader-to-b.png "leader-to-b")

-   c节点：由于节点的最大日志数据二元组是<6,11>，与leader发送过来的日志数据<6,10>不匹配，由于两者任期号相同，节点C知道自己的索引11的数据需要删除。

![leader-to-c](https://www.codedump.info/media/imgs/20180921-raft/leader-to-c.png "leader-to-c")

-   d节点：由于节点的最大日志数据二元组是<7,12>，与leader发送过来的日志数据<6,10>不匹配，索引11、12的数据将被删除。

![leader-to-d](https://www.codedump.info/media/imgs/20180921-raft/leader-to-d.png "leader-to-d")

-   e节点：由于节点的最大日志数据二元组是<4,7>，与leader发送过来的日志数据<6,10>不匹配，将返回最后一个与节点数据一致的索引5给leader，于是leader从min(5+1,10)=6开始同步数据给节点e，最终e节点上索引7的数据被覆盖。

![leader-to-e](https://www.codedump.info/media/imgs/20180921-raft/leader-to-e.png "leader-to-e")

-   f节点：由于节点的最大日志数据二元组是<3,11>，与leader发送过来的日志数据<6,10>不匹配，将返回最后一个与节点数据一致的索引3给leader，于是leader从min(3+1,10)=4开始同步数据给节点f，最终f节点上索引4之后的数据被覆盖。

![leader-to-f](https://www.codedump.info/media/imgs/20180921-raft/leader-to-f.png "leader-to-f")

## 安全性

前面章节已经将leader选举以及日志同步的机制介绍了，这一小节讲解安全性相关的内容。

### 选举限制

raft算法中，并不是所有节点都能成为leader。一个节点要成为leader，需要得到集群中半数以上节点的投票，而一个节点会投票给一个节点，其中一个充分条件是：这个进行选举的节点的日志，比本节点的日志更新。之所以要求这个条件，是为了保证每个当选的节点都有当前最新的数据。为了达到这个检查日志的目的，RequestVote RPC请求中需要带上参加选举节点的日志信息，如果节点发现选举节点的日志信息并不比自己更新，将拒绝给这个节点投票。

如果判断日志的新旧？这通过对比日志的最后一个日志条目数据来决定，首先将对比条目的任期号，任期号更大的日志数据更新；如果任期号相同，那么索引号更大的数据更新。

以上处理RequestVote请求的流程伪代码表示如下。

```markdown
follower节点收到RequestVote请求：
  对比RequestVote请求中带上的最后一条日志数据：
    如果任期号比节点的最后一条数据任期号小：
      拒绝投票给该节点
    如果索引号比节点的最后一条数据索引小：
      拒绝投票给该节点
    其他情况：
      说明选举节点的日志信息比本节点更新，投票给该节点。
```

### 提交前面任期的日志条目

如果leader在写入但是还没有提交一条日志之前崩溃，那么这条没有提交的日志是否能提交？有几种情况需要考虑，如下图所示。

![pre term log](https://www.codedump.info/media/imgs/20180921-raft/preterm-log.jpeg "pre term log")

在上图中，有以下的场景变更。

-   情况a：s1是leader，index 2位置写入了数据2，该值只写在了s1，s2上，但是还没有被提交。
    
-   情况b: s1崩溃，s5成为新的leader，该节点在index 2上面提交了另外一个值3，但是这个值只写在了s5上面，并没有被提交。
    
-   情况c: s5崩溃，s1重新成为leader，这一次，index 2的值2写到了集群的大多数节点上。
    

此时可能存在以下两种情况：

-   情况d1: s1崩溃，s5重新成为leader（投票给s5的是s4，s2和s5自身），那么index 2上的值3这一次成功的写入到集群的半数以上节点之上，并成功提交。
-   情况d2: s1不崩溃，而是将index 2为2的值成功提交。

从情况d的两种场景可以看出，在index 2值为2，且已经被写入到半数以上节点的情况下，同样存在被新的leader覆盖的可能性。

由于以上的原因，对于当前任期之前任期提交的日志，并不通过判断是否已经在半数以上集群节点写入成功来作为能否提交的依据。只有当前leader任期内的日志是通过比较写入数量是否超过半数来决定是否可以提交的。

对于任期之前的日志，Raft采用的方式，是只要提交成功了当前任期的日志，那么在日志之前的日志就认为提交成功了。这也是为什么etcd-Raft代码中，在成为leader之后，需要再提交一条dummy的日志的原因–只要该日志提交成功，leader上该日志之前的日志就可以提交成功。

# 集群成员变更

在上面描述Raft基本算法流程中，都假设集群中的节点是稳定不变的。但是在某些情况下，需要手动改变集群的配置。

## 安全性

安全性是变更集群成员时首先需要考虑到的问题，任何时候都不能出现集群中存在一个以上leader的情况。为了避免出现这种情况，每次变更成员时不能一次添加或者修改超过一个节点，集群不能直接切换到新的状态，如下图所示。

![disjoint majorities](https://www.codedump.info/media/imgs/20180921-raft/disjoint-majorities.jpg "disjoint majorities")

在上图中，server 1、2、3组成的是旧集群，server 4、5是准备新加入集群的节点。注意到如果直接尝试切换到新的状态，在某些时间点里，如图中所示，由于server 1、2上的配置还是旧的集群配置，那么可能这两个节点已经选定了一个leader；而server 3、4、5又是新的配置，它们也可能选定了一个leader，而这两个leader不是同一个，这就出现了集群中存在一个以上leader的情况了。

反之，下图所示是分别往奇数个以及偶数个集群节点中添加删除单个节点的场景。

![change single server](https://www.codedump.info/media/imgs/20180921-raft/change-single-server.jpg "change single server")

可以看到，不论旧集群节点数量是奇数还是偶数个，都不会出现同时有两个超过半数以上子集群的存在，也就不可能选出超过一个leader。

raft采用将修改集群配置的命令放在日志条目中来处理，这样做的好处是：

-   可以继续沿用原来的AppendEntries命令来同步日志数据，只要把修改集群的命令做为一种特殊的命令就可以了。
-   在这个过程中，可以继续处理客户端请求。

## 可用性

### 添加新节点到集群中

添加一个新的节点到集群时，需要考虑一种情况，即新节点可能落后当前集群日志很多的情况，在这种情况下集群出现故障的概率会大大提高，如下图所示。

![add-new-server](https://www.codedump.info/media/imgs/20180921-raft/add-new-server.jpg "add-new-server")

上图中的情况a中，s1、s2、s3是原有的集群节点，这时把节点s4添加进来，而s4上又什么数据都没有。如果此时s3发生故障，在集群中原来有三个节点的情况下，本来可以容忍一个节点的失败的；但是当变成四个节点的集群时，s3和s4同时不可用整个集群就不可用了。

因此Raft算法针对这种新添加进来的节点，是如下处理的。

-   添加进来的新节点首先将不加入到集群中，而是等待数据追上集群的进度。
-   leader同步数据给新节点的流程是，划分为多个轮次，每一轮同步一部分数据，而在同步的时候，leader仍然可以写入新的数据，只要等新的轮次到来继续同步就好。

以下图来说明同步数据的流程。

![new-server-sync-logs](https://www.codedump.info/media/imgs/20180921-raft/new-server-sync-logs.jpg "new-server-sync-logs")

如上图中，划分为多个轮次来同步数据。比如，在第一轮同步数据时，leader的最大数据索引为10，那么第一轮就同步10之前的数据。而在同步第一轮数据的同时，leader还能继续接收新的数据，假设当同步第一轮完毕时，最大数据索引变成了20，那么第二轮将继续同步从10到20的数据。以此类推。

这个同步的轮次并不能一直持续下去，一般会有一个限制的轮次数量，比如最多同步10轮。

### 删除当前集群的leader节点

当需要下线当前集群的leader节点时，leader节点将发出一个变更节点配置的命令，只有在该命令被提交之后，原先的leader节点才下线，然后集群会自然有一个节点选举超时而进行新的一轮选举。

### 处理移出集群的节点

如果某个节点在一次配置更新之后，被移出了新的集群，但是这个节点又不知道这个情况，那么按照前面描述的Raft算法流程来说，它应该在选举超时之后，将任期号递增1，发起一次新的选举。虽然最终这个节点不会赢得选举，但是毕竟对集群运行的状态造成了干扰。而且如果这个节点一直不下线，那么上面这个发起新选举的流程就会一直持续下去。

为了解决这个问题，Raft引入了一个成为“PreVote”的流程，在这个流程中，如果一个节点要发起一次新的选举，那么首先会广播给集群中的其它所有节点，询问下当前该节点上的日志是否足以赢下选举。只有在这个PreVote阶段赢得超过半数节点肯定的情况下，才真正发起一次新的选举。

然而，PreVote并不能解决所有的问题，因为很有可能该被移除节点上的日志也是最新的。

由于以上的原因，所以不能完全依靠判断日志的方式来决定是否允许一个节点发起新一轮的选举。

Raft采用了另一种机制。如果leader节点一直保持着与其它节点的心跳消息，那么就认为leader节点是存活的，此时不允许发起一轮新的选举。这样follower节点处理RequestVote请求时，就需要加上判断，除了判断请求进行选举的节点日志是否最新以外，如果当前在一段时间内还收到过来自leader节点的心跳消息，那么也不允许发起新的选举。然而这种情况与前面描述的leader迁移的情况相悖，在leader迁移时是强制要求发起新的选举的，因此RequestVote请求的处理还要加上这种情况的判断。

总结来说，RequestVote请求的处理逻辑大致是这样的。

```markdown
follower处理RequestVote请求：
  如果请求节点的日志不是最新的：
    拒绝该请求，返回

  如果此时是leader迁移的情况：
    接收该请求，返回

  如果最近一段时间还有收到来自leader节点的心跳消息：
    拒绝该请求，返回

  接收该请求    
```

# 日志压缩

日志数据如果不进行压缩处理掉的话，会一直增长下去。为此Raft使用快照数据来进行日志压缩，比如针对键值a的几次操作日志a=1、删除a、a=3最后可以被压缩成为最后的结果数据即a=3。

快照数据和日志数据的组织形式如下图。

![snapshot](https://www.codedump.info/media/imgs/20180921-raft/snapshot.png "snapshot")

在上图中：

-   未压缩日志前，日志数据保存到了<3,5>的位置，而在<2,3>的位置之前的数据都已经进行提交了，所以可以针对这部分数据进行压缩。
-   压缩日志之后，快照文件中存放了几个值：压缩时最后一条日志的二元数据是<2,3>，而针对a的几次操作最后的值为a=3，b的值为2。

# 杂项

## 高效处理只读请求

前面已经提到过，处理一个命令时，需要经历以下流程：leader向集群中其它节点广播日志，在日志被超过半数节点应答之后，leader提交该日志，最后才应答客户端。这样的流程对于一个只读请求而言太久了，而且还涉及到日志落盘的操作，对于只读请求而言这些操作是不必要的。

但是如果不经过上面的流程，leader节点在收到一个只读请求时就直接将本节点上保存的数据应答客户端，也是不安全的，因为这可能返回已经过期的数据。一方面leader节点可能已经发生了变化，只是这个节点并不知道；另一方面可能数据也发生了改变。返回过期的数据不符合一致性要求，因此这样的做法也是不允许的。

Raft中针对只读请求是这样做处理的。

1.  leader节点需要有当前已提交日志的信息。在前面提到过不能提交前面任期的日志条目，因此一个新leader产生之后，需要提交一条空日志，这样来确保上一个任期内的日志全部提交。
2.  leader节点保存该只读请求到来时的commit日志索引为readIndex，
3.  leader需要确认自己当前还是集群的leader，因为可能会由于有网络分区的原因导致leader已经被隔离出集群而不自知。为了达到这个目的，leader节点将广播一个heartbeat心跳消息给集群中其它节点，当收到半数以上节点的应答时，leader节点知道自己当前还是leader，同时readIndex索引也是当前集群日志提交的最大索引。


# etcd Raft库解析
# 序言

今年初开始学习了解Raft协议，论文读下来之后还是决定结合一个成熟的代码进行更深的理解。etcd做为一个非常成熟的作品，其Raft库实现也非常精妙，屏蔽了网络、存储等模块，提供接口由上层应用者来实现。

本篇文章解析etcd的Raft库实现，基于etcd 3.1.10版本。etcd的Raft库，位于其代码目录的Raft中。

我自己也单独将3.1.10的代码拉出了一个专门添加了我阅读代码注释的版本，目前Raft这部分基本都做了注释，见： [https://github.com/lichuang/etcd-3.1.10-codedump](https://github.com/lichuang/etcd-3.1.10-codedump)

以下在介绍的时候，可能会混用中文和英文术语，这里先列举出来：

![[Pasted image 20230423102345.png]]

# 输入及输出

既然做为一个库使用，就有其确定的输入和输出接口，先来了解这部分再进行后续的展开讨论。

作为一个一致性算法的库，不难想象使用的一般场景是这样的：

1.  应用层接收到新的写入数据请求，向该算法库写入一个数据。
2.  算法库返回是否写入成功。
3.  应用层根据写入结果进行下一步的操作。

然而，Raft库却相对而言更复杂一些，因为还有以下的问题存在：

1.  写入的数据，可能是集群状态变更的数据，Raft库在执行写入这类数据之后，需要返回新的状态给应用层。
2.  Raft库中的数据不可能一直以日志的形式存在，这样会导致数据越来越大，所以有可能被压缩成快照（snapshot）的数据形式，这种情况下也需要返回这部分快照数据。
3.  由于etcd的Raft库不包括持久化数据存储相关的模块，而是由应用层自己来做实现，所以也需要返回在某次写入成功之后，哪些数据可以进行持久化保存了。
4.  同样的，etcd的Raft库也不自己实现网络传输，所以同样需要返回哪些数据需要进行网络传输给集群中的其他节点。

以上的这些，集中在raft/node.go的Ready结构体中，其包括以下成员：

![[Pasted image 20230423102400.png]]

以上的成员说明，最开始看不一定能理解其含义和用法，不过在后续会慢慢展开讨论。

根据上面的分析，应用层在写入一段数据之后，Raft库将返回这样一个Ready结构体，其中可能某些字段是空的，毕竟不是每次改动都会导致Ready结构体中的成员都发生变化，此时使用者就需要根据情况，取出其中不为空的成员进行操作了。

在etcd项目中，也提供了使用Raft库的demo例子，在contrib/raftexample目录中，这里简单的演示了一下如何根据这个raft库实现一个简单的KV存储服务器，下面根据这里的代码结合着上面的Ready结构体，来分析如何使用etcd的Raft库。

raft库对外提供一个Node的interface，其实现有raft/node.go中的node结构体实现，这也是应用层唯一需要与这个raft库直接打交道的结构体，简单的来看看Node接口需要实现的函数：

![[Pasted image 20230423102418.png]]
这里大部分函数在这个demo中不需要进行关心，我们只看如何对接Ready结构体就好了。

raftexample中，首先在main.go中创建了两个channel：

1.  proposeC：用于提交写入的数据。
2.  confChangeC：用于提交配置改动数据。

然后分别启动如下核心的协程：

1.  启动HTTP服务器，用于接收用户的请求数据，最终会将用户请求的数据写入前面的proposeC/confChangeC channel中。
2.  启动raftNode结构体，该结构体中有上面提到的raft/node.go中的node结构体，也就是通过该结构体实现的Node接口与raft库进行交互。同时，raftNode还会启动协程监听前面的两个channel，收到数据之后通过Node接口的函数调用raft库对应的接口。

以上的交互流程就很清楚了，HTTP服务负责接收用户数据，再写入到两个核心channel中，而raftNode负责监听这两个channel：

1.  如果收到proposeC channel的消息，说明有数据提交，则调用Node.Propose函数进行数据的提交。
2.  如果收到confChangeC channel的消息，说明有配置变更，则调用Node.ProposeConfChange函数进行配置变更。
3.  设置一个定时器tick，每次定时器到时时，调用Node.Tick函数。
4.  监听Node.Ready函数返回的Ready结构体channel，有数据变更时根据Ready结构体的不同数据类型进行相应的操作，完成了之后需要调用Node.Advance函数进行收尾。

将以上流程用伪代码实现如下：

```Go
// HTTP server
HttpServer主循环:
  接收用户提交的数据：
    如果是PUT请求：
      将数据写入到proposeC中
    如果是POST请求：
      将配置变更数据写入到confChangeC中

// raft Node
raftNode结构体主循环：
  如果proposeC中有数据写入：
    调用node.Propose向raft库提交数据
  如果confChangeC中有数据写入：
    调用node.Node.ProposeConfChange向raft库提交配置变更数据
  如果tick定时器到期：
    调用node.Tick函数进行raft库的定时操作
  如果node.Ready()函数返回的Ready结构体channel有数据变更：
    依次处理Ready结构体中各成员数据
    处理完毕之后调用node.Advance函数进行收尾处理
```

到了这里，已经对raft的使用有一个基本的概念了，即通过node结构体实现的Node接口与raft库进行交互，涉及数据变更的核心数据结构就是Ready结构体，接下来可以进一步来分析该库的实现了。

# raft库代码结构及核心数据结构

现在可以来看看raft库的代码组织了。

前面已经看到了raft/node.go文件中，提供出去的是Node接口及其实现node结构体，这是外界与raft库打交道的唯一接口，除此之外该路径下的其他文件并不直接与外界打交道。

接着是raft算法的实现文件，raft/raft.go文件，其中包含两个核心数据结构：

1.  Config：与raft算法相关的配置参数都包装在该结构体中。从这个结构体的命名是大写字母开头，就可以知道是提供给外部调用的。
2.  raft：具体实现raft算法的结构体。

除去以上两个文件之外，raft目录下的其他文件，都是间接给raft结构体服务的，下面的表格做一个总结和罗列：

![[Pasted image 20230423102438.png]]

## raft库日志存储相关结构

### unstable

顾名思义，unstable数据结构用于还没有被用户层持久化的数据，而其中又包括两部分，如下图所示。

![unstable](https://www.codedump.info/media/imgs/20180922-etcd-raft/unstable.png "unstable")

在上图中，前半部分是快照数据，而后半部分是日志条目组成的数组entries，另外unstable.offset成员保存的是entries数组中的第一条数据在raft日志中的索引，即第i条entries数组数据在raft日志中的索引为i + unstable.offset。

这两个部分，并不同时存在，同一时间只有一个部分存在。其中，快照数据仅当当前节点在接收从leader发送过来的快照数据时存在，在接收快照数据的时候，entries数组中是没有数据的；除了这种情况之外，就只会存在entries数组的数据了。因此，当接收完毕快照数据进入正常的接收日志流程时，快照数据将被置空。

理解了以上unstable中数据的分布情况，就不难理解unstable各个函数成员的作用了，下面逐一进行解释。

-   maybeFirstIndex：返回unstable数据的第一条数据索引。因为只有快照数据在最前面，因此这个函数只有当快照数据存在的时候才能拿到第一条数据索引，其他的情况下已经拿不到了。
-   maybeLastIndex：返回最后一条数据的索引。因为是entries数据在后，而快照数据在前，所以取最后一条数据索引是从entries开始查，查不到的情况下才查快照数据。
-   maybeTerm：这个函数根据传入的日志数据索引，得到这个日志对应的任期号。前面已经提过，unstable.offset是快照数据和entries数组的分界线，因为在这个函数中，会区分传入的参数与offset的大小关系，小于offset的情况下在快照数据中查询，否则就在entries数组中查询了。
-   stableTo：该函数传入一个索引号i和任期号t，表示应用层已经将这个索引之前的数据进行持久化了，此时unstable要做的事情就是在自己的数据中查询，只有在满足任期号相同以及i大于等于offset的情况下，可以将entries中的数据进行缩容，将i之前的数据删除。
-   stableSnapTo：该函数传入一个索引i，用于告诉unstable，索引i对应的快照数据已经被应用层持久化了，如果这个索引与当前快照数据对应的上，那么快照数据就可以被置空了。
-   restore：从快照数据中恢复，此时unstable将保存快照数据，同时将offset成员设置成这个快照数据索引的下一位。
-   truncateAndAppend：传入日志条目数组，这段数据将添加到entries数组中。但是需要注意的是，传入的数据跟现有的entries数据可能有重合的部分，所以需要根据unstable.offset与传入数据的索引大小关系进行处理，有些数据可能会被截断。
-   slice：返回索引范围在[lo-u.offset : hi-u.offset]之间的数据。
-   mustCheckOutOfBounds：检查传入的数据索引范围是否合理。

### Storage接口

Storage接口，提供了存储持久化日志相关的接口操作。其提供出来的接口函数说明如下。

-   InitialState() (pb.HardState, pb.ConfState, error)：返回当前的初始状态，其中包括硬状态（HardState）以及配置（里面存储了集群中有哪些节点）。
-   Entries(lo, hi, maxSize uint64) ([]pb.Entry, error)：传入起始和结束索引值，以及最大的尺寸，返回索引范围在这个传入范围以内并且不超过大小的日志条目数组。
-   Term(i uint64) (uint64, error)：传入日志索引i，返回这条日志对应的任期号。找不到的情况下error返回值不为空，其中当返回ErrCompacted表示传入的索引数据已经找不到，说明已经被压缩成快照数据了；返回ErrUnavailable：表示传入的索引值大于当前的最大索引。
-   LastIndex() (uint64, error)：返回最后一条数据的索引。
-   FirstIndex() (uint64, error)：返回第一条数据的索引。
-   Snapshot() (pb.Snapshot, error)：返回最近的快照数据。

我对这个接口提供出来的接口函数比较有疑问，因为搜索了etcd的代码，该接口只有MemoryStorage一个实现，而实际上MemoryStorage这个结构体还有其他的函数，比如添加日志数据的操作，但是这个操作并没有在Storage接口中声明。

接下来看看实现了Storage接口的MemoryStorage结构体的实现，其成员主要包括以下几个部分：

-   hardState pb.HardState：存储硬状态。
-   snapshot pb.Snapshot：存储快照数据。
-   ents []pb.Entry：存储紧跟着快照数据的日志条目数组，即ents[i]保存的日志数据索引位置为i + snapshot.Metadata.Index。

### raftLog的实现

有了以上的介绍unstable、Storage的准备之后，下面可以来介绍raftLog的实现，这个结构体承担了raft日志相关的操作。

raftLog由以下成员组成。

-   storage Storage：前面提到的存放已经持久化数据的Storage接口。
-   unstable unstable：前面分析过的unstable结构体，用于保存应用层还没有持久化的数据。
-   committed uint64：保存当前提交的日志数据索引。
-   applied uint64：保存当前传入状态机的数据最高索引。

需要说明的是，一条日志数据，首先需要被提交（committed）成功，然后才能被应用（applied）到状态机中。因此，以下不等式一直成立：applied <= committed。

raftLog结构体中，几部分数据的排列如下图所示。

![raftlog](https://www.codedump.info/media/imgs/20180922-etcd-raft/raftlog.png "raftlog")

这个数据排布的情况，可以从raftLog的初始化函数中看出来：

```Go
func newLog(storage Storage, logger Logger) *raftLog {
	if storage == nil {
		log.Panic("storage must not be nil")
	}
	log := &raftLog{
		storage: storage,
		logger:  logger,
	}
	firstIndex, err := storage.FirstIndex()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	lastIndex, err := storage.LastIndex()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	// offset从持久化之后的最后一个index的下一个开始
	log.unstable.offset = lastIndex + 1
	log.unstable.logger = logger
	// Initialize our committed and applied pointers to the time of the last compaction.
	// committed和applied从持久化的第一个index的前一个开始
	log.committed = firstIndex - 1
	log.applied = firstIndex - 1

	return log
}
```

在这里：

-   firstIndex：该值取自storage.FirstIndex()，可以从MemoryStorage的实现看到，该值是MemoryStorage.ents数组的第一个数据索引，也就是MemoryStorage结构体中快照数据与日志条目数据的分界线。
-   lastIndex：该值取自storage.LastIndex()，可以从MemoryStorage的实现看到，该值是MemoryStorage.ents数组的最后一个数据索引。
-   unstable.offset：该值为lastIndex索引的下一个位置。
-   committed、applied：在初始的情况下，这两个值是firstIndex的上一个索引位置，这是因为在firstIndex之前的数据既然已经是持久化数据了，说明都是已经被提交成功的数据了。

因此，从这里的代码分析可以看出，raftLog的两部分，持久化存储和非持久化存储，它们之间的分界线就是lastIndex，在此之前都是Storage管理的已经持久化的数据，而在此之后都是unstable管理的还没有持久化的数据。

以上分析中还有一个疑问，为什么并没有初始化unstable.snapshot成员，也就是unstable结构体的快照数据？原因在于，上面这个是初始化函数，也就是节点刚启动的时候调用来初始化存储状态的函数，而unstable.snapshot数据，是在启动之后同步数据的过程中，如果需要同步快照数据时才会去进行赋值修改的数据，因此在这里并没有对它进行操作的地方。

## raft消息结构体

大体而言，raft算法本质上是一个大的状态机，任何的操作例如选举、提交数据等，最后的操作一定是封装成一个消息结构体，输入到raft算法库的状态机中。

在raft/raftpb/raft.proto文件中，定义了raft算法中传输消息的结构体。熟悉raft论文的都知道，raft算法其实由好几个协议组成，但是在这里，统一定义在了Message这个结构体之中，以下总结了该结构体的成员用途。

![[Pasted image 20230423102506.png]]

由于这个Message结构体，全部将raft协议相关的数据都定义在了一起，有些协议不是用到其中的全部数据，所以这里的字段都是optinal的，我个人感觉这样不太好，会导致混合在一起显得杂乱无章，所以这里还是将每个协议（即不同的消息类型）中使用的用途做一个记录，如下。

## MsgHup消息

![[Pasted image 20230423102517.png]]

## MsgBeat消息

![[Pasted image 20230423102525.png]]

## MsgProp消息

![[Pasted image 20230423102539.png]]

raft库的使用者向raft库propose数据时，最后会封装成这个类型的消息来进行提交，不同类型的节点处理还不尽相同。

### candidate

由于candidate节点没有处理propose数据的责任，所以忽略这类型消息。

### follower

首先会检查集群内是否有leader存在，如果当前没有leader存在说明还在选举过程中，这种情况忽略这类消息；否则转发给leader处理。

### leader

leader的处理在leader的状态机函数针对MsgProp这种case的处理下，大体如下。

1.  检查entries数组是否没有数据，这是一个保护性检查。
2.  检查本节点是否还在集群之中，如果已经不在了则直接返回不进行下一步处理。什么情况下会出现一个leader节点发现自己不存在集群之中了？这种情况出现在本节点已经通过配置变化被移除出了集群的场景。
3.  检查raft.leadTransferee字段，当这个字段不为0时说明正在进行leader迁移操作，这种情况下不允许提交数据变更操作，因此此时也是直接返回的。
4.  检查消息的entries数组，看其中是否带有配置变更的数据。如果其中带有数据变更而raft.pendingConf为true，说明当前有未提交的配置更操作数据，根据raft论文，每次不同同时进行一次以上的配置变更，因此这里会将entries数组中的配置变更数据置为空数据。
5.  到了这里可以进行真正的数据propose操作了，将调用raft算法库的日志模块写入数据，根据返回的情况向其他节点广播消息。

## MsgApp/MsgSnap消息

### MsgApp消息

![[Pasted image 20230423102706.png]]

### MsgSnap消息

![[Pasted image 20230423102720.png]]

如果说前面的MsgProp消息是集群中的节点向leader转发用户提交的数据，那么MsgApp消息就是相反的，是leader节点用于向集群中其他节点同步数据的。

在这里把MsgSnap消息和MsgApp消息放在一起，是因为MsgSnap消息做的事情其实跟前面提到的MsgApp消息是一样的：都是用于leader向follower同步数据。实际上对于leader而言，向某个节点同步数据这个操作，都封装在raft.sendAppend函数中，至于具体用的哪种消息类型由这个函数内部实现。

那么，什么情况下会用到快照数据来同步呢？raft算法中，任何的数据要提交成功，首先leader会在本地写一份日志，再广播出去给集群的其他节点，只有在超过半数以上的节点同意，leader才能进行提交操作，这一个流程在前面讲解MsgAppResp消息流程时做了解释。

但是，如果这个日志文件不停的增长，显然是不能接受的。因此，在某些时刻，节点会将日志数据进行压缩处理，就是把当前的数据写入到一个快照文件中。而leader在向某一个节点进行数据同步时，是根据该节点上的日志记录进行数据同步的。

比方说，leader上已经有最大索引为10的日志数据，而节点A的日志索引是2，那么leader将从3开始向节点A同步数据。

但是如果前面的数据已经进行了压缩处理，转换成了快照数据，而压缩后的快照数据实际上已经没有日志索引相关的信息了。这时候只能将快照数据全部同步给节点了。还是以前面的流程为例，假如leader上日志索引为7之前的数据都已经被压缩成了快照数据，那么这部分数据在同步时是需要整份传输过去的，只有当同步完成节点赶上了leader上的日志进度时，才开始正常的日志同步流程。 而同步数据时，需要区分两种情况：

## MsgAppResp消息

![[Pasted image 20230423102935.png]]

在节点收到leader的MsgApp/MsgSnap消息时，可能出现leader上的数据与自身节点数据不一致的情况，这种情况下会返回reject为true的MsgAppResp消息，同时rejectHint字段是本节点raft最后一条日志的索引ID。

而index字段则返回的是当前节点的日志索引ID，用于向leader汇报自己已经commit的日志数据ID，这样leader就知道下一次同步数据给这个节点时，从哪条日志数据继续同步了。

leader节点在收到MsgAppResp消息的处理流程大体如下（stepLeader函数中MsgAppResp case的处理流程）。

-   首先，收到节点的MsgAppResp消息，说明该节点是活跃的，因此保存节点状态的RecentActive成员置为true。
    
-   接下来，再根据msg.Reject的返回值，即节点是否拒绝了这次数据同步，来区分两种情况进行处理。
    

### msg.Reject为true的情况

如果msg.Reject为true，说明节点拒绝了前面的MsgApp/MsgSnap消息，根据msg.RejectHint成员回退leader上保存的关于该节点的日志记录状态。比如leader前面认为从日志索引为10的位置开始向节点A同步数据，但是节点A拒绝了这次数据同步，同时返回RejectHint为2，说明节点A告知leader在它上面保存的最大日志索引ID为2，这样下一次leader就可以直接从索引为2的日志数据开始同步数据到节点A。而如果没有这个RejectHint成员，leader只能在每次被拒绝数据同步后都递减1进行下一次数据同步，显然这样是低效的。

1.  因为上面节点拒绝了这次数据同步，所以节点的状态可能存在一些异常，此时如果leader上保存的节点状态为ProgressStateReplicate，那么将切换到ProgressStateProbe状态（关于这几种状态，下面会谈到）。
    
2.  前面已经按照msg.RejectHint修改了leader上关于该节点日志状态的索引数据，接着再次尝试按照这个新的索引数据向该节点再次同步数据。
    

### msg.Reject为false的情况

这种情况说明这个节点通过了leader的这一次数据同步请求，这种情况下根据msg.Index来判断在leader中保存的该节点日志数据索引是否发生了更新，如果发生了更新那么就说明这个节点通过了新的数据，这种情况下会做以下的几个操作。

1.  修改节点状态

-   如果该节点之前在ProgressStateProbe状态，说明之前处于探测状态，此时可以切换到ProgressStateReplicate，开始正常的接收leader的同步数据了。
-   如果之前处于ProgressStateSnapshot状态，即还在同步副本，说明节点之前可能落后leader数据比较多才采用了接收副本的状态。这里还需要多做一点解释，因为在节点落后leader数据很多的情况下，可能leader会多次通过snapshot同步数据给节点，而当 pr.Match >= pr.PendingSnapshot的时候，说明通过快照来同步数据的流程完成了，这时可以进入正常的接收同步数据状态了，这就是函数Progress.needSnapshotAbort要做的判断。
-   如果之前处于ProgressStateReplicate状态，此时可以修改leader关于这个节点的滑动窗口索引，释放掉这部分数据索引，好让节点可以接收新的数据了。关于这个滑动窗口设计，见下面详细解释。

2.  判断是否有新的数据可以提交（commit）了。因为raft的提交数据的流程是这样的：首先节点将数据提议（propose）给leader，leader在将数据写入到自己的日志成功之后，再通过MsgApp把这些提议的数据广播给集群中的其他节点，在某一条日志数据收到超过半数（qurom）的节点同意之后，才认为是可以提交（commit）的。因此每次leader节点在收到一条MsgAppResp类型消息，同时msg.Reject又是false的情况下，都需要去检查当前有哪些日志是超过半数的节点同意的，再将这些可以提交（commit）的数据广播出去。而在没有数据可以提交的情况下，如果之前节点处于暂停状态，那么将继续向该节点同步数据。
    
3.  最后还要做一个跟leader迁移相关的操作。如果该消息节点是准备迁移过去的新leader节点（raft.leadTransferee == msg.From），而且此时该节点上的Match索引已经跟旧的leader的日志最大索引一致，说明新旧节点的日志数据已经同步，可以正式进行集群leader迁移操作了。
    

## MsgVote/MsgPreVote消息以及MsgVoteResp/MsgPreVoteResp消息

这里把这四种消息放在一起了，因为不论是Vote还是PreVote流程，其请求和应答时传输的数据都是一样的。

先看请求数据。

![[Pasted image 20230423103422.png]]

节点调用raft.campaign函数进行投票给自己进行一次新的选举，其中的参数CampaignType有以下几种类型：

1.  campaignPreElection：对应PreVote的场景。
2.  campaignElection：正常的选举场景。
3.  campaignTransfer：由于leader迁移发生的选举。如果是这种类型的选举，那么msg.Context字段保存的是“CampaignTransfer”`字符串，这种情况下会强制进行leader的迁移。

MsgVote还需要带上几个与本节点日志相关的数据（Index、LogTerm），因为raft算法要求，一个节点要成为leader的一个必要条件之一就是这个节点上的日志数据是最新的。

### PreVote

这里需要特别解释一下PreVote的场景。

考虑到一种情况：当出现网络分区的时候，A、B、C、D、E五个节点被划分成了两个网络分区，A、B、C组成的分区和D、E组成的分区，其中的D节点，如果在选举超时到来时，都没有收到来自leader节点A的消息（因为网络已经分区），那么D节点认为需要开始一次新的选举了。

正常的情况下，节点D应该把自己的任期号term递增1，然后发起一次新的选举。由于网络分区的存在，节点D肯定不会获得超过半数以上的的投票，因为A、B、C三个节点组成的分区不会收到它的消息，这会导致节点D不停的由于选举超时而开始一次新的选举，而每次选举又会递增任期号。

在网络分区还没恢复的情况下，这样做问题不大。但是当网络分区恢复时，由于节点D的任期号大于当前leader节点的任期号，这会导致集群进行一次新的选举，即使节点D肯定不会获得选举成功的情况下（因为节点D的日志落后当前集群太多，不能赢得选举成功）。

为了避免这种无意义的选举流程，节点可以有一种PreVote的状态，在这种状态下，想要参与选举的节点会首先连接集群的其他节点，只有在超过半数以上的节点连接成功时，才能真正发起一次新的选举。

所以，在PreVote状态下发起选举时，并不会导致节点本身的任期号递增1，而只有在进行正常选举时才会将任期号加1进行选举。

### MsgVote/MsgPreVote的处理流程

来看看节点对于投票消息的处理，这些处理有两处，但是都在raft.Step函数中。

1.  首先该函数会判断msg.Term是否大于本节点的Term，如果消息的任期号更大则说明是一次新的选举。这种情况下将根据msg.Context是否等于“CampaignTransfer”字符串来确定是不是一次由于leader迁移导致的强制选举过程。同时也会根据当前的electionElapsed是否小于electionTimeout来确定是否还在租约期以内。如果既不是强制leader选举又在租约期以内，那么节点将忽略该消息的处理，在论文4.2.3部分论述这样做的原因，是为了避免已经离开集群的节点在不知道自己已经不在集群内的情况下，仍然频繁的向集群内节点发起选举导致耗时在这种无效的选举流程中。如果以上检查流程通过了，说明可以进行选举了，如果消息类型还不是MsgPreVote类型，那么此时节点会切换到follower状态且认为发送消息过来的节点msg.From是新的leader。

```go
	case m.Term > r.Term:
		// 消息的Term大于节点当前的Term
		lead := m.From
		if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote {
			// 如果收到的是投票类消息

			// 当context为campaignTransfer时表示强制要求进行竞选
			force := bytes.Equal(m.Context, []byte(campaignTransfer))
			// 是否在租约期以内
			inLease := r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout
			if !force && inLease {
				// 如果非强制，而且又在租约期以内，就不做任何处理
				// 非强制又在租约期内可以忽略选举消息，见论文的4.2.3，这是为了阻止已经离开集群的节点再次发起投票请求
				// If a server receives a RequestVote request within the minimum election timeout
				// of hearing from a current leader, it does not update its term or grant its vote
				r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d, index: %d] at term %d: lease is not expired (remaining ticks: %d)",
					r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term, r.electionTimeout-r.electionElapsed)
				return nil
			}
			// 否则将lead置为空
			lead = None
		}
		switch {
		// 注意Go的switch case不做处理的话是不会默认走到default情况的
		case m.Type == pb.MsgPreVote:
			// Never change our term in response to a PreVote
			// 在应答一个prevote消息时不对任期term做修改
		case m.Type == pb.MsgPreVoteResp && !m.Reject:
			// We send pre-vote requests with a term in our future. If the
			// pre-vote is granted, we will increment our term when we get a
			// quorum. If it is not, the term comes from the node that
			// rejected our vote so we should become a follower at the new
			// term.
		default:
			r.logger.Infof("%x [term: %d] received a %s message with higher term from %x [term: %d]",
				r.id, r.Term, m.Type, m.From, m.Term)
			// 变成follower状态
			r.becomeFollower(m.Term, lead)
		}
```

2.  在raft.Step函数的后面，会判断消息类型是MsgVote或者MsgPreVote来进一步进行处理。其判断条件是以下两个条件同时成立：

-   当前没有给任何节点进行过投票（r.Vote == None ），或者消息的任期号更大（m.Term > r.Term ），或者是之前已经投过票的节点（r.Vote == m.From)）。这个条件是检查是否可以还能给该节点投票。
-   同时该节点的日志数据是最新的（r.raftLog.isUpToDate(m.Index, m.LogTerm) ）。这个条件是检查这个节点上的日志数据是否足够的新。 只有在满足以上两个条件的情况下，节点才投票给这个消息节点，将修改raft.Vote为消息发送者ID。如果不满足条件，将应答msg.Reject=true，拒绝该节点的投票消息。

```go
	case pb.MsgVote, pb.MsgPreVote:
		// 收到投票类的消息
		// The m.Term > r.Term clause is for MsgPreVote. For MsgVote m.Term should
		// always equal r.Term.
		if (r.Vote == None || m.Term > r.Term || r.Vote == m.From) && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
			// 如果当前没有给任何节点投票（r.Vote == None）或者投票的节点term大于本节点的（m.Term > r.Term）
			// 或者是之前已经投票的节点（r.Vote == m.From）
			// 同时还满足该节点的消息是最新的（r.raftLog.isUpToDate(m.Index, m.LogTerm)），那么就接收这个节点的投票
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] cast %s for %x [logterm: %d, index: %d] at term %d",
				r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			r.send(pb.Message{To: m.From, Type: voteRespMsgType(m.Type)})
			if m.Type == pb.MsgVote {
				// Only record real votes.
				// 保存下来给哪个节点投票了
				r.electionElapsed = 0
				r.Vote = m.From
			}
		} else {
			// 否则拒绝投票
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected %s from %x [logterm: %d, index: %d] at term %d",
				r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			r.send(pb.Message{To: m.From, Type: voteRespMsgType(m.Type), Reject: true})
		}
```

### MsgVoteResp/MsgPreVoteResp的处理流程

来看节点收到投票应答数据之后的处理。

1.  节点调用raft.poll函数，其中传入msg.Reject参数表示发送者是否同意这次选举，根据这些来计算当前集群中有多少节点给这次选举投了同意票。
2.  如果有半数的节点同意了，如果选举类型是PreVote，那么进行Vote状态正式进行一轮选举；否则该节点就成为了新的leader，调用raft.becomeLeader函数切换状态，然后开始同步日志数据给集群中其他节点了。
3.  而如果半数以上的节点没有同意，那么重新切换到follower状态。

```Go
	case myVoteRespType:
		// 计算当前集群中有多少节点给自己投了票
		gr := r.poll(m.From, m.Type, !m.Reject)
		r.logger.Infof("%x [quorum:%d] has received %d %s votes and %d vote rejections", r.id, r.quorum(), gr, m.Type, len(r.votes)-gr)
		switch r.quorum() {
		case gr:	// 如果进行投票的节点数量正好是半数以上节点数量
			if r.state == StatePreCandidate {
				r.campaign(campaignElection)
			} else {
				// 变成leader
				r.becomeLeader()
				r.bcastAppend()
			}
		case len(r.votes) - gr:	// 如果是半数以上节点拒绝了投票
			// 变成follower
			r.becomeFollower(r.Term, None)
		}
```

## MsgHeartbeat/MsgHeartbeatResp消息

![[Pasted image 20230423105759.png]]

leader中会定时向集群中其他节点发送心跳消息，该消息的作用除了探测节点的存活情况之外，还包括：

1.  commit成员：leader选择min[节点上的Match，leader日志最大提交索引]，用于告知节点哪些日志可以进行提交（commit）。
2.  context：与线性一致性读相关，后面会进行解释。

## MsgUnreachable消息

![[Pasted image 20230423105925.png]]

仅leader才处理这类消息，leader如果判断该节点此时处于正常接收数据的状态（ProgressStateReplicate），那么就切换到探测状态。

## MsgSnapStatus消息

![[Pasted image 20230423105936.png]]

仅leader处理这类消息：

1.  如果reject为false：表示接收快照成功，将切换该节点状态到探测状态。
2.  否则接收失败。

## MsgCheckQuorum消息

![[Pasted image 20230423110014.png]]

leader的定时器函数，在超过选举时间时，如果当前打开了raft.checkQuorum开关，那么leader将给自己发送一条MsgCheckQuorum消息，对该消息的处理是：检查集群中所有节点的状态，如果超过半数的节点都不活跃了，那么leader也切换到follower状态。

## MsgTransferLeader消息

![[Pasted image 20230423110027.png]]

这类消息follower将转发给leader处理，因为follower并没有修改集群配置状态的权限。

leader在收到这类消息时，是以下的处理流程。

1.  如果当前的raft.leadTransferee成员不为空，说明有正在进行的leader迁移流程。此时会判断是否与这次迁移是同样的新leader ID，如果是则忽略该消息直接返回；否则将终止前面还没有完毕的迁移流程。
2.  如果这次迁移过去的新节点，就是当前的leader ID，也直接返回不进行处理。
3.  到了这一步就是正式开始这一次的迁移leader流程了，一个节点能成为一个集群的leader，其必要条件是上面的日志与当前leader的一样多，所以这里会判断是否满足这个条件，如果满足那么发送MsgTimeoutNow消息给新的leader通知该节点进行leader迁移，否则就先进行日志同步操作让新的leader追上旧leader的日志数据。

```Go
case pb.MsgTransferLeader:
  leadTransferee := m.From
  lastLeadTransferee := r.leadTransferee
  if lastLeadTransferee != None {
    // 判断是否已经有相同节点的leader转让流程在进行中
    if lastLeadTransferee == leadTransferee {
      r.logger.Infof("%x [term %d] transfer leadership to %x is in progress, ignores request to same node %x",
        r.id, r.Term, leadTransferee, leadTransferee)
      // 如果是，直接返回
      return
    }
    // 否则中断之前的转让流程
    r.abortLeaderTransfer()
    r.logger.Infof("%x [term %d] abort previous transferring leadership to %x", r.id, r.Term, lastLeadTransferee)
  }
  // 判断是否转让过来的leader是否本节点，如果是也直接返回，因为本节点已经是leader了
  if leadTransferee == r.id {
    r.logger.Debugf("%x is already leader. Ignored transferring leadership to self", r.id)
    return
  }
  // Transfer leadership to third party.
  r.logger.Infof("%x [term %d] starts to transfer leadership to %x", r.id, r.Term, leadTransferee)
  // Transfer leadership should be finished in one electionTimeout, so reset r.electionElapsed.
  r.electionElapsed = 0
  r.leadTransferee = leadTransferee
  if pr.Match == r.raftLog.lastIndex() {
    // 如果日志已经匹配了，那么就发送timeoutnow协议过去
    r.sendTimeoutNow(leadTransferee)
    r.logger.Infof("%x sends MsgTimeoutNow to %x immediately as %x already has up-to-date log", r.id, leadTransferee, leadTransferee)
  } else {
    // 否则继续追加日志
    r.sendAppend(leadTransferee)
  }
```

## MsgTimeoutNow消息

![[Pasted image 20230423112523.png]]

新的leader节点，在还未迁移之前仍然是follower，在收到这条消息后，就可以进行迁移了，此时会调用前面分析MsgVote时说过的campaign函数，传入的参数是campaignTransfer，表示这是一次由于迁移leader导致的选举流程。

## MsgReadIndex和MsgReadIndexResp消息

这两个消息一一对应，使用的成员也一样，在后面分析读一致性的时候再详细解释。

![[Pasted image 20230423112540.png]]

其中，entries数组只会有一条数据，带上的是应用层此次请求的标识数据，在follower收到MsgReadIndex消息进行应答时，同样需要把这个数据原样带回返回给leader，详细的线性读一致性的实现在后面展开分析。

# 节点状态

每个raft的节点，分为以下三种状态：

1.  candidate：候选人状态，节点切换到这个状态时，意味着将进行一次新的选举。
2.  follower：跟随者状态，节点切换到这个状态时，意味着选举结束。
3.  leader：领导者状态，所有数据提交都必须先提交到leader上。

每一个状态都有其对应的状态机，每次收到一条提交的数据时，都会根据其不同的状态将消息输入到不同状态的状态机中。同时，在进行tick操作时，每种状态对应的处理函数也是不一样的。

所以raft结构体中将不同的状态，及其不同的处理函数独立出来几个成员变量：

![[Pasted image 20230423112554.png]]

raft库中提供几个成员函数becomeCandidate、becomeFollower、becomeLeader分别进入这几种状态的，这些函数中做的事情，概况起来就是：

1.  切换raft.state成员到对应状态。
2.  切换raft.tick函数到对应状态的处理函数。
3.  切换raft.step函数到对应状态的状态机。

# 选举流程

raft算法的第一步是首先选举出一个leader出来，在没有产生leader的情况下，其他数据提交等操作都无从谈起，所以这里首先从选举的流程开始谈起。

## 发起选举的节点

只有在candidate或者follower状态下的节点，才有可能发起一个选举流程，而这两种状态的节点，其对应的tick函数都是raft.tickElection函数，这个函数的主要流程是：

1.  将选举超时递增1。
2.  当选举超时到期，同时该节点又在集群中时，说明此时可以进行一轮新的选举。此时会向本节点发送HUP消息，这个消息最终会走到状态机函数raft.Step中进行处理。

明白了raft.tickElection函数的作用，可以来看选举流程了：

-   节点启动时都以follower状态启动，同时随机选择自己的选举超时时间。之所以每个节点随机选择自己的超时时间，是为了避免同时有两个节点同时进行选举，这种情况下会出现没有任何一个节点赢得半数以上的投票从而这一轮选举失败，继续再进行下一轮选举
    
-   在follower的tick函数tickElection函数中，当选举超时到时，节点向自己发送HUP消息。
    
-   在状态机函数raft.Step函数中，在收到HUP消息之后，节点首先判断当前有没有没有apply的配置变更消息，如果有就忽略该消息。其原因在于，当有配置更新的情况下不能进行选举操作，即要保证每一次集群成员变化时只能同时变化一个，不能同时有多个集群成员的状态发生变化。
    
-   否则进入campaign函数中进行选举：首先将任期号+1，然后广播给其他节点选举消息，带上的其它字段包括：节点当前的最后一条日志索引（Index字段），最后一条日志对应的任期号（LogTerm字段），选举任期号（Term字段，即前面已经进行+1之后的任期号），Context字段（目的是为了告知这一次是否是leader转让类需要强制进行选举的消息）。
    
-   如果在一个选举超时之内，该发起新的选举流程的节点，得到了超过半数的节点投票，那么状态就切换到leader状态，成为leader的同时，leader将发送一条dummy的append消息，目的是为了提交该节点上在此任期之前的值（见疑问部分如何提交之前任期的值）
    

## 收到选举消息的节点

-   当收到任期号大于当前节点任期号的消息，同时该消息类型如果是选举类的消息（类型为prevote或者vote）时，会做以下判断：
    
    1.  首先会判断一下该消息是否为强制要求进行选举的类型（context为campaignTransfer，context为这种类型时表示在进行leader转让，流程见下面的leader转让流程）
        
    2.  判断当前是否在租约期以内，判断的条件包括：checkQuorum为true，当前节点保存的leader不为空，没有到选举超时，前面这三个条件同时满足。
        

如果不是强制要求选举，同时又在租约期以内，那么就忽略该选举消息返回不进行处理，这么做是为了避免出现那些离开集群的节点，频繁发起新的选举请求（见论文4.2.3）。

-   如果不是前面的忽略选举消息的情况，那么除非是prevote类的选举消息，在收到其他消息的情况下，该节点都切换为follower状态。
    
-   此时需要针对投票类型中带来的其他字段进行处理了，需要同时满足以下两个条件：
    
    1.  只有在没有给其他节点进行过投票，或者消息的term任期号大于当前节点的任期号，或者之前的投票给的就是这个发出消息的节点
        
    2.  进行选举的节点，它的日志是更新的，条件为：logterm比本节点最新日志的任期号大，在两者相同的情况下，消息的index大于等于当前节点最新日志的index，即总要保证该选举节点的日志比自己的大。
        

只有在同时满足以上两个条件的情况下，才能同意该节点的选举，否则都会被拒绝。这么做的原因是：保证最后能胜出来当新的leader的节点，它上面的日志都是最新的。

# 集群成员变化流程

**大原则是不能同时进行两个以上的成员变更**，因为同时进行两个以上的成员变更，可能会出现集群中有两个leader即导致了集群分裂的情况出现。

成员变化分为以下几种情况：成员删减、leader转让，下面分开讲解。

## 一般的成员删减

成员变化操作做为日志的特殊类型，当可以进行commit的情况下，各个节点拿出该消息进行节点内部的成员删减操作。

## leader转让

-   旧leader在接收到转让leader消息之后，会做如下的判断： a. 如果新的leader上的日志，已经跟当前leader上的日志同步了，那么发送timeout消息。 b. 否则继续发append消息到新的leader上，目的为了让其能够与旧leader日志同步。
    
-   当旧leader处于转让leader状态时，将停止接收新的prop消息，这样就避免出现在转让过程中新旧leader一直日志不能同步的情况。
    
-   当旧leader收到append消息应答时，如果当前处于leader转让状态，那么会判断新的leader日志是否已经与当前leader同步，如果是将发送timeout消息。
    
-   新的leader当收到timeout消息时，将使用context为campaignTransfer的选举消息发起新一轮选举，当context为该类型时，此时的选举是强制进行的（见前面的选举流程）。
    

# 如何做到线性一致性？

线性一致性（Linearizable Read）通俗来讲，就是读请求需要读到最新的已经commit的数据，不会读到老数据。

由于所有的leader和follower都能处理客户端的读请求，所以存在可能造成返回读出的旧数据的情况：

-   leader和follower之间存在状态差，因为follower总是由leader同步过去的，可能会返回同步之前的数据。
    
-   如果发生了网络分区，某个leader实际上已经被隔离出了集群之外，但是该leader并不知道，如果还继续响应客户端的读请求，也可能会返回旧的数据。
    

因此，在接收到客户端的读请求时，需要保证返回的数据都是当前最新的。

## ReadOnlySafe方式

leader在接收到读请求时，需要向集群中的超半数server确认自己仍然是当前的leader，这样它返回的就是最新的数据。

在etcd-raft中，为了实现ReadOnlySafe，有如下的数据结构：

```Go
type ReadState struct {
  Index uint64
  RequestCtx []byte
}
```

其中：

1.  Index：接收到该读请求时，当前节点的commit索引。
2.  RequestCtx：客户端读请求的唯一标识。

ReadState结构体用于保存读请求到来时的节点状态。

```Go
type readIndexStatus struct {
 req pb.Message
 index uint64
 acks map[uint64]struct{}
}
```

readIndexStatus数据结构用于追踪leader向follower发送的心跳信息，其中：

1.  req：保存原始的readIndex请求。
2.  index：leader当前的commit日志索引。
3.  acks：存放该readIndex请求有哪些节点进行了应答，当超过半数应答时，leader就可以确认自己还是当前集群的leader。

```Go
type readOnly struct {
 option ReadOnlyOption
 pendingReadIndex map[string]*readIndexStatus
 readIndexQueue []string
}
```

readOnly用于管理全局的readIndx数据，其中：

1.  option：readOnly选项。
2.  pendingReadIndex：当前所有待处理的readIndex请求，其中key为客户端读请求的唯一标识。
3.  readIndexQueue：保存所有readIndex请求的请求唯一标识数组。

有了以上的数据结构介绍，后面是流程介绍：

1.  server收到客户端的读请求，此时会调用raft.ReadIndex函数发起一个MsgReadIndex的请求，带上的参数是客户端读请求的唯一标识（此时可以对照前面分析的MsgReadIndex及其对应应答消息的格式）。
    
2.  follower将向leader直接转发MsgReadIndex消息，而leader收到不论是本节点还是由其他server发来的MsgReadIndex消息，其处理都是：
    
    a. 首先如果该leader在成为新的leader之后没有提交过任何值，那么会直接返回不做处理。
    
    b. 调用r.readOnly.addRequest(r.raftLog.committed, m)保存该MsgreadIndex请求到来时的commit索引。
    
    c. r.bcastHeartbeatWithCtx(m.Entries[0].Data)，向集群中所有其他节点广播一个心跳消息MsgHeartbeat，并且在其中带上该读请求的唯一标识。
    
    d. follower在收到leader发送过来的MsgHeartbeat，将应答MsgHeartbeatResp消息，并且如果MsgHeartbeat消息中有ctx数据，MsgHeartbeatResp消息将原样返回这个ctx数据。
    
    e. leader在接收到MsgHeartbeatResp消息后，如果其中有ctx字段，说明该MsgHeartbeatResp消息对应的MsgHeartbeat消息，是收到ReadIndex时leader消息为了确认自己还是集群leader发送的心跳消息。首先会调用r.readOnly.recvAck(m)函数，根据消息中的ctx字段，到全局的pendingReadIndex中查找是否有保存该ctx的带处理的readIndex请求，如果有就在acks map中记录下该follower已经进行了应答。
    
    f. 当ack数量超过了集群半数时，意味着该leader仍然还是集群的leader，此时调用r.readOnly.advance(m)函数，将该readIndex之前的所有readIndex请求都认为是已经成功进行确认的了，所有成功确认的readIndex请求，将会加入到readStates数组中，同时leader也会向follower发送MsgReadIndexResp。
    
    g. follower收到MsgReadIndexResp消息时，同样也会更新自己的readStates数组信息。
    
    h. readStates数组的信息，将做为ready结构体的信息更新给上层的raft协议库的使用者。
    

需要特别说明的是，处理读请求时，实际上leader需要确保当前自己是不是leader、该读请求对应的commit索引是否得到了半数投票，而当一个节点刚成为leader的时候，如果没有提交过任何数据，那么在它所在的这个任期（term）内的commit索引当时是并不知道的，因此在成为leader之后，需要马上提交一个no-op的空日志，这样拿到该任期的第一个commit索引。

![msgreadindex-1](https://www.codedump.info/media/imgs/20180922-etcd-raft/msgreadindex-1.png "msgreadindex-1")

上图中，在leader收到MsgReadIndex后：

-   向readOnly中添加与这次请求ctx相关的数据：
    -   向pendingReadIndex中添加以ctx为key的readIndexStatus，其中保存了当前的commitIndex、原始的MsgReadIndex消息、以及用于存放有哪些节点应答了该消息的acks数组。
    -   向readIndexQueue数组中添加ctx。
-   leader向集群中其他节点广播MsgHeartbeat消息，其中带上这次MsgReadIndex的ctx。

在这之后，follower应答leader的MsgHeartbeat消息，如果消息中存在ctx字段都会带上应答，于是leader中的处理：

-   收到MsgHeartbeatResp消息之后，如果发现其中有ctx，就去计算应答有没有超过半数，没有超过半数则返回。
-   走到这里就是超过半数应答了，此时拿到新的readIndexStatus数组。
-   遍历前面拿到的readIndexStatus数组，生成新的readStates数组。
-   放到Ready中下一次给客户端。

总结一下，分为四步：

-   leader检查自己在当前任期有没有commit过一条entry，没有提交过则不允许处理readIndex请求。
-   leader记录下来收到readIndex请求时候的commit index，然后leader向集群中所有节点发心跳广播，其中带上readIndex相关的ctx字段。
-   当超过半数的节点应答了第二部的心跳消息，说明此时leader还是集群的leader。
-   生成新的readStates数组放入Ready结构体中，等待下一次客户端来获取该数据。

# 杂项

## 节点的几种状态

一个节点在leader上保存的状态有:

```Go
const (
    ProgressStateProbe ProgressStateType = iota
    ProgressStateReplicate
    ProgressStateSnapshot
)
```

以下来分开解释这几种状态。

### ProgressStateProbe

探测状态，当节点拒绝了最近的append消息时，那么就会进入探测状态，此时leader会试图继续往前追述该节点的日志从哪里开始丢失的，让该节点的日志能跟leader同步上。在probe状态时，只能向它发送一次append消息，此后除非状态发生变化，否则就暂停向该节点发送新的append消息了。

只有在以下情况才会恢复取消暂停状态（调用Progress的resume函数）：

1.  收到该节点的心跳消息。
2.  该节点成功应答了前面的最后一条append消息。

至于Probe状态，只有在该节点成功应答了Append消息之后，在leader上保存的索引值发生了变化，才会修改其状态切换到Replicate状态。

### ProgressStateReplicate

正常接收副本数据的状态，当处于该状态时，leader在发送副本消息之后，就修改该节点的next索引为发送消息的最大索引+1

### ProgressStateSnapshot

接收快照状态。 当leader向某个follower发送append消息，试图让该follower状态跟上leader时，发现此时leader上保存的索引数据已经对不上了，比如leader在index为10之前的数据都已经写入快照中了，但是该follower需要的是10之前的数据，此时就会切换到该状态下，发送快照给该follower。

因为快照数据可能很多，不知道会同步多久，所以单独把这个状态抽象出来。

当快照数据同步追上之后，并不是直接切换到Replicate状态，而是首先切换到Probe状态。

## Progress上的数据索引

Progress结构体中有两个保存该follower节点日志索引的数据，其中：

-   Next：保存下一次leader发送append消息给该follower时的日志索引。
-   Match：保存该follower节点上的最大日志索引。

在正常情况下，Next = Match + 1，也就是Next总是节点当前保存最大日志索引的下一条索引。

有两种情况除外：

-   接收快照状态：此时Next = max(pr.Match+1, pendingSnapshot+1)
-   当该follower不在Replicate状态时，说明不是正常的接收副本状态。此时当leader与follower同步leader上的日志时，可能出现覆盖的情况，即此时follower上面假设Match为3，但是索引为3的数据会被leader覆盖，此时Next指针可能会一直回溯到与leader上日志匹配的位置，再开始正常同步日志，此时也会出现Next != Match + 1的情况出现。

![next-match](https://www.codedump.info/media/imgs/20180922-etcd-raft/next-match.jpg "next-match")

如上图所示，节点s1上最大日志索引为2，即Match = 2，Next = 3。 但是，由于新选出来的leader s2，其最大日志索引为3，此时s3需要同步日志到s1上，发现s1上的日志与自己的不匹配，所以会一直找到两者最开始匹配的索引位置，即最终找到索引1，因此会保存s1的Next索引为1，而Match还是2（因为此时还没有修改s1上的日志）。当最终s1上的数据与s2同步时，此时Next = 4，Match=3。

## 流量控制

Progress结构体中，使用另一个inflights的数据结构用于流量控制。 该结构体使用一个固定大小的循环缓冲区来控制给一个节点同步数据的流量控制，每当给该follower发送同步消息时，就占用该缓冲区的一个空间；反之，当收到该follower的成功接收了该同步消息的应答之后，就释放缓冲区的空间。

当该缓冲区数据饱和时，将暂停继续同步数据到该follower。

# Etcd存储的实现
# 概览

在前面已经分析了Raft算法原理、etcd raft库的实现，接着就可以看etcd如何使用raft实现存储服务的了。

以下的分析主要针对etcd V3版本的实现。

下图中展示了etcd如何处理一个客户端请求的涉及到的模块和流程。图中淡紫色的矩形表示etcd，它包括如下几个模块：

-   etcd server：对外接收客户端的请求，对应etcd代码中的etcdserver目录，其中还有一个raft.go的模块与etcd-raft库进行通信。etcdserver中与存储相关的模块是applierV3，这里封装了V3版本的数据存储，WAL（write ahead log），用于写数据日志，etcd启动时会根据这部分内容进行恢复。
-   etcd raft：etcd的raft库，前面的文章已经具体分析过这部分代码。除了与本节点的etcd server通信之外，还与集群中的其他etcd server进行交互做一致性数据同步的工作（在图中集群中其他etcd服务用橙色的椭圆表示）。

![etcd server](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-server.png "etcd server")

在上图中，一个请求与一个etcd集群交互的主要流程分为两大部分：

1.  写数据到某个etcd server中。
2.  该etcd server与集群中的其他etcd节点进行交互，当确保数据已经被存储之后应答客户端。

请求流程划分为了以下的子步骤：

-   1.1：etcd server收到客户端请求。
-   1.2：etcd server将请求发送给本模块中的raft.go，这里负责与etcd raft模块进行通信。
-   1.3：raft.go将数据封装成raft日志的形式提交给raft模块。
-   1.4：raft模块会首先保存到raftLog的unstable存储部分。
-   1.5：raft模块通过raft协议与集群中其他etcd节点进行交互。

注意在以上流程中，假设这里写入数据的etcd是leader节点，因为在raft协议中，如果提交数据到非leader节点的话需要路由到etcd leader节点去。

而应答步骤如下：

-   2.1：集群中其他节点向leader节点应答接收这条日志数据。
-   2.2：当超过集群半数以上节点应答接收这条日志数据时，etcd raft通过Ready结构体通知etcd server中的raft该日志数据已经commit。
-   2.3：raft.go收到Ready数据将首先将这条日志写入到WAL模块中。
-   2.4：通知最上层的etcd server该日志已经commit。
-   2.5：etcd server调用applierV3模块将日志写入持久化存储中。
-   2.6：etcd server应答客户端该数据写入成功。
-   2.7：最后etcd server调用etcd raft，修改其raftLog模块的数据，将这条日志写入到raftLog的storage中。

从上面的流程可以看到

-   etcd raft模块在应答某条日志数据已经commit之后，是首先写入到WAL模块中的，因为这个模块只是添加一条日志，所以速度会很快，即使在后面applierV3写入失败，重启的时候也可以根据WAL模块中的日志数据进行恢复。
-   etcd raft中的raftLog，按照前面文章的分析，其中的数据是保存到内存中的，重启即失效，上层应用真实的数据是持久化保存到WAL和applierV3中的。

以下就来分析etcd server与这部分相关的几个模块。

# etcd server与raft的交互

EtcdServer结构体，负责对外与客户端进行通信。内部有一个raftNode结构的成员，负责与etcd的raft库进行交互。

etcd V3版本的API，通过GRPC协议与客户端进行交互，其相关代码在etcdserver/v3_server.go中。以一次Put请求为例，最后将会调用的代码在函数EtcdServer::processInternalRaftRequestOnce中，代码的主要流程分析如下。

![etcd flowchart](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-flowchart.png "etcd flowchart")

1.  拿到当前raft中的apply和commit索引，如果commit索引比apply索引超出太多，说明当前有很多数据都没有apply，返回ErrTooManyRequests错误。
2.  调用s.reqIDGen.Next()函数生成一个针对当前请求的ID，注意这个ID并不是一个随机数而是一个严格递增的整数。同时将请求序列化为byte数据，这会做为raft的数据进行存储。
3.  根据第2步中的ID，调用Wait.Register函数进行注册，这会返回一个用于通知结果的channel，后续就通过监听该channel来确定是否成功储存了提交的值。
4.  调用Raft.Process函数提交数据，这里传入的参数除了前面序列化的数据之外，还有使用超时时间创建的Context。
5.  监听前面的Channel以及Context对象： a. 如果context.Done返回，说明数据提交超时，使用s.parseProposeCtxErr函数返回具体的错误。 b. 如果channel返回，说明已经提交成功。

从以上的流程可以看出，在调用Raft.Process函数向Raft库提交数据之后，等待被唤醒的Channel才是正常提交数据成功的路径。

在EtcdServer.run函数中，最终会进入一个死循环中，等待raftNode.apply返回的channel被唤醒，而raftNode继承了raft.Node的实现，从前面分析etcd raft的流程中可以明白，EtcdServer就是在向raft库提交了数据之后，做为其上层消费Ready数据的应用层。

自此，整体的流程大体已经清晰：

1.  EtcdServer对外通过GRPC协议接收客户端请求，对内有一个raftNode类型的成员，该类型继承了raft.Node的实现。
2.  客户端通过EtcdServer提交的数据修改都会通过raftNode来提交，而EtcdServer本身通过监听channel与raft库进行通信，由Ready结构体来通过EtcdServer哪些数据已经提交成功。
3.  由于每个请求都会一个对应的ID，ID绑定了Channel，所以提交成功的请求通过ID找到对应的Channel来唤醒提交流程，最后通知客户端提交数据成功。

# WAL

以上介绍了EtcdServer的大体流程，接下来看WAL的实现。

前面已经分析过了，etcd raft提交数据成功之后，将通知上面的应用层（在这里就是EtcdServer），然后再进行持久化数据存储。而数据的持久化可能会花费一些时间，因此在应答应用层之前，EtcdServer中的raftNode会首先将这些数据写入WAL日志中。这样即使在做持久化的时候数据丢失了，启动恢复的时候也可以根据WAL的日志进行数据恢复。

etcdserver模块中，给raftNode用于写WAL日志的工作，交给了接口Storage来完成，而这个接口由storage来具体实现：

```Go
type storage struct {
	*wal.WAL
	*snap.Snapshotter
}
```

可以看到，这个结构体组合了WAL和snap.Snapshotter结构，Snapshotter负责的是存储快照数据。

WAL日志文件中，每条日志记录有以下的类型：

1.  Type：日志记录类型，下面详细解释都有哪些类型。
2.  Crc：这一条日志记录的校验数据。
3.  Data：真正的数据，根据类型不同存储的数据也不同。

日志记录又有如下的类型：

1.  metadataType：存储的是元数据（metadata），每个WAL文件开头都有这类型的一条记录数据。
2.  entryType：保存的是raft的数据，也就是客户端提交上来并且已经commit的数据。
3.  stateType：保存的是当前集群的状态信息，即前面提到的HardState。
4.  crcType：校验数据。
5.  snapshotType：快照数据。

etcd使用两个目录分别存放WAL文件以及快照文件。其中，WAL文件的文件名格式是“16位的WAL文件编号-该WAL第一条entry数据的index号.wal”，这样就能从WAL文件名知道该WAL文件中保存的entry数据至少大于什么索引号。而快照文件名的格式则是“16位的快照数据最后一条日志记录任期号-16位的快照数据最后一条记录的索引号.snap”。

Etcd会管理WAL目录中的所有WAL文件，但是在生成快照文件之后，在快照数据之前的WAL文件将被清除掉，保证磁盘不会一直增长。

比如当前etcd中有三个WAL文件，可以从这些文件的文件名知道其中存放数据的索引范围。

![etcd wal1](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-wal1.png "etcd wal1")

在生成快照文件之后，此时就只剩一个WAL文件和一个快照文件了：

![etcd wal2](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-wal2.png "etcd wal2")

那么，又是在什么情况下生成快照文件呢？Etcdserver在主循环中通过监听channel获知当前raft协议返回的Ready数据，此时会做判断如果当前保存的快照数据索引距离上一次已经超过一个阈值（EtcdServer.snapCount），此时就从raft的存储中生成一份当前的快照数据，写入快照文件成功之后，就可以将这之前的WAL文件释放了。

以上流程和对应的具体函数见下面的流程图。

![etcd snapshot](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-snapshot.png "etcd snapshot")

# backend store的实现

## revision概念

Etcd存储数据时，并不是像其他的KV存储那样，存放数据的键做为key，而是以数据的revision做为key，键值做为数据来存放。如何理解revision这个概念，以下面的例子来说明。

比如通过批量接口两次更新两对键值，第一次写入数据时，写入<key1,value1>和<key2,value2>，在Etcd这边的存储看来，存放的数据就是这样的：

```ini
  revision={1,0}, key=key1, value=value1
  revision={1,1}, key=key2, value=value2
```

而在第二次更新写入数据<key1,update1>和<key2,update2>后，存储中又记录（注意不是覆盖前面的数据）了以下数据：

```ini
  revision={2,0}, key=key1, value=update1
  revision={2,1}, key=key2, value=update2
```

其中revision有两部分组成，第一部分成为main revision，每次事务递增1；第二部分称为sub revision，一个事务内的一次操作递增1。 两者结合，就能保证每次key唯一而且是递增的。

![etcd revision](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-revision.png "etcd revision")

但是，就客户端看来，每次操作的时候是根据Key来进行操作的，所以这里就需要一个Key映射到当前revision的操作了，为了做到这个映射关系，Etcd引入了一个内存中的Btree索引，整个操作过程如下面的流程所示。

![etcd keyindex](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-keyindex.png "etcd keyindex")

查询时，先通过内存中的btree索引来查询该key对应的keyIndex结构体，然后再根据这个结构体才能去boltdb中查询真实的数据返回。

所以，下面先展开讨论这个keyIndex结构体和btree索引。

## keyIndex结构

keyIndex结构体有以下成员：

-   key：存储数据真实的键。
-   modified：最后一次修改该键对应的revision。
-   generations：generation数组。

如何理解generation结构呢，可以认为每个generation对应一个数据从创建到删除的过程。每次删除key的操作，都会导致一个generation最后添加一个tombstone记录，然后创建一个新的空generation记录添加到generations数组中。

generation结构体存放以下数据：

-   ver：当前generation中存放了多少次修改，其实就是revs数组的大小-1（因为需要去掉tombstone）。
-   created：创建该generation时的revision。
-   revs：存放该generation中存放的revision数组。

以下图来说明keyIndex结构体：

![etcd keyindex struct](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-keyindex-struct.png "etcd keyindex struct")

如上图所示，存放的键为test的keyIndex结构。

它的generations数组有两条记录，其中generations[0]在revision 1.0时创建，当revision2.1的时候进行tombstone操作，因此该generation的created是1.0；对应的generations[1]在revision3.3时创建，紧跟着就做了tombstone操作。

所以该keyIndex.modifiled成员存放的是3.3，因为这是这条数据最后一次被修改的revision。

一个已经被tombstone的generation是可以被删除的，如果整个generations数组都已经被删除空了，那么整个keyIndex记录也可以被删除了。

![etcd generation_compact](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-generation-compact.png "etcd generation compact")

如上图所示，keyIndex.compact(n)函数可以对keyIndex数据进行压缩操作，将删除满足main revision < n的数据。

-   compact(2)：找到了generations[0]的1.0 revision的数据进行了删除。
-   compact(3)：找到了generations[0]的2.1 revision的数据进行了删除，此时由于generations[0]已经没有数据了，所以这一整个generation被删除，原先的generations[1]变成了generations[0]。
-   compact(4)：找到了generations[0]的3.3 revision的数据进行了删除。由于所有的generation数据都被删除了，此时这个keyIndex数据可以删除了。

## treeIndex结构

Etcd中使用treeIndex来在内存中存放keyIndex数据信息，这样就可以快速的根据输入的key定位到对应的keyIndex。

treeIndex使用开源的github.com/google/btree来在内存中存储btree索引信息，因为用的是外部库，所以不打算就这部分做解释。而如果很清楚了前面keyIndex结构，其实这部分很好理解。

所有的操作都以key做为参数进行操作，treeIndex使用btree根据key查找到对应的keyIndex，再进行相关的操作，最后重新写入到btree中。

## store

前面讲到了WAL数据的存储、内存索引数据的存储，这部分讨论持久化存储数据的模块。

etcd V3版本中，使用BoltDB来持久化存储数据（etcd V2版本的实现不做讨论）。所以这里先简单解释一下BoltDB中的相关概念。

### BoltDB相关概念

BoltDB中涉及到的几个数据结构，分别为DB、Bucket、Tx、Cursor、Tx等。

其中：

-   DB：表示数据库，类比于Mysql。
-   Bucket：数据库中的键值集合，类比于Mysql中的一张数据表。
-   键值对：BoltDB中实际存储的数据，类比于Mysql中的一行数据。
-   Cursor：迭代器，用于按顺序遍历Bucket中的键值对。
-   Tx：表示数据库操作中的一次只读或者读写事务。

### Backend与BackendTx接口

Backend和BackendTx内部的实现，封装了BoltDB，太简单就不做分析了。

### Lessor接口

etcd中没有提供针对数据设置过期时间的操作，通过租约（Lease）来实现数据过期的效果。而Lessor接口就提供了管理租约的相关接口。

比如，使用etcdctl命令可以创建一个lease：

`etcdctl lease grant 10 lease 694d67ed2bfbea03 granted with TTL(10s)`

这样就创建了一个ID为694d67ed2bfbea03的Lease，此时可以将键值与这个lease进行绑定：

`etcdctl put --lease=694d67ed2bfbea03 a b`

当时间还没超过过期时间10S时，能通过etcd拿到这对键值的数据。如果超时了就获取不到数据了。

从上面的命令可以看出，一个Lease可以与多个键值对应，由这个Lease通过管理与其绑定的键值数据的生命周期。

etcd中，将Lease ID存放在名为“lease”的Bucket中，注意在这里只存放Lease相关的数据，其键值为：<Lease ID，序列化后的Lease数据包括TTL、ID>，之所以不存放与Lease绑定的键值，是因为这些键值已经存放到另外的Bucket里了，写入数据的时候也会将这些键值绑定的Lease ID写入，这样在恢复数据的时候就可以将键值与Lease ID绑定的关系写入内存中。

即：Lease这边需要持久化的数据只有Lease ID与TTL值，而键值对这边会持久化所绑定的Lease ID，这样在启动恢复的时候可以将两者对应的关系恢复到内存中。

![etcd lease_store](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-lease-store.png "etcd lease store")

明白了以上关系再来理解Lessor的实现就很简单了。

lessor中主要包括以下的成员：

-   leaseMap map[LeaseID]*Lease：存储LeaseID与Lease实例之间的对应关系。
-   itemMap map[LeaseItem]LeaseID：leaseItem实际存放的是键值，所以这个map管理的就是键值与Lease ID之间的对应关系。
-   b backend.Backend：持久化存储，每个Lease的持久化数据会写入名为“lease”的Bucket中。
-   minLeaseTTL int64：最小过期时间，设置给每个lease的过期时间不得小于这个数据。
-   expiredC chan []*Lease：通过这个channel通知外部有哪些Lease过期了。

其他的就很简单了:

1.  lessor启动之后会运行一个goroutine协程，在这个协程里定期查询哪些Lease超时，超时的Lease将通过expiredC channel通知外部。
2.  而针对Lease的CRUD操作，都需要进行加锁才能操作。

## KV接口

有了以上的准备，可以开始分析数据存储相关的内容了。在etcd V3中，所有涉及到数据的存储，都会通过KV接口。

store结构体实现了KV接口，其中最重要的就是封装了前面提到的几个数据结构：

-   b backend.Backend：用于将持久化数据写入BoltDB中。
-   kvindex index：保存key索引。
-   changes []mvccpb.KeyValue：保存每次写操作之后进行了修改的数据，用于通知watch了这些数据变更的客户端。

在store结构体初始化时，根据传入的backend.Backend，初始化backend.BatchTx结构，后面的任何涉及到事务的操作，都可以通过这个backend.BatchTx来进行。

其实有了前面的准备，理解store结构做的事情已经不难，以一次Put操作为例，其流程主要如下图所示：

![etcd store_put](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-store-put.png "etcd store put")

# applierV3

EtcdServer内部实现中，实际使用的是applierV3接口来进行持久化数据的操作。

这个接口有以下几个实现，但是其中applierV3backend的实现是最重要的，其内部使用了前面提到的KV接口来进行数据的处理。

另外，applierV3接口还有其他几个实现，这里分别列举一下。

-   applierV3backend：基础的applierV3接口实现，其他几个实现都在此实现上做功能扩展。内部调用EtcdServer中的KV接口进行持久化数据读写操作。
-   applierV3Capped：磁盘空间不足的情况下，EtcdServer中的applierV3切换到这个实现里面来，这个实现的任何写入操作都会失败，这样保证底层存储的数据量不再增加。
-   authApplierV3：在applierV3backend的基础上扩展出权限控制的功能。
-   quotaApplierV3：在applierV3backend的基础上加上了限流功能，即底层的存储到了上限的话，会触发限流操作。

# 综述

下图将上面涉及到的关键数据结构串联在一起，看看EtcdServer在收到Raft库通过Ready channel通知的可以持久化数据之后，都做了什么操作。

![etcd raft_flow](https://www.codedump.info/media/imgs/20181125-etcd-server/etcd-raft-flow.png "etcd raft flow")

1.  raft库通过Ready Channel通知上层的raftNode哪些数据可以进行持久化。
2.  raftNode启动之后也是会启动一个Goroutine来一直监听这个Ready Channel，以便收到可以持久化数据的通知。
3.  raftNode在收到Ready数据之后，将首先写入WAL日志中。这里的WAL日志由storage结构体来管理，分为两大部分：WAL日志以及WAL快照文件数据Snapshotter，后者用来避免WAL文件一直增大。
4.  raftNode在写WAL数据完成之后，通过apply Channel通知EtcdServer。
5.  EtcdServer启动之后也是启动一个Goroutine来监听这个channel，以便收到可以持久化数据的通知。
6.  EtcdServer通过调用applierV3接口来持久化数据。applierV3backend结构体实现applierV3接口, applierV3backend结构体实现applierV3接口，内部通过调用KV接口进行持久化操作。而在实现KV接口的store结构体中，treeIndex负责在内存中维护数据键值与revision的对应关系即keyIndex数据，Backend接口负责持久化数据，最后持久化的数据将落盘到BoltDB中。



# Etcd Raft库的工程化实现

# 概述

在开始展开讨论前，先介绍这个Raft论文中的示意图，我认为能理解这幅图才能对一致性算法有个全貌的了解：

![Etcd Raft与应用层的交互](https://www.codedump.info/media/imgs/20210515-raft/statemachine.jpeg "Etcd Raft与应用层的交互")

图中分为两种进程：

-   server进程：server进程中运行着一致性算法模块、持久化保存的日志、以及按照日志提交的顺序来进行顺序操作的状态机。
-   client进程：用于向server提交日志的进程。

需要说明的是，两种进程都用叠加的矩形来表示，意指系统中这两类进程不止一个。

一个日志要被正确的提交，图中划分了几步：

1、client进程提交数据到server进程，server进程将收到的日志数据灌入一致性模块。

2、一致性模块将日志写入本地WAL，然后同步给集群中其他server进程。

3、多个节点对某条日志达成一致之后，将修改本地的提交日志索引（commit index）；落盘后的日志按照顺序灌入状态机，只要保证所有server进程上的日志顺序，那么最后状态机的状态肯定就是一致的了。

4、灌入状态机之后，server进程可以应答客户端。

所以，本质上，一个使用了一致性算法的库，划分了成了两个不同的模块：

-   一致性算法库，这里泛指Raft、Paxos、Zab等一致性协议。这类一致性算法库主要做如下的事情：
    -   用户输入库中日志（log），由库根据各自的算法来检测日志的正确性，并且通知上层的应用层。
        -   输入到库中的日志维护和管理，算法库中需要知道哪些日志提交、提交成功、以及上层的应用层已经applied过的。当发生错误的时候，某些日志还会进行回滚（rollback）操作。
    -   日志的网络收发，这部分属于可选功能。有一些库，比如braft把这个事情也揽过来自己做了，优点是使用者不需要关注这部分功能，缺点是braft和它自带的网络库brpc耦合的很紧密，不可能拆开来使用；另一些raft实现，比如这里重点提到etcd raft实现，并不自己完成网络数据收发的工作，而是通知应用层，由应用层自己实现。
    -   日志的持久化存储：这部分也属于可选功能。前面说过，一致性算法库中维护了未达成一致的日志缓冲区，达成一致的日志才通知应用层，因此在这里不同的算法库又有了分歧，braft也是自己完成了日志持久化的工作，etcd raft则是将这部分工作交给了应用层。
-   应用层：即工作在一致性算法之上的库使用者，这个就比上图中的“状态机”：只有达成一致并且落盘的数据才灌入应用层，只要保证灌入应用层的日志顺序一致那么最后的状态就是一致的。

总体来看，一个一致性算法库有以下必选和可选功能：

-   输入日志进行处理的算法（必选）。
-   日志的维护和管理（必选）。
-   日志（包括快照）数据的网络收发（可选）。
-   日志（包括快照）的持久化存储（可选）。

需要特别说明的是，即便是后面两个工作是可选的，但是可选还是必选的区别在于，这部分工作是一致性算法库自己完成，还是由算法库通知给上面的应用层去完成，并不代表这部分工作可以完全不做。

在下表中列列举了etcd raft和braft在这几个特性之间的区别：

![[Pasted image 20230423115555.png]]

两种实现各有自己的优缺点，braft类实现更适合提供一个需要集成raft的服务时，可以直接用来实现服务；etcd raft类的实现，由于与网络、存储层耦合不紧密，易于进行测试，更适合拿来做为库使用。

如果把前面的一致性算法的几个特性做一个抽象，我认为一致性算法库本质上就是一个“维护操作日志的算法库，只要大家都按照相同的顺序将日志灌入应用层”就好，其工作原理大体如下图：

![一致性算法的本质](https://www.codedump.info/media/imgs/20210515-raft/co-algo.png "一致性算法的本质")

如果把问题抽象成这样的话，那么本质上，所谓的“一致性算法库”跟一个经常看到的tcp、kcp甚至是一个应用层的协议栈也就没有什么区别了：

-   大家都要维护一个数据区：只有确认过正确的，才会抛给上一层。以TCP协议算法来说，比如发送但未确认的数据由协议栈的缓冲区维护，如果超时还未等到对端的确认，将发起超时重传等，这些都是每种协议算法的具体细节，但是本质上这些协议都要维护一个未确认数据的缓冲区。一致性算法在数据的维护上会更复杂一些，一是参与确认的节点不止通信的C/S两端，需要集群中半数以上节点的确认；同时，在未确认之前日志需要首先落盘，在提交成功之后再抛给应用层。
-   只要保证所有参与的节点，都以相同的数据灌入日志给应用层，那么得到的结果将最终一致。
-   确认的流程是可以pipeline异步化的，提交日志的进程并不需要一直等待日志被提交成功，而是提交之后等待。不妨以下面的流程来做解释：

![流水线异步化的日志提交流程](https://www.codedump.info/media/imgs/20210515-raft/pipeline.png "流水线异步化的日志提交流程")

其中：

-   clientA和clientB分别提交了两条日志数据，但是并没有阻塞等待日志提交成功，而是提交之后就继续别的操作了。
-   server将两条日志数据同步出去，达成一致之后再分别通知两个client日志提交成功。

在这里，client上通知日志提交成功的机制可以有很多，以etcd来说，会给每个提交的日志对应一个channel，提交成功之后会通过这个channel进行通知，也会给这个日志加一个定时器，超过时间仍未收到通知则认为提交失败。

# etcd raft的实现

有了上面对一致性算法库的大体了解，下面可以详细看看etcd raft的实现了。

## 概述

前面提到过，etcd raft库的实现中，并不自己实现网络数据收发、提交成功的数据持久化等工作，这些工作留给了应用层来自己实现，所以需要一个机制来通知应用层。etcd raft中将需要通知给应用层的数据封装在`Ready`结构体中，其中包括如下的成员：

![[Pasted image 20230423115724.png]]

有了数据，还需要raft线程与上面的应用层线程交互的机制，这部分封装在`node`结构体中。

`node`结构体实现`Node`接口，该接口用于表示Raft集群中的一个节点。在`node`结构体中，实现了以下几个核心的channel，由于与外界进行通信：

-   propc chan pb.Message：用于本地提交日志数据的channel。
-   recvc chan pb.Message：用于接收来自集群中其他节点日志数据的channel。
-   readyc chan Ready：用于本地Raft库通知应用层哪些数据已经准备好了，因此应用层需要关注readyc这个channel才能获得从Raft线程中提交的数据。

去掉一些不太重要的接口，`Node`接口中有如下的核心函数：

-   Tick()：应用层每次tick时需要调用该函数，将会由这里驱动raft的一些操作比如选举等，至于tick的单位是多少由应用层自己决定，只要保证是恒定时间都会来调用一次就好了。
-   Propose(ctx context.Context, data []byte) error：提议写入数据到日志中，可能会返回错误。
-   Step(ctx context.Context, msg pb.Message) error：将消息msg灌入状态机。
-   Ready() <-chan Ready：返回通知`Ready`结构体变更的channel，应用层需要关注这个channel，当发生变更时将其中的数据进行操作。
-   Advance()：Advance函数是当使用者已经将上一次Ready数据处理之后，调用该函数告诉raft库可以进行下一步的操作。

在`node`结构体的实现中，无论是通过`Propose`函数还是`Step`函数提交到Raft算法库的消息，最终都是调用内部的`step`函数的。

![Etcd Raft与应用层的交互](https://www.codedump.info/media/imgs/20210515-raft/etcd-raft.png "Etcd Raft与应用层的交互")

以上图来说明应用层与raft之间的交互流程，注意：etcd的实现中，raft是一个独立的线程，与应用层之间通过上面介绍的几个channel进行交互。

-   首先看最中间的部分，本地提交的数据通过`propc`channel通知raft线程，而应用层从外部网络接收到的日志数据通过`recvc`通知raft线程。但是不管是哪个channel，最终都是通过上面提到的`step`函数将日志数据灌入raft线程中。
-   最右边是raft线程通知应用线程有哪些日志数据已经确认提交完毕等（`Ready`结构体中不限于确认提交数据，该类型数据在上面已经列举出来），应用层可以通过`Ready`数据来持久化数据等操作。
-   最左边表示应用层线程要通过`Advance`函数通知raft线程自己已经持久化了某些数据，这时候可以推动raft线程库中的日志缓冲区的变更。

以一个简单的消息流程来继续解释上面的流程：

-   应用层收到索引为N的消息，此时通过`recvc`channel提交给Raft线程。
-   Raft线程验证消息是正确的，于是需要广播给集群中的其他节点，此时会：
    -   首先在Raft的日志缓冲区中保存下来这个消息，因为这个日志还未提交成功。
    -   将日志消息放入`Ready`结构体的`Messages`成员中，通知应用层，这样应用层就将该成员中的消息转发给集群中的其他节点。
-   Raft线程继续获得从应用层下发下来的消息，当发现下发的消息中，索引为N的消息已经被集群中半数以上的节点确认过，此时就可以认为该消息能被持久化了，将日志消息放入`Ready`结构体的`CommittedEntries`成员中，以通知应用层该消息可以被持久化了。
-   每次应用层持久化了某些消息之后，都会通过`Advance`函数通知Raft线程，这样Raft线程可以将这部分已经被持久化的消息从消息缓冲区中删除，因为前面提到过消息缓冲区仅仅是用来保存还未持久化的消息的。

这个工作流程是pipeline化，即应用层某一次提交了索引为N的消息，并不需要一直等待该消息提交成功，而是可以返回继续做别的事情，当raft线程判断消息可以被提交时，再通过`Ready`结构体来通知应用层。

以上大体描述了etcd中，应用层线程与raft线程的交互流程，下面详细看看raft线程的实现。

## Raft算法

raft算法中，有不同的角色存在：candidate、follower、leader，本质上Raft算法是输入日志数据进行处理，而每种角色对不同类型的日志数据需要有不同的处理。

所以，etcd raft的实现中，针对三种不同的角色，通过修改函数指针的方式在切换了不同角色时的处理，如下图所示：

![不同角色的Raft算法处理](https://www.codedump.info/media/imgs/20210515-raft/role-algo.png "不同角色的Raft算法处理")

具体的算法细节，不打算在本文中展开，可以回头上上面给出来的几篇文章。

## 数据管理

数据管理分为以下几部分阐述：

-   未持久化数据缓冲区
-   持久化数据内存映像
-   数据的持久化
-   数据流动的全流程
-   节点进度的管理

下面一一展开。

### 未持久化数据缓冲区

前面提到过，Raft算法中还必须要做的是维护未确认数据的缓冲区数据，每当其中的一部分数据被确认，缓冲区的窗口随之发生移动，这就类似TCP协议算法中的滑动窗口。

etcd raft中，管理未确认数据放在了`unstable`结构体（log_unstable.go）中，其内部维护三个成员：

-   snapshot \*pb.Snapshot：保存还没有持久化的快照数据
-   entries []pb.Entry：保存还未持久化的日志数据。
-   offset uint64：保存快照和日志数组的分界线。

可以看到，未持久化数据分为两部分：一部分是快照数据snapshot，另一部分就是日志数据数组。两者不会同时存在，快照数据只会在启动时进行快照数据恢复时存在，当应用层使用快照数据进行恢复之后，raft切换为可以接收日志数据的状态，后续的日志数据都会写到`entrise`数组中了，而两者的分界线就是`offset`变量。

![未持久化数据](https://www.codedump.info/media/imgs/20210515-raft/unstable.png "未持久化数据")

由于是”未持久化数据的缓冲区“，因此这其中的数据可能会发生回滚（rollback）现象，因此`unstable`结构体需要支持能回滚的操作，见函数`truncateAndAppend`：

```go
func (u *unstable) truncateAndAppend(ents []pb.Entry) {
	// 先拿到这些数据的第一个索引
	after := ents[0].Index
	switch {
	case after == u.offset+uint64(len(u.entries)):
		// 如果正好是紧接着当前数据的，就直接append
		// after is the next index in the u.entries
		// directly append
		u.entries = append(u.entries, ents...)
	case after <= u.offset:
		u.logger.Infof("replace the unstable entries from index %d", after)
		// The log is being truncated to before our current offset
		// portion, so set the offset and replace the entries
		// 如果比当前偏移量小，那用新的数据替换当前数据，需要同时更改offset和entries
		u.offset = after
		u.entries = ents
	default:
		// truncate to after and copy to u.entries
		// then append
		// 到了这里，说明 u.offset < after < u.offset+uint64(len(u.entries))
		// 那么新的entries需要拼接而成
		u.logger.Infof("truncate the unstable entries before index %d", after)
		u.entries = append([]pb.Entry{}, u.slice(u.offset, after)...)
		u.entries = append(u.entries, ents...)
	}
}
```

函数中分为三种情况：

-   如果传入的日志数据，刚好跟当前数据紧挨着（after == u.offset+uint64(len(u.entries))），就可以直接进行append操作。
-   如果传入的日志数据的第一条数据索引不大于当前的offset（after <= u.offset），说明数据发生了回滚，直接用新的数据替换旧的数据。
-   其他情况，说明u.offset < after < u.offset+uint64(len(u.entries))，这是新的未持久化数据由这两部分数据各取其中一部分数据拼装而成。

### 持久化数据内存映像

但是，仅仅有未持久化数据还不够，有时候有一些数据已经落盘，但是还需要进行查询、读取等操作。于是，etcd raft又提供了一个`Storage`接口，该接口有面对不同的组件有不同的行为：

-   对于Raft库，该接口仅仅只有读操作。（如下图中的黄色函数）
-   对于etcd 服务来说，还提供了写操作，包括：增加日志数据、生成快照、压缩数据。（如下图中的蓝色函数）

因此，这个接口及其默认实现`MemoryStorage`，呈现了稍微不太一样的行为，以致于我最开始没有完全理解：

![持久化数据的内存映像](https://www.codedump.info/media/imgs/20210515-raft/stable.png "持久化数据的内存映像")

因为持久化数据的内存映像，提供给Raft库的仅仅只需要读操作，所以`Storage`接口就只有读操作，多出来的写操作只会在应用层中才会用到，因此这些写接口并没有放在公用的接口中。

了解了持久化和未持久化数据的表示之后，etcd raft库将两者统一到`raftLog`这个结构体中：

![不同视角下的raftlog](https://www.codedump.info/media/imgs/20210515-raft/raftlog.png "不同视角下的raftlog")

### 数据的持久化

以上解释两种缓冲区的作用，数据最终还是需要持久化到磁盘上的，那么，这个持久化数据的时机在哪里？

答案：当客户端提交数据时，etcd Raft库就通过`Ready`结构体的`Entries`成员通知应用层，将这些提交的数据进行持久化了。

有代码为证。

首先来看`raft`中如何生成`Ready`数据：

```go
func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
	rd := Ready{
		// entries保存的是没有持久化的数据数组
		Entries:          r.raftLog.unstableEntries(),
		// 保存committed但是还没有applied的数据数组
		CommittedEntries: r.raftLog.nextEnts(),
		// 保存待发送的消息
		Messages:         r.msgs,
	}
	// ...
}
```

可以看到，`raft`库中将未持久化数据塞到了`Entries`数组中，而已经达成一致可以提交的日志数据放入到`CommittedEntries`数组中。

以`etcd`代码中自带的`raftexample`目录中的例子代码来看应用层在收到`Ready`数据后的做法：

```scss
func (rc *raftNode) serveChannels() {
		case rd := <-rc.node.Ready():
			// 将HardState，entries写入持久化存储中
			rc.wal.Save(rd.HardState, rd.Entries)
			if !raft.IsEmptySnap(rd.Snapshot) {
				// 如果快照数据不为空，也需要保存快照数据到持久化存储中
				rc.saveSnap(rd.Snapshot)
				rc.raftStorage.ApplySnapshot(rd.Snapshot)
				rc.publishSnapshot(rd.Snapshot)
			}
			rc.raftStorage.Append(rd.Entries)
			rc.transport.Send(rd.Messages)
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
			}
			rc.maybeTriggerSnapshot()
			rc.node.Advance()
	// ....
}
```

其中：

-   rc.wal.Save(rd.HardState, rd.Entries)：将客户端提交数据的数据写入wal中。
-   rc.raftStorage.Append(rd.Entries)：这里的`raftStorage`即前面提到的持久化数据缓冲区的`Storage`接口，由`MemoryStorage`接口实现，这一步将这些客户端提交的数据也写入持久化缓冲区的内部映像。
-   rc.publishEntries(rc.entriesToApply(rd.CommittedEntries))：这个调用分为两步，第一步调用`entriesToApply`是要从已达成一致的日志数据中过滤出真正可以进行apply的日志，因为里面的一些日志可能已经被应用层apply过，第二步将第一步过滤出来的日志数据通知给应用层。在`raftexample`这个示例代码中，最终这些已经达成一致的数据，会被遍历生成KV内存数据。

这里有一个问题：客户端提交过来的数据，还未达成集群内半数节点的一致，这时候就去做落盘操作，如果提交过程中发现出了问题，实际这条数据并不能最终达成一致，那么已落盘的数据怎么办？

在这里，`etcd`落盘客户端提交的数据时，是写入到WAL文件中的，后面发生了错误，如leader变成了follower时，日志需要进行了回滚操作等，也还是将那些正确的日志继续添加到WAL日志后面，服务如果重启，就是把这些日志按照顺序重放（replay）一遍，这里不可避免的会有一些冗余的操作，但是随着快照文件的产生，这个问题已经不大了。

其次，不论是前面提到的`未持久化数据缓冲区`，还是`持久化数据缓冲区`，在往缓冲区中添加日志的函数实现中，都会去判断日志是否发生了回滚，会将当前传入的日志按照正确的日志索引放到缓冲区合适的位置。`未持久化数据缓冲区`这部分操作在函数`unstable.truncateAndAppend`中，`持久化数据缓冲区`这部分操作在函数`MemoryStorage.Append`中，感兴趣的可以去看看具体的实现，在这里就不再展开了。

### 数据流动的全流程

以上解释了客户端提交的数据在两个缓冲区、持久化存储、以及最终达成一致之后给应用层的过程，下面以例子分别来解释客户端提交数据的流程以及快照数据恢复的流程。

#### 客户端提交数据的流程

![客户端提交数据的流动](https://www.codedump.info/media/imgs/20210515-raft/wal.png "客户端提交数据的流动")

1、客户端提交数据给服务器。

2、接着看这条数据走过的”存储“路径：

```go
1、首先，`raft`库会首先将日志数据写入`未持久化数据缓冲区`。

2、由于`未持久化数据缓冲区`中有新增的数据，会通过`Ready`结构体通知给应用层。

3、应用层收到`Ready`结构体之后，将其中的数据写入WAL持久化存储，然后更新这块数据到`已持久化数据缓冲区`。

4.1、持久化完毕后，应用层通过`Advance`接口通知`Raft`库这些数据已经持久化，于是raft库修改`未持久化数据缓冲区`将客户端刚提交的数据从这个缓冲区中删除。

4.2、持久化完毕之后，除了通知删除`未持久化数据缓冲区`，还讲数据通过网络同步给集群中其他节点。
```

3、集群中半数节点对该提交数据达成了一致，可以应答给客户端了。

#### 启动时使用快照数据恢复流程

下面以例子来实际解释etcd raft中数据在未持久化缓存、wal日志、持久化数据内容映像中的流动：

1、节点N启动，加入到集群中，此时发现N上面没有数据，于是集群中的leader节点会首先通过rpc消息将快照数据发送给节点N。

2、节点N收到快照数据，首先会保存到未持久化数据缓存中。

3、Raft通过`Ready`结构体通知应用层有快照数据。

4、应用层（也就是etcdserver）将快照数据写入wal持久化存储中，这一步可以理解为将快照数据落盘。

5、落盘之后，调用`MemoryStorage`结构体的`ApplySnapshot`将快照数据保存到持久化数据内存映像中。

6、（图中未给出）调用Raft库的`Advance`接口通知raft库传递过来的`Ready`结构体数据已经操作完毕，这时候对应的，raft库就会把第二步中保存到未持久化数据缓存的快照数据给删除了。

![快照数据的流动](https://www.codedump.info/media/imgs/20210515-raft/snapshot.png "快照数据的流动")

以上是快照数据的流动过程，在节点N接收并持久化快照数据后，后面就可以接收正常的日志了，日志数据的流动过程跟快照数据实际是差不多的，就不再阐述了。

从上面的流程中也可以看出，应用层也就是etcdserver的持久化数据，只有wal日志而已，情况确实是这样的，其接口和实现如下：

```go
type Storage interface {
	// Save function saves ents and state to the underlying stable storage.
	// Save MUST block until st and ents are on stable storage.
	Save(st raftpb.HardState, ents []raftpb.Entry) error
	// SaveSnap function saves snapshot to the underlying stable storage.
	SaveSnap(snap raftpb.Snapshot) error
	// DBFilePath returns the file path of database snapshot saved with given
	// id.
	DBFilePath(id uint64) (string, error)
	// Close closes the Storage and performs finalization.
	Close() error
}

type storage struct {
	*wal.WAL
	*snap.Snapshotter
}
```

`Storage`接口是etcdserver持久化数据的接口，其保存的数据有两个接口：

-   Save(st raftpb.HardState, ents []raftpb.Entry) error：保存日志数据。
-   SaveSnap(snap raftpb.Snapshot) error：保存快照数据。

而`Storage`接口由下面的`storage`结构体来实现，其又分为两部分：

-   wal：用于实现WAL日志的读写。
-   snap：用于实现快照数据的读写。

这里就不展开讨论了。

### 节点进度的管理

前面提到过，一致性算法与TCP之类的协议，本质上都需要管理未确认数据的缓冲区，但是不同的是，参与一致性算法确认的成员，不会像一般的点对点通信协议那样只有两个，在raft算法中，leader节点除了要维护未持久化缓冲区之外，还需要维护一个数据结构，用于保存集群中其他节点的进度，这部分数据在etcd raft中保存在结构体`Progress`中，我将我之前阅读过程中加上的注释一并贴出来：

```go
// 该数据结构用于在leader中保存每个follower的状态信息，leader将根据这些信息决定发送给节点的日志
// Progress represents a follower’s progress in the view of the leader. Leader maintains
// progresses of all followers, and sends entries to the follower based on its progress.
type Progress struct {
	// Next保存的是下一次leader发送append消息时传送过来的日志索引
	// 当选举出新的leader时，首先初始化Next为该leader最后一条日志+1
	// 如果向该节点append日志失败，则递减Next回退日志，一直回退到索引匹配为止

	// Match保存在该节点上保存的日志的最大索引，初始化为0
	// 正常情况下，Next = Match + 1
	// 以下情况下不是上面这种情况：
	// 1. 切换到Probe状态时，如果上一个状态是Snapshot状态，即正在接收快照，那么Next = max(pr.Match+1, pendingSnapshot+1)
	// 2. 当该follower不在Replicate状态时，说明不是正常的接收副本状态。
	//    此时当leader与follower同步leader上的日志时，可能出现覆盖的情况，即此时follower上面假设Match为3，但是索引为3的数据会被
	//    leader覆盖，此时Next指针可能会一直回溯到与leader上日志匹配的位置，再开始正常同步日志，此时也会出现Next != Match + 1的情况出现
	Match, Next uint64
	// State defines how the leader should interact with the follower.
	//
	// When in ProgressStateProbe, leader sends at most one replication message
	// per heartbeat interval. It also probes actual progress of the follower.
	//
	// When in ProgressStateReplicate, leader optimistically increases next
	// to the latest entry sent after sending replication message. This is
	// an optimized state for fast replicating log entries to the follower.
	//
	// When in ProgressStateSnapshot, leader should have sent out snapshot
	// before and stops sending any replication message.

	// ProgressStateProbe：在每次heartbeat消息间隔期最多发一条同步日志消息给该节点
	// ProgressStateReplicate：正常的接受副本数据状态。当处于该状态时，leader在发送副本消息之后，
	// 就修改该节点的next索引为发送消息的最大索引+1
	// ProgressStateSnapshot：接收副本状态
	State ProgressStateType
	// Paused is used in ProgressStateProbe.
	// When Paused is true, raft should pause sending replication message to this peer.
	// 在状态切换到Probe状态以后，该follower就标记为Paused，此时将暂停同步日志到该节点
	Paused bool

	// PendingSnapshot is used in ProgressStateSnapshot.
	// If there is a pending snapshot, the pendingSnapshot will be set to the
	// index of the snapshot. If pendingSnapshot is set, the replication process of
	// this Progress will be paused. raft will not resend snapshot until the pending one
	// is reported to be failed.
	// 如果向该节点发送快照消息，PendingSnapshot用于保存快照消息的索引
	// 当PendingSnapshot不为0时，该节点也被标记为暂停状态。
	// raft只有在这个正在进行中的快照同步失败以后，才会重传快照消息
	PendingSnapshot uint64

	// RecentActive is true if the progress is recently active. Receiving any messages
	// from the corresponding follower indicates the progress is active.
	// RecentActive can be reset to false after an election timeout.
	RecentActive bool

	// inflights is a sliding window for the inflight messages.
	// Each inflight message contains one or more log entries.
	// The max number of entries per message is defined in raft config as MaxSizePerMsg.
	// Thus inflight effectively limits both the number of inflight messages
	// and the bandwidth each Progress can use.
	// When inflights is full, no more message should be sent.
	// When a leader sends out a message, the index of the last
	// entry should be added to inflights. The index MUST be added
	// into inflights in order.
	// When a leader receives a reply, the previous inflights should
	// be freed by calling inflights.freeTo with the index of the last
	// received entry.
	// 用于实现滑动窗口，用来做流量控制
	ins *inflights
}
```

总结来说，`Progress`结构体做的工作：

-   维护follower节点的match、next索引，以便知道下一次从哪里开始同步数据。
-   维护着follower节点当前的状态。
-   同步快照数据的状态。
-   流量控制，避免follower节点超载。

具体的算法细节，就不在这里贴出了。

## 网络数据的收发以及日志的持久化

网络数据的收发以及日志的持久化，这两部分在etcd raft库中，并不是由raft库来实现，而是通过`Ready`结构体来通知应用层，由应用层来完成。

# 总结

这里将上面的几部分总结如下，有了整体的理解才能更好的了解细节：

![Raft算法几要素在etcd raft中的实现](https://www.codedump.info/media/imgs/20210515-raft/summary.png "Raft算法几要素在etcd raft中的实现")


# Reference
https://www.codedump.info/post/20180921-raft/
https://www.codedump.info/post/20180922-etcd-raft/#%E9%80%89%E4%B8%BE%E6%B5%81%E7%A8%8B

https://www.codedump.info/post/20181125-etcd-server/
