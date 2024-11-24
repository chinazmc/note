#grpc 


本篇文章主要分析一下，在[rpc](https://so.csdn.net/so/search?q=rpc&spm=1001.2101.3001.7020)链接建立过程中，如果链接失败了，
grpc客户端会不会重试链接，链接的间隔时间是如何计算的？

# 1、grpc客户端跟grpc服务器端建立连接失败后，是否有重试机制？

分析入口是[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)-go/clientconn.go文件中的resetTransport方法：

```go
1．func (ac *addrConn) resetTransport() {
2．   for i := 0; ; i++ {
3．      if i > 0 {
4．         ac.cc.resolveNow(resolver.ResolveNowOptions{})
5．      }
6．      ac.mu.Lock()
7．      if ac.state == connectivity.Shutdown {
8．         ac.mu.Unlock()
9．         return
10．      }

11．      addrs := ac.addrs
12．      backoffFor := ac.dopts.bs.Backoff(ac.backoffIdx)
13．      dialDuration := minConnectTimeout
14．      if ac.dopts.minConnectTimeout != nil {
15．         dialDuration = ac.dopts.minConnectTimeout()
16．         if dialDuration < backoffFor {
17．         // Give dial more time as we keep failing to connect.
18．         dialDuration = backoffFor
19．      }
20．   
21．      connectDeadline := time.Now().Add(dialDuration)

22．      ac.updateConnectivityState(connectivity.Connecting, nil)
23．      ac.transport = nil
24．      ac.mu.Unlock()
25．      newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)
26．      if err != nil {
27．         ac.mu.Lock()
28．         if ac.state == connectivity.Shutdown {
29．            ac.mu.Unlock()
30．            return
31．         }
32．         ac.updateConnectivityState(connectivity.TransientFailure, err)
33．         b := ac.resetBackoff
34．         ac.mu.Unlock()

35．         timer := time.NewTimer(backoffFor)
36．         select {
37．         case <-timer.C:
38．            ac.mu.Lock()
39．            ac.backoffIdx++
40．            ac.mu.Unlock()
41．         case <-b:
42．            timer.Stop()
43．         case <-ac.ctx.Done():
44．            timer.Stop()
45．            return
46．         }
47．        continue
48．      }
49．      ac.mu.Lock()
50．      if ac.state == connectivity.Shutdown {
51．         ac.mu.Unlock()
52．         newTr.Close()
53．         return
54．      }
55．      ac.curAddr = addr
56．      ac.transport = newTr
57．      ac.backoffIdx = 0

58．      hctx, hcancel := context.WithCancel(ac.ctx)
59．      ac.startHealthCheck(hctx)
60．      ac.mu.Unlock()

61．      <-reconnect.Done()
62．      hcancel()
63．   }
64．}
```

主要代码以及原理说明：

-   第2行：for循环，循环链接次数没有限制；因此，有一个很重要的问题，如何退出循环呢？
-   第3-5行：若i大于0，也就是说明，第一次链接可能失败了，需要重新尝试链接；
    -   第4行：想达到的目的是，调用解析器重新对用户设置的链接地址解析；(刚才链接失败，可能是链接地址的问题，因此，考虑重新解析一次，但是此方法是由每个解析器自己实现的，但是此方法并不是必须实现的，有些自定义解析器，可能没有真正实现)
-   第7-10行：链接前，先进行状态的校验，若状态为shutdown，就不再链接，直接退出
-   第12行：链接失败后，需要进行重试，计算出需要等待多长时间才能重试。backoffFor就是需要等待的时间；ac.backoffIdx表示第几次重试
-   第13-21行：计算出本次向grpc服务器端尝试建立Tcp链接的最长时间，超过这个时间还没有链接成功，就主动断开，等待尝试下一次的链接。
-   第22行：更新结构体addrConn的状态为Connecting，以及将平衡器、ClientConn的状态设置为Connecting
-   第25行：具体的链接操作，不再详细说明。
-   第26-48行：向服务器端建立链接失败后的处理逻辑。
    -   第28-31行：对状态进行校验，若当前链接的状态是Shutdown，直接退出，不会再次进行尝试链接了
    -   第32行：更新平衡器、addrConn、ClientConn状态为TransientFailure
    -   第35-46行：time跟select组成多路复用定时器：
        -   定时的超时时间就是前面计算的backoffFor
        -   通道一,timer.C: 超时结束后，重试次数累加1，继续重新进行链接；(注意，如果走通道一的话，就是说明，再进行下次链接的时候，需要等待一段时间)
        -   通道二,b: 不在等待超时时间，定时器立马结束，继续重新进行链接
        -   通道三,ac.ctx.Done(): 上下文结束，定时器立马结束，注意此时不在进行重试链接了。  
             模拟实验：可以不启动grpc服务器端，只启动grpc客户端，就会发现前面链接失败后，就会马上进行重新链接，越到后面，链接失败后，等待重新链接的时间会越长。
-   第50-64行：链接成功后的处理逻辑：
    -   第50-54行：对状态进行校验，如果是shutdown的话，就需要关闭新创建的链接，并且退出循环控制，不在进行重新链接了
    -   第55-57行：初始化操作
    -   第59行：调用健康校验；具体逻辑可参考相关章节。
    -   第61行：执行到这里说明链接成功了，会阻塞在这里，直到链接关闭；
    -   第62行：上下文取消功能。达到的效果就是，给健康检查发送信号，告诉健康检查链接关闭了，健康检查需要作出相应的处理。

然后，进入下一次的for循环，根据当前的链接状态来判断是否进行下一次的链接。

# 2、grpc框架是如何计算出下一次重试链接的等待时间？

接下来，我们分析一下重试链接时，等待的时长是如何计算出来的？

点击Backoff进入grpc-go/internal/backoff/backoff.go文件的Backoff接口，有以下实现：

grpc框架内置了3种类型的Backoff:

-   Exponential
-   backoffForever
-   noBackoff

目前有三个实现，到底是用哪个结构体呢？

进入grpc-go/clientconn.go文件的DialContext方法，通过下面的语句进行初始化：

```go
if cc.dopts.bs == nil {
   cc.dopts.bs = backoff.DefaultExponential
}
```

点击DefaultExponential，进入grpc-go/internal/backoff/backoff.go文件：

```go
var DefaultExponential = Exponential{Config: grpcbackoff.DefaultConfig}
```

可见，当前Backoff方法是由Exponential结构体实现的。

因此，进入grpc-go/internal/backoff/backoff.go文件的Backoff方法：

```go
1．func (bc Exponential) Backoff(retries int) time.Duration {
2．      if retries == 0 {
3．         return bc.Config.BaseDelay
4．      }

5．   backoff, max := float64(bc.Config.BaseDelay),  float64(bc.Config.MaxDelay)
6．   for backoff < max && retries > 0 {
7．      backoff *= bc.Config.Multiplier
8．      retries--
9．    }
10．    if backoff > max {
11．      backoff = max
12．   }
13．   // Randomize backoff delays so that if a cluster of requests start at
14．   // the same time, they won't operate in lockstep.
15．   backoff *= 1 + bc.Config.Jitter*(grpcrand.Float64()*2-1)
16．   if backoff < 0 {
17．      return 0
18．   }

19．   return time.Duration(backoff)
20．}

```

主要代码说明：  
参数retries，表示的是第几次重试，不是指要重试的次数。

-   第2-4行：如果参数retries为0的话，返还bc.Config.BaseDelay。
    -   bc的类型就是Exponential类型
    -   在哪里创建的bc呢？
        -   就是前面的语句var DefaultExponential = Exponential{Config: grpcbackoff.DefaultConfig}，
        -   因此查看一下grpcbackoff.DefaultConfig：(grpc-go/internal/backoff/backoff.go)

```Golang
var DefaultConfig = Config{
   BaseDelay:  1.0 * time.Second,
   Multiplier: 1.6,
   Jitter:     0.2,
   MaxDelay:   120 * time.Second,
}
```

第5-12行：主要是通过第7行的表达式计算出backoff； 只是一种计算方式，其实就是backoff=backoff * Multiplier的幂次方；
第10-12行：对计算出的backoff进行校验，不能超过最大延迟时间MaxDelay
   第15行：应该是在集群中避免延迟时间相同，不加这个的话，当retries确定时，每次计算出来的延迟时间都是一样的，通过第15行可以避免这种现象。

# 3、总之
-   链接失败后，grpc客户端会继续尝试链接grpc服务器端
-   随着重试链接次数的不断增加，等待下次链接的时间也会不断增加，但是不能超过MaxDelay值。如第1次重试链接时等待1秒，第10次时可能需要等待2分钟才进行重试链接。

这种计算时间的方式，重新整理后，是不是可以应用到自己的项目中去呢？

大家可以测试一下，当传输数据帧时，链接突然断了的话，grpc客户端会不会继续尝试链接呢？
