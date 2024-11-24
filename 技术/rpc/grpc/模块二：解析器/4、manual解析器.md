#grpc 

同样，本篇文章主要是分析一下manual解析器中的Build方法，做了什么事情？

# 1、manual类型的解析器，如何获取后端grpc服务器的地址列表？

有多种技术实现思路：

-   方式一：可以将grpc服务器地址列表，存储到远程服务器，如consol, etcd, zk, 文件服务器等等；然后通过接口调用，从远程服务器里获取grpc服务器地址列表，然后，在存储到  
    resolver.State(生产环境，建议使用此种方式)
-   方式二：如果仅仅是测试环境的话，可以直接手动维护，自己将grpc服务器地址列表，初始到resolver.State里；

参考一下，[grpc](https://so.csdn.net/so/search?q=grpc&spm=1001.2101.3001.7020)-go框架自带的测试用例，如下：

直接进入grpc-go/examples/features/health/client/main.go文件中：

```go
1．func main() {
2．   flag.Parse()

3．   r, cleanup := manual.GenerateAndRegisterManualResolver()
4．   defer cleanup()
5．   r.InitialState(resolver.State{
6．      Addresses: []resolver.Address{
7．         {Addr: "localhost:50051"},
8．         //{Addr: "localhost:50052"},
9．      },
10．   })
11. address := fmt.Sprintf("%s:///unused", r.Scheme())
12.	conn, err := grpc.Dial(address, options...)
    //---省略掉非核心代码
```

主要流程说明：

-   第3行：注册解析器，至于如何注册的，可以参考以前的文章
-   第5行：直接将后端服务器地址列表封装到resolver.State里，调用InitialState；
-   第11行：r.scheme是随机生成的；unused，说明manual解析器在获取后端grpc服务器地址时，并没有使用unused进行解析。

那么，调用InitialState方法，有什么用呢？

点击进入其内部：

```go
func (r *Resolver) InitialState(s resolver.State) {
     r.bootstrapState = &s
}
```

将resolver.State赋值给r.bootstrapState，这么做就是为了用户在客户端不需要自己手动调用，

而是grpc框架在build方法内部，帮我们调用UpdateState，从而触发平衡器的操作流程，

即最终向grpc服务器端发起rpc链接;  
   
 
# 2、manual类型的解析器中的Build方法，做了哪些事情呢？

grpc-go/resolver/manual/manual.go文件中的Build方法的第4-6行：

```go
1．// Build returns itself for Resolver, because it's both a builder and a resolver.
2．func (r *Resolver) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
3．   r.CC = cc

4．   if r.bootstrapState != nil {
5．      r.UpdateState(*r.bootstrapState)
6．   }
7．   
8．   return r, nil
9．}
```

主要代码说明：

-   第4行：r.bootstrapState，就是上小节中提到的resolver.State值，非nil
-   第5行：调用UpdateState，触发平衡器流程，最终触发grpc客户端向grpc服务器端发起rpc链接；具体的链接过程，在前文中已经介绍过了，不再赘述。

# 3、总结
-   manual类型的解析器的scheme，是随机生成的字符串；target也可以随机设置；因为，manual类型的解析，在获取grpc后端服务器地址时，并不是通过解析scheme,target来获取后端grpc服务器地址的;
-   在Build方法内部，调用UpdateState方法，触发平衡器流程，触发rpc连接的建立。
-   在grpc客户端需要手动显示的调用InitialState方法，至于，如何获取grpc服务器地址列表，需要考虑
