# Paxos算法
Google Chubby的作者Mike Burrows说过这个世界上只有一种一致性算法，那就是Paxos，其它的算法都是残次品。

Paxos在原作者的《Paxos Made Simple》中内容是比较精简的：

第一阶段

（a） 提议者选择一个提议编号n，并向大多数接受者发送一个编号n的准备请求。

（b） 如果承兑人收到的准备请求的编号n大于其已答复的任何准备请求的编号，则承兑人对该请求作出答复，并承诺不接受任何编号小于n且其已接受的编号最高的提案（如有）。

第二阶段

（a） 如果提案人从大多数接受人处收到对其准备请求（编号n）的响应，则它向这些接受人中的每一个发送一个接受请求，请求编号n的提案，其值为v，其中v是响应中编号最高的提案的值，或者如果响应报告没有提案，则v是任何值。

（b） 如果承兑人收到编号为n的提案的接受请求，则除非承兑人已对编号大于n的准备请求作出响应，否则接受该提案。

翻译一下：

Paxos问题指分布式系统中存在故障fault，但不存在恶意corrupt节点场景（消息可能丢失但不会造假）下的共识达成（Consensus）问题。

Paxos是第一个被证明的共识算法，**原理基于两阶段提交并进行扩展**。算法中将节点分为三种类型：

•倡议者proposer：提交一个提案，等待大家批准为结案，往往是客户端担任。
•接受者acceptor：负责对提案进行投票，往往服务器担任。提议超过半数的接受者投票及被选中。
•学习者learner：被告知提案结果，并与之统一，不参与投票过程。客户端和服务端都可担任。

每个节点在协议中可以担任多个角色。

Paxos的特点：

**•一个或多个节点可以提出提议。
•系统针对所有提案中的某个提案必须达成一致。
•最多只能对一个确定的提案达成一致。
•只要超过半数的节点存活且可互相通信，整个系统一定能达成一致状态。**
![[Pasted image 20230418151837.png]]
## Paxos的死锁情况
![[Pasted image 20230418151912.png]]

“活锁”的根本原因在于两个proposer交替提案，避免“活锁”的方式为，如果一个proposer通过accpter返回的消息知道此时有更高编号的提案被提出时，该proposer静默一段时间，而不是马上提出更高的方案，静默期长短为一个提案从提出到被接受的大概时间长度即可，静默期过后，proposer重新提案。系统中之所以要有主proposer的原因在于，如果每次数据更改都用paxos，那实在是太慢了，还是通过主节点下发请求这样来的快，因为省去了不必要的paxos时间。所以选择主proposer用paxos算法，因为选主的频率要比更改数据频率低太多。但是主proposer挂了咋整，整个集群就一直处于不可用状态，所以一般都用租约的方式，如果proposer挂了，则租约会过期，其它proposer就可以再重新选主，如果不挂，则主proposer自己续租。

小结：

Paxos协议最终解决什么问题？

当一个提议被多数派接受后，这个提议对应的值被Chosen（选定），一旦有一个值被Chosen，那么只要按照协议的规则继续交互，后续被Chosen的值都是同一个值，也就是这个Chosen值的一致性问题。

**Paxos 的目标：保证最终有一个提案会被选定，当提案被选定后，其他议员最终也能获取到被选定的提案。**

**Paxos 协议用来解决的问题可以用一句话来简化：将所有节点都写入同一个值，且被写入后不再更改。**

# Raft一致性算法

Raft算法是Paxos算法的一种简化实现。

包括三种角色：leader，candidate和follower。

**•follow:所有节点都以follower的状态开始，如果没有收到leader消息则会变成candidate状态。**

**•candidate：会向其他节点拉选票，如果得到大部分的票则成为leader，这个过程是Leader选举。**

**•leader：所有对系统的修改都会先经过leader。**

其有两个基本过程：

**•Leader选举：每个candidate随机经过一定时间都会提出选举方案，最近阶段中的票最多者被选为leader。**

**•同步log：leader会找到系统中log（各种事件的发生记录）最新的记录，并强制所有的follow来刷新到这个记录。**

Raft一致性算法是通过选出一个leader来简化日志副本的管理，例如日志项（log entry）只允许从leader流向follower。

下面是动画演示Raft，清晰理解Raft共识如何达成。

1.针对简化版拜占庭将军问题，Raft 解决方案

假设将军中没有叛军，信使的信息可靠但有可能被暗杀的情况下，将军们如何达成一致性决定？

Raft 的解决方案大概可以理解成 先在所有将军中选出一个大将军，所有的决定由大将军来做。选举环节：比如说现在一共有3个将军 A, B, C，每个将军都有一个随机时间的倒计时器，倒计时一结束，这个将军就会把自己当成大将军候选人，然后派信使去问其他几个将军，能不能选我为总将军？假设现在将军A倒计时结束了，他派信使传递选举投票的信息给将军B和C，如果将军B和C还没把自己当成候选人（倒计时还没有结束），并且没有把选举票投给其他，他们把票投给将军A，信使在回到将军A时，将军A知道自己收到了足够的票数，成为了大将军。在这之后，是否要进攻就由大将军决定，然后派信使去通知另外两个将军，如果在一段时间后还没有收到回复（可能信使被暗杀），那就再重派一个信使，直到收到回复。

# raft论文学习-raft basics & leader election

raft算法是一个分布式一致性算法，用来替代Paxos算法，因为Paxos算法太晦涩难懂，基于Paxos成熟的工程实践非常少。在2013年，斯坦福大学的Diego Ongaro和John Ousterhout发表了论文In Search of an Understandable Consensus Algorithm，raft算法就此诞生。随后，在2014年Diego Ongaro的博士论文CONSENSUS: BRIDGING THEORY AND PRACTICE中，对raft以及相关的一致性算法进行了系统的阐述。他们两人在设计raft算法时将可理解性放在了首位，在raft算法出现之后，出现多种语言的开源实现，像etcd中的raft是Go语言实现的。

In Search of an Understandable Consensus Algorithm论文中提到raft的核心有领导选举、日志、安全性和集群变更这几部分，每个部分内容不少，小编打算拆开来总结输出，本文是第一篇，主要讲论文中的raft basics和leader election两部分。

## raft基础

-   一个raft集群包含多个节点，节点的数量为「奇数」个

答：不仅是raft，还有Zookeeper等[分布式存储](https://cloud.tencent.com/product/cos?from=20065&from_column=20065)系统的节点个数都是奇数个，因为它们都是根据“少数服从多数”原则来达成一致。当集群中脑裂时，如果小集群的节点数相等，会造成没有大多数支持的局面，导致服务不可用。例如我们假定raft集群的节点数为偶数6，节点名称依次为node1,node2,node3,node4,node5和node6,当前集群中的leader节点为node1,当出现两个相同数量节点的网络分区，即每个分区有3个节点。如下图所示，分区1中由于日志节点无法满足绝大多数节点（4个及以上）导致无法commit，而分区2中无法选出新的leader节点，原因是候选者节点无法收到4个及以上节点的投票，直接导致服务不可用。如果集群中的节点是奇数个，不可能出现两分区中节点数相同的情况，只会存在要么node1所在分区是大多数节点区域，此时node1所在的分区可以继续正常运转，要么node1在少数节点区域，那么大多数节点分区会重新选出新的leader,也可以继续提供服务。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/512e11267d5b5efbbd54613b0d6d62e2.jpeg?imageView2/2/w/2560/h/7000)

---

-   raft节点是一个状态机，状态机一共有三种状态，每个节点处于leader/candidate/follower三种状态中的一个

答：raft集群中任意时刻最多只有一个leader节点，也就说集群中可以没有leader但不能有两个及以上的leader存在。raft协议通过约束条件保证集群中最多只有1个leader. 如果集群中没有leader，此时集群是无法提供服务的状态。在稳定（正常）状态下，集群中只有leader和follower角色，在非稳定状态即有选举竞选，此时集群中有candidate、follower和leader多种角色。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/25e94b2b8b89b98f06ff6a3f9ba2cfb4.jpeg?imageView2/2/w/2560/h/7000)

---

-   一次选举从开始到下一次新的选举开始对应一个任期，任期用连续的整数表示, 是单调递增的，raft把时间分割成了任意长度的任期。如果系统中没有选举，任期term是保持不变的。下图中有4个任期，蓝色表示处于选举过程中，是一个过渡状态，绿色表示系统中已有leader节点处于稳定状态，对应t3没有选举出leader，所以很快开始新的一轮选举term4.

![](https://ask.qcloudimg.com/http-save/yehe-9955853/0ecba9ac6a852913c05b36cd22a754d1.jpeg?imageView2/2/w/2560/h/7000)

答：任期的关键作用是解决leader的"脑裂“问题，假设集群中有5台机器，当前任期term为4，发生了网络分区，leader单独成为一个分区，其他4个node成为一个分区。由于其他4个follower收不到leader心跳消息，认为loader宕机了。4个follower会发起新一轮选举选出新leader, 此时term为5，开始向其他3个follower日志消息，很快网络分区恢复，之前的leader加入网络，此时网络出现了两个leader.但是当旧leader向其他follower节点发送数据时，follower发现日志里面的term=4比自己的term=5小，判断它为过期的leader,拒绝执行写入操作，同时告诉旧leader，你过期了。旧leader知道自己过期之后，自动回退成follower。从而保证了任意一个时刻只有一个leader是有效的。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/2b192a29c53e21e7dc9af908ec41ae8e.jpeg?imageView2/2/w/2560/h/7000)

5个节点构成的raft集群

![](https://ask.qcloudimg.com/http-save/yehe-9955853/62e91a1fad60fa2a78288b273411f5a8.jpeg?imageView2/2/w/2560/h/7000)

leader发生网络分区

![](https://ask.qcloudimg.com/http-save/yehe-9955853/4046487505a2d401dc04f593a10ba54a.jpeg?imageView2/2/w/2560/h/7000)

网络分区恢复

---

-   raft算法规定了节点之间采用RPC进行通信，一致性算法只需要两种类型的RPC消息，为了在[服务器](https://cloud.tencent.com/product/cvm?from=20065&from_column=20065)之间传输快照增加了第三种类型的RPC。

答：RequestVote(请求投票) RPC: 发起请求投票RPC消息，从candidate节点广播发往其他节点。
AppendEntries(追加日志)RPC,消息从leader节点发给follower节点， 用来日志和提供一种心跳机制。
此外，为了在服务器之间传输快照增加了第三种类型的RPC，称它为快照RPC,用于传输快照数据。

---

## leader选举

所有的节点在刚启动时都是follower角色，raft采用心跳机制触发选举。只要一个节点能够收到来自leader或candidate节点的RPC消息，就一直保持follower角色。leader会周期性的向所有follower节点发送心跳信息来捍卫自己的leader地位。心跳消息是一种不含有日志条目的AppendEntries RPC. 如果一个follower节点在一段选举超时的时间内没有收到任何消息，它就假定系统中没有leader, 然后开始进行选举尝试选出新的leader.

```go
// 创建一个raft对象
func newRaft(c *Config) *raft {
 ...
 // 开始时都是follower角色
 r.becomeFollower(r.Term, None)
       ...
 return r
}
```

选举流程是这样的：

1.  follower节点先增加自己当前的任期号并将状态切换到candidate状态
2.  给自己投一票
3.  并行地向集群中所有其他的节点发送投票请求(RequestVote RPC报文）
4.  保持candiate状态等待直接到遇到下面三种情况之一产生

4.1 情况1: 收到了节点回复中有过半的节点给”我“投票，这种情况”我“赢得了选举，将会切换到leader角色

4.2 情况2：收到了其他声称自己是leader的节点发送来的AppendEntries RPC，处理方法见下面的分析

4.3 情况3：一段时间内没有任何获胜者，处理方法见下面的分析

![](https://ask.qcloudimg.com/http-save/yehe-9955853/1d7c3999c4e038c27845f4507f8f6e95.gif)

对于上面的情况4.1，对于一个candidate节点获得集群中过半的节点对自己的投票，它就赢得了选举并成为leader。例如对于有5个节点的集群，至少收到3个节点的投票才会成为leader. 在投票的时候有一些约束条件：

1.  同一任期同一个节点只会投票给一个candidate，如果收到有多个相同任期的candidate需要投票，只会投票给最早的那个，遵循先来先服务(first-come-first-served)的原则。
2.  成为leader需要获得过半节点的投票，确保了最多只有一个candidate能够赢得选举。
3.  当candidate赢得选举，成为leader之后，会向其他节点发送心跳信息来捍卫自己的地位并阻止新的选举。

对于上面的情况4.2，在等待投票期间，candidate可能会收到另一个声称自己是leader节点发送来的AppendEntries RPC消息，如果这个leader的任期号（在RPC消息中）不小于candidate当前的任期号，该candidate节点会承认leader的合法性自己退回到follower状态。如果RPC中的任期号比自己的小，该candidate会拒绝响应并继续保持candidate状态。

对于上面的情况4.3，即candidate没有赢得选举也没有输，这种情况发生在有多个follower节点同时成为candidate节点，导致选票被瓜分，以至于没有一个candidate赢得过半的投票。最后的结果是每个candidate都会超时，然后通过增加当前任期号来开始新一轮的选举。当然这种情况可能一直重复，导致无法选举出leader，所以raft采用随机选举超时时间的方法确保这种情况很少出现。就算发生也能够很快解决。为了阻止选票一开始被瓜分，选举超时时间从一个固定的范围【例如150-300毫秒】随机选择。这样可以把节点分散开以至于在大多数情况下只有一个节点会选举超时，然后该节点赢得选举并在其他节点之前发送心跳。这种机制也被用来解决选票被瓜分的情况，每个candidate在开始一次选举的时候会重置一个随机的选举超时时间，然后一直等到选举超时，这样减少了新的选举中再发送选票被瓜分的情况。

上面的内容可以结合`https://raft.github.io/raftscope/index.html`动画来学习，该动画模拟了5个节点的raft集群交互，可以对节点设置超时(timte out)、宕机(stop)等场景，然后观察对集群产生的效果。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/6e6b740311544c9d15eddf98c18dec1b.jpeg?imageView2/2/w/2560/h/7000)

In Search of an Understandable Consensus Algorithm[1]raft动画[2]


# raft论文学习-log replication

发布于2022-08-15 15:09:37阅读 1670

## 日志

当leader被选举出来之后，就可以为客户端提供写入和读取服务了。客户端的每个请求都包含一条指令，该指令将会被状态机执行。leader收到客户端发来的指令之后，会做下面几个动作：

1.  将指令作为一个新的条目追加到日志里面
2.  并发的发出AppendEntries RPC消息给其他的节点
3.  当2中的条目被安全的之后，leader会应用该条目到自己的状态机中
4.  返回给客户端执行的结果

日志结构如下，包含状态指令、任期号和索引三部分。状态指令为业务层的增删改操作,例如 x <- 3. 任期号为leader收到该指令时的任期值，索引用于指明每个日志条目在日志中的位置。

下图中集群包含5个节点，每个小方框中的x<-3，y<-3是状态指令，方框中的1、2和3表示任期，最上面的1到8位日志索引。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/a84ff83fec3574929e8941a2337cf47f.jpeg?imageView2/2/w/2560/h/7000)

下面为日志Entry的数据结构定义(来自 etcd/raft/raftpb/raft.pb.go)，除了包含上面介绍的日志结构包含的三个部分，还有一个Type字段表示日志的类型，该字段用于区分该条Entry是普通数据操作还是集群变更操作。

```go
type Entry struct {
 // 任期
 Term  uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`
 // 日志索引
 Index uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
 // 有3中类型，分别表示普通数据操作，集群变更操作，集群变更操作V2
 Type  EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
 // 日志数据
 Data  []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
}
```

前面说了每条日志都包含任期和日志索引信息，那它们的值在哪里设置的呢？客户端发出的状态指令只是业务数据，里面是不包含任期和索引信息的。客户端发出的状态指令可能被follower节点或leader节点接收，如果是follower节点接收，它会转发给leader节点，无论如何状态指令都会到达leader节点。任期和日志索引信息值添加是在leader节点中的appendEntry方法中添加的，见下面的代码，可以看到Index值是连续增加的。

```go
// raft.go
func (r *raft) appendEntry(es ...pb.Entry) (accepted bool) {
 li := r.raftLog.lastIndex()
 for i := range es {
  // 设置日志条目的任期值和索引值，即向es中添加Term和Index
  // 日志条目从客户端发过来的时候，只有es.Data和es.Type有填充
  // 内容，经过这里的处理之后，es中所有的字段值都有了
  es[i].Term = r.Term
  es[i].Index = li + 1 + uint64(i)
 }
 ...
 return true
}
```
如果leader成功将日志到超过一半的节点上，这个日志为已提交日志(committed entries)，raft算法保证所有已提交的日志条目都是持久化的并且最终会被所有可用的状态机执行，结合下面的图示加以理解。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/65a4f44b7720c982aad2639a784695af.jpeg?imageView2/2/w/2560/h/7000)

为了维持不同节点之间日志的一致性，raft算法维护着下面两个特性：

-   如果不同日志中的两个条目拥有相同的索引和任期号，那么它们存储了相同的指令
-   如果不同日志中的两个条目拥有相同的索引和任期号，那么它们之前的所有日志条目也都相同

leader在特定的任期号内的一个日志索引位置最多创建一个日志条目，该特性保证了每条日志的{term,index}是唯一的。如果两个不同的日志条目它们的{term,index}是相同的，那么它们存储的相同的状态指令。对应到下面的entry1和entry2，它们的Term和Index值是相同的，那么它们的Data值也是相同的。这个点保证了上面的特性1，不会存在一个Term+Index对应多个[日志数据](https://cloud.tencent.com/solution/cloudlog?from=20065&from_column=20065)的情况。

```go
entry1:=Entry{Term:1,Index:3,Data:[]byte{...}}
entry2:=Entry{Term:1,Index:3,Data:[]byte{...}}

```



follower节点会对收到的AppendEntries RPC做一个一致性检查来保证上面的特性2，具体来说就是，在leader发送AppendEntries RPC时会将前一个日志条目中的索引位置和任期号包含在里面。如果follower在它的日志中找不到包含相同索引位置和任期号的条目，它会拒绝此新的日志条目。这个处理其实是数学中的归纳法：一开始空的日志状态肯定是满足特性2的，随后每增加一个日志条目时，都要求上一个日志条目信息与leader一致，那么最终整个日志集肯定是一致的。

节点之间消息传递是放在下面的Message对象中的，像上面说的AppendEntries RPC消息。前一个日志条目中的索引位置和任期号就是Message结构体中的LogTerm和Index字段，Message中的Entries是存放日志条目的。

```go
type Message struct {
...
 // 下面Index索引值对应的日志的所在的任期
 LogTerm    uint64   `protobuf:"varint,5,opt,name=logTerm" json:"logTerm"`
 // follower节点之前已收到的日志条目的索引的最大值
 Index      uint64   `protobuf:"varint,6,opt,name=index" json:"index"`
 // 日志条目集合
 Entries    []Entry  `protobuf:"bytes,7,rep,name=entries" json:"entries"`
...
}
```

在正常情况下，follower节点和leader节点的日志保持一致，所以follower节点对AppendEntries RPC一致性检查不会失败。然而，leader崩溃的情况会使日志处于不一致的状态。比如旧的leader可能还没有完全完它里面的所有日志条目，甚至在一系列leader和follower节点崩溃的情况更槽糕，下图展示了这些情况。
![[Pasted image 20230414123342.png]]
上图中最上面的一行是leader成功当选时候的日志，图中格子中的数字表示任期号，follower节点可能是a-f中的任意一种情况。a和b是缺少日志条目的情况，a缺少Index为10的日志，b缺少Index在[5,10]范围内的日志。c和d是存在未提交日志条目的情况，c存在未提交的Index为11的日志，d存在未提交的Index为11和12的日志。e和f是缺少日志和存在未提交的日志都有的情况，e缺少Index在[6,10]范围内的日志，多了Index为6和7的任期值为4的日志。f的情况可能是这样产生的，f节点在任期2的时候是leader，追加了一些日志条目到自己的日志中，一条都还没提交(commit)就崩溃了，该节点很快重启，在任期3又被重新当选为leader，又追加了一些日志条目到自己的日志中。在任期2和任期3中的日志都还没提交之前，该节点又宕机了，并且在接下来的任期里面一直处于宕机状态。

raft算法通过强制follower节点leader节点的日志来解决不一致问题。那现在就有一个问题，leader节点怎么知道从哪个日志索引位置发送日志条目给follower，以及follower已经的日志最大索引值是多少？

leader对每个follower节点都维护了一个nextIndex和一个matchIndex字段，nextIndex表示leader要发送给follower的下一个日志条目的索引, matchIndex表示follower节点已的最大日志条目的索引。要使得follower的日志跟leader的一致，leader必须找到两者达成一致的最大的日志条目，删除follower日志中从最大日志条目所在索引之后的所有日志条目，并将自己从最大日志条目索引之后的日志发送给follower.

当选出一个新的leader时，该leader将所有nextIndex的值都初始化为自己最后一个日志条目的index+1. 如果follower的日志和leader的不一致，在下一次AppendEntries RPC中的一致性检查就会失败，在follower拒绝之后，leader会减小nextIndex值并重试AppendEntries RPC. 最终nextIndex的值会在某个位置，leader和follower在此处达成一致，此时AppendEntries RPC成功，follower会将跟leader冲突的日志条目全部删除然后追加leader中的日志条目(如果需要追加的话)。追加成功后，follower的日志和leader就一致了，并且在该任期接下来的时间里保持一致。

通过上述机制，leader在当选之后不需要进行任何特殊的操作便可以将日志恢复到一致状态。leader只需要进行正常的appendEntries RPC，然后日志就能在回复AppendEntries一致性检查失败的时候自动趋于一致，leader从来不会覆盖或者删除自己的日志条目。


# raft论文学习-safety

发布于2022-08-15 15:10:02阅读 1220

## 安全性

在[raft论文学习-raft basics & leader election](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488754&idx=1&sn=bad27fa716efba49f298c566bced7155&chksm=873a6cb3b04de5a586b5a742a149df9186feb863efd20f5f0402b4072c994b39f062170649e9&scene=21#wechat_redirect)和[raft论文学习-log replication](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488769&idx=1&sn=7e3f5b030972a6f511a195de4313dfca&chksm=873a6cc0b04de5d6c6200055e41bea7a13aa954bb32236f059ec9789473e5588e456d9ea04d3&scene=21#wechat_redirect)文章中已经介绍了raft算法的领导人选举和日志，然而它们并不能充分的保证每个节点会按照相同的顺序执行相同的指令，所以需要一些约束条件来保证节点执行顺序的安全性。例如，当一个follower节点挂掉后，leader节点可能提交了很多条的日志条目，挂掉的follower节点很快重启后可能被选举为新的leader节点，新的leader节点接收日志条目后会给其他follower节点，会导致follower中的日志条目被覆盖，这会导致不同的节点执行的不同的指令序列。对于上述情况，raft算法通过增加约束限制来保证对给定的任意任期号，leader都包含了之前各个任期所有被提交的日志条目。

#### 选举约束

任何基于leader的一致性算法，leader最终都必须存储所有已经提交的日志条目。在viewstamped replication一致性算法中，一开始时并没有包含所有已经提交的日志条目的节点也可能被选为leader,这种算法会用一些额外的机制来识别丢失的日志条目并将它们传送给新的leader,不过这种方法会导致比较大的复杂性。raft使用了一种简单的方法，可以保证新的leader在当选时就包含了之前所有任期号中已经提交的日志条目，所以不需要再传送日志条目给新leader.这样保证了日志条目传送的单向性，只能从leader传给follower，并且leader从不会覆盖本地日志中已经存在的条目。

那raft是如何保证新的leader在当选时就包含了之前所有任期号中的已经提交的日志呢？raft做法是新的leader选出有约束限制，一个candidate并不是获得大多数节点的投票就能当选。除非该candidate节点包含了所有已经提交的日志条目。candidate节点为了赢得选举必须与集群中的过半的节点通信，而已提交的日志条目肯定存储到了过半的节点上，那么与candidate进行通信的节点中至少有一个节点包含有所有已经提交的日志。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/23433cbaf5fe48c90a32f849cc3e3166.jpeg?imageView2/2/w/2560/h/7000)

在requestVote RPC中raft限制了如果投票者自己的日志比candidate的还新，它会拒绝掉该投票请求。这里比较日志哪个更新的定义是：

-   如果两个日志的任期号不同，任期号大的日志更新
-   如果两个日志的任期号相同，日志较长的日志更新

有了这里的约束，一个candidate要想成为leader，必须赢得过半节点的投票，而这过半节点中至少有一个拥有所有已经提交的日志，所以这个candidate也拥有所有已经提交的日志，当它成为leader时已经有所有已经提交的日志了。

#### 提交之前任期内的日志条目

对应「当前任期内」的日志条目如果已经存储到过半的节点上，leader就知道该日志条目已经被提交了。然而，如果是之前任期内的日志条目已经存储到过半的节点上，leader也无法立即断定该日志条目已经被提交了。下图（来自raft论文）展示了这种情况，一个已经被存储到过半节点上的旧日志条目，任然有可能被将来的leader覆盖掉。
![[Pasted image 20230414135306.png]]
上图中的S1到S5是集群中的5个节点，方框表示日志条目，方框中的数字表示任期term, 最上面一行的数字表示日志索引index.

1.  在(a)中，S1是leader节点，部分地了index为2的日志条目到S2节点中。
2.  在(b)中，S1崩溃了，然后S5在任期3通过节点S3、S4和自己赢得选举，然后从客户端接收了一条新的日志条目放在了索引2处。这里在分析下为什么S5可以当选为leader节点，根据前一小节的约束，S5的term为3，S3和S4的term为1，所以当S3和S4收到S5的 requestVote RPC时，是满足S5的日志比S3和S4新的，因为term较大，会投票给S5.
3.  在(c)中，S5崩溃了，S1重新启动，选举成功，继续日志，来自任期2的日志已经被到了集群中的大多数节点上，但是还没有提交。这里继续分析下为啥S1又又可以成为leader节点，因为它的term为4，比S2、S3、S4和S5的term都大，所以是满足前一小节约束条件的，可以成为leader节点。
4.  在(d)中，S1又崩溃了，S5通过来自S2、S3和S4的投票重新成为leader节点，然后覆盖了S2和S3中index为2处的日志。

现在我们来思考场景(d)是正确的还是错误的？答案是正确的，是不是有点出乎意料。因为从客户端的角度看，在场景(b)的时候，S1作为leader崩溃，term为2的日志返回给客户端肯定是超时或者出错的。也就是说对于term为2的日志，在(d)中被覆盖(丢弃)，还是节点存储下来，都是正确的。

但是从raft规则的角度来看，「这不满足raft规则：一条日志一旦被到了多数节点上，就认为这条日志是commit状态，这条日志是不能被覆盖的」，但是现在却被覆盖了!!! 那怎么办呢？修改commit的规则定义。新的规则为：新的leder被选举出来后，对之前任期内的日志，即使已经被到了多数节点，任然不认为是commit的。只有等到新的leader在自己的任期内commit了日志，之前任期内的日志才算commit了。

也就是说raft不会通过计算副本数目的方式来提交「之前任期内」的日志条目。只有leader「当前任期内」的日志条目才通过计算副本的方式来提交。对应到上图中的(e),在崩溃之前，如果S1在自己的任期(term=4)里了日志条目（term=4,index=3)到大多数节点上，然后这个日志条目就会被提交（S5不可能选举成功，因为它的日志没有S1、S2和S3新，这三个节点拒绝为S5投票，S5不可能获得大多数选票)，在这种情况下,之前的(term=2,index=2)的日志也被提交了。不会出现term为2被覆盖的情况，S5老老实实成为follower节点，让term为2的日志覆盖它的term为3的日志。

场景(d)和(e)从客户端角度来看，都是正确的。场景(d)之所以出现日志被覆盖的情况，是因为term为2的日志不是一次性被到多数节点，而是跨越了多个任期，断续地被到多数节点。新选举出来的leader不能认为这种日志是commit的，需要延迟等到新的任期里面有日志被commit后，之前任期内遗留的日志被顺便变为commit状态。

raft修改commit规则增加额外复杂性是因为新选举出来的leader在之前任期内的日志条目时，这些日志条目都保留的是原来的任期号。在其他的一致性算法中，如果一个新的leader要重新之前任期里的日志时，必须使用当前新的任期号。raft算法保持原来的term有两个好处：一是更容易追溯日志条目，因为term是不变的嘛，二是新leader只需要发送更少的日志条目，其他算法必须在它们被提交之前发送更多的冗余日志条目对日志重新编号。

#### 安全性证明

leader的完整性证明采用了反证法，即假设leader的完整性是不满足的，然后推出矛盾。假设任期T的leader在任期内提交了一个日志条目，但是该日志条目没有被存储到将来的某些任期的leader中，并假定U是大于T的没有存储该日志条目的最小任期号。下图来自raft论文。
![[Pasted image 20230414140104.png]]
图中的集群有5个节点，节点S1在任期T是leader,提交了一个新的日志条目。然后S5在之后的任期U(T<U)里被选举为leader.那么肯定至少会有一个节点，比如S3，既接收了来自S1的日志条目，又给S5投票了。

1.  任期为U的leader节点一定在刚成为leader的时候就没有那条被提交的日志条目了，因为leader从不会删除或者覆盖任何日志条目
2.  任期为T的leader会日志条目给集群中过半的节点，同时任期为U的leader从集群中的过半节点赢得了选票。因此，至少有一个节点同时接受了来自leader T的日志条目并给leader U投票了
3.  投票的节点在给leader U投票之前先接受了从leader T发来的已经被提交的日志条目，否则它会拒绝来自leader T的 appendEntries请求，因为它的任期号比T大
4.  投票的节点在给leader U投票时依然保有这条日志条目，因为任何U、T之间的leader都包含该日志条目，leader从来不会删除日志，并且follower节点只在跟leader冲突的时候才会删除条目
5.  投票的节点在把自己投票给leader U节点时，leader U的日志必须至少和投票者的一样新，这就导致了下面的两个矛盾
6.  如果投票者和leader U的最后一个日志条目的任期号相同，那么leader U的日志至少和该投票者的一样长，所以leader U的日志一定包含该投票者日志中的所有日志条目。这与假设leader U不包含投票者日志是矛盾的
7.  如果6不成立，那leader U的最后一个日志条目的任期号就必须比该投票者的大，此外，该任期号也比T大，因为该投票者的最后一个日志条目的任期号至少和T一样大。创建leader U最后一个日志条目的之前的leader一定已经包含了该已被提交的日志条目。根据日志匹配特性，leader U一定也包含了该已提交的日志条目，与假设矛盾
8.  所以所有比T大的任期的leader一定都包含了任期T中提交的所有日志条目
9.  日志匹配特性保证了将来的leader也会包含间接提交的日志条目

#### follower和candidate节点崩溃情况

前面分析了leader崩溃的情况，follower和candidate节点崩溃的情况要比leader简单不少，并且它们两者的处理方式是相同的。

-   如果follower或者candidate崩溃了，那发给它们的requestVote和appendEntries RPC都会失败，raft通过「无限的重试」来处理这种失败，当崩溃的机器重启后，这些RPC就会成功地完成。
-   如果一个节点在完成了一个RPC但是还没有响应的时候崩溃了，当它重启之后会再次收到相同的请求，raft的RPC是幂等的，所以重试不会有副作用。

#### 定时和可用性

raft算法的其中一个要求是安全性不能依赖定时：整个系统不能因为某些事情运行得比预期快一点或者慢一点就产生错误的结果。但是可用性不可避免的要依赖于定时，例如当有节点崩溃的时候，消息交换的时间会比正常情况下长，没有一个稳定的leader，raft是无法工作的，所以candidate不能等待太长的时间来赢得选举。

leader选举是raft中定时最关键的要素，只要整个系统满足下面的关系，raft就可以选举并维持一个稳定的leader.

广播时间(broadcast time) << 选举超时时间(election timeout) << 平均故障间隔时间(MTBF)

广播时间：节点并行地发送RPC给集群中所有其他的节点并接收到响应的平均时间

选举超时时间：这个一般是用户设定的

平均故障间隔时间：对于一个节点来说，两次故障间隔时间的平均值

广播时间必须比选举超时时间小一个数量级，这样leader才能够可靠地发送心跳来阻止follower开始进入选举状态，再加上随机化选举超时时间的方法，使得瓜分选票的情况几乎不可能。选举超时时间需要比平均故障时间间隔小几个数量级，这样整个系统才能稳定的运行。当leader崩溃后，整个系统会有大约选举超时的时间不可用，希望这个时间只是占整个时间的很小一部分。

上面的三个时间值如何设定呢？广播时间和平均故障间隔时间是由系统决定的，我们自己设置的是选举超时时间。raft的RPC需要接收方将信息持久化的保存到稳定存储中，所以广播时间大约在[0.5,20]毫秒之间，选举超时时间在[10,500]毫秒之间，大多数的节点平均无故障间隔时间都在几个月甚至更长，很容易满足时间的要求。


# raft论文学习-cluster membership changes

发布于2022-08-15 15:11:03阅读 1240

## 集群成员变更

在实际工程环境中，会存在偶尔改变集群配置的情况。例如替换掉宕机的机器。虽然可以通过将集群所有机器下线，更新所有配置，然后重启整个集群的方式来实现。但是这会存在以下问题：

1.  集群中的机器会先停机然后重启，在这个过程中，集群不能对外提供服务，在生产环境中是不能接受的
2.  通过手工操作，存在操作失误的风险

为了使配置变更机制能够安全，在转换的过程中不能够在任何时间点使得同一个任期里可能选出两个leader，这不满足raft的规则。槽糕的是，任何机器直接从旧配置转换到新的配置的方案都是不安全的。一次性自动地转换所有机器是不可能的，在转换期间整个集群可能被划分成两个独立的大多数。

下面通过例子来说明可能会同时出现两个leader的情况。在成员变更前是3节点的集群，集群中的节点为A、B和C，其中节点A为leader节点. 现在向集群中增加节点D和E,加入成功后变成5节点集群。集群旧配置用Cold表示，则Cold={A,B,C}。新配置用Cnew表示，则Cnew={A,B,C,D,E}.

![](https://ask.qcloudimg.com/http-save/yehe-9955853/4c3831009fd4f6f19e49ed7c187bf3fa.jpeg?imageView2/2/w/2560/h/7000)

在集群变更前，A为leader节点，节点C和B已同步完节点A的最新数据。此时，站在节点A/B/C任意一个节点的角度来说，都是知道有另外两个节点存在的。例如对于C节点来说，它知道除了自己集群中还有节点A和节点B.

![](https://ask.qcloudimg.com/http-save/yehe-9955853/cec5fbd87c5d8c72e3cdeadf6f48248f.jpeg?imageView2/2/w/2560/h/7000)

现在向集群中添加节点D和E.对于节点D和E来说，在添加时它们是知道集群已存在节点A、节点B和节点C的。即在节点D和E眼中，集群的配置是有A/B/C/D/E五个节点的。新增的节点信息会同步给其他节点，同步的过程是需要一点时间的，而不是在一个时刻节点A、节点B和节点C都知道了有新的节点D和E加入。这里假设节点A先知道了D和E加入，在C和B还未知道D和E的情况下，出现了网络分区。C/B是一个分区，A/D/E是一个分区。C/B分区由于接收不到节点A的心跳，会出现竞选投票，假设节点C先出现竞选，它会收到节点B给他的投票，因为节点B的数据不会比节点C新。此时，节点C可以成功当选leader。因为它没有感知到节点D和E，所以在它眼中集群还是3个节点，已收到2个投票（节点B+自己），可以当选。A/D/E分区中节点A是可以继续作为leader节点的，虽然在A/D/E眼中，集群是有五个节点构成的，但是他们有3个节点，也构成一个5节点中的大多数。这就出现了集群中存在C和A两个leader的情况。
![[Pasted image 20230414141139.png]]
为了保证安全性，配置变更必须采用一种两阶段方法。在raft中，集群先切换到一个过渡的配置，称之为联合一致(join consensus)，一旦联合一致已经被提交后，系统可以切换到新的配置上。联合一致结合了老配置和新配置：

-   日志条目被给集群中新、老配置的所有[服务器](https://cloud.tencent.com/product/cvm?from=20065&from_column=20065)
-   新旧配置的服务器都可以成为leader
-   对于选举和提交的达成一致需要分别在两种配置上获得过半的支持

联合一致(join consensus)允许服务器在保持安全的前提下，在不同的时刻进行配置转换，并且允许集群在配置变更期间依然响应客户端请求。
![[Pasted image 20230414141152.png]]
上图来自raft论文，描述了联合一致过程中配置变更流程。图中虚线代表已创建但是未提交的配置，实线代表最新的已提交的配置。leader会首先创建Cold-Cnew日志项，并到新旧配置节点中的大多数。然后创建Cnew日志项，并到Cnew新配置的大多数。

这里说下上面配置的含义，上面配置就是集群中有哪些节点组成。通过例子加以说明。假设一个集群之前有A/B/C三个节点，现在发生配置变更，需要添加D和E两个节点。Cold,Cold-new,Cnew配置为：

```go
Cold={A,B,C}
Cold-Cnew={A,B,C},{A,B,C,D,E}
Cnew={A,B,C,D,E}
```

注意，Cold-Cnew有些文章写成Cold∪Cnew是有误导性的，我在看的时候有疑惑，如果理解成∪，那上面的Cold-Cnew进行并集处理后不是变为了{A,B,C,D,E}，这不成了Cnew. 为了理清疑惑，查看了logcabin实现，logcabin(`https://github.com/logcabin/logcabin`)是基于raft构建的[分布式存储](https://cloud.tencent.com/product/cos?from=20065&from_column=20065)系统，可提供少量高度的一致存储, 它的集群变更采用这里的联合一致性方法，Cold-Cnew配置代码如下，它们是两个独立的集合。

```go
if (newDescription.next_configuration().servers().size() == 0)
        state = State::STABLE;
    else
        state = State::TRANSITIONAL;
    id = newId;
    description = newDescription;
    oldServers.servers.clear();
    newServers.servers.clear();

    // Build up the list of old servers
    for (auto confIt = description.prev_configuration().servers().begin();
         confIt != description.prev_configuration().servers().end();
         ++confIt) {
        std::shared_ptr<Server> server = getServer(confIt->server_id());
        server->addresses = confIt->addresses();
        oldServers.servers.push_back(server);
    }

    // Build up the list of new servers
    for (auto confIt = description.next_configuration().servers().begin();
         confIt != description.next_configuration().servers().end();
         ++confIt) {
        std::shared_ptr<Server> server = getServer(confIt->server_id());
        server->addresses = confIt->addresses();
        newServers.servers.push_back(server);
    }
```
![[Pasted image 20230414141335.png]]
下面结合上述变更图，从时间角度进行一个分析。整个变更过程划分为如下的区间。

1.  当时间 t<Time1时，此时集群处于配置变更前的状态，全是Cold节点
2.  当时间 Time1<t<Time2时，集群开始进行配置变更，leader开始把Cold-Cnew节点给其他节点。这个时间段里面，Cold-Cnew和Cold节点共存，选举出来的leader可能是拥有Cold配置的节点，也有可能是拥有Cold-Cnew配置的节点。拥有Cold-Cnew配置节点选举成功的条件是：Cold和Cnew中的大多数节点都同意。如果是Cold-Cnew配置节点选举成功，根据选举成功的条件，它一定是获得了Cold中大多数节点的同意，所以单纯的Cold配置节点不能选举成功。同理，如果Cold配置中的节点选举成功，那么Cold-Cnew配置中的节点是不能成功选举的，因为不满足条件中的Cold的大多数节点同意。
3.  当时间 Time2<t<Time3时，leader已成功将Cold-Cnew配置到大多数节点，此时集群中可能有Cold配置节点。但来自Cold配置节点不能选出成为leader了。这个时间段选出的leader必然是拥有Cold-Cnew配置的节点。因为Cold-Cnew日志节点占了大多数，即使在Cold集群中，如果选举leader成功，必须有一个拥有Cold-Cnew日志的follower,但是Cold-Cnew中的follower更新，它会拒绝来自Cold中节点的选举请求。
4.  当时间 Time3<t<Time4时，选举出来的leader可能是拥有Cold-Cnew配置的节点，也有可能是拥有Cnew配置的节点。但是只能是其中一个，假设Cold-Cnew中的节点当选成功，肯定是满足了条件有大多数Cnew节点同意，所以Cnew配置中的节点不可能当选为leader。同理，来自Cnew配置中的节点当选为leader,Cold-Cnew就不能当选为leader了。
5.  当时间 t>Time4时，选举出来的leader一定是拥有Cnew配置的节点，因为如果存在一个拥有Cold-Cnew的节点请求当选leader，则需要Cnew中的大多数节点同意，但是Cnew中的大多数已经写入了Cnew日志，所以不会响应Cold-Cnew节点。

论文中还分析了配置变更其他需要解决的三个问题：

-   问题1：新加进来的节点开始时没有存储任何日志条目，当它们加入到集群中，需要一段时间来更新日志才能赶上其他节点，这段时间内它们无法提交新的日志条目。为了避免上述问题而造成的系统短时间的不可用，raft在配置变更前引入了一个额外的阶段，在该阶段，新的节点以没有投票权身份加入到集群中，leader也日志给它们，但考虑过半的时候不用考虑它们，一旦新节点追赶上了集群中的其他机器，配置变更可以按照上面描述的方式进行。
-   问题2：集群的leader可能不在新配置Cnew中，这种情况，leader一旦提交了Cnew日志条目就会退位到followr状态。这意味着有一个时间段（leader提交Cnew期间），leader管理的是一个不包括字节的集群，它日志但不把自己算在过半里面。leader转换发生在Cnew被提交的时候，这个时候是新配置可以独立做决定的最早时刻，在此之前只能从Cold中选出leader.
-   问题3：被移除的节点即不在Cnew配置中的节点可能会扰乱集群。这些节点将不会再接收到心跳，当选举超时时候，它们会进行新的选举过程。它们会发送带有新任期号的requestVote RPC,这会导致当前的leader回到follower状态。新的leader最终会被选出来，但被移除的节点会再次超时，导致系统可用性很差。处理方法是，当服务器认为当前leader存在时，会忽略requestVote PRC。特别当服务器在最小选举超时时间内收到一个requestVote RPC，它不会更新任期号或投票。这不会影响正常的选举，因为每个节点在开始一次选举之前，至少等待最小选举超时时间。


# 一致性协议之 ZAB

什么是 ZAB 协议？ZAB 协议介绍

ZAB 协议全称：Zookeeper Atomic Broadcast（Zookeeper 原子广播协议）。

ZAB 协议是为分布式协调服务 Zookeeper 专门设计的一种支持 崩溃恢复 和 原子广播 协议。

整个 Zookeeper 就是在这两个模式之间切换。简而言之，当 Leader 服务可以正常使用，就进入消息广播模式，当 Leader 不可用时，则进入崩溃恢复模式。

基于该协议，Zookeeper 实现了一种 主备模式 的系统架构来保持集群中各个副本之间数据一致性。其中所有客户端写入数据都是写入到 主进程（称为 Leader）中，然后，由 Leader 复制到备份进程（称为 Follower）中。【涉及到2PC单点问题的解决，崩溃恢复】

选择机制中的概念

**1、Serverid：服务器ID**

比如有三台服务器，编号分别是1,2,3。

编号越大在选择算法中的权重越大。

**2、Zxid：数据ID**

服务器中存放的最大数据ID。【zxid实际上是一个64位的数字，高32位是epoch（时期; 纪元; 世; 新时代）用来标识leader是否发生改变，如果有新的leader产生出来，epoch会自增，低32位用来递增计数。】

值越大说明数据越新，在选举算法中数据越新权重越大。

**3、Epoch：逻辑时钟**

或者叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的。每投完一次票这个数据就会增加，然后与接收到的其它服务器返回的投票信息中的数值相比，根据不同的值做出不同的判断。

4、Server状态：选举状态

LOOKING，竞选状态。

FOLLOWING，随从状态，同步leader状态，参与投票。

OBSERVING，观察状态,同步leader状态，不参与投票。

LEADING，领导者状态。

## zab常见的误区：

-   写入节点后的数据，立马就能被读到，这是错误的。** zk 写入是必须通过 leader 串行的写入，而且只要一半以上的节点写入成功即可。而任何节点都可提供读取服务**。例如：zk，有 1~5 个节点，写入了一个最新的数据，最新数据写入到节点 1~3，会返回成功。然后读取请求过来要读取最新的节点数据，请求可能被分配到节点 4~5 。而此时最新数据还没有同步到节点4~5。会读取不到最近的数据。**如果想要读取到最新的数据，可以在读取前使用 sync 命令**。
-   zk启动节点不能偶数台，这也是错误的。zk 是需要一半以上节点才能正常工作的。例如创建 4 个节点，半数以上正常节点数是 3。也就是最多只允许一台机器 down 掉。而 3 台节点，半数以上正常节点数是 2，也是最多允许一台机器 down 掉。4 个节点，多了一台机器的成本，但是健壮性和 3 个节点的集群一样。基于成本的考虑是不推荐的

## 选举消息内容

在投票完成后，需要将投票信息发送给集群中的所有服务器，它包含如下内容：服务器ID、数据ID、逻辑时钟、选举状态。

zookeeper是如何保证事务的顺序一致性的（保证消息有序） 在整个消息广播中，Leader会将每一个事务请求转换成对应的 proposal 来进行广播，并且在广播 事务Proposal 之前，Leader服务器会首先为这个事务Proposal分配一个全局单递增的唯一ID，称之为事务ID（即zxid），**由于Zab协议需要保证每一个消息的严格的顺序关系，因此必须将每一个proposal按照其zxid的先后顺序进行排序和处理。**

## 消息广播

1）在zookeeper集群中，数据副本的传递策略就是采用消息广播模式。zookeeper中农数据副本的同步方式与二段提交相似，但是却又不同。二段提交要求协调者必须等到所有的参与者全部反馈ACK确认消息后，再发送commit消息。要求所有的参与者要么全部成功，要么全部失败。二段提交会产生严重的阻塞问题。

2）Zab协议中 Leader 等待 Follower 的ACK反馈消息是指“只要半数以上的Follower成功反馈即可，不需要收到全部Follower反馈”。

**消息广播具体步骤**

**1）客户端发起一个写操作请求。**

**2）Leader 服务器将客户端的请求转化为事务 Proposal 提案，同时为每个 Proposal 分配一个全局的ID，即zxid。**

**3）Leader 服务器为每个 Follower 服务器分配一个单独的队列，然后将需要广播的 Proposal 依次放到队列中取，并且根据 FIFO 策略进行消息发送。**

**4）Follower 接收到 Proposal 后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后向 Leader 反馈一个 Ack 响应消息。**

**5）Leader 接收到超过半数以上 Follower 的 Ack 响应消息后，即认为消息发送成功，可以发送 commit 消息。**

**6）Leader 向所有 Follower 广播 commit 消息，同时自身也会完成事务提交。Follower 接收到 commit 消息后，会将上一条事务提交。**

zookeeper 采用 Zab 协议的核心，就是只要有一台服务器提交了 Proposal，就要确保所有的服务器最终都能正确提交 Proposal。这也是 CAP/BASE 实现最终一致性的一个体现。

**Leader 服务器与每一个 Follower 服务器之间都维护了一个单独的 FIFO 消息队列进行收发消息，使用队列消息可以做到异步解耦。Leader 和 Follower 之间只需要往队列中发消息即可。如果使用同步的方式会引起阻塞，性能要下降很多。**

## 崩溃恢复

崩溃恢复主要包括两部分：**Leader选举 和 数据恢复**

zookeeper是如何选取主leader的？

当leader崩溃或者leader失去大多数的follower，这时zk进入恢复模式，恢复模式需要重新选举出一个新的leader，让所有的Server都恢复到一个正确的状态。

**Zookeeper选主流程 选举流程详述**

**一、首先开始选举阶段，每个Server读取自身的zxid。**

**二、发送投票信息**

a、首先，每个Server第一轮都会投票给自己。

b、投票信息包含 ：所选举leader的Serverid，Zxid，Epoch。Epoch会随着选举轮数的增加而递增。

**三、接收投票信息**

**1、如果服务器B接收到服务器A的数据（服务器A处于选举状态(LOOKING 状态)**

**1）首先，判断逻辑时钟值：**

**a）如果发送过来的逻辑时钟Epoch大于目前的逻辑时钟。首先，更新本逻辑时钟Epoch，同时清空本轮逻辑时钟收集到的来自其他server的选举数据。**然后，判断是否需要更新当前自己的选举leader Serverid。判断规则rules judging：保存的zxid最大值和leader Serverid来进行判断的。**先看数据zxid,数据zxid大者胜出;其次再判断leader Serverid,leader Serverid大者胜出；然后再将自身最新的选举结果(也就是上面提到的三种数据（leader Serverid，Zxid，Epoch）广播给其他server)**

b）如果发送过来的逻辑时钟Epoch小于目前的逻辑时钟。说明对方server在一个相对较早的Epoch中，这里只需要将本机的三种数据（leader Serverid，Zxid，Epoch）发送过去就行。

c）如果发送过来的逻辑时钟Epoch等于目前的逻辑时钟。再根据上述判断规则rules judging来选举leader ，然后再将自身最新的选举结果(也就是上面提到的三种数据（leader Serverid，Zxid，Epoch）广播给其他server)。

**2）其次，判断服务器是不是已经收集到了所有服务器的选举状态**：若是，根据选举结果设置自己的角色(FOLLOWING还是LEADER)，退出选举过程就是了。

最后，若没有收集到所有服务器的选举状态：也可以判断一下根据以上过程之后最新的选举leader是不是得到了超过半数以上服务器的支持,如果是,那么尝试在200ms内接收一下数据,如果没有新的数据到来,说明大家都已经默认了这个结果,同样也设置角色退出选举过程。

**2、 如果所接收服务器A处在其它状态（FOLLOWING或者LEADING）。**

a)逻辑时钟Epoch等于目前的逻辑时钟，将该数据保存到recvset。此时Server已经处于LEADING状态，说明此时这个server已经投票选出结果。若此时这个接收服务器宣称自己是leader, 那么将判断是不是有半数以上的服务器选举它，如果是则设置选举状态退出选举过程。

b) 否则这是一条与当前逻辑时钟不符合的消息，那么说明在另一个选举过程中已经有了选举结果，于是将该选举结果加入到outofelection集合中，再根据outofelection来判断是否可以结束选举,如果可以也是保存逻辑时钟，设置选举状态，退出选举过程。【recvset：用来记录选票信息，以方便后续统计;outofelection：用来记录选举逻辑之外的选票，例如当一个服务器加入zookeeper集群时，因为集群已经存在，不用重新选举，只需要在满足一定条件下加入集群即可。】  
  

## Zab 协议如何保证数据一致性

假设两种异常情况：

1、一个事务在 Leader 上提交了，并且过半的 Folower 都响应 Ack 了，但是 Leader 在 Commit 消息发出之前挂了。

2、假设一个事务在 Leader 提出之后，Leader 挂了。

要确保如果发生上述两种情况，数据还能保持一致性，那么 Zab 协议选举算法必须满足以下要求：

Zab 协议崩溃恢复要求满足以下两个要求：

**1）确保已经被 Leader 提交的 Proposal 必须最终被所有的 Follower 服务器提交。**

**2）确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal。**

根据上述要求 Zab协议需要保证选举出来的Leader需要满足以下条件：

**1）新选举出来的 Leader 不能包含未提交的 Proposal 。即新选举的 Leader 必须都是已经提交了 Proposal 的 Follower 服务器节点。**

**2）新选举的 Leader 节点中含有最大的 zxid 。这样做的好处是可以避免 Leader 服务器检查 Proposal 的提交和丢弃工作。**

## Zab 如何数据同步

1）完成 Leader 选举后（新的 Leader 具有最高的zxid），在正式开始工作之前（接收事务请求，然后提出新的 Proposal），Leader 服务器会首先确认事务日志中的所有的 Proposal 是否已经被集群中过半的服务器 Commit。

2）Leader 服务器需要确保所有的 Follower 服务器能够接收到每一条事务的 Proposal ，并且能将所有已经提交的事务 Proposal 应用到内存数据中。等到 Follower 将所有尚未同步的事务 Proposal 都从 Leader 服务器上同步过啦并且应用到内存数据中以后，Leader 才会把该 Follower 加入到真正可用的 Follower 列表中。

Zab 数据同步过程中，如何处理需要丢弃的 Proposal。

**在 Zab 的事务编号 zxid 设计中，zxid是一个64位的数字。**

**其中低32位可以看成一个简单的单增计数器，针对客户端每一个事务请求，Leader 在产生新的 Proposal 事务时，都会对该计数器加1。而高32位则代表了 Leader 周期的 epoch 编号。**

epoch 编号可以理解为当前集群所处的年代，或者周期。每次Leader变更之后都会在 epoch 的基础上加1，这样旧的 Leader 崩溃恢复之后，其他Follower 也不会听它的了，因为 Follower 只服从epoch最高的 Leader 命令。

每当选举产生一个新的 Leader ，就会从这个 Leader 服务器上取出本地事务日志充最大编号 Proposal 的 zxid，并从 zxid 中解析得到对应的 epoch 编号，然后再对其加1，之后该编号就作为新的 epoch 值，并将低32位数字归零，由0开始重新生成zxid。

Zab 协议通过 epoch 编号来区分 Leader 变化周期，能够有效避免不同的 Leader 错误的使用了相同的 zxid 编号提出了不一样的 Proposal 的异常情况。

基于以上策略:

当一个包含了上一个 Leader 周期中尚未提交过的事务 Proposal 的服务器启动时，当这台机器加入集群中，以 Follower 角色连上 Leader 服务器后，Leader 服务器会根据自己服务器上最后提交的 Proposal 来和 Follower 服务器的 Proposal 进行比对，比对的结果肯定是 Leader 要求 Follower 进行一个回退操作，回退到一个确实已经被集群中过半机器 Commit 的最新 Proposal。

小结

ZAB 协议和我们之前看的 Raft 协议实际上是有相似之处的，比如都有一个 Leader，用来保证一致性（Paxos 并没有使用 Leader 机制保证一致性）。再有采取过半即成功的机制保证服务可用（实际上 Paxos 和 Raft 都是这么做的）。

**ZAB 让整个 Zookeeper 集群在两个模式之间转换，消息广播和崩溃恢复，消息广播可以说是一个简化版本的 2PC，通过崩溃恢复解决了 2PC 的单点问题，通过队列解决了 2PC 的同步阻塞问题。**

而支持崩溃恢复后数据准确性的就是数据同步了，数据同步基于事务的 ZXID 的唯一性来保证。通过 + 1 操作可以辨别事务的先后顺序。

### ZAB协议 选举同步过程

#### 发起投票的契机

1.  节点启动
2.  节点运行期间无法与 Leader 保持连接，
3.  Leader 失去一半以上节点的连接

#### 如何保证事务

ZAB 协议类似于两阶段提交，客户端有一个写请求过来，例如设置 **/my/test** 值为 1，Leader 会生成对应的事务提议（proposal）（当前 zxid为 0x5000010 提议的 zxid 为Ox5000011），现将**set /my/test 1**（此处为伪代码）写入本地事务日志，然后**set /my/test 1**日志同步到所有的follower。follower收到事务 proposal ，将 proposal 写入到事务日志。如果收到半数以上 follower 的回应，那么广播发起 commit 请求。follower 收到 commit 请求后。会将文件中的 zxid ox5000011 应用到内存中。

上面说的是正常的情况。有两种情况。第一种 Leader 写入本地事务日志后，没有发送同步请求，就 down 了。即使选主之后又作为 follower 启动。此时这种还是会日志会丢掉（原因是选出的 leader 无此日志，无法进行同步）。第二种 Leader 发出同步请求，但是还没有 commit 就 down 了。此时这个日志不会丢掉，会同步提交到其他节点中。

#### 服务器启动过程中的投票过程

现在 5 台 zk 机器依次编号 1~5

1.  节点 1 启动，发出去的请求没有响应，此时是 Looking 的状态
2.  节点 2 启动，与节点 1 进行通信，交换选举结果。由于两者没有历史数据，即 zxid 无法比较，此时 id 值较大的节点 2 胜出，但是由于还没有超过半数的节点，所以 1 和 2 都保持 looking 的状态
3.  节点 3 启动，根据上面的分析，id 值最大的节点 3 胜出，而且超过半数的节点都参与了选举。节点 3 胜出成为了 Leader
4.  节点 4 启动，和 1~3 个节点通信，得知最新的 leader 为节点 3，而此时 zxid 也小于节点 3，所以承认了节点 3 的 leader 的角色
5.  节点 5 启动，和节点 4 一样，选取承认节点 3 的 leader 的角色

### 服务器运行过程中选主过程

1.节点 1 发起投票，**第一轮投票先投自己**，然后进入 Looking 等待的状态 2.其他的节点（如节点 2 ）收到对方的投票信息。节点 2 在 Looking 状态，则将自己的投票结果广播出去（此时走的是上图中左侧的 Looking 分支）；如果不在 Looking 状态，则直接告诉节点 1 当前的 Leader 是谁，就不要瞎折腾选举了（此时走的是上图右侧的 Leading/following 分支） 3.此时节点 1，收到了节点 2 的选举结果。如果节点 2 的 zxid 更大，那么清空投票箱，建立新的投票箱，广播自己最新的投票结果。在同一次选举中，如果在收到所有节点的投票结果后，如果投票箱中有一半以上的节点选出了某个节点，那么证明 leader 已经选出来了，投票也就终止了。否则一直循环

zookeeper 的选举，优先比较大 zxid，zxid 最大的节点代表拥有最新的数据。如果没有 zxid，如系统刚刚启动的时候，则比较机器的编号，优先选择编号大的

### 同步的过程

在选出 Leader 之后，zk 就进入状态同步的过程。其实就是把最新的 zxid 对应的日志数据，应用到其他的节点中。此 zxid 包含 follower 中写入日志但是未提交的 zxid 。称之为服务器提议缓存队列 committedLog 中的 zxid。

同步会完成三个 zxid 值的初始化。

**peerLastZxid**：该 learner 服务器最后处理的 zxid。 **minCommittedLog**：leader服务器提议缓存队列 committedLog 中的最小 zxid。 **maxCommittedLog**：leader服务器提议缓存队列 committedLog 中的最大 zxid。 系统会根据 learner 的**peerLastZxid**和 leader 的**minCommittedLog**，**maxCommittedLog**做出比较后做出不同的同步策略

#### 直接差异化同步

场景：**peerLastZxid**介于**minCommittedLogZxid**和**maxCommittedLogZxid**间

此种场景出现在，上文提到过的，Leader 发出了同步请求，但是还没有 commit 就 down 了。 leader 会发送 Proposal 数据包，以及 commit 指令数据包。新选出的 leader 继续完成上一任 leader 未完成的工作。

例如此刻Leader提议的缓存队列为 0x20001，0x20002，0x20003，0x20004，此处learn的peerLastZxid为0x20002，Leader会将0x20003和0x20004两个提议同步给learner

#### 先回滚在差异化同步/仅回滚同步

此种场景出现在，上文提到过的，Leader写入本地事务日志后，还没发出同步请求，就down了，然后在同步日志的时候作为learner出现。

例如即将要 down 掉的 leader 节点 1，已经处理了 0x20001，0x20002，在处理 0x20003 时还没发出提议就 down 了。后来节点 2 当选为新 leader，同步数据的时候，节点 1 又神奇复活。如果新 leader 还没有处理新事务，新 leader 的队列为，0x20001, 0x20002，那么仅让节点 1 回滚到 0x20002 节点处，0x20003 日志废弃，称之为仅回滚同步。如果新 leader 已经处理 0x30001 , 0x30002 事务，那么新 leader 此处队列为0x20001，0x20002，0x30001，0x30002，那么让节点 1 先回滚，到 0x20002 处，再差异化同步0x30001，0x30002。

#### 全量同步

**peerLastZxid**小于**minCommittedLogZxid**或者leader上面没有缓存队列。leader直接使用SNAP命令进行全量同步

所有事务请求必须由一个全局唯一的服务器来协调处理，这个服务器就是 Leader 服务器，其他的服务器就是follower。leader 服务器把客户端的失去请求转化成一个事务 Proposal（提议），并把这个 Proposal 分发给集群中的所有 Follower 服务器。之后 Leader 服务器需要等待所有Follower 服务器的反馈，一旦超过半数的 Follower 服务器进行了正确的反馈，那么 Leader 就会再次向所有的Follower 服务器发送 Commit 消息，要求各个 follower 节点对前面的一个 Proposal 进行提交;

### 顺序一致性模型：

讲Zookeeper的数据同步时，提到zookeeper并不是强一致性服务，它是一个最终一致性模 型，具体情况如图所示。
![[Pasted image 20230418155115.png]]

ClientA/B/C假设只串行执行， clientA更新zookeeper上的一个值x。ClientB和clientC分别读取集群的 不同副本，返回的x的值是不一样的。clientC的读取操作是发生在clientB之后，但是却读到了过期的 值。很明显，这是一种弱一致模型。如果用它来实现锁机制是有问题的。

顺序一致性提供了更强的一致性保证，我们来观察图-5所示，从时间轴来看，B0发生在A0之前，读取 的值是0，B2发生在A0之后，读取到的x的值为1.而读操作B1/C0/C1和写操作A0在时间轴上有重叠 ，因此他们可能读到旧的值为0，也可能读到新的值1. 但是在强顺序一致性模型中，如果B1得到的x的 值为1，那么C1看到的值也一定是1。
![[Pasted image 20230418155128.png]]
需要注意的是：由于网络的延迟以及系统本身执行请求的不确定性，会导致请求发起的早的客户端不一 定会在服务端执行得早。最终以服务端执行的结果为准。

简单来说：顺序一致性是针对单个操作，单个数据对象。属于CAP中C这个范畴。一个数据被更新后， 能够立马被后续的读操作读到。 但是zookeeper的顺序一致性实现是缩水版的，在下面这个网页中，可以看到官网对于一致性这块做了 解释

[https://zookeeper.apache.org/doc/r3.6.1/zookeeperProgrammers.html#ch_zkGuarantees](https://zookeeper.apache.org/doc/r3.6.1/zookeeperProgrammers.html#ch_zkGuarantees)

zookeeper不保证在每个实例中，两个不同的客户端具有相同的zookeeper数据视图，由于网络延迟等 因素，一个客户端可能会在另外一个客户端收到更改通知之前执行更新， 考虑到2个客户端A和B的场景，如果A把znode /a的值从0设置为1，然后告诉客户端B读取 /a， 则客户 端B可能会读取到旧的值0，具体取决于他连接到那个服务器，如果客户端A和B要读取必须要读取到相 同的值，那么client B在读取操作之前执行sync方法。 **zooKeeper.sync();**

除此之外，zookeeper基于zxid以及阻塞队列的方式来实现请求的顺序一致性。如果一个client连接到一 个最新的follower上，那么它read读取到了最新的数据，然后client由于网络原因重新连接到zookeeper 节点， 而这个时候连接到一个还没有完成数据同步的follower节点，那么这一次读到的数据不就是旧的数据吗？

实际上zookeeper处理了这种情况，client会记录自己已经读取到的最大的zxid，如果client重连到 server发现client的zxid比自己大，连接会失败。

zookeeper官网还说它保证了“Single System Image”，其解释为“A client will see the same view of the service regardless of the server that it connects to.”。其实由上面zxid的原理可以看出，它表达的意思是“client只要连接过一次zookeeper，就不会有历史的 倒退”。

# [Raft对比ZAB协议](https://my.oschina.net/pingpangkuangmo/blog/782702)

# 0 一致性问题

上述分别是针对如下算法实现的讨论：

-   Raft的实现[copycat](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fgithub.com%2Fatomix%2Fcopycat)，由于Raft算法本身已经介绍的相当清晰，copycat基本上和Raft算法保持一致
-   ZAB的实现ZooKeeper，由于ZooKeeper里面的很多实现细节并没有在ZAB里体现（ZAB里面只是一个大概，没有像Raft那么具体），所以这里讨论的都是ZooKeeper的实现

一致性算法在实现状态机这种应用时，有哪些常见的问题：

-   1 leader选举

-   1.1 一般的leader选举过程选举的轮次选举出来的leader要包含更多的日志
-   1.2 leader选举的效率会不会出现split vote？以及各自的特点是？
-   1.3 加入一个已经完成选举的集群怎么发现已完成选举的leader？加入过程是否对leader处理请求的过程造成阻塞？
-   1.4 leader选举的触发谁在负责检测需要进入leader选举？

-   2 上一轮次的leader

-   2.1 上一轮次的leader的残留的数据怎么处理？
-   2.2 怎么阻止之前的leader假死的问题

-   3 请求处理流程

-   3.1 请求处理的一般流程
-   3.2 日志的连续性问题
-   3.3 如何保证顺序

-   3.3.1 正常同步过程的顺序
-   3.3.2 异常过程的顺序follower挂掉又连接leader更换

-   3.4 请求处理流程的异常

-   4 分区的处理

下面分别来看看Raft和ZooKeeper怎么来解决的

# 1 leader选举

为什么要进行leader选举？

在实现一致性的方案，可以像base-paxos那样不需要leader选举，这种方案达成一件事情的一致性还好，面对多件事情的一致性就比较复杂了，所以**通过选举出一个leader来简化实现的复杂性。**

## 1.1 一般的leader选举过程

更多的有2个要素：

-   **1.1.1 选举轮次**
-   **1.1.2 leader包含更多的日志**

1.1.1 选举投票可能会多次轮番上演，为了区分，所以需要定义你的投票是属于哪个轮次的。

-   **Raft定义了term来表示选举轮次**
-   **ZooKeeper定义了electionEpoch来表示**

他们都需要在某个轮次内达成过半投票来结束选举过程

1.1.2 投票PK的过程，更多的是日志越新越多者获胜

在选举leader的时候，通常都希望

**选举出来的leader至少包含之前全部已提交的日志**

自然想到包含的日志越新越大那就越好。

通常都是比较最后一个日志记录，如何来定义最后一个日志记录？

有2种选择，一种就是所有日志中的最后一个日志，另一种就是所有已提交中的最后一个日志。目前Raft和ZooKeeper都是采用前一种方式。**日志的越新越大表示：轮次新的优先，然后才是同一轮次下日志记录大的优先**

-   Raft：term大的优先，然后entry的index大的优先
-   ZooKeeper：peerEpoch大的优先，然后zxid大的优先ZooKeeper有2个轮次，一个是选举轮次electionEpoch，另一个是日志的轮次peerEpoch（即表示这个日志是哪个轮次产生的）。而Raft则是只有一个轮次，相当于日志轮次和选举轮次共用了。至于ZooKeeper为什么要把这2个轮次分开，这个稍后再细究，有兴趣的可以一起研究。

但是有一个问题就是：通过上述的日志越新越大的比较方式能达到我们的上述希望吗？

特殊情况下是不能的，这个特殊情况详细见上述给出Raft算法赏析的这一部分
![[Pasted image 20230418155152.png]]
这个案例就是这种比较方式会选举出来的leader可能并不包含已经提交的日志，而Raft的做法则是对于日志的提交多加一个限制条件，即不能直接提交之前term的已过半的entry，即把这一部分的日志限制成未提交的日志，从而来实现上述的希望。

ZooKeeper呢？会不会出现这种情况？又是如何处理的？

ZooKeeper是不会出现这种情况的，因为ZooKeeper在每次leader选举完成之后，都会进行数据之间的同步纠正，所以每一个轮次，大家都日志内容都是统一的

而Raft在leader选举完成之后没有这个同步过程，而是靠之后的AppendEntries RPC请求的一致性检查来实现纠正过程，则就会出现上述案例中隔了几个轮次还不统一的现象

## 1.2 leader选举的效率

Raft中的每个server在某个term轮次内只能投一次票，哪个candidate先请求投票谁就可能先获得投票，这样就可能造成split vote，即各个candidate都没有收到过半的投票，Raft通过candidate设置不同的超时时间，来快速解决这个问题，使得先超时的candidate（在其他人还未超时时）优先请求来获得过半投票

**ZooKeeper中的每个server，在某个electionEpoch轮次内，可以投多次票，只要遇到更大的票就更新，然后分发新的投票给所有人。**这种情况下不存在split vote现象，同时有利于选出含有更新更多的日志的server，但是选举时间理论上相对Raft要花费的多。

## 1.3 加入一个已经完成选举的集群

-   1.3.1 怎么发现已完成选举的leader？
-   1.3.2 加入过程是否阻塞整个请求？

1.3.1 怎么发现已完成选举的leader？

一个server启动后（该server本来就属于该集群的成员配置之一，所以这里不是新加机器），如何加入一个已经选举完成的集群

-   Raft：比较简单，该server启动后，会收到leader的AppendEntries RPC,这时就会从RPC中获取leader信息，识别到leader，即使该leader是一个老的leader，之后新leader仍然会发送AppendEntries RPC,这时就会接收到新的leader了（因为新leader的term比老leader的term大，所以会更新leader）
-   ZooKeeper：该server启动后，会向所有的server发送投票通知，这时候就会收到处于LOOKING、FOLLOWING状态的server的投票（这种状态下的投票指向的leader），则该server放弃自己的投票，判断上述投票是否过半，过半则可以确认该投票的内容就是新的leader。

1.3.2 加入过程是否阻塞整个请求？

这个其实还要看对日志的设计是否是连续的

-   如果是连续的，则leader中只需要保存每个follower上一次的同步位置，这样在同步的时候就会自动将之前欠缺的数据补上，不会阻塞整个请求过程目前Raft的日志是依靠index来实现连续的
-   如果不是连续的，则在确认follower和leader当前数据的差异的时候，是需要获取leader当前数据的读锁，禁止这个期间对数据的修改。差异确定完成之后，释放读锁，允许leader数据被修改，每一个修改记录都要被保存起来，最后一一应用到新加入的follower中。目前ZooKeeper的日志zxid并不是严格连续的，允许有空洞。

## 1.4 leader选举的触发

触发一般有如下2个时机

-   server刚开始启动的时候，触发leader选举
-   leader选举完成之后，检测到超时触发，谁来检测？

-   Raft：目前只是follower在检测。follower有一个选举时间，在该时间内如果未收到leader的心跳信息，则follower转变成candidate，自增term发起新一轮的投票，leader遇到新的term则自动转变成follower的状态
-   ZooKeeper：leader和follower都有各自的检测超时方式，leader是检测是否过半follower心跳回复了，follower检测leader是否发送心跳了。一旦leader检测失败，则leader进入LOOKING状态，其他follower过一段时间因收不到leader心跳也会进入LOOKING状态，从而出发新的leader选举。一旦follower检测失败了，则该follower进入LOOKING状态，此时leader和其他follower仍然保持良好，则该follower仍然是去学习上述leader的投票，而不是触发新一轮的leader选举

# 2 上一轮次的leader

## 2.1 上一轮次的leader的残留的数据怎么处理？

首先看下上一轮次的leader在挂或者失去leader位置之前，会有哪些数据？

-   已过半复制的日志
-   未过半复制的日志

一个日志是否被过半复制，是否被提交，这些信息是由leader才能知晓的，那么下一个leader该如何来判定这些日志呢？

下面分别来看看Raft和ZooKeeper的处理策略：

-   Raft：对于之前term的过半或未过半复制的日志采取的是保守的策略，全部判定为未提交，只有当当前term的日志过半了，才会顺便将之前term的日志进行提交
-   ZooKeeper：采取激进的策略，对于所有过半还是未过半的日志都判定为提交，都将其应用到状态机中

Raft的保守策略更多是因为Raft在leader选举完成之后，没有同步更新过程来保持和leader一致（在可以对外处理请求之前的这一同步过程）。而ZooKeeper是有该过程的

## 2.2 怎么阻止上一轮次的leader假死的问题

这其实就和实现有密切的关系了。

-   Raft的copycat实现为：每个follower开通一个复制数据的RPC接口，谁都可以连接并调用该接口，所以Raft需要来阻止上一轮次的leader的调用。每一轮次都会有对应的轮次号，用来进行区分，Raft的轮次号就是term，一旦旧leader对follower发送请求，follower会发现当前请求term小于自己的term，则直接忽略掉该请求，自然就解决了旧leader的干扰问题
-   ZooKeeper：一旦server进入leader选举状态则该follower会关闭与leader之间的连接，所以旧leader就无法发送复制数据的请求到新的follower了，也就无法造成干扰了

# 3 请求处理流程

## 3.1 请求处理的一般流程

这个过程对比Raft和ZooKeeper基本上是一致的，大致过程都是过半复制

先来看下Raft：

-   client连接follower或者leader，如果连接的是follower则，follower会把client的请求(写请求，读请求则自身就可以直接处理)转发到leader
-   leader接收到client的请求，将该请求转换成entry，写入到自己的日志中，得到在日志中的index，会将该entry发送给所有的follower(实际上是批量的entries)
-   follower接收到leader的AppendEntries RPC请求之后，会将leader传过来的批量entries写入到文件中（通常并没有立即刷新到磁盘），然后向leader回复OK
-   leader收到过半的OK回复之后，就认为可以提交了，然后应用到leader自己的状态机中，leader更新commitIndex，应用完毕后回复客户端
-   在下一次leader发给follower的心跳中，会将leader的commitIndex传递给follower，follower发现commitIndex更新了则也将commitIndex之前的日志都进行提交和应用到状态机中

再来看看ZooKeeper：

-   client连接follower或者leader，如果连接的是follower则，follower会把client的请求(写请求，读请求则自身就可以直接处理)转发到leader
-   leader接收到client的请求，将该请求转换成一个议案，写入到自己的日志中，会将该议案发送给所有的follower(这里只是单个发送)
-   follower接收到leader的议案请求之后，会将该议案写入到文件中（通常并没有立即刷新到磁盘），然后向leader回复OK
-   leader收到过半的OK回复之后，就认为可以提交了，leader会向所有的follower发送一个提交上述议案的请求，同时leader自己也会提交该议案，应用到自己的状态机中，完毕后回复客户端
-   follower在接收到leader传过来的提交议案请求之后，对该议案进行提交，应用到状态机中

## 3.2 日志的连续性问题

在需要保证顺序性的前提下，在利用一致性算法实现状态机的时候，到底是实现连续性日志好呢还是实现非连续性日志好呢？

-   如果是连续性日志，则leader在分发给各个follower的时候，只需要记录每个follower目前已经同步的index即可，如Raft
-   如果是非连续性日志，如ZooKeeper，则leader需要为每个follower单独保存一个队列，用于存放所有的改动，如ZooKeeper，一旦是队列就引入了一个问题即顺序性问题，即follower在和leader进行同步的时候，需要阻塞leader处理写请求，先将follower和leader之间的差异数据先放入队列，完成之后，解除阻塞，允许leader处理写请求，即允许往该队列中放入新的写请求，从而来保证顺序性

还有在复制和提交的时候：

-   连续性日志可以批量进行
-   非连续性日志则只能一个一个来复制和提交

其他有待后续补充

## 3.3 如何保证顺序

具体顺序是什么？

这个就是先到达leader的请求，先被应用到状态机。这就需要看正常运行过程、异常出现过程都是怎么来保证顺序的

3.3.1 正常同步过程的顺序

-   Raft对请求先转换成entry，复制时，也是按照leader中log的顺序复制给follower的，对entry的提交是按index进行顺序提交的，是可以保证顺序的
-   ZooKeeper在提交议案的时候也是按顺序写入各个follower对应在leader中的队列，然后follower必然是按照顺序来接收到议案的，对于议案的过半提交也都是一个个来进行的

3.3.2 异常过程的顺序保证

如follower挂掉又重启的过程：

-   Raft：重启之后，由于leader的AppendEntries RPC调用，识别到leader，leader仍然会按照leader的log进行顺序复制，也不用关心在复制期间新的添加的日志，在下一次同步中自动会同步
-   ZooKeeper：重启之后，需要和当前leader数据之间进行差异的确定，同时期间又有新的请求到来，所以需要暂时获取leader数据的读锁，禁止此期间的数据更改，先将差异的数据先放入队列，差异确定完毕之后，还需要将leader中已提交的议案和未提交的议案也全部放入队列，即ZooKeeper的如下2个集合数据然后再释放读锁，允许leader进行处理写数据的请求，该请求自然就添加在了上述队列的后面，从而保证了队列中的数据是有序的，从而保证发给follower的数据是有序的，follower也是一个个进行确认的，所以对于leader的回复也是有序的

-   ConcurrentMap<Long, Proposal> outstandingProposalsLeader拥有的属性，每当提出一个议案，都会将该议案存放至outstandingProposals，一旦议案被过半认同了，就要提交该议案，则从outstandingProposals中删除该议案
-   ConcurrentLinkedQueue<\Proposal> toBeAppliedLeader拥有的属性，每当准备提交一个议案，就会将该议案存放至该列表中，一旦议案应用到ZooKeeper的内存树中了，然后就可以将该议案从toBeApplied中删除

如果是leader挂了之后，重新选举出leader，会不会有乱序的问题？

-   Raft：Raft对于之前term的entry被过半复制暂不提交，只有当本term的数据提交了才能将之前term的数据一起提交，也是能保证顺序的
-   ZooKeeper:ZooKeeper每次leader选举之后都会进行数据同步，不会有乱序问题

## 3.4 请求处理流程的异常

一旦leader发给follower的数据出现超时等异常

-   Raft：会不断重试，并且接口是幂等的
-   ZooKeeper：follower会断开与leader之间的连接，重新加入该集群，加入逻辑前面已经说了

# 4 分区的应对

目前ZooKeeper和Raft都是过半即可，所以对于分区是容忍的。如5台机器，分区发生后分成2部分，一部分3台，另一部分2台，这2部分之间无法相互通信

其中，含有3台的那部分，仍然可以凑成一个过半，仍然可以对外提供服务，但是它不允许有server再挂了，一旦再挂一台则就全部不可用了。

含有2台的那部分，则无法提供服务，即只要连接的是这2台机器，都无法执行相关请求。

所以ZooKeeper和Raft在一旦分区发生的情况下是是牺牲了高可用来保证一致性，即CAP理论中的CP。但是在没有分区发生的情况下既能保证高可用又能保证一致性，所以更想说的是所谓的CAP二者取其一，并不是说该系统一直保持CA或者CP或者AP，而是一个会变化的过程。在没有分区出现的情况下，既可以保证C又可以保证A，在分区出现的情况下，那就需要从C和A中选择一样。ZooKeeper和Raft则都是选择了C。

# Reference
https://cloud.tencent.com/developer/article/2072936?areaSource=&traceId=
https://cloud.tencent.com/developer/article/2072937?areaSource=&traceId=
https://cloud.tencent.com/developer/article/2072938?areaSource=&traceId=
https://cloud.tencent.com/developer/article/2072939?areaSource=&traceId=
[https://my.oschina.net/pingpangkuangmo/blog/782702](https://my.oschina.net/pingpangkuangmo/blog/782702)