#grpc 

其实主要是分析解析构建器中Build方法，都做了哪些事情；

因此，本文主要是分析一下，gRPC-go框架自带的默认解析器passthrough中Build方法都做了哪些事情；

# 1、什么情况下使用该解析器呢？
第一种情况：没有指定Scheme时，默认使用passthrough解析器;如下所示：
```go
conn, err := grpc.Dial(“localhost:50051”, grpc.WithInsecure(), grpc.WithBlock())
```
第二情况：显示指明使用passthrough解析器
```go
conn, err := grpc.Dial(“passthrough:///localhost:50051”, grpc.WithInsecure(), grpc.WithBlock())
```
 
# 2、Build方法做了哪些事情？
好，直接进入grpc-go/internal/resolver/passthrough/passthrough.go文件的Build方法内部：
```go
1．func (*passthroughBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
2．   r := &passthroughResolver{
3．      target: target,
4．      cc:     cc,
5．   }

6．   r.start()

7．   return r, nil
8．}
```
主要代码说明：

第2-5行：创建passthroughResolver解析器；
a)target的参考值，如：{“Scheme”:“passthrough”,“Authority”:"",“Endpoint”:“localhost:50051”}
b)cc, 类型是resolver.ClientConn
第6行：这一行很重要，调用start方法；调用此方法的最终目的，就是为了触发平衡器流程，最终建立rpc请求
进入start方法内部看看：
```go
func (r *passthroughResolver) start() {
     r.cc.UpdateState(resolver.State{Addresses: []resolver.Address{{Addr: r.target.Endpoint}}})
}
```
更新ClientConn的状态，经过层层调用，最终向grpc服务器端发起rpc连接。

具体就不详细看了，在建立rpc链接相关章节中已经介绍过。此处就不在占用篇幅了。
 

# 3、总结
passthrough解析器，是grpc框架默认使用的解析器；
passthrough解析器，默认用户传入的target.Endpoint就是后端grpc服务器端的地址；
因此，在Build方法内部，只是仅仅构建了passthroughResolver结构体，以及调用了resolver.ClientConn中的UpdateState方法。
