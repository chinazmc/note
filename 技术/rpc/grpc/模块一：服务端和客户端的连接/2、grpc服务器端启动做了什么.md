 
这次分享一下当[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)服务器在启动时都做了什么事情?

可以自己先思考一下，假设让我们自己去开发一个简单版本的grpc服务器端启动时都会做什么事情呢?

-   一些初始化工作
-   监听某个端口
-   注册服务端提供的服务  
    。。。。。

好了，接下来看一下grpc-go框架服务器端启动时的流程图：

![grpc服务器端启动时都做了哪些事情](https://img-blog.csdnimg.cn/20210511101622324.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

在下面的章节中只是介绍了常用的初始化组件，有些功能需要手动显示的调用，

或者import导入才能初始化或者注册，

比方说grpc-go/encoding/gzip/gzip.go文件中的gzip压缩器需要手动导入，因此就不再一一介绍了。

一个链接请求，对应一个http2Server对象，一个帧接收器，一个帧发送器;

# 1、注册、初始化工作

下面几个小节，仅仅列出了grpc-go[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)中哪些文件实现了注册、初始化等工作。

至于详细原理介绍，会再后面的章节中分享。

## 1.1、注册服务

通过下面的形式，可以将提供的服务注册到grpc服务器端，以供客户端调用；

这里我们以源码中自带的heloworld为例，将SayHello服务注册到grpc服务器端：  
![注册服务](https://img-blog.csdnimg.cn/20210511101915261.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

## 1.2、解析器初始化

在grpc框架里内置了几种解析器，也可以自定义解析器；

像passthrough、dns解析器在grpc服务器启动时会自己注册；

像manual、xds解析器，需要在代码里显示注册才生效；

### 1.2.1、passthrough解析器(默认使用，启动时自己注册)

…/internal/resolver/passthrough/passthrough.go文件中

```go
func init() {
   resolver.Register(&passthroughBuilder{})
}
```

### 1.2.2、dns解析器(启动时自己注册)

…/internal/resolver/dns/dns_resolver.go文件中

```go
func init() {
   resolver.Register(NewBuilder())
}
```

### 1.2.3、Manual解析器(需要手动显示的注册)

…/resolver/manual/manual.go文件中

```go
func GenerateAndRegisterManualResolver() (*Resolver, func()) {
   scheme := strconv.FormatInt(time.Now().UnixNano(), 36)
   r := NewBuilderWithScheme(scheme)
   resolver.Register(r)
   return r, func() { resolver.UnregisterForTesting(scheme) }
}
```

如果想要使用manual解析器的话，需要手动显示的注册一下，如下参考例子:  
…/examples/features/health/client/main.go文件

```go
1．func main() {
2．   flag.Parse()

3．   r, cleanup := manual.GenerateAndRegisterManualResolver()
4．   defer cleanup()
5．   r.InitialState(resolver.State{
6．      Addresses: []resolver.Address{
7．         {Addr: "localhost:50051"},
8．         {Addr: "localhost:50052"},
9．      },
10．   })

11．   address := fmt.Sprintf("%s:///unused", r.Scheme())
// -------省略不相关代码------
```

核心代码说明：

-   第3行，手动注册指定解析器

…/examples/features/debugging/client/main.go文件:

```go
1．func main() {
2．   /***** Set up the server serving channelz service. *****/
3．   lis, err := net.Listen("tcp", ":50052")
4．   if err != nil {
5．      log.Fatalf("failed to listen: %v", err)
6．   }
7．   defer lis.Close()
8．   s := grpc.NewServer()
9．   service.RegisterChannelzServiceToServer(s)
10．   go s.Serve(lis)
11．   defer s.Stop()

12．   /***** Initialize manual resolver and Dial *****/
13．   r, rcleanup := manual.GenerateAndRegisterManualResolver()
14．   defer rcleanup()
15．   // Set up a connection to the server.
16．   conn, err := grpc.Dial(r.Scheme()+":///test.server", grpc.WithInsecure(), grpc.WithBalancerName("round_robin"))
17．   if err != nil {
18．      log.Fatalf("did not connect: %v", err)
19．   }
20．   defer conn.Close()
21．// ------省略不相关代码------
```

主要代码说明：

-   第13行，手动注册指定解析器

### 1.2.4、xds 解析器(需要手动显示的注册)

xds解析器，一般情况下，grpc服务器启动时并没有注册xds解析器，需要手动的注册。

…/xds/internal/resolver/xds_resolver.go文件中初始化：

```go
func init() {
   resolver.Register(&xdsResolverBuilder{})
}
```

如何注册xds解析器呢？

在…/xds/xds.go文件中:

```go
1．package xds

2．import (
3．   _ "google.golang.org/grpc/xds/internal/balancer" // Register the balancers.
4．   _ "google.golang.org/grpc/xds/internal/resolver" // Register the xds_resolver
5．)
```

主要流程说明：

-   第4行，注册xds解析器

根据go语言的特性，如果某个包没有使用的话，是不会导入的；

因此，如果我们想使用xds解析器的话，需要手动的显示导入，如下所示:

随便找一个grpc服务器端启动的例子：

```go
import (
   "context"
   "fmt"
   "github.com/CodisLabs/codis/pkg/utils/log"
   "google.golang.org/grpc"
   "google.golang.org/grpc/codes"
   "google.golang.org/grpc/metadata"
   "google.golang.org/grpc/status"
   "net"
   pb "servergrpc/model/sgrpc/model2"
   "time"
   _ "google.golang.org/grpc/balancer/grpclb"
   //_ "google.golang.org/grpc/balancer/rls/internal"
   _ "google.golang.org/grpc/xds"
)

```

代码说明：

-   第14行，将xds包导入到grpc服务器里，也就是将xds作为插件注册到了grpc服务器里

### 1.2.5、自定义解析器

在…/examples/features/name_resolving/client/main.go文件进行初始化:

```go
func init() {
   // Register the example ResolverBuilder. This is usually done in a package's
   // init() function.
   resolver.Register(&exampleResolverBuilder{})
}
```

grpc-go源码自带的测试用例中还有其他自定义解析器，这里就不一一列举了。

## 1.3、平衡构建器的注册

平衡构建器的注册是通过…/balancer/balancer.go文件中的Register函数来实现的。

```go
func Register(b Builder) {
   m[strings.ToLower(b.Name())] = b
}
```

注册时，需要传入一个构建器Builder，类型是接口：

```go
1．// Builder creates a balancer.
2．type Builder interface {
3．   // Build creates a new balancer with the ClientConn.
4．   // 在 balancer_conn_wrappers.go文件中的newCCBalancerWrapper方法里，调用此方法了
5．   Build(cc ClientConn, opts BuildOptions) Balancer

6．   // Name returns the name of balancers built by this builder.
7．   // It will be used to pick balancers (for example in service config).
8．   Name() string
9．}

```

主要代码说明：

-   第5行：为客户端连接器ClientConn创建一个平衡器
-   第8行：返还的是平衡器的名称；注册或者根据名称获取指定平衡器时使用的。

下面图片展示了当前grpc-go框架中内置的平衡构建器：  
![内置平衡构建器展示](https://img-blog.csdnimg.cn/20210511102851908.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

注意：

平衡构建器是用来创建平衡器的。

平衡器的创建是在客户端跟服务器端进行链接过程中创建的。

### 1.3.1、baseBuilder平衡构建器

在grpc-go/balancer/base/balancer.go文件中定义了baseBuilder平衡构建器。

```go
1．type baseBuilder struct {
2．   name          string
3．   pickerBuilder PickerBuilder
4．   config        Config
}
```

主要代码说明：

-   第2行：用来设置构建器的名称
-   第3行：这是一个picker构建器

看一下PickerBuilder源码：

在grpc-go/balancer/base/base.go文件中:

```go
1．// PickerBuilder creates balancer.Picker.
2．type PickerBuilder interface {
3．   // Build returns a picker that will be used by gRPC to pick a SubConn.
4．   Build(info PickerBuildInfo) balancer.Picker
}
```

主要代码说明：

-   第4行：创建一个balancer.Picker构建器；

**这个Picker到底是用来做什么呢？**

比方说，在某个场景下存在多个服务器端提供服务，客户端同时与这些服务器端建立起了链接，  
当客户端需要调用服务器端的服务时，需要从这些链接中根据平衡器策略选择一个链接进行服务调用。

（简单的说，就是picker需要从众多链接中选择一个进行帧的接收）

下面图片显示了grpc-go框架中内置实现的PickerBuilder：  
![PickerBuilder平衡构建器](https://img-blog.csdnimg.cn/20210511104412232.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)  
baseBuilder构建器是基础构建器，启动时不需要注册；

或者可以认为baseBuilder是一个平衡构建器模板，像下面的round_robin平衡构建器就是以这个模块为基础创建的。

### 1.3.2、pickFirst平衡构建器注册

在grpc-go/pickfirst.go文件中注册了pickFirst构建器：

```go
func init() {
   balancer.Register(newPickfirstBuilder())
}
```

其中，传入的构建器newPickerfirstBuilder()，代码如下:

```go
// PickFirstBalancerName is the name of the pick_first balancer.
const PickFirstBalancerName = "pick_first"

// 初始化Pickerfirst 构建器
func newPickfirstBuilder() balancer.Builder {
   return &pickfirstBuilder{}
}

type pickfirstBuilder struct{}
```

### 1.3.3、round_robin平衡器注册

在grpc-go/balancer/roundrobin/roundrobin.go文件中：

```go
1．// Name is the name of round_robin balancer.
2．const Name = "round_robin"

3．// newBuilder creates a new roundrobin balancer builder.
4．func newBuilder() balancer.Builder {
5．   return base.NewBalancerBuilder(Name, &rrPickerBuilder{}, base.Config{HealthCheck: true})
6．}

7．func init() {
8．   balancer.Register(newBuilder())
9．}

10．type rrPickerBuilder struct{}

```

主要代码说明：

-   第2行：设置平衡器的名称
-   第4-6行：创建round_robin平衡构建器函数，内部调用的是base.NewBalancerBuilder函数
-   第7-9行：将round_robin平衡器构建器注册到grpc服务器里
-   第10行：定义一个rrPickerBuilder结构体；很明显，前缀rr是round_robin的首字母缩写；在grpc-go框架中，有很多类似的情况。

接下来，看一下base.NewBalancerBuilder函数内部：

```go
1．func NewBalancerBuilder(name string, pb PickerBuilder, config Config) balancer.Builder {
2．      return &baseBuilder{
3．      name:          name,
4．      pickerBuilder: pb,
5．      config:        config,
6．   }
7．}
```

主要代码说明：

-   第2-6行：round_robin构建器内部调用的是baseBuilder构建器。

在实现平衡器的过程中最主要的就是要实现PickerBuilder，

如何众多链接中选择一个链接进行客户端跟服务器端的数据交换，也就是设置选择链接的策略。

不同的平衡器选择策略不同，如随机选择，始终选择第一个pickerFirst，或者根据链接的权重进行选择，或者轮询选择策略round_robin。

### 1.3.4、grpclb平衡器注册

在grpc-go/balancer/grpclb/grpclb.go文件中，进行了初始化:

```go
1．func init() {
2．   balancer.Register(newLBBuilder())
3．   dns.EnableSRVLookups = true
4．}

5．// newLBBuilder creates a builder for grpclb.
6．func newLBBuilder() balancer.Builder {
7．      return newLBBuilderWithFallbackTimeout(defaultFallbackTimeout)
8．}
9．func newLBBuilderWithFallbackTimeout(fallbackTimeout time.Duration) balancer.Builder {
10．   return &lbBuilder{
11．      fallbackTimeout: fallbackTimeout,
12．   }
13．}

14．type lbBuilder struct {
15．   fallbackTimeout time.Duration
16．}
```

主要代码说明：

-   第1-4行：注册lb构建器
-   第6-8行：创建lb构建器函数，内部调用的是newLBBuilderWithFallbackTimeout函数  
    需要说明的是，grpclb构建器在grpc启动时默认并没有启动，如果想要启动的话，需要显示的导入，如下所示：

在/grpc-go/examples/features/load_balancing/server/main.go测试用例中：

```go
1．package main

2．import (
3．   "context"
4．   "fmt"
5．   "github.com/CodisLabs/codis/pkg/utils/log"
6．   "net"
7．   "sync"

8．   "google.golang.org/grpc"

9．   _ "google.golang.org/grpc/balancer/grpclb"
10．   pb "servergrpc/examples/features/proto/echo"
11．)
```

主要代码说明：

-   第9行：将grpclb构建器显示的导入到服务器中

## 1.4、编解码器初始化

grpc-go框架中使用protoc作为默认的编解码器。

在grpc-go/encoding/proto/proto.go文件中：

```go
1．// Name is the name registered for the proto compressor.
2．const Name = "proto"

3．func init() {
4．   encoding.RegisterCodec(codec{})
5．}

6．// codec is a Codec implementation with protobuf. It is the default codec for gRPC.
7．type codec struct{}
```

主要代码说明：

-   第3-5行：注册编解码器codec

看一下，encoding.RegisterCodec函数内部:

```go
1．func RegisterCodec(codec Codec) {
2．   if codec == nil {
3．      panic("cannot register a nil Codec")
4．   }
5．   if codec.Name() == "" {
6．      panic("cannot register Codec with empty string result for Name()")
7．   }
8．   contentSubtype := strings.ToLower(codec.Name())
9．   registeredCodecs[contentSubtype] = codec
10．}
```

主要代码说明：

-   第2-7行：主要做一些校验工作。
-   第9行：将protoc注册到registeredCodecs容器里

注意：  
此函数并非线程安全的，在启动时如果存在多个相同名称的编码器注册的话，会以最后一个注册的编码器有效。

## 1.5、拦截器初始化

拦截器的初始化主要分为两大步骤：

### 1.5.1、自定义拦截器

例如：在grpc-go/examples/features/interceptor/server/main.go文件中：

```go
1．func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
2．   // authentication (token verification)
3．   md, ok := metadata.FromIncomingContext(ctx)
4．   if !ok {
5．      return nil, errMissingMetadata
6．   }
7．   if !valid(md["authorization"]) {
8．      return nil, errInvalidToken
9．   }
10．   m, err := handler(ctx, req)
11．   if err != nil {
12．      logger("RPC failed with error %v", err)
13．   }
14．   return m, err
15．}
```

主要代码说明：

-   第2-9行：在调用真正处理函数之前，要做的事情；比方说，打印日志之类的工作。
-   第10行：调用真正处理的函数，也就是客户端要调用的服务。 在后面分析拦截器的原理时再详细的说明。

### 1.5.2、将拦截器注册到服务器端

```go
1．func main() {
2．   flag.Parse()

3．   lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
4．   if err != nil {
5．      log.Fatalf("failed to listen: %v", err)
6．   }

7    s := grpc.NewServer(grpc.Creds(creds),grpc.UnaryInterceptor(unaryInterceptor), grpc.StreamInterceptor(streamInterceptor))

8．   // Register EchoServer on the server.
9．   pb.RegisterEchoServer(s, &server{})

10．   if err := s.Serve(lis); err != nil {
11．      log.Fatalf("failed to serve: %v", err)
12．   }
13．}
```

主要代码说明：

-   第7行：就是将拦截器注册到gprc服务器端

这里仅仅说明一下，grpc-go是如何注册拦截器的，后面的章节将详细的分享拦截器的原理。

# 2、服务器监听工作

## 2.1、grpc服务器是如何监听客户端的请求呢？

例如，在grpc-go/examples/helloworld/greeter_server/main.go文件中:

```go
1．func main() {
2．   lis, err := net.Listen("tcp", port)
3．   if err != nil {
4．      log.Fatalf("failed to listen: %v", err)
5．   }
6．   s := grpc.NewServer()
7．   pb.RegisterGreeterServer(s, &server{})
8．   if err := s.Serve(lis); err != nil {
9．      log.Fatalf("failed to serve: %v", err)
10．   }
11．}
```

主要代码说明：

-   第2行：创建一个监听目标，只监听tcp协议的请求，端口号是port
-   第6行：创建grpc服务器
-   第7行：将服务注册到grpc服务器里
-   第8行：启动grpc服务器对监听目标开始监听

进入方法Serve里：

在grpc-go/server.go文件里：(只显示了核心代码):

```go
1．func (s *Server) Serve(lis net.Listener) error {
2．   s.serve = true
3．   // 1、将监听 进行存储，true表示处于监听状态
4．   ls := &listenSocket{Listener: lis}
5．   for {
6．      rawConn, err := lis.Accept()
7．    
8．      go func() {
9．         s.handleRawConn(rawConn)
10．      }()
11．   }
12．}
```

在源码文件中，找到for循环即可。

主要代码说明：

-   第2行：表示grpc服务器端处于运行状态
-   第6行：阻塞方式监听客户端的请求，如果没有请求时会一直阻塞此处。
-   第8-10行：针对新的链接，grpc服务器开启一个协程处理客户端的链接。一个请求链接对应一个协程；

# 结尾
后面的章节再详细的介绍接收到客户端的请求后，grpc服务器端做了哪些事情。

本篇文章主要是分析了grpc服务器端启动后，都做了哪些事情；

这样的话，以后用到哪个组件时，就知道在什么地方进行的初始化，赋值等操作了。

其实很多框架都是类似的情形，如创建各种组件，初始化，注册，[监听](https://so.csdn.net/so/search?q=%E7%9B%91%E5%90%AC&spm=1001.2101.3001.7020)端口等；
