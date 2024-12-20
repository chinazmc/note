

这一次重启真的无法解决问题了：一次 **MySQL** 主动关闭，导致服务出现大量 **CLOSE_WAIT** 的全流程排查过程。

近日遇到一个线上服务 **socket** 资源被不断打满的情况。通过各种工具分析线上问题,定位到问题代码。这里对该问题发现、修复过程进行一下复盘总结。

先看两张图。一张图是服务正常时监控到的 **socket** 状态，另一张当然就是异常啦！

![image-20181208155626513](https://segmentfault.com/img/remote/1460000017313209?w=1144&h=600 "image-20181208155626513")  
图一：正常时监控

![image-20181208160100456](https://segmentfault.com/img/remote/1460000017313210?w=1172&h=596 "image-20181208160100456")  
图二：异常时监控

从图中的表现情况来看，就是从 **04:00** 开始，socket 资源不断上涨，每个谷底时重启后恢复到正常值，然后继续不断上涨不释放，而且每次达到峰值的间隔时间越来越短。

重启后，排查了日志，没有看到 **panic** ，此时也就没有进一步检查，真的以为重启大法好。

## 情况说明

该服务使用Golang开发，已经上线正常运行将近一年，提供给其它服务调用，主要底层资源有DB/Redis/MQ。

为了后续说明的方便，将服务的架构图进行一下说明。

![image-20181208162049386](https://segmentfault.com/img/remote/1460000017313211?w=932&h=882 "image-20181208162049386")  
图三：服务架构

架构是非常简单。

问题出现在早上 **08:20** 左右开始的，报警收到该服务出现 **504**，此时第一反应是该服务长时间没有重启（快两个月了），可能存在一些内存泄漏，没有多想直接进行了重启。也就是在图二第一个谷底的时候，经过重启服务恢复到正常水平（重启真好用，开心）。

将近 **14:00** 的时候，再次被告警出现了 **504** ，当时心中略感不对劲，但由于当天恰好有一场大型促销活动，因此先立马再次重启服务。直到后续大概过了1小时后又开始告警，连续几次重启后，发现需要重启的时间间隔越来越短。此时发现问题绝不简单。**这一次重启真的解决不了问题老**，因此立马申请机器权限、开始排查问题。下面的截图全部来源我的重现demo，与线上无关。

## 发现问题

出现问题后，首先要进行分析推断、然后验证、最后定位修改。根据当时的表现是分别进行了以下猜想。

_ps：后续截图全部来源自己本地复现时的截图_

### 推断一

> socket 资源被不断打满，并且之前从未出现过，今日突然出现，**怀疑是不是请求量太大压垮服务**

经过查看实时 **qps** 后，放弃该想法，虽然量有增加，但依然在服务器承受范围（远远未达到压测的基准值）。

### 推断二

> 两台机器故障是同时发生，重启一台，另外一台也会得到缓解，作为独立部署在两个集群的服务非常诡异

有了上面的的依据，推出的结果是肯定是该服务依赖的底层资源除了问题，要不然不可能独立集群的服务同时出问题。

由于监控显示是 **socket** 问题，因此通过 **netstat** 命令查看了当前tcp链接的情况（本地测试，线上实际值大的多）

/go/src/hello # netstat -na | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
LISTEN 2
CLOSE_WAIT 23 # 非常异常
TIME_WAIT 1

发现绝大部份的链接处于 **CLOSE_WAIT** 状态，这是非常不可思议情况。然后用 `netstat -an` 命令进行了检查。

![image-20181208172652155](https://segmentfault.com/img/remote/1460000017313212?w=1186&h=1066 "image-20181208172652155")  
图四：大量的CLOSE_WAIT

> CLOSED 表示socket连接没被使用。  
> LISTENING 表示正在监听进入的连接。  
> SYN_SENT 表示正在试着建立连接。  
> SYN_RECEIVED 进行连接初始同步。  
> ESTABLISHED 表示连接已被建立。  
> CLOSE_WAIT 表示远程计算器关闭连接，正在等待socket连接的关闭。  
> FIN_WAIT_1 表示socket连接关闭，正在关闭连接。  
> CLOSING 先关闭本地socket连接，然后关闭远程socket连接，最后等待确认信息。  
> LAST_ACK 远程计算器关闭后，等待确认信号。  
> FIN_WAIT_2 socket连接关闭后，等待来自远程计算器的关闭信号。  
> TIME_WAIT 连接关闭后，等待远程计算器关闭重发。

然后开始重点思考为什么会出现大量的mysql连接是 **CLOSE_WAIT** 呢？为了说清楚，我们来插播一点TCP的四次挥手知识。

#### TCP四次挥手

我们来看看 **TCP** 的四次挥手是怎么样的流程：

![image-20181208175427046](https://segmentfault.com/img/remote/1460000017313213?w=1316&h=1106 "image-20181208175427046")  
图五：TCP四次挥手

用中文来描述下这个过程：

Client: `服务端大哥，我事情都干完了，准备撤了`，这里对应的就是客户端发了一个**FIN**

Server：`知道了，但是你等等我，我还要收收尾`，这里对应的就是服务端收到 **FIN** 后回应的 **ACK**

经过上面两步之后，服务端就会处于 **CLOSE_WAIT** 状态。过了一段时间 **Server** 收尾完了

Server：`小弟，哥哥我做完了，撤吧`，服务端发送了**FIN**

Client：`大哥，再见啊`，这里是客户端对服务端的一个 **ACK**

到此服务端就可以跑路了，但是客户端还不行。为什么呢？客户端还必须等待 **2MSL** 个时间，这里为什么客户端还不能直接跑路呢？主要是为了防止发送出去的 **ACK** 服务端没有收到，服务端重发 **FIN** 再次来询问，如果客户端发完就跑路了，那么服务端重发的时候就没人理他了。这个等待的时间长度也很讲究。

> **Maximum Segment Lifetime** 报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃

这里一定不要被图里的 **client／server** 和项目里的客户端服务器端混淆，你只要记住：主动关闭的一方发出 **FIN** 包（Client），被动关闭（Server）的一方响应 **ACK** 包，此时，被动关闭的一方就进入了 **CLOSE_WAIT** 状态。如果一切正常，稍后被动关闭的一方也会发出 **FIN** 包，然后迁移到 **LAST_ACK** 状态。

既然是这样， **TCP** 抓包分析下：

```bash
/go # tcpdump -n port 3306
# 发生了 3次握手
11:38:15.679863 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [S], seq 4065722321, win 29200, options [mss 1460,sackOK,TS val 2997352 ecr 0,nop,wscale 7], length 0

11:38:15.679923 IP 172.18.0.3.3306 > 172.18.0.5.38822: Flags [S.], seq 780487619, ack 4065722322, win 28960, options [mss 1460,sackOK,TS val 2997352 ecr 2997352,nop,wscale 7], length 0

11:38:15.679936 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [.], ack 1, win 229, options [nop,nop,TS val 2997352 ecr 2997352], length 0
# mysql 主动断开链接
11:38:45.693382 IP 172.18.0.3.3306 > 172.18.0.5.38822: Flags [F.], seq 123, ack 144, win 227, options [nop,nop,TS val 3000355 ecr 2997359], length 0 # MySQL负载均衡器发送fin包给我

11:38:45.740958 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [.], ack 124, win 229, options [nop,nop,TS val 3000360 ecr 3000355], length 0 # 我回复ack给它

... ... # 本来还需要我发送fin给他，但是我没有发，所以出现了close_wait。那这是什么缘故呢？
```

> **src > dst: flags data-seqno ack window urgent options**
> 
> src > dst 表明从源地址到目的地址  
> flags 是TCP包中的标志信息,S 是SYN标志, F(FIN), P(PUSH) , R(RST) "."(没有标记)  
> data-seqno 是数据包中的数据的顺序号  
> ack 是下次期望的顺序号  
> window 是接收缓存的窗口大小  
> urgent 表明数据包中是否有紧急指针  
> options 是选项

结合上面的信息，我用文字说明下：**MySQL负载均衡器** 给我的服务发送 **FIN** 包，我进行了响应，此时我进入了 **CLOSE_WAIT** 状态，但是后续作为被动关闭方的我，并没有发送 **FIN**，导致我服务端一直处于 **CLOSE_WAIT** 状态，无法最终进入 **CLOSED** 状态。

那么我推断出现这种情况可能的原因有以下几种：

1.  **负载均衡器** 异常退出了，
    
    `这基本是不可能的，他出现问题绝对是大面积的服务报警，而不仅仅是我一个服务`
    
2.  **MySQL负载均衡器** 的超时设置的太短了，导致业务代码还没有处理完，**MySQL负载均衡器** 就关闭tcp连接了
    
    `这也不太可能，因为这个服务并没有什么耗时操作，当然还是去检查了负载均衡器的配置，设置的是60s。`
    
3.  代码问题，**MySQL** 连接无法释放
    
    `目前看起来应该是代码质量问题，加之本次数据有异常，触发到了以前某个没有测试到的点，目前看起来很有可能是这个原因`
    

## 查找错误原因

由于代码的业务逻辑并不是我写的，我担心一时半会看不出来问题，所以直接使用 `perf` 把所有的调用关系使用火焰图给绘制出来。既然上面我们推断代码中没有释放mysql连接。无非就是：

1.  确实没有调用close
2.  有耗时操作（火焰图可以非常明显看到），导致超时了
3.  mysql的事务没有正确处理，例如：rollback 或者 commit

由于火焰图包含的内容太多，为了让大家看清楚，我把一些不必要的信息进行了折叠。

![image-20181208212045848](https://segmentfault.com/img/remote/1460000017313214?w=2424&h=1232 "image-20181208212045848")  
图六：有问题的火焰图

火焰图很明显看到了开启了事务，但是在余下的部分，并没有看到 **Commit** 或者是**Rollback** 操作。这肯定会操作问题。然后也清楚看到出现问题的是：

**MainController.update** 方法内部，话不多说，直接到 update 方法中去检查。发现了如下代码：

```go
func (c *MainController) update() (flag bool) {
    o := orm.NewOrm()
    o.Using("default")
    
    o.Begin()
    nilMap := getMapNil()
    if nilMap == nil {// 这里只检查了是否为nil，并没有进行rollback或者commit
        return false
    }

    nilMap[10] = 1
    nilMap[20] = 2
    if nilMap == nil && len(nilMap) == 0 {
        o.Rollback()
        return false
    }

    sql := "update tb_user set name=%s where id=%d"
    res, err := o.Raw(sql, "Bug", 2).Exec()
    if err == nil {
        num, _ := res.RowsAffected()
        fmt.Println("mysql row affected nums: ", num)
        o.Commit()
        return true
    }

    o.Rollback()
    return false
}
```

至此，全部分析结束。经过查看 **getMapNil** 返回了nil，但是下面的判断条件没有进行回滚。

```go
if nilMap == nil {
    o.Rollback()// 这里进行回滚
    return false
}
```

## 总结

整个分析过程还是废了不少时间。最主要的是主观意识太强，觉得运行了一年没有出问题的为什么会突然出问题？因此一开始是质疑 SRE、DBA、各种基础设施出了问题（人总是先怀疑别人）。导致在这上面费了不少时间。

理一下正确的分析思路：

1.  出现问题后，立马应该检查日志，确实日志没有发现问题；
2.  监控明确显示了socket不断增长，很明确立马应该使用 `netstat` 检查情况看看是哪个进程的锅；
3.  根据 `netstat` 的检查，使用 `tcpdump` 抓包分析一下为什么连接会**被动断开**（TCP知识非常重要）；
4.  如果熟悉代码应该直接去检查业务代码，如果不熟悉则可以使用 `perf` 把代码的调用链路打印出来；
5.  不论是分析代码还是火焰图，到此应该能够很快定位到问题。

那么本次到底是为什么会出现 **CLOSE_WAIT** 呢？大部分同学应该已经明白了，我这里再简单说明一下：

由于那一行代码没有对事务进行回滚，导致服务端没有主动发起close。因此 **MySQL负载均衡器** 在达到 60s 的时候主动触发了close操作，但是通过tcp抓包发现，服务端并没有进行回应，这是因为代码中的事务没有处理，因此从而导致大量的端口、连接资源被占用。在贴一下挥手时的抓包数据：

```bash
# mysql 主动断开链接
11:38:45.693382 IP 172.18.0.3.3306 > 172.18.0.5.38822: Flags [F.], seq 123, ack 144, win 227, options [nop,nop,TS val 3000355 ecr 2997359], length 0 # MySQL负载均衡器发送fin包给我

11:38:45.740958 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [.], ack 124, win 229, options [nop,nop,TS val 3000360 ecr 3000355], length 0 # 我回复ack给它
```

希望此文对大家排查线上问题有所帮助。为了便于帮助大家理解，下面附上正确情况下的火焰图与错误情况下的火焰图。大家可以自行对比。

-   [正确情况下的火焰图](https://link.segmentfault.com/?enc=UxodfQ3EpBRG1gUVHGhYSA%3D%3D.%2B7c6ycbYIsKC3t9CjvrU7bQlyG00GE%2Bvi9i2ZRCckABCscyz3nzMh6VYAfBIuSh5)
-   [错误情况的火焰图](https://link.segmentfault.com/?enc=qws%2BGhWSR4hQj5Md%2FsIBxA%3D%3D.MOS1VN2m1rmW7NDey%2Fx6e2DpD9p%2FQewOI%2BbYYMtjh%2F8%3D)

因此说明，是SLB主动关闭了连接但是多台应用服务器都没有响应ack 导致了close_wait. 只是这样够吗？ 明显不够。继续

SLB 作为负载均衡，基本没有业务逻辑，那它会主动关闭连接的场景有哪些？

-   进程退出（正常或者非正常）
-   TCP 连接超时  
这2个情况很好判断而且大多数情况下是第二种（我遇见的也是），如果你还记得我文章一开始的环境结构图，我想基本可以得出以下结论是：

          由于调用第三方的请求API太慢而导致SLB这边请求超时引起的SLB关闭了连接.

那解决方案也很容易就有了：
-      加大SLB到应用服务器的连接超时时间
-      在调用第三方的时候采用异步请求
完了吗？ （我怎么那么啰嗦。。。）

**“再等等” 还有问题没被回答**

我们还有两个问题没回答：
1.  为啥一台机器区区几百个close_wait就导致不可继续访问？不合理啊，一台机器不是号称最大可以开到65535个端口吗？
2.  为啥明明有多个服务器承载，却几乎同时出了close_wait? 又为什么同时不能再服务？那要SLB还有啥用呢
是啊，解释了为什么出close_wait, 但并不能解释这2个问题。好吧，既然找到了第一层原因，就先重启服务让服务可以用吧。剩下的我们可以用个简单的原型代码模拟一下。此时我的目光回到了我们用的Tornado上面来，“当你有问题解释不了的时候，你还没有发现正真的问题”

Tornado是一个高性能异步非阻塞的HTTP 服务器（还不明白这个啥意思的可以看 “从韩梅梅和林涛的故事中，学习一下I/O模型 ” 这篇文章，生动！！！），其核心类就是IOLoop，默认都是用HttpServer单进程单线程的方式启动 (Tornado的process.fork_processes 也支持多进程方式，但官方并不建议)。我们还是通过图来大概说下

IOLoop干了啥：

![图片](http://mmbiz.qpic.cn/mmbiz/GjBicz1ItfB2zMg9HAtIXS1bibjQytXnM3MNSD7EZ5o503hHMicFF8gK7w1rTscylEtpscwbe1CJYBvwYMIkcJQ9w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)  

-   维护每个listen socket注册的fd；
-   当listen socket可读时回调_handle_events处理客户端请求，这里面的实现就是基于epoll 模型
好，现在我们知道：
1.  Tornado是单进程启动的服务，所以IOLoop也就一个实例在监听并轮询
2.  IOLoop在监听端口，当对应的fd ready时会回调注册的handler去处理请求，这里的handler就是我们写业务逻辑的RequestHandler
3.  如果我们启用了Tornado的  @tornado.gen.coroutine，那理论上一个请求很慢不会影响其他的请求，那一定是代码什么地方错了。  
进而查看实现代码，才真相大白，虽然我们用了 @tornado.gen.coroutine 和yield，但是在向第三方请求时用的是urllib2 库。这是一个彻头彻尾的同步库，人家就不支持AIO（Tornado 有自己AsyncHTTPClient支持AIO). 

由此让我们来总结下原因：
1.   Tornado是单进程启动的服务，所以IOLoop也就一个实例在监听并轮询
2.  Tornado在bind每个socket的时候有默认的链接队列（也叫backlog）为128个
3.  由于代码错误，我们使用了同步库urllib2 做第三方请求，导致访问第三方的时候当前RequestHandler是同步的（yield不起作用），因此当IOLoop回调这个RequestHandler时会等待它返回
4.  第三方接口真的不快！

最后来回答这两个问题：
1. 为啥一台机器区区几百个close_wait就导致不可继续访问？不合理啊，一台机器不是号称最大可以开到65535个端口吗？

[回答]： 由于原因＃4和＃3所以导致整个IOLoop慢了，进而因为＃2 导致很多请求堆积，也就是说很多请求在被真正处理前已经在backlog里等了一会了。导致了SLB这端的链接批量的超时，同时又由于close_wait状态不会自动消失，导致最终无法再这个端口上创建新的链接引起了停止服务

2. 为啥明明有多个服务器承载，却几乎同时出了close_wait? 又为什么同时不能再服务？那要SLB还有啥用呢

[回答]： 有了上一个答案，结合SLB的特性，这个也就很好解释。这就是所谓的洪水蔓延，当SLB发现下面的一个节点不可用时会吧请求routing到其他可用节点上，导致了其他节点压力增大。也犹豫相同原因，加速了其他节点出现close_wait.


查看TIME_WAIT和CLOSE_WAIT数的命令：

```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

它会显示例如下面的信息：  
TIME_WAIT 、CLOSE_WAIT 、FIN_WAIT1 、ESTABLISHED 、SYN_RECV 、LAST_ACK  
常用的三个状态是：ESTABLISHED表示正在通信 、TIME_WAIT表示主动关闭、CLOSE_WAIT表示被动关闭。

服务器出现异常最长出现的状况是：

-   服务器保持了大量的TIME_WAIT状态。
-   服务器保持了大量的CLOSE_WAIT状态。

我们也都知道Linux系统中分给每个用户的文件句柄数是有限的，而TIME_WAIT和CLOSE_WAIT这两种状态如果一直被保持，那么意味着对应数目的通道(此处应理解为socket，一般一个socket会占用服务器端一个端口，服务器端的端口最大数是65535)一直被占用，一旦达到了上限，则新的请求就无法被处理，接着就是大量Too Many Open Files异常，然后tomcat、nginx、apache崩溃。。。


![图片描述](https://segmentfault.com/img/bV9DQk?w=732&h=563 "图片描述")

查看TIME_WAIT和CLOSE_WAIT数的命令：
下面来讨论这两种状态的处理方法，网络上也有很多资料把这两种情况混为一谈，认为优化内核参数就可以解决，其实这是不恰当的。优化内核参数在一定程度上能解决time_wait过多的问题，但是应对close_wait还得从应用程序本身出发。

1.  服务器保持了大量的time_wait状态

这种情况比较常见，一般会出现在爬虫服务器和web服务器(如果没做内核参数优化的话)上，那么这种问题是怎么产生的呢？

从上图可以看出time_wait是主动关闭连接的一方保持的状态，对于爬虫服务器来说它自身就是客户端，在完成一个爬取任务后就会发起主动关闭连接，从而进入time_wait状态，然后保持这个状态2MSL时间之后，彻底关闭回收资源。这里为什么会保持资源2MSL时间呢？这也是TCP/IP设计者规定的。

TCP要保证在所有可能的情况下使得所有的数据都能够被正确送达。当你关闭一个socket时，主动关闭一端的socket将进入TIME_WAIT状 态，而被动关闭一方则转入CLOSED状态，这的确能够保证所有的数据都被传输。当一个socket关闭的时候，是通过两端四次握手完成的，当一端调用 close()时，就说明本端没有数据要发送了。这好似看来在握手完成以后，socket就都可以处于初始的CLOSED状态了，其实不然。原因是这样安排状态有两个问题， 首先，我们没有任何机制保证最后的一个ACK能够正常传输，第二，网络上仍然有可能有残余的数据包(wandering duplicates)，我们也必须能够正常处理。

TIMEWAIT就是为了解决这两个问题而生的。

-   假设最后的一个ACK丢失，那么被动关闭一方收不到这最后一个ACK则会重发FIN。此时主动关闭一方必须保持一个有效的(time_wait状态下维持)状态信息，以便可以重发ACK。如果主动关闭的socket不维持这种状态而是进入close状态，那么主动关闭的一方在收到被动关闭方重新发送的FIN时则响应给被动方一个RST。被动方收到这个RST后会认为此次回话出错了。所以如果TCP想要完成必要的操作而终止双方的数据流传输，就必须完全正确的传输四次握手的四步，不能有任何的丢失。这就是为什么在socket在关闭后，任然处于time_wait状态的第一个原因。因为他要等待可能出现的错误(被动关闭端没有接收到最后一个ACK)，以便重发ACK。
-   假设目前连接的通信双方都调用了close(),双方同时进入closed的终结状态，而没有走 time_wait状态。则会出现如下问题：假如现在有一个新的连接建立起来，使用的IP地址与之前的端口完全相同，现在建立的一个连接是之前连接的完全复用，我们还假定之前连接中有数据报残存在网络之中，这样的话现在的连接收到的数据有可能是之前连接的报文。为了防止这一点。TCP不允许新的连接复用time_wait状态下的socket。处于time_wait状态的socket在等待2MSL时间后（之所以是两倍的MSL，是由于MSL是一个数据报在网络中单向发出 到认定丢失的时间，即(Maximum Segment Lifetime)报文最长存活时间，一个数据报有可能在发送途中或是其响应过程中成为残余数据报，确认一个数据报及其响应的丢弃需要两倍的MSL），将会转为closed状态。这就意味着，一个成功建立的连接，必须使得之前网络中残余的数据报都丢失了。

再引用网络中的一段话：

-   值得一说的是，基于TCP的http协议，一般(此处为什么说一般呢，因为当你在keepalive时间内 主动关闭对服务器端的连接时，那么主动关闭端就是客户端,否则客户端就是被动关闭端。下面的爬虫例子就是这种情况)主动关闭tcp一端的是server端，这样server端就会进入time_wait状态，可想而知，对于访问量大的web服务器，会存在大量的time_wait状态，假如server一秒钟接收1000个请求，那么就会积压240*1000=240000个time_wait状态。(RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。)，维持这些状态给服务器端带来巨大的负担。当然现代操作系统都会用快速的查找算法来管理这些 TIME_WAIT，所以对于新的 TCP连接请求，判断是否hit中一个TIME_WAIT不会太费时间，但是有这么多状态要维护总是不好。
-   HTTP协议1.1版本规定default行为是keep-Alive，也就是会重用tcp连接传输多个 request/response。之所以这么做的主要原因是发现了我们上面说的这个问题。

服务器保持了大量的close_wait状态

time_wait问题可以通过调整内核参数和适当的设置web服务器的keep-Alive值来解决。因为time_wait是自己可控的，要么就是对方连接的异常，要么就是自己没有快速的回收资源，总之不是由于自己程序错误引起的。但是close_wait就不一样了，从上图中我们可以看到服务器保持大量的close_wait只有一种情况，那就是对方发送一个FIN后，程序自己这边没有进一步发送ACK以确认。换句话说就是在对方关闭连接后，程序里没有检测到，或者程序里本身就已经忘了这个时候需要关闭连接，于是这个资源就一直被程序占用着。这个时候快速的解决方法是：

-   关闭正在运行的程序，这个需要视业务情况而定。
-   尽快的修改程序里的bug，然后测试提交到线上服务器。

# Reference
https://segmentfault.com/a/1190000017313251?utm_source=tag-newest
https://mp.weixin.qq.com/s?__biz=MzI4MjA4ODU0Ng==&mid=402163560&idx=1&sn=5269044286ce1d142cca1b5fed3efab1&3rd=MzA3MDU4NTYzMw==&scene=6#rd

