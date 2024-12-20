---
sr-due: 2023-01-07
sr-interval: 4
sr-ease: 270
---


#grpc #note 


本文主要介绍了 gRPC 客户端负载均衡策略及其配置与使用，包括 Name Resolver、ServiceConfig 等。

> gRPC 系列相关代码见 [Github](https://github.com/lixd/grpc-go-example)

## [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#1-%E6%A6%82%E8%BF%B0)1. 概述

gRPC 负载均衡包括客户端负载均衡和服务端负载均衡两种方向。本文主要介绍的是客户端负载均衡。

gRPC 的客户端负载均衡主要分为两个部分：

-   1）Name Resolver
-   2）Load Balancing Policy

### [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#1-nameresolver)1. NameResolver

具体可以参考[官方文档-Name Resolver](https://github.com/grpc/grpc/blob/master/doc/naming.md)或者[gRPC系列教程(十一)—NameResolver 实战及原理分析](https://www.lixueduan.com/post/grpc/11-name-resolver/)

gRPC 中的默认 name-system 是DNS，同时在客户端以插件形式提供了自定义 name-system 的机制。

gRPC NameResolver 会根据 name-system 选择对应的解析器，用以解析用户提供的服务器名，最后返回具体地址列表（IP+端口号）。

例如：默认使用 DNS name-system，我们只需要提供服务器的域名即端口号，NameResolver 就会使用 DNS 解析出域名对应的IP列表并返回。

在本例中我们会自定义一个 NameResolver。

### [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#12-load-balancing-policy)1.2 Load Balancing Policy

具体可以参考[官方文档-Load Balancing Policy](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)

常见的 gRPC 库都内置了几个负载均衡算法，比如 [gRPC-Go](https://github.com/grpc/grpc-go/tree/master/examples/features/load_balancing#pick_first) 中内置了`pick_first`和`round_robin`两种算法。

-   pick_first：尝试连接到第一个地址，如果连接成功，则将其用于所有RPC，如果连接失败，则尝试下一个地址（并继续这样做，直到一个连接成功）。
-   round_robin：连接到它看到的所有地址，并依次向每个后端发送一个RPC。例如，第一个RPC将发送到backend-1，第二个RPC将发送到backend-2，第三个RPC将再次发送到backend-1。

本例中我们会分别测试两种负载均衡策略的效果。

## [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#2-demo)2. Demo

### [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#21-server)2.1 Server

主要通过一个 for 循环，在 50051 和 50052 这两个端口上启动了服务。

### [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#22-client)2.2 Client

可以看到，在客户端是分别使用不同的负载均衡策略建立了两个连接，首先是默认的策略 pick_first，然后则是 round_robin，核心代码为：

```go
grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`)
```

同时由于是本地测试，不方便使用内置的 dns Resolver 所以自定义了一个 Name Resolver，相关代码如下：

```go
// Following is an example name resolver implementation. Read the name
// resolution example to learn more about it.

type exampleResolverBuilder struct{}

func (*exampleResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	r := &exampleResolver{
		target: target,
		cc:     cc,
		addrsStore: map[string][]string{
			exampleServiceName: addrs,
		},
	}
	r.start()
	return r, nil
}
func (*exampleResolverBuilder) Scheme() string { return exampleScheme }

type exampleResolver struct {
	target     resolver.Target
	cc         resolver.ClientConn
	addrsStore map[string][]string
}

func (r *exampleResolver) start() {
	addrStrs := r.addrsStore[r.target.Endpoint]
	addrs := make([]resolver.Address, len(addrStrs))
	for i, s := range addrStrs {
		addrs[i] = resolver.Address{Addr: s}
	}
	r.cc.UpdateState(resolver.State{Addresses: addrs})
}
func (*exampleResolver) ResolveNow(o resolver.ResolveNowOptions) {}
func (*exampleResolver) Close()                                  {}
```

### [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#3-test)3. Test

分别运行服务端和客户端查看结果

```sh
lixd@17x:~/17x/projects/grpc-go-example/features/load_balancing/server$ go run main.go 
2021/05/23 09:47:59 serving on 0.0.0.0:50052
2021/05/23 09:47:59 serving on 0.0.0.0:50051
```

```sh
lixd@17x:~/17x/projects/grpc-go-example/features/load_balancing/client$ go run main.go 
--- calling helloworld.Greeter/SayHello with pick_first ---
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)

--- calling helloworld.Greeter/SayHello with round_robin ---
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
```

可以看到 pick_first 负载均衡策略时一直请求第一个服务 50051，round_robin 时则会交替请求，这也和负载均衡策略相符合。

## [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#3-%E5%B0%8F%E7%BB%93)3. 小结

本文介绍的 负载均衡属于 **客户端负载均衡**，需要在客户端做较大改动，因为 gRPC-go 中已经实现了对应的代码，所以使用起来还是很简单的。

gRPC 内置负载均衡实现：

-   1）根据提供了服务名，使用对应 name resolver 解析获取到具体的 ip+端口号 列表
-   2）根据具体服务列表，分别建立连接
    -   gRPC 内部也维护了一个连接池
-   3）根据负载均衡策略选取一个连接进行 rpc 请求

比如之前的例子，服务名为`example:///lb.example.grpc.lixueduan.com`，使用自定义的 name resolver 解析出来具体的服务列表为`localhost:50051,localhost:50052`.

然后调用 dial 建立连接时会分别与这两个服务建立连接。最后根据负载均衡策略选择一个连接来发起 rpc 请求。所以 pick_first会一直请求50051服务，而 round_robin 会交替请求 50051和50052。

## [](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/#4-%E5%8F%82%E8%80%83)4. 参考

`https://github.com/grpc/grpc/blob/master/doc/naming.md`

`https://github.com/grpc/grpc/blob/master/doc/load-balancing.md`

Updated on 2021-05-22

 [gRPC](https://www.lixueduan.com/tags/grpc/)

Back | [Home](https://www.lixueduan.com/)

[gRPC(Go)教程(十一)---NameResolver 实战及原理分析](https://www.lixueduan.com/posts/grpc/11-name-resolver/ "gRPC(Go)教程(十一)---NameResolver 实战及原理分析")[gRPC(Go)教程(十三)--- Kubernetes 环境下的 gRPC 负载均衡](https://www.lixueduan.com/posts/grpc/13-loadbalance-on-k8s/ "gRPC(Go)教程(十三)--- Kubernetes 环境下的 gRPC 负载均衡")

Powered by [Hugo](https://gohugo.io/ "Hugo 0.100.2") | Theme -  [LoveIt](https://github.com/dillonzq/LoveIt "LoveIt 0.2.11")

2018 - 2022 [意琦行](https://github.com/lixd) | [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) | [渝ICP备20005680号-1](https://beian.miit.gov.cn/)

[](https://www.lixueduan.com/posts/grpc/12-client-side-loadbalance/# "Back to Top")