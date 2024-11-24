#redis #延时队列 


有了 Redis，对于那些只有一组消费者的消息队列，使用 Redis 就可以非常轻松的搞定。Redis 的消息队列不是专业的消息队列，它没有非常多的高级特性， 没有 ack 保证，如果对消息的可靠性有着极致的追求，那么它就不适合使用。

所谓延时队列就是延时的消息队列，下面说一下一些业务场景

**实践场景**

订单支付失败，每隔一段时间提醒用户

用户并发量的情况，可以延时2分钟给用户发短信

**先来看看Redis实现普通的消息队列**

我们知道，对于专业的消息队列中间件，如Kafka和RabbitMQ，消费者在消费消息之前要进行一系列的繁琐过程。

如RabbitMQ发消息之前要创建 Exchange，再创建 Queue，还要将 Queue 和 Exchange 通过某种规则绑定起来，发消息的时候要指定 routingkey，还要控制头部信息

但是绝大 多数情况下，虽然我们的消息队列只有一组消费者，但还是需要经历上面一些过程。

有了 Redis，对于那些只有一组消费者的消息队列，使用 Redis 就可以非常轻松的搞定。Redis 的消息队列不是专业的消息队列，它没有非常多的高级特性， 没有 ack 保证，如果对消息的可靠性有着极致的追求，那么它就不适合使用

**异步消息队列基本实现**

Redis 的 list(列表) 数据结构常用来作为异步消息队列使用，使用 rpush/lpush 操作入队列， 使用 lpop 和 rpop 来出队列

[![](https://s4.51cto.com/oss/202009/29/ea075f2ee81f856fcb78149cd0b1b9d1.png)](https://s4.51cto.com/oss/202009/29/ea075f2ee81f856fcb78149cd0b1b9d1.png)

复制

```
> rpush queue 月伴飞鱼1 月伴飞鱼2 月伴飞鱼3 
(integer) 3 
> lpop queue 
"月伴飞鱼1" 
> llen queue 
(integer) 2 
```

**问题1:如果队列空了**

客户端是通过队列的 pop 操作来获取消息，然后进行处理。处理完了再接着获取消息， 再进行处理。如此循环往复，这便是作为队列消费者的客户端的生命周期。

可是如果队列空了，客户端就会陷入 pop 的死循环，不停地 pop，没有数据，接着再 pop， 又没有数据。这就是浪费生命的空轮询。空轮询不但拉高了客户端的 CPU，redis 的 QPS 也 会被拉高，如果这样空轮询的客户端有几十来个，Redis 的慢查询可能会显著增多。

通常我们使用 sleep 来解决这个问题，让线程睡一会，睡个 1s 钟就可以了。不但客户端 的 CPU 能降下来，Redis 的 QPS 也降下来了

**问题2:队列延迟**

用上面睡眠的办法可以解决问题。同时如果只有 1 个消费者，那么这个延迟就是 1s。如果有多个消费者，这个延迟会有所下降，因 为每个消费者的睡觉时间是岔开来的。

**有没有什么办法能显著降低延迟呢?**

那就是 blpop/brpop。

这两个指令的前缀字符 b 代表的是 blocking，也就是阻塞读。

阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。消 息的延迟几乎为零。用 blpop/brpop 替代前面的 lpop/rpop，就完美解决了上面的问题。

**问题3:空闲连接自动断开**

其实他还有个问题需要解决—— 空闲连接的问题。

如果线程一直阻塞在哪里，Redis 的客户端连接就成了闲置连接，闲置过久，服务器一般 会主动断开连接，减少闲置资源占用。这个时候 blpop/brpop 会抛出异常来。

所以编写客户端消费者的时候要小心，注意捕获异常，还要重试。

分布式锁冲突处理

假如客户端在处理请求时加分布式锁没加成功怎么办。

**一般有 3 种策略来处理加锁失败：**

1、直接抛出异常，通知用户稍后重试;

2、sleep 一会再重试;

3、将请求转移至延时队列，过一会再试;

**直接抛出特定类型的异常**

这种方式比较适合由用户直接发起的请求，用户看到错误对话框后，会先阅读对话框的内 容，再点击重试，这样就可以起到人工延时的效果。如果考虑到用户体验，可以由前端的代码 替代用户自己来进行延时重试控制。它本质上是对当前请求的放弃，由用户决定是否重新发起 新的请求。

**sleep**

sleep 会阻塞当前的消息处理线程，会导致队列的后续消息处理出现延迟。如果碰撞的比 较频繁或者队列里消息比较多，sleep 可能并不合适。如果因为个别死锁的 key 导致加锁不成 功，线程会彻底堵死，导致后续消息永远得不到及时处理。

**延时队列**

这种方式比较适合异步消息处理，将当前冲突的请求扔到另一个队列延后处理以避开冲突。

**延时队列的实现**

我们可以使用 zset这个命令，用设置好的时间戳作为score进行排序，使用 zadd score1 value1 ....命令就可以一直往内存中生产消息。再利用 zrangebysocre 查询符合条件的所有待处理的任务，通过循环执行队列任务即可。也可以通过 zrangebyscore key min max withscores limit 0 1 查询最早的一条任务，来进行消费

[![](https://s4.51cto.com/oss/202009/29/9dd9dcd01500a65cf07e6caf8d96d5fc.png)](https://s4.51cto.com/oss/202009/29/9dd9dcd01500a65cf07e6caf8d96d5fc.png)

```java
private Jedis jedis; 
 
public void redisDelayQueueTest() { 
    String key = "delay_queue"; 
 
    // 实际开发建议使用业务 ID 和随机生成的唯一 ID 作为 value, 随机生成的唯一 ID 可以保证消息的唯一性, 业务 ID 可以避免 value 携带的信息过多 
    String orderId1 = UUID.randomUUID().toString(); 
    jedis.zadd(queueKey, System.currentTimeMillis() + 5000, orderId1); 
 
    String orderId12 = UUID.randomUUID().toString(); 
    jedis.zadd(queueKey, System.currentTimeMillis() + 5000, orderId2); 
 
    new Thread() { 
        @Override 
        public void run() { 
            while (true) { 
                Set<String> resultList; 
                // 只获取第一条数据, 只获取不会移除数据 
                resultList = jedis.zrangebyscore(key, System.currentTimeMillis(), 0, 1); 
                if (resultList.size() == 0) { 
                    try { 
                        Thread.sleep(1000); 
                    } catch (InterruptedException e) { 
                        e.printStackTrace(); 
                        break; 
                    } 
                } else { 
                    // 移除数据获取到的数据 
                    if (jedis.zrem(key, resultList.get(0)) > 0) { 
                        String orderId = resultList.get(0); 
                        log.info("orderId = {}", resultList.get(0)); 
                        this.handleMsg(orderId); 
                    } 
                } 
            } 
        } 
    }.start(); 
} 
 
public void handleMsg(T msg) { 
    System.out.println(msg); 
} 
```

上面的实现, 在多线程逻辑上也是没有问题的, 假设有两个线程 T1, T2和其他更多线程, 处理逻辑如下, 保证了多线程情况下只有一个线程处理了对应的消息:

1.T1, T2 和其他更多线程调用 zrangebyscore 获取到了一条消息 A

2.T1 准备开始删除消息 A, 由于是原子操作, T2 和其他更多线程等待 T1 执行 zrem 删除消息 A 后再执行 zrem 删除消息 A

3.T1 删除了消息 A, 返回删除成功标记 1, 并对消息 A 进行处理

4.T2 其他更多线程开始 zrem 删除消息 A, 由于消息 A 已经被删除, 所以所有的删除均失败, 放弃了对消息 A 的处理

同时，我们要注意一定要对 handle_msg 进行异常捕获，避免因为个别任务处理问题导致循环异常退 出

**进一步优化**

上面的算法中同一个任务可能会被多个进程取到之后再使用 zrem 进行争抢，那些没抢到 的进程都是白取了一次任务，这是浪费。可以考虑使用 lua scripting 来优化一下这个逻辑，将 zrangebyscore 和 zrem 一同挪到服务器端进行原子化操作，这样多个进程之间争抢任务时就不 会出现这种浪费了

**使用调用Lua脚本进一步优化**

Lua 脚本, 如果有超时的消息, 就删除, 并返回这条消息, 否则返回空字符串:

复制

```java
String luaScript = "local resultArray = redis.call('zrangebyscore', KEYS[1], 0, ARGV[1], 'limit' , 0, 1)\n" + 
        "if #resultArray > 0 then\n" + 
        "    if redis.call('zrem', KEYS[1], resultArray[1]) > 0 then\n" + 
        "        return resultArray[1]\n" + 
        "    else\n" + 
        "        return ''\n" + 
        "    end\n" + 
        "else\n" + 
        "    return ''\n" + 
        "end"; 
 
jedis.eval(luaScript, ScriptOutputType.VALUE, new String[]{key}, String.valueOf(System.currentTimeMillis())); 
```

**Redis延时队列优势**

**Redis用来进行实现延时队列是具有这些优势的：**

1.Redis zset支持高性能的 score 排序。

2.Redis是在内存上进行操作的，速度非常快。

3.Redis可以搭建集群，当消息很多时候，我们可以用集群来提高消息处理的速度，提高可用性。

4.Redis具有持久化机制，当出现故障的时候，可以通过AOF和RDB方式来对数据进行恢复，保证了数据的可靠性

**Redis延时队列劣势**

使用 Redis 实现的延时消息队列也存在数据持久化, 消息可靠性的问题

没有重试机制 - 处理消息出现异常没有重试机制, 这些需要自己去实现, 包括重试次数的实现等

没有 ACK 机制 - 例如在获取消息并已经删除了消息情况下, 正在处理消息的时候客户端崩溃了, 这条正在处理的这些消息就会丢失, MQ 是需要明确的返回一个值给 MQ 才会认为这个消息是被正确的消费了

如果对消息可靠性要求较高, 推荐使用 MQ 来实现

**Redission实现延时队列**

基于Redis的Redisson分布式延迟队列结构的RDelayedQueue Java对象在实现了RQueue接口的基础上提供了向队列按要求延迟添加项目的功能。该功能可以用来实现消息传送延迟按几何增长或几何衰减的发送策略

[![](https://s3.51cto.com/oss/202009/29/002f3f883780344589fcf4cb64d0d547.png)](https://s3.51cto.com/oss/202009/29/002f3f883780344589fcf4cb64d0d547.png)

复制

```
RQueue<String> distinationQueue = ... 
RDelayedQueue<String> delayedQueue = getDelayedQueue(distinationQueue); 
// 10秒钟以后将消息发送到指定队列 
delayedQueue.offer("msg1", 10, TimeUnit.SECONDS); 
// 一分钟以后将消息发送到指定队列 
delayedQueue.offer("msg2", 1, TimeUnit.MINUTES); 
```

在该对象不再需要的情况下，应该主动销毁。仅在相关的Redisson对象也需要关闭的时候可以不用主动销毁。

复制

```
RDelayedQueue<String> delayedQueue = ... 
delayedQueue.destroy(); 
```
