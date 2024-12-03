#review/redis  
# **一、redis4.0之基于LFU的热点key发现机制**

业务中存在访问热点是在所难免的，然而如何发现热点key一直困扰着许多用户，redis4.0为我们带来了许多新特性，其中便包括基于LFU的热点key发现机制。

## **Redis中的LFU思路**

Least Frequently Used——简称LFU，意为最不经常使用，是redis4.0新增的一类内存逐出策略，

从LFU的字面意思我们很容易联想到key的访问频率，但是4.0最初版本仅用来做内存逐出，对于访问频率并没有很好的记录，那么经过一番改造，redis于4.0.3版本开始正式支持基于LFU的热点key发现机制。它也是基于局部性原理：如果一个数据在最近一段时间内使用次数最少，那么在将来一段时间内被使用的可能性也很小

在`LFU`算法中，可以为每个key维护一个计数器。每次key被访问的时候，计数器增大。计数器越大，可以约等于访问越频繁。

## **1.1、LFU算法介绍**

在redis中每个对象都有24 bits空间来记录LRU/LFU信息：
当这24 bits用作LFU时，其被分为两部分：

1.高16位用来记录访问时间（单位为分钟）
2.低8位用来记录访问频率，简称counter

### **1.1.1、counter：基于概率的对数计数器**

这里读者可能会有疑问，8 bits最大值也就是255，只用8位来记录访问频率够用吗？对于counter，redis用了一个trick的手段，counter并不是一个简单的线性计数器，而是用基于概率的对数计数器来实现：

对应的概率分布计算公式为：

```
1/((counter-LFU_INIT_VAL)*server.lfu_log_factor+1)
```

`counter`并不是简单的访问一次就+1，而是采用了一个0-1之间的p因子控制增长。`counter`最大值为255。取一个0-1之间的随机数r与p比较，当`r<p`时，才增加`counter`控制产出的策略。p取决于当前`counter`值与`lfu_log_factor`因子，`counter`值与`lfu_log_factor`因子越大，p越小，`r<p`的概率也越小，`counter`增长的概率也就越小。

LRU本质上是一个概率计数器，称为morris counter.随着访问次数的增加,counter的增加会越来越缓慢。如下是访问次数与counter值之间的关系  

factor即server.lfu_log_facotr配置值，默认为10.可以看到,一个key访问一千万次以后counter值才会到达255.factor值越小, counter越灵敏.可见`counter`增长与访问次数呈现对数增长的趋势，随着访问次数越来越大，`counter`增长的越来越慢。

其中LFU_INIT_VAL为5，我们看下概率分布图会有一个更直观的认识，以默认server.lfu_log_factor=10为例：

![[Pasted image 20241203180702.png]]
从上图可以看到，counter越大，其增加的概率越小，8 bits也足够记录很高的访问频率，

也就是说，默认server.lfu_log_factor为10的情况下，8 bits的counter可以表示1百万的访问频率。

### **1.1.2、新生key策略**

另外一个问题是，当创建新对象的时候，对象的`counter`如果为0，很容易就会被淘汰掉，还需要为新生key设置一个初始`counter`，`createObject`:
`counter`会被初始化为`LFU_INIT_VAL`，默认5。

### **1.1.3、counter的衰减因子**

从上一小节的counter增长函数LFULogIncr中我们可以看到，随着key的访问量增长，counter最终都会收敛为255，这就带来一个问题，如果counter只增长不衰减就无法区分热点key。

为了解决这个问题，redis提供了衰减因子server.lfu_decay_time，其单位为分钟，计算方法也很简单，如果一个key长时间没有访问那么它的计数器counter就要减少，减少的值由衰减因子来控制：

默认为1的情况下也就是N分钟内没有访问，counter就要减N。

函数首先取得高16 bits的最近降低时间`ldt`与低8 bits的计数器`counter`，然后根据配置的`lfu_decay_time`计算应该降低多少。

`LFUTimeElapsed`用来计算当前时间与`ldt`的差值：

具体是当前时间转化成分钟数后取低16bits，然后计算与`ldt`的差值`now-ldt`。当`ldt>now`时，默认为过了一个周期(16bits，最大65535)，取值`65535-ldt+now`。

然后用差值与配置`lfu_decay_time`相除，`LFUTimeElapsed(ldt)/ server.lfu_decay_time`，已过去n个`lfu_decay_time`，则将`counter`减少n，`counter - num_periods`。

### **1.1.4、LFU配置**

`Redis`4.0之后为`maxmemory_policy`淘汰策略添加了两个`LFU`模式：

•`volatile-lfu`：对有过期时间的key采用`LFU`淘汰算法

•`allkeys-lfu`：对全部key采用`LFU`淘汰算法

还有2个配置可以调整`LFU`算法：

```
lfu-log-factor 10
lfu-decay-time 1
```

`lfu-log-factor`可以调整计数器`counter`的增长速度，`lfu-log-factor`越大，`counter`增长的越慢。

`lfu-decay-time`是一个以分钟为单位的数值，可以调整`counter`的减少速度

1.热点key发现

介绍完LFU算法，接下来就是我们关心的热点key发现机制。

其核心就是在每次对key进行读写访问时，更新LFU的24 bits域代表的访问时间和counter，这样每个key就可以获得正确的LFU值：

# **二、redis6.0 客户端缓存方案**

## **1.1 client cache的问题**

clientcache的问题是缓存应该何时失效，更确切的说是如何保持与远端数据的一致性。为client cache设置过期时间是一个选择，但时间设置多久是一个问题。太长会有时效性问题，太短缓存的效果会打折扣。

## **1.2 redis 6.0 的解决方式**

### **1.2.1 整体思想**

redis在服务端记录访问的连接和相关的key， 当key有变化时，通知相应的连接(应用)。**应用收到请求后自行处理有变化的key**, 进而实现client cache与redis的一致。
![[Pasted image 20241203180757.png]]
﻿﻿redis对客户端缓存的支持方式被称为**Tracking**，分为两种模式：默认模式，广播模式。

### **1.2.2 默认模式**

Server 端记录每个Client访问的Key，当发生变更时，向client推送数据过期消息。

当tracking开启时， Redis会「记住」每个客户端请求的 key，当 key的值发现变化时会发送失效信息给客户端 (invalidation message)。失效信息可以通过 RESP3协议发送给请求的客户端，或者转发给一个不同的连接 (支持 RESP2 + Pub/Sub) 的客户端。

Server 端将 Client 访问的 key以及该 key 对应的客户端 ID 列表信息存储在全局唯一的表（TrackingTable），当表满了，回移除最老的记录，同时触发该记录已过期的通知给客户端。

每个 Redis 客户端又有一个唯一的数字 ID，TrackingTable 存储着每一个 Client ID，当连接断开后，清除该 ID 对应的记录。

TrackingTable 表中记录的 Key 信息不考虑是哪个 database 的，虽然访问的是 db1 的 key，db2 同名 key 修改时会客户端收到过期提示，但这样做会减少系统的复杂性，以及表的存储数据量。

Redis 用TrackingTable存储键的指针和客户端 ID 的映射关系。因为键对象的指针就是内存地址，也就是长整型数据。客户端缓存的相关操作就是对该数据的增删改查：
![[Pasted image 20241203180841.png]]
•优点：只对Client发送其访问过的被修改的数据

•缺点：Server端需要额外存储较大的数据量。

### **1.2.3 广播模式**

客户端订阅key前缀的广播(空串表示订阅所有失效广播)，服务端记录key前缀与client的对应关系。当相匹配的key发生变化时，通知client。

当广播模式 (broadcasting) 开启时，服务器不会记住给定客户端访问了哪些键，因此这种模式在服务器端根本不消耗任何内存。

在这个模式下，服务端会给客户端广播所有 key 的失效情况，如果 key 被频繁修改，服务端会发送大量的失效广播消息，这就会消耗大量的网络带宽资源。

所以，在实际应用中，我们设置让客户端注册只跟踪指定前缀的 key，当注册跟踪的 key 前缀匹配被修改，服务端就会把失效消息广播给所有关注这个 key前缀的客户端。

这种监测带有前缀的 key 的广播模式，和我们对 key 的命名规范非常匹配。我们在实际应用时，会给同一业务下的 key 设置相同的业务名前缀，所以，我们就可以非常方便地使用广播模式。
![[Pasted image 20241203180859.png]]
•优点：服务端记录信息比较少

•缺点：client会收到自己未访问过的key的失效通知。

### 1.2.4 RESP3协议

redis6.0开始使用新的协议RESP3。该协议增加了很多数据类型。

# 三、参考

1.﻿https://redis.io/docs/manual/client-side-caching/  
  
# Reference
https://mp.weixin.qq.com/s/EdYDCGxZDv3oHfBBQcPF2w