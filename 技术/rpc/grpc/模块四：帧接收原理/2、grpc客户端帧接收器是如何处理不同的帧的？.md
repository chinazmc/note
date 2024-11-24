#grpc 

  上一小节，我们针对[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)客户端的帧接收器原理进行了介绍；接收到帧后，帧分发器会根据帧的类型将接收到的帧分发给不同的帧处理器进行处理；  
  目前，grpc客户端的帧接收器主要对7种类型的帧，进行处理：

-   头帧
-   数据帧
-   RST帧
-   设置帧
-   Ping帧
-   GoAway帧
-   窗口更新帧

接下来会对这些帧的处理流程依次说明。

# 1、源码分析入口
对帧的处理入口(调用入口)，都是在帧接收器里grpc-go/internal/transport/http2_client.go文件中的reader方法里：如下所示：
```go
1．func (t *http2Client) reader() {
2．     //-----省略了一些非相关性代码
3．		switch frame := frame.(type) {
4．		case *http2.MetaHeadersFrame:
5．			t.operateHeaders(frame)
6．		case *http2.DataFrame:
7．			t.handleData(frame)
8．		case *http2.RSTStreamFrame:
9．			t.handleRSTStream(frame)
10．	case *http2.SettingsFrame:
11．		t.handleSettings(frame, false)
12．	case *http2.PingFrame:
13．		t.handlePing(frame)
14．	case *http2.GoAwayFrame:
15．		t.handleGoAway(frame)
16．	case *http2.WindowUpdateFrame:
17．		t.handleWindowUpdate(frame)
18．	default:
19．		errorf("transport: http2Client.reader got unhandled frame type %v.", frame)
20．	}
21．}
```

# 2、头帧处理流程(http2.MetaHeadersFrame)
注意：  
头帧的处理流程调用了两次，即该方法operateHeaders执行了两次。
第一次：触发头帧处理器主要做了什么？
  客户端接收到服务器端发送过来的头帧，表明服务器已经全部接收完了客户端发送过去的数据，接下来，就可以处理客户端的请求了

第二次：触发头帧处理器主要做了什么？
  客户端接收到服务器端发送过来的头帧，表明服务器已经处理完客户端的请求了，并且已经将处理结果，全部发送给客户端了，服务器端给客户端发送最后一个状态，表示已经处理完成了。客户端此时可以关闭流了。

直接进入grpc-go/internal/transport/http2_client.go文件中的operateHeaders方法里：

```go
// operateHeaders takes action on the decoded headers.
1．func (t *http2Client) operateHeaders(frame *http2.MetaHeadersFrame) {
2．	s := t.getStream(frame)
3．	if s == nil {
4．		return
5．	}
6．	endStream := frame.StreamEnded()
7．	atomic.StoreUint32(&s.bytesReceived, 1)
8．	initialHeader := atomic.LoadUint32(&s.headerChanClosed) == 0

9．	if !initialHeader && !endStream {
10．		// As specified by gRPC over HTTP2, a HEADERS frame (and associated CONTINUATION frames) can only appear at the start or end of a stream. Therefore, second HEADERS frame must have EOS bit set.
11．		st := status.New(codes.Internal, "a HEADERS frame cannot appear in the middle of a stream")
12．		t.closeStream(s, st.Err(), true, http2.ErrCodeProtocol, st, nil, false)
13．		return
14．	}

15．	state := &decodeState{}
16．	// Initialize isGRPC value to be !initialHeader, since if a gRPC Response-Headers has already been received, then it means that the peer is speaking gRPC and we are in gRPC mode.
17．	state.data.isGRPC = !initialHeader
18．	if err := state.decodeHeader(frame); err != nil {
19．		t.closeStream(s, err, true, http2.ErrCodeProtocol, status.Convert(err), nil, endStream)
20．		return
21．	}

22．	isHeader := false
23．	defer func() {
24．		if t.statsHandler != nil {
25．			if isHeader {
26．				inHeader := &stats.InHeader{
27．					Client:      true,
28．					WireLength:  int(frame.Header().Length),
29．					Header:      s.header.Copy(),
30．					Compression: s.recvCompress,
31．				}
32．				t.statsHandler.HandleRPC(s.ctx, inHeader)
33．			} else {
34．				inTrailer := &stats.InTrailer{
35．					Client:     true,
36．					WireLength: int(frame.Header().Length),
37．					Trailer:    s.trailer.Copy(),
38．				}
39．				t.statsHandler.HandleRPC(s.ctx, inTrailer)
40．			}
41．		}
42．	}()

43．	// If headerChan hasn't been closed yet
44．	if atomic.CompareAndSwapUint32(&s.headerChanClosed, 0, 1) {
45．		s.headerValid = true
46．		if !endStream {
47．			// HEADERS frame block carries a Response-Headers.
48．			isHeader = true
49．			// These values can be set without any synchronization because
50．			// stream goroutine will read it only after seeing a closed
51．			// headerChan which we'll close after setting this.
52．			s.recvCompress = state.data.encoding
53．			if len(state.data.mdata) > 0 {
54．				s.header = state.data.mdata
55．			}
56．		} else {
57．			// HEADERS frame block carries a Trailers-Only.
58．			s.noHeaders = true
59．		}
60．		
61．		close(s.headerChan)
62．	}
63．	
64．	if !endStream {
65．		return
66．	}

67．	// if client received END_STREAM from server while stream was still active, send RST_STREAM
68．	rst := s.getState() == streamActive
69．	t.closeStream(s, io.EOF, rst, http2.ErrCodeNo, state.status(), state.data.mdata, true)
70．}
```

该方法的核心，其实就是解析头帧，从头帧里解析协议字段，从协议字段获取相应的信息，如grpc-encoding，从而帮助客户端处理接收到的数据，比方说，要不要解压等。  
主要代码说明：

-   第2-5行：根据帧来获取对应的流；底层是通过t.activeStreams[f.StreamID]来实现的
-   第6行：表明该流是处于运行状态，还是即将结束。
    -   a)endStream=false，表示该流没有结束
    -   b)endStream=true，表示服务端已经完全处理了，不再发送任何帧了。
    -   c)在整个运行期间，operateHeaders方法，会调用两次；一次是刚开始接收服务器端发送的头帧，一次是服务器端已经处理完成了，告诉客户端。
    -   d)grpc服务器端分别在什么地方发送了两次头帧？在grpc-go/server.go文件中的processUnaryRPC方法里
        -   第一次：grpc服务器端已经处理完成客户端的请求了，开始将处理结果返回给客户端，使用语句”if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {” ,其中s.sendResponse方法的底层向客户端发送了头帧
        -   第二次：等grpc服务器端已经完成发送时，需要给客户端发送一个状态标志，是通过头帧来发送的，语句”err = t.WriteStatus(stream, statusOK)”，底层向grpc客户端发送了头帧。
-   第8行：headerChanClosed表示头帧通道是否关闭；headerChanClosed=0，表示没有关闭，headerChanClosed=1表示已经关闭了
-   第9-14行：主要是想表达，头帧不能出现在中间位置，可以在开头，也可以在结尾；如果出现在中间，就表示出问题了。正常情况下，endStream为false时，initialHeader为true；endStream为true时，initialHeader为false
-   第18行：调decodeHeader方法，底层其实调用的是processHeaderField方法，专门用来处理服务器端发送的头帧中的协议字段，如context-type, grpc-encoding, grpc-status,grpc-timeout,status等，这些信息是用来协助客户端接收服务器的数据的。
-   第44-62行：如果头帧通道headerChan没有关闭，即headerChanClosed为0时，并且流没有结束，即endStream为false时，可以从头帧里获取相应的信息。获取完后，调用close(s.headerChan)关闭头帧通道。
    -   头帧通道headerChan是在什么地方阻塞了呢？  
        在grpc-go/internal/transport/transport.go文件中的waitOnHeader方法里，有提供了阻塞场景，如case <- s.headerChan。至于谁调用的waitOnHeader方法，情况比较多了，如客户端接收服务器端的数据时，调用的RecvCompress方法。
    -   那么，头帧通道headerChan是在什么场景下解除阻塞了呢？  
        如，第一次接收到服务器发送过来的头帧后，对头帧里的协议字段进行了解析，就可以关闭头帧通道了，如第61行，调用close(s.headerChan)关闭头帧通道
-   第67-69行：只要当endStream为true，才会执行closeStream方法。

# 3、数据帧处理流程(http2.DateFrame)

![数据帧处理流程](https://img-blog.csdnimg.cn/20210612174421745.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

主要流程是：

-   当帧接收器接收到帧后，根据帧的类型，若是数据帧，就交于数据帧处理器处理
-   数据帧处理器最终的目的是将接收到数据，缓存到Stream中的buf缓存里
-   grpc框架内部读取数据时，可以通过go语言原生的read方法去读取数据

进入grpc-go/internal/transport/http2_client.go文件中的handleData方法里：

```go
1．func (t *http2Client) handleData(f *http2.DataFrame) {
2．	size := f.Header().Length
3．	var sendBDPPing bool
4．	if t.bdpEst != nil {
5．		sendBDPPing = t.bdpEst.add(size)
6．	}
7．	//---省略不相关代码
8．	if w := t.fc.onData(size); w > 0 {
9．		t.controlBuf.put(&outgoingWindowUpdate{
10．			streamID:  0,
11．			increment: w,
12．		})
13．	}
14．	if sendBDPPing {
15．		// Avoid excessive ping detection (e.g. in an L7 proxy)
16．		// by sending a window update prior to the BDP ping.

17．		if w := t.fc.reset(); w > 0 {
18．			t.controlBuf.put(&outgoingWindowUpdate{
19．				streamID:  0,
20．				increment: w,
21．			})
22．		}

23．		t.controlBuf.put(bdpPing)
24．	}
25．	// Select the right stream to dispatch.
26．	s := t.getStream(f)
27．	if s == nil {
28．		return
29．	}
30．	if size > 0 {
31．		if err := s.fc.onData(size); err != nil {
32．			t.closeStream(s, io.EOF, true, http2.ErrCodeFlowControl, status.New(codes.Internal, err.Error()), nil, false)
33．			return
34．		}
35．		if f.Header().Flags.Has(http2.FlagDataPadded) {
36．			if w := s.fc.onRead(size - uint32(len(f.Data()))); w > 0 {
37．				t.controlBuf.put(&outgoingWindowUpdate{s.id, w})
38．			}
39．		}
40．		if len(f.Data()) > 0 {
41．			buffer := t.bufferPool.get()
42．			buffer.Reset()
43．			buffer.Write(f.Data())

44．			s.write(recvMsg{buffer: buffer})
45．		}
46．	}
47．	if f.FrameHeader.Flags.Has(http2.FlagDataEndStream) {
48．		t.closeStream(s, io.EOF, false, http2.ErrCodeNo, status.New(codes.Internal, "server closed the stream without sending trailers"), nil, true)
49．	}
50．}
```

主要代码说明：

-   第2-39行：这些代码跟滑动窗口有关，在滑动窗口章节介绍。
-   第41行：获取一个缓存池，底层使用sync.Pool实现的。就是一个简单的包装
-   第43行：将数据帧里的数据f.Data()写入buffer缓存里
-   第54行：将数据写入到流Stream的buf里；

到此为止，帧接收器将接收到的数据帧的数据缓存到内存里(流自己的缓存buf)，其他程序可以从流里获取客户端发送的数据帧。

# 4、RST帧处理流程(http2.RSTStreamFrame)

进入grpc-go/internal/transport/http2_client.go文件中的handleRSTStream方法里：

```go
1．func (t *http2Client) handleRSTStream(f *http2.RSTStreamFrame) {
2．	s := t.getStream(f)
3．	if s == nil {
4．		return
5．	}
6．	if f.ErrCode == http2.ErrCodeRefusedStream {
7．		// The stream was unprocessed by the server.
8．		atomic.StoreUint32(&s.unprocessed, 1)
9．	}
10．	statusCode, ok := http2ErrConvTab[f.ErrCode]
11．	if !ok {
12．		warningf("transport: http2Client.handleRSTStream found no mapped gRPC status for the received http2 error %v", f.ErrCode)
13．		statusCode = codes.Unknown
14．	}
15．	if statusCode == codes.Canceled {
16．		if d, ok := s.ctx.Deadline(); ok && !d.After(time.Now()) {
17．			// Our deadline was already exceeded, and that was likely the cause
18．			// of this cancelation.  Alter the status code accordingly.
19．			statusCode = codes.DeadlineExceeded
20．		}
21．	}
22．	t.closeStream(s, io.EOF, false, http2.ErrCodeNo, status.Newf(statusCode, "stream terminated by RST_STREAM with error code: %v", f.ErrCode), nil, false)
23．}
```

主要代码说明:

-   第2-5行：通过帧从http2Client获取对应的流
-   第6-9行：客户端接收到的帧的错误码如果是ErrCodeRefusedStream，说明grpc服务器端并没有处理此流，拒绝了。客户端这一侧需要将unprocessed设置为1，以表示此流被拒绝。被拒绝的原因有很多，在其他章节介绍。
-   第10-21行：核心就是为了得到状态码statusCode；其实，就是得到grpc服务器端反馈的状态码，以表明grpc服务器端为什么拒绝此流。
    -   第10行：将http2状态码转换成grpc框架内定义的状态码
    -   第15行：判断状态码是否是codes.Canceled
    -   第16行：从流中获取Deadline时间，判断deadline时间是否在当前时间time.Now后
    -   第17行：执行到这行，表明deadline时间在当前时间time.Now前，已经超时了。因此，设置状态码statusCode= codes.DeadlineExceeded
-   第22行：关闭流

也就是说，客户端在一旦接收到服务器端的RST帧，客户端就需要关闭相应的流。

# 5、设置帧处理流程(http2.SettingsFrame)

设置帧的使用场景是:

-   在底层连接建立后，需要做的一些初始化设置，如服务器端需要通知客户端，自己能够处理的最大并发流是多少，客户端需要设置SettingMaxConcurrentStreams参数等；
-   在数据传输过程中，需要做的一些更新设置；如：服务器端需要更新客户端的帧发送器的窗口大小时，就可以设置SettingInitialWindowSize，grpc-go/internal/transport/controlbuf.go文件中的applySetting方法里有，如l.oiws = s.Val，将窗口大小更新到l.oiws里。
-   服务器端对客户端发送的设置帧的确认时，需要一个ACK

grpc-go/internal/transport/http2_client.go文件中的handleSettings方法里:

```go
1．func (t *http2Client) handleSettings(f *http2.SettingsFrame, isFirst bool) {
2．	if f.IsAck() {
3．		return
4．	}
5．	var maxStreams *uint32
6．	var ss []http2.Setting
7．	var updateFuncs []func()
8．	f.ForeachSetting(func(s http2.Setting) error {
9．		switch s.ID {
10．		case http2.SettingMaxConcurrentStreams:
11．			maxStreams = new(uint32)
12．			*maxStreams = s.Val
13．		case http2.SettingMaxHeaderListSize:
14．			updateFuncs = append(updateFuncs, func() {
15．				t.maxSendHeaderListSize = new(uint32)
16．				*t.maxSendHeaderListSize = s.Val
17．			})
18．		default:
19．			ss = append(ss, s)
20．		}
21．		return nil
22．	})
23．	
24．	if isFirst && maxStreams == nil {
25．		maxStreams = new(uint32)
26．		*maxStreams = math.MaxUint32
27．	}
28．	
29．	sf := &incomingSettings{
30．		ss: ss,
31．	}
32．	if maxStreams != nil {
33．		updateStreamQuota := func() {
34．			delta := int64(*maxStreams) - int64(t.maxConcurrentStreams)
35．			t.maxConcurrentStreams = *maxStreams
36．			t.streamQuota += delta
37．			
38．			if delta > 0 && t.waitingStreams > 0 {
39．				close(t.streamsQuotaAvailable) // wake all of them up.
40．				t.streamsQuotaAvailable = make(chan struct{}, 1)
41．			}
42．		}
43．		updateFuncs = append(updateFuncs, updateStreamQuota)
44．	}

45．	t.controlBuf.executeAndPut(func(interface{}) bool {
46．		for _, f := range updateFuncs {
47．			f()
48．		}
49．		return true
50．	}, sf)
51．}
```

主要代码说明：

-   第2-4行：判断帧是否是ACK帧；如果是的话，就表明客户端给服务器端发送的设置帧，服务器端已经处理完毕了，已经将设置更新到服务器端的进程里了，而且更新成功；因此，给客户端一个反馈信息。
-   第10-12行：服务器端通知客户端，自己最大能够处理的并发流是多少；
-   第13-16行：服务器端通知客户端，头帧里面的所有协议字段名称以及对应的值的累加和长度不能超过maxSendHeaderListSize ；(描述的不是很准确，主要目的是对协议字段的限制)
-   第19行：其他参数设置，直接追加到ss里
-   第24-27行：如果是第一次进行设置，并且服务器端并没有设置最大并发流时，客户端默认服务器端能够处理的最大并发流是math.MaxUint32
-   第29-31行：构建一个incomingSettings结构体
-   第33-42行：如果服务器端设置了最大并发流的话，那么客户端需要作出相应的调整，如设置maxConcurrentStreams以及streamQuota值；
    -   对第38-41行，感到疑惑？疑惑的前提是，最大并发参数只是在链接建立后，进行设置的，在数据传输阶段是不会进行设置的，也就是说，在整个过程中，只会设置一次，那么此时流还没有创建，就不会出现waitingStreams > 0 的情况，因此也就不会执行第38-41行了。
    -   如果也可以在传输阶段进行最大并发流的更新的话，可能会执行第38-41行了，但是一直没有找到证据，也没有测试出来。
-   第45-50行：将incomingSettings帧存储到controlBuf里，帧发送器会进行处理，将设置参数更新到本地进程里。

# 6、Ping帧处理流程(http2.PingFrame)

grpc-go/internal/transport/http2_client.go文件中的handlePing方法里:

```go
1．func (t *http2Client) handlePing(f *http2.PingFrame) {
2．	if f.IsAck() {
3．		// Maybe it's a BDP ping.
4．		if t.bdpEst != nil {
5．			t.bdpEst.calculate(f.Data)
6．		}
7．		return
8．	}

9．	pingAck := &ping{ack: true}
10．	copy(pingAck.data[:], f.Data[:])
11．	t.controlBuf.put(pingAck)
12．}
```

主要代码说明：

-   第2-8行：当客户端给服务器端发送ping帧后，服务器端接收到ping帧后，需要给客户端一个ack反馈，就会执行第2-8行
    -   第5行：方法calculate，主要是用来计算当前的BDP，以及抽样带宽，以及决定是否更新窗口大小
-   第9-11行：构建一个Ping帧，并且ack设置为ture，存储到controlBuf里，帧发送器会将此帧发送到grpc服务器端

# 7、GoAway帧处理流程(http2.GoAwayFrame)

什么情况下，客户端会接收到服务器端发送的GoAway帧呢？  
主要分三种情况：

-   开启了keepalive功能的情况下，客户端发送ping的次数超过了keepalive规定的次数时，服务器端会发送goaway帧，并且会关闭此链接
-   开启了keepalive功能的情况下，链接处于IDLE的时间，超过了keepalive规定的最大IDLE时间参数MaxConnectionIdle，服务器端会发送goaway帧
-   开启了keepalive功能的情况下，keepalive规定，任何链接都不能超过一定的时间，假设keepalive规定，链接的存活时间是1分钟，keepalive会将超过一分钟的链接关闭，不管有没有执行完成。

grpc-go/internal/transport/http2_client.go文件中的handleGoAway方法里:

```go
1．func (t *http2Client) handleGoAway(f *http2.GoAwayFrame) {
2．	t.mu.Lock()
3．	if t.state == closing {
4．		t.mu.Unlock()
5．		return
6．	}
7．	if f.ErrCode == http2.ErrCodeEnhanceYourCalm {
8．		infof("Client received GoAway with http2.ErrCodeEnhanceYourCalm.")
9．	}
10．	id := f.LastStreamID
11．	if id > 0 && id%2 != 1 {
12．		t.mu.Unlock()
13．		t.Close()
14．		return
15．	}
16．	// A client can receive multiple GoAways from the server (see
17．	// https://github.com/grpc/grpc-go/issues/1387).  The idea is that the first
18．	// GoAway will be sent with an ID of MaxInt32 and the second GoAway will be
19．	// sent after an RTT delay with the ID of the last stream the server will
20．	// process.
21．	//
22．	// Therefore, when we get the first GoAway we don't necessarily close any
23．	// streams. While in case of second GoAway we close all streams created after
24．	// the GoAwayId. This way streams that were in-flight while the GoAway from
25．	// server was being sent don't get killed.
26．	select {
27．	case <-t.goAway: // t.goAway has been closed (i.e.,multiple GoAways).
28．		// If there are multiple GoAways the first one should always have an ID greater than the following ones.
29．		if id > t.prevGoAwayID {
30．			t.mu.Unlock()
31．			t.Close()
32．			return
33．		}
34．	default:
35．		t.setGoAwayReason(f)
36．		close(t.goAway)
37．		t.controlBuf.put(&incomingGoAway{})
38．		// Notify the clientconn about the GOAWAY before we set the state to
39．		// draining, to allow the client to stop attempting to create streams
40．		// before disallowing new streams on this connection.
41．		// clientconn.go文件中的createTransport方法里，设置了onGoAway方法
42．		t.onGoAway(t.goAwayReason)
43．		t.state = draining
44．	}
45．	// All streams with IDs greater than the GoAwayId
46．	// and smaller than the previous GoAway ID should be killed.
47．	upperLimit := t.prevGoAwayID
48．	if upperLimit == 0 { // This is the first GoAway Frame.
49．		upperLimit = math.MaxUint32 // Kill all streams after the GoAway ID.
50．	}
51．	for streamID, stream := range t.activeStreams {
52．		if streamID > id && streamID <= upperLimit {
53．			// The stream was unprocessed by the server.
54．			atomic.StoreUint32(&stream.unprocessed, 1)
55．			t.closeStream(stream, errStreamDrain, false, http2.ErrCodeNo, statusGoAway, nil, false)
56．		}
57．	}
58．	t.prevGoAwayID = id
59．	active := len(t.activeStreams)
60．	t.mu.Unlock()
61．	if active == 0 {
62．		t.Close()
63．	}
64．}
```

主要代码说明：

-   第3-6行：如果http2Client的状态已经是closing时，对最新接收到的goAway帧，不处理。
-   第10-15行：如果服务器端最新处理的流ID不是奇数的话，就调用http2Client的close方法，关闭连接
-   第26-44行：多路复用器。提供了默认选择。
    -   a)第35行：主要是设置http2Client中的属性goAwayReason
    -   b)第36行：关闭t.goAway通道
    -   c)第37行：构建incomingGoAway帧，交由帧发送器处理，主要是更新了帧发送器的属性draing为true，表明连接正在关闭
    -   d)第42行：更新了平衡器的状态为connecting，更新了picker里的error为ErrNoSubConnAvailable，这样的话，在创建流时，就不能选择Picker，也就不能创建客户端流了，就更不能创建流了; 从而实现了，让客户端停止正在创建的流；
    -   e)第43行：更新http2Client的状态，阻止了头帧发送到服务器端，同时阻止了新创建的流注册到帧发送器里；(grpc-go/internal/transport/http2_client.go文件中的NewStream方法里，创建headerFrame结构体中，初始化initStreams时，有对htt2Client状态的校验；而initStreams函数是在controlBuf.go文件中的originateStream方法里调用的，调用时发现http2Client状态不符合要求，就不会注册新的流了。)
-   第47-50行：就是为了确定upperLimit值
-   第51-57行：将符合条件的流，关闭掉。

# 8、窗口更新帧处理流程(http2.WindowUpdateFrame)

grpc-go/internal/transport/http2_client.go文件中的handleWindowUpdate方法里:

```go
func (t *http2Client) handleWindowUpdate(f *http2.WindowUpdateFrame) {
	t.controlBuf.put(&incomingWindowUpdate{
		streamID:  f.Header().StreamID,
		increment: f.Increment,
	})
}
```

将接收到的窗口更新帧WindowUpdateFrame，转换为grpc框架内的incomingWindowUpdate帧，存储到controlBuf里，交由帧发送器处理。

# 9、总结

  本小节，我们主要对不同类型的帧处理器进行了[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)剖析；是从单个功能角度进行的介绍，可能理解起来不如从业务的角度分析好；不过，没事，对这些有个印象就行；在做测试，或者其他章节涉及到某个帧处理器时，再参考即可。