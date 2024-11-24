
> 导语：hashicorp/raft是raft算法的一种比较流行的golang实现，基于它能够比较方便的构建具有强一致性的分布式系统。本文通过实现一个简单的分布式缓存系统来介绍使用hashicorp/raft来构建分布式应用程序的方法。

# 1. 背景

对于后台开发来说，随着业务的发展，由于访问量增大的压力和数据容灾的需要，一定会需要使用分布式的系统，而分布式势必会引入一致性的问题。

一般把一致性分为三种类型：弱一致性、最终一致性、强一致性。这三种模型的一致性强度逐渐递增，实现代价也越来越大。通常弱一致性和最终一致性可以异步冗余，强一致性则是同步冗余，而同步也就意味着影响性能。

对常见的互联网业务来说，使用弱一致性或者最终一致性即可。而使用强一致性一方面会影响系统的性能，另一方面实现也比较困难。常见的一致性协议如zab、raft、paxos，如果由业务纯自己来实现的话代价较大，而且很可能会因为考虑不周而引入其他问题。

对于一些需要强一致性，而又希望花费较小代价的业务来说，使用开源的一致性协议实现组件会是个不错的选择。hashicorp/raft是raft协议的一种golang实现，由hashicorp公司实现并开源，已经在consul等软件中使用。它封装了raft协议的leader选举、log同步等底层实现，基于它能够相对比较容易的构建强一致性的分布式系统，下面以实现一个简单的分布式缓存服务(取名叫stcache)来演示hashicorp/raft的具体使用，完整代码可以在[github](https://github.com/KunTjz/stcache)上下载。

# 2. raft简介

首先还是简单介绍下raft协议。这里不详细介绍raft协议，只是为了方便理解后面的hashicorp/raft的使用步骤而简单列举出raft的一点原理。具体的raft协议可以参考[raft的官网](https://raft.github.io/)，如果已经了解raft协议可以直接跳过这一节。

raft是一种相对易于理解的一致性的协议。它属于leader-follower型的协议，有且只有一个leader，所有的事务请求都由leader处理，leader征求follower的意见，在集群内部达成一致，决定是否执行事务。当leader出现故障，集群中的follower会通过投票的方式选出一个新的leader，维持集群运行。

raft的理论基础是Replicated State Machine，Replicated State Machine需要满足如下的条件：一个server可以有多个state，多个server从同一个start状态出发，都执行相同的command序列，最终到达的stare是一样的。如上图，一般使用replicated log来记录command序列，client的请求被leader转化成log entry，然后通过一致性模块把log同步到各个server，让各个server的log一致。每个server都有state Machine，从start出发，执行完这些log中的command后，server处于相同的state。所以raft协议的关键就是保证各个server的log一致，然后每个server通过执行相同的log来达到一致的状态，理解这点有助于掌握后面对hashicorp/raft的具体使用。

![](https://ask.qcloudimg.com/draft/996040/99cjpf9obi.png)

# 3. hashicorp/raft使用

## 3.1 单机版

首先我们创建一个单机版本的stcache，它是一个简单的缓存[服务器](https://cloud.tencent.com/product/cvm?from=20065&from_column=20065)，在服务内部用一个map来保存数据，只提供简单的get和set操作。

```go
type cacheManager struct {
        data map[string]string
        sync.RWMutex
}
```



然后stcache开启一个http服务，提供两个api，第一个是set接口，用于设置数据到缓存，成功时返回ok，失败返回错误信息：

![](https://ask.qcloudimg.com/draft/996040/ourn8lbmit.png)

第二个是get接口，根据key查询具体的value：

![](https://ask.qcloudimg.com/draft/996040/ikqo4fqomz.png)

下面我们在单机版stcache的基础上逐步扩充，让它成为一个具有强一致性的分布式系统。

## 3.2 创建节点

```go
// NewRaft is used to construct a new Raft node. It takes a configuration, as well
// as implementations of various interfaces that are required. If we have any
// old state, such as snapshots, logs, peers, etc, all those will be restored
// when creating the Raft node.
func NewRaft(conf *Config,
	fsm FSM,
	logs LogStore,
	stable StableStore,
	snaps SnapshotStore,
	trans Transport) (*Raft, error) {
```

hashicorp/raft库提供NewRaft方法来创建一个raft节点，这也是使用这个库的最重要的一个api。NewRaft需要调用层提供6个参数，分别是：

-   Config： 节点配置
-   FSM： finite state machine，有限状态机
-   LogStore： 用来存储raft的日志
-   StableStore： 稳定存储，用来存储raft集群的节点信息等
-   SnapshotStore: 快照存储，用来存储节点的快照信息
-   Transport： raft节点内部的通信通道

下面从这些参数入手看应用程序需要做哪些工作。

## 3.3 Config

config是节点的配置信息，我们直接使用raft默认的配置，然后用监听的地址来作为节点的id。config里面还有一些可配置的项，后面我们用到的时候再说。

```go
    raftConfig := raft.DefaultConfig()
    raftConfig.LocalID = raft.ServerID(opts.raftTCPAddress)
    raftConfig.Logger = log.New(os.Stderr, "raft: ", log.Ldate|log.Ltime)
```



## 3.4 LogStore 和 StableStore

LogStore、StableStore分别用来存储raft log、节点状态信息，hashicorp提供了一个raft-boltdb来实现底层存储，它是一个嵌入式的[数据库](https://cloud.tencent.com/solution/database?from=20065&from_column=20065)，能够持久化存储数据，我们直接用它来实现LogStore和StableStore.

```go
   logStore, err := raftboltdb.NewBoltStore(filepath.Join(opts.dataDir, 
                                            "raft-log.bolt"))                            
   stableStore, err := raftboltdb.NewBoltStore(filepath.Join(opts.dataDir, 
                                               "raft-stable.bolt"))                      
```



## 3.5 SnapshotStore

SnapshotStore用来存储快照信息，对于stcache来说，就是存储当前的所有的kv数据，hashicorp内部提供3中快照存储方式，分别是：

-   DiscardSnapshotStore： 不存储，忽略快照，相当于/dev/null，一般用于测试
-   FileSnapshotStore： 文件持久化存储
-   InmemSnapshotStore： 内存存储，不持久化，重启程序会丢失

这里我们使用文件持久化存储。snapshotStore只是提供了一个快照存储的介质，还需要应用程序提供快照生成的方式，后面我们再具体说。

```go
    snapshotStore, err := raft.NewFileSnapshotStore(opts.dataDir, 1, os.Stderr)
```

## 3.6 Transport

Transport是raft集群内部节点之间的通信渠道，节点之间需要通过这个通道来进行日志同步、leader选举等。hashicorp/raft内部提供了两种方式来实现，一种是通过TCPTransport，基于tcp，可以跨机器跨网络通信；另一种是InmemTransport，不走网络，在内存里面通过channel来通信。显然一般情况下都使用TCPTransport即可，在stcache里也采用tcp的方式。

```go
func newRaftTransport(opts *options) (*raft.NetworkTransport, error) {
    address, err := net.ResolveTCPAddr("tcp", opts.raftTCPAddress)
    if err != nil {
        return nil, err
    }
    transport, err := raft.NewTCPTransport(address.String(), address, 3, 10*time.Second, os.Stderr)
    if err != nil {
        return nil, err
    }
    return transport, nil
}
```

## 3.7 FSM

最后再看FSM，它是一个interface，需要应用程序来实现3个funcition。

```go
/*FSM provides an interface that can be implemented by
clients to make use of the replicated log.*/
type FSM interface {
	/* Apply log is invoked once a log entry is committed.
	It returns a value which will be made available in the
	ApplyFuture returned by Raft.Apply method if that
	method was called on the same Raft node as the FSM.*/
	Apply(*Log) interface{}
	// Snapshot is used to support log compaction. This call should
	// return an FSMSnapshot which can be used to save a point-in-time
	// snapshot of the FSM. Apply and Snapshot are not called in multiple
	// threads, but Apply will be called concurrently with Persist. This means
	// the FSM should be implemented in a fashion that allows for concurrent
	// updates while a snapshot is happening.
	Snapshot() (FSMSnapshot, error)
	// Restore is used to restore an FSM from a snapshot. It is not called
	// concurrently with any other command. The FSM must discard all previous
	// state.
	Restore(io.ReadCloser) error
}
```



第一个是Apply，当raft内部commit了一个log entry后，会记录在上面说过的logStore里面，被commit的log entry需要被执行，就stcache来说，执行log entry就是把数据写入缓存，即执行set操作。我们改造doSet方法， 这里不再直接写缓存，而是调用raft的Apply方式，为这次set操作生成一个log entry，这里面会根据raft的内部协议，在各个节点之间进行通信协作，确保最后这条log 会在整个集群的节点里面提交或者失败。

```go
// doSet saves data to cache, only raft master node provides this api
func (h *httpServer) doSet(w http.ResponseWriter, r *http.Request) {
    // ... get params from request url

    event := logEntryData{Key: key, Value: value}
    eventBytes, err := json.Marshal(event)
    if err != nil {
        h.log.Printf("json.Marshal failed, err:%v", err)
        fmt.Fprint(w, "internal error\n")
        return
    }

    applyFuture := h.ctx.st.raft.raft.Apply(eventBytes, 5*time.Second)
    if err := applyFuture.Error(); err != nil {
        h.log.Printf("raft.Apply failed:%v", err)
        fmt.Fprint(w, "internal error\n")
        return
    }

    fmt.Fprintf(w, "ok\n")
}
```



对follower节点来说，leader会通知它来commit log entry，被commit的log entry需要调用应用层提供的Apply方法来执行日志，这里就是从logEntry拿到具体的数据，然后写入缓存里面即可。

```go
// Apply applies a Raft log entry to the key-value store.
func (f *FSM) Apply(logEntry *raft.Log) interface{} {
        e := logEntryData{}
        if err := json.Unmarshal(logEntry.Data, &e); err != nil {
                panic("Failed unmarshaling Raft log entry.")
        }
        ret := f.ctx.st.cm.Set(e.Key, e.Value)
        return ret
} 
```
### 3.7.1 snapshot

FSM需要提供的另外两个方法是Snapshot()和Restore()，分别用于生成一个快照结构和根据快照恢复数据。首先我们需要定义快照，hashicorp/raft内部定义了快照的interface，需要实现两个func，Persist用来生成快照数据，一般只需要实现它即可；Release则是快照处理完成后的回调，不需要的话可以实现为空函数。

```go
// FSMSnapshot is returned by an FSM in response to a Snapshot
// It must be safe to invoke FSMSnapshot methods with concurrent
// calls to Apply.
type FSMSnapshot interface {
	// Persist should dump all necessary state to the WriteCloser 'sink',
	// and call sink.Close() when finished or call sink.Cancel() on error.
	Persist(sink SnapshotSink) error
	// Release is invoked when we are finished with the snapshot.
	Release()
}
```
我们定义一个简单的snapshot结构，在Persist里面，自己把缓存里面的数据用json格式化的方式来生成快照，sink.Write就是把快照写入snapStore，我们刚才定义的是FileSnapshotStore，所以会把数据写入文件。

```go
type snapshot struct {
        cm *cacheManager
}
// Persist saves the FSM snapshot out to the given sink.
func (s *snapshot) Persist(sink raft.SnapshotSink) error {
        snapshotBytes, err := s.cm.Marshal()
        if err != nil {
                sink.Cancel()
                return err
        }
        if _, err := sink.Write(snapshotBytes); err != nil {
                sink.Cancel()
                return err
        }
        if err := sink.Close(); err != nil {
                sink.Cancel()
                return err
        }
        return nil
}
func (f *snapshot) Release() {}
```
### 3.7.2 snapshot保存与恢复

而快照生成和保存的触发条件除了应用程序主动触发外，还可以在Config里面设置SnapshotInterval和SnapshotThreshold，前者指每间隔多久生成一次快照，后者指每commit多少log entry后生成一次快照。需要两个条件同时满足才会生成和保存一次快照，默认config里面配置的条件比较高，我们可以自己修改配置，比如在stcache里面配置SnapshotInterval为20s，SnapshotThreshold为2，表示当满足距离上次快照保存超过20s，且log增加2条的时候，保存一个新的快照。

```go
    raftConfig := raft.DefaultConfig()
    raftConfig.LocalID = raft.ServerID(opts.raftTCPAddress)
    raftConfig.Logger = log.New(os.Stderr, "raft: ", log.Ldate|log.Ltime)
    raftConfig.SnapshotInterval = 20 * time.Second
    raftConfig.SnapshotThreshold = 2
```
服务重启的时候，会先读取本地的快照来恢复数据，在FSM里面定义的Restore函数会被调用，这里我们就简单的对数据解析json反序列化然后写入内存即可。至此，我们已经能够正常的保存快照，也能在重启的时候从文件恢复快照数据。

```go
// Restore stores the key-value store to a previous state.
func (f *FSM) Restore(serialized io.ReadCloser) error {
        return f.ctx.st.cm.UnMarshal(serialized)
}

// UnMarshal deserializes cache data
func (c *cacheManager) UnMarshal(serialized io.ReadCloser) error {
        var newData map[string]string
        if err := json.NewDecoder(serialized).Decode(&newData); err != nil {
                return err
        }
        c.Lock()
        defer c.Unlock()
        c.data = newData
        return nil
}
```
## 3.8 集群建立

集群最开始的时候只有一个节点，我们让第一个节点通过bootstrap的方式启动，它启动后成为leader。

```go
    if opts.bootstrap {
        configuration := raft.Configuration{
            Servers: []raft.Server{
                {
                    ID:      raftConfig.LocalID,
                    Address: transport.LocalAddr(),
                },
            },
        }
        raftNode.BootstrapCluster(configuration)
    }
```
后续的节点启动的时候需要加入集群，启动的时候指定第一个节点的地址，并发送请求加入集群，这里我们定义成直接通过http请求。

```go
// joinRaftCluster joins a node to raft cluster
func joinRaftCluster(opts *options) error {
    url := fmt.Sprintf("http://%s/join?peerAddress=%s", 
                       opts.joinAddress, 
                       opts.raftTCPAddress)
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return err
    }
    if string(body) != "ok" {
        return errors.New(fmt.Sprintf("Error joining cluster: %s", body))
    }
    return nil
}
```
先启动的节点收到请求后，获取对方的地址（指raft集群内部通信的tcp地址），然后调用AddVoter把这个节点加入到集群即可。申请加入的节点会进入follower状态，这以后集群节点之间就可以正常通信，leader也会把数据同步给follower。

```go
// doJoin handles joining cluster request
func (h *httpServer) doJoin(w http.ResponseWriter, r *http.Request) {
    vars := r.URL.Query()

    peerAddress := vars.Get("peerAddress")
    if peerAddress == "" {
        h.log.Println("invalid PeerAddress")
        fmt.Fprint(w, "invalid peerAddress\n")
        return
    }
    addPeerFuture := h.ctx.st.raft.raft.AddVoter(raft.ServerID(peerAddress), 
                                                 raft.ServerAddress(peerAddress), 
                                                 0, 0)
    if err := addPeerFuture.Error(); err != nil {
        h.log.Printf("Error joining peer to raft, peeraddress:%s, err:%v, code:%d", peerAddress, err, http.StatusInternalServerError)
        fmt.Fprint(w, "internal error\n")
        return
    }
    fmt.Fprint(w, "ok")
}
```
## 3.9 故障切换

当集群的leader故障后，集群的其他节点能够感知到，并申请成为leader，在各个follower中进行投票，最后选取出一个新的leader。leader选举是属于raft协议的内容，不需要应用程序操心，但是对有些场景而言，应用程序需要感知leader状态，比如对stcache而言，理论上只有leader才能处理set请求来写数据，follower应该只能处理get请求查询数据。为了模拟说明这个情况，我们在stcache里面我们设置一个写标志位，当本节点是leader的时候标识位置true，可以处理set请求，否则标识位为false，不能处理set请求。

```go
// doSet saves data to cache, only raft master node provides this api
func (h *httpServer) doSet(w http.ResponseWriter, r *http.Request) {
        if !h.checkWritePermission() {
                fmt.Fprint(w, "write method not allowed\n")
                return
        }
        // ... set data  
}
```



当故障切换的时候，follower变成了leader，应用程序如何感知到呢？ 在raft结构里面提供有一个eaderCh，它是bool类型的channel，不带缓存，当本节点的leader状态有变化的时候，会往这个channel里面写数据，但是由于不带缓冲且写数据的协程不会阻塞在这里，有可能会写入失败，没有及时变更状态，所以使用leaderCh的可靠性不能保证。好在raft Config里面提供了另一个channel NotifyCh，它是带缓存的，当leader状态变化时会往这个chan写数据，写入的变更消息能够缓存在channel里面，应用程序能够通过它获取到最新的状态变化。

我们首先在初始化config时候创建一个带缓存的chan，把它赋值给config里面的NotifyCh，然后在节点启动后监听这个chan，当本节点的leader状态变化时（变成leader或者从leader变成follower），就能够从这个chan里面读取到bool值，并调整我们先前设置的写标志位，控制是否能否处理set操作。

```go
func newRaftNode(opts *options, ctx *stCachedContext) (*raftNodeInfo, error) {
    raftConfig := raft.DefaultConfig()
    raftConfig.LocalID = raft.ServerID(opts.raftTCPAddress)
    raftConfig.Logger = log.New(os.Stderr, "raft: ", log.Ldate|log.Ltime)
    raftConfig.SnapshotInterval = 20 * time.Second
    raftConfig.SnapshotThreshold = 2
    leaderNotifyCh := make(chan bool, 1)
    raftConfig.NotifyCh = leaderNotifyCh
    // ... 
}
```
# 4. 成果演示

做完上面的工作后，我们来测试下效果，我们同一台机器上启动3个节点来构成一个集群，第一个节点用bootstrapt的方式启动，成为leader

![](https://ask.qcloudimg.com/draft/996040/c8wbc72vdy.png)

第二个节点和第三个节点启动时指定加入集群，成为follower

![](https://ask.qcloudimg.com/draft/996040/dz4tjh7air.png)

![](https://ask.qcloudimg.com/draft/996040/bxaweywn5g.png)

现在集群中有3个节点，leader监听127.0.01:6000对外提供set和get接口，两个follower分别监听127.0.0.1:6001和127.0.0.1:6002，对外提供get接口。

## 4.1 集群数据同步

通过调用leader的set接口写入一个数据，key是ping，value是pong

![](https://ask.qcloudimg.com/draft/996040/28maveg8ym.png)

这时候能在两个follower上看见apply的日志，follower节点写入了log，并收到leader的通知提交数据。

![](https://ask.qcloudimg.com/draft/996040/1c0qkctq9t.png)

通过查询接口，也能从follower里面查询到刚才写入的数据，证明数据同步没有问题。

![](https://ask.qcloudimg.com/draft/996040/pks5r2yg5u.png)

有一点需要说明的事，我们这里从follower是可能读不到最新数据的。由于leader对set操作返回的时候，follower可能还没有apply数据，所以从follower的get查询可能返回旧数据或者空数据。如果要保证能从follower查询到的一定是最新的数据还需要很多额外的工作，即做到linearizable read，有兴趣可以看这篇[测试文章](https://aphyr.com/posts/316-call-me-maybe-etcd-and-consul)，这里不再展开。

## 4.2 快照保存与恢复

我们再通过set接口写入两个数据，能看见节点开始保存快照

![](https://ask.qcloudimg.com/draft/996040/5dtk97iu5x.png)

在指定的目录下面，能看见快照的具体信息，有两个文件，meta.json保存了版本号、log序号、集群节点地址等集群信息；state.bin里面是快照数据，这里就是我们刚刚写入的数据被json序列化后的字符串。

![](https://ask.qcloudimg.com/draft/996040/ohp600lfas.png)

现在把节点都停止，然后重新启动leader，内存的数据都丢失，它会从保存的快照文件里面恢复数据。重启follower也一样会从自己保存的快照里面加载数据。

![](https://ask.qcloudimg.com/draft/996040/ypci7nehf1.png)

## 4.3 leader切换

把leader和follower都重启恢复，现在leader监听127.0.01:6000，只有它能执行set操作，follower只能执行get操作

![](https://ask.qcloudimg.com/draft/996040/jk0qcr8e12.png)

我们停掉leader节点，两个follower会开始选举，这里node2赢得了选举，进入leader状态，并且它开始打开set操作

![](https://ask.qcloudimg.com/draft/996040/xwbpepsj31.png)

我们再请求node2监听的127.0.0.1:6001，发现已经可以正常写入数据了，leader切换顺利完成。

![](https://ask.qcloudimg.com/draft/996040/qq0pjr9167.png)

我们再重启原来的leader节点，它会变成follower，并从新的leader(也就是node2)这里同步它所缺失的数据。

# 5. 总结

上面所创建的stcache只是一个简单的示例程序，真正要做到在线上使用还有很多问题需要考虑，目前基于hashicorp/raft比较成熟的开源软件有[consul](https://www.consul.io/)，如果有兴趣可以通过它做进一步研究。

总的来说，hashicorp/raft封装了raft的内部协议，提供简洁明了的使用方法，基于它能够很快速地构建出具有强一致性的应用程序。



# Reference
https://cloud.tencent.com/developer/beta/article/1183490