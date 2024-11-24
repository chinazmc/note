#grpc 


本篇文章主要是分析一下dnsResolver类型的解析器的核心原理；

并且进行实际测试；

最后分析一下，当dnsResolver解析失败时，实现重试机制的原理；

# 1、dnsResolver解析器原理介绍

dns解析器的原理:

其实**底层调用**的是

golang自带net包中的**LookupHost、LookupSRV、LookupTXT**三个函数来实现解析的。

通过这三个函数远程去访问dns服务器，最终将用户设置的链接[地址转换](https://so.csdn.net/so/search?q=%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2&spm=1001.2101.3001.7020)成后端服务器地址列表。

直接进入[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)-go/internal/resolver/dns/dns_resolver.go中的Build方法：

```go
1．// Build creates and starts a DNS resolver that watches the name resolution of the target.
2．// resolver_conn_wrapper.go 文件里的newCCResolverWrapper方法里调用
3．func (b *dnsBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
4．   host, port, err := parseTarget(target.Endpoint, defaultPort)
5．   if err != nil {
6．      return nil, err
7．   }

8．   // IP address.
9．   if ipAddr, ok := formatIP(host); ok {
10．      addr := []resolver.Address{{Addr: ipAddr + ":" + port}}
11．      cc.UpdateState(resolver.State{Addresses: addr})
12．      return deadResolver{}, nil
13．   }

14．   // DNS address (non-IP).
15．   ctx, cancel := context.WithCancel(context.Background())
16．   d := &dnsResolver{
17．      host:                 host,
18．      port:                 port,
19．      ctx:                  ctx,
20．      cancel:               cancel,
21．      cc:                   cc,
22．      rn:                   make(chan struct{}, 1),
23．      disableServiceConfig: opts.DisableServiceConfig,
24．   }

25．   if target.Authority == "" {
26．      d.resolver = defaultResolver
27．   } else {
28．      d.resolver, err = customAuthorityResolver(target.Authority)
29．      if err != nil {
30．         return nil, err
31．      }
32．   }

33．   d.wg.Add(1)
34．   go d.watcher()
35．   d.ResolveNow(resolver.ResolveNowOptions{})
36．   return d, nil
37．}
```

主要代码说明：

-   第4行：主要是对用户输入的target.Endpoint进行解析，如localhost:50051,sayHello:50051；
-   第8-13行：如果用户输入的是具体的IP(如192.168.43.214)，而不是域名的话，就直接更新cc的状态，最终实现向grpc服务器端发起连接
-   第16-24行：构建dnsResolver构建结构体
-   第25-32行：核心目的就是初始化dnsResolver结构体中的resolver,类型是netResolver接口
-   第34行：启动一个协程，监听；去向dns服务器发起请求，获取注册到dns服务器上的后端服务器地址列表
-   第35行：调用解析器的ResolveNow方法；

进入ResolveNow方法内部看看：

```go
func (d *dnsResolver) ResolveNow(resolver.ResolveNowOptions) {
   select {
   case d.rn <- struct{}{}:
   default:
   }
}
```

使用了多路复用器，这里执行的是case分支；

如果case不满足的情况下，会执行default。

要结合第34行watcher方法里的第7行一起看。

从ResolveNow方法的名称，可以看出来，就是开始解析的意思；

其实就是告诉wacther方法里第7行的通道d.rn发送信号，目的解除阻塞，继续执行下面的代码。

点击第34行，进入watcher方法里：

```go
1．func (d *dnsResolver) watcher() {
2．   defer d.wg.Done()
3．   for {
4．      select {
5．      case <-d.ctx.Done():
6．         return
7．      case <-d.rn:
8．      }

9．      state, err := d.lookup()
10．      if err != nil {
11．         d.cc.ReportError(err)
12．      } else {
13．         d.cc.UpdateState(*state)
14．      }

15．      // Sleep to prevent excessive re-resolutions. Incoming resolution requests
16．      // will be queued in d.rn.
17．      t := time.NewTimer(minDNSResRate)
18．      select {
19．      case <-t.C:
20．      case <-d.ctx.Done():
21．           t.Stop()
22．           return
23．      }
24．   }
25．}
```

watcher方法的核心目的：

  调用第9行实现dns解析，获得后端grpc服务器地址，传递给第13行，从而实现grpc客户端向grpc服务器端发起连接请求。

主要代码说明：

-   第4-8行：是一个select实现的多路复用器；有什么作用呢？
    -   提供退出for循环的出口
    -   d.ctx.Done()，如果给解析器的上下文发送结束信号，这里就可以退出for循环了；
    -   第7行：d.rn通道处于阻塞的情况时，阻止程序继续往下执行，可以防止多次dns解析ResolveNow方法里的语句case d.rn <- struct{}{}:可以解除阻塞。
-   第9行: 调用lookup实现dns地址解析；
    -   内部调用的是golang原生自带的net包中的lookupSRV、lookupHost、lookupTXT方法，对三个方法获得的结果转换成grpc框架中的resolver.State结构体即可。
    -   这就是dns解析器最核心的原理，最终将用户输入的地址target转换成了grpc服务器地址；
-   第13行：更新State，最终实现的是向grpc服务器端发起链接请求
-   第17-23行：使用timer+select实现一个定时器。minDNSResRate的值就是30秒；如果定时器结束时就会执行第19行，如果解析器结束时就会执行第20行。这个定时器有什么目的呢？
    -   如果dns解析失败了的话，防止频繁的解析
    -   解析器结束时，退出for循环

  这个方法使用了for循环，两个select，目的应该是在dns解析失败的情况下，可以提供重试机制；并不是所有的解析器都有重试机制，比方说passthrough解析器里就没有。

  这里为什么会有重试机制呢？

  可能是dns解析器拿到的地址是域名，并不是后端提供服务的地址，需要向dns服务器发起请求来获得；(这里仅仅是一个猜测，或者不同的解析器可以根据实际情况来考虑要不要提供)  
   
 

# 2、dnsResolver实际测试

## 2.1、测试环境说明

我这里的测试环境是在Mac物理机上启动了grpc客户端，grpc服务器端，启动虚拟机；

其中虚拟机里安装了centos系统，在centos系统里以容器的方式启动了coredns服务器。

具体看下图所示：  
![dns解析器实际测试流程图](https://img-blog.csdnimg.cn/20210527203039987.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

## 2.2、Mac环境下设置dns服务配置

需要在Mac物理机上配置coredns服务器的地址，这样当grpc客户端使用dns解析地址时，就会转向dns服务器。

系统偏好设置-网络-高级-dns  
![mac设置dns](https://img-blog.csdnimg.cn/20210527203159269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70)

> 如果是window下环境的话，可以查询一下百度。目前，这里没有window环境。

## 2.3、配置、启动coredns

此次dns服务选择coredns来实现，使用镜像方式部署服务。

### 2.3.1、coredns镜像说明

镜像tag:coredns/coredns:latest

```go
[root@master ~]# docker images | grep coredns
coredns/coredns                                             latest              296a6d5035e2        7 months ago        42.5MB
[root@master ~]#  
```

> 镜像已经上传到百度网盘里，可以下载使用;  
> docker load < coredns.tar  
> 即可

### 2.3.2、以容器方式启动coredns服务

命令如下：

```go
docker run -d --name core-dns -p 53:53/tcp -p 53:53/udp -v /root/coredns/hosts:/etc/hosts -v /etc/dnscore:/Corefile coredns/coredns:latest
```

其中，  
/root/coredns/hosts文件内容：

```
192.168.1.102  sayHello
```

假设192.168.1.102就是grpc服务器端的地址，sayHello就是客户端解析的域名;

> grpc客户端向dns发起dns解析器请求后，dns服务器会将192.168.1.102服务器地址反馈给grpc客户端；  
> 这样的话，客户端就知道了SayHello服务的后端grpc服务器地址了，就可以发起请求了。

/etc/dnscore文件内容：

```go
.:53 {
    hosts {
        fallthrough
    }
    forward . 202.96.128.86 114.114.114.114 8.8.8.8
    errors
    cache
}
```

## 2.4、grpc客户端测试用例

可以在grpc-go框架[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)提供的测试用例里随便找一个就可以。

这里仅仅列出需要修改的语句：

```go
conn, err := grpc.Dial("dns:///sayHello:50051", grpc.WithInsecure(), grpc.WithBlock())
```

> 注意:  
> 要使用dns协议；域名，端口根据自己的实际情况进行修改。

这样的话，就会使用dns解析器了。

## 2.5、grpc服务器端用例

可以在grpc-go框架源码提供的测试用例里随便找一个就可以。不再占用额外的篇幅了。

# 3、dnsResolver解析失败时，实现重试机制的原理？

再想分享一下，当dns解析失败时，会不会有重试机制？

进入到grpc-go/internal/resolver/dns/dns_resolver.go文件中的watcher方法里：

```go
1．func (d *dnsResolver) watcher() {
2．   defer d.wg.Done()
3．   for {
4．      select {
5．      case <-d.ctx.Done():
6．         return
7．      case <-d.rn:
8．      }

9．      state, err := d.lookup()
10．      if err != nil {
11．         d.cc.ReportError(err)
12．      } else {
13．         d.cc.UpdateState(*state)
14．      }

15．      // Sleep to prevent excessive re-resolutions. Incoming resolution requests
16．      // will be queued in d.rn.
17．      t := time.NewTimer(minDNSResRate)
18．      select {
19．      case <-t.C:
20．      case <-d.ctx.Done():
21．           t.Stop()
22．           return
23．      }
24．   }
25．}
```

该方法里使用了for循环，看似可以实现重试机制，其实仅仅有for是不够的；

因为在进行调用lookup解析前，有多路复用器select，暗含着阻塞的作用。

就是这个分支”case <- d.rn”, 当第一次执行完该分支后，通道d.rn处于阻塞状态，不会继续往下执行程序，

而ResolveNow方法里，select只会执行一次，不在向通道d.rn再次灌入数据了；

那么此时dns解析地址失败了，如何重试呢？

(换句话说，如何再次向通道d.rn灌入数据呢)

绘制了一个整体的流程图：  
![[Pasted image 20220523151456.png]]

> 此图第一眼看过去，肯定很吓人，  
>   
> 但是我相信，只要能沉下来心来，肯定会有收获的。  
>   
> 从上往下，从左到右顺序看，即可。  
>   
> 一边翻阅代码，一边看图。  
>   
> 已经将高清图片，上传到百度网盘里了，可以下载查看。  
>   
> 链接: https://pan.baidu.com/s/1za02qnUII78n-XhlrLf7RA  
> 密码: 3tok

根据上面的流程图，分析一下dns解析器是如何实现重试的？

先看一下watcher方法：

-   当for循环第一次执行轮询时，执行完第7行后，d.rn进入阻塞状态；
-   调用d.lookup方法开始解析，从dns服务器里获取解析地址，并转换为grpc-go框架的结构体resolver.State
-   拿到后端服务器地址，执行第13行，

最终进入grpc-go/resolver_conn_wrapper.go文件的UpdateState方法里：

```go
1. func (ccr *ccResolverWrapper) UpdateState(s resolver.State) {
2.   if ccr.done.HasFired() {
3.   return
4.}
5.      channelz.Infof(ccr.cc.channelzID, "ccResolverWrapper: sending update to cc: %v", s)
6.   if channelz.IsOn() {
7.      ccr.addChannelzTraceEvent(s)
8.   }
9.   ccr.curState = s
10.   ccr.poll(ccr.cc.updateResolverState(ccr.curState, nil))
11.}
```

主要代码说明：

-   第9行：更新解析器包装类的状态
-   第10行：调用ccr.cc.updateResolverState更新解析器的状态

进入grpc-go/clientconn.go文件中的updateResolverState方法里，找到下面的语句：

```go
uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
```

假设，此次采用的是pickfirst平衡器，因此进入grpc-go/pickfirst.go文件中的UpdateClientConnState方法里：

```go
1. func (b *pickfirstBalancer) UpdateClientConnState(cs balancer.ClientConnState) error {
2.   if len(cs.ResolverState.Addresses) == 0 {
3.      b.ResolverError(errors.New("produced zero addresses"))
4.      return balancer.ErrBadResolverState
5.   }
6.   // ----------省略代码--------
7.}
```

主要代码说明：

-   第2-5行：校验解析地址Addresses的个数是否为0，若为0的话，就返回balancer.ErrBadResolverState，也就是”bad resolver state”

经过层层返回，最终到grpc-go/resolver_conn_wrapper.go文件的UpdateState方法的第10行ccr.poll方法：

```go
1．func (ccr *ccResolverWrapper) poll(err error) {
2．   ccr.pollingMu.Lock()
3．   defer ccr.pollingMu.Unlock()
4．      if err != balancer.ErrBadResolverState {
5．      // stop polling
6．      if ccr.polling != nil {
7．         close(ccr.polling)
8．         ccr.polling = nil
9．       }
10．     
11．      return
12．   }
13．   
14．   if ccr.polling != nil {
15．      // already polling
16．      return
17．   }
18．  
19．   p := make(chan struct{})
20．   ccr.polling = p
21．   go func() {
22．     for i := 0; ; i++ {
23．        ccr.resolveNow(resolver.ResolveNowOptions{})
24．         t := time.NewTimer(ccr.cc.dopts.resolveNowBackoff(i))
25．         select {
26．         case <-p:
27．           t.Stop()
28．            return
29．         case <-ccr.done.Done():
30．             t.Stop()
31．            return
32．         case <-t.C:
33．            select {
34．            case <-p:
35．               return
36．            default:
37．               }
38．            // Timer expired; re-resolve.
39．         }
40．      }
41．   }()
42．}

```

该方法的核心目的就是，

如果允许解析器重新解析的话，就专门启动一个协程，调用resolveNow方法，

不同的解析器会根据自己的实际情况实现自己的resolverNow方法，

比方说dns解析器，给d.rn通道发送消息，从而让d.rn通道由阻塞状态，进入非阻塞状态，从而继续dns解析。

主要代码说明：

-   第4-9行：判断错误信息是否是”bad resolver state”
    -   是，就继续执行后面的代码
    -   否，继续判断是否已经处于轮询状态了，也就是ccr.polling是否为nil，若不为nil，则关闭通道；因为只允许错误类型是bad resolver state的进行轮询，现在错误字符串发生了变化，轮询协程自然可以关闭了。
-   第14-17行：判断是否已经处于轮询状态了，若是的话，就返回；防止多次轮询
-   第19-20行：初始化ccr.polling，可以认为已经处于轮询状态了
-   第21-41行：启动一个协程，专门轮询，其实主要是调用了ccr.resolveNow方法
-   第25-37行：是一个多路复用器：
    -   什么情况下，会执行第26行呢？  
        比方说，已经处于轮询状态了，也就是说已经报了”bad resolver state”错，后来，错误信息又更新了如“timed out waiting for server handshake”，此时就会进入第7行，关闭通道的同时，就会给通道发送一个信息，从而协程里就会执行第26行了
    -   什么情况下，会执行29行呢？  
        当解析器的上下文结束时，会执行；
    -   什么情况下，会执行第32行呢？  
        当定时器时间结束时，就会执行这里；然后继续判断p是否处于阻塞状态，
        -   如果没有阻塞的话，该协程就结束了；
        -   如果处于阻塞状态，就执行default分支；然后，又重新进行for循环阶段，从第23行开始执行

当程序启动完协程后(第21-41行的协程)，poll方法就结束了，

也就是说grpc-go/resolver_conn_wrapper.go文件的UpdateState方法结束了，

返还到grpc-go/internal/resolver/dns/dns_resolver.go文件中的watcher方法里的第13行就执行结束了。

继续执行watcher方法里的第17行-23行:

-   定时器固定时间是30秒，因此需要等待30秒种后，重新从watcher方法的第4行开始执行，此时也就是for循环的第二次轮询了；
-   此时watcher方法里第7行中的d.rn通道已经非阻塞了，因为poll方法中的第23行里，已经向d.rn通道灌入了数据，因此d.rn通道就处于非阻塞状态了，从而继续调用d.loolup方法进行dns解析了，从而实现了解析器解析失败后的重试。

# 4、做一个简单的总结:

-   并不是对所有的解析错误都进行重试的，只针对错误字符串是”bad resolver state”的进行重试
-   并不是所有的解析器都支持重试机制，目前发现只有dns解析器有重试机制。
-   如果读者对dns解析器解析失败后的重试机制，没有理解的话，不会影响后面的阅读的。
-   dns解析器的底层，
    -   其实就是利用golang原生自带的net包中的LookupHost、LookupSRV、LookupTXT三个函数来实现解析的；
    -   知道了这个细节，当我们自己的项目中，遇到类似的场景，就可以进行经验的转化，也可以仿照，为自己的项目服务。

