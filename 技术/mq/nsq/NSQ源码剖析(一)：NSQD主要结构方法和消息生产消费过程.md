#nsq 


# 1 概述

NSQ包含3个组件:

-   nsqd：每个nsq实例运行一个nsqd进程，负责接收生产者消息、向nsqlookupd注册、向消费者推送消息
-   nsqlookupd：集群注册中心，可以有多个，负责接收nsqd的注册信息，向消费者提供服务发现
-   nsqadmin：用于监控和管理的web ui

生产者将消息写入到指定的主题Topic，同一个Topic下则可以关联多个管道Channel，每个Channel都会传输对应Topic的完整副本。  
消费者则订阅Channel的消息。于是多个消费者订阅不同的Channel的话，他们各自都能拿到完整的消息副本；但如果多个消费者订阅同一个Channel，则是共享的，即消息会随机发送给其中一个消费者。

接下来我们来分析下nsq的源码：

-   源码地址：[https://github.com/nsqio/nsq](https://github.com/nsqio/nsq)
-   2020年1月18日最新的master分支

nsq各组件均使用上述代码仓库，通过apps目录下的不同的main包启动。比如nsqd的main函数在apps/nsqd目录下，其他类同。

本文档主要分析nsqd的主要结构体和方法，及消息生产和消费的过程。主要以TCP api为例来分析，HTTP/HTTPS的api类同。

# 2 主要结构体及方法

## 2.1 NSQD

nsqd/nsqd.go文件，NSQD是主实例，一个nsqd进程创建一个nsqd结构体实例，并通过此结构体的Main()方法启动所有的服务。

```go
type NSQD struct {
	clientIDSequence int64 // 递增的客户端ID，每个客户端连接均从这里取一个递增后的ID作为唯一标识

	sync.RWMutex

	opts atomic.Value // 参数选项，真实类型是apps/nsqd/option.go:Options结构体

	dl        *dirlock.DirLock
	isLoading int32
	errValue  atomic.Value
	startTime time.Time

	topicMap map[string]*Topic	// 保存当前所有的topic

	clientLock sync.RWMutex
	clients    map[int64]Client

	lookupPeers atomic.Value

	tcpServer     *tcpServer
	tcpListener   net.Listener
	httpListener  net.Listener
	httpsListener net.Listener
	tlsConfig     *tls.Config

	poolSize int				// 当前工作协程组的协程数量

	notifyChan           chan interface{}
	optsNotificationChan chan struct{}
	exitChan             chan int
	waitGroup            util.WaitGroupWrapper

	ci *clusterinfo.ClusterInfo
}
```

**主要方法**

```go
/*
	程序启动时调用本方法，执行下面的动作：
		- 启动TCP/HTTP/HTTPS服务
		- 启动工作协程组：NSQD.queueScanLoop
		- 启动服务注册：NSQD.lookupLoop
*/
func (n *NSQD) Main() error

// 负责管理工作协程组的数量，每调用一次NSQD.queueScanWorker()方法启动一个工作协程
func (n *NSQD) queueScanLoop()

// 由queueScanLoop()调用，负责启动工作协程组并动态调整协程数量。工作协程的数量为当前的channel数 * 0.25
func (n *NSQD) resizePool(num int, workCh chan *Channel, responseCh chan bool, closeCh chan int)

// 这是具体的工作协程，监听workCh，对收到的待处理Channel做两个动作，一是将超时的消息重新入队；二是将到时间的延时消息入队
func (n *NSQD) queueScanWorker(workCh chan *Channel, responseCh chan bool, closeCh chan int)

/*
	lookupLoop()方法与nsqlookupd建立连接，负责向nsqlookupd注册topic，并定时发送心跳包
*/
func (n *NSQD) lookupLoop()
```

## 2.2 tcpServer

nsqd/tcp.go文件，tcpServer通过Handle()方法接收TCP请求。  
tcpServer是nsqd结构的成员，全局也就只有一个实例，但在protocol包的TCPServer方法中，每创建一个新的连接，均会调用一次tcpServer.Handle()

```haskell
type tcpServer struct {
	ctx   *context
	conns sync.Map
}
```

**主要方法**

```scss
/*
	p.nsqd.Main()启动protocol.TCPServer()，这个方法里会为每个客户端连接创建一个新协程，协程执行tcpServer.Handle()方法
	本方法首先对新连接读取4字节校验版本，新连接必须首先发送4字节"  V2"。
	然后阻塞调用nsqd.protocolV2.IOLoop()处理客户端接下来的请求。
*/
func (p *tcpServer) Handle(clientConn net.Conn)
```

## 2.3 protocolV2

nsqd/protocol_v2.go文件，protocolV2负责处理过来的具体的用户请求。  
每个连接均会创建一个独立的protocolV2实例（由tcpServer.Handle()创建）

```haskell
type protocolV2 struct {
	ctx *context
}
```

**主要方法**

```scss
/*
	tcpServer.Handle()阻塞调用本方法
	1. 启用一个独立协程向消费者推送消息protocolV2.messagePump()
	2. for循环接收并处理客户端请求protocolV2.Exec()
*/
func (p *protocolV2) IOLoop(conn net.Conn) error

// 组装消息并调用protocolV2.Send()发送给消费者
func (p *protocolV2) SendMessage(client *clientV2, msg *Message) error

// 向客户端发送数据帧
func (p *protocolV2) Send(client *clientV2, frameType int32, data []byte) error

// 解析客户端请求的指令，调用对应的指令方法
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error)

// 负责向消费者推送消息
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool)

// 下面这组方法是NSQD支持的指令对应的处理方法
func (p *protocolV2) IDENTIFY(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) AUTH(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) RDY(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) FIN(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) REQ(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) CLS(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) NOP(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) MPUB(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) DPUB(client *clientV2, params [][]byte) ([]byte, error)
func (p *protocolV2) TOUCH(client *clientV2, params [][]byte) ([]byte, error)
```

## 2.4 clientV2

nsqd/client_v2.go文件，保存每个客户端的连接信息。  
clientV2实例由protocolV2.IOLoop()创建，每个连接均有一个独立的实例。

```haskell
type clientV2 struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	ReadyCount    int64
	InFlightCount int64
	MessageCount  uint64
	FinishCount   uint64
	RequeueCount  uint64

	pubCounts map[string]uint64

	writeLock sync.RWMutex
	metaLock  sync.RWMutex

	ID        int64
	ctx       *context
	UserAgent string

	// original connection
	net.Conn

	// connections based on negotiated features
	tlsConn     *tls.Conn
	flateWriter *flate.Writer

	// reading/writing interfaces
	Reader *bufio.Reader
	Writer *bufio.Writer

	OutputBufferSize    int
	OutputBufferTimeout time.Duration

	HeartbeatInterval time.Duration

	MsgTimeout time.Duration

	State          int32
	ConnectTime    time.Time
	Channel        *Channel
	ReadyStateChan chan int
	ExitChan       chan int

	ClientID string
	Hostname string

	SampleRate int32

	IdentifyEventChan chan identifyEvent
	SubEventChan      chan *Channel

	TLS     int32
	Snappy  int32
	Deflate int32

	// re-usable buffer for reading the 4-byte lengths off the wire
	lenBuf   [4]byte
	lenSlice []byte

	AuthSecret string
	AuthState  *auth.State
}
```

## 2.5 Topic

nsqd/topic.go文件，对应于每个topic实例，由系统启动时创建或者发布消息/消费消息时自动创建。

```go
type Topic struct {
	messageCount uint64	// 累计消息数
	messageBytes uint64	// 累计消息体的字节数

	sync.RWMutex

	name              string				// topic名，生产和消费时需要指定此名称
	channelMap        map[string]*Channel	// 保存每个channel name和channel指针的映射
	backend           BackendQueue			// 磁盘队列，当内存memoryMsgChan满时，写入硬盘队列
	memoryMsgChan     chan *Message			// 消息优先存入这个内存chan
	startChan         chan int
	exitChan          chan int
	channelUpdateChan chan int
	waitGroup         util.WaitGroupWrapper
	exitFlag          int32
	idFactory         *guidFactory

	ephemeral      bool
	deleteCallback func(*Topic)
	deleter        sync.Once

	paused    int32
	pauseChan chan int

	ctx *context
}
```

**主要方法**

```scss
/*
	下面两个方法负责将消息写入topic，底层均调用topic.put()方法
	1. topic.memoryMsgChan未满时，优先写入内存memoryMsgChan
	2. 否则，写入磁盘topic.backend
*/
func (t *Topic) PutMessage(m *Message) error
func (t *Topic) PutMessages(msgs []*Message) error

/*
	NewTopic创建新的topic时会为每个topic启动一个独立线程来处理消息推送，即messagePump()
	此方法循环随机从内存memoryMsgChan和磁盘队列backend中取消息写入到topic下每一个chnnel中
*/
func (t *Topic) messagePump()
```

## 2.6 channel

nsqd/channel.go文件，对应于每个channel实例

```go
type Channel struct {
	requeueCount uint64
	messageCount uint64
	timeoutCount uint64

	sync.RWMutex

	topicName string
	name      string
	ctx       *context

	backend BackendQueue		// 磁盘队列，当内存memoryMsgChan满时，写入硬盘队列

	memoryMsgChan chan *Message	// 消息优先存入这个内存chan
	exitFlag      int32
	exitMutex     sync.RWMutex

	// state tracking
	clients        map[int64]Consumer
	paused         int32
	ephemeral      bool
	deleteCallback func(*Channel)
	deleter        sync.Once

	// Stats tracking
	e2eProcessingLatencyStream *quantile.Quantile

	// TODO: these can be DRYd up
	deferredMessages map[MessageID]*pqueue.Item	// 保存尚未到时间的延迟消费消息
	deferredPQ       pqueue.PriorityQueue		// 保存尚未到时间的延迟消费消息，最小堆
	deferredMutex    sync.Mutex
	inFlightMessages map[MessageID]*Message		// 保存已推送尚未收到FIN的消息
	inFlightPQ       inFlightPqueue				// 保存已推送尚未收到FIN的消息，最小堆
	inFlightMutex    sync.Mutex
}
```

**主要方法**

```scss
/*
	将消息写入channel，逻辑与topic的一致，内存未满则优先写内存chan，否则写入磁盘队列
*/
func (c *Channel) PutMessage(m *Message) error
func (c *Channel) put(m *Message) error

// 消费超时相关
func (c *Channel) StartInFlightTimeout(msg *Message, clientID int64, timeout time.Duration) error
func (c *Channel) pushInFlightMessage(msg *Message) error
func (c *Channel) popInFlightMessage(clientID int64, id MessageID) (*Message, error)
func (c *Channel) addToInFlightPQ(msg *Message)
func (c *Channel) removeFromInFlightPQ(msg *Message)
func (c *Channel) processInFlightQueue(t int64) bool

// 延时消费相关
func (c *Channel) StartDeferredTimeout(msg *Message, timeout time.Duration) error
func (c *Channel) pushDeferredMessage(item *pqueue.Item) error
func (c *Channel) popDeferredMessage(id MessageID) (*pqueue.Item, error)
func (c *Channel) addToDeferredPQ(item *pqueue.Item)
func (c *Channel) processDeferredQueue(t int64) bool
```

# 3 启动过程

nsqd的main函数在apps/nsqd/main.go文件。  
启动时调用了一个第三方包svc，主要作用是拦截syscall.SIGINT/syscall.SIGTERM这两个信号，最终还是调用了main.go下的3个方法：

-   program.Init()：windows下特殊操作
-   program.Start()：加载参数和配置文件、加载上一次保存的Topic信息并完成初始化、创建nsqd并调用p.nsqd.Main()启动
-   program.Stop()：退出处理

p.nsqd.Main()的逻辑也很简单，代码不贴了，依次启动了TCP服务、HTTP服务、HTTPS服务这3个服务。除此之外，还启动了以下两个协程：

-   queueScanLoop：消息延时/超时处理
-   lookupLoop：服务注册

**TCPServer**  
protocol包的TCPServer的核心代码就是下面这几行，循环等待客户端连接，并为每个连接创建一个独立的协程：

```go
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) error {
	for {
		// 等待生产者或消费者连接
		clientConn, err := listener.Accept()

		// 每创建一个连接wg +1
		wg.Add(1)
		go func() {
			// 每个连接均启动一个独立的协程来接收处理请求
			handler.Handle(clientConn)
			wg.Done()
		}()
	}

	// 等待所有协程退出
	wg.Wait()

	return nil
}
```

TCPServer的核心是为每个连接启动的协程处理方法handler.Handle(clientConn)，实际调用的是下面这个方法，连接建立时先读取4字节，必须是" V2"，然后启动prot.IOLoop(clientConn)处理接下来的客户端请求：

```go
func (p *tcpServer) Handle(clientConn net.Conn) {
	// 无论是生产者还是消费者，建立连接时，必须先发送4字节的"  V2"进行版本校验
	buf := make([]byte, 4)
	_, err := io.ReadFull(clientConn, buf)
	protocolMagic := string(buf)

	var prot protocol.Protocol
	switch protocolMagic {
	case "  V2":
		prot = &protocolV2{ctx: p.ctx}
	default:
		protocol.SendFramedResponse(clientConn, frameTypeError, []byte("E_BAD_PROTOCOL"))
		clientConn.Close()
		p.ctx.nsqd.logf(LOG_ERROR, "client(%s) bad protocol magic '%s'",
			clientConn.RemoteAddr(), protocolMagic)
		return
	}

	// 版本校验通过，保存连接信息，key-是ADDR，value是当前连接指针
	p.conns.Store(clientConn.RemoteAddr(), clientConn)

	// 启动
	err = prot.IOLoop(clientConn)
	if err != nil {
		p.ctx.nsqd.logf(LOG_ERROR, "client(%s) - %s", clientConn.RemoteAddr(), err)
	}

	p.conns.Delete(clientConn.RemoteAddr())
}
```

# 4 消费和生产过程

## 4.1 消息生产

生产者pub消息时，消息会首先写入对应topic的队列（内存优先，内存满了写磁盘），topic的messagePump()方法再将消息拷贝给每个channel。  
每个channel均各执一份完整的消息。

**1.消息写入topic**  
消息生产由生产者调用PUB/MPUB/DPUB这类指令实现，底层都是调用topic.PutMessage(msg)，进一步调用topic.put(msg)：

```go
func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m:	// 优先写入内存memoryMsgChan
	default:					// 当内存case失败即memoryMsgChan满时，走default，将msg以字节形式写入磁盘队列topic.backend
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, t.backend)
		bufferPoolPut(b)
		t.ctx.nsqd.SetHealth(err)
		if err != nil {
			t.ctx.nsqd.logf(LOG_ERROR,
				"TOPIC(%s) ERROR: failed to write message to backend - %s",
				t.name, err)
			return err
		}
	}
	return nil
}
```

消息写入topic的逻辑比较简单，优先写memoryMsgChan，如果memoryMsgChan满了，则写入磁盘队列topic.backend。  
**这里留个思考题：NSQ是否支持不写内存，全部写磁盘队列？**

**2.topic将消息复制给每个channel**  
第二章介绍结构体和方法时，介绍了topic结构体的messagePump()方法，正是这个方法将第1步写入的消息复制给每个channel的：

```go
func (t *Topic) messagePump() {
	/* 准备工作有代码我们略过 */
	// 主消息处理循环
	for {
		select {
		case msg = <-memoryMsgChan:
		case buf = <-backendChan:
			msg, err = decodeMessage(buf)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
		case <-t.channelUpdateChan:
			chans = chans[:0]
			t.RLock()
			for _, c := range t.channelMap {
				chans = append(chans, c)
			}
			t.RUnlock()
			if len(chans) == 0 || t.IsPaused() {
				memoryMsgChan = nil
				backendChan = nil
			} else {
				memoryMsgChan = t.memoryMsgChan
				backendChan = t.backend.ReadChan()
			}
			continue
		case <-t.pauseChan:
			if len(chans) == 0 || t.IsPaused() {
				memoryMsgChan = nil
				backendChan = nil
			} else {
				memoryMsgChan = t.memoryMsgChan
				backendChan = t.backend.ReadChan()
			}
			continue
		case <-t.exitChan:
			goto exit
		}

		for i, channel := range chans {
			chanMsg := msg
			/* channel消费消息时，需要处理延时/超时等问题，所以这里复制了消息，给每个channel传递的是独立的消息实例 */
			if i > 0 {
				chanMsg = NewMessage(msg.ID, msg.Body)
				chanMsg.Timestamp = msg.Timestamp
				chanMsg.deferred = msg.deferred
			}
			if chanMsg.deferred != 0 {
				channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
				continue
			}
			err := channel.PutMessage(chanMsg)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR,
					"TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
					t.name, msg.ID, channel.name, err)
			}
		}
	}
}
```

topic.messagePump()方法代码还蛮长的，前面是些准备工作，主要就是后面的for循环。其中for循环中select的前两项，memoryMsgChan来源于topic.memoryMsgChan，而backendChan则是topic.backend.ReadChan()，分别对应于内存和磁盘队列。注意只有这两个case会往下传递消息，其他的case处理退出和更新机制的，会continue或exit外层的for循环。  
虽然通道channel是有序的，但select的case具有随机性，这就决定了每轮循环读的是内存还是磁盘是随机的，消息的消费顺序是不可控的。  
select语句获取的消息，交给第2层for循环处理，逻辑比较简单，遍历每一个chan，调用channel.PutMessage()写入。由于每个channel对应于不同的消费者，有不同的延时/超时和消费机制，所以这里拷贝了message实例。

## 4.2 消息消费

每个连接均会启动一个运行protocolV2.messagePump()方法的协程，这个协程负责监听channel的消息队列并向客户端推送消息。客户端只有触发SUB指令之后，才会将channel传递给protocolV2.messagePump()，这之后消费推送才会正式开启。  
**启动消息推送**  
前面讲Tcpserver时有提到，客户端创建连接时，会调用tcpserver.Handle()，里面再调用protocolV2.IOLoop()。protocolV2.IOLoop()方法开头有下面这行：

```css
	go p.messagePump(client, messagePumpStartedChan)
```

这行创建了一个独立线程，调用的protocolV2.messagePump()负责向消费者推送消息。  
有个小细节是无论是生产者还是消费者，都会创建这个协程，protocolV2.messagePump()创建后并不会立即推送消息，而是需要调用SUB指令，以protocolV2.SUB()方法为例，方法末尾有这么一行：

```go
	client.SubEventChan <- channel
```

将当前消费者订阅的channel传入client.SubEventChan，这个会由protocolV2.messagePump()接收，这个方法核心是下面这个for循环（限于篇幅，我省略了大量无关代码）：

```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	for {
		if subChannel == nil || !client.IsReadyForMessages() {
			// the client is not ready to receive messages...
			memoryMsgChan = nil
			backendMsgChan = nil
			flusherChan = nil
			// force flush
			client.writeLock.Lock()
			err = client.Flush()
			client.writeLock.Unlock()
			if err != nil {
				goto exit
			}
			flushed = true
		} else if flushed {
			// last iteration we flushed...
			// do not select on the flusher ticker channel
			memoryMsgChan = subChannel.memoryMsgChan
			backendMsgChan = subChannel.backend.ReadChan()
			flusherChan = nil
		}
		
		select {
		case subChannel = <-subEventChan:
			// you can't SUB anymore
			subEventChan = nil
		case b := <-backendMsgChan:
			if sampleRate > 0 && rand.Int31n(100) > sampleRate {
				continue
			}

			msg, err := decodeMessage(b)
			if err != nil {
				p.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg)
			if err != nil {
				goto exit
			}
			flushed = false
		case msg := <-memoryMsgChan:
			if sampleRate > 0 && rand.Int31n(100) > sampleRate {
				continue
			}
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg)
			if err != nil {
				goto exit
			}
			flushed = false
		case <-client.ExitChan:
			goto exit
		}
	}
}
```

客户端建立连接初始，subChannel为空，循环一直走第1个if语句。直到客户端调用SUB指令，select语句执行"case subChannel = <-subEventChan:"，此时subChannel非空，接下来backendMsgChan和memoryMsgChan被赋值，此后开始推送消息：

-   消息会随机从内存和磁盘队列取，因为如果内存和磁盘都有数据，select是随机的
-   消息通过protocolV2.SendMessage()推送给消费者

**当多个消费者订阅同一个channel时情况会如何？**  
上面我们提到消费者发起SUB指令订阅消息，protocolV2.SUB()会将chan传给protocolV2.messagePump()，即这一行“client.SubEventChan <- channel”，那么我们来看下这个channel变量怎么来的：

```css
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error) {
	...
	
	topic := p.ctx.nsqd.GetTopic(topicName)
	channel = topic.GetChannel(channelName)
	
	...
	client.SubEventChan <- channel
```

SUB方法包含多种逻辑：

-   当channel不存在时，topic.GetChannel()方法自动创建并与这个消费者绑定
-   当channel存在，比如事先通过http-api创建好了，但没有消费者订阅，则当前消费者独立绑定这个channel
-   当channel存在，且已经有消费者订阅了，topic.GetChannel()方法依然会返回这个channel，这时就有多个消费者同时订阅了这个channel，大家共用一个通道chan变量

由于是多个消费者共用一个通道chan变量，每个消费者都有一个for select在循环监听这个通道，根据chan变量的特性，消费会随机发送给一位消费者，且一条消息只会推送给一个消费者。

**消费超时处理**  
protocolV2.messagePump()方法，无论是“case b := <-backendMsgChan:”还是“case msg := <-memoryMsgChan:”，在向消费者推送消息前都调用了下面这行代码：

```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout) // 省略其他代码
}

func (c *Channel) StartInFlightTimeout(msg *Message, clientID int64, timeout time.Duration) error {
	msg.pri = now.Add(timeout).UnixNano() // pri成员保存本消息超时时间
	err := c.pushInFlightMessage(msg)
	c.addToInFlightPQ(msg)
}
```

channel.StartInFlightTimeout()将消息保存到channel的inFlightMessages和inFlightPQ队列中，这两个缓存是用来处理消费超时的。  
值得注意的一个小细节是c.addToInFlightPQ(msg)将msg压入最小堆时，将msg在数组的偏移量保存到了msg.index成员中（最小堆底层是数组实现）

我们先简单看下FIN指令会做啥：

```go
func (p *protocolV2) FIN(client *clientV2, params [][]byte) ([]byte, error) {
	err = client.Channel.FinishMessage(client.ID, *id) // 省略其他代码
}

func (c *Channel) FinishMessage(clientID int64, id MessageID) error {
	// 省略其他代码
	msg, err := c.popInFlightMessage(clientID, id)

	c.removeFromInFlightPQ(msg)
}
```

FIN的动作比较简单，主要就是调用channel.FinishMessage()方法把上面写入超时缓存的msg给删除掉。

FIN从inFlightMessages中删除消息比较容易，这是个map，key是msg.id。客户端发送FIN消息时附带了msg.id。但如何从最小堆inFlightPQ中删除对应的msg呢？前面提到在入堆时的一个细节，即保存了msg的偏移量，此时正好用上。通过msg.index直接定位到msg的位置并调整堆即可。

说了这么多，最小堆的作用是啥？别急，接下来我们看下超时逻辑：  
超时逻辑由程序启动时开启的工作线程组来处理，即NSQD.queueScanLoop()方法：

```go
func (n *NSQD) queueScanLoop() {
	n.resizePool(len(channels), workCh, responseCh, closeCh)

	for {
		select {
		case <-workTicker.C: // 定时触发工作
			if len(channels) == 0 {
				continue
			}
		case <-refreshTicker.C: // 动态调整协程组的数量
			channels = n.channels()
			n.resizePool(len(channels), workCh, responseCh, closeCh)
			continue
		case <-n.exitChan:
			goto exit
		}

		num := n.getOpts().QueueScanSelectionCount
		if num > len(channels) {
			num = len(channels)
		}

	loop:
		for _, i := range util.UniqRands(num, len(channels)) {
			workCh <- channels[i] // 触发协程组工作
		}

		numDirty := 0
		for i := 0; i < num; i++ {
			if <-responseCh {
				numDirty++
			}
		}

		if float64(numDirty)/float64(num) > n.getOpts().QueueScanDirtyPercent {
			goto loop
		}
	}
}
```

NSQD.queueScanLoop()方法主要有一个for循环，内层是一个select和一个loop循环。select中，第1个定时器case <-workTicker.C的作用是定时触发工作，只有这个case会跳出select走到下面的loop。第2个定时器负责启动工作协程组并动态调整协程数量，我们来看下第2个定时器调用的resizePool()方法：

```go
func (n *NSQD) resizePool(num int, workCh chan *Channel, responseCh chan bool, closeCh chan int) {
	idealPoolSize := int(float64(num) * 0.25) // 协程数量设定为channel数的1/4
	if idealPoolSize < 1 {
		idealPoolSize = 1
	} else if idealPoolSize > n.getOpts().QueueScanWorkerPoolMax {
		idealPoolSize = n.getOpts().QueueScanWorkerPoolMax
	}
	for {
		if idealPoolSize == n.poolSize {		// 当协程数量达到协程数量设定为channel数的1/4时，退出
			break
		} else if idealPoolSize < n.poolSize {	// 否则如果当前协程数大于目标值，则通过closeCh通知部分协程退出
			// contract
			closeCh <- 1
			n.poolSize--
		} else {								// 否则协程数不够，则启动新的协程
			// expand
			n.waitGroup.Wrap(func() {
				n.queueScanWorker(workCh, responseCh, closeCh)
			})
			n.poolSize++
		}
	}
}
```

resizePool()方法上面的注释已经说的很清楚了，作用就是保持工作协程数量为当前channel数的1/4。

接下来我们看具体的工作逻辑，queueScanWorker()方法：

```go
func (n *NSQD) queueScanWorker(workCh chan *Channel, responseCh chan bool, closeCh chan int) {
	for {
		select {
		case c := <-workCh:
			now := time.Now().UnixNano()
			dirty := false
			if c.processInFlightQueue(now) {
				dirty = true
			}
			if c.processDeferredQueue(now) {
				dirty = true
			}
			responseCh <- dirty
		case <-closeCh:
			return
		}
	}
}
```

queueScanWorker()方法的代码很短，一是监听closeCh的退出信号；二是监听workCh的工作信号。workCh会将需要处理的channel传入，然后调用processInFlightQueue()清理超时的消息，调用processDeferredQueue()清理到时间的延时消息：

```go
func (c *Channel) processInFlightQueue(t int64) bool {
	dirty := false
	for {
		c.inFlightMutex.Lock()
		msg, _ := c.inFlightPQ.PeekAndShift(t)
		c.inFlightMutex.Unlock()

		if msg == nil {
			goto exit
		}
		dirty = true

		_, err := c.popInFlightMessage(msg.clientID, msg.ID)
		if err != nil {
			goto exit
		}
		atomic.AddUint64(&c.timeoutCount, 1)
		c.RLock()
		client, ok := c.clients[msg.clientID]
		c.RUnlock()
		if ok {
			client.TimedOutMessage()
		}
		c.put(msg)
	}

exit:
	return dirty
}

func (pq *inFlightPqueue) PeekAndShift(max int64) (*Message, int64) {
	if len(*pq) == 0 {
		return nil, 0
	}

	x := (*pq)[0]
	if x.pri > max {
		return nil, x.pri - max
	}
	pq.Pop()

	return x, 0
}
```

前面提到msg.pri成员保存本消息超时时间，所以PeekAndShift()返回的是最小堆里已经超时且超时时间最长的那条消息。processInFlightQueue()则将消息从超时队列中删，同时将消息重新put进channel。注意此时超时的消息put进channel后实际是排在队尾的，消费顺序将发生改变。  
processInFlightQueue()方法如果存在超时消息，返回值dirty标识true。queueScanWorker()将dirty写入responseCh。再往回看，queueScanLoop()方法统计了dirty的数量，超过一定比例会继续执行loop，而不是等待下一次定时执行。

## 4.2 延迟消费

生产者调用DPUB发布的消息，可以指定延时多少再推送给消费者。  
我们来看下DPUB的逻辑：

```css
func (p *protocolV2) DPUB(client *clientV2, params [][]byte) ([]byte, error) {
	timeoutMs, err := protocol.ByteToBase10(params[2])
	timeoutDuration := time.Duration(timeoutMs) * time.Millisecond
	msg := NewMessage(topic.GenerateID(), messageBody)
	msg.deferred = timeoutDuration
	err = topic.PutMessage(msg)
}
```

从上面截取的PUB()方法代码可以看出，DPUB的消息会将延时时间写入msg.deferred成员。4.1章节第2部分介绍的Topic.messagePump()方法有下面这段：

```go
func (t *Topic) messagePump() {
			if chanMsg.deferred != 0 {
				channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
				continue
			}
}
```

当chanMsg.deferred != 0时表示延时消息，此时不是直接调用putMessage()方法写入channel，而是调用channel.PutMessageDeferred(chanMsg, chanMsg.deferred)，消息被写入了延时队列Channel.deferredMessages和Channel.deferredPQ。之后的逻辑是在工作协程组NSQD.queueScanLoop()中被识别并put进channel，这与超时的处理逻辑是一样的，不展开说。

# Reference
https://www.cnblogs.com/JoZSM/p/12248378.html