
#grpc

[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)客户端如果想访问远程grpc服务器端的某个方法的话，首先得有一个基本的链接吧，有了链接，才能进行数据的传输；
因此，本篇文章主要是分享一下，[rpc](https://so.csdn.net/so/search?q=rpc&spm=1001.2101.3001.7020)链接是如何建立起来的；这里的链接包括底层tcp链路连接以及http2帧的设置。

# 1、grpc客户端跟grpc服务器端链接建立过程流程图？

![grpc客户端跟grpc服务器端链接建立过程流程图](https://img-blog.csdnimg.cn/2021051111213323.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

## 1.1、服务器端一侧，tcp链接前要做的事情？
-   启动grpc服务器时，主要做了一些初始化设置，如拦截器的设置、加密认证设置等
-   将提供的服务，如SayHello注册到grpc服务器里
-   grpc服务器端启动监听端口，监听grpc客户端发起的链接请求；如果没有请求，就会一直阻塞着。

## 1.2、在客户端一侧，tcp链接前要做的事情？
-   启动grpc客户端时，主要做了一些初始化设置，如设置服务链接地址，拦截器设置，是否是阻塞式链接，链接安全性设置，加密认证设置等
-   解析器的最终目的是，根据设置的服务链接地址，对此地址进行解析，最终获取到该服务后端对应的地址列表；在grpc框架中，解析器采用的是插件式设置，用户可以自己定义解析器；在解析器章节，也提供了相应的参考例子。
-   平衡器的最终目的是，拿到解析器获取到的后端服务链接地址，从启动时注册的平衡器中获取链接策略，开始触发tcp链接。在grpc框架，平衡器的设计同样是以插件形式实现的。用户可以自己定义平衡器；同样，平衡器章节提供了参考样例。

## 1.3、客户端如何向服务器发起真正的tcp链接？
-   客户端一侧，调用golang原生自带的net包中的Dialer对象向grpc服务器端发起链接请求
-   grpc服务器端接收到grpc客户端发起的链接请求为其专门创建一个协程来处理此请求；也就是说，在服务器端一侧，客户端的一次请求，服务器端都为其单独创建一个协程来处理。

## 1.4、帧设置阶段？
-   由于grpc采用的http2协议，因此在tcp链接建立后，需要进行帧的设置，双方同步一下信息，如帧大小的设置，初始窗口大小的设置；以及PRI校验（PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n）等；
-   接下来，grpc客户端就可以使用新创建的tcp链接向grpc服务器端发送rpc请求了；如，调用SayHello方法

接下来，从[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)的角度去深入了解。

为了节省篇幅，只会展示一些最主要，跟本次主题相关的代码，其他核心功能代码，不会展示。

先从服务器端开始：

# 2.源码分析，服务器端一侧

## 2.1.服务器端一侧，注册、启动grpc服务
例如以examples/helloworld/greeter_server/main.go为例：

```go
1．func main() {
2．   lis, err := net.Listen("tcp", port)
3．   //---省略异常处理逻辑
4．   s := grpc.NewServer()
5．   pb.RegisterGreeterServer(s, &server{})
6．   if err := s.Serve(lis); err != nil {
7．      log.Fatalf("failed to serve: %v", err)
8．   }
9．}
```
主要代码说明:
-   第2行：创建一个监听，端口号是port，协议是tcp
-   第4行：创建grpc服务器
-   第5行：将能够提供的服务，注册到grpc服务器里
-   第6行：启动grpc服务器

## 2.2、服务器端如何监听到客户端的tcp请求？

进入Serve内部(grpc-go/server.go文件里):

```go
1. func (s *Server) Serve(lis net.Listener) error {
2.       for {
3.		        rawConn, err := lis.Accept()
4.		       //----省略不相关代码
5.		        go func() {
6.			       s.handleRawConn(rawConn)
7.		        }()
8.	  }
9.  }
```

这块代码的**核心思想**：就是循环阻塞式监听grpc客户端的请求；

服务器端使用golang原生自带的net包中的Listener来监听客户端的tcp请求。

当没有请求时就会阻塞在第3行；

当监听到grpc客户端的请求后，会创建一个协程来处理此请求。

也就说一个grpc客户端的请求，grpc服务器端会专门启动一个协程来处理。

## 2.3、服务器端是如何具体处理客户端请求的？

一次客户端的rpc请求，服务器端会创建一个协程专门来处理该请求。

接下来分析的是，当服务器端监听到客户端请求后的，服务器端一侧，如何来处理。

进入grpc-go/server.go文件的handleRawConn方法里：

```go
1．func (s *Server) handleRawConn(rawConn net.Conn) {
2．   if s.quit.HasFired() {
3．      rawConn.Close()
4．      return
5．   }
6．   rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))

7．   conn, authInfo, err := s.useTransportAuthenticator(rawConn)
 //---省略异常处理逻辑
8．   st := s.newHTTP2Transport(conn, authInfo)
 //----省略不相关代码
9．   go func() {
10．      s.serveStreams(st)
11．      s.removeConn(st)
12．   }()
13．}

```

主要流程说明：

-   第2-5行：校验grpc服务状态
-   第6行：设置tcp链路的deadline
-   第7行：给链路添加认证
-   第8行：http2握手，创建http2Server, 跟客户端交换帧的初始化信息，如帧的大小，窗口大小等
-   第9-10行：创建一个协程，专门来处理流

至此，跟客户端已经建立起了grpc链接，接下来，就是处理客户端发送过来是数据，

为此，服务器端一侧，是用流来处理客户端发送过来的数据帧，头帧等类型的数据的。

具体流程，会在以后的章节中介绍。

## 2.4.服务器端跟客户端帧握手阶段？
接下来，主要是分析一下，服务器端是如何跟客户端进行帧的握手的；

进入grpc-go/server.go文件中的newHTTP2Transport方法：

```go
func (s *Server) newHTTP2Transport(c net.Conn, authInfo credentials.AuthInfo) transport.ServerTransport {
   //----省略不相关代码
   st, err := transport.NewServerTransport("http2", c, config)
   //---省略异常处理逻辑

   return st
}
```

直接进入grpc-go/internal/transport/transport.go文件中的NewServerTransport方法：

```go
func NewServerTransport(protocol string, conn net.Conn, config *ServerConfig) (ServerTransport, error) {
   return newHTTP2Server(conn, config)
}
```

直接进入grpc-go/internal/transport/http2_server.go文件中的newHTTP2Server方法：

由于该方法比较长，只介绍跟本次主题相关的代码：

```go
1．func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
2．	 //----省略不相关代码
3．framer := newFramer(conn, writeBufSize, readBufSize, maxHeaderListSize)
4．	isettings := []http2.Setting{{
5．		ID:  http2.SettingMaxFrameSize,
6．		Val: http2MaxFrameLen,
7． 	}}
8．   //----为了节省篇幅，省略掉 设置帧的其他初始化，代码风格类似
9． 	if err := framer.fr.WriteSettings(isettings...); err != nil {
10．		return nil, connectionErrorf(false, err, "transport: %v", err)
11．	}

12．	delta := uint32(icwz - defaultWindowSize)
13．	if delta > 0 {
14．		if err := framer.fr.WriteWindowUpdate(0, delta); err != nil {
15．			return nil, connectionErrorf(false, err, "transport: %v", err)
16．		}
17．	}
18．   //----省略不相关代码
19．	t := &http2Server{
20．       //----省略初始化相关代码
21．	}
22．    //----省略不相关代码

23．	preface := make([]byte, len(clientPreface))
24．	if _, err := io.ReadFull(t.conn, preface); err != nil {
25．		//---省略异常处理逻辑
26．	}
27．	if !bytes.Equal(preface, clientPreface) {
28．		//---省略异常处理逻辑
29．	}
30．	frame, err := t.framer.fr.ReadFrame()
31．     //----省略不相关代码
32．	sf, ok := frame.(*http2.SettingsFrame)
33．    	//---省略异常处理逻辑
34．	t.handleSettings(sf)
35．	go func() {
36．		t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst)
37．		t.loopy.ssGoAwayHandler = t.outgoingGoAwayHandler
38．		if err := t.loopy.run(); err != nil {
39．	//----省略不相关代码
40．	}()
41．	go t.keepalive()

42．	return t, nil
43．}

```

主要流程说明：
-   第3行：在服务器端创建了Framer，Framer内部其实是调用的是http2包中的newFramer方法，通过此方法来创建http2的帧。
-   第4-11行： 初始化设置帧，类型是http2.Setting切片；包括：帧的最大值，最大流数，初始化窗口的大小等的设置，然后将这些信息发送给客户端，客户端会根据这些信息，更新本地的相应设置。
-   第12-17行：向客户端发送窗口更新帧；
-   第23-29行：从tcp链接里读取客户端发送过来的PRI值，跟服务器端存储的clientPreface值进行比较，校验
-   第30-34行：读取客户端发送过来的帧，并转换为设置帧，然后将设置帧的内容更新到本地。
-   第35-40行：创建一个协程，来启动帧发送器。
-   第41行：创建一个协程，来启动keepalive； 说明服务器端的keepalive，默认是启动的。

至此，在服务器端一侧，完成了跟客户端的帧交互；
即服务器端将自己的帧的大小，初始化窗口大小等信息已经发送给了客户端，
同时也接收到客户端的帧信息，并在本地进行相应的设置。
通过这些操作，双方已经知道了对方接收数据的能力，接下来就进入到了流的处理阶段了。

# 3.源码分析，客户端一侧

## 3.1.grpc客户端，链接参数设置？

以/grpc-go/examples/helloworld/greeter_client/main.go为例:

```go
1．func main() {
2．   conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())

3．   //---省略不相关代码
4．   r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
}
```

主要代码说明：

-   第2行：通过grpc.Dial跟grpc服务器端建立连接的；链接参数包括：
    -   链接地址的设置address,
    -   链路是否采用加密设置，
    -   是否是阻塞式链接(如果设置了grpc.WithBlocak就说明是阻塞式设置，也就是说必须等到链路链接完成后，才能进行rpc请求，也就是调用SayHello方法)
-   第4行：是双方建立连接后，grpc客户端做的事情，比方说调用sayHello方法。

进入Dial方法里(grpc-go/clientconn.go):

```go
func Dial(target string, opts ...DialOption) (*ClientConn, error) {
      // example:///lb.example.grpc.io
      return DialContext(context.Background(), target, opts...)
}
```

Dial函数中参数值的举例说明:
-   参数target的值，大概形式如：如localhost:50051，或者example:///lb.example.grpc.io
-   参数opts的值，大概形式如：grpc.WithInsecure(), grpc.WithBlock()  
    进入DialContext函数里(/grpc-go/clientconn.go)

## 3.2、创建客户端连接器ClientConn、并进行相应的初始化

由于DialContext函数比较长，这里就不整体展示了，按照核心代码一块一块的解释就行。

### 3.2.1、创建ClientConn
```go
1．cc := &ClientConn{
2．   target:            target,
3．   csMgr:             &connectivityStateManager{},

4．   conns:             make(map[*addrConn]struct{}),
5．   dopts:             defaultDialOptions(),
6．   blockingpicker:    newPickerWrapper(),

7．   czData:            new(channelzData),
8．   firstResolveEvent: grpcsync.NewEvent(),
9．}
```

主要代码说明：
-   第1-9行：创建了ClientConn对象
    -   第2行：初始化target(如localhost:50051)，主要是说明，该ClientConn属于哪个target, 将target跟ClientConn建立起对应关系
    -   第3行：初始化cmMgr，链接过程中，链接的状态会发生变化，链接状态如Idle、Connecting、Ready、TransientFailure、Shutdown
    -   第4行：一个target可能对应多个后端服务地址(也就是说可能存在多个grpc服务器端提供相同的服务)，一个服务地址对应一个addrConn
-   第5行：设置默认的链接参数，如重试机制、健康校验函数、写缓存、读缓存、是否采用代理模式等

### 3.2.2、将用户设置的链接参数，更新到客户端连接器ClientConn

继续看下面的代码：
```go
for _, opt := range opts {
   opt.apply(&cc.dopts)
}
```

这块代码的核心思想就是，

将opts的值初始化到默认连接参数cc.dopts里;

也就是说，将用户指定的链接参数(如是否阻塞式链接，[拦截器](https://so.csdn.net/so/search?q=%E6%8B%A6%E6%88%AA%E5%99%A8&spm=1001.2101.3001.7020)的设置，超时机制的设置，重试机制，keepalive等等)更新到cc.dopts里。

注意：
-   此块代码，只是将用户设置的链接参数，初始化到客户端连接器ClientConn里;
-   如是否是安全链接，是否是阻塞式链接;
-   但是，事情还没有结束，会在接下来的代码中，会判断用户是否设置了链接参数，如果设置了会进行相应的处理；
-   比方说用户设置服务配置，那么会进行相应的反序列化，将用户设置的服务配置转换到grpc框架内置的结构体ServiceConfig里；
-   如果用户设置了安全连接的话，会继续判断是否设置了认证参数等；
-   也就是说，代码实现的思路是，先将用户设置的连接参数初始化到ClientConn里，然后再对提供的链接参数进行相应的处理。

### 3.2.3、用户设置的连接参数，是如何具体更新到客户端连接器ClientConn里的呢？

opts的类型是变参DialOption类型，相当于切片DialOption类型；

opts的值，如用户设置的grpc.WithInsecure(), grpc.WithBlock()

而DialOpiton是一个interface类型，如(grpc-go/dialoptions.go)

```go
type DialOption interface {
	apply(*dialOptions)
}
```

DialOption接口中有一个函数apply，接收的参数是dialOptions；

grpc框架中，funcDialOption实现了该接口，(grpc-go/dialoptions.go):

```go
1．type funcDialOption struct {
2．	f func(*dialOptions)
3．}

4．func (fdo *funcDialOption) apply(do *dialOptions) {
5．	fdo.f(do)
6．}

7．func newFuncDialOption(f func(*dialOptions)) *funcDialOption {
8．	return &funcDialOption{
9．		f: f,
10．	}
11．}
```

主要流程说明：

-   第1-3行：创建funcDialOption 结构体，内部声明了一个func(\*dialOptions)类型的变量f
-   第4-6行：重写了apply方法，参数为\*dialOptions；
    -   内部第5行，就是将结构体funcDialOption 中的函数f变量执行一次，
    -   也就是说当执行apply方法的时候，其实真正执行的是结构体funcDialOption中的函数变量f，参数是由apply的参数传进去的。
-   第7-10行：创建函数newFuncDialOption，来创建结构体funcDialOption ；newFuncDialOption的参数类型为一个函数func(\*dialOptions)；

最主要明白一点，调用apply方法，其实就是调用结构体funcDialOption中的函数变量f。

接下来，我们看看grpc.WithBlock是如何执行的？

```go
1．func WithBlock() DialOption {
2．	return newFuncDialOption(
3．           func(o *dialOptions) {
4．	       	o.block = true
5．	        }
6．       )
}
```

WithBlock函数，内部调用的是newFuncDialOption函数， 而参数为第3-5行，为函数类型；
前文中的opt.apply(&cc.dopts)，opt的参考值，如grpc.WithBlock函数，返还的结果是newFuncDialOption，
也就是说grpc.WithBlock创建了一个funcDialOption 结构体
该结构体中的函数变量f，就是第3-5行；
经刚才的分析已经知道，当执行apply的时候，其实就是执行第3-5行；
再换一句话说，
当批量执行apply的时候，其实，就是执行类似的第3-5行，apply中的参数，就是第3行中的参数；
最终将用户设置的链接参数，更新到了客户端连接器ClientConn的dopts变量里;

### 3.2.4、客户端连接器ClientConn如何设置拦截器的？
相关代码：
```go
chainUnaryClientInterceptors(cc)
chainStreamClientInterceptors(cc)
```

主要是设置拦截器(拦截器的原理，在其他章节会详细介绍)。

这块代码，针对的链接参数，如

```go
grpc.WithStreamInterceptor(),
grpc.WithChainStreamInterceptor(),
grpc.WithUnaryInterceptor(),
grpc.WithChainUnaryInterceptor
```
相关参考例子：
examples/features/interceptor
经过上面的两个函数，最终将用户设置的拦截器，
更新到客户端连接器ClientConn的unaryInt、chainUnaryInts、streamInt、chainStreamInts里

### 3.2.5、客户端连接器ClientConn是如何处理链接安全的？

相关代码：

```go
if !cc.dopts.insecure {
   if cc.dopts.copts.TransportCredentials == nil && cc.dopts.copts.CredsBundle == nil {
      return nil, errNoTransportSecurity
   }
   if cc.dopts.copts.TransportCredentials != nil && cc.dopts.copts.CredsBundle != nil {
      return nil, errTransportCredsAndBundle
   }
} else {
   if cc.dopts.copts.TransportCredentials != nil || cc.dopts.copts.CredsBundle != nil {
      return nil, errCredentialsConflict
   }
   for _, cd := range cc.dopts.copts.PerRPCCredentials {
      if cd.RequireTransportSecurity() {
         return nil, errTransportCredentialsMissing
      }
   }
}
```

核心思想就是对链接安全进行校验操作；

这块代码针对的链接参数，

如grpc.WithInsecure()、grpc.WithPerRPCCredentials()、grpc.WithTransportCredentials()

如果想设置此次的tcp链接为非安全连接的话，需要显示的设置grpc.WithInsecure()；

不设置的话，默认为安全连接。

参考例子，框架自带的参考例子，如examples/features/authentication、examples/features/encryption

### 3.2.6、客户端连接器ClientConn如何解析用户设置的服务配置ServiceConfig

```go
if cc.dopts.defaultServiceConfigRawJSON != nil {
		scpr := parseServiceConfig(*cc.dopts.defaultServiceConfigRawJSON)
		if scpr.Err != nil {
			return nil, fmt.Errorf("%s: %v", invalidDefaultServiceConfigErrPrefix, scpr.Err)
		}
		cc.dopts.defaultServiceConfig, _ = scpr.Config.(*ServiceConfig)
}
```

这块代码针对的链接参数，如grpc.WithDefaultServiceConfig();

参考例子，如grpc框架自带的测试用例examples/features/retry/client/main.go中有使用。

核心目的就是

通过上面的代码，将用户通过grpc.WithDefaultServiceConfig()设置的配置，反序列化到grpc框架中的ServiceConfig结构体里

### 3.2.7、客户端连接器ClientConn如何设置一个dialer函数的？专门用于创建tcp链接？

相关代码：
```go
1．if cc.dopts.copts.Dialer == nil {
2．   cc.dopts.copts.Dialer = func(ctx context.Context, addr string) (net.Conn, error) {
3．      network, addr := parseDialTarget(addr)

4．      return (&net.Dialer{}).DialContext(ctx, network, addr)
5．   }

6．   if cc.dopts.withProxy {
7．      cc.dopts.copts.Dialer = newProxyDialer(cc.dopts.copts.Dialer)
8．   }
9．}
```

这块代码很重要，在向grpc服务器端发起链接请求时就是通过这个函数实现的

主要代码说明：

-   第2行：Dialer 很明显是函数类型，接收两个参数；这里仅仅是赋值，并不是调用Dialer函数；
    -   第2个参数addr的值，类似于:localhost:50051 这种形式
-   第3行：对addr进行解析，返还两个变量
    -   network的有效值是tcp, unix;
    -   addr的值，类似于：localhost:50051
-   第4行：调用golang原生自带的net包中的Dialer结构体的DialContext方法进行链接
-   第7行：如果设置了代理的话，可以通过代理的方式进行链接；

dialer函数，专门用于创建tcp链接；grpc框架内置了一个， 如上面的第2-8行。

当然，用户也可以通过grpc.WithContextDialer(自定义dialer函数)来启动自定义的dialer函数。

### 3.2.8、客户端连接器ClientConn如何获取解析构建器的？

相关代码：
```go
1. cc.parsedTarget = grpcutil.ParseTarget(cc.target)
2. resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)

3. if resolverBuilder == nil {
4.   cc.parsedTarget = resolver.Target{
5.      Scheme:   resolver.GetDefaultScheme(),
6.      Endpoint: target,
7.   }
8.  
9.   resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)

10.   if resolverBuilder == nil {
11.      return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
12.   }
13.}
```

这块代码的核心目的是

创建解析构建器resolverBuilder，

有了这个resolverBuilder对象，就可以调用构建方法build，从而创建出解析器resolver来。(解析器原理可以参考相关章节)

主要代码说明：

-   第1行：通过对cc.target进行解析得到ClientConn中的属性parseTarget结构体，该结构体有三个属性：
    -   Scheme string: 参考值，如：dns，grpclb-internal，example，passthrough等等；可以简单的认为是解析器的名称；如dns表示该解析器底层是通过调用dns服务来解析用户指定的地址的
    -   Authority string:
    -   Endpoint string: 参考值如localhost:50051,或者域名foo.bar等等
-   第2行：根据scheme获取解析构建器resolverBuilder，该值可能存在也可能不存在
-   第3-12行：如果resolverBuilder不存在的话，就会使用grpc内置的默认解析构建器passthrough

具体使用哪个解析构建器，是根据设置的具体地址的协议来选择的。

grpc框架中可以同时存在多种解析器；  
解析器构建器的注册，有两种方式：

-   一种是通过resolver.Register，进行注册；
-   一种是通过设置grpc.WithResolvers(自定义多个解析构建器)来注册；  
    grpc.WithResolvers的优先级别高于resolver.Register;

### 3.2.9、客户端连接器ClentConn是如何设置阻塞式链接的？

相关代码：
```go
if cc.dopts.block {
		for {
			s := cc.GetState()
			if s == connectivity.Ready {
				break
			} else if cc.dopts.copts.FailOnNonTempDialError && s == 
          //---省略异常处理逻辑
			}

			if !cc.WaitForStateChange(ctx, s) {
				if err = cc.connectionError(); err != nil && cc.dopts.returnLastError {
					// 说明：超时失败的原因，是因为链接失败导致的。并不是真正的因为用户设置的超时时间
					return nil, err
				}
				// 如果返回这里的话，说明，链接还没有进行，用户设置的超时时间，就已经到期了
				return nil, ctx.Err()
			}
		}
    }
```

基本思路就是：
跟grpc服务器端建立tcp链接是异步方式，这里可以不断的获取链接的状态，
如果链接的状态为Ready，就是说明，链接完成了。否则异常。
针对的是用户设置的链接参数grpc.WithBlock();
如果想实现必须跟服务器端建立起tcp链接后，才能进行流的处理的话，就必须显示的设置grpc.WithBlock()

## 3.3、如何通过用户设置的链接地址来获取后端对应的服务地址列表呢？(解析器)

### 3.3.1、grpc框架获取后端服务地址列表的基本思路？
对于同一个服务，后端可能存在多个服务器来提供服务的场景；
如SayHello服务，可能192.168.1.110, 192.168.1.111,192.168.1.112上都能提供SayHello服务；

这些服务地址列表，可能会经常变化，那么当客户端请求SayHello服务时设置的链接地址，不可能全部写上；
**如何解决呢？**
可以设置一个字符串，该字符串是有具体格式的，会在解析器章节具体介绍。
也就是说，该字符串就表示提供SayHello服务的地址列表；
无论提供服务的后端地址如何变化，只要该字符串不变即可。
比如kubernetes中的service跟pod的关系，容器地址是变化的，只要service不变就行了。
### 3.3.2、实现解析器插件的基本思路？
grpc框架中的解析器就是解析字符串来获取后端对应的服务地址列表的。
grpc框架中，解析器是通过插件形式来实现的。
用户可以根据实际需求来自定义解析器。
大致思路是：
-   grpc框架将地址解析的处理流程都规定好，第一步做什么，第二步做什么，第三步做什么等等；只需要在某些步骤上使用接口编程来实现即可。
-   当用户想实现自己的解析器时，就可以实现这些接口，从而实现了自己的解析器。
### 3.3.3、创建解析器的包装类ccResolverWrapper？将解析器封装到此类里

接下来进入newCCResolverWrapper函数里：

```go
1． func newCCResolverWrapper(cc *ClientConn, rb resolver.Builder) (*ccResolverWrapper, error) {
2．   ccr := &ccResolverWrapper{
3．      cc:   cc,
4．      done: grpcsync.NewEvent(),
5．   }

6．   //---省略一些初始化之类的代码
7．   ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, rbo)
8．   //---省略异常处理逻辑
9．   return ccr, nil
10．} 
```

主要代码说明：

-   第2-5行：创建ccResolverWrapper对象；这是解析器的包装类，跟解析器相关的一些操作都封装到这里。
-   第7行：这一行比较重要，就是通过Build来构建解析器resolver的；  
    解析构建器是用来创建解析器的。有多种实现方式：

假设使用默认自带的解析构建器passthoughBuilder的话，查看相关代码：  
(grpc-go/internal/resolver/passthrough/passthrough.go）

```go
1．func (*passthroughBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
2．   r := &passthroughResolver{
3．      target: target,
4．      cc:     cc,
5．   }

6．   r.start()

7．   return r, nil
8．}
```
主要代码说明：
-   第2-5行：创建passthroughResolver解析器；
    -   target的参考值，如：{“Scheme”:“passthrough”,“Authority”:"",“Endpoint”:“localhost:50051”}
    -   cc，就是前文创建的ccResolverWrapper,可以表明，该解析器属于哪个解析器包装类。
-   第6行：这一行很重要，调用start方法；

上面第3行中，赋值的target，就是后端对应的服务器地址；

不同的解析器获取后端服务地址列表的策略可能不一样。

比方说：

-   一类解析器是，即使后端存在多个相同服务的地址，只要获取其中一个即可。
-   而另一类解析器是，需要将后端提供相同服务的地址信息，全部获取到。

测试用例中，用户设置的链接地址，其实就是后端服务的真实地址。

因此，不需要再去远程访问获取后端服务地址列表了。

也就是说，到此为止，解析器已经完成了根据设置的链接字符串来获取后端对应的服务地址了。

### 3.3.4、将获取到的后端服务地址列表信息更新到解析器里

进入start方法内部看看：

```go
func (r *passthroughResolver) start() {
     r.cc.UpdateState(resolver.State{Addresses: []resolver.Address{{Addr: r.target.Endpoint}}})
}
```

将获取到后端服务地址信息封装到解析器的Addresses切片里；然后封装到resolver.State里；

### 3.3.5、更新解析器的状态

继续进入UpdateState内部：  
grpc-go/resolver_conn_wrapper.go

```go
1．func (ccr *ccResolverWrapper) UpdateState(s resolver.State) {
2．   if ccr.done.HasFired() {
3．      return
4．   }
//---省略掉不相关代码

5．   ccr.curState = s

6．   ccr.poll(ccr.cc.updateResolverState(ccr.curState, nil))
7．}
```

在这块代码块中，

-   通过第5行，将获取到的后端服务地址列表信息，最终存储到了解析器包装类ccResolverWrapper里的curState里；注意一下：state是状态的意思，但是，这里并不是Running，Waiting之类的意思，经常会混淆。
-   最核心的是第11行中的updateResolverState方法。至于ccr.poll方法，可以先不管。在dns解析器相关章节会详细介绍。  
    ccr.curState值的参考例子，如：

```go
{"Addresses":[{"Addr":"localhost:50051","ServerName":"","Attributes":null,"Type":0,"Metadata":null},{"Addr":"localhost:50052","ServerName":"","Attributes":null,"Type":0,"Metadata":null}],"ServiceConfig":null,"Attributes":null}
```

解析器还有另外一个作用，就是触发平衡器的流程的开始。

## 3.4、如何向服务器端发起tcp链接？主要是链接的策略是什么？(平衡器)

### 3.4.1、根据ServiceConfig的配置策略来触发平衡器的创建？

接下来进入updateResolverState方法，  
grpc-go/clientconn.go（列出的代码是删减过的，不影响主流程）

```go
1．func (cc *ClientConn) updateResolverState(s resolver.State, err error) error {
2．   var ret error
3．   if cc.dopts.disableServiceConfig || s.ServiceConfig == nil {
4．      cc.maybeApplyDefaultServiceConfig(s.Addresses)
5．      } else {
6．      if sc, ok := s.ServiceConfig.Config.(*ServiceConfig); s.ServiceConfig.Err == nil && ok {
7．         cc.applyServiceConfigAndBalancer(sc, s.Addresses)
8．      } else {
  //---省略异常处理逻辑}
9．   var balCfg serviceconfig.LoadBalancingConfig
10．   if cc.dopts.balancerBuilder == nil && cc.sc != nil && cc.sc.lbConfig != nil {
11．      balCfg = cc.sc.lbConfig.cfg
12．     }
13．     cbn := cc.curBalancerName
14．    bw := cc.balancerWrapper
15．   cc.mu.Unlock()
16．     if cbn != grpclbName {
17．      for i := 0; i < len(s.Addresses); {
18．         if s.Addresses[i].Type == resolver.GRPCLB {
19．           copy(s.Addresses[i:], s.Addresses[i+1:])
20．            s.Addresses = s.Addresses[:len(s.Addresses)-1]
21．            continue
22．         }
23．         i++
24．      }
25．   }
26．  uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
//---省略掉不相关代码
27．}

```

主要代码说明：

-   第3-7行：根据初始条件下是否配置了serviceConfig，会执行不同的分支；maybeApplyDefaultServiceConfig方法内部其实调用的是applyServiceConfigAndBalancer。
    -   如果用户没有设置ServiceConfig或者显示的设置了grpc.WithDisableServiceConfig()的话，就会执行maybeApplyDefaultServiceConfig方法
    -   如果用户设置了ServiceConfig的话，就会执行applyServiceConfigAndBalancer
    -   二者的最终目的其实，都是创建平衡器包装类
-   第16-25行：这一块代码的核心目的是过滤掉地址类型是grpclb的;
-   第26行：这一行代码，很重要。更新ClientConn的状态；将解析器的状态State(后端服务地址列表信息)，以及loadBalancingConfig信息封装到balancer.ClientConnState里；最终更新ClientConnState

其实，核心目的就是创建平衡器包装类，然后更新平衡器的状态。

### 3.4.2、如何选择平衡器？也就是选择平衡器的策略？

grpc框架中，平衡器的实现同样是插件式的，允许多种平衡器的存在，

因此，需要采用一定的策略，来选择平衡器。

平衡器获取到解析器解析的后端服务地址列表后，接下来要做的事情，就是要发起连接吧。

但是，可能会存在多个连接地址，而且每个服务地址对应的服务的负载又不一样；

那么，应该有一个连接的策略，比方说，是选择第一个地址连接，还是hash随机选择一个服务地址连接，还是将所有的服务地址全部连接呢，或者根据负载大小，选择负载最小的服务地址连接呢？

应该是有一策略来维护的。

平衡器就是来解决这些事情的。

可以使用接口来实现，当用户想实现自己的链接策略时，就可以实现相应的接口，即可。

从而自定义了平衡器。

接下来看看applyServiceConfigAndBalancer方法内部，到底是做什么的：

```go
1．func (cc *ClientConn) applyServiceConfigAndBalancer(sc *ServiceConfig, addrs []resolver.Address) {
2．    //---省略掉不相关代码
3．    cc.sc = sc
4．    //---省略掉不相关代码

5．   if cc.dopts.balancerBuilder == nil {
6．     var newBalancerName string
7．     if cc.sc != nil && cc.sc.lbConfig != nil {
8．         newBalancerName = cc.sc.lbConfig.name
9．        } else {
10．         var isGRPCLB bool
11．         for _, a := range addrs {
12．            if a.Type == resolver.GRPCLB {
13．               isGRPCLB = true
14．               break
15．            }
16．         }
17．         if isGRPCLB {
18．            newBalancerName = grpclbName
19．         } else if cc.sc != nil && cc.sc.LB != nil {
20．            newBalancerName = *cc.sc.LB
21．         } else {
22．            newBalancerName = PickFirstBalancerName
23．         }
24．      }
25．     cc.switchBalancer(newBalancerName)
26．   } else if cc.balancerWrapper == nil {
27．      cc.curBalancerName = cc.dopts.balancerBuilder.Name()
28．      cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
29．   }
30．}

```

从方法的名称可以看出该方法至少会涉及到两方面的内容:serviceConfig和平衡器Balancer  
主要代码说明：

-   第3行：将用户设置的serviceConfig赋值到cc.sc
-   第5-30行：主要是涉及如何构建平衡器Balancer，优先判断平衡构建器balancerBuild是否存在  
     a)若不存在：  
      i.第6-24行：核心目的是如何获取平衡器的名称newBalancerName，优先判断cc.sc 是否赋值了 并且判断 cc.sc.lbConfig 是否赋值了  
       1.若赋值了：就从cc.sc.lbConfig中的name属性获取，赋值给newBalancerName  
       2.若没有赋值：从addrs中判断是否存在resolver.GRPCLB类型的地址  
        a)若存在：就将“grpclb”赋值给newBalancerName  
        b)若不存在：就判断cc.sc是否存在并且cc.sc.LB是否存在  
         i.若存在：就将\*cc.sc.LB赋值给newBalancerName  
         ii.若不存在：就使用默认平衡器名称“pick_first”，赋值给newBalancerName  
      ii.第25行：根据平衡器名称，创建平衡器。  
     b)若存在，并且cc中的属性balancerWrapper没有初始化;  
      i.就使用用户指定的平衡器名称，并且调用newCCBalancerWrapper方法，构建ccBalancerWrapper对象

### 3.4.3、如何创建平衡器

进入switchBalancer方法内部，看看主要做了哪些事情：  
grpc-go/clientconn.go文件里：

```go
1．func (cc *ClientConn) switchBalancer(name string) {
   //---省略掉不相关代码
2．   builder := balancer.Get(name)
3．   if builder == nil {
4．      builder = newPickfirstBuilder()
5．   } 
   //---省略掉不相关代码
6．   cc.curBalancerName = builder.Name()
7．   cc.balancerWrapper = newCCBalancerWrapper(cc, builder, cc.balancerBuildOpts)
8．}
```

主要代码说明：

-   第2行：根据传入的平衡器名称从注册里获取平衡构建器builder
-   第3-5行：如果没有找到指定的平衡构建器builder，就使用默认的Pickerfirst构建器
-   第6行：从平衡构建器里获取平衡器的名称赋值给 cc.curBalancerName
-   第7行：创建平衡器包装对象，赋值给cc.balancerWrapper，newCCBalancerWrapper方法主要是创建ccBalancerWrapper对象，并且创建平衡器对象；详细分享可以参考平衡器原理章节，这里就不详细讲解了。

到此为止，知道平衡器的名称，创建了平衡器，创建了平衡器包装类。

### 3.4.4、更新平衡器的状态，updateClientConnState？

接下来返回到grpc-go/clientconn.go文件中updateResolverState方法，

继续往下看，

```go
uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
```

进入updateClientConnState方法内部：

```go
func (ccb *ccBalancerWrapper) updateClientConnState(ccs *balancer.ClientConnState) error {
   ccb.balancerMu.Lock()
   defer ccb.balancerMu.Unlock()

   return ccb.balancer.UpdateClientConnState(*ccs)
}
```

实际调用的是平衡器balancer的UpdateClientConnState方法，其中balancer类型是接口；

这里我们依旧选择grpc框架内置的默认平衡器pickfirstBalancer，

进入grpc-go/pickfirst.go文件中的UpdateClientConnState方法：

```go
1．func (b *pickfirstBalancer) UpdateClientConnState(cs balancer.ClientConnState) error {
2．   if len(cs.ResolverState.Addresses) == 0 {
3．      b.ResolverError(errors.New("produced zero addresses"))
4．      return balancer.ErrBadResolverState
5．   }
6．   if b.sc == nil {
7．      var err error
8．      b.sc, err = b.cc.NewSubConn(cs.ResolverState.Addresses, balancer.NewSubConnOptions{})
//---省略异常处理逻辑
9．       b.state = connectivity.Idle
10．      b.cc.UpdateState(balancer.State{ConnectivityState: connectivity.Idle, Picker: &picker{result: balancer.PickResult{SubConn: b.sc}}})
11．      b.sc.Connect()
12．      } else {
13．         b.sc.UpdateAddresses(cs.ResolverState.Addresses)
14．         b.sc.Connect()
15．   }

16．   return nil
17．}

```

该方法最核心的目的，就是调用SubConn结构体的Connect方法，开始建立链接  
主要代码说明：

-   第2-5行：校验解析器的地址数量是否满足要求，至少得有一个吧。
-   第8行：主要根据cs.ResolverState.Addresses构建SubConn结构体；注意：一个SubConn，管理着多个后端服务地址Addr；也就是说，管理着多个链接。
-   第9行：在真正tcp链接前，将平衡器的状态设置为Idle
-   第10行：更新ClientConn的状态为Idle，以及初始化picker；关于picker的说明，可以参考平衡器原理章节，此处可以不用关心。
-   第11行：调用Connect方法，开始链接。

接下来，进入Connect方法：

### 3.4.5、平衡器时如何发起tcp链接的

平衡器包装类里的addrConn发起链接的。

Connect()是有grpc框架实现的，用户不用实现，

这里选择acBalancerWrapper,  
grpc-go/balancer_conn_wrappers.go文件中的Connect()：

```go
func (acbw *acBalancerWrapper) Connect() {
   acbw.mu.Lock()
   defer acbw.mu.Unlock()

   acbw.ac.connect()
}

```

继续进入connect()：  
grpc-go/clientconn.go文件中的：

```go
1．func (ac *addrConn) connect() error {
2．   ac.mu.Lock()
3．   if ac.state == connectivity.Shutdown {
4．      ac.mu.Unlock()
5．      return errConnClosing
6．   }

7．   if ac.state != connectivity.Idle {
8．      ac.mu.Unlock()
9．      return nil
10．   }
11．   ac.updateConnectivityState(connectivity.Connecting, nil)
12．   ac.mu.Unlock()

13．   go ac.resetTransport()
14．   return nil
15．}

```

主要代码说明：

-   第3-10行：链接前的状态校验
-   第11行：将addrConn结构体更新为Connecting；注意，平衡器的状态应该还是Idle
-   第13行：专门启动一个协程去链接grpc服务器，异步链接。

进入grpc-go/clientconn.go文件中的resetTransport方法：

(此方法有90行左右，此处就不展示了，不影响本次讲解；resetTransport该方法会在其他章节介绍)，

找到最关键的一行

```go
 newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)，
```

进入grpc-go/clientconn.go文件中的tryAllAddrs方法：

```go
1．func (ac *addrConn) tryAllAddrs(addrs []resolver.Address, connectDeadline time.Time) (transport.ClientTransport, resolver.Address, *grpcsync.Event, error) {
2．   var firstConnErr error
3．   for _, addr := range addrs {
4．      ac.mu.Lock()
5．      if ac.state == connectivity.Shutdown {
6．         ac.mu.Unlock()
7．         return nil, resolver.Address{}, nil, errConnClosing
8．      }
9．      //---省略掉不相关代码

10．      newTr, reconnect, err := ac.createTransport(addr, copts, connectDeadline)
11．  		if err == nil {
12．			return newTr, addr, reconnect, nil
13．		}
14．   }
15．   return nil, resolver.Address{}, nil, firstConnErr
16．}

```

主要代码说明：

-   第3-14行：就是对所有的后端服务地址进行连接; 但是只要有一个链接成功，就退出。
    -   此时有一个疑问，如果想对所有的服务地址进行链接的话，这里不是有问题么？
        -   实际上，这里的addrs虽然是切片，但是，只会传递一个地址。
        -   不同平衡器的链接策略是在UpdateClientConnState方法里的设置的，
        -   如果想链接所有的服务器地址的话，可以在此方法里遍历执行。
        -   比方说baseBalancer平衡器，就是在UpdateClientConnState里链接所有的服务地址列表的
-   第5-8行：连接前，先进行状态校验，若addrConn的状态为shutdown就不进行连接了，退出。
-   第10行：进行连接

至此，经历了如何选择平衡器，如何创建平衡器，采用链接的策略(如，是全部连接，还是链接其中一个)； 接下来，进入真正的tcp链接过程；

## 3.5、进入tcp链接阶段

进入grpc-go/clientconn.go文件中的createTransport方法里：

找到最核心的一行

```go
newTr, err := transport.NewClientTransport(connectCtx, ac.cc.ctx, addr, copts, onPrefaceReceipt, onGoAway, onClose)，
```

其他语句主要是为了调用该方法做的初始化赋值操作，就不展示了。

继续进入grpc-go/internal/transport/transport.go文件中的NewClientTransport方法：

```go
func NewClientTransport(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (ClientTransport, error) {
     return newHTTP2Client(connectCtx, ctx, addr, opts, onPrefaceReceipt, onGoAway, onClose)
}
```

最终调用的是grpc-go/internal/transport/http2_client.go文件中的newHTTP2Client方法：(只展示跟本次主题相关的代码)

```go
1．func newHTTP2Client(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (_ *http2Client, err error) {
2．   scheme := "http"
3．   ctx, cancel := context.WithCancel(ctx)
4．   conn, err := dial(connectCtx, opts.Dialer, addr.Addr)
5．  
6．}
```

主要代码说明：

-   第4行: 就是通过dial函数实现跟grpc服务器端的链接；底层调用调用是golang原生的net进行的tcp链接。

进入grpc-go/internal/transport/http2_client.go文件中的dial方法里：

```go
1．func dial(ctx context.Context, fn func(context.Context, string) (net.Conn, error), addr string) (net.Conn, error) {
2．   if fn != nil {
3．      return fn(ctx, addr)
4．   }

5．   return (&net.Dialer{}).DialContext(ctx, "tcp", addr)
6．}

```

参数说明：

-   fn，就是在grpc-go/clientconn.go文件中DialContext方法内部，初始化ClientConn结构体时设置的；就是下面的代码：

```go
if cc.dopts.copts.Dialer == nil {
   cc.dopts.copts.Dialer = func(ctx context.Context, addr string) (net.Conn, error) {
         network, addr := parseDialTarget(addr)
         return (&net.Dialer{}).DialContext(ctx, network, addr)
      }
   if cc.dopts.withProxy {
      cc.dopts.copts.Dialer = newProxyDialer(cc.dopts.copts.Dialer)
   }
}

```

其中，fn就是cc.dopts.copts.Dialer

-   addr, 值的参考形式，如localhost:50051  
    可见，底层是通过调用golang自带的net包中的Dialer结构体的DialContext方法进行连接的。

## 3.6、客户端一侧，进入帧设置阶段

前文分析，tcp链路已经建立起来了，接下来，需要双方进行一些帧的初始化设置，包括传输时的帧大小设置，窗口大小设置等。

grpc-go/internal/transport/http2_client.go文件中的newHTTP2Client方法：(只展示跟本次主题相关的代码)

```go
1．func newHTTP2Client(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (_ *http2Client, err error) {
2．   //---省略掉不相关代码
3．
4．   t := &http2Client{
5．      ctx:       ctx,
       //---省略掉不相关代码
6．  }
7．   go t.reader()

8．   n, err := t.conn.Write(clientPreface)
9．   var ss []http2.Setting

10．   if t.initialWindowSize != defaultWindowSize {
11．      ss = append(ss, http2.Setting{
12．         ID:  http2.SettingInitialWindowSize,
13．         Val: uint32(t.initialWindowSize),
14．      })
15．   }
16．   if opts.MaxHeaderListSize != nil {
17．      ss = append(ss, http2.Setting{
18．         ID:  http2.SettingMaxHeaderListSize,
19．         Val: *opts.MaxHeaderListSize,
20．      })
21．   }
22．   err = t.framer.fr.WriteSettings(ss...)
23．   if delta := uint32(icwz - defaultWindowSize); delta > 0 {
24．   if err := t.framer.fr.WriteWindowUpdate(0, delta); err != nil {
25．      //----省略掉异常处理逻辑
26．   }
27．   t.connectionID = atomic.AddUint64(&clientConnectionCounter, 1)
28．  if err := t.framer.writer.Flush(); err != nil {
29．      return nil, err
30．   }
```

主要流程说明：

-   第4-6行：创建http2Client对象；
-   第7行：启动一个协程，负责读取grpc服务器端发送的各种类型的帧；在帧握手阶段接收到服务器端发送的设置帧，窗口更新帧，将这些信息更新到客户端的本地；也就是说，服务器端已经将自己能够发送、接收的帧的最大值，窗口大小告诉客户端了，客户端需要更新到本地。
-   第8行：链接建立来后，向grpc服务器端发送"PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"；
-   第9-22行：向grpc服务器端发送设置帧http2.Setting；
-   第23-26行：向grpc服务器端发送窗口更新帧；

到此为止，

grpc客户端跟grpc服务器端建立起了tcp链接，双方进行了帧的握手；

为底层连接创建了http2Client, http2Server；

创建了帧的发送器，帧的接收器；

接下来就是流的层面，开始传递头帧，数据帧
