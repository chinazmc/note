#grpc 

pickFirst平衡器主要做了两件事:：

-   跟grpc服务器建立rpc链接
-   从链接中选择第一个已经建立好的链接给流使用，进行帧的传输

先看一下平衡器pickfirstBalancer[结构体](https://so.csdn.net/so/search?q=%E7%BB%93%E6%9E%84%E4%BD%93&spm=1001.2101.3001.7020)都有哪些属性：

```go
1.type pickfirstBalancer struct {
2.  	state  connectivity.State
3. 	    cc    balancer.ClientConn
4. 	    sc    balancer.SubConn
5.}
```

主要代码说明：

-   第2行：表示平衡器的状态，如IDLE、Connecting、Ready、TransientFailure、Shutdown
-   第3行：balancer.ClientConn类型是接口，
    -   主要是负责子链接SubConn的创建，移除，更新客户端链接器ClientConn的状态等；
    -   ccBalancerWrapper结构体实现了该接口；
-   第4行：balancer.SubConn类型是接口，主要是负责具体向grpc服务端发起链接等；acBalancerWrapper结构体实现了该接口；

直接进入平衡器的入口:

# 1、分析入口UpdateClientConnState

可参考一下，调用链：  
从客户端的main.go文件开始，方法的调用链如下所示：

```
main.go→grpc.Dial→DialContext→newCCResolverWrapper →rb.Build →r.start()→r.cc.UpdateState→UpdateState→ccr.cc.updateResolverState→bw.updateClientConnState→ccb.balancer.UpdateClientConnState
```

或者：

直接进入[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)-go/pickfirst.go文件中UpdateClientConnState方法里：

```go
1. func (b *pickfirstBalancer) UpdateClientConnState(cs balancer.ClientConnState) error {
2.   if len(cs.ResolverState.Addresses) == 0 {
3.      b.ResolverError(errors.New("produced zero addresses"))
4.      return balancer.ErrBadResolverState
5.   }
6.   if b.sc == nil {
7.      var err error
8.      //cs.ResolverState.Addresses: {localhost:50051  <nil> 0 <nil>}
9.      b.sc, err = b.cc.NewSubConn(cs.ResolverState.Addresses, balancer.NewSubConnOptions{})
10.      if err != nil {
11.         if grpclog.V(2) {
12.            grpclog.Errorf("pickfirstBalancer: failed to NewSubConn: %v", err)
13.         }
14.         b.state = connectivity.TransientFailure
15.         b.cc.UpdateState(balancer.State{ConnectivityState: connectivity.TransientFailure,
16.            Picker: &picker{err: fmt.Errorf("error creating connection: %v", err)},
17.         })
18.         return balancer.ErrBadResolverState
19.      }
20.      b.state = connectivity.Idle
21.      b.cc.UpdateState(balancer.State{ConnectivityState: connectivity.Idle, Picker: &picker{result: balancer.PickResult{SubConn: b.sc}}})
22.      // 开始链接
23.      b.sc.Connect()
24.   } else {
25.      b.sc.UpdateAddresses(cs.ResolverState.Addresses)
26.      b.sc.Connect()
27.   }
28.
29.   return nil
30.}

```

UpdateClientConnState这个方法的名称，至少会对我产生异议，

从名称中感觉是更新ClientConn的状态，State第一感觉应该是Idle, connecting, ready之类的值，但实际上不是；

先简单的说一下，这段代码实现了什么功能？

-   第2-5行：主要是做校验工作，先判断是否有链接的地址，或者说解析器是否正确解析地址了。
-   第6-27行：核心思想就是，先判断是否已经创建链接了，
    -   如果没有的话就创建一个新的链接，并向grpc服务端进行链接connect；
    -   如果已经链接过了，就更新地址，满足条件的话，也需要重新链接。

再详细的介绍代码：

-   第1行：参数balancer.ClientConnState的主要属性结构，如下：  
    ![平衡器ClientConnState主要属性结构](https://img-blog.csdnimg.cn/20210531211958776.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)  
    从上面的图，可以看出参数cs主要有哪些属性。
-   第9行：为sc.ResolverState.Addresses创建一个子链接
-   第10-19行：创建子链接失败时的处理逻辑；主要是更新平衡器的状态，更新连接器的状态
-   第20行：在子链接创建成功的情况下，更新平衡器的状态为IDLE
-   第21行：更新连接器ClientConn的状态为IDLE，
    -   注意，这里初始化了Picker；
    -   并且，告诉Picker，在创建流的时候，进行pick操作时，返还的子链接就是第9行创建的子链接；
    -   让流通过这个子链接，传输帧；
-   第23行：通过子链接sc去调用connect方法，最终实现向grpc服务器进行链接。
-   第25-26行：如果平衡器中子链接sc已经初始化过了的话，就需要重新更新addresses，需要校验是不是需要重新向grpc服务器端进行链接

# 2、如何创建子链接实例？

接下来，再返回到第9行，看看是如何创建子链接的：

b.cc的类型是接口，ccBalancerWrapper结构体实现了该接口，

好，直接进入ccBalancerWrapper结构体的NewSubConn方法里(grpc-go/balancer_conn_wrappers.go):

```go
1．func (ccb *ccBalancerWrapper) NewSubConn(addrs []resolver.Address, opts balancer.NewSubConnOptions) (balancer.SubConn, error) {
2．   if len(addrs) <= 0 {
3．      return nil, fmt.Errorf("grpc: cannot create SubConn with empty address list")
4．   }
5．   ccb.mu.Lock()
6．   defer ccb.mu.Unlock()
7．
8．   if ccb.subConns == nil {
9．      return nil, fmt.Errorf("grpc: ClientConn balancer wrapper was closed")
10．   }
11．
12．   // 创建addrConn
13．   ac, err := ccb.cc.newAddrConn(addrs, opts)
14．
15．   if err != nil {
16．      return nil, err
17．   }
18．
19．   // ac,应该就是 addrConn
20．   acbw := &acBalancerWrapper{ac: ac}
21．
22．   acbw.ac.mu.Lock()
23．   ac.acbw = acbw
24．   acbw.ac.mu.Unlock()
25．   ccb.subConns[acbw] = struct{}{}
26．
27．   return acbw, nil
28．}
```

主要目的说明:
创建子链接，形成子包装类acBalancerWrapper，并将子包装类注册到平衡器包装类ccBalancerWrapper里

主要代码说明：
-   第2-10行：主要是做些校验性的工作
-   第13行：用平衡器创建子链接，其实最终是调用客户端链接器ClientConn结构体的newAddrConn方法实现的
-   第20行：为addrConn结构体构建一个包装类acBalancerWrapper
-   第25行：将包装类acBalancerWrapper注册到包装类ccBalancerWrapper里

到目前为止，

已经知道NewSubConn方法如何创建了子链接，其实就是调用客户端链接器ClientConn结构体的newAddrConn方法实现的

因此，返还到grpc-go/pickfirst.go文件中UpdateClientConnState方法里：

# 3、在pickfirst平衡器下，选择子链接的策略？也就是如何创建Picker?

继续看第15行：(就是下面的代码)

```go
 b.cc.UpdateState(balancer.State{ConnectivityState: connectivity.Idle, Picker: &picker{result: balancer.PickResult{SubConn: b.sc}}})
```

在pickfirst平衡器下，picker选择连接的策略就是:

直接返回第一次链接好的tcp链接b.sc，即可。

直接进入grpc-go/balancer_conn_wrappers.go文件中的UpdateState方法里:

```go
1．func (ccb *ccBalancerWrapper) UpdateState(s balancer.State) {
2．		 ccb.mu.Lock()
3．	    defer ccb.mu.Unlock()
4．	    if ccb.subConns == nil {
5．			return
6．	    }
7．  	ccb.cc.blockingpicker.updatePicker(s.Picker)
8．  	ccb.cc.csMgr.updateState(s.ConnectivityState)
9．}
```

主要代码说明:

-   第7行：就是将最新的Picker赋值给客户端连接器ClientConn中的blockingpicker属性
-   第8行：就是将最新的链接状态赋值给客户端连接器ClientConn中的csMgr属性

> 注意：  
>   
>   最好是先更新Picker，然后再更新链接状态。

# 4、开始子链接

返还到grpc-go/pickfirst.go文件中UpdateClientConnState方法里：

继续看第23行：(就是下面的代码)

```go
b.sc.Connect()
```

继续进入到grpc-go/balancer_conn_wrappers.go文件中的Connect方法里:

```go
func (acbw *acBalancerWrapper) Connect() {
	acbw.mu.Lock()
	defer acbw.mu.Unlock()

	// 底层 调用的是http2_client.go文件中的newHTTP2Client方法
	acbw.ac.connect()
}
```

继续进入到[rpc](https://so.csdn.net/so/search?q=rpc&spm=1001.2101.3001.7020)-go/clientconn.go文件中connect方法里:

```go
1．func (ac *addrConn) connect() error {
2．   ac.mu.Lock()
3．	  if ac.state == connectivity.Shutdown {
4．		ac.mu.Unlock()
5．		return errConnClosing
6．	  }

7．	  // 只处理状态为idle的
8．	  if ac.state != connectivity.Idle {
9．		 ac.mu.Unlock()
10．	 return nil
11．  }
12．	// Update connectivity state within the lock to prevent subsequent or
13．	// concurrent calls from resetting the transport more than once.
14．	// 就是将connecting写入SubConn的balance.Unbound里
15．	ac.updateConnectivityState(connectivity.Connecting, nil)
16．	ac.mu.Unlock()

17．	// Start a goroutine connecting to the server asynchronously.
18．	go ac.resetTransport()

19．	return nil
20．}
```

主要代码说明:

-   第3-11行：主要是对addrConn的状态进行校验，达到的效果就是，如果当前的链接状态不符合要求的话，就退出，不允许进行链接
-   第15行：将链接状态更新为connectivity.Connecting
-   第18行: 启动一个协程专门负责具体的链接；这里就不在详细讲解了，相关章节有介绍该方法resetTransport

也就说，到目前pickerFirst平衡器主要做了以下几个事情::

-   通过调用客户端连接器ClientConn的newAddrConn方法创建子链接SubConn, 也就是创建了addrConn结构体
-   更新了客户端连接器ClientConn中的Picker以及状态
-   启动协程，专门负责向grpc服务器端进行链接。

# 5、总结

到目前为止，

-   介绍了pickFirst平衡器如何注册，
-   如何构建pickFirst平衡器对象实例，
-   如何创建子链接，
-   如何创建了Picker，
-   如何通过子链接向grpc服务器端进行链接；

涉及pickFirst平衡器的原理分享完成了。  