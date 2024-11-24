## 面试题 & 真实经历

> **面试题**：在数据量很大的情况下，怎么实现深度分页？

大家在面试时，或者准备面试中可能会遇到上述的问题，大多的回答基本上是`分库分表建索引`，这是一种很`标准的正确回答`，但现实总是很骨感，所以面试官一般会追问你一句，现在工期不足，人员不足，该怎么实现深度分页？

这个时候没有实际经验的同学基本麻爪，So，请听我娓娓道来。

## 惨痛的教训

**首先必须明确一点**：深度分页可以做，但是深度随机跳页绝对需要禁止。

上一张图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/15/1734e4083013095f~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

你们猜，我点一下第`142360`页，服务会不会爆炸？

像`MySQL`，`MongoDB`数据库还好，本身就是专业的数据库，处理的不好，最多就是慢，但如果涉及到`ES`，性质就不一样了，我们不得不利用 `SearchAfter` Api，去循环获取数据，这就牵扯到内存占用的问题，如果当时代码写的不优雅，直接就可能导致内存溢出。

## 为什么不能允许随机深度跳页

从技术的角度浅显的聊一聊为什么不能允许随机深度跳页，或者说为什么不建议深度分页

### MySQL

分页的基本原理：

`SELECT * FROM test ORDER BY id DESC LIMIT 10000, 20; 复制代码`

LIMIT 10000 , 20的意思扫描满足条件的10020行，扔掉前面的10000行，返回最后的20行。如果是LIMIT 1000000 , 100，需要扫描1000100 行，在一个高并发的应用里，每次查询需要扫描超过100W行，不炸才怪。

### MongoDB

分页的基本原理：

`db.t_data.find().limit(5).skip(5); 复制代码`

同样的，随着页码的增大，skip 跳过的条目也会随之变大，而这个操作是通过 cursor 的迭代器来实现的，对于cpu的消耗会非常明显，当页码非常大时且频繁时，必然爆炸。

### ElasticSearch

从业务的角度来说，`ElasticSearch`不是典型的数据库，它是一个搜索引擎，如果在筛选条件下没有搜索出想要的数据，继续深度分页也不会找到想要的数据，退一步讲，假如我们把`ES`作为数据库来使用进行查询，在进行分页的时候一定会遇到`max_result_window`的限制，看到没，官方都告诉你最大偏移量限制是一万。

查询流程：

1.  如查询第501页，每页10条，客户端发送请求到某节点
2.  此节点将数据广播到各个分片，各分片各自查询前 5010 条数据
3.  查询结果返回至该节点，然后对数据进行整合，取出前 5010 条数据
4.  返回给客户端

由此可以看出为什么要限制偏移量，另外，如果使用 `Search After` 这种滚动式API进行深度跳页查询，也是一样需要每次滚动几千条，可能一共需要滚动上百万，千万条数据，就为了最后的20条数据，效率可想而知。

## 再次和产品对线

俗话说的好，技术解决不了的问题，就由业务来解决！

在实习的时候信了产品的邪，必须实现深度分页 + 跳页，如今必须`拨乱反正`，业务上必须有如下更改：

-   尽可能的增加默认的筛选条件，如：时间周期，目的是为了减少数据量的展示
-   修改跳页的展现方式，改为滚动显示，或小范围跳页

滚动显示参考图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/15/1734e40c5b213f17~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

小规模跳页参考图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/15/1734e40ecc634570~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

## 通用解决方案

短时间内快速解决的方案主要是以下几点：

-   必备：对排序字段，筛选条件务必设置好索引
-   核心：利用小范围页码的已知数据，或者滚动加载的已知数据，减少偏移量
-   额外：如果遇到不好处理的情况，也可以获取多余的数据，进行一定的截取，性能影响并不大

### MySQL

原分页SQL：

``# 第一页 SELECT * FROM `year_score` where `year` = 2017 ORDER BY id limit 0, 20;  # 第N页 SELECT * FROM `year_score` where `year` = 2017 ORDER BY id limit (N - 1) * 20, 20;  复制代码``

通过上下文关系，改写为：

``# XXXX 代表已知的数据 SELECT * FROM `year_score` where `year` = 2017 and id > XXXX ORDER BY id limit 20; 复制代码``

在 [没内鬼，来点干货！SQL优化和诊断](https://juejin.cn/post/6844904135964229646#heading-9 "https://juejin.cn/post/6844904135964229646#heading-9") 一文中提到过，LIMIT会在满足条件下停止查询，因此该方案的扫描总量会急剧减少，效率提升Max！

### ES

方案和`MySQL`相同，此时我们就可以随用所欲的使用 `FROM-TO` Api，而且不用考虑最大限制的问题。

### MongoDB

方案基本类似，基本代码如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/15/1734e4119ea67105~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

相关性能测试：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/15/1734e4138e0146b1~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

## 如果非要深度随机跳页

如果你没有杠过产品经理，又该怎么办呢，没关系，还有一丝丝的机会。

在 [SQL优化](https://juejin.cn/post/6844904135964229646#heading-8 "https://juejin.cn/post/6844904135964229646#heading-8") 一文中还提到过`MySQL`深度分页的处理技巧，代码如下：

`# 反例（耗时129.570s） select * from task_result LIMIT 20000000, 10;  # 正例（耗时5.114s） SELECT a.* FROM task_result a, (select id from task_result LIMIT 20000000, 10) b where a.id = b.id;  # 说明 # task_result表为生产环境的一个表，总数据量为3400万，id为主键，偏移量达到2000万 复制代码`

该方案的核心逻辑即基于`聚簇索引`，在不通过`回表`的情况下，快速拿到指定偏移量数据的主键ID，然后利用`聚簇索引`进行回表查询，此时总量仅为10条，效率很高。

因此我们在处理`MySQL`，`ES`，`MongoDB`时，也可以采用一样的办法：

1.  限制获取的字段，只通过筛选条件，深度分页获取主键ID
2.  通过主键ID定向查询需要的数据

瑕疵：当偏移量非常大时，耗时较长，如文中的 5s

  
# Reference
https://juejin.cn/post/6850037275456339975  