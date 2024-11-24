#nsq 

# 使用示例

NSQ的golang版客户端：[https://github.com/nsqio/go-nsq](https://github.com/nsqio/go-nsq)  
提供了生产者和消费者的接口封装。  
doc.go文件描述的蛮清楚的:  
**生产者Producer**

```go
	// 创建一个生产者实例
	// 值得注意的是，Producer与Nsqd是一对一的关系，消息不会在Nsqd间传输
	// 所以如果要实现多Nsqd实例负载
	config := nsq.NewConfig()
	producer, err := nsq.NewProducer("127.0.0.1:4150", config)
	if err != nil {
		log.Fatal(err)
	}

	messageBody := []byte("hello")
	topicName := "topic"

	// 以同步的方式推送消息
	// 用户也可以选择其他的api方法以异步或批量的方式推送
	err = p.Publish(topicName, messageBody)
	if err != nil {
		log.Fatal(err)
	}

	// 优雅退出
	producer.Stop()
```

**消费者Consumer**

```go
	type myMessageHandler struct {} // 自定义结构体，用于实现nsq.Handler接口

	// 由用户实现的方法，用于实现nsq.Handler接口
	// nsq.Handler接口只有一个方法，即'HandleMessage(message *Message) error'
	// Consumer收到消息后，会回调此方法完成处理
	func (h *myMessageHandler) HandleMessage(m *nsq.Message) error {
		if len(m.Body) == 0 {
			// 空消息返回nil，将自动向Nsqd响应FIN用于销毁消息
			return nil
		}

		err := processMessage(m.Body)

		// 消息消费异常时可以返回err，将向Nsqd响应REQ。此时本条消息将会重新入队继续推送
		return err
	}

	func main() {
		// 创建Consumer实例
		config := nsq.NewConfig()
		consumer, err := nsq.NewConsumer("topic", "channel", config)
		if err != nil {
			log.Fatal(err)
		}

		// 指定自定义的nsq.Handler接口，收到的消息通过回调此接口来消费
		consumer.AddHandler(&myMessageHandler{})

		// 通过nsqlookupd来实现服务自动发现，会定时获取新的注册信息。
		// 类似的还有：ConnectToNSQD, ConnectToNSQDs, ConnectToNSQLookupds.
		err = consumer.ConnectToNSQLookupd("localhost:4161")
		if err != nil {
			log.Fatal(err)
		}

		// 优雅退出
		consumer.Stop()
	}
```

# 配置参数

无论是生产者还是消费者，创建时均需要提供一个Config结构体，nsq.NewConfig()方法可以创建并生成一个有默认参数的Config。  
Config源码见config.go。

注意：

-   **只能通过nsq.NewConfig()方法来创建Config实例，该方法提供Config的默认初始化和验证句柄。否则使用时可能会抛Panic**
-   可以调用Config.Set()方法来修改参数值。注意config实例传递给Producer或Consumer时是深度拷贝的，所以传递之后修改原有的config实例是无效的。
-   参数有默认值和可设置的范围限制，具体可参考下表。

下面是所有可选参数的介绍：  
_下表仅供参考，不同版本的代码可能会有改动，请查看自己的config.go文件_
![[Pasted image 20220309173210.png]]
![[Pasted image 20220309173222.png]]


# 生产者Producer

## 总结

Producer源码见producer.go。

-   Producer支持并发，底层通过一个channel将消息发送给一个独立的协程来最终发布给Nsqd，从而解决并发冲突。
-   一个Producer只支持一个Nsqd。如果要支持集群负载均衡，需要自己实现。

Producer提供了6个方法用于发布消息：

-   Publish()：阻塞发布1条消息。底层调用"PUB"指令
-   PublishAsync()：非阻塞发布1条消息。相比Publish()，多了一个额外的doneChan参数，通过此chan来异步接收发布结果。
-   MultiPublish()：阻塞发布多条消息。底层调用"MPUB"指令
-   MultiPublishAsync()：非阻塞发布多条消息。通过doneChan来异步接收发布结果。
-   DeferredPublish()：阻塞发布1条带延时的消息。相比Publish()，多了一个delay参数来指定延时多久才推送给消费者。底层调用"DPUB"指令
-   DeferredPublishAsync()：非阻塞发布1条带延时的消息。通过doneChan来异步接收发布结果。

## Producer结构体

每一个Producer实例，均是一个结构体：

```go
type Producer struct {
	id     int64			// 用于打印日志时标示实例。由instCount全局变量控制，从0开始，每创建一个Producer或Consumer时+1
	addr   string			// 连接的Nsqd的地址
	conn   producerConn		// 连接实例
	config Config			// 配置参数

	logger   []logger
	logLvl   LogLevel
	logGuard sync.RWMutex

	responseChan chan []byte	// conn收到Nsqd生产成功的响应后，通过此chan告知router()，router再通知到生产消息的线程
	errorChan    chan []byte	// conn收到Nsqd错误信息的响应后，通过此chan告知router()，router再通知到生产消息的线程
	closeChan    chan int		// conn断开时通过此chan告知router()结束

	transactionChan chan *ProducerTransaction	// 生产过程将消息推送到这个chan，再异步接收成功的结果
	transactions    []*ProducerTransaction		// 一个先入先出队列，用于router协程处理消息写入结果
	state           int32						// 连接状态，初始状态/连接断开/已连接

	concurrentProducers int32	// 统计正在等待发往transactionChan的消息数，Producer在退出前会将这些消息置为ErrNotConnected
	stopFlag            int32
	exitChan            chan int
	wg                  sync.WaitGroup	// 用于等待router()协程退出
	guard               sync.Mutex		// Producer全局锁
}
```

## Producer发布消息的方法介绍

Producer提供了6个方法用于发布消息：

-   Publish()：阻塞发布1条消息。底层调用"PUB"指令
-   PublishAsync()：非阻塞发布1条消息。相比Publish()，多了一个额外的doneChan参数，通过此chan来异步接收发布结果。
-   MultiPublish()：阻塞发布多条消息。底层调用"MPUB"指令
-   MultiPublishAsync()：非阻塞发布多条消息。通过doneChan来异步接收发布结果。
-   DeferredPublish()：阻塞发布1条带延时的消息。相比Publish()，多了一个delay参数来指定延时多久才推送给消费者。底层调用"DPUB"指令
-   DeferredPublishAsync()：非阻塞发布1条带延时的消息。通过doneChan来异步接收发布结果。

上述6个方法，最终均调用sendCommandAsync()方法来完成发送。  
对于阻塞调用的3个方法，会先调用sendCommand()，创建一个临时chan，再调用sendCommandAsync()。

-   sendCommand()方法创建一个临时的doneChan来接收发布结果。
-   sendCommandAsync()负责将消息写入Producer.transactionChan。下一章节的router协程负责接收并将消息发往Nsqd，并将发布结果通过doneChan返回。如果连接尚未创建，这里会自动重建连接。

```go
func (w *Producer) sendCommand(cmd *Command) error {
	doneChan := make(chan *ProducerTransaction)		// 临时的doneChan用于接收发布结果
	err := w.sendCommandAsync(cmd, doneChan, nil)	// 发送消息
	if err != nil {
		close(doneChan)
		return err
	}
	t := <-doneChan									// 等待发送结果
	return t.Error
}

func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction,
	args []interface{}) error {
	// keep track of how many outstanding producers we're dealing with
	// in order to later ensure that we clean them all up...
	atomic.AddInt32(&w.concurrentProducers, 1)
	defer atomic.AddInt32(&w.concurrentProducers, -1)

	if atomic.LoadInt32(&w.state) != StateConnected {
		err := w.connect()	// 未连接时自动重建连接
		if err != nil {
			return err
		}
	}

	t := &ProducerTransaction{
		cmd:      cmd,
		doneChan: doneChan,
		Args:     args,
	}

	select {
	case w.transactionChan <- t:	// 将消息发送给router协程
	case <-w.exitChan:
		return ErrStopped
	}

	return nil
}
```

sendCommand()方法是Producer实现并发的核心，即并发的消息发布，最终都会写入transactionChan channel，由router协程独立处理，不存在并发冲突。

## Producer主要的辅助方法介绍

**自动创建连接**  
nsq.NewProducer()方法只创建Producer实例，连接会在后续自动管理。  
每次发送指令Ping()或sendCommandAsync()时，如果尚未连接，会自动调用Producer.connect()方法。

```go
// 创建连接，修改连接状态，启动router()协程
func (w *Producer) connect() error
```

**router协程异步发送和接收消息响应**  
Producer通过起一个router协程异步发送和接收消息响应的方式来实现Producer并发写入的问题。无论用户有多少个线程在生产消息，最终都得调用sendCommandAsync()方法将消息写入一个chan，并由router单协程处理，这就避免了并发冲突。  
router协程在上文的Producer.connect()方法被启动。  
Router()方法起了个for循环持续监听几个chan，我们重点关注：

-   transactionChan：所有消息最终均通过sendCommandAsync()方法写入这个chan。router协程负责将从transactionChan收到的消息，写入Producer.conn，并最终发送到Nsqd。
-   responseChan：Nsqd每正确接收到一个消息，会响应一个确认帧回来。Producer.conn则通过此chan来告知router消息写入成功。
-   errorChan：同responseChan，收到错误信息时，通过此chan告知router。

无论是写入成功还是有错误，router协程均调用popTransaction()方法来处理。这个方法有个细节，Nsqd并没有告知写入成功或失败的消息是哪条，Producer又是怎么知道的呢？原理是底层使用的TCP通讯，同学们可以回想下TCP的特点，TCP是有序的。写入消息的只有router一个协程，所以消息是按顺序写入的，恰恰Nsqd端也是单线程处理同一个生产者。所以router收到的响应，必然是针对transactions队列中第1条消息的（这是一个用切片实现的先入先出队列，router会在写入conn的同时将消息写入这个队列）。

收到Nsqd的响应后，router将结束写入ProducerTransaction.doneChan，用于通知消息的写入协程。

```go
func (w *Producer) router() {
	for {
		select {
		case t := <-w.transactionChan:	// 这是待发布的消息
			w.transactions = append(w.transactions, t)	// 先入先出队列，用于处理消息发布结果
			err := w.conn.WriteCommand(t.cmd)			// 发布消息
			if err != nil {
				w.log(LogLevelError, "(%s) sending command - %s", w.conn.String(), err)
				w.close()
			}
		case data := <-w.responseChan:	// 发布成功的响应
			w.popTransaction(FrameTypeResponse, data)	// 处理发布结果，将结果写入doneChan
		case data := <-w.errorChan:		// 发布失败的响应
			w.popTransaction(FrameTypeError, data)		// 处理发布结果，将结果写入doneChan
		case <-w.closeChan:
			goto exit
		case <-w.exitChan:
			goto exit
		}
	}

exit:
	w.transactionCleanup()
	w.wg.Done()
	w.log(LogLevelInfo, "exiting router")
}

func (w *Producer) popTransaction(frameType int32, data []byte) {
	t := w.transactions[0]
	w.transactions = w.transactions[1:]		// 发布成功或失败的消息，出队
	if frameType == FrameTypeError {
		t.Error = ErrProtocol{string(data)}	// 发布失败的错误信息
	}
	t.finish()								// 通知到doneChan
}

func (t *ProducerTransaction) finish() {
	if t.doneChan != nil {
		t.doneChan <- t
	}
}
```

**Ping**

```go
// Ping方法一般用于刚创建的Producer实例。自动connect()方法创建连接，并发送一条Nop指令，以确认连接是否正常
func (w *Producer) Ping() error
```

**优雅退出**  
注意：_主动退出Producer时，建议使用Stop()方法，用于结束正在等待发送的消息，同时结束router协程。否则两者将可能一直阻塞_

```scss
// Stop()方法用于优雅退出当前Producer。
// 正在等待发送的消息将被置为ErrNotConnected或ErrStopped
func (w *Producer) Stop()
```

# 消费者Consumer

## 总结

Producer源码见producer.go。

-   Producer支持并发，底层通过一个channel将消息发送给一个独立的协程来最终发布给Nsqd，从而解决并发冲突。
-   一个Producer只支持一个Nsqd。如果要支持集群负载均衡，需要自己实现。

Producer提供了6个方法用于发布消息：

-   Publish()：阻塞发布1条消息。底层调用"PUB"指令
-   PublishAsync()：非阻塞发布1条消息。相比Publish()，多了一个额外的doneChan参数，通过此chan来异步接收发布结果。
-   MultiPublish()：阻塞发布多条消息。底层调用"MPUB"指令
-   MultiPublishAsync()：非阻塞发布多条消息。通过doneChan来异步接收发布结果。
-   DeferredPublish()：阻塞发布1条带延时的消息。相比Publish()，多了一个delay参数来指定延时多久才推送给消费者。底层调用"DPUB"指令
-   DeferredPublishAsync()：非阻塞发布1条带延时的消息。通过doneChan来异步接收发布结果。

Consumer源码见consumer.go。

-   Consumer支持并发，调用AddConcurrentHandlers()方法指定创建多个handlerLoop协程进行处理即可。
-   ConnectToNSQD()和ConnectToNSQDs()方法可以指定1个或多个Nsqd创建连接，但不具备服务发现和动态调整。
-   ConnectToNSQLookupd()和ConnectToNSQLookupds()方法可以指定1个或多个lookup创建连接，支持服务发现和定时动态调整。

## Consumer结构体

每个Consumer实例对应于一个Consumer结构体：

```go
type Consumer struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	messagesReceived uint64
	messagesFinished uint64
	messagesRequeued uint64
	totalRdyCount    int64
	backoffDuration  int64
	backoffCounter   int32
	maxInFlight      int32

	mtx sync.RWMutex

	logger   []logger
	logLvl   LogLevel
	logGuard sync.RWMutex

	behaviorDelegate interface{}

	id      int64
	topic   string
	channel string
	config  Config

	rngMtx sync.Mutex
	rng    *rand.Rand

	needRDYRedistributed int32

	backoffMtx sync.Mutex

	incomingMessages chan *Message

	rdyRetryMtx    sync.Mutex
	rdyRetryTimers map[string]*time.Timer

	pendingConnections map[string]*Conn
	connections        map[string]*Conn

	nsqdTCPAddrs []string

	// used at connection close to force a possible reconnect
	lookupdRecheckChan chan int
	lookupdHTTPAddrs   []string
	lookupdQueryIndex  int

	wg              sync.WaitGroup
	runningHandlers int32
	stopFlag        int32
	connectedFlag   int32
	stopHandler     sync.Once
	exitHandler     sync.Once

	// read from this channel to block until consumer is cleanly stopped
	StopChan chan int
	exitChan chan int
}
```

## 注册回调函数和消息消费

Consumer每收到一条消息，会调用我们指定的回调接口来消费消息。  
这个回调接口如下：

```go
// 返回nil表示消费成功，Consumer将向Nsqd发送FIN指令销毁消息。
// 非nil表示消费失败或需要重复消费，Consumer将向Nsqd发送REQ指令将消息重新入队推送。
type Handler interface {
	HandleMessage(message *Message) error
}
```

我们在启动Consumer，需要先用一个结构体实现上述Handler接口，将调用AddHandler()方法将该结构体传给Consumer：

```go
/*自定义结构体示例*/
type myMessageHandler struct {} // 自定义结构体，用于实现nsq.Handler接口

func (h *myMessageHandler) HandleMessage(m *nsq.Message) error {
	if len(m.Body) == 0 {
		return nil
	}

	err := processMessage(m.Body)

	return err
}

// 在启动Consumer前调用：consumer.AddHandler(&myMessageHandler{})方法将结构体传递给Consumer
```

每调用一次consumer.AddHandler()方法会启动一个handlerLoop协程用于循环接收消息：

```go
// 必须在Consumer连接之前调用AddHandler()方法，否则会抛panic
// 每调用一次AddHandler()就创建一个handlerLoop协程，可以多次调用来创建多个协程
func (r *Consumer) AddHandler(handler Handler) {
	r.AddConcurrentHandlers(handler, 1)
}

func (r *Consumer) AddConcurrentHandlers(handler Handler, concurrency int) {
	if atomic.LoadInt32(&r.connectedFlag) == 1 {
		panic("already connected") // 必须在Consumer连接之前调用AddHandler()方法，否则会抛panic
	}

	atomic.AddInt32(&r.runningHandlers, int32(concurrency))
	for i := 0; i < concurrency; i++ {
		go r.handlerLoop(handler) // 每调用一次AddHandler()就创建一个handlerLoop协程，可以多次调用来创建多个协程
	}
}
```

**如果消费速度较慢，可以在连接之前直接调用AddConcurrentHandlers()以指定创建多个协程来并发处理**

每个handlerLoop协程都在监听incomingMessages，收到消息则调用Handler消费：

```scss
func (r *Consumer) handlerLoop(handler Handler) {
	r.log(LogLevelDebug, "starting Handler")

	for {
		message, ok := <-r.incomingMessages // 可以创建多个协程并发监听这个channel
		if !ok {
			goto exit
		}

		if r.shouldFailMessage(message, handler) {
			message.Finish()
			continue
		}

		err := handler.HandleMessage(message)
		if err != nil {
			r.log(LogLevelError, "Handler returned error (%s) for msg %s", err, message.ID)
			if !message.IsAutoResponseDisabled() {
				message.Requeue(-1)
			}
			continue
		}

		if !message.IsAutoResponseDisabled() {
			message.Finish()
		}
	}

exit:
	r.log(LogLevelDebug, "stopping Handler")
	if atomic.AddInt32(&r.runningHandlers, -1) == 0 {
		r.exit()
	}
}
```

## 创建连接

Consumer提供了2个方法用于创建连接：

-   ConnectToNSQD()：指定一个Nsqd连接。推荐使用lookupd以便自动服务发现。
-   ConnectToNSQDs()：指定多个Nsqd连接。推荐使用lookupd以便自动服务发现。

注意，直接连接Nsqd的方法不推荐使用，建议使用下一章节的lookup服务发现，以便自动发现和管理。

ConnectToNSQDs()方法就是个for循环，调用多次ConnectToNSQD()。  
ConnectToNSQD()方法比较长，关键的点是，如果未注册回调方法，会报错；置状态为已连接；新建一个连接，建立连接时会发送"IDENTIFY"指令通告此连接的相关参数；向Nsqd发送Sub指令开始订阅；订阅前连接暂时放在pendingConnections中，订阅成功后从pendingConnections移到connections中。  
ConnectToNSQD()最终对connections中所有连接执行maybeUpdateRDY()方法用于调整RDY计数。

```go
func (r *Consumer) ConnectToNSQD(addr string) error {
	if atomic.LoadInt32(&r.stopFlag) == 1 {
		return errors.New("consumer stopped")
	}

	if atomic.LoadInt32(&r.runningHandlers) == 0 { // 如果未注册回调方法，会报错
		return errors.New("no handlers")
	}

	atomic.StoreInt32(&r.connectedFlag, 1)

	conn := NewConn(addr, &r.config, &consumerConnDelegate{r})
	conn.SetLoggerLevel(r.getLogLevel())
	format := fmt.Sprintf("%3d [%s/%s] (%%s)", r.id, r.topic, r.channel)
	for index := range r.logger {
		conn.SetLoggerForLevel(r.logger[index], LogLevel(index), format)
	}
	r.mtx.Lock()
	_, pendingOk := r.pendingConnections[addr]
	_, ok := r.connections[addr]
	if ok || pendingOk {
		r.mtx.Unlock()
		return ErrAlreadyConnected
	}
	r.pendingConnections[addr] = conn
	if idx := indexOf(addr, r.nsqdTCPAddrs); idx == -1 {
		r.nsqdTCPAddrs = append(r.nsqdTCPAddrs, addr)
	}
	r.mtx.Unlock()

	r.log(LogLevelInfo, "(%s) connecting to nsqd", addr)

	cleanupConnection := func() {
		r.mtx.Lock()
		delete(r.pendingConnections, addr)
		r.mtx.Unlock()
		conn.Close()
	}

	resp, err := conn.Connect()
	if err != nil {
		cleanupConnection()
		return err
	}

	if resp != nil {
		if resp.MaxRdyCount < int64(r.getMaxInFlight()) {
			r.log(LogLevelWarning,
				"(%s) max RDY count %d < consumer max in flight %d, truncation possible",
				conn.String(), resp.MaxRdyCount, r.getMaxInFlight())
		}
	}

	cmd := Subscribe(r.topic, r.channel)
	err = conn.WriteCommand(cmd)
	if err != nil {
		cleanupConnection()
		return fmt.Errorf("[%s] failed to subscribe to %s:%s - %s",
			conn, r.topic, r.channel, err.Error())
	}

	r.mtx.Lock()
	delete(r.pendingConnections, addr)
	r.connections[addr] = conn
	r.mtx.Unlock()

	// pre-emptive signal to existing connections to lower their RDY count
	for _, c := range r.conns() {
		r.maybeUpdateRDY(c)
	}

	return nil
}
```

maybeUpdateRDY()方法调用perConnMaxInFlight()获取当前连接的最大允许RDY计数，最大允许RDY计数为配置参数maxInFlight / 当前连接数。其中maxInFligh如果未主动配置的话默认为1，那么所有连接的RDY均为1。之后再调用updateRDY()方法来调整RDY计数。最终每个连接会向Nsqd发送RDY指令调整计数。  
整段代码调用较长，也没太多可分析的点，所以代码就不贴了。  
_maybeUpdateRDY()只是初调整，还有一个由NewConsumer()启动的rdyLoop()协程会定时根据每个连接的状态进一步调整，调整原则是将长时间未收到消息的连接的RDY置为最小值1_

讲了这么多，那么消息是怎么接收并传递过来的呢？  
前面的handlerLoop协程，监听incomingMessages通道。所以我们全局搜incomingMessages通道，发现Consumer.onConnMessage()方法会向这个通道写入消息。而这个方法由consumerConnDelegate.OnMessage()调用。进一步定位到每个连接的Conn.readLoop()方法。  
Conn.readLoop()方法的逻辑比较简单，就是一个for循环阻塞等待Nsqd推送消息，解析后如果是一条消息，则调用consumerConnDelegate.OnMessage()将消息写入到incomingMessages通道。

```haskell
func (c *Conn) readLoop() {
	for {
		frameType, data, err := ReadUnpackedResponse(c) // 阻塞等待Nsqd推送消息，并完成消息解析
		
		switch frameType {
		case FrameTypeResponse:
			c.delegate.OnResponse(c, data)
		case FrameTypeMessage:
			msg, err := DecodeMessage(data)
			if err != nil {
				c.log(LogLevelError, "IO error - %s", err)
				c.delegate.OnIOError(c, err)
				goto exit
			}
			msg.Delegate = delegate
			msg.NSQDAddress = c.String()

			atomic.AddInt64(&c.messagesInFlight, 1)
			atomic.StoreInt64(&c.lastMsgTimestamp, time.Now().UnixNano())

			c.delegate.OnMessage(c, msg)	// 将消息写入incomingMessages通道
		case FrameTypeError:
			c.log(LogLevelError, "protocol error - %s", data)
			c.delegate.OnError(c, data)
		default:
			c.log(LogLevelError, "IO error - %s", err)
			c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
		}
	}
}
```

## 服务发现和自动连接管理

Consumer提供了2个方法用于服务发现：

-   ConnectToNSQLookupd()：通过lookupd服务发现，再自动创建连接。支持定时更新服务发现。
-   ConnectToNSQLookupds()：通过多个lookupd服务发现，再自动创建连接。支持定时更新服务发现。

ConnectToNSQLookupds()方法就是为每个lookupd调用一次ConnectToNSQLookupd()。  
我们来关注ConnectToNSQLookupd()做了啥。ConnectToNSQLookupd()也要求必须先注册回调。对于第1个lookup，创建lookupdLoop协程完成自动的服务发现。后续的lookup只需要加入到lookupdHTTPAddrs中，由lookupdLoop协程来协调。

```go
func (r *Consumer) ConnectToNSQLookupd(addr string) error {
	if atomic.LoadInt32(&r.stopFlag) == 1 {
		return errors.New("consumer stopped")
	}
	if atomic.LoadInt32(&r.runningHandlers) == 0 {
		return errors.New("no handlers") // 同ConnectToNSQD()，必须先注册回调
	}

	if err := validatedLookupAddr(addr); err != nil {
		return err
	}

	atomic.StoreInt32(&r.connectedFlag, 1) // 置连接位

	r.mtx.Lock()
	for _, x := range r.lookupdHTTPAddrs {
		if x == addr {
			r.mtx.Unlock()
			return nil
		}
	}
	r.lookupdHTTPAddrs = append(r.lookupdHTTPAddrs, addr)	// 将lookupd的addr加入到lookupdHTTPAddrs切片
	numLookupd := len(r.lookupdHTTPAddrs)
	r.mtx.Unlock()

	// 只有第1个loopupd才会创建lookupdLoop协程。
	if numLookupd == 1 {
		r.queryLookupd()	// 执行一次服务发现，里面最多重复3次，如果都失败了，也不影响程序继续运行
		r.wg.Add(1)
		go r.lookupdLoop()	// 创建lookupdLoop协程完成自动的服务发现
	}

	return nil
}
```

我们重点看下lookupdLoop协程的工作，这是服务发现和自动管理的关键：  
lookupdLoop协程启动时先用jitter变量执行了一段随机延时，用于防止多个消费者监听同一Topic且同时重启的情况。由r.config.LookupdPollInterval变量创建了一个定时器，这是定时服务发现的间隔，查看config.go可知这个参数最小10ms，最大5m，默认为60s。  
接着lookupdLoop协程进入for循环，主要做两件事，一是通过ticker.C定时做一次服务发现，二是通过r.lookupdRecheckChan做一次服务发现，这个通道是在连接异常断开时会触发。

```go
func (r *Consumer) lookupdLoop() {
	// add some jitter so that multiple consumers discovering the same topic,
	// when restarted at the same time, dont all connect at once.
	r.rngMtx.Lock()
	jitter := time.Duration(int64(r.rng.Float64() *
		r.config.LookupdPollJitter * float64(r.config.LookupdPollInterval)))
	r.rngMtx.Unlock()
	var ticker *time.Ticker

	select {
	case <-time.After(jitter):
	case <-r.exitChan:
		goto exit
	}

	ticker = time.NewTicker(r.config.LookupdPollInterval)

	for {
		select {
		case <-ticker.C:
			r.queryLookupd()
		case <-r.lookupdRecheckChan:
			r.queryLookupd()
		case <-r.exitChan:
			goto exit
		}
	}

exit:
	if ticker != nil {
		ticker.Stop()
	}
	r.log(LogLevelInfo, "exiting lookupdLoop")
	r.wg.Done()
}
```

具体的服务发现由queryLookupd()方法完成，该方法每次执行调用nextLookupdEndpoint()方法取下个lookup并通过http的方式发送lookup指令，参数包含需要消费的Topic。最多请求3个lookup，有1次成功即停止，如果3次都失败就退出等待下一次服务发现。lookup返回lookupResp结构体，内容为包含请求的Topic的Nsqd相关信息列表：

```go
type lookupResp struct {
	Channels  []string    `json:"channels"`
	Producers []*peerInfo `json:"producers"`
	Timestamp int64       `json:"timestamp"`
}
```

服务发现信息包含该lookup的channel列表和生产者列表。  
然后对每一个生产者的Nsqd调用ConnectToNSQD()方法，该方法会完成去重，已经连接的Nsqd不会重复连接。

# 性能和故障分析

## Consumer

消费者Consumer客户端对并发和负载较友好，可以通过以下手段来增强并发和负载：

-   同时连接多个Nsqd。注意Nsqd之间不共享数据，需要生产者端将消息写入不同的Nsqd。
-   调用AddConcurrentHandlers来启用多个消息处理协程，增大消息消费的能力。
-   合适配置参数增加网络吞吐量。注意参数配置需要配套合理，比如如果将Flight消息数提高很高，但消费能力不行，就有可能出现Nsqd端大量消息超时的问题。可以提升网络吞吐量的参数主要有这几个：
    -   MaxInFlight：该参数控制Nsqd端允许的已发送但尚未收到FIN帧的消息数，增大该值可以增大网络中正在发送的消息数。
    -   OutputBufferSize：增大Nsqd端发送缓冲区的大小。

## Producer

一条一条发布消息较慢，如果可以，建议用MPUB相关的接口批量发布。

但一个Nsqd处理能力有限，有时我们需要集群来负载。遗憾的是，Producer与Nsqd是一对一的关系。可以有以下思路优化：

-   不同的Topic发布到不同的Nsqd上。
-   将消息分散发布到多个Nsqd上，需要自行实现。

## 重复消费

即使只发布一份的消息，在极端情况下也存在重复消费的可能，当某条消息超时（回顾：[NSQ源码剖析(一)：NSQD主要结构方法和消息生产消费过程](https://www.cnblogs.com/JoZSM/p/12248378.html "NSQ源码剖析(一)：NSQD主要结构方法和消息生产消费过程") ），消息会重新入队。入队之后如果消费者端因网络或性能问题现在才完成消费，此时FIN指令将被响应"E_FIN_FAILED"。消息还在队列中等待第二次推送。

## 消费顺序

Nsqd不保证消息推送顺序，这与延时/超时/磁盘队列等设计有关，这里不展开讨论，请回顾：[NSQ源码剖析(一)：NSQD主要结构方法和消息生产消费过程](https://www.cnblogs.com/JoZSM/p/12248378.html "NSQ源码剖析(一)：NSQD主要结构方法和消息生产消费过程")

## 单点问题

存在两种单点问题，一是异常退出的Nsqd会丢失掉尚在内存的Msg；二是与宕机的Nsqd相连的Producer将无法工作。

**宕机时丢失内存Msg**  
可以有两种思路进行规避：

-   降低内存channel的大小，甚至是0，让所有数据都压入磁盘队列。这种方式会降低性能，同时磁盘队列本身也有内存缓冲，存在丢失可能。
-   消息副本。生产者发布消息时，创建副本发布到不同的Nsqd上。这要求消费者有能力识别出重复消费的情况，或者实现消费幂等。

**Nsqd宕机导致Producer不工作**  
前面介绍过Producer与Nsqd是一对一的关系，一旦Nsqd宕机，与之相连的Producer也将无法工作。这种情况可以自行实现异常检测和故障转移，比如检测到宕机时连接到另一台Nsqd实例；或者实现一个自动的负载均衡机制。  
无论使用哪种方式，都要求Consumer端支持服务发现，这点上官方api就支持，使用ConnectToNSQLookupd()或ConnectToNSQLookupds()即可。

由于时间因素，本来打算实现一个Producer的负载均衡算法的，现在来不及。有兴趣的同学可以尝试下，或者关注我的github，欢迎参与进来讨论和研究。

# Reference
https://www.cnblogs.com/JoZSM/p/12443544.html