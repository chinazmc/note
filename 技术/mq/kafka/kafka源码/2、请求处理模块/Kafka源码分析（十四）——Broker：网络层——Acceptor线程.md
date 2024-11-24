
**_Network Layer（网络层）_**是Kafka Broker处理所有请求的入口。Kafka基于Java NIO实现了一套**Reactor线程模型**，其核心流程就是与客户端建立连接，然后对请求进行解析，封装成Request对象传递给API层，同时接受API层的处理结果，封装后响应给客户端。

本章，我将先对网络层的整体架构进行分析，然后对其中的一个核心组件——**Acceptor线程**进行讲解。

## 一、整体架构

我们来看下网络层的整体架构：

  
![](https://files.tpvlog.com/tpvlog/kafka/source/20210606223903534.png)  

整个网路层，包含的核心组件和功能说明如下：

```
SocketServer                    # 网络层的整体封装类
   |-- Acceptor                 # Acceptor线程，负责监听并建立与客户端的连接
   |-- Processor                # Processor线程，负责监听读写事件，并解析请求/响应
Selector                        # 对Java NIO中的Selector进行了封装
KafkaChannel                    # 对Java NIO中的SocketChannel进行了封装
TransportLayer                  # 对KafkaChannel屏蔽底层的字节读写
RequestChannel                  # 请求队列，是与API层进行通信交互的通道
  |-- Request                   # 传递给API层的请求Request对象
  |-- Response                  # 接受API层返回的Response对象

```

### 1.1 处理流程

我先根据上图，讲解下网络层处理请求的大概流程，便于大家有个印象，后续再对每个组件的源码进行分析：

1.  首先，每个Broker启动后，会根据`server.properties`中的参数配置创建三类核心线程：
    
    **Acceptor线程：**通过`listeners`配置，每一组IP/端口都会创建一个Endpoint对象和Acceptor线程，并建立映射，EndPoint是对端口IP的抽象；
    
    **Processor线程：**通过`num.network.threads`配置，默认3个，即一个Acceptor线程对应3个Processor线程；
    
    **RequestHandler线程：**通过`num.io.threads`配置，默认8个，被封装在一个**KafkaRequestHandlerPool线程池**中。
    
2.  Acceptor线程启动后，默认监听Broker的本机地址和9092端口，底层基于Java NIO监听Socket的连接事件`OP_ACCEPT`；
    
3.  接着，当客户端请求建立连接时，Acceptor会监听到该事件，然后完成连接的建立，并把建立好连接的SocketChannel通过Round Robin轮询的方式分配给各个Processor线程；
    
4.  每个Processor线程会把接受到的SocketChannel，缓存到自己内部的一个队列（ConcurrentLinkedQueue）中；
    
5.  当SocketChannel监听读事件`OP_READ`发生时，每个Processor会通过底层的NIO组件读取请求字节，封装成Request对象，扔到一个名为**RequestChannel**的组件中；
    
6.  RequestChannel内部有一个缓存Request请求的全局队列（ArrayBlockingQueue），默认最多可以缓存500个请求，可通过参数`queued.max.requests`配置，同时有N个（N为Processor线程的总数）缓存Reponse响应的队列（ArrayBlockingQueue）；
    
7.  接着，**KafkaRequestHandlerPool线程池**中的RequestHandler线程，会不断从RequestChannel中获取Request请求，交给Kafka API层进行处理；
    
8.  **Kafka API层**完成消息处理后，会将结果封装成Response对象，并入队到RequestChannel内部响应队列中；
    
9.  Processor线程会对RequestChannel的响应队列中的Response对象进行处理，当它内部的SocketChannel监听到`OP_WRITE`写事件后，就会解析Reponse，利用底层NIO组件响应给客户端。
    

以上就是Broker的网络层处理消息请求/响应的核心流程。可以看到，**Kafka Server使用Reactor模式，整个网络通讯架构非常清晰，Acceptor线程负责建立并分发连接，Processor线程们负责监听读写事件并解析请求和响应，同时将请求分发给工作线程RequestHandler，RequestHandler负责具体的业务处理。**这是一整套标准的Reactor模式，非常具有工业参考价值！

## 二、Acceptor线程

了解了Kafka Server网络层的整个工作流程，我们来看Acceptor线程的内部细节，因为它是处理所有请求的入口。

Acceptor线程的上述整体工作流程和内部结构，可以用下面这张图来表示：
![[Pasted image 20221115175416.png]]

### 2.1 初始化

KafkaServer在启动过程中有下面这么两行代码：

```
// KafkaServer.scala

socketServer = new SocketServer(config, metrics, time, credentialProvider)
socketServer.startup()

```

SocketServer内部封装了网络层的核心组件，启动它就是创建并启动Acceptor线程和Processer线程，并把Acceptor线程与Processor线程关联：

```
// SocketServer.scala

class SocketServer(val config: KafkaConfig, val metrics: Metrics, val time: Time, val credentialProvider: CredentialProvider) extends Logging with KafkaMetricsGroup {
    // 监听的端口/IP
    private val endpoints = config.listeners.map(l => l.listenerName -> l).toMap
    // Processor线程数，默认3
    private val numProcessorThreads = config.numNetworkThreads
    // RequstChannel最大可缓存Request的数目，默认500
    private val maxQueuedRequests = config.queuedMaxRequests
    // 总Processor线程数
    private val totalProcessorThreads = numProcessorThreads * endpoints.size

    // 每个IP最多可以建立多少个连接
    private val maxConnectionsPerIp = config.maxConnectionsPerIp
    private val maxConnectionsPerIpOverrides = config.maxConnectionsPerIpOverrides

    this.logIdent = "[Socket Server on Broker " + config.brokerId + "], "

    // RequestChannel组件
    val requestChannel = new RequestChannel(totalProcessorThreads, maxQueuedRequests)
    private val processors = new Array[Processor](totalProcessorThreads)

    // Acceptor线程和EndPoint的映射
    private[network] val acceptors = mutable.Map[EndPoint, Acceptor]()
    private var connectionQuotas: ConnectionQuotas = _

    // 启动
    def startup() {
        this.synchronized {
            connectionQuotas = new ConnectionQuotas(maxConnectionsPerIp, maxConnectionsPerIpOverrides)

            // 底层Socket发送缓存区大小
            val sendBufferSize = config.socketSendBufferBytes
            // 底层Socket接收缓存区大小
            val recvBufferSize = config.socketReceiveBufferBytes
            // Broker ID
            val brokerId = config.brokerId

            var processorBeginIndex = 0
            config.listeners.foreach { endpoint =>
                val listenerName = endpoint.listenerName
                val securityProtocol = endpoint.securityProtocol
                val processorEndIndex = processorBeginIndex + numProcessorThreads

                for (i <- processorBeginIndex until processorEndIndex)
                    // 创建Processor线程
                    processors(i) = newProcessor(i, connectionQuotas, listenerName, securityProtocol)
                // 创建Acceptor线程，一个Acceptor线程默认关联3个Processor线程，内部会启动Processor线程
                val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId,
                                            processors.slice(processorBeginIndex, processorEndIndex),
                                            connectionQuotas)
                    acceptors.put(endpoint, acceptor)

                // 启动Acceptor线程
                Utils.newThread(s"kafka-socket-acceptor-$listenerName-$securityProtocol-${endpoint.port}", acceptor, false).start()
                acceptor.awaitStartup()

                processorBeginIndex = processorEndIndex
            }
        }
    }
}

```

### 2.2 启动

Acceptor本质是一个Java Runnable任务，基于线程运行，它启动后会创建一个Java NIO中的组件**ServerSocketChannel**，绑定到EndPoint对应的IP和端口上，然后不断循环监听`OP_ACCEPT`事件，也就是客户端一旦请求建立连接，就会监听到：

```
// Acceptor.scala

private[kafka] class Acceptor(val endPoint: EndPoint,
                              val sendBufferSize: Int,
                              val recvBufferSize: Int,
                              brokerId: Int,
                              processors: Array[Processor],
                              connectionQuotas: ConnectionQuotas) extends AbstractServerThread(connectionQuotas) with KafkaMetricsGroup {

  // 创建一个ServerSocketChannel
  private val nioSelector = NSelector.open()
  val serverChannel = openServerSocket(endPoint.host, endPoint.port)

  // 创建并启动Processor线程
  this.synchronized {
    processors.foreach { processor =>
      Utils.newThread(s"kafka-network-thread-$brokerId-${endPoint.listenerName}-${endPoint.securityProtocol}-${processor.id}",
        processor, false).start()
    }
  }

  /**
   * 循环执行，监听客户端请求建立连接的事件
   */
  def run() {
    // 监听OP_ACCEPT事件
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    startupComplete()
    try {
      var currentProcessor = 0
      while (isRunning) {
        try {
          val ready = nioSelector.select(500)
          if (ready > 0) {
            val keys = nioSelector.selectedKeys()
            val iter = keys.iterator()
            while (iter.hasNext && isRunning) {
              try {
                val key = iter.next
                iter.remove()
                // 发生了OP_ACCEPT事件
                if (key.isAcceptable)
                  // 处理事件
                  accept(key, processors(currentProcessor))
                else
                  throw new IllegalStateException("Unrecognized key state for acceptor thread.")
                currentProcessor = (currentProcessor + 1) % processors.length
              } catch {
                case e: Throwable => error("Error while accepting connection", e)
              }
            }
          }
        }
        catch {
          case e: ControlThrowable => throw e
          case e: Throwable => error("Error occurred", e)
        }
      }
    } finally {
      debug("Closing server socket and selector.")
      swallowError(serverChannel.close())
      swallowError(nioSelector.close())
      shutdownComplete()
    }
  }
}

```

### 2.3 建立连接

可以看到，ServerSocketChannel上发生OP_ACCEPT事件后，Acceptor线程在accept方法中进行处理，最终会将建立的连接对应的SocketChannel对象转交给Processor线程处理：

```
// SocketServer.scala

def accept(key: SelectionKey, processor: Processor) {
  // 获取一个SocketChannel 
  val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
  val socketChannel = serverSocketChannel.accept()
  try {
    // 增加连接数
    connectionQuotas.inc(socketChannel.socket().getInetAddress)

    // 配置SocketChannel  
    socketChannel.configureBlocking(false)
    socketChannel.socket().setTcpNoDelay(true)
    socketChannel.socket().setKeepAlive(true)
    if (sendBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
      socketChannel.socket().setSendBufferSize(sendBufferSize)

    debug("Accepted connection from %s on %s and assigned it to processor %d, sendBufferSize [actual|requested]: [%d|%d] recvBufferSize [actual|requested]: [%d|%d]"
          .format(socketChannel.socket.getRemoteSocketAddress, socketChannel.socket.getLocalSocketAddress, processor.id,
                socketChannel.socket.getSendBufferSize, sendBufferSize,
                socketChannel.socket.getReceiveBufferSize, recvBufferSize))

    // 将SocketChanel交给Processor处理
    processor.accept(socketChannel)
  } catch {
    case e: TooManyConnectionsException =>
      info("Rejected connection from %s, address already has the configured maximum of %d connections.".format(e.ip, e.count))
      close(socketChannel)
  }
}

```

## 三、总结

本章，我对Kafka Server的Network Layer网络层的整体架构和Acceptor线程对请求的处理流程进行讲解。Acceptor线程其实只是监听指定端口的请求连接，然后完成连接的建立，并将连接对应的SocketChannel转交给Processor线程处理。
