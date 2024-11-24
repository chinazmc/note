- thrift: http://thrift.apache.org。由Facebook设计，目前已成为apache的一部分，是目前广泛应用的一个二进制协议，有广泛的语言绑定。thrift有丰富的数据类型：
    - 基础数据类型：bool, byte, i16, i32, i64, double, string
    - struct
    - 集合数据类型：list, set, collection thrift 支持 binary, compact binary 等模式，后者可以有更好的网络传输性能。thrift 使用tag来标记字段（这个做法事实上也不是thrift的原创，在tuxedo中就有类似的做法），理论上，thrift可以使用JSON、XML来传输数据（这在thrift的官方实现中确实包含了，不过有不少BUG，基本上不能真实使用）

在设计dapeng的数据协议之时，还没有flatbuffer，作者亲自经历过上述的除flatbuffers之外的7种模式，首先在文本协议和二进制协议之间犹豫甚久。作为一名UNIX的爱好者，受UNIX的影响很深，也享受过文本流协议来带的各种便利，也同时感觉到：“性能”在大部分的应用场景中其实是一个伪命题，摩尔定律存在的目的就是让软件大量的浪费内存和CPU，要不根本不会有今天的互联网。在多次的犹豫之后，还是放弃了文本协议：主要是考虑到SOA应用需要提供对编程语言友好的数据表达能力，我们需要如下的特性：

- 数据结构丰富
- 强类型支持。强类型还是弱类型，也是两大江湖门派，各有所爱。作者个人还是更倾向于强类型多一些。
- 性能。虽说，大部分应用场景都不会在序列号上有极致的要求，选择JSON可能也能够满足。但dapeng的目标还是希望应对大型互联网项目、上千个服务节点、每天数亿级的服务请求（考虑到一个PV可能到来的大量内部服务调用），还是希望选择高效的二进制协议。

dapeng的最早设计是在2013年，当时thrift和protobuf都已经成熟，最终我们选择thrift作为我们的底层数据协议（2者差异不大，选A或B都没有太大差别）。虽然是基于thrift的协议模型，但是dapeng还是做了很多的改进：

- 重新设计 Java POJO 对象的映射模型。官方生成的POJO完全不是面向开发者的，一大堆杂乱无章的序列化相关的代码。（protobuf也是同样的风格，阅读protobuf生成的代码对人来说完全是一种折磨），考虑到 SOA 的接口模型中，数据对象是API最为核心的部分，我们希望生成的代码是符合开发者阅读体验的，所以我们将 POJO 和 序列化代码进行了隔离。
- 更好的 optional 映射。 null是Java中一个尴尬的存在，NPE则是服务开发者中的一个小噩梦。我们希望使用 Optional[T] 来强语意的表达 null。
- 扩展数据类型支持。最为常见的，是对 Timestamp 和 BigDecimal 的支持。在商务类服务中，不支持这两个数据类型，是不完备的。

随着webserivces的谢幕，restful的崛起，JSON逐步成为互联网应用中霸主级的存在，使用thrift虽然可以在性能上碾压对手，但在开发便利性、跨语言支持上，却存在不足。如何完美的支持JSON、restful的应用，也是dapeng需要面对的问题。dapeng在设计之初，就将服务的元信息(metadata)作为重要的基础设施，基于metadata，我们可以在动态、高效中达成尽可能的平衡。基于metadata，我们可以完成从json <-> thrift binary的转换。这里的metadata相对与xml中的xsd，相当于java object的反射的Class对象。

# dapeng 应用层/会话层协议设计

## 请求/返回结构

```
i32  frameLength
byte[frameLength-4]  frameData
    byte stx = 0x02
    byte version = 0x01
    byte protocol
    i32  seqId
    byte[?]  header-thrift-struct
    byte[?]  body-thrift-struct
    byte  etx = 0x03
```

这里的i32/byte等都是采用非压缩的thrift二进制格式，其中 header-thrift-struct 由dapeng core定义，对应于 `com.github.dapeng.core.SoaHeader`, 包含了本次请求/回应的重要信息：包括serviceName, version, method，也包括了请求和返回的更多信息，这些信息是服务路由、服务限流、调用跟踪等诸多服务治理所需要的，详情在下面有介绍。

整个传输包采用带有包长度界定的方式，首先传输整个包的长度，然后是包的内容，包的内容中由以0x02开头，0x03结束，这样设计的优点：

- 可以高效的进行拆包、粘包的处理。在最外层可以不需要关系协议内部的具体内容，而简单的进行断包。
- 有简单的容错机制，如果包不是0x02开头，0x03结尾，则当前的会话出现了通信错乱，当前会话需要废弃。

version 字段目前值为 1, 这个字段是保留给后续进行版本升级的，在设计上，新版本的dapeng应该有能力兼容旧版本的协议。如果version升级，version后的内容将不需要沿用目前的版本，可以灵活扩展。

seqId 是当前请求/返回的序列号，在用一个会话（TCP连接）上，可以同时发送多个请求，而无需等待前面的请求完成，才能发送后面的请求，这一般称之为多路复用。采用多路复用技术后，可以大大提高客户端是多线程环境下的请求处理能力，不再需要每一个线程创建一个TCP连接，而是大家可以共享一个连接。这种模式在异步编程中也是非常有必要的。在多路复用时，seqId用来标注当前的返回所对应的是哪一个请求。

采用上述的包结构，也设定了dapeng不是面向流的，而是面向 request - response的。所以，目前来说，dapeng 不适合流处理，比如说一次请求，后续会逐步的收到多个数据返回的情况。

## SoaHeader

SoaHeader是dapeng服务调用的一个核心数据结构，用于传输在应用层（方法参数）之外的参数，传统的thrift调用是没有这一块的开销的，dapeng引入了这个新的结构体，带来了额外的开销，也带来了丰富的额外功能，这些构成了服务化治理的各种收益。

```
1 optional string serviceName
2 optional string methodName
3 optional string version
4 optional string callerMid
5 optional string callerIp
6 optional i32 callerPort
7 optional string sessionTid
8 optional string userIp
9 optional string callerTid
10 optional i32 timeout
11 optional string respCode
12 optional string respMessage
13 optional string calleeTid
14 optional string calleeIp
15 optional i64 operatorId
16 optional i32 calleePort
17 optional i64 userId
18 optional string calleeMid
19 optional i32 transactionId
20 optional i32 transactionSequence
21 optional i32 calleeTime1
22 optional i32 calleeTime2
23 map<string,string> cookie
```

- serviceName, version 对请求包来说是必须的，对返回包来说是不必须的。dapeng使用service:version来定位服务，同一个服务可以有多个版本(major.minor.patch):
    - major 大版本。当大版本号升级时，表示新服务版本在API上出现重大变更，并对当前版本不兼容。即3.y.z的服务不兼容2.y.z的服务。在同一个大版本下，服务需要提供向下兼容，即1.1.10应该兼容1.1.9，也应该兼容1.0.12。
    - minor 小版本。当API升级但同时兼容当前的版本时，可以增加minor版本号，如：
        - 新增新的方法
        - 在请求结构体中新增可选的字段
        - 在返回结构体中新增字段
        - 不改变现有结构体的字段tag值，及数据类型(未来版本可能会支持从byte,i16,i32到i16,i32,i64的升级) 开发人员可以使用dapeng的命令行工具来检查服务的版本兼容性，防止在破坏了兼容性的情况下，没有正确同步升级服务版本。
    - patch 一般的，在API不发生变化的情况下，但是对服务的实现进行了升级，或者进行了bug修复。这时，可以升级patch版本号。
- method 一个服务可以包含一组method，method是最小的可调用的服务单元。
- 客户端信息相关信息
    - callerMid: 调用者ModuleId。mid用来标识一个分布式应用中的一个参与者，包括：
        - web前端
        - 服务。
        - 定时任务
        - 脚本或工具
        - 消息消费者 通过为每一个参与者分配一个mid，可以在调用链跟踪系统中统计，监控这些参与者之间的调用情况，包括调用次数，服务时间，成功率等指标
    - callerIp：服务调用者IP
    - callerPort：服务调用者Port
    - userIp：用于跟踪最原始的userIp，这个值是在服务调用链传递的，可用于反馈最初发起服务的user所在的浏览器IP。
    - operatorId：内部操作员ID
    - userId：外部用户ID
- 服务端相关信息
    - calleeMid
    - calleeIp
    - calleePort
    - respCode
    - respMessage
    - calleeTime1
    - calleeTime2
- 线索相关信息：这一块在服务调用跟踪中详细介绍
    - sessionTid
    - callerTid
    - calleeTid
- 分布式事务：在目前的dapeng 2.0版本中暂不支持。
    - transactionId
    - transactionSequence
- cookie: 在整个服务调用链中是传输的。后续版本还会支持服务端可以返回新的cookie给到客户端。


### Dapeng-soa 服务的生命周期

2018-12-05 • Ever

随着搭载于dapeng框架之上的业务系统一个个上线并趋于稳定，不少开发同事们在抓虫子之余，也萌生了一些“不满”：用了dapeng好久了，但是它就像一个黑盒子，我们知之甚少，能否系统的介绍一下啊。

确实，dapeng框架提供了丰富的脚手架以及详尽的开发指南(详见[dapeng官网](https://dapeng-soa.github.io/)), 极易上手。但了解一下dapeng的内部机制，有利于开阔眼界，并在开发中充分利用各个特性，对于提高服务的质量也很有好处。 [!![[Pasted image 20230612141127.png]]

## 1. 概述

本文着眼于分析一个服务从启动到结束的整个过程。

> 后续会分专题陆续介绍dapeng框架的各个特性，敬请留意。

## 2. dapeng 容器目录结构

```
dapeng-container
├── apps                           # 1. business services
│   ├── order_service
│   └── stock_service
├── bin
│   ├── dapeng-bootstrap.jar                      # 2. dapeng bootstrap module
│   ├── lib                                       # 3. dapeng container jars(including dependency)
│   ├── shutdown.sh                               # 4. startup/shutdown shell
│   └── startup.sh
├── conf                                          # 5. configuration files for container
│   └── logback.xml
├── lib                                           # 6. core lib
│   ├── dapeng-core-2.1.1.jar
│   └── logback-*.jar
├── logs                                          # 7. directory where log files located
└── plugin                                        # 8. plugin directory
```

1. 业务服务，该目录由dapeng的`ApplicationClassLoader`加载， 可在一个dapeng容器中放多个服务，但是建议一个容器一个服务
2. dapeng bootstrap模块，容器的启动入口，负责生成其它的定制classLoader并启动容器。
3. dapeng container jar目录，容器的实现类及其三方依赖，通过`ContainerClassLoader`加载
4. 启动以及停止脚本
5. 容器的配置文件
6. 核心接口库以及日志，由`CoreClassLoader`负责加载，可用于服务端以及客户端
7. 日志目录
8. 插件目录，由`PluginClassLoader`加载

**注意**, 不同的目录由不同的classLoader加载。常见的错误是：apps路径下的服务有个对象(通过`ApplicationClassLoader`加载)，假设叫a，其类型为A，它是单例对象(通过`A.getInstance()`获得）。但是在bin路径下(也就是通过`ContainerClassLoader`加载), 通过`A.getInstance()`得到的a’, 就跟a是完全不同的对象了。

## 3. zk节点结构

```
/soa/
├── runtime/service/                                   # 1. runtime info
│   ├── com.today.soa.idgen.service.IDService/
│   │   ├── 192.168.10.130:9081:1.0.0:0000000181        # 2. node info
│   │   ├── 192.168.20.101:9081:1.0.0:0000000179        # 3. master node
│   │   └── 192.168.10.138:9081:1.0.0:0000000183
│   ├── com.today.api.order.service.OrderService2/ 
│   │   ├── 192.168.10.133:9099:1.0.0:0000000368        
│   │   ├── 192.168.10.126:9099:1.0.0:0000000372       
│   │   ├── 192.168.20.102:9099:1.0.0:0000000367      
│   │   └── 192.168.10.131:9099:1.0.0:0000000366        # master node
│   └── ...                                             # 4. other services
└── config/
    ├── services/                                       # 5. config info
    │   ├── com.today.soa.idgen.service.IDService
    │   └── com.today.api.order.service.OrderService2
    ├── freq/                                           # 6. rate limit rules
    ├── routes/                                         # 7. router rules
    └── cookies/                                        # 8. cookie rules
```

### 3.1 服务运行时信息 `/soa/runtime/service/`

服务运行时信息由服务在启动的时候注册到zk的`/soa/runtime/service/${serviceName}`路径上，其表现形式为挂靠在该路径下的**临时节点**，格式为：`ip:port:version:seq`, seq由zookeeper自动生成，且能保证在同目录下唯一并单调递增。

**其中，dapeng容器根据本容器中服务运行时信息的seq，判断是否为master主节点(seq最低者为master)**

> 一个服务对应一个临时节点。 我们可通过该路径得知目前服务集群中该服务的节点数量

### 3.2 服务配置信息 `/soa/config/`

服务配置信息存放了dapeng-soa所需要的一些配置项，可通过[命令行工具](https://github.com/dapeng-soa/dapeng-cli)或者[配置中心](https://github.com/dapeng-soa/dapeng-config-server)进行管理。

服务的配置信息当前有四种，分布在`/soa/config/`不同的子目录下。它们的结构都类似，都是把服务名字作为节点挂在配置子目录下，然后配置信息作为节点的内容。

#### 3.2.1 普通配置信息 `/soa/config/services/`

普通配置信息包括负载均衡策略、服务超时等信息。

它包含两个层次：

1. 全局性配置： 直接写进`/soa/config/services/`节点
    
    ```
    timeout/100ms;loadbalance/LeastActive;
    ```
    
2. 服务私有配置： 写到具体的服务节点上， 例如`/soa/config/services/com.today.api.order.service.OrderService2`
    
    ```
    timeout/100ms,createOrder:500ms,createOrderPayment:60000ms;loadbalance/LeastActive,createOrder:Random,createOrderPayment:RoundRobin;
    ```
    

#### 3.2.2 限流配置信息 `/soa/config/freq/`

dapeng 的流控做在服务端，所以**该节点只对服务端有效**。

限流信息直接写在具体的服务节点上，例如如下订单的限流配置写在`/soa/config/freq/com.today.api.order.service.OrderService2`上

```
[rule1]
match_app = listOrder # 针对具体方法限流
rule_type = callerIp # 对每个请求端IP
min_interval = 60,5  # 每分钟请求数不超过5次
mid_interval = 3600,100 # 每小时请求数不超过100次
max_interval = 86400,200 # 每天请求数不超过200次

[rule2]
match_app = * # 针对订单服务限流
rule_type = callerIp # 对每个请求端IP
min_interval = 60,600  # 每分钟请求数不超过600
mid_interval = 3600,10000 # 每小时请求数不超过1万
max_interval = 86400,80000 # 每天请求数不超过8万
```

> 详见[dapeng-soa限流文档](https://github.com/dapeng-soa/dapeng-soa/wiki/DapengFreqControl)

#### 3.2.3 路由配置信息 `/soa/config/routes/`

服务路由信息也是直接写在具体的服务节点上，例如下面订单的路由配置写在`/soa/config/routes/com.today.api.order.service.OrderService2`上

```
method match r"create.*" => ip"192.168.10.0/24"
cookie_storeId match %"10n+1..6" => ip"192.168.20.128"
```

> 详见[dapeng-soa路由文档](https://github.com/dapeng-soa/dapeng-soa/wiki/Dapeng-Service-Route%EF%BC%88%E6%9C%8D%E5%8A%A1%E8%B7%AF%E7%94%B1%E6%96%B9%E6%A1%88%EF%BC%89)

#### 3.2.4 cookie 规则信息 `/soa/config/cookies/`

cookie规则信息也是直接写在具体的服务节点上，例如针对来自dapengCli的手工对订单接口的调用，我们为这些调用打开TRACE功能，我们把规则配置到`/soa/config/cookies/com.today.api.order.service.OrderService2`:

```
callerIp match ip"192.168.20.200" => c"thread-log-level#TRACE"
```

## 4. 容器的生命周期

容器提供了一个LifeCycleAware接口以及若干事件，在事件发生的时候会触发相应的业务逻辑。

业务可通过实现该接口， 做一些初始化以及清理的动作。

> 例如某服务的业务，只在主节点启动一个工作线程。那么它就可以监听`MASTER_CHANGE`事件。当主节点发生变更的时候，就启动或者停止工作线程。

```java
public enum LifeCycleEventEnum {
    /**
        * dapeng 容器启动
        */
    START,
    PAUSE,
    MASTER_CHANGE,
    CONFIG_CHANGE,
    STOP
}
```

```java
/**
 * 提供给业务的lifecycle接口，四种状态
 *
 * @author hui
 * @date 2018/7/26 11:21
 */
public interface LifeCycleAware {

    /**
     * 容器启动时回调方法
     */
    void onStart(LifeCycleEvent event);

    /**
     * 容器暂停时回调方法
     */
    default void onPause(LifeCycleEvent event) {
    }

    /**
     * 容器内某服务master状态改变时回调方法
     * 业务实现方可自行判断具体的服务是否是master, 从而执行相应的逻辑
     */
    default void onMasterChange(LifeCycleEvent event) {
    }

    /**
     * 容器关闭
     */
    void onStop(LifeCycleEvent event);

    /**
     * 配置变化
     */
    default void onConfigChange(LifeCycleEvent event) {
    }
}
```

## 5. 容器的启动过程

容器启动需要协调各个插件的顺序，避免在服务还没准备好的情况下，客户端请求就涌进来。

通过脚本`startup.sh`启动容器: `java -server $JAVA_OPTS -cp ./dapeng-bootstrap.jar com.github.dapeng.bootstrap.Bootstrap` [![dapeng-soa bootstrap](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/dapeng-bootstrap.png)](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/dapeng-bootstrap.png)

- Bootstrap创建三个`classLoader`, 分别是`CoreClassLoader`(负责加载lib目录的类)、`ContainerClassLoader`(负责加载bin/lib目录的类)以及`ApplicationClassLoader`(负责加载apps目录下的类)。
- Bootstrap通过`ContainerClassLoader`加载`ContainerFactory`并调用其`getContainer`方法， 获得`DapengContainer`实例。
- `DapengContainer`创建五个Plugin，并依次调用其`start`方法:
    - 启动`NettyPlugin`， 打开服务监听端口(例如9090)
    - 启动`ZkPlugin`, 跟注册中心`zookeeper`建立连接。
    - 启动`SpringPlugin`，
        - 通过`ApplicationClassLoader`加载服务(一个服务表现为一个`Application对`象)
        - 对每个服务(假设为OrderService)，通过`ZkPlugin`把服务信息注册到`/soa/runtime/service/com.today.api.order.service.OrderService2`目录里，并启动一个对该路径的zk `watcher`(主要用于跟踪服务集群中master节点的变化, 当发生master变换时，需触发`MASTER_CHANGE`事件。 一旦服务完成注册，嗷嗷待哺的客户端就会如潮水般涌进该服务节点。
    - 启动`SchedulePlugin`, 定时任务就绪
    - 启动`JmxPlugin`, `Jmx` 端口就绪
    - 如果是开发模式(默认), 那么启动`ApiDocPlugin`, 内置的文档站点可访问。该插件在生产环境下不会启动
- 触发容器`START`事件
- 加载服务端`Filter`(详情下回分解)。
- 最后，注册Jvm进程的`ShutdownHook`。

至此，容器启动完毕， 服务生命周期开始。

## 6. 容器的优雅关闭过程

容器的关闭过程，同样需要协调插件的关闭顺序，确保进来的请求尽量处理完毕后再关闭容器，避免对业务产生影响。

为此， `dapeng` 容器会维护一个请求计数器`requestCounter`，计数值是当前容器内尚未处理完的请求数目。

- 启动脚本`startup.sh`会监听`kill`信号。收到`kill`信号后，脚本转发该信号到容器Jvm进程。
- 容器的`ShutdownHook`给触发,依次执行:
    - `ZkPlugin.stop`方法，断开跟`zookeeper`的连接，从而把本节点服务信息从`/soa/runtime/services/${serviceName}`上摘除，进而新的请求就不会再路由到本节点。
    - 休眠若干次，直到`requestCounter`数目为0或者超时。
    - 关闭其它插件( `Netty`有自己的优雅关闭机制，能确保`outbound`队列的消息能全部发送出去)

至此，容器关闭，服务结束了其使命。


# Reference
https://dapeng-soa.github.io/dapeng/2018/12/05/lifecycle-of-dapeng.html
