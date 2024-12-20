## 方式一：RDB快照

**RDB持久化功能所生成的RDB文件是一个经过压缩的二进制文件，通过该文件可以还原生成RDB文件时的数据库状态**

因为RDB文件是保存在硬盘里面的，所以即使Redis服务器进程退出，甚至运行Redis服务器的计算机停机，但只要RDB文件仍然存在，Redis服务器就可以用它来还原数据库状态。本章首先介绍Redis服务器保存和载入RDB文件的方法，重点说明SAVE命令和BGSAVE命令的实现方式。

和SAVE命令直接阻塞服务器进程的做法不同，BGSAVE命令会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求

有两个Redis命令可以用于生成RDB文件，一个是SAVE，另一个是BGSAVE。SAVE命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求

和使用SAVE命令或者BGSAVE命令创建RDB文件不同，RDB文件的载入工作是在服务器启动时自动执行的，所以Redis并没有专门用于载入RDB文件的命令，只要Redis服务器在启动时检测到RDB文件存在，它就会自动载入RDB文件。但是服务器在载入RDB文件期间，会一直处于阻塞状态，直到载入工作完成为止。

另外值得一提的是，因为AOF文件的更新频率通常比RDB文件的更新频率高，所以：

❑如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态。

❑只有在AOF持久化功能处于关闭状态时，服务器才会使用RDB文件来还原数据库状态。

**Redis 快照** 是最简单的 Redis 持久性模式。当满足特定条件时，它将生成数据集的时间点快照，例如，如果先前的快照是在2分钟前创建的，并且现在已经至少有 _100_ 次新写入，则将创建一个新的快照。此条件可以由用户配置 Redis 实例来控制，也可以在运行时修改而无需重新启动服务器。快照作为包含整个数据集的单个 .rdb 文件生成。

但我们知道，Redis 是一个 **单线程** 的程序，这意味着，我们不仅仅要响应用户的请求，还需要进行内存快照。而后者要求 Redis 必须进行 IO 操作，这会严重拖累服务器的性能。

还有一个重要的问题是，我们在 **持久化的同时**，**内存数据结构** 还可能在 **变化**，比如一个大型的 hash 字典正在持久化，结果一个请求过来把它删除了，可是这才刚持久化结束，咋办？

### [使用系统多进程 COW(Copy On Write) 机制 | fork 函数](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-collection/Redis(7)%E2%80%94%E2%80%94%E6%8C%81%E4%B9%85%E5%8C%96?id=%e4%bd%bf%e7%94%a8%e7%b3%bb%e7%bb%9f%e5%a4%9a%e8%bf%9b%e7%a8%8b-cowcopy-on-write-%e6%9c%ba%e5%88%b6-fork-%e5%87%bd%e6%95%b0)

操作系统多进程 **COW(Copy On Write) 机制** 拯救了我们。**Redis** 在持久化时会调用 glibc 的函数 fork 产生一个子进程，简单理解也就是基于当前进程 **复制** 了一个进程，主进程和子进程会共享内存里面的代码块和数据段：
![[Pasted image 20230516164322.png]]

这里多说一点，**为什么 fork 成功调用后会有两个返回值呢？** 因为子进程在复制时复制了父进程的堆栈段，所以两个进程都停留在了 fork 函数中 \_(都在同一个地方往下继续"同时"执行)__，等待返回，所以 *__一次在父进程中返回子进程的 pid，另一次在子进程中返回零，系统资源不够时返回负数**。 *(伪代码如下)_

![[Pasted image 20230516164347.png]]

所以 **快照持久化** 可以完全交给 **子进程** 来处理，**父进程** 则继续 **处理客户端请求**。**子进程** 做数据持久化，它 **不会修改现有的内存数据结构**，它只是对数据结构进行遍历读取，然后序列化写到磁盘中。但是 **父进程** 不一样，它必须持续服务客户端请求，然后对 **内存数据结构进行不间断的修改**。

这个时候就会使用操作系统的 COW 机制来进行 **数据段页面** 的分离。数据段是由很多操作系统的页面组合而成，当父进程对其中一个页面的数据进行修改时，会将被共享的页面复 制一份分离出来，然后 **对这个复制的页面进行修改**。这时 **子进程** 相应的页面是 **没有变化的**，还是进程产生时那一瞬间的数据。

子进程因为数据没有变化，它能看到的内存里的数据在进程产生的一瞬间就凝固了，再也不会改变，这也是为什么 **Redis** 的持久化 **叫「快照」的原因**。接下来子进程就可以非常安心的遍历数据了进行序列化写磁盘了。

但是，在BGSAVE命令执行期间，服务器处理SAVE、BGSAVE、BGREWRITEAOF三个命令的方式会和平时有所不同。

首先，在BGSAVE命令执行期间，客户端发送的SAVE命令会被服务器拒绝，服务器禁止SAVE命令和BGSAVE命令同时执行是为了避免父进程（服务器进程）和子进程同时执行两个rdbSave调用，防止产生竞争条件。

其次，在BGSAVE命令执行期间，客户端发送的BGSAVE命令会被服务器拒绝，因为同时执行两个BGSAVE命令也会产生竞争条件。

最后，BGREWRITEAOF和BGSAVE两个命令不能同时执行：

❑如果BGSAVE命令正在执行，那么客户端发送的BGREWRITEAOF命令会被延迟到BGSAVE命令执行完毕之后执行。

❑如果BGREWRITEAOF命令正在执行，那么客户端发送的BGSAVE命令会被服务器拒绝。

因为BGREWRITEAOF和BGSAVE两个命令的实际工作都由子进程执行，所以这两个命令在操作方面并没有什么冲突的地方，不能同时执行它们只是一个性能方面的考虑——并发出两个子进程，并且这两个子进程都同时执行大量的磁盘写入操作，这怎么想都不会是一个好主意。

### 设置RDB自动保存条件

因为BGSAVE命令可以在不阻塞服务器进程的情况下执行，所以Redis允许用户通过设置服务器配置的save选项，让服务器每隔一段时间自动执行一次BGSAVE命令。

当Redis服务器启动时，用户可以通过指定配置文件或者传入启动参数的方式设置save选项，如果用户没有主动设置save选项，那么服务器会为save选项设置默认条件：
![[Pasted image 20230516164559.png]]
![[Pasted image 20230516164547.png]]

## [方式二：AOF](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-collection/Redis(7)%E2%80%94%E2%80%94%E6%8C%81%E4%B9%85%E5%8C%96?id=%e6%96%b9%e5%bc%8f%e4%ba%8c%ef%bc%9aaof)

除了RDB持久化功能之外，Redis还提供了AOF（Append Only File）持久化功能。**与RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态的**

**RDB持久化保存数据库状态的方法是将键的键值对保存到RDB文件中，而AOF持久化保存数据库状态的方法则是将服务器执行的命令保存到AOF文件中。**

服务器在启动时，可以通过载入和执行AOF文件中保存的命令来还原服务器关闭之前的数据库状态。

Redis读取AOF文件并还原数据库状态的详细步骤如下：

1）创建一个不带网络连接的伪客户端（fakeclient）：因为Redis的命令只能在客户端上下文中执行，而载入AOF文件时所使用的命令直接来源于AOF文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行AOF文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样。

2）从AOF文件中分析并读取出一条写命令。

3）使用伪客户端执行被读出的写命令。

4）一直执行步骤2和步骤3，直到AOF文件中的所有写命令都被处理完毕为止。当完成以上步骤之后，AOF文件所保存的数据库状态就会被完整地还原出来

AOF持久化功能的实现可以分为命令追加（append）、文件写入、文件同步（sync）三个步骤。

当AOF持久化功能处于打开状态时，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾

**快照不是很持久**。如果运行 Redis 的计算机停止运行，电源线出现故障或者您 kill -9 的实例意外发生，则写入 Redis 的最新数据将丢失。尽管这对于某些应用程序可能不是什么大问题，但有些使用案例具有充分的耐用性，在这些情况下，快照并不是可行的选择。

**AOF(Append Only File - 仅追加文件)** 它的工作方式非常简单：每次执行 **修改内存** 中数据集的写操作时，都会 **记录** 该操作。假设 AOF 日志记录了自 Redis 实例创建以来 **所有的修改性指令序列**，那么就可以通过对一个空的 Redis 实例 **顺序执行所有的指令**，也就是 **「重放」**，来恢复 Redis 当前实例的内存数据结构的状态。

为了展示 AOF 在实际中的工作方式，我们来做一个简单的实验：

./redis-server --appendonly yes # 设置一个新实例为 AOF 模式

然后我们执行一些写操作：

```bash
redis 127.0.0.1:6379> set key1 Hello

OK

redis 127.0.0.1:6379> append key1 " World!"

(integer) 12

redis 127.0.0.1:6379> del key1

(integer) 1

redis 127.0.0.1:6379> del non_existing_key

(integer) 0
```

前三个操作实际上修改了数据集，第四个操作没有修改，因为没有指定名称的键。这是 AOF 日志保存的文本：

```bash
$ cat appendonly.aof
*2
$6
SELECT
$1
0
*3
$3
set
$4
key1
$5
Hello
*3
$6
append
$4
key1
$7
World!
*2
$3
del
$4
key1
```

如您所见，最后的那一条 DEL 指令不见了，因为它没有对数据集进行任何修改。

就是这么简单。当 Redis 收到客户端修改指令后，会先进行参数校验、逻辑处理，如果没问题，就 **立即** 将该指令文本 **存储** 到 AOF 日志中，也就是说，**先执行指令再将日志存盘**。这一点不同于 MySQL、LevelDB、HBase 等存储引擎，如果我们先存储日志再做逻辑处理，这样就可以保证即使宕机了，我们仍然可以通过之前保存的日志恢复到之前的数据状态，但是 **Redis 为什么没有这么做呢？**

**Emmm... 没找到特别满意的答案，引用一条来自知乎上的回答吧：**

-   **@缘于专注** - 我甚至觉得没有什么特别的原因。仅仅是因为，由于AOF文件会比较大，为了避免写入无效指令（错误指令），必须先做指令检查？如何检查，只能先执行了。因为语法级别检查并不能保证指令的有效性，比如删除一个不存在的key。而MySQL这种是因为它本身就维护了所有的表的信息，所以可以语法检查后过滤掉大部分无效指令直接记录日志，然后再执行。
-   更多讨论参见：[为什么Redis先执行指令，再记录AOF日志，而不是像其它存储引擎一样反过来呢？ - https://www.zhihu.com/question/342427472](https://www.zhihu.com/question/342427472)

### [AOF 重写](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-collection/Redis(7)%E2%80%94%E2%80%94%E6%8C%81%E4%B9%85%E5%8C%96?id=aof-%e9%87%8d%e5%86%99)

**Redis** 在长期运行的过程中，AOF 的日志会越变越长。如果实例宕机重启，重放整个 AOF 日志会非常耗时，导致长时间 Redis 无法对外提供服务。所以需要对 **AOF 日志 "瘦身"**。

**Redis** 提供了 bgrewriteaof 指令用于对 AOF 日志进行瘦身。其 **原理** 就是 **开辟一个子进程** 对内存进行 **遍历** 转换成一系列 Redis 的操作指令，**序列化到一个新的 AOF 日志文件** 中。序列化完毕后再将操作期间发生的 **增量 AOF 日志** 追加到这个新的 AOF 日志文件中，追加完毕后就立即替代旧的 AOF 日志文件了，瘦身工作就完成了。

这也就是说，在子进程执行AOF重写期间，服务器进程需要执行以下三个工作：

1）执行客户端发来的命令。

2）将执行后的写命令追加到AOF缓冲区。

3）将执行后的写命令追加到AOF重写缓冲区。

这样一来可以保证：

❑AOF缓冲区的内容会定期被写入和同步到AOF文件，对现有AOF文件的处理工作会如常进行。

❑从创建子进程开始，服务器执行的所有写命令都会被记录到AOF重写缓冲区里面。当子进程完成AOF重写工作之后，它会向父进程发送一个信号，父进程在接到该信号之后，会调用一个信号处理函数，并执行以下工作：

1）将AOF重写缓冲区中的所有内容写入到新AOF文件中，这时新AOF文件所保存的数据库状态将和服务器当前的数据库状态一致。

2）对新的AOF文件进行改名，原子地（atomic）覆盖现有的AOF文件，完成新旧两个AOF文件的替换。这个信号处理函数执行完毕之后，父进程就可以继续像往常一样接受命令请求了。**在整个AOF后台重写过程中，只有信号处理函数执行时会对服务器进程（父进程）造成阻塞，在其他时候，AOF后台重写都不会阻塞父进程，这将AOF重写对服务器性能造成的影响降到了最低。**

### [fsync](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-collection/Redis(7)%E2%80%94%E2%80%94%E6%8C%81%E4%B9%85%E5%8C%96?id=fsync)

AOF 日志是以文件的形式存在的，当程序对 AOF 日志文件进行写操作时，实际上是将内容写到了内核为文件描述符分配的一个内存缓存中，然后内核会异步将脏数据刷回到磁盘的。

就像我们 _上方第四步_ 描述的那样，我们需要借助 glibc 提供的 fsync(int fd) 函数来讲指定的文件内容 **强制从内核缓存刷到磁盘**。但 **"强制开车"** 仍然是一个很消耗资源的一个过程，需要 **"节制"**！通常来说，生产环境的服务器，Redis 每隔 1s 左右执行一次 fsync 操作就可以了。

Redis 同样也提供了另外两种策略，一个是 **永不** **fsync**，来让操作系统来决定合适同步磁盘，很不安全，另一个是 **来一个指令就** **fsync** **一次**，非常慢。但是在生产环境基本不会使用，了解一下即可。
![[Pasted image 20230516164925.png]]

如果用户没有主动为appendfsync选项设置值，那么appendfsync选项的默认值为everysec

## [Redis 4.0 混合持久化](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-collection/Redis(7)%E2%80%94%E2%80%94%E6%8C%81%E4%B9%85%E5%8C%96?id=redis-40-%e6%b7%b7%e5%90%88%e6%8c%81%e4%b9%85%e5%8c%96)

重启 Redis 时，我们很少使用 rdb 来恢复内存状态，因为会丢失大量数据。我们通常使用 AOF 日志重放，但是重放 AOF 日志性能相对 rdb来说要慢很多，这样在 Redis 实例很大的情况下，启动需要花费很长的时间。

**Redis 4.0** 为了解决这个问题，带来了一个新的持久化选项——**混合持久化**。将 rdb 文件的内容和增量的 AOF 日志文件存在一起。这里的 AOF 日志不再是全量的日志，而是 **自持久化开始到持久化结束** 的这段时间发生的增量 AOF 日志，通常这部分 AOF 日志很小：

![](https://cdn.nlark.com/yuque/0/2021/png/2725209/1632823866538-d1e05d49-353f-49eb-9752-0a36f4aea04e.png)

于是在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志就可以完全替代之前的 AOF全量文件重放，重启效率因此大幅得到提升。

到这里为止，其实还是停留在简单学习知识的程度，学会了redis的持久化的原理和操作，但是在企业中，持久化到底是怎么去用得呢？

企业级的数据备份和各种灾难下的数据恢复，是怎么做得呢？

1、企业级的持久化的配置策略

在企业中，RDB的生成策略，用默认的也差不多

save 60 10000：如果你希望尽可能确保说，RDB最多丢1分钟的数据，那么尽量就是每隔1分钟都生成一个快照，低峰期，数据量很少，也没必要

10000->生成RDB，1000->RDB，这个根据你自己的应用和业务的数据量，你自己去决定

AOF一定要打开，fsync，everysec

auto-aof-rewrite-percentage 100: 就是当前AOF大小膨胀到超过上次100%，上次的两倍

auto-aof-rewrite-min-size 64mb: 根据你的数据量来定，16mb，32mb

2、企业级的数据备份方案

RDB非常适合做冷备，每次生成之后，就不会再有修改了

数据备份方案

（1）写crontab定时调度脚本去做数据备份

（2）每小时都copy一份rdb的备份，到一个目录中去，仅仅保留最近48小时的备份

（3）每天都保留一份当日的rdb的备份，到一个目录中去，仅仅保留最近1个月的备份

（4）每次copy备份的时候，都把太旧的备份给删了

（5）每天晚上将当前服务器上所有的数据备份，发送一份到远程的云服务上去

/usr/local/redis

每小时copy一次备份，删除48小时前的数据

```bash
crontab -e

0 * * * * sh /usr/local/redis/copy/redis_rdb_copy_hourly.sh

redis_rdb_copy_hourly.sh

#!/bin/sh

cur_date=`date +%Y%m%d%k`

rm -rf /usr/local/redis/snapshotting/$cur_date

mkdir /usr/local/redis/snapshotting/$cur_date

cp /var/redis/6379/dump.rdb /usr/local/redis/snapshotting/$cur_date

del_date=`date -d -48hour +%Y%m%d%k`

rm -rf /usr/local/redis/snapshotting/$del_date
```

每天copy一次备份

```bash
crontab -e

0 0 * * * sh /usr/local/redis/copy/redis_rdb_copy_daily.sh

redis_rdb_copy_daily.sh

#!/bin/sh

cur_date=`date +%Y%m%d`

rm -rf /usr/local/redis/snapshotting/$cur_date

mkdir /usr/local/redis/snapshotting/$cur_date

cp /var/redis/6379/dump.rdb /usr/local/redis/snapshotting/$cur_date

del_date=`date -d -1month +%Y%m%d`

rm -rf /usr/local/redis/snapshotting/$del_date
```

每天一次将所有数据上传一次到远程的云服务器上去

3、数据恢复方案

（1）如果是redis进程挂掉，那么重启redis进程即可，直接基于AOF日志文件恢复数据

不演示了，在AOF数据恢复那一块，演示了，fsync everysec，最多就丢一秒的数

（2）如果是redis进程所在机器挂掉，那么重启机器后，尝试重启redis进程，尝试直接基于AOF日志文件进行数据恢复

AOF没有破损，也是可以直接基于AOF恢复的

AOF append-only，顺序写入，如果AOF文件破损，那么用redis-check-aof fix

（3）如果redis当前最新的AOF和RDB文件出现了丢失/损坏，那么可以尝试基于该机器上当前的某个最新的RDB数据副本进行数据恢复

当前最新的AOF和RDB文件都出现了丢失/损坏到无法恢复，一般不是机器的故障，人为

大数据系统，hadoop，有人不小心就把hadoop中存储的大量的数据文件对应的目录，rm -rf一下，我朋友的一个小公司，运维不太靠谱，权限也弄的不太好

/var/redis/6379下的文件给删除了

找到RDB最新的一份备份，小时级的备份可以了，小时级的肯定是最新的，copy到redis里面去，就可以恢复到某一个小时的数据

容灾演练

appendonly.aof + dump.rdb，优先用appendonly.aof去恢复数据，但是我们发现redis自动生成的appendonly.aof是没有数据的

然后我们自己的dump.rdb是有数据的，但是明显没用我们的数据

redis启动的时候，自动重新基于内存的数据，生成了一份最新的rdb快照，直接用空的数据，覆盖掉了我们有数据的，拷贝过去的那份dump.rdb

你停止redis之后，其实应该先删除appendonly.aof，然后将我们的dump.rdb拷贝过去，然后再重启redis

很简单，就是虽然你删除了appendonly.aof，但是因为打开了aof持久化，redis就一定会优先基于aof去恢复，即使文件不在，那就创建一个新的空的aof文件

停止redis，暂时在配置中关闭aof，然后拷贝一份rdb过来，再重启redis，数据能不能恢复过来，可以恢复过来

脑子一热，再关掉redis，手动修改配置文件，打开aof，再重启redis，数据又没了，空的aof文件，所有数据又没了

在数据安全丢失的情况下，基于rdb冷备，如何完美的恢复数据，同时还保持aof和rdb的双开

停止redis，关闭aof，拷贝rdb备份，重启redis，确认数据恢复，直接在命令行热修改redis配置，打开aof，这个redis就会将内存中的数据对应的日志，写入aof文件中

此时aof和rdb两份数据文件的数据就同步了

redis config set热修改配置参数，可能配置文件中的实际的参数没有被持久化的修改，再次停止redis，手动修改配置文件，打开aof的命令，再次重启redis

（4）如果当前机器上的所有RDB文件全部损坏，那么从远程的云服务上拉取最新的RDB快照回来恢复数据

（5）如果是发现有重大的数据错误，比如某个小时上线的程序一下子将数据全部污染了，数据全错了，那么可以选择某个更早的时间点，对数据进行恢复

举个例子，12点上线了代码，发现代码有bug，导致代码生成的所有的缓存数据，写入redis，全部错了

找到一份11点的rdb的冷备，然后按照上面的步骤，去恢复到11点的数据，不就可以了吗