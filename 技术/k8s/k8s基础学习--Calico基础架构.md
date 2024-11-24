#k8s  #网络 

**什么是Calico**

  

Calico是一个开源网络和网络安全解决方案，适用于容器、虚拟机和基于主机的本地工作负载。Calico支持广泛的平台，包括Kubernetes、OpenShift、Mirantis Kubernetes Engine (MKE)、OpenStack和裸机服务。（翻译来自Calico官网[[https://docs.projectcalico.org/archive/v3.20/about/about-calico](https://link.zhihu.com/?target=https%3A//docs.projectcalico.org/archive/v3.20/about/about-calico)]） Calico使用标准的Linux网络工具为云本地应用程序提供两种主要服务：

  

1.工作负载之间的网络连接。

2.工作负载之间的网络安全策略实施。

  

**Calico架构**

  

![](https://pic4.zhimg.com/80/v2-c99bfafc2d0b36f16cf5afc576a28843_720w.jpg)

  

由于Calico是一种纯三层的实现，因此可以避免与二层方案相关的数据包封装的操作，中间没有任何的NAT，没有任何的overlay，所以它的转发效率可能是所有方案中最高的，因为它的包直接走原生TCP/IP的协议栈，它的隔离也因为这个栈而变得好做。因为TCP/IP的协议栈提供了一整套的防火墙的规则，所以它可以通过IPTABLES的规则达到比较复杂的隔离逻辑。

  

Calico网络模型主要工作组件：

  

1.Felix：运行在每一台Host的agent进程，主要负责网络接口管理和监听、路由、ARP管理、ACL管理和同步、状态上报等。Felix会监听Etcd中心的存储，从它获取事件，比如说用户在这台机器上加了一个IP，或者是创建了一个容器等，用户创建Pod后，Felix负责将其网卡、IP、MAC都设置好，然后在内核的路由表里面写一条，注明这个IP应该到这张网卡。同样，用户如果制定了隔离策略，Felix同样会将该策略创建到ACL中，以实现隔离。

  

2.etcd：分布式键值存储，主要负责网络元数据一致性，确保Calico网络状态的准确性，可以与kubernetes共用。

  

3.BGP Client（BIRD）：Calico为每一台Host部署一个BGP Client，使用BIRD实现，BIRD是一个单独的持续发展的项目，实现了众多动态路由协议比如BGP、OSPF、RIP等。在Calico的角色是监听Host上由Felix注入的路由信息，然后通过BGP协议广播告诉剩余Host节点，从而实现网络互通。BIRD是一个标准的路由程序，它会从内核里面获取哪一些IP的路由发生了变化，然后通过标准BGP的路由协议扩散到整个其他的宿主机上，让外界都知道这个IP在这里，你们路由的时候得到这里来。

  

4.BGP Route Reflector：在大型网络规模中，如果仅仅使用BGP client形成mesh全网互联的方案就会导致规模限制，因为所有节点之间俩俩互联，需要N^2个连接，为了解决这个规模问题，可以采用BGP的Router Reflector的方法，使所有BGP Client仅与特定RR节点互联并做路由同步，从而大大减少连接数。

  

**Node之间的两种网络**

  

**IPIP 工作模式**

  

简单的来说，IPIP模式就是把IP层封装到IP层的一个tunnel。它的作用其实基本上就相当于一个基于IP层的网桥。一般来说，普通的网桥是基于mac层的，根本不需IP，而这个ipip则是通过两端的路由做一个tunnel，把两个本来不通的网络通过点对点连接起来。

  

1.1 IPIP 连接方式

  

![](https://pic3.zhimg.com/80/v2-449deec322ac5550d374e47b0f4d9152_720w.jpg)

  

1.2 IPIP测试

  

同节点不同Pod通信。

  

reso-service和lcs-time在同一个node节点slave-2上。

  

![](https://pic4.zhimg.com/80/v2-8ddd41313e90708d40828fed90859a83_720w.jpg)

  

```text
kubectl exec -it lsh-mcp-lcs-reso-service-6bcc475cbc-bhzrx bash   # 进入lsh-mcp-lcs-reso-serviceservice
```

  

通过ifconfig查看网卡，查询到mac地址（EA:18:B7:86:A4:A9）和32位掩码的主机地址（inet addr:20.201.157.181 Bcast:0.0.0.0 Mask:255.255.255.255）。

  

![](https://pic1.zhimg.com/80/v2-e22958dfea4b1be507049ebc4a3ed388_720w.jpg)

  

通过命令route -n查看路由，网关地址位169.254.1.1。

![](https://pic1.zhimg.com/80/v2-136a4086263e4a16a68f6cec910f3b34_720w.jpg)

  

在Calico网络中，每台主机作为它所承载的工作负载的网关路由器。在容器部署中，Calico使用169.254.1.1作为Calico路由器的地址。通过使用链路本地地址，Calico节省了宝贵的IP地址，并避免了用户配置合适地址的负担。

  

虽然路由表对于习惯于配置LAN网络的人来说可能有点奇怪，但是在WAN网络中使用显式路由而不是子网-本地网关是相当普遍的。

  

进入另外一个pod（lsh-mcp-lcs-timer-5686fd9cc-f7vxh）。

  

通过`route -n` 的命令查看其路由。

  

![](https://pic4.zhimg.com/80/v2-9bc73b333288c60dc736fdda08dfd3ff_720w.jpg)

  

Calico努力避免干扰主机上的任何其他配置。Calico没有将网关地址添加到每个工作负载接口的主机端，而是在接口上设置proxy_arp标志。这使得主机表现得像一个网关，响应169.254.1.1的arp，而不需要实际分配IP地址到接口。因此在主机上是看不到169.254.1.1的地址。

  

接着观察一下主机上的路由目录。  

`route -n` 如下所示。

  

![](https://pic4.zhimg.com/80/v2-412d40c1a21f7b3c60363e4ea6ef6a6b_720w.jpg)

  

其中20.201.157.181为pod（lsh-mcp-lcs-reso-service），20.201.157.178为pod（lsh-mcp-lcs-timer），对此我们可以画一下我们的网络拓扑图。

  

![](https://pic3.zhimg.com/80/v2-023e631c268a32bd41467e38ec23648a_720w.jpg)

  

此时Linux Host就扮演者一个Router的角色。那么问题就变成了路由上的两台主机的通信问题。

  

不同节点不同Pod通信

  

环境准备：

  

一个master，两个node节点，node节点分别为slave-1和slave-2。

  

pod1（ lsh-mcp-iam-apigateway-service）位于slave-1上。

  

![](https://pic1.zhimg.com/80/v2-c8f1b8bb0783bc58ce20e2d4ff16f30c_720w.jpg)

  

pod2（lsh-mcp-lcs-reso-service）位于slave-2上。

  

![](https://pic1.zhimg.com/80/v2-78111cb9b6af4779fb27d87d856f48cc_720w.jpg)

  

查看pod 1的ifconfig。

  

```text
kubectl exec -it lsh-mcp-iam-apigateway-service-9d86698bd-8xvxp -- ifconfig
```

  

![](https://pic1.zhimg.com/80/v2-e22958dfea4b1be507049ebc4a3ed388_720w.jpg)

  

查看pod 2的ifconfig。

  

![](https://pic4.zhimg.com/80/v2-47ab70848984e24efd96c1df820ae6bb_720w.jpg)

  

pod2 ping pod1。

  

![](https://pic1.zhimg.com/80/v2-e5ce9eccb5c5cfa8d527c3e26cf165c4_720w.jpg)

  

根据pod2上路由。

  

![](https://pic1.zhimg.com/80/v2-136a4086263e4a16a68f6cec910f3b34_720w.jpg)

  

我们可以得出ping 20.201.27.28的数据包都会发往网关169.254.1.1，然后从eth0网卡发送出去。

  

![](https://pic2.zhimg.com/80/v2-bafb4fa991502d062ff04132cec2d721_720w.jpg)

  

其中pod1在slave-1（8.16.0.168）上，pod2在slave-2（8.16.0.169）上，当pod2 ping pod1时，在slave-1的主机上则会匹配到20.201.27.0/26这条路由上去，数据包就通过设备tunl0发往到slave-1这个节点上。

  

接着查看slave-1上的路由。

  

![](https://pic2.zhimg.com/80/v2-6282426ae64e109ba8d414d595d2461d_720w.jpg)

  

slave-1接收到数据包之后，发往目的IP：20.201.27.28，通过查看路由发现这个地址是与主机直连的，因此所有的数据包会发往caliccfd1b15368，这个设备就是veth pair的一端。在创建pod1时calico会给pod1创建一个veth pair设备，一端是pod1的网卡，另外一端是caliccfd1b15368。可以安装ethtool工具，然后使用ethtool -S eth0，查看veth pair另一端的设备号。

  

**BGP 工作模式**

  

边界网关协议（Border Gateway Protocol, BGP）是互联网上一个核心的去中心化自治路由协议。它通过维护IP路由表或‘前缀’表来实现自治系统（AS）之间的可达性，属于矢量路由协议。BGP不使用传统的内部网关协议（IGP）的指标，而使用基于路径、网络策略或规则集来决定路由。因此，它更适合被称为矢量性协议，而不是路由协议。BGP，通俗的讲就是讲接入到机房的多条线路（如电信、联通、移动等）融合为一体，实现多线单IP，BGP 机房的优点：服务器只需要设置一个IP地址，最佳访问路由是由网络上的骨干路由器根据路由跳数与其它技术指标来确定的，不会占用服务器的任何系统。

  

**两种网络对比**

  

IPIP网络：

  

流量：tunl0设备封装数据，形成隧道，承载流量。

  

适用网络类型：适用于互相访问的Pod不在同一个网段中，跨网段访问的场景，外层封装的IP能够解决跨网段的路由问题。

  

效率：流量需要tunl0设备封装，效率略低。

  

BGP网络：

  

流量：使用路由信息导向流量。

  

使用网络类型：适用于互相访问的Pod在同一个网段。

  

效率：原生hostGW，效率高。

# Reference
https://zhuanlan.zhihu.com/p/443307407