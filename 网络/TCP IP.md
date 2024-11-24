---
sr-due: 2022-10-12
sr-interval: 4
sr-ease: 270
---

#网络  #tcp  #note 

废话少说，首先，我们需要知道TCP在网络OSI的七层模型中的第四层——Transport层，IP在第三层——Network层，ARP在第二层——Data Link层，在第二层上的数据，我们叫Frame，在第三层上的数据叫Packet，第四层的数据叫Segment。

首先，我们需要知道，我们程序的数据首先会打到TCP的Segment中，然后TCP的Segment会打到IP的Packet中，然后再打到以太网Ethernet的Frame中，传到对端后，各个层解析自己的协议，然后把数据交给更高层的协议处理。

# TCP头格式
![[Pasted image 20221008121128.png]]
你需要注意这么几点：

-   TCP的包是没有IP地址的，那是IP层上的事。但是有源端口和目标端口。
-   一个TCP连接需要四个元组来表示是同一个连接（src_ip, src_port, dst_ip, dst_port）准确说是五元组，还有一个是协议。但因为这里只是说TCP协议，所以，这里我只说四元组。
-   注意上图中的四个非常重要的东西：
    -   **Sequence Number**是包的序号，**用来解决网络包乱序（reordering）问题。**
    -   **Acknowledgement Number**就是ACK——用于确认收到，**用来解决不丢包的问题**。
    -   **Window又叫Advertised-Window**，也就是著名的滑动窗口（Sliding Window），**用于解决流控的**。
    -   **TCP Flag** ，也就是包的类型，**主要是用于操控TCP的状态机的**。

关于其它的东西，可以参看下面的图示
![[Pasted image 20221008121156.png]]


标准的端口号由 Internet 号码分配机构(IANA)分配。这组数字被划分为特定范围，包括 熟知端口号(0 - 1023)、注册端口号(1024 - 49151)和动态/私有端口号(49152 - 65535)。

> 如果我们测试这些标准服务和其他 TCP/IP 服务(Telnet、 FTP、 SMTP等) 使用的端口号，会发现它们大多数是奇数。这是有历史原困的，这些端口号从 NCP 端口号派生而来(NCP 是网络控制协议，在 TCP 之前作为 ARPANET 的传输层协议)。NCP 虽然简单，但不是全双工的，困此每个应用需要两个连接，并为每个应用保留奇偶成对的端口号。当 TCP 和 UDP 成为标准的传输层协议时，每个应用只需要一个端口号，因此来自 NCP 的奇数端口号被使用。

在 TCP 数据报中，有一个 序列号 (Sequence Number)。如果序列号被人猜出来，就会展现出 TCP 的脆弱性。

如果选择合适的序列号、IP地址以及端口号，那么任何人都能伪造出一个 TCP 报文段，从而 打断 TCP 的正常连接[RFC5961]。一种抵御上述行为的方法是使初始序列号(或者临时端口 号[RFC6056])变得相对难以被猜出，而另一种方法则是加密。

Linux 系统采用一个相对复杂的过程来选择它的初始序列号。它采用基于时钟的方案，并且针对每一个连接为时钟设置随机的偏移量。随机偏移量是在连接标识(由 2 个 IP 地址与 2 个端口号构成的 4 元组，即 4 元组)的基础上利用加密散列函数得到的。散列函数的输人每隔 5 分钟就会改变一次。在 32 位的初始序列号中，最高的 8 位是一个保密的序列号，而剩余的备位则由散列函数生成。上述方法所生成的序列号很难被猜出，但依然会随着时间而逐步增加。据报告显示， Windows 系统使用了一种基于 RC4[S94] 的类似方案。

# TCP的状态机
其实，**网络上的传输是没有连接的，包括TCP也是一样的**。而TCP所谓的“连接”，其实只不过是在通讯的双方维护一个“连接状态”，让它看上去好像有连接一样。所以，TCP的状态变换是非常重要的。

下面是：“**TCP协议的状态机**”（[图片来源](http://www.tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm)） 和 “**TCP建链接**”、“**TCP断链接**”、“**传数据**” 的对照图，我把两个图并排放在一起，这样方便在你对照着看。另外，下面这两个图非常非常的重要，你一定要记牢。（吐个槽：看到这样复杂的状态机，就知道这个协议有多复杂，复杂的东西总是有很多坑爹的事情，所以TCP协议其实也挺坑爹的）
![[Pasted image 20221008121355.png]]
![[Pasted image 20221008121414.png]]
很多人会问，为什么建链接要3次握手，断链接需要4次挥手？

-   **对于建链接的3次握手，**主要是要初始化Sequence Number 的初始值。通信的双方要互相通知对方自己的初始化的Sequence Number（缩写为ISN：Inital Sequence Number）——所以叫SYN，全称Synchronize Sequence Numbers。也就上图中的 x 和 y。这个号要作为以后的数据通信的序号，以保证应用层接收到的数据不会因为网络上的传输的问题而乱序（TCP会用这个序号来拼接数据）。

-   **对于4次挥手，**其实你仔细看是2次，因为TCP是全双工的，所以，发送方和接收方都需要Fin和Ack。只不过，有一方是被动的，所以看上去就成了所谓的4次挥手。如果两边同时断连接，那就会就进入到CLOSING状态，然后到达TIME_WAIT状态。下图是双方同时断连接的示意图（你同样可以对照着TCP状态机看）：
![[Pasted image 20221008121610.png]]
另外，有几个事情需要注意一下：

-   **关于建连接时SYN超时**。试想一下，如果server端接到了clien发的SYN后回了SYN-ACK后client掉线了，server端没有收到client回来的ACK，那么，这个连接处于一个中间状态，即没成功，也没失败。于是，server端如果在一定时间内没有收到的TCP会重发SYN-ACK。在Linux下，默认重试次数为5次，重试的间隔时间从1s开始每次都翻售，5次的重试时间间隔为1s, 2s, 4s, 8s, 16s，总共31s，第5次发出后还要等32s都知道第5次也超时了，所以，总共需要 1s + 2s + 4s+ 8s+ 16s + 32s = 2^6 -1 = 63s，TCP才会把断开这个连接。

-   **关于SYN Flood攻击**。一些恶意的人就为此制造了SYN Flood攻击——给服务器发了一个SYN后，就下线了，于是服务器需要默认等63s才会断开连接，这样，攻击者就可以把服务器的syn连接的队列耗尽，让正常的连接请求不能处理。于是，Linux下给了一个叫**tcp_syncookies**的参数来应对这个事——当SYN队列满了后，TCP会通过源地址端口、目标地址端口和时间戳打造出一个特别的Sequence Number发回去（又叫cookie），如果是攻击者则不会有响应，如果是正常连接，则会把这个 SYN Cookie发回来，然后服务端可以通过cookie建连接（即使你不在SYN队列中）。请注意，**请先千万别用tcp_syncookies来处理正常的大负载的连接的情况**。因为，synccookies是妥协版的TCP协议，并不严谨。对于正常的请求，你应该调整三个TCP参数可供你选择，第一个是：tcp_synack_retries 可以用他来减少重试次数；第二个是：tcp_max_syn_backlog，可以增大SYN连接数；第三个是：tcp_abort_on_overflow 处理不过来干脆就直接拒绝连接了。

-   **关于ISN的初始化**。ISN是不能hard code的，不然会出问题的——比如：如果连接建好后始终用1来做ISN，如果client发了30个segment过去，但是网络断了，于是 client重连，又用了1做ISN，但是之前连接的那些包到了，于是就被当成了新连接的包，此时，client的Sequence Number 可能是3，而Server端认为client端的这个号是30了。全乱了。[RFC793](https://tools.ietf.org/html/rfc793)中说，ISN会和一个假的时钟绑在一起，这个时钟会在每4微秒对ISN做加一操作，直到超过2^32，又从0开始。这样，一个ISN的周期大约是4.55个小时。因为，我们假设我们的TCP Segment在网络上的存活时间不会超过Maximum Segment Lifetime（缩写为MSL – [Wikipedia语条](https://en.wikipedia.org/wiki/Maximum_Segment_Lifetime)），所以，只要MSL的值小于4.55小时，那么，我们就不会重用到ISN。

-   **关于 MSL 和 TIME_WAIT**。通过上面的ISN的描述，相信你也知道MSL是怎么来的了。我们注意到，在TCP的状态图中，从TIME_WAIT状态到CLOSED状态，有一个超时设置，这个超时设置是 2*MSL（[RFC793](https://tools.ietf.org/html/rfc793)定义了MSL为2分钟，Linux设置成了30s）为什么要这有TIME_WAIT？为什么不直接给转成CLOSED状态呢？主要有两个原因：1）TIME_WAIT确保有足够的时间让对端收到了ACK，如果被动关闭的那方没有收到Ack，就会触发被动端重发Fin，一来一去正好2个MSL，2）有足够的时间让这个连接不会跟后面的连接混在一起（你要知道，有些自做主张的路由器会缓存IP数据包，如果连接被重用了，那么这些延迟收到的包就有可能会跟新连接混在一起）。你可以看看这篇文章《[TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)》

-   **关于TIME_WAIT数量太多**。从上面的描述我们可以知道，TIME_WAIT是个很重要的状态，但是如果在大并发的短链接下，TIME_WAIT 就会太多，这也会消耗很多系统资源。只要搜一下，你就会发现，十有八九的处理方式都是教你设置两个参数，一个叫**tcp_tw_reuse**，另一个叫**tcp_tw_recycle**的参数，这两个参数默认值都是被关闭的，后者recyle比前者resue更为激进，resue要温柔一些。另外，如果使用tcp_tw_reuse，必需设置tcp_timestamps=1，否则无效。这里，你一定要注意，**打开这两个参数会有比较大的坑——可能会让TCP连接出一些诡异的问题**（因为如上述一样，如果不等待超时重用连接的话，新的连接可能会建不上。正如[官方文档](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)上说的一样“**It should not be changed without advice/request of technical experts**”）。

	-   **关于tcp_tw_reuse**。官方文档上说tcp_tw_reuse 加上tcp_timestamps（又叫PAWS, for Protection Against Wrapped Sequence Numbers）可以保证协议的角度上的安全，但是你需要tcp_timestamps在两边都被打开（你可以读一下[tcp_twsk_unique](http://lxr.free-electrons.com/ident?i=tcp_twsk_unique)的源码 ）。我个人估计还是有一些场景会有问题。
	-   **关于tcp_tw_recycle**。如果是tcp_tw_recycle被打开了话，会假设对端开启了tcp_timestamps，然后会去比较时间戳，如果时间戳变大了，就可以重用。但是，如果对端是一个NAT网络的话（如：一个公司只用一个IP出公网）或是对端的IP被另一台重用了，这个事就复杂了。建链接的SYN可能就被直接丢掉了（你可能会看到connection time out的错误）（如果你想观摩一下Linux的内核代码，请参看源码 [tcp_timewait_state_process](http://lxr.free-electrons.com/ident?i=tcp_timewait_state_process)）。
	-   **关于tcp_max_tw_buckets**。这个是控制并发的TIME_WAIT的数量，默认值是180000，如果超限，那么，系统会把多的给destory掉，然后在日志里打一个警告（如：time wait bucket table overflow），官网文档说这个参数是用来对抗DDoS攻击的。也说的默认值180000并不小。这个还是需要根据实际情况考虑。

**Again，使用tcp_tw_reuse和tcp_tw_recycle来解决TIME_WAIT的问题是非常非常危险的，因为这两个参数违反了TCP协议（[RFC 1122](https://tools.ietf.org/html/rfc1122)）** 

其实，TIME_WAIT表示的是你主动断连接，所以，这就是所谓的“不作死不会死”。试想，如果让对端断连接，那么这个破问题就是对方的了，呵呵。另外，如果你的服务器是于HTTP服务器，那么设置一个[HTTP的KeepAlive](https://en.wikipedia.org/wiki/HTTP_persistent_connection)有多重要（浏览器会重用一个TCP连接来处理多个HTTP请求），然后让客户端去断链接（你要小心，浏览器可能会非常贪婪，他们不到万不得已不会主动断连接）。

# TCP 最大段大小

最大段大小是指 TCP 协议所允许的从对方接收到的最大报文段，因此这也是通信对方在发送数据时能够使用的最大报文段。根据 [RFCO879]，最大段大小只记录 TCP 数据的字节数而不包括其他相关的 TCP 与 IP 头部。当建立一条 TCP 连接时，通信的每一方都要在 SYN 报文段的 MSS 选项中说明自已允许的最大段大小。这 16 位的选项能够说明最大段大小的数值。在没有事先指明的情况下，最大段大小的默认数值为 536 字节。任何主机都应该能够处理至少 576 字节的 IPv4 数据报。如果接照最小的 IPv4 与 TCP 头部计算， TCP 协议要求在每次发送时的最大段大小为 536 字节，这样就正好能够组成一个 576 (20+20+536=576)字节的 IPv4 数据报。

**最大段大小的数值为 1460。 这是 IPv4 协议中的典型值**，因此 IPv4 数据报的大小也相应增加 40 个字节(总共 1500 字节，以太网中最大传输单元与互联网路径最大传输单元的典型数值): 20 字节的 TCP 头部加 20 字节的 IP 头部。

当使用 IPv6 协议时，最大段大小通常为 1440 字节。由于 IPv6 的头部比 IPv4 多 20 个字节，因此最大段大小的数值相 应减少 20 字节。在 [RFC2675] 中 65535 是一个特殊数值，与 IPv6 超长数据报一起用来指定一个表示无限大的有效最大段大小值。在这种情况下，发送方的最大段大小等于路径 MTU 的数值减去 60 字节(40 字节用于 IPv6 头部， 20 字节用于 TCP 头部)。值得注意的是，最大段大小并不是 TCP 通信双方的协商结果，而是一个限定的数值。当通信的一方将自已的最大段大小选项发送给对方时，它已表明自已不愿意在整个连接过程中接收任何大于该尺寸的报文段。

[![](https://github.com/halfrost/Halfrost-Field/raw/master/contents/images/TCP-IP-package.png)](https://github.com/halfrost/Halfrost-Field/blob/master/contents/images/TCP-IP-package.png)

# 数据传输中的Sequence Number
下图是我从Wireshark中截了个我在访问coolshell.cn时的有数据传输的图给你看一下，SeqNum是怎么变的。（使用Wireshark菜单中的Statistics ->Flow Graph… ）
![[Pasted image 20221008122354.png]]
你可以看到，**SeqNum的增加是和传输的字节数相关的**。上图中，三次握手后，来了两个Len:1440的包，而第二个包的SeqNum就成了1441。然后第一个ACK回的是1441，表示第一个1440收到了。

**注意**：如果你用Wireshark抓包程序看3次握手，你会发现SeqNum总是为0，不是这样的，Wireshark为了显示更友好，使用了Relative SeqNum——相对序号，你只要在右键菜单中的protocol preference 中取消掉就可以看到“Absolute SeqNum”了

# TCP重传机制
TCP要保证所有的数据包都可以到达，所以，必需要有重传机制。

注意，接收端给发送端的Ack确认只会确认最后一个连续的包，比如，发送端发了1,2,3,4,5一共五份数据，接收端收到了1，2，于是回ack 3，然后收到了4（注意此时3没收到），此时的TCP会怎么办？我们要知道，因为正如前面所说的，**SeqNum和Ack是以字节数为单位，所以ack的时候，不能跳着确认，只能确认最大的连续收到的包**，不然，发送端就以为之前的都收到了。

## 超时重传机制

一种是不回ack，死等3，当发送方发现收不到3的ack超时后，会重传3。一旦接收方收到3后，会ack 回 4——意味着3和4都收到了。

但是，这种方式会有比较严重的问题，那就是因为要死等3，所以会导致4和5即便已经收到了，而发送方也完全不知道发生了什么事，因为没有收到Ack，所以，发送方可能会悲观地认为也丢了，所以有可能也会导致4和5的重传。

对此有两种选择：

-   一种是仅重传timeout的包。也就是第3份数据。
-   另一种是重传timeout后所有的数据，也就是第3，4，5这三份数据。

这两种方式有好也有不好。第一种会节省带宽，但是慢，第二种会快一点，但是会浪费带宽，也可能会有无用功。但总体来说都不好。因为都在等timeout，timeout可能会很长（在下篇会说TCP是怎么动态地计算出timeout的）

## 快速重传机制

于是，TCP引入了一种叫**Fast Retransmit** 的算法，**不以时间驱动，而以数据驱动重传**。也就是说，如果，包没有连续到达，就ack最后那个可能被丢了的包，如果发送方连续收到3次相同的ack，就重传。Fast Retransmit的好处是不用等timeout了再重传。

比如：如果发送方发出了1，2，3，4，5份数据，第一份先到送了，于是就ack回2，结果2因为某些原因没收到，3到达了，于是还是ack回2，后面的4和5都到了，但是还是ack回2，因为2还是没有收到，于是发送端收到了三个ack=2的确认，知道了2还没有到，于是就马上重转2。然后，接收端收到了2，此时因为3，4，5都收到了，于是ack回6。示意图如下：
![[Pasted image 20221008122710.png]]
Fast Retransmit只解决了一个问题，就是timeout的问题，它依然面临一个艰难的选择，就是，是重传之前的一个还是重传所有的问题。对于上面的示例来说，是重传#2呢还是重传#2，#3，#4，#5呢？因为发送端并不清楚这连续的3个ack(2)是谁传回来的？也许发送端发了20份数据，是#6，#10，#20传来的呢。这样，发送端很有可能要重传从2到20的这堆数据（这就是某些TCP的实际的实现）。可见，这是一把双刃剑。

## SACK 方法

另外一种更好的方式叫：**Selective Acknowledgment (SACK)**（参看[RFC 2018](https://tools.ietf.org/html/rfc2018)），这种方式需要在TCP头里加一个SACK的东西，ACK还是Fast Retransmit的ACK，SACK则是汇报收到的数据碎版。参看下图：
![[Pasted image 20221008122910.png]]

这样，在发送端就可以根据回传的SACK来知道哪些数据到了，哪些没有到。于是就优化了Fast Retransmit的算法。当然，这个协议需要两边都支持。在 Linux下，可以通过**tcp_sack**参数打开这个功能（Linux 2.4后默认打开）。

这里还需要注意一个问题——**接收方Reneging，所谓Reneging的意思就是接收方有权把已经报给发送端SACK里的数据给丢了**。这样干是不被鼓励的，因为这个事会把问题复杂化了，但是，接收方这么做可能会有些极端情况，比如要把内存给别的更重要的东西。**所以，发送方也不能完全依赖SACK，还是要依赖ACK，并维护Time-Out，如果后续的ACK没有增长，那么还是要把SACK的东西重传，另外，接收端这边永远不能把SACK的包标记为Ack。**

注意：SACK会消费发送方的资源，试想，如果一个攻击者给数据发送方发一堆SACK的选项，这会导致发送方开始要重传甚至遍历已经发出的数据，这会消耗很多发送端的资源。详细的东西请参看《[TCP SACK的性能权衡](https://www.ibm.com/developerworks/cn/linux/l-tcp-sack/)》
### Duplicate SACK – 重复收到数据的问题

Duplicate SACK又称D-SACK，**其主要使用了SACK来告诉发送方有哪些数据被重复接收了**。[RFC-2883](https://www.ietf.org/rfc/rfc2883.txt) 里有详细描述和示例。下面举几个例子（来源于[RFC-2883](https://www.ietf.org/rfc/rfc2883.txt)）

D-SACK使用了SACK的第一个段来做标志，

-   如果SACK的第一个段的范围被ACK所覆盖，那么就是D-SACK

-   如果SACK的第一个段的范围被SACK的第二个段覆盖，那么就是D-SACK

**示例一：ACK丢包**

下面的示例中，丢了两个ACK，所以，发送端重传了第一个数据包（3000-3499），于是接收端发现重复收到，于是回了一个SACK=3000-3500，因为ACK都到了4000意味着收到了4000之前的所有数据，所以这个SACK就是D-SACK——旨在告诉发送端我收到了重复的数据，而且我们的发送端还知道，数据包没有丢，丢的是ACK包。
```bash
  Transmitted  Received    ACK Sent
  Segment      Segment     (Including SACK Blocks)

  3000-3499    3000-3499   3500 (ACK dropped)
  3500-3999    3500-3999   4000 (ACK dropped)
  3000-3499    3000-3499   4000, SACK=3000-3500
                                        ---------
```
**示例二，网络延误**

下面的示例中，网络包（1000-1499）被网络给延误了，导致发送方没有收到ACK，而后面到达的三个包触发了“Fast Retransmit算法”，所以重传，但重传时，被延误的包又到了，所以，回了一个SACK=1000-1500，因为ACK已到了3000，所以，这个SACK是D-SACK——标识收到了重复的包。

这个案例下，发送端知道之前因为“Fast Retransmit算法”触发的重传不是因为发出去的包丢了，也不是因为回应的ACK包丢了，而是因为网络延时了。
```bash
    Transmitted    Received    ACK Sent
    Segment        Segment     (Including SACK Blocks)

    500-999        500-999     1000
    1000-1499      (delayed)
    1500-1999      1500-1999   1000, SACK=1500-2000
    2000-2499      2000-2499   1000, SACK=1500-2500
    2500-2999      2500-2999   1000, SACK=1500-3000
    1000-1499      1000-1499   3000
                   1000-1499   3000, SACK=1000-1500
                                          ---------
```

---------

可见，引入了D-SACK，有这么几个好处：

1）可以让发送方知道，是发出去的包丢了，还是回来的ACK包丢了。

2）是不是自己的timeout太小了，导致重传。

3）网络上出现了先发的包后到的情况（又称reordering）

4）网络上是不是把我的数据包给复制了。

 **知道这些东西可以很好得帮助TCP了解网络情况，从而可以更好的做网络上的流控**。

Linux下的tcp_dsack参数用于开启这个功能（Linux 2.4后默认打开）

# TCP的RTT算法

从前面的TCP重传机制我们知道Timeout的设置对于重传非常重要。

-   设长了，重发就慢，丢了老半天才重发，没有效率，性能差；
-   设短了，会导致可能并没有丢就重发。于是重发的就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发。

而且，这个超时时间在不同的网络的情况下，根本没有办法设置一个死的值。只能动态地设置。 为了动态地设置，TCP引入了RTT——Round Trip Time，也就是一个数据包从发出去到回来的时间。这样发送端就大约知道需要多少的时间，从而可以方便地设置Timeout——RTO（Retransmission TimeOut），以让我们的重传机制更高效。 听起来似乎很简单，好像就是在发送端发包时记下t0，然后接收端再把这个ack回来时再记一个t1，于是RTT = t1 – t0。没那么简单，这只是一个采样，不能代表普遍情况。

##### 经典算法

[RFC793](https://tools.ietf.org/html/rfc793) 中定义的经典算法是这样的：

1）首先，先采样RTT，记下最近好几次的RTT值。

2）然后做平滑计算SRTT（ Smoothed RTT）。公式为：（其中的 α 取值在0.8 到 0.9之间，这个算法英文叫Exponential weighted moving average，中文叫：加权移动平均）

**SRTT = ( α * SRTT ) + ((1- α) * RTT)**

3）开始计算RTO。公式如下：

**RTO = min [ UBOUND,  max [ LBOUND,   (β * SRTT) ]  ]**

其中：

-   UBOUND是最大的timeout时间，上限值
-   LBOUND是最小的timeout时间，下限值
-   β 值一般在1.3到2.0之间。

##### Karn / Partridge 算法

但是上面的这个算法在重传的时候会出有一个终极问题——你是用第一次发数据的时间和ack回来的时间做RTT样本值，还是用重传的时间和ACK回来的时间做RTT样本值？

这个问题无论你选那头都是按下葫芦起了瓢。 如下图所示：

-   情况（a）是ack没回来，所以重传。如果你计算第一次发送和ACK的时间，那么，明显算大了。
-   情况（b）是ack回来慢了，但是导致了重传，但刚重传不一会儿，之前ACK就回来了。如果你是算重传的时间和ACK回来的时间的差，就会算短了。

![](https://coolshell.cn/wp-content/uploads/2014/05/Karn-Partridge-Algorithm.jpg)

所以1987年的时候，搞了一个叫[Karn / Partridge Algorithm](https://en.wikipedia.org/wiki/Karn's_Algorithm)，这个算法的最大特点是——**忽略重传，不把重传的RTT做采样**（你看，你不需要去解决不存在的问题）。

但是，这样一来，又会引发一个大BUG——**如果在某一时间，网络闪动，突然变慢了，产生了比较大的延时，这个延时导致要重转所有的包（因为之前的RTO很小），于是，因为重转的不算，所以，RTO就不会被更新，这是一个灾难**。 于是Karn算法用了一个取巧的方式——只要一发生重传，就对现有的RTO值翻倍（这就是所谓的 Exponential backoff），很明显，这种死规矩对于一个需要估计比较准确的RTT也不靠谱。

##### Jacobson / Karels 算法

前面两种算法用的都是“加权移动平均”，这种方法最大的毛病就是如果RTT有一个大的波动的话，很难被发现，因为被平滑掉了。所以，1988年，又有人推出来了一个新的算法，这个算法叫Jacobson / Karels Algorithm（参看[RFC6289](https://tools.ietf.org/html/rfc6298)）。这个算法引入了最新的RTT的采样和平滑过的SRTT的差距做因子来计算。 公式如下：（其中的DevRTT是Deviation RTT的意思）

**SRTT** **= S****RTT** **+ α** **(****RTT** **– S****RTT****)**  —— 计算平滑RTT

**DevRTT** **= (1-β****)*****DevRTT** **+ β*****(|****RTT-SRTT****|)** ——计算平滑RTT和真实的差距（加权移动平均）

**RTO= µ * SRTT + ∂ *DevRTT** —— 神一样的公式

（其中：在Linux下，α = 0.125，β = 0.25， μ = 1，∂ = 4 ——这就是算法中的“调得一手好参数”，nobody knows why, it just works…） 最后的这个算法在被用在今天的TCP协议中（Linux的源代码在：[tcp_rtt_estimator](http://lxr.free-electrons.com/source/net/ipv4/tcp_input.c?v=2.6.32#L609)）。

# TCP滑动窗口

需要说明一下，如果你不了解TCP的滑动窗口这个事，你等于不了解TCP协议。我们都知道，**TCP必需要解决的可靠传输以及包乱序（reordering）的问题**，所以，TCP必需要知道网络实际的数据处理带宽或是数据处理速度，这样才不会引起网络拥塞，导致丢包。

所以，TCP引入了一些技术和设计来做网络流控，Sliding Window是其中一个技术。 前面我们说过，**TCP头里有一个字段叫Window，又叫Advertised-Window，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据**。**于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来**。 为了说明滑动窗口，我们需要先看一下TCP缓冲区的一些数据结构：

![](https://coolshell.cn/wp-content/uploads/2014/05/sliding_window.jpg)

上图中，我们可以看到：

-   接收端LastByteRead指向了TCP缓冲区中读到的位置，NextByteExpected指向的地方是收到的连续包的最后一个位置，LastByteRcved指向的是收到的包的最后一个位置，我们可以看到中间有些数据还没有到达，所以有数据空白区。

-   发送端的LastByteAcked指向了被接收端Ack过的位置（表示成功发送确认），LastByteSent表示发出去了，但还没有收到成功确认的Ack，LastByteWritten指向的是上层应用正在写的地方。

于是：

-   接收端在给发送端回ACK中会汇报自己的AdvertisedWindow = MaxRcvBuffer – LastByteRcvd – 1;

-   而发送方会根据这个窗口来控制发送数据的大小，以保证接收方可以处理。

下面我们来看一下发送方的滑动窗口示意图：

![](https://coolshell.cn/wp-content/uploads/2014/05/tcpswwindows.png)

（[图片来源](http://www.tcpipguide.com/free/t_TCPSlidingWindowAcknowledgmentSystemForDataTranspo-6.htm)）

上图中分成了四个部分，分别是：（其中那个黑模型就是滑动窗口）

-   #1已收到ack确认的数据。
-   #2发还没收到ack的。
-   #3在窗口中还没有发出的（接收方还有空间）。
-   #4窗口以外的数据（接收方没空间）

下面是个滑动后的示意图（收到36的ack，并发出了46-51的字节）：

![](https://coolshell.cn/wp-content/uploads/2014/05/tcpswslide.png)

下面我们来看一个接受端控制发送端的图示：

![](https://coolshell.cn/wp-content/uploads/2014/05/tcpswflow.png)

（[图片来源](http://www.tcpipguide.com/free/t_TCPWindowSizeAdjustmentandFlowControl-2.htm)）

##### Zero Window

上图，我们可以看到一个处理缓慢的Server（接收端）是怎么把Client（发送端）的TCP Sliding Window给降成0的。此时，你一定会问，如果Window变成0了，TCP会怎么样？是不是发送端就不发数据了？是的，发送端就不发数据了，你可以想像成“Window Closed”，那你一定还会问，如果发送端不发数据了，接收方一会儿Window size 可用了，怎么通知发送端呢？

解决这个问题，TCP使用了Zero Window Probe技术，缩写为ZWP，也就是说，发送端在窗口变成0后，会发ZWP的包给接收方，让接收方来ack他的Window尺寸，一般这个值会设置成3次，第次大约30-60秒（不同的实现可能会不一样）。如果3次过后还是0的话，有的TCP实现就会发RST把链接断了。

**注意**：只要有等待的地方都可能出现DDoS攻击，Zero Window也不例外，一些攻击者会在和HTTP建好链发完GET请求后，就把Window设置为0，然后服务端就只能等待进行ZWP，于是攻击者会并发大量的这样的请求，把服务器端的资源耗尽。（关于这方面的攻击，大家可以移步看一下[Wikipedia的SockStress词条](https://en.wikipedia.org/wiki/Sockstress)）

另外，Wireshark中，你可以使用tcp.analysis.zero_window来过滤包，然后使用右键菜单里的follow TCP stream，你可以看到ZeroWindowProbe及ZeroWindowProbeAck的包。

##### Silly Window Syndrome

Silly Window Syndrome翻译成中文就是“糊涂窗口综合症”。正如你上面看到的一样，如果我们的接收方太忙了，来不及取走Receive Windows里的数据，那么，就会导致发送方越来越小。到最后，如果接收方腾出几个字节并告诉发送方现在有几个字节的window，而我们的发送方会义无反顾地发送这几个字节。

要知道，我们的TCP+IP头有40个字节，为了几个字节，要达上这么大的开销，这太不经济了。

另外，你需要知道网络上有个MTU，对于以太网来说，MTU是1500字节，除去TCP+IP头的40个字节，真正的数据传输可以有1460，这就是所谓的MSS（Max Segment Size）注意，TCP的RFC定义这个MSS的默认值是536，这是因为 [RFC 791](https://tools.ietf.org/html/rfc791)里说了任何一个IP设备都得最少接收576尺寸的大小（实际上来说576是拨号的网络的MTU，而576减去IP头的20个字节就是536）。

**如果你的网络包可以塞满MTU，那么你可以用满整个带宽，如果不能，那么你就会浪费带宽**。（大于MTU的包有两种结局，一种是直接被丢了，另一种是会被重新分块打包发送） 你可以想像成一个MTU就相当于一个飞机的最多可以装的人，如果这飞机里满载的话，带宽最高，如果一个飞机只运一个人的话，无疑成本增加了，也而相当二。

所以，**Silly Windows Syndrome这个现像就像是你本来可以坐200人的飞机里只做了一两个人**。 要解决这个问题也不难，就是避免对小的window size做出响应，直到有足够大的window size再响应，这个思路可以同时实现在sender和receiver两端。

-   如果这个问题是由Receiver端引起的，那么就会使用 David D Clark’s 方案。在receiver端，如果收到的数据导致window size小于某个值，可以直接ack(0)回sender，这样就把window给关闭了，也阻止了sender再发数据过来，等到receiver端处理了一些数据后windows size 大于等于了MSS，或者，receiver buffer有一半为空，就可以把window打开让send 发送数据过来。

-   如果这个问题是由Sender端引起的，那么就会使用著名的 [Nagle’s algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm "Nagle's algorithm")。这个算法的思路也是延时处理，他有两个主要的条件：1）要等到 Window Size>=MSS 或是 Data Size >=MSS，2）收到之前发送数据的ack回包，他才会发数据，否则就是在攒数据。

另外，Nagle算法默认是打开的，所以，对于一些需要小包场景的程序——**比如像telnet或ssh这样的交互性比较强的程序，你需要关闭这个算法**。你可以在Socket设置TCP_NODELAY选项来关闭这个算法（关闭Nagle算法没有全局参数，需要根据每个应用自己的特点来关闭）

setsockopt(sock_fd, IPPROTO_TCP, TCP_NODELAY, (char *)&value,sizeof(int));

另外，网上有些文章说TCP_CORK的socket option是也关闭Nagle算法，这不对。**TCP_CORK其实是更新激进的Nagle算汉，完全禁止小包发送，而Nagle算法没有禁止小包发送，只是禁止了大量的小包发送**。最好不要两个选项都设置。

# TCP的拥塞处理 – Congestion Handling

上面我们知道了，TCP通过Sliding Window来做流控（Flow Control），但是TCP觉得这还不够，因为Sliding Window需要依赖于连接的发送端和接收端，其并不知道网络中间发生了什么。TCP的设计者觉得，一个伟大而牛逼的协议仅仅做到流控并不够，因为流控只是网络模型4层以上的事，TCP的还应该更聪明地知道整个网络上的事。

具体一点，我们知道TCP通过一个timer采样了RTT并计算RTO，但是，**如果网络上的延时突然增加，那么，TCP对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，于是，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的TCP连接都这么行事，那么马上就会形成“网络风暴”，TCP这个协议就会拖垮整个网络。**这是一个灾难。

所以，TCP不能忽略网络上发生的事情，而无脑地一个劲地重发数据，对网络造成更大的伤害。对此TCP的设计理念是：**TCP不是一个自私的协议，当拥塞发生的时候，要做自我牺牲。就像交通阻塞一样，每个车都应该把路让出来，而不要再去抢路了。**

关于拥塞控制的论文请参看《[Congestion Avoidance and Control](http://ee.lbl.gov/papers/congavoid.pdf)》(PDF)

拥塞控制主要是四个算法：**1）慢启动**，**2）拥塞避免**，**3）拥塞发生**，**4）快速恢复**。这四个算法不是一天都搞出来的，这个四算法的发展经历了很多时间，到今天都还在优化中。 备注:

-   1988年，TCP-Tahoe 提出了1）慢启动，2）拥塞避免，3）拥塞发生时的快速重传
-   1990年，TCP Reno 在Tahoe的基础上增加了4）快速恢复

##### 慢热启动算法 – Slow Start

首先，我们来看一下TCP的慢热启动。慢启动的意思是，刚刚加入网络的连接，一点一点地提速，不要一上来就像那些特权车一样霸道地把路占满。新同学上高速还是要慢一点，不要把已经在高速上的秩序给搞乱了。

慢启动的算法如下(cwnd全称Congestion Window)：

1）连接建好的开始先初始化cwnd = 1，表明可以传一个MSS大小的数据。

2）每当收到一个ACK，cwnd++; 呈线性上升

3）每当过了一个RTT，cwnd = cwnd*2; 呈指数让升

4）还有一个ssthresh（slow start threshold），是一个上限，当cwnd >= ssthresh时，就会进入“拥塞避免算法”（后面会说这个算法）

所以，我们可以看到，如果网速很快的话，ACK也会返回得快，RTT也会短，那么，这个慢启动就一点也不慢。下图说明了这个过程。

![](https://coolshell.cn/wp-content/uploads/2014/05/tcp.slow_.start_.jpg)

这里，我需要提一下的是一篇Google的论文《[An Argument for Increasing TCP’s Initial Congestion Window](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/36640.pdf)》Linux 3.0后采用了这篇论文的建议——把cwnd 初始化成了 10个MSS。 而Linux 3.0以前，比如2.6，Linux采用了[RFC3390](https://www.rfc-editor.org/rfc/rfc3390.txt)，cwnd是跟MSS的值来变的，如果MSS< 1095，则cwnd = 4；如果MSS>2190，则cwnd=2；其它情况下，则是3。

#####  拥塞避免算法 – Congestion Avoidance

前面说过，还有一个ssthresh（slow start threshold），是一个上限，当cwnd >= ssthresh时，就会进入“拥塞避免算法”。一般来说ssthresh的值是65535，单位是字节，当cwnd达到这个值时后，算法如下：

1）收到一个ACK时，cwnd = cwnd + 1/cwnd

2）当每过一个RTT时，cwnd = cwnd + 1

这样就可以避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。很明显，是一个线性上升的算法。

##### 拥塞状态时的算法

前面我们说过，当丢包的时候，会有两种情况：

1）等到RTO超时，重传数据包。TCP认为这种情况太糟糕，反应也很强烈。

-   sshthresh =  cwnd /2
-   cwnd 重置为 1
-   进入慢启动过程

2）Fast Retransmit算法，也就是在收到3个duplicate ACK时就开启重传，而不用等到RTO超时。

-   TCP Tahoe的实现和RTO超时一样。

-   TCP Reno的实现是：
    -   cwnd = cwnd /2
    -   sshthresh = cwnd
    -   进入快速恢复算法——Fast Recovery

上面我们可以看到RTO超时后，sshthresh会变成cwnd的一半，这意味着，如果cwnd<=sshthresh时出现的丢包，那么TCP的sshthresh就会减了一半，然后等cwnd又很快地以指数级增涨爬到这个地方时，就会成慢慢的线性增涨。我们可以看到，TCP是怎么通过这种强烈地震荡快速而小心得找到网站流量的平衡点的。

##### 快速恢复算法 – Fast Recovery

**TCP Reno**

这个算法定义在[RFC5681](https://tools.ietf.org/html/rfc5681 ""TCP Congestion Control"")。快速重传和快速恢复算法一般同时使用。快速恢复算法是认为，你还有3个Duplicated Acks说明网络也不那么糟糕，所以没有必要像RTO超时那么强烈。 注意，正如前面所说，进入Fast Recovery之前，cwnd 和 sshthresh已被更新：

-   cwnd = cwnd /2
-   sshthresh = cwnd

然后，真正的Fast Recovery算法如下：

-   cwnd = sshthresh  + 3 * MSS （3的意思是确认有3个数据包被收到了）
-   重传Duplicated ACKs指定的数据包
-   如果再收到 duplicated Acks，那么cwnd = cwnd +1
-   如果收到了新的Ack，那么，cwnd = sshthresh ，然后就进入了拥塞避免的算法了。

如果你仔细思考一下上面的这个算法，你就会知道，**上面这个算法也有问题，那就是——它依赖于3个重复的Acks**。注意，3个重复的Acks并不代表只丢了一个数据包，很有可能是丢了好多包。但这个算法只会重传一个，而剩下的那些包只能等到RTO超时，于是，进入了恶梦模式——超时一个窗口就减半一下，多个超时会超成TCP的传输速度呈级数下降，而且也不会触发Fast Recovery算法了。

通常来说，正如我们前面所说的，SACK或D-SACK的方法可以让Fast Recovery或Sender在做决定时更聪明一些，但是并不是所有的TCP的实现都支持SACK（SACK需要两端都支持），所以，需要一个没有SACK的解决方案。而通过SACK进行拥塞控制的算法是FACK（后面会讲）

**TCP New Reno**

于是，1995年，TCP New Reno（参见 [RFC 6582](https://tools.ietf.org/html/rfc6582) ）算法提出来，主要就是在没有SACK的支持下改进Fast Recovery算法的——

-   当sender这边收到了3个Duplicated Acks，进入Fast Retransimit模式，开发重传重复Acks指示的那个包。如果只有这一个包丢了，那么，重传这个包后回来的Ack会把整个已经被sender传输出去的数据ack回来。如果没有的话，说明有多个包丢了。我们叫这个ACK为Partial ACK。

-   一旦Sender这边发现了Partial ACK出现，那么，sender就可以推理出来有多个包被丢了，于是乎继续重传sliding window里未被ack的第一个包。直到再也收不到了Partial Ack，才真正结束Fast Recovery这个过程

我们可以看到，这个“Fast Recovery的变更”是一个非常激进的玩法，他同时延长了Fast Retransmit和Fast Recovery的过程。

##### 算法示意图

下面我们来看一个简单的图示以同时看一下上面的各种算法的样子：

![](https://coolshell.cn/wp-content/uploads/2014/05/tcp.fr_-1024x359.jpg)

##### FACK算法

FACK全称Forward Acknowledgment 算法，论文地址在这里（PDF）[Forward Acknowledgement: Refining TCP Congestion Control](http://conferences.sigcomm.org/sigcomm/1996/papers/mathis.pdf) 这个算法是其于SACK的，前面我们说过SACK是使用了TCP扩展字段Ack了有哪些数据收到，哪些数据没有收到，他比Fast Retransmit的3 个duplicated acks好处在于，前者只知道有包丢了，不知道是一个还是多个，而SACK可以准确的知道有哪些包丢了。 所以，SACK可以让发送端这边在重传过程中，把那些丢掉的包重传，而不是一个一个的传，但这样的一来，如果重传的包数据比较多的话，又会导致本来就很忙的网络就更忙了。所以，FACK用来做重传过程中的拥塞流控。

-   这个算法会把SACK中最大的Sequence Number 保存在**snd.fack**这个变量中，snd.fack的更新由ack带秋，如果网络一切安好则和snd.una一样（snd.una就是还没有收到ack的地方，也就是前面sliding window里的category #2的第一个地方）

-   然后定义一个**awnd = snd.nxt – snd.fack**（snd.nxt指向发送端sliding window中正在要被发送的地方——前面sliding windows图示的category#3第一个位置），这样awnd的意思就是在网络上的数据。（所谓awnd意为：actual quantity of data outstanding in the network）

-   如果需要重传数据，那么，**awnd = snd.nxt – snd.fack + retran_data**，也就是说，awnd是传出去的数据 + 重传的数据。

-   然后触发Fast Recovery 的条件是： ( **( snd.fack – snd.una ) > (3*MSS)** ) || (dupacks == 3) ) 。这样一来，就不需要等到3个duplicated acks才重传，而是只要sack中的最大的一个数据和ack的数据比较长了（3个MSS），那就触发重传。在整个重传过程中cwnd不变。直到当第一次丢包的snd.nxt<=snd.una（也就是重传的数据都被确认了），然后进来拥塞避免机制——cwnd线性上涨。

我们可以看到如果没有FACK在，那么在丢包比较多的情况下，原来保守的算法会低估了需要使用的window的大小，而需要几个RTT的时间才会完成恢复，而FACK会比较激进地来干这事。 但是，FACK如果在一个网络包会被 reordering的网络里会有很大的问题。

# 四. CLOSE_WAIT过多的解决方法

情景描述：系统产生大量 Too many open files

原因分析：在服务器与客户端通信过程中，因服务器发生了 socket 未关导致的 closed_wait 发生，致使监听 port 打开的句柄数到了 1024 个，且均处于 close_wait 的状态，最终造成配置的port被占满出现 “Too many open files”，无法再进行通信。  
close_wait 状态出现的原因是被动关闭方未关闭socket造成，如附件图所示：
解决办法：有两种措施可行

一、解决： 原因是因为调用 ServerSocket 类的 accept() 方法和 Socket 输入流的 read() 方法时会引起线程阻塞，所以应该用 setSoTimeout() 方法设置超时（缺省的设置是0，即超时永远不会发生）；超时的判断是累计式的，一次设置后，每次调用引起的阻塞时间都从该值中扣除，直至另一次超时设置或有超时异常抛出。  
比如，某种服务需要三次调用 read()，超时设置为1分钟，那么如果某次服务三次 read()调用的总时间超过 1 分钟就会有异常抛出，如果要在同一个 Socket 上反复进行这种服务，就要在每次服务之前设置一次超时。

二、规避：  
调整系统参数，包括句柄相关参数和TCP/IP的参数；

注意：  
/proc/sys/fs/file-max 是整个系统可以打开的文件数的限制，由 sysctl.conf 控制； ulimit 修改的是当前 shell 和它的子进程可以打开的文件数的限制，由 limits.conf 控制； lsof 是列出系统所占用的资源，但是这些资源不一定会占用打开文件号的；比如：共享内存，信号量，消息队列，内存映射等，虽然占用了这些资源，但不占用打开文件号；  
因此，需要调整的是当前用户的子进程打开的文件数的限制，即 limits.conf 文件的配置； 如果 `cat /proc/sys/fs/file-max` 值为 65536 或甚至更大，不需要修改该值； 若 ulimit -a ；其 open files 参数的值小于 4096（默认是1024)， 则采用如下方法修改 open files 参数值为8192；方法如下：

1.使用root登陆，修改文件 /etc/security/limits.conf vi /etc/security/limits.conf 添加 xxx - nofile 8192 xxx 是一个用户，如果是想所有用户生效的话换成 * ，设置的数值与硬件配置有关，别设置太大了。
![[Pasted image 20220527152941.png]]

2.使这些限制生效 确定文件 /etc/pam.d/login 和 /etc/pam.d/sshd 包含如下行： session required pam_limits.so 然后用户重新登陆一下即可生效。 3. 在 bash 下可以使用 ulimit -a 参看是否已经修改：

一、 修改方法：（暂时生效，重新启动服务器后，会还原成默认值）

sysctl -w net.ipv4.tcp_keepalive_time=600   
sysctl -w net.ipv4.tcp_keepalive_probes=2 
sysctl -w net.ipv4.tcp_keepalive_intvl=2 

注意：Linux 的内核参数调整的是否合理要注意观察，看业务高峰时候效果如何。

二、 若做如上修改后，可起作用；则做如下修改以便永久生效。 vi /etc/sysctl.conf

若配置文件中不存在如下信息，则添加：

net.ipv4.tcp_keepalive_time = 1800 
net.ipv4.tcp_keepalive_probes = 3 
net.ipv4.tcp_keepalive_intvl = 15 

编辑完 /etc/sysctl.conf ，要重启 network 才会生效 /etc/rc.d/init.d/network restart  
然后，执行 sysctl 命令使修改生效，基本上就算完成了。

---

修改原因：

当客户端因为某种原因先于服务端发出了 FIN 信号，就会导致服务端被动关闭，若服务端不主动关闭 socket 发 FIN 给 Client，此时服务端 Socket 会处于 CLOSE_WAIT 状态（而不是 LAST_ACK 状态）。通常来说，一个 CLOSE_WAIT 会维持至少 2 个小时的时间（系统默认超时时间的是 7200 秒，也就是 2 小时）。如果服务端程序因某个原因导致系统造成一堆 CLOSE_WAIT 消耗资源，那么通常是等不到释放那一刻，系统就已崩溃。因此，解决这个问题的方法还可以通过修改 TCP/IP 的参数来缩短这个时间，于是修改 tcp_keepalive_*系列参数：

tcp_keepalive_time：  
/proc/sys/net/ipv4/tcp_keepalive_time  
INTEGER，默认值是7200(2小时)  
当 keepalive 打开的情况下，TCP 发送 keepalive 消息的频率。建议修改值为 1800秒。

tcp_keepalive_probes：INTEGER  
/proc/sys/net/ipv4/tcp_keepalive_probes  
INTEGER，默认值是9  
TCP 发送 keepalive 探测以确定该连接已经断开的次数。(注意:保持连接仅在SO_KEEPALIVE 套接字选项被打开是才发送.次数默认不需要修改，当然根据情形也可以适当地缩短此值。设置为 5 比较合适)

tcp_keepalive_intvl：INTEGER  
/proc/sys/net/ipv4/tcp_keepalive_intvl  
INTEGER，默认值为75  
当探测没有确认时，重新发送探测的频度。探测消息发送的频率（在认定连接失效之前，发送多少个 TCP 的 keepalive 探测包）。乘以 tcp_keepalive_probes 就得到对于从开始探测以来没有响应的连接杀除的时间。默认值为 75 秒，也就是没有活动的连接将在大约 11 分钟以后将被丢弃。(对于普通应用来说，这个值有一些偏大，可以根据需要改小.特别是 web 类服务器需要改小该值，15 是个比较合适的值)

【检测办法】

1.  系统不再出现“Too many open files”报错现象。
    
2.  处于 TIME_WAIT 状态的 sockets 不会激长。
    

在 Linux 上可用以下语句看了一下服务器的 TCP 状态(连接状态数量统计)：

netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}' 

返回结果范例如下：

ESTABLISHED 1423  
FIN_WAIT1 1  
FIN_WAIT2 262  
SYN_SENT 1  
TIME_WAIT 962

# TIME_WAIT 状态

TIME_WAIT 状态也称为 2MSL 等待状态。在该状态中， TCP 将会等待两倍于最大段生存期(Maximum Segment Lifetime， MSL)的时间，有时也被称作加倍等待。每个实现都必 须为最大段生存期选择一个数值。它代表任何报文段在被丢弃前在网络中被允许存在的最长 时间。我们知道这个时限是有限制的，因为 TCP 报文段是以 IP 数据报的形式传输的， IP数据报拥有 TTL 字段和跳数限制字段。这两个字段限制了 IP 数据报的有效生存时间。

[RFC0793] 将最大段生存期设为 2 分钟。然而在常见实现中，最大段生存期的数值可 以为 30 秒、 1 分钟或者 2 分钟。在绝大多数情况下，这一数值是可以修改的。在 Linux系统中， `net.ipv4.tcp_fin_timeout` 的数值记录了 2MSL 状态需要等待的超时时间(以秒为单位)。 在 Windows 系统，下面的注册键值也保存了超时时间:

HKLM\SYSTEM\currentcontrolSet\Services\Tcpip\parameters\TcpTimedWaitDelay

该键值的取值范围是30 - 300秒。对于 IPv6 而言，只需要将键值中的 `Tcpip` 替换为`Tcpip6` 即可。

至于为什么要设置 2MSL 等待时间，答案可以看[这里](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/TCP:IP.md#%E4%B8%BA%E4%BB%80%E4%B9%88%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%87%8A%E6%94%BE%E6%9C%80%E5%90%8E%E9%9C%80%E8%A6%81-time-wait-%E7%AD%89%E5%BE%85-2msl-%E5%91%A2)

# TCP 重置报文段

TCP 头部的 RST 位字段。一个将该字段置位的报文段被称作“重置报文 段”或简称为“重置” 。一般来说，当发现一个到达的报文段对于相关连接而言是不正确的时， TCP 就会发送一个重置报文段。 (此处，相关连接是指由重置报文段的 TCP 与 IP 头部的4元组所指定的连接)。重置报文段通常会导致 TCP 连接的快速拆卸。

重置报文会有以下的用途：

-   1.  针对不存在端口的连接请求。
-   2.  终止一条连接。
-   3.  半开连接。
-   4.  时间等待错误。

解释一下时间等待错误：

客户端处于 TIME_WAIT 阶段，这个时候突然接收到服务器发过来的旧报文段时， TCP 会发送一个 ACK 作为响应，其中包含了最新的序列号与 ACK 号(分别是 K 与 L)。然而，当服务器接收到这个报文段以后， 它没有关于这条连接的任何信息，因此发送一个重置报文段作为响应。这并不是服务器的问题，但它却会使客户端过早地从 TIME_WAIT 状态转移至 CLOSED 状态。许多系统规定当处于 TIME_WAIT 状态时不对重置报文段做出反应，从而避免了上述间题。

#  Nagle 算法

在 ssh 连接中，通常单次击键就会引发数据流的传输。如果使用 IPv4，一次按键会生成约 88 字节大小的 TCP/IPv4 包(使用加密和认证) : 20 字节的 IP头部， 20 字节的 TCP 头部(假设没有选项)，数据部分为 48 字节。这些小包(称为微型报 tinygram )会造成相当高的网络传输代价。也就是说，与包的其他部分相比，有效的应用数据所占比例甚微。该问题对于局域网不会有很大影响，因为大部分局域网不存在拥塞， 而且这些包无须传输很远。然而对于广域网来说则会加重拥塞，严重影响网络性能。John Nagle 在 [RFCO896] 中提出了一种简单有效的解决方法，现在称其为 Nagle 算法。

Nagle 算法要求，当一个 TCP 连接中有在传数据(即那些已发送但还未经确认的数据)， 小的报文段(长度小于 SMSS )就不能被发送，直到所有的在传数据都收到ACK。并且，在收到 ACK 后， TCP 需要收集这些小数据，将其整合到一个报文段中发送。这种方法迫使 TCP 遵循停等( stop-and-Wait )规程一只有等接收到所有在传数据的 ACK 后才能继续发送。该算法的精妙之处在于它实现了自时钟(self-CIocking)控制: ACK 返回越快，数据传输也越快。在相对高延迟的广域网中，更需要减少微型报的数目，该算法使得单位时间内发 送的报文段数目更少。也就是说， RTT 控制着发包速率。

## Nagle 算法和延时 ACK 带来的问题

若将延时 ACK 与 Nagle 算法直接结合使用，得到的效果可能不尽如人意。考虑如下情形，客户端使用延时 ACK 方法发送一个对服务器的请求，而服务器端的响应数据并不适合在同一个包中传输。

[![](https://github.com/halfrost/Halfrost-Field/raw/master/contents/images/nagle.png)](https://github.com/halfrost/Halfrost-Field/blob/master/contents/images/nagle.png)

从上图中可以看到，在接收到来自服务器端的两个包以后，客户端并不立即发送 ACK，而是处于等待状态，希望有数据一同捎带发送。通常情况下， TCP 在接收到两个全长的数据包后就应返回一个 ACK，但这里并非如此。在服务器端，由于使用了 Nagle 算法，直到收到 ACK 前都不能发送新数据，因为任一时刻只允许至多一个包在传。因此延时 ACK 与 Nagle 算法的结合导致了某种程度的死锁(两端互相等待对方做出行动) [MMSV99] [MMO1]。幸运的是，这种死锁并不是永久的，在延时 ACK 计时器超时后死锁会解除。客户端即使仍然没有要发送的数据也无需再等待，而可以只发送 ACK 给服务器。然而，在死锁期间整个传输连接处于空闲状态，使性能变差。在某些情况下，如这里的 ssh 传输，可以禁用 Nagle 算法。

**要求延迟较小的应用，实时网络游戏等，都要禁用这个 Nagle 算法**。

禁用 Nagle 算法有多种方式，主机需求 RFC [RFCl122] 列出了相关方法。若某个应用使 用 Berkeley 套接字 API，可以设置 TCP_NODELAY 选项。另外，也可以在整个系统中禁用该算法。在 Windows 系统中，使用如下的注册表项:

    HKLM\SOFTWARE\Microsoft\MSMQ\parameters\TCPNoDelay

这个双字节类型的值必须由用户添加，应将其设为1。为使更改生效，消息队列也需要重新设置。

# TCP 安全协议与分层

链路层的安全服务致力于保护一跳通信中的信息，网络层的安全服务致力于保护两个主机之间传输的信息，传输层的安全服务致力于保护进程与进程之间的通信，应用层的安全服务致力于保护应用程序操纵的信息。在通信的备层次中，由独立于通信层的应用来负责保护数据的工作也是可以的(例如，文件能够经过加密并以电子邮件附件的 方式发送出去)。

[![](https://github.com/halfrost/Halfrost-Field/raw/master/contents/images/tcp_sec.png)](https://github.com/halfrost/Halfrost-Field/blob/master/contents/images/tcp_sec.png)

上图展示了与 TCP/IP 结合使用的最常见的安全协议。

有些安全协议会针对某一个协议层，而有一些安全协议则会跨越多个协议层。虽然不会 像 TCP/IP 协议那样被经常讨论，但是一些链路技术(包括它们自已的加密与认证协议)从第 2 层起就开始了保障安全的工作。在 TCP/IP 协议中， EAP 用于建立包含多种机制的身份验证，比如机器证书、用户证书、智能卡、密码等。EAP 常用于拥有后台认证或 AAA 服务器 的企业设置。EAP 还可以用于其他协议的认证，比如 IPsec。

IPsec 是一个提供第 3 层安全的协议集合，其中包括 IKE、 AH 以及ESP。IKE 建立并管理双方之间的安全关联。安全关联涉及认证(AH)或加密(ESP)，并且能够运行于传输或隧道模式。在传输模式中，会修改 IP 头部以进行认证或加密，而在隧道模式中， IP 数据报会被完全放置在一个新的IP数据报中。ESP 是 IPsec 最流行的协议。所有的 IPsec 协议能够使用不同的算洼与参数(加密套件)进行加密、完整性保护、 DH密钥协商以及身份认证。

沿协议栈向上查看，传输层安全(当前版本为TLS l.3)保护了两个应用程序之间的信息。它拥有自已的内部分层，包含一个记录层协议和三个信息交换协议:密码更改协议、警告协议、握手协议。此外，记录协议支持应用数据。记录层负责根据握手协议提供的参数加密数据并保障它们的完整性。密码更改协议用于将之前设定的挂起协议状态更改为活动协议状态。警告协议会指出错误或连接间题。与 TCP/IP 一起使用的 TLS 是使用最广泛的安全 协议，并且它还支持加密的 Web 测览器连接(HTTPS)。TLS 的一个变种称为 DTLS，它将 TLS 应用于数据报协议，比如 UDP 与 DCCP。

#  短连接，并行连接，持久连接与长连接

## [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E7%9F%AD%E8%BF%9E%E6%8E%A5)短连接

短连接多用于操作频繁，点对点的通讯，而且连接数不能太多的情况。每个 TCP 连接的建立都需要三次握手，每个 TCP 连接的断开要四次挥手。适用于并发量大，但是每个用户又不需频繁操作的情况。

但是在用户需要频繁操作的业务场景下(如新用户注册，网购提交订单等)，频繁的使用短连接则会使性能时延产生叠加。

用户登录这些不频繁的操作可以考虑用短连接。

## [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E5%B9%B6%E8%A1%8C%E8%BF%9E%E6%8E%A5)并行连接

针对短连接，人们想出了优化的办法，连接多条，形成并行连接。

并行连接允许客户端打开多条连接，并行地执行多个事务，每个事务都有自己的TCP连接。这样可以克服单条连接的空载时间和带宽限制，时延可以重叠起来，而且如果单条连接没有充分利用客户端的网络带宽，可以将未用带宽分配来装载其他对象。

在PC时代，利用并行连接来充分利用现代浏览器的多线程并发下载能力的场景非常广泛。

但是并行连接也会产生一定的问题，首先并行连接不一定更快，因为带宽资源有限，每个连接都会去竞争这有限的带宽，这样带来的性能提升就很小，甚至没什么提升。

**一般机器上面并行连接的条数 4 - 6 条**。

## [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E6%8C%81%E4%B9%85%E8%BF%9E%E6%8E%A5)持久连接

HTTP1.0 版本以后，允许 HTTP 设备在事务处理结束之后将 TCP 连接保持在打开状态，以便为未来的 HTTP 请求重用现存的连接。在事务处理结束之后仍然保持在打开状态的 TCP 连接被称为持久连接。

持久连接的时间参数，通常由服务器设定，比如 nginx 的 keepalivetimeout，keepalive timout 时间值意味着：一个 http 产生的 tcp 连接在传送完最后一个响应后，还需要 hold 住 keepalive_timeout 秒后，才开始关闭这个连接；

**在 HTTP 1.1 中 所有的连接默认都是持续连接**，除非特殊声明不支持。HTTP 持久连接不使用独立的 keepalive 信息，而是仅仅允许多个请求使用单个连接。然而，Apache 2.0 httpd 的默认连接过期时间是仅仅 15 秒，对于 Apache 2.2 只有 5 秒。短的过期时间的优点是能够快速的传输多个 web 页组件，而不会绑定多个服务器进程或线程太长时间。

持久连接与并行连接相比，带来的优势如下：

1.  避免了每个事务都会打开/关闭一条新的连接，造成时间和带宽的耗费；
2.  避免了 TCP 慢启动特性的存在导致的每条新连接的性能降低；
3.  可打开的并行连接数量实际上是有限的，持久连接则可以减少建立的连接的数量；

## [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E9%95%BF%E8%BF%9E%E6%8E%A5)长连接

  长连接与持久连接本质上非常的相似，持久连接侧重于 HTTP 应用层，特指一次请求结束之后，服务器会在自己设置的 keepalivetimeout 时间到期后才关闭已经建立的连接。长连接则是 client 方与 server 方先建立连接，连接建立后不断开，然后再进行报文发送和接收，直到有一方主动关闭连接为止。

长连接的适用场景也非常的广泛：

1.  监控系统：后台硬件热插拔、LED、温度、电压发生变化等；
2.  IM 应用：收发消息的操作；
3.  即时报价系统：例如股市行情 push 等；
4.  推送服务：各种 App 内置的 push 提醒服务；

## [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E5%8D%81-%E5%BF%83%E8%B7%B3%E9%97%AE%E9%A2%98)十. 心跳问题

### [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E5%BF%83%E8%B7%B3%E7%9A%84%E5%BF%85%E8%A6%81%E6%80%A7)心跳的必要性

虽然TCP提供了KeepAlive机制，但是并不能替代应用层心跳保活。原因主要如下：

(1) Keep Alive 机制开启后，TCP 层将在定时时间到后发送相应的 KeepAlive 探针以确定连接可用性。默认时间为7200s(两小时)，失败后重试10次，每次超时时间75s。显然默认值无法满足移动网络下的需求；

(2) 即便修改了(1)中的默认值，也不能很好的满足业务需求。TCP 的 KeepAlive 用于检测连接的死活而不能检测通讯双方的存活状态。比如某台服务器因为某些原因导致负载超高，无法响应任何业务请求，但是使用 TCP 探针则仍旧能够确定连接状态，这就是典型的连接活着但业务提供方已死的状态，对客户端而言，这时的最好选择就是断线后重新连接其他服务器，而不是一直认为当前服务器是可用状态，一直向当前服务器发送些必然会失败的请求。

(3) socks 代理会让 Keep Alive 失效。socks 协议只管转发 TCP 层具体的数据包，而不会转发 TCP 协议内的实现细节的包。所以，一个应用如果使用了 socks 代理，那么 TCP 的 KeepAlive 机制就失效了。

(4) 部分复杂情况下 Keep Alive 会失效，如路由器挂掉，网线直接被拔除等；

**KeepAlive 并不适用于检测双方存活的场景，这种场景还得依赖于应用层的心跳。应用层心跳也具备着更大的灵活性，可以控制检测时机，间隔和处理流程，甚至可以在心跳包上附带额外信息。**

### [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E5%BF%83%E8%B7%B3%E6%97%B6%E9%97%B4%E5%BD%B1%E5%93%8D%E5%9B%A0%E7%B4%A0)心跳时间影响因素

应用层心跳是检测连接有效性以及判断双方是否存活的有效方式。但是心跳过于频繁会带来耗电和耗流量的弊病，心跳频率过低则会影响连接检测的实时性。业内关于心跳时间的设置和优化，主要基于如下几个因素：

1.  NAT 超时–大部分移动无线网络运营商在链路一段时间没有数据通讯时，会淘汰 NAT 表中的对应项，造成链路中断；
2.  DHCP 租期– DHCP 租期到了需要主动续约，否则会继续使用过期IP导致长连接偶然的断连；
3.  网络状态变化–手机网络和 WIFI 网络切换、网络断开和连上等情况有网络状态的变化，也会使长连接变为无效连接；

以下是网络运营商的一些 NAT 超时时间。

地区/网络

NAT超时时间

备注

中国移动3G和2G

5分钟

中国联通2G

5分钟

中国电信3G

大于28分钟

美国3G

大于28分钟

台湾3G

大于28分钟

所以一般心跳包设置时间都在 3 分钟左右。

### [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E6%99%BA%E8%83%BD%E5%BF%83%E8%B7%B3)智能心跳

a）延迟心跳测试法：这是测试结果准确的前提保障，我们认为长连接建立后连续三次成功的短心跳就可以很大程度的保证下一次心跳环境是正常的。

b）成功一次认定，失败连续累积认定：成功是绝对的，连续失败多次才可能是失败。

c）临界值避免：我们使用比计算出的心跳稍微小一点的值做为稳定心跳避免临界值。

d）动态调整：即使在一次完整的智能心跳计算过程中，我们没有找到最好的值，我们还有机会来进行校正。

需要心跳包的背景：

a. 运营商的信令风暴。  
b. 运营商网络换代，NAT超时趋于增大。  
c. Alarm耗电，心跳耗流量。

动态心跳引入下列状态：

a. 前台活跃态：亮屏，微信在前台， 周期minHeart (4.5min) ，保证体验。  
b. 后台活跃态：微信在后台10分钟内，周期minHeart ，保证体验。  
c. 自适应计算态：步增心跳，尝试获取最大心跳周期(sucHeart)。  
d. 后台稳定态：通过最大周期，保持稳定心跳。

老版本微信心跳时间维持在 4.5 分钟，即 270 s 左右。

策略可以如下：

初始心跳 180 s，每次发送心跳包都延迟 30 s，如果每次心跳包都能成功发送，就一直往后延，这样做的目的是为了找到最长心跳时间。一旦发现连接失败，就重连，重连上来以后，先把断线以前累计的心跳时长减少 20 s，再尝试以这个新的心跳时长发送心跳。逐渐找到一个最佳值。如果连续 5 次都连接失败，再次重连以初始心跳 180 s 尝试重连。

## [](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md#%E5%8D%81%E4%B8%80-qq%E5%BE%AE%E4%BF%A1)十一. QQ，微信

早期的时候，QQ 还是主要使用 TCP 协议，而后来就转向了采用 UDP 的方式来保持在线，TCP 的方式来上传和下载数据。现在，UDP 是 QQ 的默认工作方式，表现良好。相信这个也被沿用到了微信上。

简单的考证：登录 PC 版 QQ，关闭多余的 QQ 窗口只留下主窗口，并将其最小化。几分钟过后，查看系统网络连接，会发现 QQ 进程已不保有任何 TCP 连接，但有 UDP 网络活动。这时在发送聊天信息，或者打开其他窗口和功能，将发现 QQ 进程会启用 TCP 连接。

登陆成功之后，QQ 有一个 TCP 连接来保持在线状态。这个 TCP 连接的远程端口一般是80，采用 UDP 方式登陆的时候，端口是 8000。

QQ 客户端之间的消息传送采用了 UDP，因为国内的网络环境非常复杂，而且很多用户采用的方式是通过代理服务器共享一条线路上网的方式，在这些复杂的情况下，客户端之间能彼此建立起来 TCP 连接的概率较小，严重影响传送信息的效率。而 UDP 包能够穿透大部分的代理服务器，因此 QQ 选择了 UDP 作为客户之间的主要通信协议。

腾讯采用了上层协议来保证可靠传输：如果客户端使用 UDP 协议发出消息后，服务器收到该包，需要使用 UDP 协议发回一个应答包。如此来保证消息可以无遗漏传输。之所以会发生在客户端明明看到“消息发送失败”但对方又收到了这个消息的情况，就是因为客户端发出的消息服务器已经收到并转发成功，但客户端由于网络原因没有收到服务器的应答包引起的。

总结一下:
登陆采用 TCP 协议和 HTTP 协议，好友之间发送消息，主要采用 UDP 协议，内网传文件采用了 P2P ，不需要服务器中转。

-----------------------------------------
# Reference
https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/Advance_TCP.md

https://coolshell.cn/articles/11564.html