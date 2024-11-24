#grpc 


  本小节主要是介绍一下round_robin类型的平衡器的基本原理；

其实，就是如何处理两个场景：

-   场景一：跟grpc服务器端链接的策略是什么，比方说同一个服务，可能是由多个grpc服务器端提供的；那么此时，grpc客户端的平衡器选择什么策略来连接grpc服务器端呢？这里的策略指的是，跟其中一个grpc服务器端相连接，还是全部连接，还是根据grpc服务器端的负载情况进行链接呢？等等
-   场景二：如果grpc客户端的平衡器(如round_robin平衡器)链接了多个grpc服务器地址，那么，在真正进行帧的传输时，应该如何从刚链接好的众多grpc服务器地址中选择一个tcp链接进行帧的传输呢？如,可以始终选择第一个链接进行传输数据，或者轮询的方式依次选择连接，或者随机选择连接等等

那么，接下来，主要是介绍上面说的情况，看看round_robin平衡器是如何处理的？

  round_robin平衡器，其实底层创建的是baseBalancer平衡器，round_robin平衡器的核心是按照某种策略获取某个子链接，也就是在Pick方法的实现上；  
# 1、分析入口UpdateClientConnState
同样，先从平衡器的入口进行分析：
[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)-go/balancer/base/balancer.go文件中的UpdateClientConnState方法：
或者  
可参考一下，调用链：
从客户端的main.go文件开始，方法的调用链如下所示：
```
main.go→grpc.Dial→DialContext→newCCResolverWrapper →rb.Build →r.start()→r.cc.UpdateState→UpdateState→ccr.cc.updateResolverState→bw.updateClientConnState→ccb.balancer.UpdateClientConnState
```

此时，选择的是baseBalancer类型的实现方式，即可进入下面的代码；
```go
1．func (b *baseBalancer) UpdateClientConnState(s balancer.ClientConnState) error {
2．	// TODO: handle s.ResolverState.ServiceConfig?
3．	if grpclog.V(2) {
4．		grpclog.Infoln("base.baseBalancer: got new ClientConn state: ", s)
5．	}
6．	// Successful resolution; clear resolver error and ensure we return nil.
7．	b.resolverErr = nil
8．	// addrsSet is the set converted from addrs, it's used for quick lookup of an address.
9．	addrsSet := make(map[resolver.Address]struct{})
10．	for _, a := range s.ResolverState.Addresses {
11．		addrsSet[a] = struct{}{}
12．		if _, ok := b.subConns[a]; !ok {
13．			// a is a new address (not existing in b.subConns).
14．			sc, err := b.cc.NewSubConn([]resolver.Address{a}, balancer.NewSubConnOptions{HealthCheckEnabled: b.config.HealthCheck})
15．			if err != nil {
16．				grpclog.Warningf("base.baseBalancer: failed to create new SubConn: %v", err)
17．				continue
18．			}
19．			b.subConns[a] = sc
20．			b.scStates[sc] = connectivity.Idle

21．			// 调用的是balancer_conn_wrapper.go文件中Connect方法
22．			// baseBalancer底层是 异步链接
23．			sc.Connect()
24．		}
25．	}
26．	for a, sc := range b.subConns {
27．		// a was removed by resolver.
28．		if _, ok := addrsSet[a]; !ok {
29．			b.cc.RemoveSubConn(sc)
30．			delete(b.subConns, a)
31．			// Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
32．			// The entry will be deleted in UpdateSubConnState.
33．		}
34．	}
35．	// If resolver state contains no addresses, return an error so ClientConn
36．	// will trigger re-resolve. Also records this as an resolver error, so when
37．	// the overall state turns transient failure, the error message will have
38．	// the zero address information.
39．	if len(s.ResolverState.Addresses) == 0 {
40．		b.ResolverError(errors.New("produced zero addresses"))
41．		return balancer.ErrBadResolverState
42．	}
43．	
44．	return nil
45．}
```

主要代码说明：

-   第2-25行： 将s.ResolverState.Addresses地址列表的所有地址依次向grpc服务器端进行链接；一个地址，对应一个链接；因此可能会出现多个链接。
-   第14行：创建子链接，具体可以参考前面的介绍
-   第19行：将创建好的链接sc, 注册到平衡器里
-   第20行：设值子链接的状态，链接前的状态是IDLE
-   第23行：向grpc服务器端发起链接，具体可以参考前面的介绍
-   第26-34行：更新平衡器中的子链接集b.subConns，移除没用的子链接，或者说只保留s.ResolverState.Addresses地址列表里对应的子链接
-   第39-42行：做个校验，地址数为0的情况下，返还 balancer.ErrBadResolverState错误；ccResolverWrapper结构体的poll方法里，可以根据解析器的错误类型，来判断是否进行轮询操作，进行重新解析；

从上面的代码中，可以看出来：

round_robin类型的平衡器跟grpc服务器端的连接策略是：

从解析器中获得的grpc服务器端的地址列表进行全部连接，也就是建立起多个tcp连接链路。  
   
# 2、在round_robin平衡器中，如何从众多的tcp链路中选择一个链路进行帧的传输呢？即Picker的原理？

## 2.1、round_robin平衡器中，是如何创建Picker的？

在grpc-go/balancer/roundrobin/roundrobin.go文件中：

```go
1．const Name = "round_robin"

2．// newBuilder creates a new roundrobin balancer builder.
3．func newBuilder() balancer.Builder {
4．   return base.NewBalancerBuilder(Name, &rrPickerBuilder{}, base.Config{HealthCheck: true})
5．}
```

主要是看一下第4行中，生成平衡构建器的参数中，传入的是rrPickerBuilder结构体的Picker构建器；

至于，如何注册一个round_robin类型的平衡器，可以参考<<[grpc服务器端启动时都做了哪些事情](https://blog.csdn.net/u011582922/article/details/116643989)>>文章中的1.3.3小节；

## 2.2、如何从众多已经创建好的tcp连接中，选择一个tcp连接，进行帧的传输呢？

那么，直接进入grpc-go/balancer/roundrobin/roundrobin.go中的Pick方法：

```go
1．func (p *rrPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
2． 	p.mu.Lock()

3． 	next := p.next
4．	
5． 	sc := p.subConns[next]
6． 	p.next = (next + 1) % len(p.subConns)
7． 	p.mu.Unlock()

8． 	return balancer.PickResult{SubConn: sc}, nil
9．}
```

该方法的核心:

就是按照某种策略获取子链接，供给流使用，进行帧的传输，就是传输头帧，数据帧，设置帧等；

主要代码说明：

-   第3-5行：根据当前下标next值从子链接集p.subConns中获取一个子链接
-   第6行：其实，就是提供了一种策略，也可以说是一种算法，目的是计算出下次使用tcp链接的下标；这是轮询策略，依次选择tcp链接；
-   第8行：返回tcp链接，在流的阶段，就可以通过这个链接进行帧的传输；  
       
     

# 3、总结

  到目前为止，已经介绍了

-   如何注册round_robin平衡器，
-   如何构建round_robin平衡器对象实例，
-   如何创建子链接，
-   如何向grpc服务端进行链接，
-   如何从众多的子链接中，获取一个链接供流使用，

round_robin平衡器的大部分内部都介绍完了；