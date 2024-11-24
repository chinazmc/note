#flink 

### [](http://www.54tianzhisheng.cn/2018/10/29/flink-sink/#%E5%89%8D%E8%A8%80)前言

再上一篇文章中 [《从0到1学习Flink》—— Data Source 介绍](http://www.54tianzhisheng.cn/2018/10/28/flink-sources/) 讲解了 Flink Data Source ，那么这里就来讲讲 Flink Data Sink 吧。

首先 Sink 的意思是：

![](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/1atUyo.jpg)

大概可以猜到了吧！Data sink 有点把数据存储下来（落库）的意思。

![](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/raHGlh.jpg)

如上图，Source 就是数据的来源，中间的 Compute 其实就是 Flink 干的事情，可以做一系列的操作，操作完后就把计算后的数据结果 Sink 到某个地方。（可以是 MySQL、ElasticSearch、Kafka、Cassandra 等）。这里我说下自己目前做告警这块就是把 Compute 计算后的结果 Sink 直接告警出来了（发送告警消息到钉钉群、邮件、短信等），这个 sink 的意思也不一定非得说成要把数据存储到某个地方去。其实官网用的 Connector 来形容要去的地方更合适，这个 Connector 可以有 MySQL、ElasticSearch、Kafka、Cassandra RabbitMQ 等。

### [](http://www.54tianzhisheng.cn/2018/10/29/flink-sink/#Flink-Data-Sink)Flink Data Sink

前面文章 [《从0到1学习Flink》—— Data Source 介绍](http://www.54tianzhisheng.cn/2018/10/28/flink-sources/) 介绍了 Flink Data Source 有哪些，这里也看看 Flink Data Sink 支持的有哪些。

![](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/siWsAK.jpg)

看下源码有哪些呢？

![](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/F38tbg.jpg)

可以看到有 Kafka、ElasticSearch、Socket、RabbitMQ、JDBC、Cassandra POJO、File、Print 等 Sink 的方式。

### [](http://www.54tianzhisheng.cn/2018/10/29/flink-sink/#SinkFunction)SinkFunction

![](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/g5mhoE.jpg)

从上图可以看到 SinkFunction 接口有 invoke 方法，它有一个 RichSinkFunction 抽象类。

上面的那些自带的 Sink 可以看到都是继承了 RichSinkFunction 抽象类，实现了其中的方法，那么我们要是自己定义自己的 Sink 的话其实也是要按照这个套路来做的。

这里就拿个较为简单的 PrintSinkFunction 源码来讲下：
```java
@PublicEvolving  
public class PrintSinkFunction<IN> extends RichSinkFunction<IN> {  
 private static final long serialVersionUID = 1L;  
  
 private static final boolean STD_OUT = false;  
 private static final boolean STD_ERR = true;  
  
 private boolean target;  
 private transient PrintStream stream;  
 private transient String prefix;  
  
 /**  
 * Instantiates a print sink function that prints to standard out.  
 */  
 public PrintSinkFunction() {}  
  
 /**  
 * Instantiates a print sink function that prints to standard out.  
 *  
 * @param stdErr True, if the format should print to standard error instead of standard out.  
 */  
 public PrintSinkFunction(boolean stdErr) {  
 target = stdErr;  
 }  
  
 public void setTargetToStandardOut() {  
 target = STD_OUT;  
 }  
  
 public void setTargetToStandardErr() {  
 target = STD_ERR;  
 }  
  
 @Override  
 public void open(Configuration parameters) throws Exception {  
 super.open(parameters);  
 StreamingRuntimeContext context = (StreamingRuntimeContext) getRuntimeContext();  
 // get the target stream  
 stream = target == STD_OUT ? System.out : System.err;  
  
 // set the prefix if we have a >1 parallelism  
 prefix = (context.getNumberOfParallelSubtasks() > 1) ?  
 ((context.getIndexOfThisSubtask() + 1) + "> ") : null;  
 }  
  
 @Override  
 public void invoke(IN record) {  
 if (prefix != null) {  
 stream.println(prefix + record.toString());  
 }  
 else {  
 stream.println(record.toString());  
 }  
 }  
  
 @Override  
 public void close() {  
 this.stream = null;  
 this.prefix = null;  
 }  
  
 @Override  
 public String toString() {  
 return "Print to " + (target == STD_OUT ? "System.out" : "System.err");  
 }  
}  
```

可以看到它就是实现了 RichSinkFunction 抽象类，然后实现了 invoke 方法，这里 invoke 方法就是把记录打印出来了就是，没做其他的额外操作。

### [](http://www.54tianzhisheng.cn/2018/10/29/flink-sink/#%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%EF%BC%9F)如何使用？

1  

SingleOutputStreamOperator.addSink(new PrintSinkFunction<>();  

这样就可以了，如果是其他的 Sink Function 的话需要换成对应的。

使用这个 Function 其效果就是打印从 Source 过来的数据，和直接 Source.print() 效果一样。

![](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/wK45iZ.jpg)

下篇文章我们将讲解下如何自定义自己的 Sink Function，并使用一个 demo 来教大家，让大家知道这个套路，且能够在自己工作中自定义自己需要的 Sink Function，来完成自己的工作需求。

### [](http://www.54tianzhisheng.cn/2018/10/29/flink-sink/#%E6%9C%80%E5%90%8E)最后

本文主要讲了下 Flink 的 Data Sink，并介绍了常见的 Data Sink，也看了下源码的 SinkFunction，介绍了一个简单的 Function 使用, 告诉了大家自定义 Sink Function 的套路，下篇文章带大家写个。