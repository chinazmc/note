![](https://zhuanlan.zhihu.com/p/430650789)  

# 一、前言

随着用户量级的快速增长，vivo官方商城v1.0的单体架构逐渐暴露出弊端：模块愈发臃肿、开发效率低下、性能出现瓶颈、系统维护困难。

从2017年开始启动的v2.0架构升级，基于业务模块进行垂直的系统物理拆分，拆分出来业务线各司其职，提供服务化的能力，共同支撑主站业务。

商品模块是整个链路的核心，模块的增多严重影响系统的性能，服务化改造势在必行。

本文将介绍vivo商城商品系统建设的过程中遇到的问题和解决方案，分享架构设计经验。

# 二、商品系统演进

将商品模块从商城拆分出来，独立为商品系统，逐渐向底层发展，为商城，搜索，会员、营销等提供基础标准化服务。

商品系统架构图如下：

![](https://pic4.zhimg.com/80/v2-60da753558dc66a0e064d6d11a5e88bf_720w.webp)

前期商品系统比较杂乱，包含业务模块比较多，如商品活动业务、秒杀业务，库存管理，随着业务的不断发展，商品系统承载更多的业务不利于系统扩展和维护。

故思考逐渐将商品业务逐渐下沉并作为最底层、最基础的业务系统，并为众多调用方提供高性能的服务，下面介绍商品系统的升级历史。

## 2.1 商品活动、赠品剥离

随着商品活动的不断增多，玩法多样，同时与活动相关的额外属性也相应增加，这些都并不是与商品信息强关联，更偏向于用户营销，不应该与核心商品业务耦合在一起，故将其合并入商城促销系统。

赠品不仅仅是手机、配件，有可能会是积分、会员等，这些放在商品系统都不合适，也不属于商品模块的内容，故同步将其合并入商城促销系统。

## 2.2 秒杀独立

众所周知，秒杀活动的特点是：

-   限时：时间范围很短，超过设置的时间就结束了
-   限量：商品数量很少，远低于实际库存
-   访问量大：价格低，可以吸引非常多的用户

基于以上特性，做好一个秒杀活动不是一蹴而就，由于系统资源共享，当突发的大流量冲击会造成商品系统其他业务拒绝服务，会对核心的交易链路造成阻塞的风险，故将其独立为单独的秒杀系统，单独对外提供服务。

## 2.3 代销系统成立

我们商城的主要销售品类还是手机以及手机配件等，商品的品类比较少，为了解决非手机商品品类不丰富的问题，运营考虑与知名电商进行合作，期望引入更多的商品品类。

为了方便后续扩展，以及对原有系统的不侵入性，我们经过考虑专门独立出一个子系统，用于承接代销业务，最后期望做成一个完备平台，后续通过提供开放API的方式让其他电商主动接入我们业务。

## 2.4 库存剥离

库存管理的痛点：

-   由于我们的库存都是到商品维度，仅仅一个字段标识数量，每次编辑商品都需要为商品调整库存，无法动态实现库存管理；
-   同时营销系统也有自己活动库存管理机制，入口分散，关联性较弱；
-   可售库存和活动库存管理的依据都是实际库存，造成容易配置错误。

基于以上痛点，同时为了更方便运营管理库存，也为未来使用实际库存进行销售打下基础，我们成立库存中心，并提供以下主要功能：

-   与ecms实际库存进行实时同步；
-   可以根据实际库存的仓库分布情况，计算商品的预计发货仓库和发货时间，从而计算商品预计送达时间；
-   完成低库存预警，可以根据可用库存、平均月销等进行计算，动态提醒运营订货。

# 三、挑战

作为最底层的系统，最主要的挑战就是具备稳定性，高性能，数据一致性的能力。

## 3.1 稳定性

-   避免单机瓶颈：根据压测选择合适的节点数量，不浪费，同时也能保证沟通，可以应对突发流量。
-   业务限流降级：对核心接口进行限流，优先保证系统可用，当流量对系统压力过大时将非核心业务进行降级，优先保证核心业务。
-   设置合理的超时时间：对Redis、数据库的访问设置合理超时时间，不宜过长，避免流量较大时导致应用线程被占满。
-   监控&告警：日志规范化，同时接入公司的日志监控和告警平台，做到主动发现问题并及时。
-   熔断：外部接口接入熔断，防止因为外部接口异常导致本系统受到影响。

## 3.2 高性能

**多级缓存**

为了提升查询速度，降低数据库的压力，我们采用多级缓存的方式，接口接入热点缓存组件，动态探测热点数据，如果是热点则直接从本地获取，如果不是热点则直接从redis获取。

**读写分离**

数据库采用读写分离架构，主库进行更新操作，从库负责查询操作。

**接口限流**

接入限流组件， 直接操作数据库的接口会进行限流，防止因为突发流量、或者不规范调用导致数据库压力增加，影响其他接口。

**不过早期也踩过一些坑：**

1、商品列表查询造成redis key过多，导致redis内存不够的风险

由于是列表查询，进行缓存的时候是对入参进行hash，获取唯一的key，由于入参商品较多，某些场景下入参是随时变化的，根据排列组合，会造成基本每次请求都会回源，再缓存，可能造成数据库拒绝服务或者redis内存溢出。

**方案一**：循环入参列表，每次从redis获取数据，然后返回；

![](https://pic3.zhimg.com/80/v2-e975300b34b8a428abe9a6bb1ca926ea_720w.webp)

这个方案解决了key过多导致内存溢出的问题，但是很明显，它增加了很多的网络交互，如果有几十个key，可想而知，对性能会有不小的影响，那有什么其他办法能减少网络交互呢，下面我们看方案二。

**方案二**：我们通过对原有的Redis 组件进行增强，由于Redis集群模式不支持mget，故我们采用pipeline的方式实现，先根据key计算出其所在的slot，然后聚合一次性提交，这样每个商品数据只需缓存一次即可，同时采用mget也大大提升了查询速度。

![](https://pic4.zhimg.com/80/v2-8a4af10d138d1c26b50665961dbe146b_720w.webp)

这就即解决了key值过多的问题，也解决了方案一中多次网络交互的问题，经过压测对比，方案二比方案一性能提升50%以上，key越多，效果越明显。

2、热点数据，导致redis单机瓶颈

商城经常有新品发布会，发布会结束后会直接跳转到新品商详页，此时新品商详页就会出现流量特别大且突发、数据单一，这就导致Redis节点负载不平衡，有些10%不到，有些达到90%多，而一些常规的扩容是没有效果的。

针对热点问题我们有以下解决方案：

-   key的散列，将key分散到不同的节点
-   采用本地缓存的方式

开始我们采用的是基于开源的Caffeine完成本地缓存组件，本地自动计算请求量，当达到一定的阀值就缓存数据，根据不同的业务场景缓存不同的时间，一般不超过15秒，主要解决热点数据的问题。

后来替换成我们自己研发的热点缓存组件，支持热点动态探测，热点上报，集群广播等功能。

# 3.3 数据一致性

1、对于Redis的数据一致性比较好解决，采用“Cache Aside Pattern”：

对于读请求采用先读缓存，命中直接返回，未命中读数据库再缓存。对于写请求采用先操作数据库，再删除缓存。

2、由于库存剥离出去，维护入口还是在商品系统，这就导致存在跨库操作，平常的单库事务无法解决。

开始我们采用异常捕获，本地事务回滚的方式，操作麻烦点，但也能解决这个问题。

后来我们通过开源的seata完成分布式事务组件，通过改写代码引入公司的基础组件，目前已经接入使用。

# 四、总结

本篇主要介绍商城商品系统如何进行拆分、并慢慢下沉为最基础的系统，使其职责更加单一，能够提供高性能的商品服务，并分享在此过程中遇到的技术问题和解决方案，后续会有库存系统的演进历史、分布式事务相关内容，敬请期待。

# Reference
https://zhuanlan.zhihu.com/p/430650789