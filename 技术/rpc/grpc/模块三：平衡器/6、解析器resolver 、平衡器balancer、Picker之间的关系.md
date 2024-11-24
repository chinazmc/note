#grpc 

  本篇文章主要是想再分享一下，解析器resolver 、平衡器balancer、Picker之间的关系，如果不是很熟悉的话，可能还是有点模糊；

# 1、整体流程说明

解析器resolver 、平衡器balancer、Picker之间的关系，如下如所示：  
![resolver、balancer、picker之间的关系](https://img-blog.csdnimg.cn/20210605094836494.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

从图中，可以看出来：

grpc客户端跟grpc服务器端处理的整个生命周期，经历了两个阶段：

-   rpc链接建立阶段
-   rpc请求阶段，或者说，流处理阶段，或者说，帧传输、处理阶段(头帧、数据帧、设置帧)

整体流程说明：

-   grpc客户端首先通过resolver接口，获得grpc服务器端的地址列表，然后触发平衡器的流程
-   平衡器根据链接策略，以及从resolver接口获取到的grpc服务器链接地址，创建addrConn负责向grpc服务器端发起tcp链接，实际使用时，一个addrConn对应一个grpc服务器地址；链接建立后，双方还需要进行帧的初始化，从而实现了rpc链接。
-   平衡器会将已经成功建立起的rpc链接缓存起来
-   grpc客户端在本地调用方法时，如调用SayHello方法时，底层会每调用一次SayHello方法，都会创建一个ClientStream与之对应，ClientStream会调用Picker接口，从缓存中选择一个rpc链接，
-   grpc客户端跟某个grpc服务器端会使用刚才选择好的rpc链接进行数据的传输、处理。

# 2、resolver接口，balancer接口，Picker接口主要用在什么阶段

resolver接口，balancer接口主要是用在建立链接阶段
Picker接口，主要是用在帧传输阶段。

# 3、Picker接口主要实现什么目的？也可以认为是链接选择器？

在帧的传输阶段，需要一个链接来传输帧，如何从众多的链接里选择一个链接，就是picker接口要做的事情。  
Picker接口的实现，需要参考balancer接口的具体实现；  
比方说，当balancer是pickerfirst的时候，其实，就不需要专门实现Picker接口了，就是从连接列表里选择第一个链接，就可以了，所以才称为first。  
当balancer是round_robin的时候，就需要专门实现Picker接口，从众多的链接里按照某种策略选择一个链接。

# 4、是否可以自定义解析器resolver,平衡器 balancer, picker呢?

从上面的图中，可以看出来，在[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)框架中，组件的整体调用流程已经定义好了，就是谁在前，谁在后，已经规定好了；resolver在前，balancer在后，最后是picker的处理。  
既然resovler, balancer, picker 这三个都是接口类型，我们就可以按照自己的业务需要实现响应的接口就可以了；

# 5、解析器、平衡器的主要调用链

从客户端的启动文件，到解析器，到平衡器的主要调用链，如下图所示:  
![resolver-balancer-pikcer调用链](https://img-blog.csdnimg.cn/20210605100624108.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

此图，希望能将grpc客户端跟grpc服务器端的链接建立过程的主要调用链表达清楚，从调用链中可以看出来主要经历了哪些阶段：

-   解析器阶段
-   平衡器阶段
-   真正rpc链接阶段
-   以及链接后，做了哪些工作，如创建帧发送器，创建帧接收器

以及不同阶段之间是通过什么来连接起来的，或者说，应该是有个桥梁将不同阶段串联起来了，  
如：

-   ClientConn结构体里的updateResolverState方法，将解析器和平衡器串联起来了
-   aacBalancerWraper结构体里的connect方法，将平衡器和真正的tcp连接的功能串联起来了。

这样的话，通过这个调用链，就会有一个整体的概念，可能会对研究分析整个连接过程有帮助。

为了快速的找到相应的代码，列出了所属的文件。
![[Pasted image 20220524105309.png]]

# 6、总结

  到目前为止，我们已经将grpc客户端跟grpc服务器端如何建立起[rpc](https://so.csdn.net/so/search?q=rpc&spm=1001.2101.3001.7020)链接的整个过程基本介绍完了；  
此过程中，涉及到的解析器，平衡器，如何真正的链接的等等，都介绍完了；并提供了相应的测试用例；

  下一阶段，我们将重点介绍，流的处理过程，比方说，重点分析一下，grpc客户端调用SayHello方法时，底层都经历了什么，

-   grpc服务器端是如何知道grpc客户端请求的哪个服务，
-   该服务哪个方法，
-   方法的具体参数值是多少等等，都会介绍。
