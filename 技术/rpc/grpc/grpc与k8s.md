# Kubernetes 应用改造（一）：Headless Service

## 使用 gRPC+Headless-Service 构建后端服务

## 1、 Kubernetes 从入门到放弃

网上有很多关于 `Kubernetes` 的高质量文章，以及阿里的公开课，可以帮助读者构建 `Kubernetes` 的生态认知。`Kubernetes` 的几个核心概念中，只要理解了分层的概念，穿透各个组件都是围绕此来设计和实现的。下层为上层提供服务，上层不要知道下层的具体实现细节，只需使用下层提供的服务。而层与层之间联系的桥梁就是接口 (`Interface`)

## 2、 开发需要掌握的 Kubernetes

就我个人而言，作为一名合格的云开发者，我认为需要理清楚下面的知识概念：

1. `POD` –> `Deployment`–> `Service` —> `Ingress` 服务应用如何对照入座，在 K8S 中部署的应用无法脱离这几种应用形态 (去掉 POD)
2. `Statefulset` 和 `Damonset` 的应用场景
3. `JOB/crontabJOB` 的应用场景
4. `PV` 和 `PVC` 的实践
5. `CONFIG-MAP` 与 `Secret` 如何与应用搭配（或者说更安全、更灵活的搭配）
6. 如何实现 `CONFIG-MAP` 的热更新
7. `POD` 之间如何共享数据，`POD` 生产的日志如何持久化
8. 容器的健康检查（`Healthy Check`），就绪探针与保活探针的使用
9. `POD` 之间的互相访问和服务发现方式
10. `Kubernetes` 上的负载均衡如何做
11. `Kubernetes` 中的认证与授权、`RBAC`
12. `RBAC` 的应用之，如何使用 `ServiceAccount`+`Token`+ 证书来调用 `ApiServer` 的相关操作（当然了，必须对 SA 授权）
13. 如何通过 `Kubernetes` 的 `API` 来获取 `POD` 列表的实时变化（实现 `gRPC`+`Kubernetes` 的自定义负载均衡的算法）
14. 如何理解 `Kubernetes` 的资源限制，如何应用 `HPA（Horizontal Pod Autoscaler）`

## 3、 进阶话题

- `Kubernetes` 路由的实现原理，`IPVS`
- `Istio`，感觉这个开源的作品在未来若干年会成为主流的管理应用
- `Kube-CRD` 开发，满足各种现网的特殊需求，原生提供的 `Crontroller` 可能无法满足，需要定制化适合业务的 `Controller`

## 4、 Service 的本质

理解 `Service` 从下面几点出发：

1. 发现后端 `Pod` 服务
2. 为一组具有相同功能的容器应用提供一个统一的入口地址
3. 将请求进行负载分发到后端的各个 `Container` 应用上的控制器

## 5、 Service 的常用类型

1. `ClusterIP`
    - `Service`：通过为 `Kubernetes` 的 `Service` 分配一个集群内部可访问的固定虚拟 IP（`Cluster IP`），实现集群内的访问
    - `Headless Service`：该服务不会分配 `Cluster IP`，也不通过 `Kube-proxy` 做反向代理和负载均衡。而是通过 `DNS` 提供稳定的网络 ID 来访问，`DNS` 会将 `Headless Service` 的后端直接解析为 `Pod IP` 列表
2. `NodePort`：除了使用 `Cluster IP` 之外，还通过将 `Service` 的 `port` 映射到集群内每个节点的相同一个端口，实现通过 `nodeIP:nodePort` 从集群外访问服务![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/K8S-HEADLESS-SVC-1.png)
    
3. `LoadBalancer`：和 `NodePort` 类似，不过除了使用一个 `Cluster IP` 和 `NodePort` 之外，还会向所使用的公有云申请一个负载均衡器（负载均衡器后端映射到各节点的 `NodePort`），实现从集群外通过 LB 访问服务，一般需要依赖公有云的实现（比如 `TKE` 上一般为公网 `IP` 或是一个公网域名）![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/K8S-HEADLESS-SVC-2.png)
    
4. `Ingress`: 类似于反向代理，可以配置按特定条件将流量分发到不同的 `Service` 上![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/K8S-HEADLESS-SVC-3.png)

## 6、 Loadbalancer In Kubernetes

引用一下 GKE 的图：  

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/K8S-HEADLESS-SVC-4.png)

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/K8S-HEADLESS-SVC-5.png)

一般应用中都使用 `Load Balancer` 的方式，这个实现起来，也并不复杂，后续会新开一篇文章来讲解下如何实现。

## 7 Kubernetes+gRPC 的负载均衡的问题

使用 `Kubernetes` 默认的 `Service` 做负载均衡的情况下，`gRPC` 的 `RPC` 通信是基于 `HTTP/2` 标准实现的，`HTTP/2` 的一大特性就是不需要像 `HTTP/1.1` 一样，每次发出请求都要新建立一个连接，而会复用原有的连接。所以这将导致 `Kube-proxy` 只有在连接建立时才会做负载均衡，而在这之后的每一次 `RPC` 请求都会利用原本的连接，那么实际上后续的每一次的 `RPC` 请求都跑到了同一个地方。看下面的这篇文章 [gRPC Load Balancing on Kubernetes without Tears](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)

既然在 `HTTP/2` 中存在了这样一个特性，那么 `gRPC` 如何与 `Kubernetes` 进行整合呢？我们从 `Kubernetes Service` 的几个类型中来寻找解决 `LB` 问题的方案

## 8、 再看 Headless Service

如前所说，`Kubernetes` 提供了一个基础的 `Pod` 控制方式给用户，参加官方文档 [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)

`Headless Service` 需要将 `spec.clusterIP` 设置成 `None`，因为没有 `ClusterIP`，`Kube-proxy` 并不处理此类服务，因为没有 `Load balancing` 或 `proxy` 代理设置，在访问服务的时候回返回后端的全部的 `Pods IP` 地址，主要用于开发者自己根据 pods 进行负载均衡器的开发（设置了 `selector`）

## 9 测试 Headless Service

这里用到 `gRPC Service` 的 `yaml` 文件如下（注意 `clusterIP` 字段必须设置为 `None`）：

```yml
apiVersion: v1
kind: Service
metadata:
  name: serv-ca
  namespace: nspace
spec:
  selector:
    name: test
  clusterIP: None # This is important!
  ports:
    - name: tcp
      port: 8088
      targetPort: 8088
```

下面尝试执行 `nslookup` 命令来获取服务 `serv-ca` 对应的所有 `Pod` 的列表：

```
[root@docker-79b475496-4nbl8 /]# nslookup -q=srv bf-serv-ca
Server:         1.2.3.4
Address:        1.2.3.4#53

serv-ca.nspace.svc.cluster.local service = 0 25 8088 172-16-0-20.serv-ca.nspace.svc.cluster.local.
serv-ca.nspace.svc.cluster.local service = 0 25 8088 172-16-0-21.serv-ca.nspace.svc.cluster.local.
serv-ca.nspace.svc.cluster.local service = 0 25 8088 172-16-1-17.serv-ca.nspace.svc.cluster.local.
serv-ca.nspace.svc.cluster.local service = 0 25 8088 172-16-1-18.serv-ca.nspace.svc.cluster.local.
```

可以发现这个 `serv-ca` 的服务有 4 个 `pod`，给出了分别对应的 `IP` 以及对应端口是 `8088`。于是通过将服务地址配置为服务端地址后，就可以很简单地实现负载均衡了，Here We Go！！

## 0x0A gRPC+Headless Service 应用 

接上篇，如何将 `gRPC`、`CoreDNS`（集群中的默认 `DNS` 插件）和 `Headless Service` 这三者融合起来，实现长连接 + 负载均衡呢？ 答案就是 [gRPC-DnsResolver](https://github.com/grpc/grpc-go/blob/master/internal/resolver/dns/dns_resolver.go)，默认的 DNS 查询时间是 `30s`

经过改造后的方案效果，见下图，以 `Pod` 视角来看，成功与 4 个后端 `Pod` 建立了长连接，大功告成。![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/K8S-HEADLESS-SVC-6.png)

相关的gRPC客户端连接代码如下：

```go
func ClientConnect(){
  //...
  // addr 是 headless service 的 service name:port
  addr := "myserviceName:12345"

  // 使用 scheme dns:/// ，这样就会使用dns解析到 server 端的 pod ip
  conn, err := grpc.Dial(
      "dns:///"+addr,
      grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`), //grpc.WithBalancerName(roundrobin.Name) 是旧版本写法，已废弃
      grpc.WithInsecure(),
      grpc.WithBlock(),
    )
    //...
}
```

## 0x0B gRPC+DNS解析方案的不足 

但是，在实践中使用该方式可能会存在问题，当发生了Pod扩缩容、上下线的时候；在工程中，Pod 扩容是十分常见的场景，在扩容的时候希望客户端可以及时对新启动的 Pod 快速感知到请求并接收处理。但是对于DNS服务发现，存在下面的[问题](https://github.com/grpc/grpc/issues/12295)：

- 当缩容后，由于 grpc 有连接探活机制，会自动丢弃（剔除）无效连接
- **当扩容后，由于没有感知机制，会导致后续的 Pod 无法被请求到**

上述问题出现的原因是：由于DNS有缓存，gRPC-DNSresolver 解析会缓存解析结果，解析后每 `30min` 才会刷新一次，当 Pod 被销毁时，gRPC 会剔除不健康的 Pod-ip，但是新增的 Pod-ip 必须要在重新链接之后才能解析到

# Kubernetes 应用改造（三）：负载均衡

## Kubernetes 中的负载均衡总结

## 0x00 前言 

   在前文中，曾经讨论过 “Kubernetes 负载均衡失效” 的问题。即 Kubernetes 的默认负载平衡通常不能与 gRPC 一起使用，在不使用 LoadBalance service 的情况下，因为 HTTP/2 链接复用特性，导致客户端的所有请求都发往同一个 Pod，导致负载不均衡。具体原因可见此文：[gRPC Load Balancing on Kubernetes without Tears](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)，原文中对 gRPC 失效的原因的摘抄如下：

> However, gRPC also breaks the standard connection-level load balancing, including what’s provided by Kubernetes. This is because gRPC is built on HTTP/2, and HTTP/2 is designed to have a single long-lived TCP connection, across which all requests are multiplexed—meaning multiple requests can be active on the same connection at any point in time. Normally, this is great, as it reduces the overhead of connection management. However, it also means that (as you might imagine) connection-level balancing isn’t very useful. Once the connection is established, there’s no more balancing to be done. All requests will get pinned to a single destination pod …

问题在于 gRPC 是基于 HTTP/2 长连接的实现，gRPC 客户端与服务端建立长连接后会保持并通过这个连接持续的发送请求。而 Kubernetes Service 属于基于 TCP 连接级别的负载均衡（Connection-level Balancing），虽然支持 gRPC 通信，但其负载均衡的作用却会失效。

#### 解决办法 

gRPC 官方博客的文章 [gRPC Load Balancing](https://grpc.io/blog/grpc-load-balancing/) 给出了 gRPC 负载均衡的思路：

1. 客户端型负载均衡（ZooKeeper/Etcd/Consul/DNS 等），适用于流量较高的场景
2. 代理式的负载均衡（Haproxy/Nginx/Traefik/Envoy/Linkerd 等），适用于服务器需要对外提供统一的入口场景，通常我们说的 ServiceMesh 就是这种方式

此外，这里汇总了目前主流的 [Ingress 网关](https://docs.google.com/spreadsheets/d/16bxRgpO1H_Bn-5xVZ1WrR_I-0A-GOI6egmhvqqLMOmg/edit#gid=1351816353)，可以用作参考。

## 0x01 DnsResolver + Headless-Service 

   这是我们项目最开始使用的 LB 解析方案，也是 TKE 集群上默认自带的服务发现方式。Headless-Service 可以将 Service 的 Endpoint 直接更新到 kubernetes 的 coreDNS 中，所以客户端只要使用 coreDNS 作为名字解析，就能获取下游所有的 Endpoint(s)，然后客户端通过传入的 LB 算法来实现负载均衡策略。

![coredns.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-headless-service.png)

## 0x02 DNS 解析的缺陷 

   在使用 gRPC+DnsResolver+HeadlessService 实现 Kubernetes 负载均衡方案时，有个明显的缺陷就是，**DnsResolver 是定时触发主动去解析获取当前 ServiceName 对应的 Pod 列表的，如果在两次解析的间隔期间，某个 Pod 发生了重启或者漂移（导致 IP 改变），那么这种情况对我们的解析器 DNSResolver 是无法立即感知到的**。

设想一下，有没有一种方法，可以使得我们像使用 Etcd 的 Watch 方法那样，可以监听在某个事件（在 Kubernetes 环境为 Pod）的增删上面呢？翻阅下 Kubernetes 的手册，发现提供查询的 API`/api/v1/watch/namespaces/{namespace}/endpoints/${service}`，这样使得我们可以主动监听某个 Service 下面的 Podlist 变化。

这种方案就是 Watch Endpoint 方式，该方案主要在客户端实现负载均衡，通过 kubernetes API 获取 Service 的 Endpont。**客户端和每个 POD 保持一个长连接**，然后使用 gRPC Client 的负载均衡策略解决问题。开源项目 [kuberesolver](https://github.com/sercand/kuberesolver) 已经实现了这种方式。在下面的篇幅中会简单分析下该项目的代码。

![kuberesolver.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-watcher-endpoint.png)

对比下上述两种常用方法的优缺点：
| LB策略 | 优点 | 缺点 |
| :—–:| :—-: | :—-: |
| Headless Service | 使用方便，原生支持 | 当pod ip 发生改变时，client 无法实时检测到，定时断开连接的解决方式不合理 | 
| Watch Endpoint | 原生支持，支持实时感知pod上下线 | 需要特殊配置 RBAC 权限 |

后文给出了基于TKE环境下Watch Endpoint方式的实践。

## 0x03 代理方案之 Proxy1 - Linkerd 

   Linkerd 是一种基于 CNCF 的 Kubernetes 服务网格，它通过 Sidecar 的方式来实现负载均衡。Linkerd 作为 Service Sidecar 部署在 Pod 中，自动检测 HTTP/2 和 HTTP/1.x 并执行 L7 负载平衡。为服务添加 Linkerd，等价于为每个 Pod 添加了一个很小并且很高效的代理，由这些代理来监控 Kubernetes 的 API，并自动实现 gRPC 的负载均衡。  

Linkerd 通过 webhook，在有 Pod 新增的时候，会向 Pod 中注入额外的 Container，以此获取所有的 Pod。当然作为 service mesh，拥有更多的 feature 保证服务之间的调用，如监控、请求状态、Controller 等等。

![linkerd.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-linkerd.png)

## 0x04 代理方案之 Proxy2 - Nginx 

Nginx 1.13.10 版本支持了 HTTP/2 的负载均衡，[Introducing gRPC Support with NGINX 1.13.10](https://www.nginx.com/blog/nginx-1-13-10-grpc/)

它的原理是：gRPC 客户端与 Nginx 建立 HTTP/2 连接后，Nginx 再与后端 server 建立多个 Http/2 连接。

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/gRPC-nginx-routing.png)

Nginx gRPC 负载均衡的一般配置如下：gRPC 代理要用 `grpc_pass` 指令，协议要用 `grpc://`，同时 `upstream` 机制对 gRPC 也是同样适用的。

```
worker_processes  4;     # cpu core

error_log  logs/error.log;

pid        /run/nginx.pid;

events {
    worker_connections  1o24; # max connection =  worker_processes * worker_connections
#    multi_accept on;
#    use epoll;
}


http {
    log_format  main  '$remote_addr - $remote_user [$time_local]"$request" '
                      '$status $body_bytes_sent"$http_referer" '
                      '"$http_user_agent"';

    upstream grpc_servers {
        server 1.1.1.1:50051;
        server 2.2.2.2:50051;
    }

    server {
        listen 30051 http2;

        access_log logs/access.log main;

        location / {
            grpc_pass grpc://grpc_servers;
        }
    }
}
```

## 0x05 代理方案之 Proxy3 - Istio（Envoy）

   同 Nginx 一样，Envoy 也是非常优秀的 Ingress，都是支持 HTTP/2 的 Load Balancer。Envoy 支持多种 [不同的服务发现机制](https://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html)。

在基于 Istio+Envoy 实现的服务网格中，Istio 的角色是控制平面，它是实现了 Envoy 的发现协议集 xDS 的管理服务器端。Envoy 本身则作为网格的数据平面，和 Istio 通信，获得各种资源的配置并更新自身的代理规则。这里以 Istio 配置 gRPC 负载均衡为例：

![envoy.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-envoy.png)

1、配置一个常规的 Kubernetes Service，作为 Backend：

```
apiVersion: v1
kind: Service
metadata:
  name: grpc-helloworld
  namespace: grpc-example
spec:
  externalIPs:
  - 172.30.10.1
  - 172.30.10.2
  ports:
  - name: grpc
    port: 50051
    protocol: TCP
    targetPort: 50051
  selector:
    app: grpc-helloworld
  sessionAffinity: None
  type: ClusterIP
```

2、配置一个 Istio Gateway，关联 `ingressgateway` 的 80 端口：

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-helloworld-gateway
  namespace: grpc-example
  uid: e5613eb1-fecb-11e8-854e-0050568156a5
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: grpc
      number: 80
      protocol: HTTP
```

3、配置一个 Istio VirtualService，将 Istio Gateway 与 Kubernetes service 关联：

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grpc-helloworld
  namespace: grpc-example
spec:
  gateways:
  - grpc-helloworld-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        regex: .*
    route:
    - destination:
        host: grpc-helloworld
        port:
          number: 50051
```

## 0x06 TKE环境下使用watcher方式实践 

本小结介绍下在TKE环境中，使用Watch Endpoint方式来解决负载均衡的问题。核心原理还是在客户端实现[自定义服务发现](https://github.com/sercand/kuberesolver)的逻辑，即kuberesolver；具体就是客户端在服务发现模块中调用 [Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#-strong-read-operations-endpoints-v1-core-strong-) 监听 Service 对应 Endpoint 变动，通过 Read API 获取 pod ip，然后通过 watch API 得到连接更新信息。

#### kuberesolver 

该库的核心是调用 k8s API 去获取 server 端的 pod 信息，然后基于gRPC提供的Resolver及Balancer接口更新gRPC的连接池；关于此库的分析可见：[Kubernetes 应用改造（六）：使用原生 API 实现 gRPC 负载均衡](https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/)

#### 使用方式 

client 端核心代码如下，从测试结果可以看出，当 pod ip 发生变化时可以正常检测到，并且达到负载均衡，已经得到预期效果。

```go
import (
  kuberesolver "github.com/sercand/kuberesolver/v3"
  "google.golang.org/grpc/resolver"
)
func ClientRun(){
  //...
  kuberesolver.RegisterInCluster()
  //等价于 resolver.Register(kuberesolver.NewBuilder(nil, "kubernetes"))
  var (
    namespace = "YourNamespace"
    servicename = "YourSVCname"
    serviceport = 12345
    endpoint  string 
  )
  
  endpoint = fmt.Sprintf("kubernetes:///%s.%s:%d",servicename,namespace,serviceport)

  conn, err := grpc.Dial("kubernetes:///endpoint",
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`), //新写法
    grpc.WithInsecure())

  //...
}
```

注意上面的`endpoint`构成如下：

- `kubernetes:///`：固定scheme
- `servicename`：服务名字
- `namespace`：namespace
- `serviceport`：服务端口

#### 配置headless Service 

![headless-svc](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/tke/headless-1.png)

#### 配置RBAC 

由于Pod调用了kubernetes的watch API，所以需要为POD生成相应的权限；否则客户端调用会报无权限的错误：`enter image description here` 。需要配置 RBAC 赋予 Client 所在 pod 赋予 endpoint 资源的 `get` 和 `watch` 权限，这样子才可以通过 k8s API 获取到 Server 端的信息。TKE的管理端提供了配置RBAC的便捷接口（使用`yaml`创建，如下图）：

![CALL-ERROR](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/tke/call-error-0.png)

1、配置RBAC  
![rbac-tke](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/tke/config-role-2.png)需要配置 `ServiceAccount`、`Role`以及`RoleBinding`这三个属性：

`ServiceAccount`配置：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: bifrost
  name: grpclb-sa
```

`Role`配置：

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: bifrost
  name: grpclb-role
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "watch"]
```

`RoleBinding`配置（关系）：

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: grpclb-rolebinding
  namespace: bifrost
subjects:
- kind: ServiceAccount
  name: grpclb-sa
  namespace: bifrost
roleRef:
  kind: Role
  name: grpclb-role
  apiGroup: rbac.authorization.k8s.io
```

2、将RBAC配置附加到Client端  
创建上述`3`个实例后，再在 client 端配置 `yaml`，指定刚才创建的 `serviceAccountName`即可：

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: bifrost
  namespace: bifrost
spec:
  containers:
  - name: xxxxxx
    image: xxxxxx
  serviceAccount: grpclb-sa
```

## 0x07 总结

本文分析了 gRPC+Kubernetes 环境的负载均衡的问题，以及对应的解决方案。从当前云原生的发展来看，个人比较看好 Istio。

# Kubernetes 应用改造（六）：使用原生 API 实现 gRPC 负载均衡

## 基于 kubernetes-API 实现 gRPC-LB：kuberesolver

## 0x00 前言

上一篇文章聊到了 Kubernetes 中的负载均衡实现：[Kubernetes 应用改造（三）：负载均衡](https://pandaychen.github.io/2020/06/01/K8S-LOADBALANCE-WITH-KUBERESOLVER/)，其中提到了一款开源项目：[kuberesolver](https://github.com/sercand/kuberesolver)，这篇文章就来分析下它的实现。

## 0x01 Kubernetes 相关概念

了解 Kubernetes 的朋友应该知道，可以通过 Kubernetes APIserver 提供的 [REST API](https://k8smeetup.github.io/docs/tasks/administer-cluster/access-cluster-api/) 直接操作 Kubernetes 的内置资源（如 `Pod`、`Service`、`Deployment`、`ReplicaSet`、`StatefulSet`、`Job`、`CronJob` 等），当然前提是操作方（如某个业务 `Pod` 容器）需要有相关的接口操作权限。参见 [一文带你彻底厘清 Kubernetes 中的证书工作机制](https://zhuanlan.zhihu.com/p/142990931)

#### APIServer
在 Kubernetes 架构中，Etcd 存储位于 APIServer 之后，集群内的各种组件均需要需要统一经过 APIServer 做身份认证和鉴权等安全控制，即认证 + 授权，准入后才可以访问对应的资源数据（存储在 Etcd 中）。

- `GET`（SELECT）：从服务器读取资源，`GET` 请求对应 Kubernetes API 的获取信息功能。需要获取信息时需要使用 GET 请求。
- `POST`（CREATE）：在服务器新建一个资源。`POST` 请求对应 Kubernetes API 的创建功能。需要创建 Pods、ReplicaSet 或者 Service 的时候请使用这种方式发起请求。
- `PUT`（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源），对应更新 Nodes 或 Pods 的状态、ReplicaSet 的自动备份数量等。
- `PATCH`（UPDATE）：在服务器更新资源（客户端提供改变的属性）
- `DELETE`（DELETE）：删除资源，如删除 Pods 等

## 0x02 Kubernetes 的 List-Watch 机制

Kubernetes 提供的 Watch 功能是建立在对 Etcd 的 Watch 之上的，当 Etcd 的 Key-Value（Kubernetes 中资源的持久化信息） 出现变化时，会通知 APIServer。

对于 Kubernetes 集群内的各种资源，Kubernetes 的控制管理器和调度器需要感知到各种资源的状态变化（比如创建、更新、删除），然后根据变化事件履行自己的管理职责。Kubernetes 的实现是使用 APIServer 来充当简单的消息总线（Message Bus）的角色，提供两类机制（Push && Pull）来实现消息传递：

- `List` 机制：提供查询当前最新状态的接口，以 `HTTP` 短连接方式提供
- `Watch`（监听）机制：所有的组件通过 `Watch` 机制建立 `HTTP` 长链接，随时获悉自己感兴趣的资源的变化事件，实现对应的功能后还是调用 APIServer 来写入组件的 `Spec`，比如客户端 （Kubelet/Scheduler/Controller-manager） 通过 List-Watch 机制监听 APIServer 中资源 (Pods/ReplicaSet/Service 等等) 的 Create//Update/Delete 事件，并针对事件类型调用相应的事件处理函数。

![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/K8S-architecture.jpg)

以 Pods 为例，List API 一般为 `GET /api/v1/pods`， Watch API 一般为 `GET /api/v1/watch/pods`，并且会带上 `watch=true`，表示采用 HTTP 长连接持续监听 Pods 相关事件，每当有事件来临，返回一个 WatchEvent。

## Watch 的实现机制

通常我们使用使用 HTTP 大都是短连接方式，那么 Watch 是如何通过 HTTP 长连接获取 APIServer 发来的资源变更事件呢？答案就是 [Chunked transfer encoding](https://zh.wikipedia.org/zh-hans/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)。如下，注意看 HTTP Response 中的 `Transfer-Encoding: chunked`：

```bash
$ curl -i http://{kube-api-server-ip}:8080/api/v1/watch/pods?watch=yes
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 02 Jan 2020 20:22:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

![http_transfer_encoding](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/K8S-httptransfer.png)

下面给出两个实际的查询例子：

#### List 查询[](https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/#list-%E6%9F%A5%E8%AF%A2)

```
$ curl -X GET -i http://127.0.0.1:8080/api/v1/pods
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 15 Mar 2016 08:18:25 GMT
Transfer-Encoding: chunked
```

#### Watcher 查询[](https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/#watcher-%E6%9F%A5%E8%AF%A2)

```
$ curl -X GET -i http://127.0.0.1:8080/api/v1/pods?watch=true
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Date: Tue, 15 Mar 2016 08:00:09 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
```

通过上面的例子对 kuberesolver 中 HTTP 接口返回的接口解析部分的代码有所帮助。

## 0x03 看看 kuberesolver

先简单介绍下 kuberesolver 的使用方法，在 gRPC 客户端做如下形式的调用：

```
import "github.com/sercand/kuberesolver/v3"

// Register kuberesolver to grpc before calling grpc.Dial
kuberesolver.RegisterInCluster()

// it is same as
resolver.Register(kuberesolver.NewBuilder(nil /*custom kubernetes client*/ , "kubernetes"))

// if schema is 'kubernetes' then grpc will use kuberesolver to resolve addresses
cc, err := grpc.Dial("kubernetes:///service.namespace:portname", opts...)
// 或者
grpc.DialContext(ctx,  "kubernetes:///service:grpc", grpc.WithBalancerName("round_robin"), grpc.WithInsecure())
```

kuberesolver 支持如下形式的寻址方式，在 [`parseResolverTarget` 方法](https://github.com/sercand/kuberesolver/blob/master/builder.go#L76) 中实现 服务发现寻址 的解析：

```
kubernetes:///service-name:8080
kubernetes:///service-name:portname
kubernetes:///service-name.namespace:8080

kubernetes://namespace/service-name:8080
kubernetes://service-name:8080/
kubernetes://service-name.namespace:8080/
```

#### kuberesolver 整体架构 

根据先前已有对 gRPC `+` Etcd/Consul 负载均衡器的实现经验，思路大致都是一样的：

1. 实现 `Resolver`：创建全局 `Resolver` 的标识符及实现 `grpc.Resolver` 的接口
2. 实现 `Watcher`（一般作为 `Resolver` 的子协程单独创建）：接收 Kubernetes 的 `API` 的改变通知并调用 gRPC 的接口通知 gRPC 内部

kuberesolver 的整体项目架构如下所示：![kuberesolver 整体架构](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/K8S-kuberesolver.png)

下面按照架构图的模块对源码进一步分析。

## 0x04 结构定义 && 功能 

[`models.go`](https://github.com/sercand/kuberesolver/blob/master/models.go#L5) 中定义了 kubernetes 的 API 返回结果的 `JSON` 结构体，对于 HTTP-Watcher 接口返回的结果，即监听的事件分为三种：

```
const (
	Added    EventType = "ADDED"
	Modified EventType = "MODIFIED"
	Deleted  EventType = "DELETED"
	Error    EventType = "ERROR"
)
```

## 0x05 Resolver 实现 

先看看 [`Resolver` 的实现](https://github.com/sercand/kuberesolver/blob/master/builder.go)，`Resolver` 的 name 为 `kubernetes`，此值会用在 `grpc.Dial()` 的模式中。

```
const (
	kubernetesSchema = "kubernetes"
	defaultFreq      = time.Minute * 30
)
```

#### kubeBuilder 及 kubeBuilder.Build 方法[](https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/#kubebuilder-%E5%8F%8A-kubebuilderbuild-%E6%96%B9%E6%B3%95)

`kubeBuilder` 结构及 `NewBuilder` 初始化，注意其中的 `k8sClient`，本质就是一个 [`http.Client` 封装](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L25)：

```
// NewBuilder creates a kubeBuilder which is used by grpc resolver.
func NewBuilder(client K8sClient, schema string) resolver.Builder {
	return &kubeBuilder{
		k8sClient: client,
		schema:    schema,
	}
}

type kubeBuilder struct {
	k8sClient K8sClient
	schema    string
}
```

下面是核心函数 [`Build`](https://github.com/sercand/kuberesolver/blob/master/builder.go#L122)，构造生成 `resolver.Builder` 的方法：  

```
// Build creates a new resolver for the given target.
//
// gRPC dial calls Build synchronously, and fails if the returned error is
// not nil.
func (b *kubeBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	if b.k8sClient == nil {
		if cl, err := NewInClusterK8sClient(); err == nil {
			b.k8sClient = cl
		} else {
			return nil, err
		}
	}

	// 解析用户传入的 resolver 地址
	ti, err := parseResolverTarget(target)
	if err != nil {
		return nil, err
	}
	if ti.serviceNamespace == "" {
		ti.serviceNamespace = getCurrentNamespaceOrDefault()
	}

	// 初始化 kResolver 结构体
	ctx, cancel := context.WithCancel(context.Background())
	r := &kResolver{
		target:    ti,
		ctx:       ctx,
		cancel:    cancel,
		cc:        cc,
		rn:        make(chan struct{}, 1),
		k8sClient: b.k8sClient,
		t:         time.NewTimer(defaultFreq),
		freq:      defaultFreq,

		endpoints: endpointsForTarget.WithLabelValues(ti.String()),
		addresses: addressesForTarget.WithLabelValues(ti.String()),
	}

	// 开启单独的 watcher 模式
	go until(func() {
		r.wg.Add(1)
		err := r.watch()
		if err != nil && err != io.EOF {
			grpclog.Errorf("kuberesolver: watching ended with error='%v', will reconnect again", err)
		}
	}, time.Second, ctx.Done())
	return r, nil
}
```

此方法的核心步骤（通用范式）是：  
1、首先，解析 `parseResolverTarget` 传入的地址，本项目中支持如下格式

```
kubernetes:///service-name:8080
kubernetes:///service-name:portname
kubernetes:///service-name.namespace:8080

kubernetes://namespace/service-name:8080
kubernetes://service-name:8080/
kubernetes://service-name.namespace:8080/
```

上面的地址格式被解析为如下结构 `targetInfo`，在 Kubernetes 的定义中，APIServer 的调用服务地址必须包含如下关键信息：

- `serviceName`：服务 name
- `serviceNamespace`：命名空间
- `port`：APIServer 的端口

```
type targetInfo struct {
	serviceName       string
	serviceNamespace  string
	port              string
	resolveByPortName bool
	useFirstPort      bool
}

func (ti targetInfo) String() string {
	return fmt.Sprintf("kubernetes://%s/%s:%s", ti.serviceNamespace, ti.serviceName, ti.port)
}
```

2、第二步，根据解析的结果，初始化 `kResolver` 结构体，此结构体包含了独立 goroutine 运行所需要的所有成员：

```
type kResolver struct {
	target targetInfo			//APISERVER 地址
	ctx    context.Context		// 用于 goroutine 退出
	cancel context.CancelFunc
	cc     resolver.ClientConn	// 用于调用 grpc 的方法通知
	// rn channel is used by ResolveNow() to force an immediate resolution of the target.
	rn        chan struct{}		// 用于触发 ResolveNow() 逻辑的信号 signal
	k8sClient K8sClient			// 用于请求 APISERVER 的 HTTP 客户端
	// wg is used to enforce Close() to return after the watcher() goroutine has finished.
	wg   sync.WaitGroup
	t    *time.Timer			// 用于 LIST 方式（定时拉取一次全量，用于初始化时）
	freq time.Duration

	endpoints prometheus.Gauge
	addresses prometheus.Gauge
}
```

3、最后，开启独立的子 goroutine，实现 `Watcher` 逻辑

```
...
	// 开启单独的 watcher 模式
	go until(func() {
		r.wg.Add(1)
		err := r.watch()
		if err != nil && err != io.EOF {
			grpclog.Errorf("kuberesolver: watching ended with error='%v', will reconnect again", err)
		}
	}, time.Second, ctx.Done())
...
```

#### kResolver 的实现 

作为 `resovler.Builder` 的实例化实现（在 `return` 中返回 `resovler.Builder`），`kResolver` 必须实现 `resovler.Builder` 定义的方法：

- `ResolveNow`：立即执行一次 resolve，本方法非必要实现
- `Close`：关闭 Watcher，回收资源

```
// ResolveNow will be called by gRPC to try to resolve the target name again.
// It's just a hint, resolver can ignore this if it's not necessary.
func (k *kResolver) ResolveNow(resolver.ResolveNowOptions) {
	select {
	case k.rn <- struct{}{}:
	default:
	}
}

// Close closes the resolver.
func (k *kResolver) Close() {
	k.cancel()
	k.wg.Wait()
}
```

#### kResolver.resolve 

`kResolver.resolve` 是 Watcher 的核心方法，这是个经典的 `for-select-loop`，主要接受如下几类触发的 channel 事件：

1. `k.ctx.Done()`：上层通知退出
2. `k.t.C`：定时器触发
3. `k.rn`：`ResolveNow` 触发
4. `up, hasMore := <-sw.ResultChan()`：`streamWatcher` 有新消息（事件）通知时，调用 `k.handle(up.Object)` 处理事件，即监听的资源发生了改变（扩容 / 缩容 / 下线等）

```
func (k *kResolver) handle(e Endpoints) {
	// 通过 makeAddresses 得到全量 IP 列表
	result, _ := k.makeAddresses(e)
	// k.cc.NewServiceConfig(sc)
	if len(result) > 0 {
		k.cc.NewAddress(result)
	}

	k.endpoints.Set(float64(len(e.Subsets)))
	k.addresses.Set(float64(len(result)))
}

func (k *kResolver) resolve() {
	e, err := getEndpoints(k.k8sClient, k.target.serviceNamespace, k.target.serviceName)
	if err == nil {
		k.handle(e)
	} else {
		grpclog.Errorf("kuberesolver: lookup endpoints failed: %v", err)
	}
	// Next lookup should happen after an interval defined by k.freq.
	k.t.Reset(k.freq)
}

func (k *kResolver) watch() error {
	defer k.wg.Done()
	// watch endpoints lists existing endpoints at start
	sw, err := watchEndpoints(k.k8sClient, k.target.serviceNamespace, k.target.serviceName)
	if err != nil {
		return err
	}
	for {
		select {
		case <-k.ctx.Done():
			return nil
		case <-k.t.C:
			k.resolve()
		case <-k.rn:
			k.resolve()
		case up, hasMore := <-sw.ResultChan():
			if hasMore {
				k.handle(up.Object)
			} else {
				return nil
			}
		}
	}
}
```

## 0x06 K8sClient 分析 

在分析 Watcher 的逻辑之前，我们先看下 [`k8sClient` 封装](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go)，正如文章开始我们说的那样，提供了 `List` 和 `Watch` 两种调用方式：

- [`watchEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L120)：提供 `Watch` 方式的增量接口
- [`getEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#97)：提供 `List` 方式的全量接口

#### k8sClient 抽象接口 && 定义 && 初始化 

```
// K8sClient is minimal kubernetes client interface
// 大写为 interface{}
type K8sClient interface {
	Do(req *http.Request) (*http.Response, error)
	GetRequest(url string) (*http.Request, error)
	Host() string
}

// 小写为具体实现
type k8sClient struct {
	host       string
	token      string		//http-token 认证方式
	httpClient *http.Client	// 真正的 http.Client
}
```

注意看初始化里面：

1. APIServer 的证书加载路径
2. `net.http` 默认是长连接方式，不过这里并未涉及超时等相关参数，默认为阻塞 client

```
const(
	serviceAccountToken     = "/var/run/secrets/kubernetes.io/serviceaccount/token"
	serviceAccountCACert    = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
	kubernetesNamespaceFile = "/var/run/secrets/kubernetes.io/serviceaccount/namespace"
)

// NewInClusterK8sClient creates K8sClient if it is inside Kubernetes
func NewInClusterK8sClient() (K8sClient, error) {
	host, port := os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_SERVICE_PORT")
	if len(host) == 0 || len(port) == 0 {
		return nil, fmt.Errorf("unable to load in-cluster configuration, KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT must be defined")
	}
	token, err := ioutil.ReadFile(serviceAccountToken)
	if err != nil {
		return nil, err
	}
	ca, err := ioutil.ReadFile(serviceAccountCACert)
	if err != nil {
		return nil, err
	}
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(ca)
	transport := &http.Transport{TLSClientConfig: &tls.Config{
		MinVersion: tls.VersionTLS10,
		RootCAs:    certPool,
	}}
	httpClient := &http.Client{Transport: transport, Timeout: time.Nanosecond * 0}

	return &k8sClient{
		host:       "https://" + net.JoinHostPort(host, port),
		token:      string(token),
		httpClient: httpClient,
	}, nil
}
```

#### getEndpoints && watchEndpoints[](https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/#getendpoints--watchendpoints)

[`getEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L97) 就是普通的 GET 查询，采用 HTTP 短连接方式

```
func getEndpoints(client K8sClient, namespace, targetName string) (Endpoints, error) {
	u, err := url.Parse(fmt.Sprintf("%s/api/v1/namespaces/%s/endpoints/%s",
		client.Host(), namespace, targetName))
	if err != nil {
		return Endpoints{}, err
	}
	req, err := client.GetRequest(u.String())
	if err != nil {
		return Endpoints{}, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return Endpoints{}, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return Endpoints{}, fmt.Errorf("invalid response code %d", resp.StatusCode)
	}
	result := Endpoints{}
	err = json.NewDecoder(resp.Body).Decode(&result)
	return result, err
}
```

而 [`watchEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L120) 则实现了 Http 的长连接方式（由服务端控制），通过调用 `newStreamWatcher` 返回一个 `streamWatcher`，传入参数为 `resp.Body`。  

特别注意这里的 HTTP 客户端的初始化方式，是适用于 `Transfer-Encoding: chunked` 模式的：

```
func watchEndpoints(client K8sClient, namespace, targetName string) (watchInterface, error) {
	u, err := url.Parse(fmt.Sprintf("%s/api/v1/watch/namespaces/%s/endpoints/%s",
		client.Host(), namespace, targetName))
	if err != nil {
		return nil, err
	}
	req, err := client.GetRequest(u.String())
	if err != nil {
		return nil, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	if resp.StatusCode != http.StatusOK {
		defer resp.Body.Close()
		return nil, fmt.Errorf("invalid response code %d", resp.StatusCode)
	}
	return newStreamWatcher(resp.Body), nil
}
```

## 0x07 Watcher 实现 

接上一小节，Watcher 逻辑代码主要 [在此](https://github.com/sercand/kuberesolver/blob/master/stream.go)，其本质就是 [`streamWatcher`](https://github.com/sercand/kuberesolver/blob/master/stream.go#L26)，`streamWatcher` 扮演了这么一种角色，先回想下一般的 HTTP 请求模型：

```
	//...
	resp, err := client.Do(req)
	if err != nil {
		//...
	}
	defer resp.Body.Close()

	json.NewDecoder(resp.Body).Decode(&result)
	//...
```

上面这个是非常普通的 HTTP 请求代码，其中的 `resp.Body` 的类型是 `io.ReadCloser`，在 HTTP 的 `Transfer-Encoding: chunked` 模式下，如果不关闭 `resp.Body`，那么我们就可以实现在这个 stream 长连接上不断的读取数据。  
切记不要调用 `defer resp.Body.Close()`。

#### streamWatcher 的结构 

`streamWatcher` 的结构如下，其中的 `result chan Event` 用于向外部传递解析的结果（HTTP-API 接口返回）

```
// Interface can be implemented by anything that knows how to watch and report changes.
type watchInterface interface {
	// Stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// Returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, this channel will be closed, in which case the
	// watch should be completely cleaned up.
	ResultChan() <-chan Event
}

// StreamWatcher turns any stream for which you can write a Decoder interface
// into a watch.Interface.
type streamWatcher struct {
	result  chan Event		// 用来向外部传递解析 respose 的结果
	r       io.ReadCloser
	decoder *json.Decoder
	sync.Mutex
	stopped bool
}
```

注意上面的 `watchInterface` 暴露的对外部的接口：

1. `ResultChan` 方法：返回 `streamWatcher` 结构中的 `result chan Event` 成员给外部
2. `Stop` 方法：关闭整个 `streamWatcher` 的监听逻辑，清理资源并退出

`streamWatcher` 的实现是个非常经典的观察者模式。

#### 初始化 NewStreamWatcher && Watcher[](https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/#%E5%88%9D%E5%A7%8B%E5%8C%96-newstreamwatcher--watcher)

在 `newStreamWatcher` 中，通过单独调用 `go sw.receive()` 来开启独立的 Watcher 逻辑：

```
// NewStreamWatcher creates a StreamWatcher from the given io.ReadClosers.
func newStreamWatcher(r io.ReadCloser) watchInterface {
	sw := &streamWatcher{
		r:       r,
		decoder: json.NewDecoder(r),
		result:  make(chan Event),
	}
	go sw.receive()
	return sw
}

// receive reads result from the decoder in a loop and sends down the result channel.
func (sw *streamWatcher) receive() {
	defer close(sw.result)
	defer sw.Stop()		// 在此方法中关闭 resp.Body
	//watcher LOOP
	for {
		// 通过 sw.Decode() 不停的触发 Read
		obj, err := sw.Decode()
		if err != nil {
			// 错误处理
			// Ignore expected error.
			if sw.stopping() {
				return
			}
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				grpclog.Infof("kuberesolver: Unexpected EOF during watch stream event decoding: %v", err)
			default:
				grpclog.Infof("kuberesolver: Unable to decode an event from the watch stream: %v", err)
			}
			return
		}
		sw.result <- obj
	}
}
```

最后看下 `Decode` 方法，解析 HTTP-API 的响应，将 JSON 序列化的结果返回：

```
// Decode blocks until it can return the next object in the writer. Returns an error
// if the writer is closed or an object can't be decoded.
func (sw *streamWatcher) Decode() (Event, error) {
	var got Event
	if err := sw.decoder.Decode(&got); err != nil {
		return Event{}, err
	}
	switch got.Type {
	case Added, Modified, Deleted, Error:
		return got, nil
	default:
		return Event{}, fmt.Errorf("got invalid watch event type: %v", got.Type)
	}
}
```

# gRPC 应用篇之客户端 Connection Pool

## 连接池的实现分析 && 是否需要 Tcp/gRPC 客户端连接池 ？&& 通用连接池的实现

## 0x00 连接池 

之前分析过 `go-redis` 的连接池实现：[Go-Redis 连接池（Pool）源码分析](https://pandaychen.github.io/2020/02/22/A-REDIS-POOL-ANALYSIS/)，在项目应用中，连接池的最大好处是减少 TCP 建立握手 / 挥手的时间，实现 TCP 连接复用，从而降低通信时延和提高性能。

通常一些高性能中间件都提供了内置的 TCP 连接池，如刚才说的 `go-redis`,[`go-sql-driver`](https://github.com/go-sql-driver/mysql#connection-pool-and-timeouts) 等等，关于连接池，一个良好的设计是对用户屏蔽底层的实现，如存储 / keepalive / 关闭 / 自动重连等，只对用户提供简单的获取接口。

## 0x01 gRPC 连接池的实现 

#### 实现原则 

1、连接池的基本属性  
首先要选择合适的数据结构来存放连接池（如slice、map、linklist等），通常连接池属性包含**最大空闲连接数**、**最大活跃连接数**以及**最小活跃连接数**等，定义分别如下：

- 最大空闲连接数：连接池一直保持的连接数，无论这些连接被使用与否都会被保持。如果客户端对连接池的使用量不大，便会造成服务端连接资源的浪费
- 最大活跃连接数：连接池最多保持的连接数，如果客户端请求超过次数，便要根据池满的处理机制来处理没有得到连接的请求
- 最小活跃连接数：连接池初始化或是其他时间，连接池内一定要存储的活跃连接数

2、连接池的扩缩容机制如何实现  

- 扩容：当请求到来时，如果连接池中没有空闲的连接，同时连接数也没有达到最大活跃连接数，便会按照特定的增长策略创建新的连接服务该请求，同时**用完之后归还到池中**，而非关闭连接
- 缩容：当连接池一段时间没有被使用，同时池中的连接数超过了最大空闲连接数，那么便会关闭一部分连接，使池中的连接数始终维持在最大空闲连接数

3、空闲连接的超时与keepalive  

- 超时：如果连接没有被客户端使用的话，便会成为空闲连接，在一段时间后，服务端可能会根据自己的超时策略关闭空闲连接，此时空闲连接已经失效，如果客户端再使用失效的连接，便会通信失败。为了避免这种情况发生，通常连接池中的连接设有最大空闲超时时间(最好略小于服务器的空闲连接超时时间)，在从池中获取连接时，判断是否空闲超时，如果超时则关闭，没有超时则可以继续使用
    
- keepalive：一般使用某种机制（如心跳包等）保活某个连接，防止服务端/客户端主动Reset；如果服务器发生重启，那么连接池中的连接便会全部失效，即连接池失效了，如何优化此类场景呢？
    

如何解决上述场景keepalive的失效问题呢？

1. 连接池设置一个Ping函数，专门用来做连接的保活。在从池中获取连接的时候，Ping一下服务器，如果得到响应，则连接依然有效，便可继续使用，如果超时无响应，则关闭该连接，生成新的连接，由于每次都要Ping一下，必然会增加延迟。也可以后台用一个线程或者协程定期的执行Ping函数，进行连接的保活，缺点是感知连接的失效会有一定的延迟，从池中仍然有可能获取到失效的连接。
    
2. 客户端加入相应的重试机制。比如重试`3`次，前两次从池中获取连接执行，如果报的错是失效的连接等有关连接问题的错误，那么第3次从池中获取的时候带上参数，指定获取新建的连接，同时连接池移除前两次获取的失效的连接。
    

4、连接池满的处理机制  
当连接池容量超上限时，有`2`种处理机制：

1. 对于连接池新建的连接，并返回给客户端，当客户端用完时，如果池满则关闭连接，否则放入池中
2. 设置一定的超时时间来等待空闲连接。需要客户端加入重试机制，避免因超时之后获取不到空闲连接产生的错误

5、连接池异常的容错机制  
连接池异常时，退化为新建连接的方式，避免影响正常请求，同时，需要相关告警通知开发人员

6、开启异步连接池回收  
参考go-redis的连接池实现，对于空闲连接（超过允许最大空闲时间）超时后，主动关闭连接池中的连接

#### 连接池实现的步骤 

1. 服务启动时建立连接池
2. 初始化连接池，建立最大空闲连接数个连接
3. 请求到来时，从池中获取一个连接。如果没有空闲连接且连接数没有达到最大活跃连接数，则新建连接；如果达到最大活跃连接数，允许设置一定的超时时间，等待获取空闲连接
4. 获取到连接后进行通信服务
5. 释放连接，此时是将连接放回连接池，如果池满则关闭连接
6. 释放连接池，关闭所有连接

#### gRPC的特性 

在实现gRPC的连接池之前，需要先了解gRPC的多路复用及超时重连两个特性：

1、多路复用  
gRPC使用HTTP/2作为应用层的传输协议，HTTP/2会复用底层的TCP连接。每一次RPC调用会产生一个新的Stream，每个Stream包含多个Frame，Frame是HTTP/2里面最小的数据传输单位。同时每个Stream有唯一的ID标识，如果是客户端创建的则ID是奇数，服务端创建的ID则是偶数。如果一条连接上的ID使用完了，Client会新建一条连接，Server也会给Client发送一个`GOAWAY Frame`强制让Client新建一条连接。一条gRPC连接允许并发的发送和接收多个Stream，控制的参数便是`MaxConcurrentStreams`

2、超时重连  
在通过调用`Dial`/`DialContext`方法创建连接时，默认只是返回`ClientConn`结构体指针，同时会启动一个goroutine异步的去建立连接。如果想要等连接建立完再返回，可以指定`grpc.WithBlock()`传入`Options`来实现。

超时机制很简单，在调用的时候传入一个timeout的`context`就可以了。重连机制通过启动一个goroutine异步的去建立连接实现的，可以避免服务器因为连接空闲时间过长关闭连接、服务器重启等造成的客户端连接失效问题。也就是说通过**gRPC的重连机制可以完美的解决连接池设计原则中的空闲连接的超时与keepalive问题**。

3、gRPC默认参数优化（基于大块数据传输场景）  

```go
MaxSendMsgSizeGRPC	//最大允许发送的字节数，默认4MiB，如果超过了GRPC会报错。Client和Server我们都调到4GiB

MaxRecvMsgSizeGRPC	//最大允许接收的字节数，默认4MiB，如果超过了GRPC会报错。Client和Server我们都调到4GiB

InitialWindowSize	//基于Stream的滑动窗口，类似于TCP的滑动窗口，用来做流控，默认64KiB，吞吐量上不去，Client和Server我们调到1GiB

InitialConnWindowSize	//基于Connection的滑动窗口，默认16 * 64KiB，吞吐量上不去，Client和Server我们也都调到1GiB

KeepAliveTime	//每隔KeepAliveTime时间，发送PING帧测量最小往返时间，确定空闲连接是否仍然有效，我们设置为10s

KeepAliveTimeout	//超过KeepAliveTimeout，关闭连接，我们设置为3s

PermitWithoutStream	//如果为true，当连接空闲时仍然发送PING帧监测，如果为false，则不发送忽略。我们设置为true
```

## 0x02 gRPC Pool 分析 

滴滴开源的 [grpc 连接池](https://github.com/shimingyah/pool)，代码不长。简单分析下：

#### grpc.conn 封装 

代码主要 [在此](https://github.com/shimingyah/pool/blob/master/conn.go)，

```go
// Conn single grpc connection inerface
type Conn interface {
	// Value return the actual grpc connection type *grpc.ClientConn.
	Value() *grpc.ClientConn

	// Close decrease the reference of grpc connection, instead of close it.
	// if the pool is full, just close it.
	Close() error
}

// Conn is wrapped grpc.ClientConn. to provide close and value method.
type conn struct {
	cc   *grpc.ClientConn       // 封装真正的 grpc.conn
	pool *pool                  // 指向的 pool
	once bool
}
```

#### Pool 封装 

gRPC-Pool 封装的主要代码 [在此](https://github.com/shimingyah/pool/blob/master/pool.go)，一个 `interface`，一个 `struct`：  
Pool 对外部暴露的接口就 `3` 个：

- `Get`：从连接池获取连接
- `Close`：关闭连接池
- `Status`：打印连接池信息

```go
// Pool interface describes a pool implementation.
// An ideal pool is threadsafe and easy to use.
type Pool interface {
	// Get returns a new connection from the pool. Closing the connections puts
	// it back to the Pool. Closing it when the pool is destroyed or full will
	// be counted as an error. we guarantee the conn.Value() isn't nil when conn isn't nil.
	Get() (Conn, error)     // 从池中取连接

	// Close closes the pool and all its connections. After Close() the pool is
	// no longer usable. You can't make concurrent calls Close and Get method.
	// It will be cause panic.
	Close() error           // 关闭池，关闭池中的连接

	// Status returns the current status of the pool.
	Status() string
}

// gRPC 连接池的定义
type pool struct {
	// atomic, used to get connection random
	index uint32
	// atomic, the current physical connection of pool
	current int32
	// atomic, the using logic connection of pool
	// logic connection = physical connection * MaxConcurrentStreams
	ref int32
	// pool options
	opt Options
    // all of created physical connections
    // 真正存储连接的结构
	conns []*conn
	// the server address is to create connection.
	address string
	// control the atomic var current's concurrent read write.
	sync.RWMutex
}
```

```go
// New return a connection pool.
func New(address string, option Options) (Pool, error) {
	if address == "" {
		return nil, errors.New("invalid address settings")
	}
	if option.Dial == nil {
		return nil, errors.New("invalid dial settings")
	}
	if option.MaxIdle <= 0 || option.MaxActive <= 0 || option.MaxIdle> option.MaxActive {
		return nil, errors.New("invalid maximum settings")
	}
	if option.MaxConcurrentStreams <= 0 {
		return nil, errors.New("invalid maximun settings")
	}

	p := &pool{
		index:   0,
		current: int32(option.MaxIdle),
		ref:     0,
		opt:     option,
		conns:   make([]*conn, option.MaxActive),
		address: address,
	}

	for i := 0; i < p.opt.MaxIdle; i++ {
		c, err := p.opt.Dial(address)
		if err != nil {
			p.Close()
			return nil, fmt.Errorf("dial is not able to fill the pool: %s", err)
		}
		p.conns[i] = p.wrapConn(c, false)
	}
	log.Printf("new pool success: %v\n", p.Status())

	return p, nil
}

func (p *pool) incrRef() int32 {
	newRef := atomic.AddInt32(&p.ref, 1)
	if newRef == math.MaxInt32 {
		panic(fmt.Sprintf("overflow ref: %d", newRef))
	}
	return newRef
}

func (p *pool) decrRef() {
	newRef := atomic.AddInt32(&p.ref, -1)
	if newRef < 0 {
		panic(fmt.Sprintf("negative ref: %d", newRef))
	}
	if newRef == 0 && atomic.LoadInt32(&p.current) > int32(p.opt.MaxIdle) {
		p.Lock()
		if atomic.LoadInt32(&p.ref) == 0 {
			log.Printf("shrink pool: %d ---> %d, decrement: %d, maxActive: %d\n",
				p.current, p.opt.MaxIdle, p.current-int32(p.opt.MaxIdle), p.opt.MaxActive)
			atomic.StoreInt32(&p.current, int32(p.opt.MaxIdle))
			p.deleteFrom(p.opt.MaxIdle)
		}
		p.Unlock()
	}
}

func (p *pool) reset(index int) {
	conn := p.conns[index]
	if conn == nil {
		return
	}
	conn.reset()
	p.conns[index] = nil
}

func (p *pool) deleteFrom(begin int) {
	for i := begin; i < p.opt.MaxActive; i++ {
		p.reset(i)
	}
}

// Get see Pool interface.
func (p *pool) Get() (Conn, error) {
	// the first selected from the created connections
	nextRef := p.incrRef()
	p.RLock()
	current := atomic.LoadInt32(&p.current)
	p.RUnlock()
	if current == 0 {
		return nil, ErrClosed
	}
	if nextRef <= current*int32(p.opt.MaxConcurrentStreams) {
		next := atomic.AddUint32(&p.index, 1) % uint32(current)
		return p.conns[next], nil
	}

	// the number connection of pool is reach to max active
	if current == int32(p.opt.MaxActive) {
		// the second if reuse is true, select from pool's connections
		if p.opt.Reuse {
			next := atomic.AddUint32(&p.index, 1) % uint32(current)
			return p.conns[next], nil
		}
		// the third create one-time connection
		c, err := p.opt.Dial(p.address)
		return p.wrapConn(c, true), err
	}

	// the fourth create new connections given back to pool
	p.Lock()
	current = atomic.LoadInt32(&p.current)
	if current <int32(p.opt.MaxActive) && nextRef > current*int32(p.opt.MaxConcurrentStreams) {
		// 2 times the incremental or the remain incremental
		increment := current
		if current+increment > int32(p.opt.MaxActive) {
			increment = int32(p.opt.MaxActive) - current
		}
		var i int32
		var err error
		for i = 0; i < increment; i++ {
			c, er := p.opt.Dial(p.address)
			if er != nil {
				err = er
				break
			}
			p.reset(int(current + i))
			p.conns[current+i] = p.wrapConn(c, false)
		}
		current += i
		log.Printf("grow pool: %d ---> %d, increment: %d, maxActive: %d\n",
			p.current, current, increment, p.opt.MaxActive)
		atomic.StoreInt32(&p.current, current)
		if err != nil {
			p.Unlock()
			return nil, err
		}
	}
	p.Unlock()
	next := atomic.AddUint32(&p.index, 1) % uint32(current)
	return p.conns[next], nil
}

// Close see Pool interface.
func (p *pool) Close() error {
	atomic.StoreUint32(&p.index, 0)
	atomic.StoreInt32(&p.current, 0)
	atomic.StoreInt32(&p.ref, 0)
	p.deleteFrom(0)
	log.Printf("close pool success: %v\n", p.Status())
	return nil
}

// Status see Pool interface.
func (p *pool) Status() string {
	return fmt.Sprintf("address:%s, index:%d, current:%d, ref:%d. option:%v",
		p.address, p.index, p.current, p.ref, p.opt)
}
```

#### grpc.Pool 的使用 

本小节给出基于 gRPC 连接池的 CS 调用例子，如下：  

服务端代码：

```go
func main() {
	flag.Parse()

	listen, err := net.Listen("tcp", fmt.Sprintf("127.0.0.1:%v", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// 调整 grpc 的默认参数
	s := grpc.NewServer(
		grpc.InitialWindowSize(pool.InitialWindowSize),
		grpc.InitialConnWindowSize(pool.InitialConnWindowSize),
		grpc.MaxSendMsgSize(pool.MaxSendMsgSize),
		grpc.MaxRecvMsgSize(pool.MaxRecvMsgSize),
		grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
			PermitWithoutStream: true,
		}),
		grpc.KeepaliveParams(keepalive.ServerParameters{
			Time:    pool.KeepAliveTime,
			Timeout: pool.KeepAliveTimeout,
		}),
	)
	pb.RegisterEchoServer(s, &server{})

	if err := s.Serve(listen); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

客户端代码：

```go
func main() {
	flag.Parse()

	p, err := pool.New(*addr, pool.DefaultOptions)
	if err != nil {
		log.Fatalf("failed to new pool: %v", err)
	}
	defer p.Close()

	conn, err := p.Get()
	if err != nil {
		log.Fatalf("failed to get conn: %v", err)
	}
	defer conn.Close()

	client := pb.NewEchoClient(conn.Value())
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	res, err := client.Say(ctx, &pb.EchoRequest{Message: []byte("hi")})
	if err != nil {
		log.Fatalf("unexpected error from Say: %v", err)
	}
	fmt.Println("rpc response:", res)
}
```

## 0x03 通用 TCP 连接池的实现 

基于前面的分析，如何实现一个通用的 Tcp 连接池呢？以此项目[A golang general network connection poolction pool](https://github.com/silenceper/pool)

- 连接池中连接类型为`interface{}`，更通用
- 连接的最大空闲时间，超时的连接将关闭丢弃，可避免空闲时连接自动失效问题
- 支持用户设定 ping 方法，检查连接的连通性，无效的连接将丢弃
- 使用channel高效处理池中的连接

#### 使用方法[](https://pandaychen.github.io/2020/10/03/DO-WE-NEED-GRPC-CLIENT-POOL/#%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95)

```
//factory 创建连接的方法
factory := func() (interface{}, error) { 
	return net.Dial("tcp", "127.0.0.1:12345") 
}

//close 关闭连接的方法
close := func(v interface{}) error { 
	return v.(net.Conn).Close() 
}

//创建一个连接池： 初始化5，最大空闲连接是20，最大并发连接30
poolConfig := &pool.Config{
	InitialCap: 5,//资源池初始连接数
	MaxIdle:   20,//最大空闲连接数
	MaxCap:     30,//最大并发连接数
	Factory:    factory,
	Close:      close,
	//Ping: ping,
	//连接最大空闲时间，超过该时间的连接 将会关闭，可避免空闲时连接EOF，自动失效的问题
	IdleTimeout: 15 * time.Second,
}
p, err := pool.NewChannelPool(poolConfig)
if err != nil {
	fmt.Println("err=", err)
}

//从连接池中取得一个连接
v, err := p.Get()

//do something
//conn=v.(net.Conn)

//将连接放回连接池中
p.Put(v)
//释放连接池中的所有连接
p.Release()
//查看当前连接中的数量
current := p.Len()
```

## 0x04 后记：是否需要gRPC的连接池[](https://pandaychen.github.io/2020/10/03/DO-WE-NEED-GRPC-CLIENT-POOL/#0x04-%E5%90%8E%E8%AE%B0%E6%98%AF%E5%90%A6%E9%9C%80%E8%A6%81grpc%E7%9A%84%E8%BF%9E%E6%8E%A5%E6%B1%A0)

由于现网中，笔者使用的场景大多数都是基于RPC的长连接（如Etcd/Consul/kubernetes-endpoint等），即gRPC 内建的 balancer 已经提供了优秀的连接管理支持（而且还可以自己实现池及Loadbalancer策略），每个后端实例一个 HTTP2 物理连接。**个人认为连接池机制比较适合于短连接的场景**

如果连接池是用于短链接，那么对于k8s的dns机制就比较有用了，链接定时断开，重新去解析svc背后的连接来建立。
## 0x05 总结[](https://pandaychen.github.io/2020/10/03/DO-WE-NEED-GRPC-CLIENT-POOL/#0x05-%E6%80%BB%E7%BB%93)

- Redis/MySQL 连接池这种是单个连接只能负载一个并发，没有可用连接时会阻塞执行，并发跟不上的时候连接池相应调大点，性能会提升
- gRPC 内建的 balancer 已经有很好的连接管理的支持了，每个后端实例一个 HTTP2 物理连接
- gRPC 的 HTTP2 连接有复用能力，N 个 goroutine 用一个 HTTP2 连接没有任何问题，并不会单纯因为没有可用连接而阻塞

# RPC与HTTP
一个 api 可以用 restful api 表达 也可以用 GRPC 表达.  
为什么要选择用 GRPC, 而不是 restful api?

1. protobuf 的优势 -- 其他 RPC 也可以自定义自己的二进制协议. 但是 http 是文本协议.  
1.1 如果基于 http 做出了基于文本协议的二进制协议. 那其实也可以称之为另一种 RPC 实现.  
2. RPC 可以基于 tcp 也可以基于 http. 无论基于什么, 最终的表现形式就是 调用一个 api(通过远程方式).  
当然 restful api 也同样是通过远程方式调用一个 api. 只不过 http 协议本身支持的功能是有限的. 譬如:  
1) protocol buffers  
2) 连接池  
3) 参数如何表达  
4) 返回值如何表达  
5) "服务发现"，"负载均衡"，"熔断降级"  
如果将这些功能都加进来, 实际上就变成了 RPC  
3. RPC 本身也有很多分类 譬如 简单的 json-RPC. 高效的 GPRC...  
  
总之,RPC 有很多种表现形式,RPC 的裁剪能力是很强的.各种各样的 RPC 实现满足了各种各样的需求.  
但是 http 协议本身的限制以及基于 http 的生态(nginx) 限制了 restful api 做为服务内部通信的发展.

## 从学术方面来说
HTTP 创建之初就是为纯文本通讯而设计的,它天然的通讯协议开销是无法通过技术手段消除的，HTTP2/HTTP3 都无法解决这个问题，虽然它们做了很多优化和压缩。  
gRPC 这个设计最初就是为服务之间的相互通讯设计的，它一开始就采用了和语言无关的二进制通讯模式，这使得它天然更加适合服务通讯，HTTP 协议提供的很多功能在服务相互调用时根本不需要。

1：protocol buffers 和任何基于文本的序列化性能差异是很大的，它是一个规范，只要最终大家都支持，那它就会成为 HTTP 那样的共识协议，这个共识在二进制序列化上要达成只要 google 这种全球化的公司才有可能完成。  
2：HTTP 2 通过新的链接复用，在传输上已经解决了低效问题，然后就是对基于文本头的魔值定义再次压缩的通讯的协议头大小，基本上在传输上效率大大提高了，但它的核心问题还是 body 部分需要一个统一的二进制规范。  
3:  最核心的是 HTTP 协议有很多功能是为服务器-浏览器模式定制的，这些功能在服务器-服务器的时候没啥作用，这部分功能在 gRPC 通讯层可以和直接屏蔽掉，不对最终消费 gRPC 的用户开放。
## 从一些功能方面来说
1. 基于 http2.0 比 1.1 传输高效（多路复用什么的）  
2. Keepalive 机制  
3. \*.proto 的文件定义比看文档方便  
4. 协议帧的二进制通讯，提高了 API 被爬的门槛


RPC ，client 像调用本地函数一样调用接口，语义更加明确，更加规范，无需关心连接、序列化等细节，这些都由 RPC 框架帮你做了  
  
「通用的 HTTP 库」，如果这个库把连接池、编码等等功能都实现了，调用方每个接口的调用无须做重复性劳动，那么这个「通用的 HTTP 库」何尝不是一个 RPC
rpc 是一种编程概念，就是远程调用。  
http 只是一种实现而已。你单独 socket 实现也行。  
http 本身不是为了 rpc 出生。但是适合绝大部分情况了。

1. 基于 HTTP2 组帧压缩、TCP 连接多路复用等特性，提供了低延时高吞吐  
2. Protobuf 序列化的消息始终小于等效的 JSON 消息  
3. 出色的对双向流式传输可以实时推送接收消息，无需轮询  
4. 多语言支持和代码生成，节约大量开发时间  
5. http api 没有正式规范，站内的的争论也很多，gRpc 规范消除了争论，节省了开发时间

成熟的 rpc 库相对 http 容器，更多的是封装了“服务发现”，"负载均衡"，“熔断降级”一类面向服务的高级特性。  
rpc 框架是面向服务的更高级的封装，针对服务的可用性和效率等都做了优化。单纯使用 http 调用缺少了这些特性。  
而且使用类似 grpc gateway 类似插件也可以提供外部 http 服务，也很方便。

# 一些不保证正确的观念
能问出这个问题，说明你至少现阶段完全不需要 rpc 。你这个比较严格来说是不成立的，http 也可以是 rpc 的一种通信方式，当然我理解你问的肯定是 http + json 和狭义的 rpc ，也就是 grpc/thrift 等的区别。  
  
gRPC 无非就是 protobuf3 + http2 之上做了一些优化，而这些优化几乎只有在大厂才能体现出价值。比如说，大厂内部团队之间要撕逼，没有个 pb 定义的结构，到时候可有得扯皮。虽然 http+json 也有 swagger 和 jsonschema 等工具，但是远不如 pb 或者 thrift 类型丰富且成熟。在大厂里，序列化都可能会成为一个性能瓶颈，我记得之前公司从 thrift 转 pb ，还是 pb 转 thrift 来着，折腾了好久，性能也有很大提升。在这方面，json 序列化的性能就根本不在考虑范围内了。  
  
对于小厂来说，真没必要上什么 rpc ，http + json 撑到上市都没问题。重要的是产品，而不是炫技。rpc 解决了大厂的问题，但是也是有代价的。举例来说，rpc 所谓的跨语言几乎都只是理论上的跨语言而已，gRPC 的 python 支持就没那么好，经常会和 multiprocessing 打架，大厂自然有框架组专门的人来解决，至少规避这个问题，小公司有吗？  
  
最后，技术选型选的不只是某一个技术，更是一个公司的人员组织结构选型。小公司没那么多人，就别给自己找点没用的事儿干，除非你是面向简历编程。

总结一下上面的话，也就是pb可以避免多团队撕逼。不需要对yapi，只要拿过去pb直接gen就行。性能什么的在我们这种级别没有那么悬殊，都差不多。

# gRPC 客户端长连接机制实现及 keepalive 分析

## 如何实现针对 gRPC 客户端的自动重连机制

## 0x00 前言 

`HTTP2` 是一个全双工的流式协议, 服务端也可以主动 `ping` 客户端, 且服务端还会有一些检测连接可用性和控制客户端 `ping` 包频率的配置。gRPC 就是采用 `HTTP2` 来作为其基础通信模式的，所以默认的 gRPC 客户端都是长连接。  

有这么一种场景，需要客户端和服务端保持持久的长连接，即无论服务端、客户端异常断开或重启，长连接都要具备重试保活（当然前提是两方重启都成功）的需求。在 gRPC 中，对于已经建立的长连接，服务端异常重启之后，客户端一般会收到如下错误：  

> rpc error: code = Unavailable desc = transport is closing

大部分的 gRPC 客户端封装都没有很好的处理这类 case，参见 [Warden 关于 Server 端服务重启后 Client 连接断开之后的重试问题](https://github.com/go-kratos/kratos/issues/177)，对于这种错误，推荐有两种处理方法：  

1. 重试：在客户端调用失败时，选择以指数退避（Exponential Backoff ）来优雅进行重试
2. 增加 keepalive 的保活策略
3. 增加重连（auto reconnect）策略

这篇文章就来分析下如何实现这样的客户端保活（keepalive）逻辑。提到保活机制，我们先看下 gRPC 的 [keepalive 机制](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)。

## 0x01 HTTP2 的 GOAWAY 帧[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#0x01-http2-%E7%9A%84-goaway-%E5%B8%A7)

`HTTP2` 使用 GOAWAY 帧信号来控制连接关闭，GOAWAY 用于启动连接关闭或发出严重错误状态信号。  
GOAWAY 语义为允许端点正常停止接受新的流，同时仍然完成对先前建立的流的处理，当 client 收到这个包之后就会主动关闭连接。下次需要发送数据时，就会重新建立连接。GOAWAY 是实现 `grpc.gracefulStop` 机制的重要保证。

## 0x02 gRPC 客户端 keepalive[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#0x02----grpc-%E5%AE%A2%E6%88%B7%E7%AB%AF-keepalive)

gRPC 客户端提供 keepalive 配置如下：

```
var kacp = keepalive.ClientParameters{
	Time:                10 * time.Second, // send pings every 10 seconds if there is no activity
	Timeout:             time.Second,      // wait 1 second for ping ack before considering the connection dead
	PermitWithoutStream: true,             // send pings even without active streams
}
//Dial 中传入 keepalive 配置
conn, err := grpc.Dial(*addr, grpc.WithInsecure(), grpc.WithKeepaliveParams(kacp))
```

`keepalive.ClientParameters` 参数的含义如下:

- `Time`：如果没有 activity， 则每隔 `10s` 发送一个 ping 包
- `Timeout`： 如果 ping ack 1s 之内未返回则认为连接已断开
- `PermitWithoutStream`：如果没有 active 的 stream， 是否允许发送 ping

联想到，在项目中 [`ssh` 客户端](https://pandaychen.github.io/2019/10/20/HOW-TO-BUILD-A-SSHD-WITH-GOLANG/#%E5%AE%A2%E6%88%B7%E7%AB%AF-keepalive-%E6%9C%BA%E5%88%B6) 和 `mysql` 客户端中都有着类似的实现，即单独开启协程来实现 keepalive： 如下面的代码（以 `ssh` 为例）：

```
go func() {
    t := time.NewTicker(2 * time.Second)
    defer t.Stop()
    for range t.C {
        _, _, err := client.Conn.SendRequest("keepalive@golang.org", true, nil)
        if err != nil {
            return
        }
    }
}()
```

#### gPRC 的实现[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#gprc-%E7%9A%84%E5%AE%9E%E7%8E%B0)

在 grpc-go 的 [newHTTP2Client](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_client.go#L166) 方法中，有下面的逻辑：  
即在新建一个 `HTTP2Client` 的时候会启动一个 `goroutine` 来处理 keepalive

```
// newHTTP2Client constructs a connected ClientTransport to addr based on HTTP2
// and starts to receive messages on it. Non-nil error returns if construction
// fails.
func newHTTP2Client(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (_ *http2Client, err error) {
    ...
	if t.keepaliveEnabled {
		t.kpDormancyCond = sync.NewCond(&t.mu)
		go t.keepalive()
    }
    ...
}
```

接下来，看下 [`keepalive` 方法](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_client.go#L1350) 的实现：

```
func (t *http2Client) keepalive() {
	p := &ping{data: [8]byte{}} //ping 的内容
	timer := time.NewTimer(t.kp.Time) // 启动一个定时器, 触发时间为配置的 Time 值
	//for loop
	for {
		select {
		// 定时器触发
		case <-timer.C:
			if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
				timer.Reset(t.kp.Time)
				continue
			}
			// Check if keepalive should go dormant.
			t.mu.Lock()
			if len(t.activeStreams) < 1 && !t.kp.PermitWithoutStream {
				// Make awakenKeepalive writable.
				<-t.awakenKeepalive
				t.mu.Unlock()
				select {
				case <-t.awakenKeepalive:
					// If the control gets here a ping has been sent
					// need to reset the timer with keepalive.Timeout.
				case <-t.ctx.Done():
					return
				}
			} else {
				t.mu.Unlock()
				if channelz.IsOn() {
					atomic.AddInt64(&t.czData.kpCount, 1)
				}
				// Send ping.
				t.controlBuf.put(p)
			}

			// By the time control gets here a ping has been sent one way or the other.
			timer.Reset(t.kp.Timeout)
			select {
			case <-timer.C:
				if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
					timer.Reset(t.kp.Time)
					continue
				}
				t.Close()
				return
			case <-t.ctx.Done():
				if !timer.Stop() {
					<-timer.C
				}
				return
			}
		// 上层通知 context 结束
		case <-t.ctx.Done():
			if !timer.Stop() {
				// 返回 false，表示 timer 未被销毁
				<-timer.C
			}
			return
		}
	}
}
```

从客户端的 `keepalive` 实现中梳理下执行逻辑：

1. 填充 `ping` 包内容, 为 `[8]byte{}`，创建定时器, 触发时间为用户配置中的 `Time`
2. 循环处理，select 的两大分支，一为定时器触发后执行的逻辑，另一分支为 `t.ctx.Done()`，即 `keepalive` 的上层应用调用了 `cancel` 结束 context 子树
3. 核心逻辑在定时器触发的过程中

## 0x03 gRPC 服务端的 keepalive[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#0x03----grpc-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%9A%84-keepalive)

gRPC 的服务端主要有两块逻辑：

1. 接收并相应客户端的 ping 包
2. 单独启动 goroutine 探测客户端是否存活

gRPC 服务端提供 keepalive 配置，分为两部分 `keepalive.EnforcementPolicy` 和 `keepalive.ServerParameters`，如下：

```
var kaep = keepalive.EnforcementPolicy{
	MinTime:             5 * time.Second, // If a client pings more than once every 5 seconds, terminate the connection
	PermitWithoutStream: true,            // Allow pings even when there are no active streams
}

var kasp = keepalive.ServerParameters{
	MaxConnectionIdle:     15 * time.Second, // If a client is idle for 15 seconds, send a GOAWAY
	MaxConnectionAge:      30 * time.Second, // If any connection is alive for more than 30 seconds, send a GOAWAY
	MaxConnectionAgeGrace: 5 * time.Second,  // Allow 5 seconds for pending RPCs to complete before forcibly closing connections
	Time:                  5 * time.Second,  // Ping the client if it is idle for 5 seconds to ensure the connection is still active
	Timeout:               1 * time.Second,  // Wait 1 second for the ping ack before assuming the connection is dead
}

func main(){
	...
	s := grpc.NewServer(grpc.KeepaliveEnforcementPolicy(kaep), grpc.KeepaliveParams(kasp))
	...
}
```

`keepalive.EnforcementPolicy`：

- `MinTime`：如果客户端两次 ping 的间隔小于 `5s`，则关闭连接
- `PermitWithoutStream`： 即使没有 active stream, 也允许 ping

`keepalive.ServerParameters`：

- `MaxConnectionIdle`：如果一个 client 空闲超过 `15s`, 发送一个 GOAWAY, 为了防止同一时间发送大量 GOAWAY, 会在 `15s` 时间间隔上下浮动 `15*10%`, 即 `15+1.5` 或者 `15-1.5`
- `MaxConnectionAge`：如果任意连接存活时间超过 `30s`, 发送一个 GOAWAY
- `MaxConnectionAgeGrace`：在强制关闭连接之间, 允许有 `5s` 的时间完成 pending 的 rpc 请求
- `Time`： 如果一个 client 空闲超过 `5s`, 则发送一个 ping 请求
- `Timeout`： 如果 ping 请求 `1s` 内未收到回复, 则认为该连接已断开

#### gRPC 的实现[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#grpc-%E7%9A%84%E5%AE%9E%E7%8E%B0)

服务端处理客户端的 `ping` 包的 response 的逻辑在 [`handlePing` 方法](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_server.go#L693) 中。  
`handlePing` 方法会判断是否违反两条 policy, 如果违反则将 `pingStrikes++`, 当违反次数大于 `maxPingStrikes(2)` 时, 打印一条错误日志并且发送一个 goAway 包，断开这个连接，具体实现如下：

```
func (t *http2Server) handlePing(f *http2.PingFrame) {
	if f.IsAck() {
		if f.Data == goAwayPing.data && t.drainChan != nil {
			close(t.drainChan)
			return
		}
		// Maybe it's a BDP ping.
		if t.bdpEst != nil {
			t.bdpEst.calculate(f.Data)
		}
		return
	}
	pingAck := &ping{ack: true}
	copy(pingAck.data[:], f.Data[:])
	t.controlBuf.put(pingAck)

	now := time.Now()
	defer func() {
		t.lastPingAt = now
	}()
	// A reset ping strikes means that we don't need to check for policy
	// violation for this ping and the pingStrikes counter should be set
	// to 0.
	if atomic.CompareAndSwapUint32(&t.resetPingStrikes, 1, 0) {
		t.pingStrikes = 0
		return
	}
	t.mu.Lock()
	ns := len(t.activeStreams)
	t.mu.Unlock()
	if ns < 1 && !t.kep.PermitWithoutStream {
		// Keepalive shouldn't be active thus, this new ping should
		// have come after at least defaultPingTimeout.
		if t.lastPingAt.Add(defaultPingTimeout).After(now) {
			t.pingStrikes++
		}
	} else {
		// Check if keepalive policy is respected.
		if t.lastPingAt.Add(t.kep.MinTime).After(now) {
			t.pingStrikes++
		}
	}

	if t.pingStrikes > maxPingStrikes {
		// Send goaway and close the connection.
		if logger.V(logLevel) {
			logger.Errorf("transport: Got too many pings from the client, closing the connection.")
		}
		t.controlBuf.put(&goAway{code: http2.ErrCodeEnhanceYourCalm, debugData: []byte("too_many_pings"), closeConn: true})
	}
}

```

注意，对 `pingStrikes` 累加的逻辑：

- `t.lastPingAt.Add(defaultPingTimeout).After(now)`：
- `t.lastPingAt.Add(t.kep.MinTime).After(now)`：

```
func (t *http2Server) handlePing(f *http2.PingFrame) {
	...
	if ns < 1 && !t.kep.PermitWithoutStream {
		// Keepalive shouldn't be active thus, this new ping should
		// have come after at least defaultPingTimeout.
		if t.lastPingAt.Add(defaultPingTimeout).After(now) {
			t.pingStrikes++
		}
	} else {
		// Check if keepalive policy is respected.
		if t.lastPingAt.Add(t.kep.MinTime).After(now) {
			t.pingStrikes++
		}
	}
	if t.pingStrikes > maxPingStrikes {
		// Send goaway and close the connection.
		errorf("transport: Got too many pings from the client, closing the connection.")
		t.controlBuf.put(&goAway{code: http2.ErrCodeEnhanceYourCalm, debugData: []byte("too_many_pings"), closeConn: true})
	}
}
```

#### keepalive 相关代码[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#keepalive-%E7%9B%B8%E5%85%B3%E4%BB%A3%E7%A0%81)

gRPC 服务端新建一个 HTTP2 server 的时候会启动一个单独的 goroutine 处理 keepalive 逻辑，[`newHTTP2Server` 方法](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_server.go#L129)：

```
func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
	...
	go t.keepalive()
	...
}
```

简单分析下 `keepalive` 的实现，核心逻辑是启动 `3` 个定时器，分别为 `maxIdle`、`maxAge` 和 `keepAlive`，然后在 `for select` 中处理相关定时器触发事件：

- `maxIdle` 逻辑： 判断 client 空闲时间是否超出配置的时间, 如果超时, 则调用 `t.drain`, 该方法会发送一个 GOAWAY 包
- `maxAge` 逻辑： 触发之后首先调用 `t.drain` 发送 GOAWAY 包, 接着重置定时器, 时间设置为 `MaxConnectionAgeGrace`, 再次触发后调用 `t.Close()` 直接关闭（有些 graceful 的意味）
- `keepalive` 逻辑： 首先判断 activity 是否为 `1`, 如果不是则置 `pingSent` 为 `true`, 并且发送 ping 包, 接着重置定时器时间为 `Timeout`, 再次触发后如果 activity 不为 1（即未收到 ping 的回复） 并且 `pingSent` 为 `true`, 则调用 `t.Close()` 关闭连接

```go
func (t *http2Server) keepalive() {
	p := &ping{}
	var pingSent bool
	maxIdle := time.NewTimer(t.kp.MaxConnectionIdle)
	maxAge := time.NewTimer(t.kp.MaxConnectionAge)
	keepalive := time.NewTimer(t.kp.Time)
	// NOTE: All exit paths of this function should reset their
	// respective timers. A failure to do so will cause the
	// following clean-up to deadlock and eventually leak.
	defer func() {
		// 退出前，完成定时器的回收工作
		if !maxIdle.Stop() {
			<-maxIdle.C
		}
		if !maxAge.Stop() {
			<-maxAge.C
		}
		if !keepalive.Stop() {
			<-keepalive.C
		}
	}()
	for {
		select {
		case <-maxIdle.C:
			t.mu.Lock()
			idle := t.idle
			if idle.IsZero() { // The connection is non-idle.
				t.mu.Unlock()
				maxIdle.Reset(t.kp.MaxConnectionIdle)
				continue
			}
			val := t.kp.MaxConnectionIdle - time.Since(idle)
			t.mu.Unlock()
			if val <= 0 {
				// The connection has been idle for a duration of keepalive.MaxConnectionIdle or more.
				// Gracefully close the connection.
				t.drain(http2.ErrCodeNo, []byte{})
				// Resetting the timer so that the clean-up doesn't deadlock.
				maxIdle.Reset(infinity)
				return
			}
			maxIdle.Reset(val)
		case <-maxAge.C:
			t.drain(http2.ErrCodeNo, []byte{})
			maxAge.Reset(t.kp.MaxConnectionAgeGrace)
			select {
			case <-maxAge.C:
				// Close the connection after grace period.
				t.Close()
				// Resetting the timer so that the clean-up doesn't deadlock.
				maxAge.Reset(infinity)
			case <-t.ctx.Done():
			}
			return
		case <-keepalive.C:
			if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
				pingSent = false
				keepalive.Reset(t.kp.Time)
				continue
			}
			if pingSent {
				t.Close()
				// Resetting the timer so that the clean-up doesn't deadlock.
				keepalive.Reset(infinity)
				return
			}
			pingSent = true
			if channelz.IsOn() {
				atomic.AddInt64(&t.czData.kpCount, 1)
			}
			t.controlBuf.put(p)
			keepalive.Reset(t.kp.Timeout)
		case <-t.ctx.Done():
			return
		}
	}
}
```

## 0x04 实现健壮的长连接客户端[](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/#0x04----%E5%AE%9E%E7%8E%B0%E5%81%A5%E5%A3%AE%E7%9A%84%E9%95%BF%E8%BF%9E%E6%8E%A5%E5%AE%A2%E6%88%B7%E7%AB%AF)

官方提供了 keepalive 的实例：

- [服务端](https://github.com/grpc/grpc-go/blob/master/examples/features/keepalive/server/main.go)
- [客户端](https://github.com/grpc/grpc-go/blob/master/examples/features/keepalive/client/main.go)





# Reference
https://pandaychen.github.io/2019/10/20/K8S-HEADLESS-SVC/
https://pandaychen.github.io/2020/06/01/K8S-LOADBALANCE-WITH-KUBERESOLVER/
https://pandaychen.github.io/2020/10/13/K8S-LOAD-BALANCER-KUBERESOLVER-ANALYSIS/
https://pandaychen.github.io/2020/10/03/DO-WE-NEED-GRPC-CLIENT-POOL/
https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/




