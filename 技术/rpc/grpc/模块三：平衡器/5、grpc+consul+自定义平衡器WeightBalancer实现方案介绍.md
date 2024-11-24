#grpc 

  在前文我们已经介绍了如何实现一个平衡器，那么本节我们将尝试自定义一个平衡器；

该平衡器的核心目的是：

根据子链接的权重来选择已经创建好的rpc链接，用来传输各种类型的帧，即rpc请求。

# 1、整体流程介绍

[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)+weight-balancer的整体处理流程，如下图所示：  
![grpc+weight-balance](https://img-blog.csdnimg.cn/20210605092356458.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

环境说明：  
在Mac上：

-   3个grpc服务器端
-   1个grpc客户端
-   在虚拟机里启动1个docker方式运行的consul服务

主要流程如下：

-   grpc服务器端启动后，向consul进行注册，并且添加上自己的权重大小；
-   grpc客户端使用自定义consul解析器，向consul服务发起请求，获得grpc服务端的地址信息列表
-   consul解析器更新客户端状态时，会触发自定义平衡器weight_balancer
-   weight平衡器根据获取的grpc服务器端地址列表，全部向grpc服务器端进行连接
-   grpc流客户端根据平衡器的策略，如权重大小，选择其中的一个子链接，进行帧的传输；比方说红色实线。

# 2、consul服务启动说明

[consul](https://so.csdn.net/so/search?q=consul&spm=1001.2101.3001.7020)服务是采用docker方式运行的；  
提供一个启动脚本:

```go
#!/bin/bash
docker stop consul-grpc
docker rm consul-grpc
docker run --name consul-grpc -it -p 8500:8500 -d consul:latest 
```

# 3、grpc服务器端测试用例

由于grpc服务器端需要向consul进行注册，因此，需要添加了consul客户端；

```go
1．package main

2．import (
3．	"context"
4．	"fmt"
5．	"github.com/CodisLabs/codis/pkg/utils/log"
6．	"google.golang.org/grpc"
7．	_ "google.golang.org/grpc/balancer/grpclb"
8．	//_ "google.golang.org/grpc/encoding/gzip"
9．	"google.golang.org/grpc/codes"
10．	"google.golang.org/grpc/metadata"
11．	"google.golang.org/grpc/status"
12．	"net"
13．	pb "servergrpc/model/sgrpc/model2"
14．	"time"
15．	consulapi "github.com/hashicorp/consul/api"
16．)

17．const (
18．	port = ":50051"
19．	port2 = 50051
20．	consulServer = "10.211.55.10:8500"
21．	ip = "192.168.43.214"
22．)

23．const (
24．	timestampFormat = time.StampNano
25．	streamingCount  = 10
26．)
27．// server1 is used to implement helloworld.GreeterServer.
28．type server struct {
29．	pb.UnimplementedGreeterServer
30．}

31．// SayHello implements helloworld.GreeterServer
32．func (s *server) SayHello1(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
33．	return &pb.HelloReply{Message: "---From Server---50051---" + in.GetName()}, nil
34．}

35．func main() {
36．	lis, err := net.Listen("tcp", port)
37．	if err != nil {
38．		log.Errorf("failed to listen: %v", err)
39．	}

40．	s := grpc.NewServer()

41．	// 将helloworld_grpc.pb.go文件里_Greeter_serviceDesc声明的服务，注册到grpc服务里
42．	pb.RegisterGreeterServer(s, &server{})

43．	registerServerConsul()

44．	if err := s.Serve(lis); err != nil {
45．		log.Errorf("failed to serve: %v", err)
46．	}
47．}

48．// consul 服务注册
49．func registerServerConsul()  {
50．	// 创建consul客户端
51．	config := consulapi.DefaultConfig()
52．	config.Address = consulServer
53．	client, err := consulapi.NewClient(config)
54．	if err != nil {
55．		log.Errorf("consul client error : ", err)
56．	}

57．	registration := new(consulapi.AgentServiceRegistration)
58．	registration.ID = fmt.Sprintf("%s:%d", ip, port2)
59．	registration.Name = "SayHello1"      // 服务名称
60．	registration.Port = port2           // 服务端口
61．	registration.Tags = []string{"SayHello1"} // tag，可以为空
62．	registration.Address = ip     // 服务 IP
63．	// https://www.consul.io/docs/discovery/services  
64．	registration.Meta = map[string]string{
65．		"weight": "1",
66．	}

67．	// 服务注册
68．	err = client.Agent().ServiceRegister(registration)
69．	if err != nil {
70．		log.Errorf("register server consul error : ", err)
71．	}
72．}

```

主要代码说明：

-   第49-72行：注册信息到consul里
-   第64-66行：使用Meta来存储权重信息；客户端从Meta里获取权重信息即可。

分别设置端口是50051的服务的权重是1，端口50052对应的权重是3，端口50053对应的权重是9；  
下图是consul里注册的SayHello1服务：  
![grpc-weight-balance-consul](https://img-blog.csdnimg.cn/20210605093123708.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)  
查看端口是50052的权重信息：如下图所示：  
![grpc-weight-balance](https://img-blog.csdnimg.cn/20210605093219372.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

# 4、grpc客户端测试用例

在grpc客户端里，创建了自定义consul解析器，weight平衡器；  
grpc客户端测试用例代码结构，如下图中的红色框:  
![grpc-weight-balance代码结构](https://img-blog.csdnimg.cn/202106050933279.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

## 4.1、启动文件main.go代码：

```go
1．package main

2．import (
3．	"context"
4．	"github.com/CodisLabs/codis/pkg/utils/log"
5．	"io/ioutil"
6．	"sync"

7．	"os"
8．	"time"

9．	pb "grpcClientService/model/sgrpc/model2"
10．	"google.golang.org/grpc"
11．	"google.golang.org/grpc/resolver"
12．	consulapi "github.com/hashicorp/consul/api"
13．	"fmt"
14．	"google.golang.org/grpc/attributes"
15．	_ "grpcClientService/balancer/custom-balancer/custombalancer"

16．)
17．const (
18．	address     = "consul:///SayHello1"
19．	consulServer = "10.211.55.10:8500"
20．	defaultName = "----this is consul resolver test-------"
21．)
22．const (
23．	timestampFormat = time.StampNano // "Jan _2 15:04:05.000"
24．	streamingCount  = 10
25．)

26．func main() {
27．	reWG := &sync.WaitGroup{}
28．	for i:=0; i < 1;  i++ {
29．		reWG.Add(1)
30．		go request(reWG, int8(i))
31．	}

32．	reWG.Wait()
33．}

34．func request(wg *sync.WaitGroup, index int8)  {
35．	// 创建ClientConn, 跟服务器address地址
36．	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock(), grpc.WithBalancerName("weight_balancer"))
37．	if err != nil {
38．		log.Errorf("did not connect: %v", err)
39．	}
40．	defer conn.Close()

41．	// 根据ClientConn, 创建GreeterClient
42．	c := pb.NewGreeterClient(conn)

43．	// Contact the server and print out its response.
44．	name := defaultName
45．	if len(os.Args) > 1 {
46．		name = os.Args[1]
47．	}
48．	ctx, cancel := context.WithTimeout(context.Background(), time.Second * 10 )
49．	defer cancel()

50．	for i :=0; i<20;i++{
51．		msg, err := c.SayHello1(ctx, &pb.HelloRequest{Name: name})
52．		if err != nil {
53．			log.Errorf("--->msg error\t: %v", err)
54．		}

55．		log.Infof("--->msg\t: %s", msg.GetMessage())
56．	}

57．	wg.Done()
58．}

59．const consulResolverScheme = "consul"

60．type consulResolverBuilder struct {}

61．func (*consulResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
62．	r := &consulResolver{
63．		target: target,
64．		cc:     cc,
65．	}

66．	// 真正的发起 链接
67．	r.start()

68．	return r, nil
69．}
70．func (*consulResolverBuilder) Scheme() string { return consulResolverScheme }

71．type consulResolver struct {
72．	target     resolver.Target
73．	cc         resolver.ClientConn
74．}

75．func (r consulResolver) start()  {
76．	// 创建consul客户端
77．	config := consulapi.DefaultConfig()
78．	config.Address = consulServer
79．	client, err := consulapi.NewClient(config)
80．	if err != nil {
81．		log.Errorf("consul client error : %v", err.Error())
82．	}

83．	// 服务发现
84．	services, _, err := client.Health().Service("SayHello1", "SayHello1", true, &consulapi.QueryOptions{})
85．	if err != nil {
86．		log.Errorf("error retrieving instances from Consul: %v", err)
87．	}

88．	// 将发现的地址，转换成grpc框架中的resolver.Address结构体
89．	addrs := make([]resolver.Address, len(services))
90．	// service 遍历
91．	for index, service := range services {
92．		weight := service.Service.Meta["weight"]
93．		address := fmt.Sprintf("%s:%d", service.Service.Address, service.Service.Port)
94．		addrs[index] = resolver.Address{Addr: address, Attributes: attributes.New("weight", weight)}
95．	}

96．	// 更新解析器State，最终向grpc服务器端发起连接
97．	r.cc.UpdateState(resolver.State{Addresses: addrs})
98．}

99．func (*consulResolver) ResolveNow(o resolver.ResolveNowOptions) {}
100．func (*consulResolver) Close()                                  {}

101．func init() {
102．	resolver.Register(&consulResolverBuilder{})
103．}
```

主要代码说明：

-   第36行：指定使用consul解析器，使用weight_balancer平衡器
-   第59-103行：实现一个consul类型的自定义解析器
-   第77-84行：从consul服务里获取grpc服务端的地址列表；
-   第91-95行：将地址列表转换成grpc框架内部的结构体；注意，这里将获取到的权重信息，存储到结构体resolver.Address里的attributes属性里；这样的话，构建平衡器时，就可以从resolver.Address获取相应的信息。
-   第77行：触发平衡器，真正的向grpc服务器端发起连接

好，接下来，看一下自定义weight平衡器：

## 4.2、自定义平衡器weight-balance代码

这里我主要参考的是grpc-go/balancer/roundrobin/roundrobin.go文件中的round_robin平衡器:

```go
1．// Package roundrobin defines a roundrobin balancer. Roundrobin balancer is
2．// installed as one of the default balancers in gRPC, users don't need to
3．// explicitly install this balancer.
4．// 自定义平衡器
5．package custombalancer

6．import (
7．	"github.com/CodisLabs/codis/pkg/utils/log"
8．	"sync"
9．	"reflect"
10．	"encoding/json"

11．	"google.golang.org/grpc/balancer"
12．	"google.golang.org/grpc/balancer/base"
13．	"strconv"
14．	"math/rand"
15．	"time"
16．)

17．// Name is the name of round_robin balancer.
18．const Name = "weight_balancer"

19．// newBuilder creates a new weight-balancer builder.
20．func newBuilder() balancer.Builder {
21．		return base.NewBalancerBuilder(Name, &weightPickerBuilder{}, base.Config{HealthCheck: true})
22．}

23．func init() {
24．	balancer.Register(newBuilder())
25．}

26．type weightPickerBuilder struct{}

27．func (*weightPickerBuilder) Build(info base.PickerBuildInfo) balancer.Picker {
28．	if len(info.ReadySCs) == 0 {
29．		return base.NewErrPicker(balancer.ErrNoSubConnAvailable)
30．	}
31．	var scs []balancer.SubConn
32．	// 下面map的遍历形式是，获取的是key
33．	for subConn, subConnInfo := range info.ReadySCs {
34．		addr := subConnInfo.Address
35．		weightInterface := addr.Attributes.Value("weight")
36．		weight := reflect.ValueOf(weightInterface).String()
37．		w2 , _ := strconv.Atoi(weight)
38．		for i :=0; i < w2; i++{
39．			scs = append(scs, subConn)
40．		}

41．	}

42．	return &weightPicker{
43．		subConns: scs,
44．		next: rand.New(rand.NewSource(time.Now().UnixNano())).Intn(len(scs)),
45．	}
46．}

47．type weightPicker struct {
48．	// subConns is the snapshot of the roundrobin balancer when this picker was
49．	// created. The slice is immutable. Each Get() will do a round robin
50．	// selection from it and return the selected SubConn.
51．	subConns []balancer.SubConn
52．	mu   sync.Mutex
53．	next int
54．}

55．/**
56．	在 picker_wrapper.go文件中的pick方法调用
57． */
58．func (p *weightPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
59．	p.mu.Lock()
60．	next := p.next
61．	sc := p.subConns[next]

62．	p.next = (next + 1) % len(p.subConns)
63．	p.mu.Unlock()
64．	
65．	return balancer.PickResult{SubConn: sc}, nil
66．}
```

主要代码说明：

-   第19-21行：构建一个weight-balancer 类型的平衡构建器；注意，底层其实调用的是base.NewBalancerBuilder构建器
-   第23-25行：启动时，自动注册weight-balancer构建器
-   第27-46行：返还一个基于权重策略的Picker构建器
    -   a)第28-30行：校验当前状态是Ready的数量，等于的0时，就报错
    -   b)第31-41行：就是从Attributes获取权重信息，根据权重的大小，来初始化scs 数组；比方说子链接1，对应的权重是1，那么在scs里，只有一个子链接1；子链接2，对应的权重是3，那么在scs里，有3个子链接2；子链接3，对应的权重是9，那么在scs里就会有9个子链接3；
    -   c)第42-45行：返还一个Picker权重；注意：这里的next的值，是随机生成的；
-   第58-66行：根据next值，从p.subConns数组里获取一个子链接；并且通过第62行计算，重新赋值next值（轮询获取），用来表明，下次获取子链接的下标值。

# 5、总结

  本篇文章主要介绍了，如何自定义一个平衡器，提供了相应的测试用例；使用consul服务作为服务注册中心，使用自定义的weight-balance平衡器来发起grpc链接，根据grpc服务注册的权重来选择合适的子链接供后续的帧的传输使用。

  解析器、平衡器都是通过各自的注册函数，将各自的构建器Build注册到grpc框架里，然后在真正的用到解析器，平衡器的时候，才创建。(单例模式中的懒汉式是不是也是这样的)

  至此，平衡器的相关原理介绍，相关测试用例基本介绍完了。  
   