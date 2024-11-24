![](https://zhuanlan.zhihu.com/p/526674700)  
# 开篇语

本论文的观点比较新颖，抓住人们通常容易忽略掉的热点问题——KVS系统本身的热点问题，由于缺少热点感知，热点Key-Value pairs随机分布在collision chain中，导致热点访问性能严重下降。论文中的hotspot-aware和lock-free设计，两者缺一不可，对本人有较大启发，尤其是lock-free rehash解题思路。

#  0   摘要 

内存KVSes广泛应用于缓存热数据，解决基于disk的本地存储或是分布式系统中的热点问题。然而，内存KVSes本身的热点问题，很容易被忽视。现有KVSes由于缺少hotspot-awareness，严重影响其性能和可靠性，尤其是在highly skewed workloads场景。

本文深入探讨基于内存索引结构的KVS中hotspot-aware设计问题。首先分析理想hotspot-aware索引潜在优势，然后讨论实际hotspot-awareness设计存在的挑战，比如：热点偏移和并发访问。基于这些分析结果，提出一种新型hotspot-aware KVS——HotRing，适用于大量并发访问小部分索引数据的应用场景。HotRing的关键设计特点：

• 基于有序环哈希索引结构，通过直接移动head指针，快速访问热点数据；

• 应用轻量级热点偏移检测策略，实时检测热点偏移问题；

• 无锁设计结构，充分发挥多核CPU性能；

实验证明，在highly skewed workloads场景，相比现有KVS，HotRing性能提升~2.58x。

  Q1.     Highly Skewed wolkload，KVS中的items访问，极度不均衡，详见§1。  

  Q2.     HotRing为NoSQL Tair子组件，广泛应用于阿里内部和阿里云环境。  

# 1  概述 

内存KVS作为存储基础设施中必不可少的组件，用于缓存热点数据，有助于提升系统整体性能和吞吐量，尤其是在每秒数十亿高并发请求的应用场景。许多现有KVS，比如：Memcached[44]、Redis[31]及其变体[8,15,17,33]，广泛应用于企业生产环境中，如：Facebook、Amazon、Twitter和LinkedIn。

热点问题，是现实环境中普遍存在的问题，备受关注和研究[4,10,20,23,27]：

• 有许多方法可以解决集群范围的热点问题，如：一致性哈希[29]、数据迁移[9,11,46]、前端数据缓存[16,26,32,36]；

• 另外，也有一些办法可以很好解决单机热点问题，如：计算机存储体系结构，采用分层存储架构（disk，RAM，CPU cache），在低时延存储介质中保存最近频繁访问的数据块。

• 一些存储系统，如：LevelDB[18]、RocksDB[14]，利用内存KVSes管理热点item。

![](https://pic1.zhimg.com/80/v2-d695282fd722202d6b9a2c57d5c8530c  720w.webp)

然而，内存KVS本身的热点问题一直被忽视。我们收集统计了阿里云生产环境中内存KVSes的访问情况，如Figure 1所示，约60%（Daily case）~90%（Extreme case）的请求集中访问1%的Key。可能造成该现象的原因有以下几点：

• 在线应用的活跃用户持续增长。诸如在线促销活动、即时新闻等这类实时事件，很容易在短时间内吸引数十亿次点击访问量，如何快速响应热点数据访问至关重要。据报道称，时延每增长0.1s，会导致Amazon交易额下降1%；每增长0.5s，会导致Google搜索流量下降20%。

• 针对该类应用的基础设施通常都比较复杂，很容易出现一些很小的错误，如：软件BUG、配置错误等，导致重复访问某些key，出现虚假热点。因此，保障KVS高可靠和高性能尤其重要。

现有的一些索引数据结构可以用于实现KVS，如：跳跃表[14,18]、平衡树/字典树（Masstree[37]）、哈希表（Memcached[44]、MemC3[15]、MICA[33]、FASTER[8]），其中哈希表查找性能最优且最受欢迎。然而，这些实现大多数都是hotspot-unaware，采用相同策略无差别管理所有item，导致访问热item与冷item需要相同的内存访问次数。根据理论分析（详见2.2），与理想情况相比，现有哈希索引结构需要额外大量工作用于识别热点item。虽然现有一些机制可以减少内存访问次数，但收益甚微，比如：CPU cache有助于加速热点数据访问，但容量仅有32MB；Rehash可以减少冲突链长度，但会显著增加内存占用。这些解决办法为我们进一步优化highly skewed workloads场景热点问题，提供一些启示和帮助。

本论文提出一种hotspot-aware KVS——HotRing，采用哈希索引结构优化热点问题。HotRing最初设计想法，将查找item所需的内存访问次数与其热度关联，热点item需要更少的内存访问次数。实现这个目的面临两点挑战：

• 热点转移，应用访问模式会随时间变化，需要及时检测热点item并调整。HotRing采用ordered-ring结构替代冲突链，热点item变化时，只需移动哈希buckets头指针即可。同时，采用轻量级检测机制，实时检测热点item变化。

• 并发访问，需要支持热点items高并发访问。HotRing采用无锁数据结构设计[19,50]。

本论文进行了大量实验，对比测试HotRing与其他基于无锁链式哈希结构KVS的性能表现，实验结果表明：

• highly skewed workload场景，HotRing查询性能高达565M ops/sec，提升~2.58x；

• in-place-updates和read-copy-update性能分别提升~2.17x和~1.32x；

# 2  背景&动机 

## 2.1 哈希索引与热点问题

![](https://pic3.zhimg.com/80/v2-3eb1e431b29a0243772f1eb0245dc1a2  720w.webp)

内存KVSes通常采用哈希索引结构实现，尤其是在上层应用不需要范围查找的场景。典型哈希索引结构，如Figure 2所示，Key查找，首先需要计算其哈希值并定位到对应的哈希bucket，然后逐项对比查找冲突链。具体实现时，n-bits哈希值可以拆分成两部分：哈希表部分和tag部分，每个item中包含tag部分，避免查找时对比长keys[8,33]。由于哈希索引结构hotspot-unaware，热点items可能均匀分布在所有的冲突链中。如果热点item位于冲突链尾，就会导致热点item查找比前面的item需要更多的内存访问次数。在highly skewed workloads场景，即使是轻微的额外内存访问开销，都会导致系统整体性能急剧下降。

现有一些办法可以用于减少热点item访问的额外内存访问开销，但效果并不明显，比如：

• 采用CPU caches加速热点数据访问（以64bytes cache line方式组织）。然而，对于绝大多数商用服务器而言，CPU caches容量仅有~32MB，而内存容量可以高达256GB，仅可以缓存0.012%的内存数据，远远无法满足实际应用需求。一些cache-friendly索引结构设计[8,33]，尝试用于提升CPU caches的利用率。

• 采用rehash缩短冲突链长度。然而，当哈希表本身较大时，不建议采用该方式。比如，连续两次rehash，第二次rehash需要2倍内存，但仅获得一半收益（取决于冲突链长度）。

总之，现有办法只能在一定程度上缓解热点问题的影响。

## 2.2 Hotspot-aware优势

传统的基于链式哈希索引，热item随机分布在各个冲突链中，访问热item和冷item需要相同的内存访问开销。假设有N个items存储在哈希表中，哈希桶数为B，则冲突链平均长度为L=N/B，在哈希表中查找一个item需要的平均内存访问次数：
![[Pasted image 20230424102703.png]]
理想情况下，hotspot-aware哈希索引设计，访问item所需要的内存访问次数与其热度成反比，item热度越高需要越少的内存访问开销。因此，可以采用Zipfian分布对item热度  x-th  和访问频率  f  建模：
![[Pasted image 20230424102721.png]]
为了简化分析，我们假设热点items均匀分布在B个buckets中，即每个bucket包含一个top B热点item，一个top B+1~2B热点item，以此类推。此时，如果冲突链中item按照访问频率降序排序，则查找一个item所需的内存访问次数：
![[Pasted image 20230424102730.png]]
其中  F(k)  表示冲突链中  k-th   item累计访问频率。

![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='397' height='169'></svg>)

上述两种哈希索引设计，查找一个item所需要的平均内存访问次数与冲突链长度关系，如Figure 3所示，我们发现，随着冲突链长度增长，hotspot-aware哈希索引的访问效率几户不受影响，充分证明了hotspot-aware设计的优势。

  Q3.     Zipfian分布：在自然语言语料库里，一个单词出现的频率与它在频率表里的排名成反比。频率最高的单词出现的频率大约是出现频率第二位的单词的2倍，是出现频率第三位单词的3倍。  

## 2.3 挑战与设计原则

Hotspot-aware哈希索引设计面临的挑战：

• 热点转移，在实际应用场景中，访问模式会随时间发生变化，因此，无法根据items热点简单排序，需要采用某种轻量级策略跟踪热点转移；

• 并发访问，需要支持热点items高并发读写访问；

针对热点转移问题的设计原则，采用移动head指针指向最热item或者是全局最优位置的方式，避免对items重排序。为了确保移动head指针后，仍能够访问到所有的items，我们采用ordered-ring结构——HotRing，取代冲突链式结构。虽然该设计无法完全达到理想hotspot-aware的效果，但实验表明其访问效率最够高效。另外，我们采用两种轻量级策略实时检测热点转移。

针对并发访问的设计原则，采用无锁结构设计，避免锁或同步开销。许多研究表明[2,5,21]，无锁设计可以显著提升系统整体性能。比如：基于原子Compare-And-Swap（CAS）的Read-Copy-Update（RCU）[13]和Hazard Pointers[40]。在HotRing设计中，我们采用现有的研究成果[19,50]，巧妙地解决了并发删除/插入的问题，同时将该无锁设计进一步扩展支持所有的基本操作，包括：热点检测、head指针移动、rehash。

  Q4.     CAS：  

  Q5.     RCU：  

# 3  设计实现 

## 3.1 ordered-ring哈希索引
![[Pasted image 20230424102921.png]]
HotRing的哈希索引结构设计，如Figure 4所示，采用哈希冲突环式结构，替换传统的哈希冲突链式结构，哈希表head指针可以指向冲突环中任意item。冲突环式设计，不但方便HotRing根据item热度移动head指针，而且支持从任意位置遍历所有items。

基于环式结构设计，存在一个比较严重的问题：当目标item不存在时，可能会引起无限循环遍历。因此，确认遍历查找操作何时安全结束尤其重要。但是，不能简单地以head指针指向的位置作为循环结束标志，因为存在并发修改操作，如：delete。HotRing采用ordered-ring结构设计解决上述问题。直观的，我们可以根据key排序，对于查找不存在的item的情况，当遇到连续的两个item，其key分别小于和大于目标key时，可以认为查找操作安全结束。更进一步，由于长key对比影响性能，可以优先比较item的tag。换句话说，可以根据tag和key对items排序，item   k  的rank可以表示为  orderk=(tagk, keyk)  。

在查找item   k  的过程中，假设当前正访问item   i  ，当下列任一条件满足时，表示本次查找操作安全结束：

• Hit
![[Pasted image 20230424103736.png]]
• Miss
![[Pasted image 20230424103745.png]]
如Figure 5所示，列举了查找操作所有可能的情况，图示以字典序对item排序，比如：item  C  排在item  A  后面： tagA < tagC  ；item  D  排在item  C  后面： tagC=tagD，keyC<keyD  。根据  tagA < tagB < tagC  可以确认item  B  不存在，根据  tagG < tagI < tagF  和  tagI < tagF < tagH  可以分别确认item  G  和item  H  不存在（对应head指针发生移动的情况）。

## 3.2 热点转移检测

ordered-ring哈希索引设计，可以轻松解决目标item是否存在的问题，遗留如何识别热点item的问题，以及如何根据热点变化移动head指针的问题。

根据哈希分布具有的强均匀分布特性，热点items均匀分布在所有的哈希buckets中，因此，我们只需要关注每个bucket中的热点item检测即可。在实际应用中，每个bucket中的冲突item数目相对较少，一般是5~10个，热点items通常也只有一个，占比10%~20%。因此，我们可以通过head指针指向唯一热点item的方式，提升热点item的访问性能，无需重新组织数据且节约内存开销。为了获得更好地性能收益，需要权衡考虑两个指标：

• 识别准确度，根据热点识别比例衡量；

• 检测延迟，表示从热点出现到成功检测的耗时；

我们首先介绍随机移动策略，可以在极低检测时延内识别热点item。然后我们提出一种更高识别准确度的办法——统计抽样策略，但需要相对较高的检测时延。

下文需要用的相关专业术语定义：

• hot item，head指针指向的item；

• cold item，hot item之外的items；

• hot access，访问hot item；

• cold access，访问cold item；

### 3.2.1 随机移动策略

随机移动策略的基本思想，根据即时决策（不记录任何历史元数据），周期性移动head指针指向一个可能的热点item。具体地，每线程分配一个线程局部参数，用于记录已处理的请求数。每处理R次请求后，线程决定是否需要移动head指针。若R-th请求为hot access，则head指针保持不变，否则，移动head指针指向本次cold access访问的item。参数R影响热点检测延时和识别准确度，R值越小，检测延时越低，但会导致频繁且无效的head指针移动操作。在highly skewed wolkload场景， head指针移动不会很频繁。因此，参数R的经验值默认为5，能提供较低的检测时延，同时对性能影响较小（详见Figure15(b)）。

随机移动策略只适用于highly skewed workloads场景，且每个冲突环仅存在一个hot item。

### 3.2.2 统计抽样策略
![[Pasted image 20230424104240.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='372' height='174'></svg>)

统计抽样策略的设计目标，提供更高的热点识别准确度，而检测时延不会明显增加。我们首先介绍HotRing的item和pointer数据结构，如何利用现有数据结构管理统计信息。然后详细描述统计抽样策略如何评估item的访问频率。最后我们提出一种最优head指针移动办法，可以处理同时存在多个热点itmes的情况。

1） 索引结构

现代服务器的物理地址仅占48bits（支持64-bits原子CAS更新），我们可以利用剩余的16bits记录元数据。如Figure 6(a)，HotRing的Head Pointer结构由3部分组成：Active标志（1bit）、Counter（15bits）和Address（48bits），其中Active标志用于控制统计抽样，Counter用于记录该冲突链的访问计数。HotRing的item结构，如Figure 6(b)，其Item Pointer成员结构由4部分组成：Rehash标志（1bit）、Occupied标志（1bit）、Counter（14bits）和Address（48bits），其中Rehash标志用于控制Rehash过程，Occupied标志用于控制并发访问，Counter用于记录item的访问计数。

2） 统计采样

如何高效动态识别热点，非常具有挑战性。Hash Table通常都比较大，包含227~230个buckets。如果同时频繁更新rings和items的统计计数，会严重影响性能。HotRing的周期性采样设计，可以同时获得高准确度和低性能开销。具体地，每线程采用线程局部参数R，记录已经处理的请求数。每处理R次请求后，线程决定是否需要触发新一轮采样（通过修改Head Pointer的Active标志位）。如果R-th访问为hot access，表示当前识别的热点仍然有效，无需重新采样。否则，意味着已经发生热点转移，需要重新采样。参数R经验值默认为5。当Head Pointer的Active标志置位，则后续访问该ring，需要同时更新ring Counter和item Counter。该采样过程需要额外的CAS操作，会有临时额外的内存访问开销。为了缩短采样时间，我们以每个ring中的item个数作为采样数。

3） 热点调整

根据采样统计计数，我们可以确定新的hot item，并移动Head指针指向该item。采样过程结束后，由最后一个访问线程负责访问频率计算和热点调整。首先，该线程采用CAS原语原子复位Active标志位，保证后续任务仅由该线程独占处理。然后，统计该ring中所有item的访问频率。itme  k  的访问频率可以表示为  nk/N  ，其中  N  表示ring的Counter， nk  表示item的Counter。并计算Head指针指向某个item可获得的收益，Head指针指向item  t（0<t<k） 可获得的收益  Wt  ，计算公式如下：
![[Pasted image 20230424104717.png]]
  Wt  表示Head指针指向item   t  时，访问该ring中的items所需的平均内存访问次数。因此，选择  min(Wt)  的item作为hot item，可加速热点数据访问。如果计算得到的热点位置与之前不同，则需要采用CAS原语更新head指针指向新hot item。Head指针更新完成后，该线程重置所有的Counter，为下次采样做准备。

统计采样策略同样可以处理多热点的情况，通过计算获取最优的热点位置（可能并不是最热的item），避免频繁移动Head指针。

  Q6.     上述计算获取的position只是相对最优，热点item还是随机分布在ring中。  

4） 写密集型热点RCU
![[Pasted image 20230424104730.png]]
对于update操作，若value长度小于等于8bytes，可以采用in-place-update办法。现代计算机都支持64-bits原子更新操作，因此，热点item的读、写操作可以等同处理。而对于更长的value，则需要采用RCU原语，如Figure 7所示，Update过程中，需要修改其前一个item指针重新指向新item。若update的是head item，则需要遍历整个ring获取其前一个item。也就是说，写密集型hot item会加热其前一个item。基于这一点，我们稍微调整统计采样策略，对于RCU update，增加其前一个item的Counter。这样有利于让写密集型hot item的前一个item成为新的hot item，加速RCU update。

  Q7.     为什么更新value需要替换整个item？item是一块连续的内存，inline value成员仅有8bytes？  

5） 热点继承

RCU更新或删除head item时，需要移动head指针指向其他item。如果采取随机移动的方式，会有很大概率指向cold item，导致频繁触发热点识别，严重影响性能。

首先，若ring中仅有一个item，则直接采用CAS修改head指针的方式完成更新或修改。若存在多个items，HotRing针对RCU更新和删除操作，分别设计不同的head指针移动策略。对于RCU更新，基于访问局部性原理，更新head指针指向更新后的head item即可。对于删除操作，head指针直接指向下一个item。

  Q8.     热点调整，需要多次反复采样统计才能趋于稳定，比较适合于读多写少的场景。  

## 3.3 并发操作
![[Pasted image 20230424104912.png]]
Head指针移动使得无锁设计变得更加复杂，主要体现在以下方面：

• 一方面，head指针移动操作可能与其他线程的修改操作并发执行，需要保证head指针有效性；

• 另一方面，更新或删除head item，需要正确、高效移动head指针；

本章节，我们介绍HotRing的并发访问控制机制，通过实现一套完整的无锁设计[19,50]，提升系统的并发和吞吐量。CAS原子操作可以保证不会存在两个线程同时修改item指针，若多个线程并发修改item指针，则只有一个线程会成功，其他线程都会失败，其他线程需要重新执行其操作。

1） Read

无需额外操作保证正确性，因此，read操作是完全lock-free。

2） Insertion

创建新item（如Figure 8(a) item   C  ），并修改前一个item指针。有可能存在两个并发insertion修改同一个item指针的情况，CAS原语保证仅有一个修改成功，另一个需要重试。

  Q9.     维持Ring有序性（tag排序），需要多次内存访问，写入开销大，适合读多写少的场景。  

3） Update

根据value长度，设计两种更新策略：对于value小于等于8bytes，采用CAS执行原地更新；对于更长的value，采用RCU更新。以RCU update&insertion为例，如Figure 8(a)，一个线程执行insert操作（item   C  ），需要修改item   B  的指针，另一个线程并发执行RCU更新操作（item B -> B’）。由于两个线程采用CAS修改不同的item指针，因此，这两个操作都会成功。然而，由于item   B  此时已不可见，即使insert操作成功执行，item   C  也无法被访问。Figure 8(b)场景同样存在该问题。为了解决该问题，HotRing引入Occupied标志位（如Figure 6(b)），分两步实施Update操作：首先，置位item   B  的Occupied标志（会引起并发insert操作重试）；然后，原子修改item   A  指针指向item   B’  ，并复位item   B’  的Occupied标志。

4） Deletion

删除操作，将待删除item的前一个item的指针重新指向其后一个item即可，在删除过程中，需要保证待删除item的指针不变。同样地，我们利用Occupied标志保证并发操作的正确性。以并发RCU更新和删除操作为例，如Figure 8(c)所示，RCU更新item   D->D’  ，需要修改item   B  的指针指向item   D’  ，而另一个线程并发执行删除item   B  。删除操作开始前先置位item B的Occupied标志，会引起并发更新操作失败重试，直到删除操作完成为止。

5） Head指针移动操作

为了保证head指针移动操作的有效性，需要解决以下两个主要问题：

• 热点调整触发的head指针移动操作，与其他常规操作并发；

• 更新或删除head item，触发head指针移动；

对于热点调整触发的head指针移动操作，采用Occupied标志位，保证并发操作的正确性。在移动head指针指向新item前，先置位该item的Occupied标志位，避免其他线程并发更新或删除该item。对于head item更新触发的head指针移动操作，在完成head item更新之后，移动head指针指向新版本item之前，需要保证置位新版本item的Occupied标志，避免其他线程更新或删除新版本item。对于head item删除触发的head指针移动操作，需要同时置位该item及其下一个item的Occupied标志，避免其他线程更新或删除其下一个item（见  3.2.2  ）。

## 3.4 Lock-free Rehash
![[Pasted image 20230424105220.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='554' height='132'></svg>)

随着不断插入新的数据，哈希冲突也随之增加，导致查找item需要更多额外的内存访问次数，严重影响KVSes性能。为了解决这个问题，HotRing引入弹性lock-free rehash策略。传统的rehash策略，只关心哈希表的负载因子（比如，冲突链平均长度），并未考虑热点item的影响（存在无效rehash现象）。HotRing根据访问开销（查找item所需的平均内存访问次数），决定是否需要执行rehash，包含以下三个步骤：

• 初始化

HotRing创建一个后台线程负责rehash。Rehash线程首先初始化一个新的哈希表，其大小为原来的2倍。如图Figure 9(a)所示，每个head指针分裂为两个新的head指针，哈希值有效位由  k  扩展为  k+1  。HotRing根据tag值范围将数据集分裂为两个子集： [0, T/2)  、  [T/2，T)  （  T=2n-k  ，  n  为哈希值比特位数），分别由两个新的head指针管理。与此同时，Rehash线程创建一个rehash node，包含两个rehash item，分别对应两个新的head指针。Rehash item与data item具有相同的格式，唯一区别是无有效KV pair，可以通过Rehash标志位识别rehash item。如Figure 9(b)所示，初始化阶段，两个rehash item的tag值分别设置为  0  和  T/2  。

• 分裂

在分裂阶段，Rehash线程将两个rehash item插入到冲突环中，实现哈希分裂。如Figure 9(c)所示，两个rehash item分别插入到item B和item E的前面，作为哈希分裂的边界。当插入操作完成后，新的哈希table处于活跃状态，后续的访问操作均由新table提供服务，而原先针对旧table的访问操作，可以通过判断Rehash标志继续进行。至此，所有items在逻辑上已经分裂成两部分，查找一个item最多只需遍历一半的items，比如，查找item F的遍历路径Head1 -> E -> F。

• 删除

在删除阶段，Rehash线程删除rehash node，如Figure 9(d)所示。但是，在删除操作执行前，需要维护一个过渡阶段，确保所有针对旧table的操作已完成，类似于RCU同步原语[13]。当所有针对旧table的访问操作已完成，就可以安全删除旧table和rehash node。整个删除过程，仅会阻塞rehash线程，对其他线程操作无影响。

  Q10.     触发rehash阈值多少？平均内存访问开销增长一倍？  

# 4 实验效果

本章节通过模拟真实工作负载，测试HotRing吞吐量和可扩展性能，并与其他KVS系统对比。同时，通过一系列实验测试，证明HotRing的关键设计点对性能的影响。

## 4.1 实验设置

1） 环境配置

![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='381' height='169'></svg>)
![[Pasted image 20230424105243.png]]
服务器硬件配置如Table 1所示，OS版本为CentOS 7.4（3.10 kernel）。为了获得更好的性能，每个线程绑定指定的CPU核心。

  Q11.     通常，作为KVS服务器的CPU性能都比较好，内存配置数百GB。  

  Q12.     实验集群规模多大？  

2） 工作负载
![[Pasted image 20230424105259.png]]
![](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='402' height='191'></svg>)

采用YCSB[10]工具模拟真实的工作负载（workload E scan操作除外）。对于每个item，其key长度设置为8bytes，value设置为8bytes（for in-place-update）和100bytes（for RCU）两种。每次实验测试，key总量固定为2.5亿，通过改变key-bucket-ratio控制冲突链平均长度。另外，通过调整YCSB中zipfian分布参数  θ  以模拟daily和extreme场景的工作负载。不同热点率  α  和zipfian分布参数  θ  下的热点访问率，如Table 2所示，结合Figure 1生产环境中工作负载分布，我们发现，  θ  值范围为[0.9，0.99]对应daily场景，  θ  值范围为[1,1.22]对应extreme场景。因此，  θ  值设置为0.99和1.22，分别用于模拟daily和extreme场景。

3） 基线

修改Memcached，实现lock-free链式哈希索引结构，支持CAS原语插入，并以此作为baseline。其他对比KVS系统包括：FASTER[7]、Masstree[30]（in-memory range索引结构，non-hash索引结构的代表）、lock-based Memcached。

为了保证公平对比，所有KVS系统的内存大小保持一致。对于每个测试，特殊说明除外，保持默认设置：64线程，YCSB参数  θ  值为1.22、workload B，HotRing的key-bucket-ratio为8。

## 4.2 对比测试

1） 整体性能
![[Pasted image 20230424105341.png]]
不同YCSB工作负载条件下的系统整体吞吐量对比，如Figure 10所示，其中，HotRing-r为采用随机移动策略，HotRing-s为采用抽样统计策略。我们发现：

• 在所有工作负载下（尤其是workload B/C），HotRing均取得更好地性能表现，HotRing-s的整体吞吐量是其他KVS的2.10x~7.75x。单线程和64线程吞吐量分别为12.90M ops/sec和565.36M ops/sec，表现出良好的可扩展性。

• 在workload D/F下，HotRing表现出良好的插入性能，HotRing-s插入操作吞吐量是其他KVS的1.81x~6.46x，得益于其ordered-ring索引结构和tag值排序，减小插入查找开销。

• 虽然HotRing-r的热点识别准确率低于HotRing-s（~7%）,但相比其他KVS，仍能显著提升系统整体吞吐量。

  Q13.     插入操作，Chaining Hash和FASTER都需要遍历整个冲突链，查找是否存在同名key。而HotRing平均只需要一半查找即可，且通过tag值比较。  

2） 冲突链长度
![[Pasted image 20230424105409.png]]
不同冲突链长度（key-bucket-ratio从2到16，YCSB workload B）条件下的系统吞吐量对比，如Figure 11（a）所示，我们发现：

• Key-bucket-ratio为2时，Chaining Hash和FASTER均表现出良好性能，但随着Key-bucket-ratio值增加，访问热点item的内存访问开销也随之增加，导致系统性能严重下降。

• 而HotRing即使在冲突链特别长的情况下，仍能表现出令人满意的性能。其read操作吞吐量，在Key-bucket-ratio为2时，分别是Chaining Hash和FASTER的1.93x和1.31x，而在Key-bucket-ratio为16时分别增加到4.02x和3.91x。得益于Ordered-ring结构和hotspot-aware设计，热点items总是就近于head指针。

3） Access skewness

不同zipfian分布参数  θ  （  θЄ[0.5，1.22]，  模拟不同程度的热点访问工作负载）条件下的读操作吞吐量对比，如Figure 11(b)所示，我们发现：

• 随着  θ  值增加（热点访问越来越集中），Chaining Hash和FASTER系统吞吐量并没有明显提升，原因在于该类系统对热点无感知。

• HotRing系统吞吐量随着  θ  值增加而明显提升，尤其是在  θ  值大于0.8时。即使θЄ[0.5，0.8]时（类似于多热点场景），HotRing-s性能也优于其他KVS系统。得益于HotRing-s的hotspot-aware设计，且可以很好处理多热点情况，采用采样统计策略计算最优head指针位置。

• HotRing-r由于无法处理多热点情况，在  θЄ[0.5，0.8]  时，出现head指针频繁移动的现象，影响性能。

4） RCU
![[Pasted image 20230424105437.png]]
不同写密集程度（用YCSB分别模拟50% write和write only两种场景，value大小为100bytes，用于证明热点item的RCU操作性能）条件下的更新操作吞吐量对比，如Figure 12所示，其中HotRing-s为采用采样统计策略（RCU更新操作会递增其前一个item的Counter），而HotrRing-s(w/o)为采用普通RCU策略（不更新前一个item的Counter），我们发现：

• HotRing-s(w/o)性能较差，甚至低于Chaining Hash和FASTER的性能。原因在于，其更新hot item时，需要遍历整个冲突环。

• HotRing-s采用采样统计策略优化hot item的RCU更新操作，显著提升性能。虽然在key-bucket-ratio为2时，由于额外内存访问（Counter标志和Occupied标志），其性能略低于FASTER。但是，随着key-bucket-ratio增大，额外内存访问的影响大大减小，在key-bucket-ratio为8时，其热点item更新操作吞吐量为Chaining Hash和FASTER的1.32x。

## 4.3 设计细节因素

本章节，通过一些列设计细节因素对比HotRing和传统Chaining Hash，说明hotspot-aware设计的优势。

1） Break-down cost
![[Pasted image 20230424105451.png]]
Read操作过程所涉及的不同调用函数的执行开销对比（key-bucket-ratio为8），如Figure 13所示，其中，HeadPointer为head指针移动开销，HeadItem为访问head item开销，Non-HeadItem为访问其他item开销，HashValue为哈希计算开销，Benchmark为读取解析命令开销，Other为系统内核开销。我们发现：

• 对于Chaining Hash，Non-HeadItem为主要开销，在  θ  值为1.22/0.99时分别为193ns/660ns。原因在于，hot item随机分布在冲突链中，导致访问开销剧增。

• 对于HotRing-r和HotRing-s，Non-HeadItem开销明显减小。尤其是HotRing-s，在  θ  值为1.22/0.99时，Non-HeadItem开销分别只有10ns/136ns。Non-HeadItem开销与热点识别准确率成负相关，以上结果表明，HotRing-s具有更高的热点识别准确度。

2） 检测延迟
![[Pasted image 20230424105510.png]]
热点发生转移后，系统吞吐量（workload C）随时间变化趋势，如Figure 14所示，我们发现：

• HotRing-r的检测延迟优于HotRing-s，但其吞吐量峰值低于HotRing-s，尤其是在  θ  值为0.99时（多热点场景）。

• Chaining Hash不受影响（hotspot-unaware）。

3） Read miss
![[Pasted image 20230424105518.png]]
Read miss情况下的吞吐量对比，如Figure 15(a)所示，我们发现，随着key-bucket-ratio增大，HotRing的性能优势约明显，在key-bucket-ratio为2和16时，其吞吐量分别是Chaining Hash的1.17x和1.8x。原因在于，HotRing最大只需要遍历一半items即可确定目标item是否存在。

4） Parameter R

不同工作负载条件下R值（R值越小，检测延迟越短，但可能引起频繁且无效的head指针移动）对系统吞吐量影响，如Figure 15(b)所示，我们发现，R值过小（Head指针移动频繁）或过大（head指针移动不及时），都会导致性能下降，R值设置为5比较适中。

5） 尾部延迟
![[Pasted image 20230424105528.png]]
统计10万次访问的时延分布，如Figure 16所示，在  θ  值为1.22时，99%的访问时延在2us以内，但长尾访问时延达8.8us。同样，  θ  值为0.99时，99%的访问时延在3us以内，而长尾访问时延达9.6us。原因在于，HotRing-s采用抽样统计策略，最后一个采样线程需要重新计算最优head指针位置。后续可以针对优化，将额外计算工作移至后台专门线程负责。

6） Lock-free rehash
![[Pasted image 20230424105541.png]]
使用YCSB工具构造50% read(  θ=1.22  )和50% insert工作负载，模拟哈希表大小不断增长的场景（初始key总量为2.5亿，key-bucket-ratio为8），测试lock-free rehash对系统吞吐量的影响，如Figure 17所示，其中，I、T、S分别表示rehash的三个阶段：initialization、transition、splitting。我们发现，随着数据量增长，rehash操作有利于保证系统吞吐量稳定。Rehash期间出现短暂的吞吐量下降，原因在于，新table暂时hotspot-unaware（需要一采样统计）。

# 5 相关研究

简述当前in-memory KVS相关研究现状和成果，详见原论文。

# 6 结论

总结HotRing的实验测试结果及其在Alibaba生产环境中使用情况，详见原论文。

#  References 
https://zhuanlan.zhihu.com/p/526674700