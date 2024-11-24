#grpc 


  在客户端跟服务器端交互过程中，还有一个问题没有介绍；就是帧是如何发送给对方的，对方又是如何接收帧的；因此，本章节会从这个角度介绍。  
  在接下来的文章中，先介绍[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)客户端帧接收器的原理；

# 1、grpc客户端和grpc服务器端建立起的链接，帧接收器，流之间的关系？

  一个底层连接，对应着一个帧接收器，而一个帧接收器可以接收多个流。  
  至于流的概念，将来在其他章节再介绍。

# 2、grpc客户端的帧接收器是在什么地方启动的？

在grpc客户端跟grpc服务器端底层连接建立后，创建的；  
进入grpc-go/internal/transport/http2_client.go文件中的newHTTP2Client函数里：

```go
1．   if t.keepaliveEnabled {
2．		t.kpDormancyCond = sync.NewCond(&t.mu)
3．		go t.keepalive()
4．	}
5．	// Start the reader goroutine for incoming message. Each transport has
6．	// a dedicated goroutine which reads HTTP2 frame from network. Then it
7．	// dispatches the frame to the corresponding stream entity.
8．	go t.reader()

9．	// Send connection preface to server.
10．	//	clientPreface, 就是 http2 中的"PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"
11．	n, err := t.conn.Write(clientPreface)
12．	if err != nil {
13．		t.Close()
14．		return nil, connectionErrorf(true, err, "transport: failed to write client preface: %v", err)
15．  }

```

第8行，异步方式启动一个帧接收器；

# 3、帧接收器原理

## 3.1、帧接收器整体流程处理图

如下图所示：

![grpc帧接收器整体原理图](https://img-blog.csdnimg.cn/20210612095539951.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)  
grpc客户端的帧接收器的原理，主要分为两个阶段：

-   第一阶段：适用于rpc连接建立阶段；
    -   当grpc客户端跟grpc服务器端建立tcp连接后，双方开始进行http2帧的初始化阶段，grpc客户端需要读取grpc服务器端发送的设置帧，如通过设置帧告诉grpc客户端，grpc服务器端的窗口大小，默认帧的大小等基本信息
-   第二阶段：适用于rpc请求阶段，即流处理阶段，如SayHello方法，具体请求、处理阶段

主要流程说明：

-   第一阶段说明：
    -   1.grpc客户端调用原生http2里的frame中的readFrame方法，来读取grpc服务器端发送的帧
    -   2.将接收到的第一个帧，强制转换为设置帧
    -   3.调用设置帧的处理器，进行处理，即将grpc服务器端中的窗口大小，默认帧大小等信息，更新到grpc客户端进程里。
-   第二阶段说明：
    -   1.首选判断帧缓冲器的阀门是否达到阈值；
        -   a)通过阀门来控制帧的接收速度
        -   b)当帧的接收速度远远超过帧的处理速度时，就会启动阀门
        -   c)从而导致，帧接收器暂停接收帧，等待解除阀门的通知；即当grpc框架有能力处理新的帧时，才解除阀门。
    -   2.若grpc框架帧的接收速度没有远超帧的接收速度时，就会阻塞式读取帧
    -   3.将读取到的帧，获取该帧的类型；如，是数据帧，还是Ping帧，还是头帧等等
    -   4.帧分发器根据帧的类型，来触发不同的帧处理器
    -   5.帧处理器处理完成后，继续返回，判断帧缓冲器阀门，继续阻塞式读取下一帧

死循环式读取该链路上的所有流的帧；

## 3.2、帧接收器源码分析

进入grpc-go/internal/transport/http2_client.go文件中的reader方法里：

```go
// reader runs as a separate goroutine in charge of reading data from network
// connection.
//
// TODO(zhaoq): currently one reader per transport. Investigate(研究) whether this is
// optimal.
// TODO(zhaoq): Check the validity of the incoming frame sequence.
1．func (t *http2Client) reader() {
2．	defer close(t.readerDone)
3．	// Check the validity of server preface.
4．	frame, err := t.framer.fr.ReadFrame()
5．	if err != nil {
6．		t.Close() // this kicks off resetTransport, so must be last before return
7．		return
8．	}
9．	t.conn.SetReadDeadline(time.Time{}) // reset deadline once we get the settings frame (we didn't time out, yay!)
10．	if t.keepaliveEnabled {
11．	   atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
12．	}
13．	sf, ok := frame.(*http2.SettingsFrame)
14．	if !ok {
15．		t.Close() // this kicks off resetTransport, so must be last before return
16．		return
17．	}

16．	t.onPrefaceReceipt()
17．	t.handleSettings(sf, true)
18．	// loop to keep reading incoming messages on this transport.
19．	for {
20．		t.controlBuf.throttle()

21．		frame, err := t.framer.fr.ReadFrame()
22．		if t.keepaliveEnabled {
23．			atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
24．		}
25．		if err != nil {
26．			// Abort an active stream if the http2.Framer returns a
27．			// http2.StreamError. This can happen only if the server's response
28．			// is malformed http2.
29．			if se, ok := err.(http2.StreamError); ok {
30．				t.mu.Lock()
31．				s := t.activeStreams[se.StreamID]
32．				t.mu.Unlock()
33．				if s != nil {
34．					// use error detail to provide better err message
35．					code := http2ErrConvTab[se.Code]
36．					msg := t.framer.fr.ErrorDetail().Error()
37．					t.closeStream(s, status.Error(code, msg), true, http2.ErrCodeProtocol, status.New(code, msg), nil, false)
38．				}
39．				continue
40．			} else {
41．				// Transport error.
42．				t.Close()
43．				return
44．			}
45．		}
46．	
47．		switch frame := frame.(type) {
48．		case *http2.MetaHeadersFrame:
49．			t.operateHeaders(frame)
50．		case *http2.DataFrame:
51．			t.handleData(frame)
52．		case *http2.RSTStreamFrame:
53．			t.handleRSTStream(frame)
54．		case *http2.SettingsFrame:
55．			t.handleSettings(frame, false)
56．		case *http2.PingFrame:
57．			t.handlePing(frame)
58．		case *http2.GoAwayFrame:
59．			t.handleGoAway(frame)
60．		case *http2.WindowUpdateFrame:
61．			t.handleWindowUpdate(frame)
62．		default:
63．			errorf("transport: http2Client.reader got unhandled frame type %v.", frame)
64．		}
65．	}
66．}

```

主要代码说明:

-   第4-8行：调用http2原生方法ReadFrame读取帧；并提供了异常处理逻辑。
-   第9行：首次收到grpc服务器端发送过来的设置帧，需要更新deadline时间
-   第10-12行：设置keeaplive时间
-   第13-17行：对接收到的帧，强制转换为SettingFrame
-   第16行：这是个回调；核心目的是，grpc客户端接收到grpc服务器端发送过来的信息，表明一次真正的HTTP2链接已经建立好了。
-   第17行：调用handleSettings方法，也就是在接收到服务器端的设置帧后，需要将具体设置更新到grpc客户端；
-   第20行：对controlBuf存储添加了一个节流阀，目的是：比方说，当controlBuf里存储着大量的Ping帧，incoming帧时，帧接收器需要进入阻塞状态，暂停接收服务器发送过来的帧。也就是说，第20行，会进入阻塞状态。
-   第21-45行：读取帧；并提供了异常处理逻辑；
-   第47-64行：根据接收到帧的类型，触发不同的处理逻辑；

刚才整体看了一下，帧接收器是如何工作的，接下来，针对重要执行语句进行详细的分析一下：

1 )第16行，这个回调函数onPrefaceReceipt，是在什么地方初始化，并进行的调用的？

grpc-go/clientconn.go文件中的createTransport方法里：

```go
1．func (ac *addrConn) createTransport(addr resolver.Address, copts transport.ConnectOptions, connectDeadline time.Time) (transport.ClientTransport, *grpcsync.Event, error) {
2．// -----省略本次分析不相关代码----

3．	onPrefaceReceipt := func() {
4．		close(prefaceReceived)
5．	}
6．	connectCtx, cancel := context.WithDeadline(ac.ctx, connectDeadline)
7．	defer cancel()
8．	if channelz.IsOn() {
9．		copts.ChannelzParentID = ac.channelzID
10．	}
11．	newTr, err := transport.NewClientTransport(connectCtx, ac.cc.ctx, addr, copts, onPrefaceReceipt, onGoAway, onClose)
12．	if err != nil {
13．		// newTr is either nil, or closed.
14．		channelz.Warningf(ac.channelzID, "grpc: addrConn.createTransport failed to connect to %v. Err: %v. Reconnecting...", addr, err)
15．		return nil, nil, err
16．	}
17．	select {
18．	case <-time.After(time.Until(connectDeadline)):
19．		// We didn't get the preface in time.
20．		newTr.Close()
21．		channelz.Warningf(ac.channelzID, "grpc: addrConn.createTransport failed to connect to %v: didn't receive server preface in time. Reconnecting...", addr)
22．		return nil, nil, errors.New("timed out waiting for server handshake")
23．	case <-prefaceReceived:
24．		// We got the preface - huzzah! things are good.
25．	case <-onCloseCalled:
26．		// The transport has already closed - noop.
27．		return nil, nil, errors.New("connection closed")
28．		// TODO(deklerk) this should bail on ac.ctx.Done(). Add a test and fix.
29．	}
30．	return newTr, reconnect, nil
31．}
```

主要代码说明：

-   第3-5行：创建一个匿名函数赋值给onPrefaceReceipt
-   第11行：作为参数，传递给transport.NewClientTransport使用
-   第23行：在这里阻塞着，直到帧接收器接收到服务器端发送过来的设置帧。执行t.onPrefaceReceipt()语句，其实执行的是close(prefaceReceived)，关闭prefaceReceived通道，就是给prefaceReceived通道发送消息；从而使得第23行，结束阻塞。

2 )第20行：t.controlBuf.throttle()，即帧缓存器阀门

grpc-go/internal/transport/controlbuf.go文件中的throttle方法里：

```go
1．func (c *controlBuffer) throttle() {
2．	ch, _ := c.trfChan.Load().(*chan struct{})

3．	if ch != nil {
4．		select {
5．		case <-*ch:
6．		case <-c.done
7．		}
8．	}
9．}
```

主要代码说明：

-   第2行：从trfChan取出值，并强制转换为通道类型；
-   第3-8行：如果通道ch存在的话，就进入多路复用器select中，提供了两个通道；如果两个通道里都没有数据的话，就会进入阻塞状态；从而使得第20行，进入了阻塞状态，即达到了帧接收器暂停接收帧的效果。

那么，什么情况下会使得第5行中的ch通道处于阻塞状态呢? 以及什么情况下会解除阻塞状态?

  
(备注：下面仅仅是我的猜测，并没有实现真正的模拟一次)  
可以通过下面的图，来进行分析：  
![帧缓存器阀门](https://img-blog.csdnimg.cn/20210612163258826.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

  首先说明一下，grpc服务器端向grpc客户端发送的设置帧，某些情况下，需要grpc客户端向grpc服务器端进行反馈的，如Ping帧，Setting帧等。  
  假设当grpc客户端中的帧接收器接收了大量的设置帧，或者Ping帧，经过各自的处理器进行处理后，某些情况下，需要向grpc服务器端进行反馈，需要创建大量的设置帧或者Ping帧，会调用 grpc-go/internal/transport/controlbuf.go文件中的Put方法或者executeAndPut方法，将帧存储到controlBuf里；

好，看一下executeAndPut方法：

```go
1．func (c *controlBuffer) executeAndPut(f func(it interface{}) bool, it cbItem) (bool, error) {
2．//-----省略了一些非相关性代码
3．	c.list.enqueue(it)

4．	if it.isTransportResponseFrame() {
5．		c.transportResponseFrames++

6．		if c.transportResponseFrames == maxQueuedTransportResponseFrames {
7．			// We are adding the frame that puts us over the threshold; create
8．			// a throttling channel.
9．			ch := make(chan struct{})
10．			c.trfChan.Store(&ch)
11．		}
12．	}

13．//-----省略了一些非相关性代码
14．}
```

主要代码说明：

-   第1行：it参数，就是要存储的各种帧
-   第3行：将帧存储到controlBuf结构体中的list里
-   第4-11行：当帧的isTransportResponseFrame为true时，就会累加transportResponseFrames值，当该值等于maxQueuedTransportResponseFrames 值时(默认是50)，就会创建通道ch，并将cn存储到controlBuf的trfChan里。这里对应的就是throttle方法中的第5行中的通道；执行到第10行，就表明已经往controlBuf里存储里大量的设置帧；

其实，

执行t.controlBuf.throttle()语句的根本原因可能是：

  帧发送器对帧的处理速度赶不上将帧存储到ControlBuf里的速度，从而导致controlBuf里存储着大量的设置帧，设置帧的数量等于设置的最大反馈设置帧的数量时，就会通知帧接收器暂停接收帧，等待帧发送器的处理结果。

接下来，分析一下，什么地方会解除通道ch的阻塞呢？

直接进入grpc-go/internal/transport/controlbuf.go文件中的Get方法：

```go
1．func (c *controlBuffer) get(block bool) (interface{}, error) {
2．	for {
3．	//-----省略了一些非相关性代码
4．		if !c.list.isEmpty() {
5．			h := c.list.dequeue().(cbItem)
6．			if h.isTransportResponseFrame() {
7．				if c.transportResponseFrames == maxQueuedTransportResponseFrames {
8．					// We are removing the frame that put us over the
9．					// threshold; close and clear the throttling channel.
10．					ch := c.trfChan.Load().(*chan struct{})
11．					close(*ch)
12．					c.trfChan.Store((*chan struct{})(nil))
13．				}
14．				c.transportResponseFrames--
15．			}
16．			c.mu.Unlock()
17．			return h, nil
18．		}
19．		//-----省略了一些非相关性代码
20．	}
21．}
```

帧发送器会调用Get方法从ControlBuf里获取帧，对帧进行处理后，会发送到grpc服务器端。  
主要代码说明:

-   第5行：从controlBuf的list里获取帧，并强制类型转换
-   第6-13行：判断帧的isTransportResponseFrame是否为true，其实就是判断该帧是不是反馈给服务器端的设置帧；并且controlBuf里存储的反馈给服务器端的设置帧的数量transportResponseFrames 等于maxQueuedTransportResponseFrames 时，就会将ch通道关闭，并且将transportResponseFrames 递减1，这样的话，throttle方法里的第5行的ch通道就会解除阻塞状态。从而，帧接收器可以继续接收帧了。

到此为止，grpc客户端的帧接收器的主体流程已经介绍完了，接下来，介绍grpc客户端帧接收器如何处理不同的帧的？

# 4、总结

  本小节我们通过帧接收器整体流程处理图，概况的介绍了grpc客户端帧接收器的原理；并对帧接收器的[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)进行深入分析；

  最核心的原理，其实就是调用http2中的frame结构体下的readFrame方法来读取各种类型的帧，并根据帧的类型，来调用不同的帧处理器，也就是处理不同的业务。

  并且分析了通过阀门原理来控制帧接收器的速度，防止超过后台处理帧的最大能力。

  阀门原理，经过整理后，应该可以运用到我们的项目中去；有点类似于内存级别的生产者和消费者的使用场景。
