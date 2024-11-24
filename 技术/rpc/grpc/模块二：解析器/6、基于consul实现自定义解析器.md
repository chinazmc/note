#grpc 

前面的章节中，我们介绍了解析器的原理，如何注册解析器，如何实现一个解析器；

那么，本小节我们就尝试一下，自己开发一个解析器，解析器的[Scheme](https://so.csdn.net/so/search?q=Scheme&spm=1001.2101.3001.7020)定为consul;

[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)服务器启动时，自动将服务的相关信息注册到consul服务里；

grpc客户端通过[consul](https://so.csdn.net/so/search?q=consul&spm=1001.2101.3001.7020)客户端获取到服务的地址和端口信息；

其实，就是实现了服务注册和服务发现功能。

本节涉及到的代码，已经上传到百度网盘里了  
(链接: https://pan.baidu.com/s/1za02qnUII78n-XhlrLf7RA 密码: 3tok)

# 1、测试环境介绍：

下面的图表示了几个服务之间的关系：  
![grpc+consul+ 自定义解析器](https://img-blog.csdnimg.cn/20210530094403519.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70)

Mac物理机的IP：192.168.43.214

grpc客户端，grpc服务器端 运行在Mac上

虚拟机中consul服务的地址是：10.211.55.10:8500

主要流程：

-   grpc客户端首先向consul发起请求，consul://SayHello1
-   cosnul服务返还SayHello1服务的IP:Port
-   grpc客户端获取到SayHello1服务的IP:Port向grpc服务器端 发起底层tcp连接请求,rpc请求。

# 2、consul配置、启动

采用容器的方式启动consul服务；

consul的镜像：

```
[root@master ~]# docker images | grep consul
consul                          latest              2823bc69f80f        3 days ago          120MB
[root@master ~]#   
```

启动consul服务：

```go
docker run --name consul-test -it -p 8500:8500 -d consul:latest
```

# 3、grpc服务器端 代码

服务器端启动时，需要自动将服务的ip，端口号等信息注册到consul里：  
下面，仅仅列出了主要相关代码：

```go
1．func main() {
2．   lis, err := net.Listen("tcp", port)
3．   if err != nil {
4．      log.Errorf("failed to listen: %v", err)
5．   }

6．   s := grpc.NewServer()

7．   // 将helloworld_grpc.pb.go文件里_Greeter_serviceDesc声明的服务，注册到grpc服务里
8．   pb.RegisterGreeterServer(s, &server{})

9．   registerServerConsul()

10．   if err := s.Serve(lis); err != nil {
11．      log.Errorf("failed to serve: %v", err)
12．   }
13．}

14．// consul 服务注册
15．func registerServerConsul()  {
16．   // 创建consul客户端
17．   config := consulapi.DefaultConfig()
18．   config.Address = "10.211.55.10:8500"
19．   client, err := consulapi.NewClient(config)
20．   if err != nil {
21．      log.Errorf("consul client error : ", err)
22．   }

23．   registration := new(consulapi.AgentServiceRegistration)
24．   registration.ID = "lx-mbp"      // 服务节点的名称
25．   registration.Name = "SayHello1"      // 服务名称
26．   registration.Port = 50051              // 服务端口
27．   registration.Tags = []string{"SayHello1"} // tag，可以为空
28．   registration.Address = "192.168.43.214"      // 服务 IP

29．   // 服务注册
30．   err = client.Agent().ServiceRegister(registration)
31．   if err != nil {
32．      log.Errorf("register server consul error : ", err)
33．   }
34．}

```

registerServerConsul函数的目的：

  就是将提供SayHello1服务的IP，端口注册到consul服务里

主要代码说明：

-   第18行：配置consul服务的IP和端口
-   第24行：配置grpc服务器运行节点的名称
-   第25行：配置服务的名称
-   第26行：配置grpc服务的端口
-   第28行：配置grpc服务的IP
-   第30行：注册到consul服务里

# 4、grpc客户端

下面，仅仅列出了主要相关代码：

```go
1．package main

2．import (
3．  //---省略导入代码
4．)

5．const (
6．   address     = "consul:///SayHello1"
7．   defaultName = "----this is consul resolver test-------"
8．)
9．const (
10．   timestampFormat = time.StampNano // "Jan _2 15:04:05.000"
11．   streamingCount  = 10
12．)

13．func main() {
14．   reWG := &sync.WaitGroup{}
15．   for i:=0; i < 1;  i++ {
16．      reWG.Add(1)
17．      go request(reWG, int8(i))
18．   }

19．   reWG.Wait()
20．   log.Infof("---->结束啦-------")
21．}

22．func readText(path string) string {
23．   p, _ :=ioutil.ReadFile(path)

24．   return string(p)
25．}

26．func request(wg *sync.WaitGroup, index int8)  {
27．   // 创建ClientConn, 跟服务器address地址
28．   conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
29．   if err != nil {
30．      log.Errorf("did not connect: %v", err)
31．   }
32．   defer conn.Close()

33．   // 根据ClientConn, 创建GreeterClient
34．   c := pb.NewGreeterClient(conn)

35．   // Contact the server and print out its response.
36．   name := defaultName
37．   if len(os.Args) > 1 {
38．      name = os.Args[1]
39．   }
40．   ctx, cancel := context.WithTimeout(context.Background(), time.Second * 10 )
41．   defer cancel()

42．   for i :=0; i<1;i++{
43．      msg, err := c.SayHello1(ctx, &pb.HelloRequest{Name: name})
44．      if err != nil {
45．         log.Errorf("--->msg error\t: %v", err)
46．      }

47．      log.Infof("--->msg\t: %s", msg.GetMessage())
48．   }

49．   wg.Done()
50．}

51．const consulResolverScheme = "consul"

52．type consulResolverBuilder struct {}

53．func (*consulResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
54．   r := &consulResolver{
55．      target: target,
56．      cc:     cc,
57．   }

58．   // 真正的发起 链接
59．   r.start()

60．   return r, nil
61．}
62．func (*consulResolverBuilder) Scheme() string { return consulResolverScheme }

63．type consulResolver struct {
64．   target     resolver.Target
65．   cc         resolver.ClientConn
66．}

67．func (r consulResolver) start()  {
68．   // 创建consul客户端
69．   config := consulapi.DefaultConfig()
70．   // 配置consul服务的地址
71．   config.Address = "10.211.55.10:8500"
72．   client, err := consulapi.NewClient(config)
73．   if err != nil {
74．      log.Errorf("consul client error : %v", err.Error())
75．   }

76．   // 服务发现
77．   services, _, err := client.Health().Service("SayHello1", "SayHello1", true, &consulapi.QueryOptions{})
78．   if err != nil {
79．      log.Errorf("error retrieving instances from Consul: %v", err)
80． }

81．   // 将发现的地址services，转换成grpc框架中的resolver.Address结构体
82．   addrs := make([]resolver.Address, len(services))
83．   for index, service := range services {
84．      address := fmt.Sprintf("%s:%d", service.Service.Address, service.Service.Port)
85．      addrs[index] = resolver.Address{Addr: address}
86．   }

87．   // 更新解析器State，最终向grpc服务器端发起连接
88．   r.cc.UpdateState(resolver.State{Addresses: addrs})
89．}

90．func (*consulResolver) ResolveNow(o resolver.ResolveNowOptions) {}
91．func (*consulResolver) Close()                                  {}

92．func init() {
93．   resolver.Register(&consulResolverBuilder{})
94．}
```

在前面的章节中，我们介绍过实现一个解析器的话，需要实现两个接口：

Builder接口和Resolver接口；以及如何注册一个解析器。

因此，上面的代码中，我们定义了：

-   consulResolverBuilder 结构体实现了Builder接口，需要实现Build方法和Scheme方法
-   consulResolver结构体实现了Resolver接口，需要实现ResolveNow方法和Close方法
-   并且通过init函数实现了consul解析器的启动注册。

> 注意  
> 自定义的解析器consulResolver并没有解析器重试功能。

在start方法中，需要向consul服务器发送请求，获取服务SayHello1的后端服务器地址列表，

并且封装成resolver.Address，然后需要调用UpdateState方法，触发平衡器的流程，

最终向grpc服务器端发送tcp链接请求。

# 5、总结

在gRPC-go框架中，解析器的实现采用的是插件式的编程方式，

gRPC-go框架把主流规定好了，某些步骤是用接口实现的，

这种编程方式，有利于功能的扩展，可以将这种思路应用到自己的项目中去；

比方说，在我负责设计的容器级灰度升级系统中，涉及到多种灰度发布方案，

我只需要把灰度升级系统的整体流程设计好，中间的灰度发布方案采用接口编程方式即可。

只要有新的灰度发布方案实现我规定的接口，即可；  
   