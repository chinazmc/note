#grpc 

前文已经分析了Resolver解析器有什么用，如何实现一个解析器，如何注册一个解析器；
那么，本篇文章主要是想分享一下，[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)-go框架是如何来使用解释器的，看看人家是怎么来用的;

# 1、在什么地方可以指明使用什么类型的解析器呢？

随便找一个客户端[测试用例](https://so.csdn.net/so/search?q=%E6%B5%8B%E8%AF%95%E7%94%A8%E4%BE%8B&spm=1001.2101.3001.7020)，找到grpc.Dial语句：
```go
conn, err := grpc.Dial(target, grpc.WithInsecure(), grpc.WithBlock())
```
其中，target中指定本次链接使用哪种解析器。

# 2、grpc框架根据用户在grpc.Dial中设置的target来获取解析器构建器

然后依次点击，最终进入DialContext函数(grpc-go/clientconn.go):

```go
1．resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)

2．if resolverBuilder == nil {
3．  //---省略非核心代码
4．   cc.parsedTarget = resolver.Target{
5．       Scheme:   resolver.GetDefaultScheme(),
6．       Endpoint: target,
7．   }
8．  resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)

9．   if resolverBuilder == nil {
10．      return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
11．   }
12．}
//---省略非核心代码
13．rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)
14．if err != nil {
15．   return nil, fmt.Errorf("failed to build resolver: %v", err)
16．}
```
主要代码说明
-   第1行：通过调用cc的getResolver方法，根据Scheme来获取解析构建器resolverBuilder
-   第2-11行：如果没有对应的解析构建器的话，就使用默认的解析器passthrough
-   第13行：创建解析器包装结构体

# 3、grpc框架对解析器进行了二次包装，形成解析器包装类ccResolverWrapper

## 3.1、通过newCCResolverWrapper函数来创建一个解析器包装类ccResolverWrapper

进入grpc-go/resolver_conn_wrapper.go文件的newCCResolverWrapper方法里：

```go
1．func newCCResolverWrapper(cc *ClientConn, rb resolver.Builder) (*ccResolverWrapper, error) {
2．  ccr := &ccResolverWrapper{
3．      cc:   cc,
4．      done: grpcsync.NewEvent(),
5．   }
  //---省略非核心代码
6．   rbo := resolver.BuildOptions{
7．      DisableServiceConfig: cc.dopts.disableServiceConfig,
8．      DialCreds:            credsClone,
9．      CredsBundle:          cc.dopts.copts.CredsBundle,
10．      Dialer:               cc.dopts.copts.Dialer,
11．   }

12．   var err error
13．   ccr.resolverMu.Lock()
14．   defer ccr.resolverMu.Unlock()
15．   ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, rbo)
16．   if err != nil {
17．      return nil, err
18．   }

19．   return ccr, nil
20．}
```

主要代码说明:
-   第2-5行: 构建ccResolverWrapper结构体
-   第6-11行：构建resolver.BuildOptions结构体，下面构建解析器时使用到的
-   第15行: 通过解析构建器Build去创建解析器

## 3.2、ccResolverWrapper解析器包装类都定义了什么属性

在grpc[框架](https://so.csdn.net/so/search?q=%E6%A1%86%E6%9E%B6&spm=1001.2101.3001.7020)中是对解析器Resolver进行了二次包装，形成一个包装类ccResolverWrapper(grpc-go/resolver_conn_wrapper.go)

```go
1．// ccResolverWrapper is a wrapper on top of cc for resolvers.
2．// It implements resolver.ClientConn interface.
3．type ccResolverWrapper struct {
4．   cc         *ClientConn
5．   resolverMu sync.Mutex
6．   resolver   resolver.Resolver
7．   done       *grpcsync.Event
8．   curState   resolver.State

9．   pollingMu sync.Mutex
10．   polling   chan struct{}
11．}
```

主要代码说明:
-   第4行：客户端连接器，表明解析器属于哪个ClientConn
-   第6行: 解析器，
-   第7行：事件(事件主要用来干什么，可以参考相关章节)
-   第8行：解析器状态；参考值如下：

```json
{
	"Addresses": [{
		"Addr": "localhost:50051",
		"ServerName": "",
		"Attributes": null,
		"Type": 0,
		"Metadata": null
	}, {
		"Addr": "localhost:50052",
		"ServerName": "",
		"Attributes": null,
		"Type": 0,
		"Metadata": null
	}],
	"ServiceConfig": null,
	"Attributes": null
}
```

## 3.3、grpc框架通过解析器的Build方法来创建一个解析器，那么Build方法，都有哪些不同形式的实现呢？

![解析器中的build有哪些实现](https://img-blog.csdnimg.cn/20210512051420164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1ODI5MjI=,size_16,color_FFFFFF,t_70#pic_center)

解析器的核心原理，其实主要是分析解析构建器中Build方法，都做了哪些事情；

