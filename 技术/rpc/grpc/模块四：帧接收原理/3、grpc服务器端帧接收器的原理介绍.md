#grpc 

# 1、帧接收器的入口？

随便找一个grpc服务器端的启动文件，如main.go：

```go
1．func main() {
2．	lis, err := net.Listen("tcp", port)
	//-------省略不相关代码------
3．	s := grpc.NewServer(grpc.MaxConcurrentStreams(2))
4．	pb.RegisterGreeterServer(s, &server{})

5．	if err := s.Serve(lis); err != nil {
6．		log.Errorf("failed to serve: %v", err)
7．	}
8．}
```

直接点击第5行中的Serve方法，进入grpc-go/server.go文件中Serve方法里：

```go
1．func (s *Server) Serve(lis net.Listener) error {
2．    //-------省略不相关代码------
3．	ls := &listenSocket{Listener: lis}

9．	for {
10．		rawConn, err := lis.Accept()

11．      //-------省略不相关代码------

12．		// 一个请求，对应一个goroutine； 一个处理请求对应一个WG
13．		go func() {
14．			s.handleRawConn(rawConn)
15．			s.serveWG.Done()
16．		}()
17．	}
18．}
```

第13-16行：服务器端针对每一个请求，都会异步处理  
点击第14行中的handleRawConn方法，进入grpc-go/server.go文件中handleRawConn方法里：

```go
// handleRawConn forks a goroutine to handle a just-accepted connection that
// has not had any I/O performed on it yet.
1．func (s *Server) handleRawConn(rawConn net.Conn) {
2．   //-------省略不相关代码------

3．   // Finish handshaking (HTTP2)
4．	st := s.newHTTP2Transport(conn, authInfo)

5．   //-------省略不相关代码------

6．	go func() {
7．		s.serveStreams(st)
8．		s.removeConn(st)
9．	}()
```
主要代码说明：
-   第4行：底层跟grpc客户端建立起链接，创建一个帧发送器；
-   第6-9行：又异步处理流  
    点击serveStreams，直接进入grpc-go/server.go文件中serveStreams方法里：

```go
1．func (s *Server) serveStreams(st transport.ServerTransport) {
2．	//-------省略不相关代码------
3．	st.HandleStreams(func(stream *transport.Stream) {
4．       //-------省略不相关代码------
5．   })
6．	
7．	wg.Wait()
8．}
```

其中，第3行HandleStreams方法，就是grpc服务器端的帧接收器。  
至此，grpc服务器端会为每一个请求，创建一个协程，异步处理。为每一个请求，创建一个http2Server，一个帧发送器，一个帧接收器。  
在帧接收器创建前，已经跟grpc客户端建立起了底层连接。  

# 2、帧接收器的原理？
HandleStreams方法是ServerTransport接口里的方法，grpc框架中由http2Server结构体实现了ServerTransport接口；  
因此，进入grpc-go/internal/transport/http2_server.go文件中的HandleStreams方法里：

```go
// HandleStreams receives incoming streams using the given handler. This is
// typically run in a separate goroutine.
// traceCtx attaches trace to ctx and returns the new context.
1．func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
2．	defer close(t.readerDone)

3．	for {
4．		t.controlBuf.throttle()
5．		frame, err := t.framer.fr.ReadFrame()
6．		atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
7．	    //-------省略---读取帧异常时的处理逻辑-----

8．		switch frame := frame.(type) {
9．		case *http2.MetaHeadersFrame:
10．			if t.operateHeaders(frame, handle, traceCtx) {
11．				t.Close()
12．				break
13．			}
14．		case *http2.DataFrame:
15．			t.handleData(frame)
16．		case *http2.RSTStreamFrame:
17．			t.handleRSTStream(frame)
18．		case *http2.SettingsFrame:
19．			t.handleSettings(frame)
20．		case *http2.PingFrame:
21．			t.handlePing(frame)
22．		case *http2.WindowUpdateFrame:
23．			t.handleWindowUpdate(frame)
24．		case *http2.GoAwayFrame:
25．			// TODO: Handle GoAway from the client appropriately.
26．		default:
27．			errorf("transport: http2Server.HandleStreams found unhandled frame type %v.", frame)
28．		}
29．	}
30．}
```

我们会发现grpc服务器端的帧接收器原理，跟grpc客户端的帧接收器原理非常相似，  
通过死循环的方式接收器帧，根据帧的类型，适配到不同的帧处理器中。

至于其中涉及到帧处理器，在以后的章节中，涉及到时再介绍。  

# 3、服务器端接收到客户端的头帧后，如何处理？
## 3.1、服务器端头帧处理流程介绍

如图所示：  
![在这里插入图片描述](https://img-blog.csdnimg.cn/21a51b6af3e5477bbcd7b66b91161446.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

主要流程如下：

1 )、客户端一侧：

-   创建头帧
-   将头帧存储到帧缓存controlBuf里
-   帧发送器将头帧发送器出去

2 )、服务器端一侧：

-   帧接收器：
    -   a)帧接收器接收到头帧，
    -   b)帧分发器根据头帧的类型，将头帧分发给头帧处理器
    -   c)头帧处理器：
        -   对头帧进行解析，获取到协议信息，如客户端请求方法的名称，请求类型，编码类型，超时时间等信息
        -   构建流，并将协议字段信息初始化流的属性里；此时，流里已经存储了客户端请求方法的名称，请求类型，编码类型等信息了
        -   将流注册到http2Server前需要做一些校验工作：
            -   校验http2Server的状态是否符合要求，不符合要求时，拒绝将流注册进来，也就是拒绝了客户端的请求
            -   校验传进来的streamID是否符合要求，是不是大于当前http2Server维护的最大流ID
            -   校验http2Server中的维护的流的数量是否超过了服务器端最大处理流的并发数
        -   将新创建的流注册到http2Server中的activeStreams里进行维护
        -   创建registerStream，调用帧存储器put，存储到帧缓存controlBuf里
-   帧发送器：
    -   帧加载器从帧缓存里接收到registerStream后，
    -   帧分发器根据registerStream类型，转发给registerStream处理器处理
    -   registerStream处理器处理:
        -   创建outStream，存储了流ID，写流控wq等
        -   将新创建的outStream注册到帧发送器中，即l.estdStreams[streamID] = outStream  
             

## 3.2、服务器端头帧处理流程operateHeaders源码分析

接下来，先分析一下头帧处理器的原理：  
进入grpc-go/internal/transport/http2_server.go文件中的operateHeaders方法里：

```go
// operateHeader takes action on the decoded headers.
1．func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
2．	streamID := frame.Header().StreamID
3．	state := &decodeState{
4．		serverSide: true,
5．	}
6．	if err := state.decodeHeader(frame); err != nil {
7．		if se, ok := status.FromError(err); ok {
8．			t.controlBuf.put(&cleanupStream{
9．				streamID: streamID,
10．				rst:      true,
11．				rstCode:  statusCodeConvTab[se.Code()],
12．				onWrite:  func() {},
13．			})
14．		}
15．		return false
16．	}

17．	buf := newRecvBuffer()
18．	s := &Stream{
19．		id:             streamID,
20．		st:             t,
21．		buf:            buf,
22．		fc:             &inFlow{limit: uint32(t.initialWindowSize)},
23．		recvCompress:   state.data.encoding,
24．		method:         state.data.method,
25．		contentSubtype: state.data.contentSubtype,
26．	}

27．	// ----省略掉---一些初始化代码----

28．	t.mu.Lock()
29．	
30．	if t.state != reachable {
31．		t.mu.Unlock()
32．		s.cancel()
33．		return false
34．	}
35．	if uint32(len(t.activeStreams)) >= t.maxStreams {
36．		t.mu.Unlock()
37．		t.controlBuf.put(&cleanupStream{
38．			streamID: streamID,
39．			rst:      true,
40．			rstCode:  http2.ErrCodeRefusedStream,
41．			onWrite:  func() {},
42．		})
43．		s.cancel()
44．		return false
45．	}
46．	if streamID%2 != 1 || streamID <= t.maxStreamID {
47．		t.mu.Unlock()
48．		// illegal gRPC stream id.
49．		errorf("transport: http2Server.HandleStreams received an illegal stream id: %v", streamID)
50．		s.cancel()
51．		return true
52．	}
53．	t.maxStreamID = streamID
54．	t.activeStreams[streamID] = s
55．	if len(t.activeStreams) == 1 {
56．		t.idle = time.Time{}
57．	}
58．	t.mu.Unlock()
59．	
60．	// ----省略掉---一些初始化代码----
61．	// Register the stream with loopy.
62．	t.controlBuf.put(&registerStream{
63．		streamID: s.id,
64．		wq:       s.wq,
65．	})

66．	handle(s)
67．	return false
68．}
```

主要代码说明：

-   第2行：从头帧里获取流ID
-   第3-5行：构建一个decodeState结构体
-   第6-16行：调用decodeHeader方法，解析头帧里的协议字段，如context-type, grpc-encoding, grpc-status,grpc-timeout,status等，这些信息是用来协助服务器端接收客户端的数据的。
-   第17行：帧接收器接收到数据后，需要将数据存储到内存里，buf就是用来存储数据帧里的数据的，以供read方法读取数据。
-   第18-26行：创建Stream；并初始化了StreamID，这样的话，客户端的流ID就传递到了服务器端，两端使用相同的流ID。并初始化了method属性，该属性存储了客户端请求服务器端的哪个服务，如/helloworld.Greeter/SayHello
-   第30-52行：将新创建的流或者说接收客户端流注册到grpc服务器端前，从三个方面进行了校验，满足条件即可注册，否则拒绝接受新创建的流，也就是拒绝接受客户端的请求。
    -   a)第30-34行：校验http2Server的状态是否满足要求，不满足要求的话，就会执行，不会将新创建的流注册到grpc服务器端，即不会注册到http2Server里
    -   b)第35-45行：服务器端维护流的数量是有限制的，或者说服务器端能够处理客户端请求的数量是有限制的，不能超过t.maxStreams，超过后的请求，是拒绝的。
    -   c)第46-52行：对流ID进行校验，
        -   i.不能是偶数；
        -   ii.因为流的ID号是不断累加的，因此最新请求的ID号，是不能小于服务器端已经存在的ID号的
-   第53行: 更新服务器端maxStreamID的值
-   第54行：将新创建的流注册到grpc服务器端
-   第62-65行：创建一个registerStream，存储到controlBuf里，交由帧发送器处理，帧发送器会将registerStream结构体，转换为outStream结构体，然后将outStream注册到帧发送器里l.estdStreams[h.streamID] = str。
-   第66行：再其他章节，分析handle。完成的功能由：处理客户端的请求，如SayHello，将请求结果发送给客户端等都是有handle完成的。  
     

## 3.3、服务器端是如何知道客户端请求的哪个服务，哪个方法的，超时时间设置等信息的？

客户端将请求服务方法的名称，请求类型，超时时间等信息存储在头帧里传输给服务器端；  
流ID是从原生http2的Frame帧里获取的。  
可以接着上一小节中，继续分析第6行：decodeHeader方法：  
grpc-go/internal/transport/http_util.go文件中的decodeHeader方法里：

```go
1．func (d *decodeState) decodeHeader(frame *http2.MetaHeadersFrame) error {
2．	// ----省略不相关代码
3．	for _, hf := range frame.Fields {
4．		d.processHeaderField(hf)
5．	}
6．   // ----省略不相关代码
```

主要流程说明：

-   第3行：对frame.Fields进行遍历，Fields存储的就是各自协议字段，如content-type，grpc-encoding，grpc-message，grpc-timeout，:path，:status等等
-   第4行：对协议字段进行处理，主要是根据接收到的协议字段，对类型为decodeState的变量d进行相应的初始化。

进入processHeaderField方法里：

```go
1．func (d *decodeState) processHeaderField(f hpack.HeaderField) {
2．	switch f.Name {
3．	case "content-type":
4．		contentSubtype, validContentType := contentSubtype(f.Value)
5．		d.data.contentSubtype = contentSubtype
6．		d.addMetadata(f.Name, f.Value)
7．		d.data.isGRPC = true
8．	case "grpc-encoding":
9．		d.data.encoding = f.Value
10．	case "grpc-timeout":
11．		d.data.timeoutSet = true
12．		var err error
13．		d.data.timeout, err = decodeTimeout(f.Value);
14．	case ":path":
15．		d.data.method = f.Value

16．// ----省略掉---类似代码---
```

主要将协议字段转存到类型为decodeState的变量d里；

可以从下图中，了解一下协议字段的流转趋势：  
![grpc头帧协议字段](https://img-blog.csdnimg.cn/20210612190301821.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

1 ）客户端一侧：

-   客户端创建好的协议字段存储在头帧里
-   将头帧交由帧发送器发送出去

2 ）服务器端一侧：

-   帧接收器接收到头帧后，交由头帧处理器处理
-   对头帧中的协议字段进行解析
-   将协议字段先存储到decodeState结构体里，然后再转存到Stream中的属性里。

协议字段→decodeState→Stream  
   
 

# 4、总结

  本小节我们重点介绍了grpc服务器的帧接收器原理；提供了整体流程处理图；grpc服务器端的帧接收器原理跟grpc客户端的帧接收器原理很类似；我们重点分析了grpc服务器端是如何处理头帧的，并且分析了客户端存储的数据信息，经过rpc传输后，最终存储在grpc服务器端的流里；

  至于，其他帧处理器，本小节就不在写了，以后的分享中，如果用到再分析。  