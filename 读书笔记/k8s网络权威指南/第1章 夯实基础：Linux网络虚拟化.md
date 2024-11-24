#k8s  #网络

# 1.1 网络虚拟化基石：network namespace
顾名思义，Linux的namespace（名字空间）的作用就是“隔离内核资源”。在Linux的世界里，文件系统挂载点、主机名、POSIX进程间通信消息队列、进程PID数字空间、IP地址、user ID数字空间等全局系统资源被namespace分割，装到一个个抽象的独立空间里。而隔离上述系统资源的namespace分别是Mount namespace、UTSnamespace、IPC namespace、PIDnamespace、network namespace和usernamespace。对进程来说，要想使用namespace里面的资源，首先要“进入”（具体操作方法，下文会介绍）到这个namespace，而且还无法跨namespace访问资源。Linux的namespace给里面的进程造成了两个错觉：
（1）它是系统里唯一的进程。
（2）它独享系统的所有资源。

说到network namespace，它在Linux内核2.6版本引入，作用是隔离Linux系统的设备，以及IP地址、端口、路由表、防火墙规则等网络资源。因此，每个网络namespace里都有自己的网络设备（如IP地址、路由表、端口范围、/proc/net目录等）。从网络的角度看，network namespace使得容器非常有用，一个直观的例子就是：由于每个容器都有自己的（虚拟）网络设备，并且容器里的进程可以放心地绑定在端口上而不必担心冲突，这就使得在一个主机上同时运行多个监听80端口的Web服务器变为可能

和其他namespace一样，network namespace可以通过系统调用来创建，我们可以调用Linux的clone（）（其实是UNIX系统调用fork（）的延伸）API创建一个通用的namespace，然后传入CLONE_NEWNET参数表面创建一个networknamespace。
创建一个名为netns1的network namespace可以使用以下命令：
```bash
ip netns add netns1
```
当ip命令创建了一个network namespace时，系统会在/var/run/netns路径下面生成一个挂载点。挂载点的作用一方面是方便对namespace的管理，另一方面是使namespace即使没有进程运行也能继续存在。
一个network namespace被创建出来后，可以使用ip netns exec命令进入，做一些网络查询/配置的工作。
![[Pasted image 20220424113455.png]]
如上所示，就是进入netns1这个network namespace查询网卡信息的命令。目前，我们没有任何配置，因此只有一块系统默认的本地回环设备lo。
想查看系统中有哪些network namespace，可以使用以下命令：
```bash
ip netns list
```
想删除network namespace，可以通过以下命令实现：
```bash
ip netns delete netns1
```
注意，上面这条命令实际上并没有删除netns1这个network namespace，它只是移除了这个network namespace对应的挂载点（下文会解释）。只要里面还有进程运行着，networknamespace便会一直存在。

## 1.1.2 配置network namespace
通过上文的阅读我们已经知道，一个全新的network namespace会附带创建一个本地回环地址。除此之外，没有任何其他的网络设备。而且，细心的读者应该已经发现，network namespace自带的lo设备状态还是DOWN的，因此，当尝试访问本地回环地址时，网络也是不通的。下面的小测试就说明了这一点。
![[Pasted image 20220424114410.png]]
在我们的例子中，如果想访问本地回环地址，首先需要进入netns1这个network namespace，把设备状态设置成UP。
```bash
ip netns exec netns1 ip link set dev lo up
```
但是，仅有一个本地回环设备是没法与外界通信的。如果我们想与外界（比如主机上的网卡）进行通信，就需要在namespace里再创建一对虚拟的以太网卡，即所谓的veth pair。顾名思义，vethpair总是成对出现且相互连接，它就像Linux的双向管道（pipe），报文从veth pair一端进去就会由另一端收到。关于veth pair更详细的介绍，参见1.2节，本节不再赘述。
下面的命令将创建一对虚拟以太网卡，然后把vethpair的一端放到netns1 network namespace。
![[Pasted image 20220424114607.png]]
如上所示，我们创建了veth0和veth1这么一对虚拟以太网卡。在默认情况下，它们都在主机的根network namespce中，将其中一块虚拟网卡veth1通过ip link set命令移动到netns1 networknamespace。那么，veth0和veth1之间能直接通信吗？还不能，因为这两块网卡刚创建出来还都是DOWN状态，需要手动把状态设置成UP。这个步骤的操作和上文对lo网卡的操作类似，只是多了一步绑定IP地址，如下所示：
![[Pasted image 20220424114655.png]]
另外，不同network namespace之间的路由表和防火墙规则等也是隔离的，因此我们刚刚创建的netns1 network namespace没法和主机共享路由表和防火墙，这一点通过下面的测试就能说明。
需要注意的是，用户可以随意将虚拟网络设备分配到自定义的network namespace里，而连接真实硬件的物理设备则只能放在系统的根networknamesapce中。并且，任何一个网络设备最多只能存在于一个network namespace中。


```bash
ip netns exec netns1 ip link set veth1 netns 1
```
该怎么理解上面这条看似有点复杂的命令呢？分解成两部分：
（1）ip netns exec netns1进入netns1 networknamespace。
（2）ip link set veth1 netns 1把netns1network namespace下的veth1网卡挪到PID为1的进程（即init进程）所在的networknamespace。

通常，init进程都在主机的根networknamespace下运行，因此上面这条命令其实就是把veth1从netns1 network namespace移动到系统根network namespace。有两种途径索引network namespace：名字（例如netns1）或者属于该namespace的进程PID，上文中用的就是后者。

如果用户希望屏蔽这一行为，则需要结合PIDnamespace和Mount namespace的隔离特性做到network namespace之间的完全不可达

## 1.1.3 network namespace API的使用
### 1、 创建namespace的黑科技：clone系统调用
### 2、 维持namespace存在：/proc/PID/ns目录的奥秘
每个Linux进程都拥有一个属于自己的/proc/PID/ns，这个目录下的每个文件都代表一个类型的namespace。
Linux内核提供的黑科技允许：只要打开文件描述符，不需要进程存在也能保持namespace存在！
### 3、 往namespace里添加进程：setns系统调用
我们已经使用一些黑科技使得namespace即使没有进程在其中也能保持开放。接下来，我们就要往这个namespace里“扔”进程，Linux系统调用setns（）就是用来做这个工作的，其主要功能就是把一个进程加入一个已经存在的namespace中。
### 4、 帮助进程逃离namespace：unshare系统调用
与namespace相关的最后一个系统调用是unshare（），用于帮助进程“逃离”namespace。
基于unshare（）系统调用的，它的作用就是在当前shell所在的namespace外执行一条命令。unshare命令的用法如下所示：
```bash
unshare [options] program [arguments]
```
Linux会为需要执行的命令（在上面的例子中，即program）启动一个新进程，然后在另外一个namespace中执行操作，这样就可以起到执行结果和原（父）进程隔离的效果。

## 小结
我们知道通过Linux的network namespace技术可以自定义一个独立的网络栈，简单到只有loopback设备，复杂到具备系统完整的网络能力，这就使得network namespace成为Linux网络虚拟化技术的基石——不论是虚拟机还是容器时代。network namespace的另一个隔离功能在于，系统管理员一旦禁用namespace中的网络设备，即使里面的进程拿到了一些系统特权，也无法和外界通信。最后，网络对安全较为敏感，即使network namespace能够提供网络资源隔离的机制，用户还是会结合其他类型的namespace一起使用，以提供更好的安全隔离能力。

# 1.2 千呼万唤始出来：veth pair
部署过Docker或Kubernetes的读者肯定有这样的经历：在主机上输入ifconfig或ip addr命令查询网卡信息的时候，总会出来一大堆vethxxxx之类的网卡名，这些是Docker/Kubernetes为容器而创建的虚拟网卡。
veth是虚拟以太网卡（Virtual Ethernet）的缩写。veth设备总是成对的，因此我们称之为vethpair。veth pair一端发送的数据会在另外一端接收，非常像Linux的双向管道。根据这一特性，veth pair常被用于跨network namespace之间的通信，即分别将veth pair的两端放在不同的namespace里
veth pair设备的原理较简单，就是向veth pair设备的一端输入数据，数据通过内核协议栈后从vethpair的另一端出来。veth pair的基本工作原理如图1-3所示。
![[Pasted image 20220424151059.png]]

## 1.2.2 容器与host veth pair的关系
我们要学习的经典容器组网模型就是vethpair+bridge的模式。容器中的eth0实际上和外面host上的某个veth是成对的（pair）关系，那么，有没有办法知道host上的vethxxx和哪个container eth0是成对的关系呢？
### 方法1
首先，在目标容器里查看：
![[Pasted image 20220424151252.png]]
然后，在主机上遍历/sys/claas/net下面的全部目录，查看子目录ifindex的值和容器里查出来的iflink值相当的veth名字，这样就找到了容器和主机的veth pair关系。例如，下面的例子中主机上veth63a89a3的ifindex刚好是5，意味着是目标容器veth pair的另一端。
![[Pasted image 20220424151342.png]]
###  方法2
在目标容器里执行以下命令：
![[Pasted image 20220424151548.png]]
从上面的命令输出可以看到116:eth0@if117，其中116是eth0接口的index，117是和它成对的veth的index。
当host执行下面的命令时，可以看到对应117的veth网卡是哪一个，这样就得到了容器和vethpair的关系。
![[Pasted image 20220424151631.png]]
### 方法3
可以通过ethtool-S命令列出veth pair对端的网卡index，例如：
![[Pasted image 20220424151706.png]]
而主机上index为6的网卡为：
![[Pasted image 20220424151716.png]]

# 1.3 连接你我他：Linux bridge
顾名思义，Linux bridge就是Linux系统中的网桥，但是Linux bridge的行为更像是一台虚拟的网络交换机，任意的真实物理设备（例如eth0）和虚拟设备（例如，前面讲到的veth pair和后面即将介绍的tap设备）都可以连接到Linux bridge上。需要注意的是，Linux bridge不能跨机连接网络设备。
Linux bridge则有多个端口，数据可以从任何端口进来，进来之后从哪个口出去取决于目的MAC地址，原理和物理交换机差不多。

## Linux bridge初体验
我们先用iproute2软件包里的ip命令创建一个bridge：
```bash
ip link add name br0 type bridge
ip link set br0 up
```
除了ip命令，我们还可以使用bridge-utils软件包里的brctl工具管理网桥，例如新建一个网桥：
```bash
brctl addbr br0
```
![[Pasted image 20220424152528.png]]
```bash
ip link add veth0 type veth peer name veth1
ip addr add 1.2.3.101/24 dev veth0
ip addr add 1.2.3.102/24 dev veth1
ip link set veth0 up
ip link set veth1 up
ip link set dev veth0 master br0 
#(brctl addif br0 veth0)
bridge link
# brctl show

```
![[Pasted image 20220424152841.png]]
br0和veth0相连之后发生了如下变化：
·br0和veth0之间连接起来了，并且是双向的通道；
·协议栈和veth0之间变成了单通道，协议栈能发数据给veth0，但veth0从外面收到的数据不会转发给协议栈；
·br0的MAC地址变成了veth0的MAC地址。这就好比Linux bridge在veth0和协议栈之间做了一次拦截，在veth0上面做了点小动作，将veth0本来要转发给协议栈的数据拦截，全部转发给bridge。同时，bridge也可以向veth0发数据。

让我们做个小实验来验证以上观点。首先，从veth0 ping veth1：
![[Pasted image 20220424153106.png]]
如上所示，veth0 ping veth1失败。为什么veth0加入bridge之后，就ping不通对端的veth1了呢？1.2.3.102原本应该是能ping通的，让我们通过抓包深入分析。先抓veth1网卡上的报文：
![[Pasted image 20220424153152.png]]
如上所示，由于veth0的ARP缓存里没有veth1的MAC地址，所以ping之前先发ARP请求。veth1抓取的报文显示，veth1收到了ARP请求，并且返回了应答。
再抓veh0网卡上的报文：
![[Pasted image 20220424153223.png]]
如上所示，veth0上的数据包都发出去了，而且也收到了响应。再看br0上的数据包，发现只有应答，如下所示：
![[Pasted image 20220424153239.png]]
通过分析以下报文可以看出，包的去和回的流程都没有问题，问题就出在veth0收到应答包后没有给协议栈，而是给了br0，于是协议栈得不到veth1的MAC地址，导致通信失败。

## 1.3.2 把IP让给Linux bridge
通过上面的分析可以看出，给veth0配置IP没有意义，因为就算协议栈传数据包给veth0，回程报文也回不来。这里我们就把veth0的IP地址“让给”Linux bridge：
![[Pasted image 20220424153330.png]]
以上命令将原本分配给veth0的IP地址配置到br0上。于是，绑定IP地址的bridge设备的网络拓扑如图1-6所示。
![[Pasted image 20220424153347.png]]
图1-6将协议栈和veth0之间的联系去掉了，veth0相当于一根网线。实际上，veth0和协议栈之间是有联系的，但由于veth0没有配置IP，所以协议栈在路由的时候不会将数据包发给veth0。就算强制要求数据包通过veth0发送出去，由于veth0从另一端收到的数据包只会给br0，协议栈还是没法收到相应的ARP应答包，同样会导致通信失败。
这时，再通过br0 ping veth1，结果成功收到了ICMP的回程报文,但ping网关还是失败
因为这个br0上只有1.2.3.101和1.2.3.102这两个网络设备，不知道1.2.3.1在哪儿。

## 1.3.3 将物理网卡添加到Linux bridge
下面，我们演示如何将主机上的物理网卡eth0添加到Linux bridge：
![[Pasted image 20220424153617.png]]
Linux bridge不会区分接入进来的到底是物理设备还是虚拟设备，对它来说没有区别。因此，eth0加入br0后，落得和上面veth0一样的“下场”，从外面网络收到的数据包将无条件地转发给br0，自己变成了一根网线。
这时，通过eth0 ping网关失败。因为br0通过eth0这根网线连上了外面的物理交换机，所以连在br0上的设备都能ping通网关，这里连上的设备就是veth1和br0自己，veth1是通过eth0这根网线连上去的，而br0有一块自带的网卡。
通过br0 ping网关成功,通过veth1 ping网关成功,通过eth0 ping网关失败
因为eth0的功能已经和网线差不多，所以在eth0上配置IP没有意义，还会影响协议栈的路由选择。例如，如果ping的时候不指定网卡，则协议栈有可能优先选择eth0，导致ping不通。因此，需要将eth0上的IP去掉。在以上测试过程中，由于eth0上有IP，在访问1.2.3.0/24网段时，会优先选择eth0。可以通过查看主机路由表来验证我们的判断：
eth0接入了br0，因此它收到的数据包都会转发给br0，于是协议栈收不到ARP应答包，导致ping失败。
将eth0的ip删除
```bash
ip addr del 1.2.3.1/24 dev eth0
```
这时，再从eth0 ping一次网关，成功收到ICMP响应报文

当我们删除eth0的IP后，路由表里就没有它了，于是数据包会从veth1出去。可以通过查看主机路由表来验证我们的判断。

通过观察以上路由表信息可以看出：原来的默认路由进过eth0，eth0的IP被删除后，默认路由不见了，想要连接1.2.3.0/24以外的网段，需要手动将默认网关加回来。添加默认网关：
```bash
sudo ip route add default via 1.2.3.1
```
再ping外网，成功返回ICMP报文
经过上面一系列的操作，将物理网卡添加到bridge设备的网络拓扑如图1-7所示。注：要完成以上所有实验步骤，需要打开eth0网卡的混杂模式（下文会详细介绍Linux bridge的混杂模式），不然veth1的网络会不通。当eth0不在混杂模式时，只会接收目的MAC地址是自己的报文，丢掉目的MAC地址是veth1的数据包。

## 1.3.4 Linux bridge在网络虚拟化中的应用
以上例子是为了阐述Linux bridge的底层机制而设计的，下面将通过Linux bridge的两种常见的部署方式说明其在现代网络虚拟化技术中的地位。
### 1、 虚拟机
虚拟机通过tun/tap或者其他类似的虚拟网络设备，将虚拟机内的网卡同br0连接起来，这样就达到和真实交换机一样的效果，虚拟机发出去的数据包先到达br0，然后由br0交给eth0发送出去，数据包都不需要经过host机器的协议栈，效率高，如图1-8所示。如果有多个虚拟机，那么这些虚拟机通过tun/tap设备连接到网桥。tun/tap设备的详细介绍将在1.4节展开。
![[Pasted image 20220424154230.png]]
### 2.、容器
容器运行在自己单独的network namespace里，因此都有自己单独的协议栈。Linux bridge在容器场景的组网和上面的虚拟机场景差不多，但也存在一些区别。例如，容器使用的是veth pair设备，而虚拟机使用的是tun/tap设备。在虚拟机场景下，我们给主机物理网卡eth0分配了IP地址；而在容器场景下，我们一般不会对宿主机eth0进行配置。在虚拟机场景下，虚拟器一般会和主机在同一个网段；而在容器场景下，容器和物理网络不在同一个网段内。Linux bridge在容器中的应用如图1-9所示。
![[Pasted image 20220424154317.png]]
在容器中配置其网关地址为br0，在我们的例子中即1.2.3.101（容器网络网段是1.2.3.0/24）。因此，从容器发出去的数据包先到达br0，然后交给host机器的协议栈。由于目的IP是外网IP，且host机器开启了IP forward功能，数据包会通过eth0发送出去。因为容器所分配的网段一般都不在物理网络网段内（在我们的例子中，物理网络网段是10.20.30.0/24），所以一般发出去之前会先做NAT转换（NAT转换需要自己配置，可以使用iptables，1.5节会介绍iptables）。