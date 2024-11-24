#网络 #http2

超文本传输协议(HTTP)是一种非常成功的协议。 但是，HTTP/1.1 使用底层传输的方式([[RFC7230]，第 6 节](https://tools.ietf.org/html/rfc7230#section-6))，其中有几个特性对今天的应用程序性能有负面影响。

特别是，HTTP/1.0 在给定的 TCP 连接上一次只允许一个请求未完成。HTTP/1.1 添加了请求流水线操作(request pipelining)，但这只是部分地解决了请求并发性，并且仍然受到**队首阻塞**的影响。因此，需要发出许多请求的 HTTP/1.0 和 HTTP/1.1 客户端使用多个连接到服务器以实现并发，从而减少延迟。

![](https://img.halfrost.com/Blog/ArticleImage/124_4.jpg)

此外，HTTP 头字段通常是重复且冗长的，导致不必要的网络流量以及导致初始 [TCP](https://tools.ietf.org/html/rfc7540#ref-TCP) 拥塞窗口被快速的填满。当在新的 TCP 连接上发出多个请求时，这可能导致过多的延迟。

HTTP/2 通过定义了一个优化过的 HTTP 语义，它与底层连接映射，用这种方式来解决这些问题。具体而言，它允许在同一连接上交错请求和响应消息，并使用 HTTP 头字段的有效编码。它还允许对请求进行优先级排序，使更多重要请求更快地完成，从而进一步提高性能。

HTTP/2 对网络更友好，因为与 HTTP/1.x 相比，可以使用更少的 TCP 连接。这意味着与其他流量和长连接的竞争减少，反过来可以更好地利用可用网络容量。最后，HTTP/2 还可以通过使用二进制消息帧来更有效地处理消息。

> HTTP/2 最大限度的兼容 HTTP/1.1 原有行为：
> 
> 1.  在应用层上修改，基于并充分挖掘 TCP 协议性能。
> 2.  客户端向服务端发送 request 请求的模型没有变化。
> 3.  scheme 没有发生变化，没有 http2://
> 4.  使用 HTTP/1.X 的客户端和服务器可以无缝的通过代理方式转接到 HTTP/2 上。
> 5.  不识别 HTTP/2 的代理服务器可以将请求降级到 HTTP/1.X。

## 一. HTTP/2 Protocol Overview

![](https://img.halfrost.com/Blog/ArticleImage/129_5.png)

HTTP/2 为 HTTP 语义提供了优化的传输。 HTTP/2 支持 HTTP/1.1 的所有核心功能，但旨在通过多种方式提高效率。

HTTP/2 中的基本协议单元是一个帧([第 4.1 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E4%B8%80-frame-format-%E5%B8%A7%E6%A0%BC%E5%BC%8F))。每种帧类型都有不同的用途。例如，HEADERS 和 DATA 帧构成了 HTTP 请求和响应的基础([第 8.1 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Semantics.md#%E4%B8%80-http-requestresponse-exchange))；其他帧类型(如 SETTINGS，WINDOW_UPDATE 和 PUSH_PROMISE)用于支持其他 HTTP/2 功能。

> HTTP/2 是一个彻彻底底的二进制协议，头信息和数据包体都是二进制的，统称为“帧”。对比 HTTP/1.1 ，在 HTTP/1.1 中，头信息是文本编码(ASCII编码)，数据包体可以是二进制也可以是文本。使用二进制作为协议实现方式的好处，更加灵活。在 HTTP/2 中定义了 10 种不同类型的帧。

通过使每个 HTTP 请求/响应交换与其自己的 stream 流相关联来实现请求的多路复用([第 5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E5%9B%9B-stream-%E6%B5%81%E7%8A%B6%E6%80%81%E6%9C%BA))。stream 流在很大程度上是彼此独立的，因此阻塞或停止的请求或响应不会阻止其他 stream 流的通信。

> 由于 HTTP/2 的数据包是乱序发送的，因此在同一个连接里会收到不同请求的 response。不同的数据包携带了不同的标记，用来标识它属于哪个 response。
> 
> HTTP/2 把每个 request 和 response 的数据包称为一个数据流(stream)。每个数据流都有自己全局唯一的编号。每个数据包在传输过程中都需要标记它属于哪个数据流 ID。规定，客户端发出的数据流，ID 一律为奇数，服务器发出的，ID 为偶数。
> 
> 数据流在发送中的任意时刻，客户端和服务器都可以发送信号(RST_STREAM 帧)，取消这个数据流。HTTP/1.1 中想要取消数据流的唯一方法，就是关闭 TCP 连接。而 HTTP/2 可以取消某一次请求，同时保证 TCP 连接还打开着，可以被其他请求使用。

流量控制和优先级确保可以有效地使用多路复用流。流量控制([第 5.2 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E4%BA%94-%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6))有助于确保只传输接收者可以使用的数据。确定优先级([第 5.3 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E5%85%AD-stream-%E4%BC%98%E5%85%88%E7%BA%A7))可确保首先将有限的资源定向到最重要的流。

HTTP/2 添加了一种新的交互模式，服务器可以将响应推送到客户端([第 8.2 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Semantics.md#%E4%BA%8C-server-push))。服务器推送允许服务器推测性地将数据发送到服务器预测客户端将需要这些数据的客户端，通过牺牲一些网络流量来抵消潜在的延迟。服务器通过合成请求来完成此操作，并将其作为 PUSH_PROMISE 帧发送。然后，服务器能够在单独的流上发送对合成请求的响应。

由于连接中使用的 HTTP 头字段可能包含大量冗余数据，因此压缩包含它们的帧([第 4.3 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E4%B8%89-header-compression-and-decompression))。允许将许多请求压缩成一个分组的做法对于通常情况下的请求大小具有特别有利的影响。

> HTTP 协议不带有状态，每次请求都必须附上所有信息。所以，请求的很多字段都是重复的，比如 Cookie 和 User Agent，每次请求即使是完全一样的内容，依旧必须每次都携带，这会浪费很多带宽，也影响速度。HTTP/1.1 虽然可以压缩请求体，但是不能压缩消息头。有时候消息头部很大。
> 
> HTTP/2 对这一点做了优化，引入了头信息压缩机制(header compression)。一方面，头信息使用 gzip 或 compress 压缩后再发送；另一方面，客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就提高速度了。
> 
> 头部压缩大概可能有 95% 左右的提升，HTTP/1.1 统计的平均响应头大小有 500 个字节左右，而 HTTP/2 的平均响应头大小只有 20 多个字节，提升比较大。

![](https://img.halfrost.com/Blog/ArticleImage/129_1.png)

接下来分 4 部分详细讨论 HTTP/2。

-   解开 HTTP/2 的面纱：HTTP/2 是如何建立连接的([第三章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#1-http2-version-identification))
-   帧([第四章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E4%B8%80-frame-format-%E5%B8%A7%E6%A0%BC%E5%BC%8F))和流([第五章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E5%9B%9B-stream-%E6%B5%81%E7%8A%B6%E6%80%81%E6%9C%BA))层描述了 HTTP/2 帧的结构和形成多路复用流的方式。
-   帧([第六章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E4%B8%80-data-%E5%B8%A7))和错误([第七章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E5%8D%81%E4%B8%80-error-codes))定义了包括 HTTP/2 中使用的帧和错误类型的详细信息。
-   HTTP 映射([第八章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Semantics.md#%E4%B8%80-http-requestresponse-exchange))和附加要求([第九章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-Considerations.md#1-%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86))描述了如何使用帧和流表示 HTTP 语义。

虽然一些帧层和流层概念与 HTTP 隔离，但是该规范没有定义完全通用的帧层。帧层和流层是根据 HTTP 协议和服务器推送的需要而定制的。

## 二. Starting HTTP/2

HTTP/2 连接是在 TCP 连接([TCP](https://tools.ietf.org/html/rfc7540#ref-TCP))之上运行的应用层协议。客户端是 TCP 连接发起者。

HTTP/2 使用 HTTP/1.1 使用的相同 "http" 和 "https" URI scheme。HTTP/2 共享相同的默认端口号: "http" URI 为 80，"https" URI 为 443。因此，需要处理对目标资源 URI (例如 "[http://example.org/foo](http://example.org/foo)" 或 "[https://example.com/bar](https://example.com/bar)")的请求的实现，首先需要发现上游服务器(客户端希望建立连接的直接对等方)是否支持 HTTP/2。

对于 "http" 和 "https" URI，确定支持 HTTP/2 的方式是不同的。"http" URI 的发现在 [3.2 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#2-starting-http2-for-http-uris)中描述。[第 3.3 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#4-starting-http2-for-https-uris)描述了 "https" URI 的发现。

### 1. HTTP/2 Version Identification

本文档中定义的协议有两个标识符。

-   字符串 "h2" 标识 HTTP/2 使用传输层安全性(TLS)[TLS12](https://tools.ietf.org/html/rfc7540#ref-TLS12)的协议。该标识符用于 TLS 应用层协议协商(ALPN)扩展[TLS-ALPN](https://tools.ietf.org/html/rfc7540#ref-TLS-ALPN)字段以及识别 HTTP/2 over TLS 的任何地方。

"h2"字符串被序列化为 ALPN 协议标识符，作为两个八位字节序列：0x68,0x32。

-   字符串 "h2c" 标识通过明文 TCP 运行 HTTP/2 的协议。此标识符用于 HTTP/1.1 升级标头字段以及标识 HTTP/2 over TCP 的任何位置。

"h2c" 字符串是从 ALPN 标识符空间保留的，但描述了不使用 TLS 的协议。协商 "h2" 或 "h2c" 意味着使用本文档中描述的传输，安全性，成帧和消息语义。

### 2. Starting HTTP/2 for "http" URIs

在没有关于下一跳支持 HTTP/2 的 prior knowledge 的情况下请求 "http" URI 的客户端使用 HTTP 升级机制([[RFC7230]的第 6.7 节](https://tools.ietf.org/html/rfc7230#section-6.7))。客户端通过发出包含带有 "h2c" 标记的 Upgrade 头字段的HTTP/1.1 请求来完成此操作。这样的 HTTP/1.1 请求必须包含一个 HTTP2-Settings([第 3.2.1 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#3-http2-settings-header-field))头字段。

例如：

C

```c
     GET / HTTP/1.1
     Host: server.example.com
     Connection: Upgrade, HTTP2-Settings
     Upgrade: h2c
     HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```

在客户端可以发送 HTTP/2 帧之前，必须完整地发送包含有效负载主体的请求。这意味着大型请求可以阻止连接的使用，直到完全发送为止。

如果初始请求与后续请求的并发性很重要，则可以使用 OPTIONS 请求执行升级到 HTTP/2，但需要额外的往返。不支持 HTTP/2 的服务器可以响应请求，就像没有 Upgrade 头字段一样：

C

```c
     HTTP/1.1 200 OK
     Content-Length: 243
     Content-Type: text/html

     ...
```

服务器必须忽略 Upgrade 头字段中的 "h2" 标记。具有 "h2" 的令牌的存在意味着 HTTP/2 over TLS，这种方式替代 [3.3 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#4-starting-http2-for-https-uris)中所述协商过程。

支持 HTTP/2 的服务器通过 101(交换协议)响应接受升级。在响应 101 末尾的空行之后，服务器可以开始发送 HTTP/2 帧。这些帧必须包括对启动升级的请求的响应。

例如：

C

```c
     HTTP/1.1 101 Switching Protocols
     Connection: Upgrade
     Upgrade: h2c

     [ HTTP/2 connection ...
```

服务器发送的第一个 HTTP/2 帧必须是由 SETTINGS 帧([第 6.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E4%BA%94-settings-%E5%B8%A7))组成的服务器连接前奏([第 3.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#6-http2-connection-preface))。收到 101 响应后，客户端必须发送连接前奏([第3.5节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#6-http2-connection-preface))，其中包括 SETTINGS 帧。

在升级之前发送的 HTTP/1.1 请求被赋予 stream 流标识符 1 (参见[第 5.1.1 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#1-stream-%E6%A0%87%E8%AF%86%E7%AC%A6))，它是默认优先级值([第 5.3.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#5-%E9%BB%98%E8%AE%A4%E4%BC%98%E5%85%88%E7%BA%A7))。Stream 流 1 从客户端隐式"半封闭"的流向服务器(参见[第5.1节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#%E5%9B%9B-stream-%E6%B5%81%E7%8A%B6%E6%80%81%E6%9C%BA))，因为请求是作为 HTTP/1.1 请求完成的。在开始 HTTP/2 连接之后，stream 流 1 用于响应。

### 3. HTTP2-Settings Header Field

从 HTTP/1.1 升级到 HTTP/2 的请求必须包含一个 "HTTP2-Settings" 头字段。HTTP2-Settings 标头字段是一个特定于连接的 header 字段，其中包含管理 HTTP/2 连接的参数，这个参数在服务器接受升级请求的情况下提供的。

C

```c
     HTTP2-Settings    = token68
```

如果此 header 字段不存在或存在多个连接，则服务器不得升级到 HTTP/2 的连接。服务器不得发送此 header 字段。

HTTP2-Settings 头字段的内容是 SETTINGS 帧的有效负载([第 6.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E4%BA%94-settings-%E5%B8%A7))，编码为 base64url 字符串(即[[RFC4648]第 5 节](https://tools.ietf.org/html/rfc4648#section-5)中描述的 URL 和文件名安全的 Base64 编码，省略任何尾随的 '=' 字符)。ABNF [RFC5234](https://tools.ietf.org/html/rfc5234) 生成 "token68" 在 [[RFC7235]的第 2.1 节](https://tools.ietf.org/html/rfc7235#section-2.1)中定义。

由于升级仅用于立即连接，因此发送 HTTP2-Settings header 字段的客户端也必须在 Connection 头字段中发送 "HTTP2-Settings" 作为连接选项，以防止它被转发(参见[[RFC7230]中的第 6.1 节](https://tools.ietf.org/html/rfc7230#section-6.1)）。

服务器解码并解释这些值，就像任何其他 SETTINGS 帧一样。不必明确确认这些设置([第 6.5.3 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#3-settings-synchronization)），因为 101 响应用作隐式确认。在升级请求中提供这些值，目的的为了使客户端有机会在从服务器接收任何帧之前提供参数。

### 4. Starting HTTP/2 for "https" URIs

向 "https" URI发出请求的客户端使用 TLS [TLS12](https://tools.ietf.org/html/rfc7540#ref-TLS12) 和应用层协议协商(ALPN)扩展 [TLS-ALPN](https://tools.ietf.org/html/rfc7540#ref-TLS-ALPN)。

HTTP/2 over TLS 使用 "h2" 协议标识符。"h2c" 协议标识符不得由客户端发送或由服务器选择; "h2c" 协议标识符描述了一个不使用 TLS 的协议。

一旦 TLS 协商完成，客户端和服务器都必须发送连接前奏([第 3.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#6-http2-connection-preface))。

### 5. Starting HTTP/2 with Prior Knowledge

客户端可以通过其他方式了解特定服务器是否支持 HTTP/2。例如，[ALT-SVC](https://tools.ietf.org/html/rfc7540#ref-ALT-SVC) 描述了一种可以获得服务器是否支持 HTTP/2 的机制。

客户端必须发送连接前奏([第 3.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#6-http2-connection-preface))，然后可以立即将 HTTP/2 帧发送到服务器; 服务器可以通过连接前奏的存在来识别这些连接。这只影响通过明文 TCP 建立 HTTP/2 连接; 通过 TLS 支持 HTTP/2 的实现必须在 TLS [TLS-ALPN](https://tools.ietf.org/html/rfc7540#ref-TLS-ALPN) 中使用协议协商。同样，服务器必须发送连接前奏([第3.5节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-begin.md#6-http2-connection-preface))。

如果没有其他信息，先前对 HTTP/2 的支持并不是一个强信号，即给定服务器将支持 HTTP/2 以用于将来的连接。例如，可以更改服务器配置，使群集服务器中的实例之间的配置不同，或者更改网络条件。

### 6. HTTP/2 Connection Preface

> "连接前奏" 有些地方也会翻译成 "连接序言"。

在 HTTP/2 中，每个端点都需要发送连接前奏作为正在使用的协议的最终确认，并建立 HTTP/2 连接的初始设置。客户端和服务器各自发送不同的连接前奏。

客户端连接前奏以 24 个八位字节的序列开始，以十六进制表示法为：

C

```c
     0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a
```

也就是说，连接前奏以字符串 "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n" 开头。该序列必须后跟 SETTINGS 帧([第 6.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E4%BA%94-settings-%E5%B8%A7))，该帧可以为空。客户端在收到 101 (交换协议)响应(指示成功升级)或作为 TLS 连接的第一个应用程序数据八位字节后立即发送客户端连接前奏。如果启动具有服务器对协议支持的 prior knowledge 的 HTTP/2 连接，则在建立连接时发送客户端连接前奏。

> 注意：选择客户端连接前奏，以便大部分 HTTP/1.1 或 HTTP/1.0 服务器和中间件不会尝试处理更多帧。请注意，这并未解决 [TALKING](https://tools.ietf.org/html/rfc7540#ref-TALKING) 中提出的问题。

> 连接前奏里面的字符串连起来是 PRISM ，这个单词的意思是“棱镜”，就是 2013 年斯诺登爆出的“棱镜计划”。

服务器连接前奏包含一个可能为空的 SETTINGS 帧([第 6.5 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E4%BA%94-settings-%E5%B8%A7))，该帧必须是服务器在 HTTP/2 连接中发送的第一帧。

作为连接前奏的一部分从对等端收到的 SETTINGS 帧，必须在发送连接前奏后得到确认(参见[6.5.3 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#3-settings-synchronization))。

为避免不必要的延迟，允许客户端在发送客户端连接前奏后立即向服务器发送其他帧，而无需等待接收服务器连接前奏。但是，需要注意的是，服务器连接前奏 SETTINGS 帧可能包含参数，这些参数是客户端希望与服务器通信时必须的参数。在接收到 SETTINGS 帧后，客户端应该遵守所建立的任何参数。在某些配置中，服务器可以在客户端发送附加帧之前发送 SETTINGS，从而提供避免此问题的机会。

客户端和服务器必须将无效的连接前奏视为 PROTOCOL_ERROR 类型的连接错误([第 5.4.1 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames.md#1-%E8%BF%9E%E6%8E%A5%E9%94%99%E8%AF%AF%E7%9A%84%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86))。在这种情况下可以省略 GOAWAY 帧([第 6.8 节](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#%E5%85%AB-goaway-%E5%B8%A7))，因为无效的连接前奏表明对等方没有使用 HTTP/2。

最后，我们抓包看一下 HTTP/2 over TLS 是如何建立连接的。当 TLS 握手结束以后(TLS 握手的流程这里暂时省略，想要了解的同学可以看这里的[系列文章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTPS-TLS1.2_handshake.md))，客户端和服务端已经通过 ALPN 协商出了接下来应用层使用 HTTP/2 协议进行通信，于是会见到类似如下的抓包图：

![](https://img.halfrost.com/Blog/ArticleImage/124_1.png)

可以看到在 TLS 1.3 Finished 消息之后，紧接着就是 HTTP/2 的连接序言，Magic 帧。

![](https://img.halfrost.com/Blog/ArticleImage/124_2_0.png)

客户端连接前奏以 24 个八位字节的序列开始，以十六进制表示法为：

C

```c
     0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a
```

连接前奏就是字符串 "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n" 开头。Magic 帧之后紧跟着 SETTINGS 帧。当服务端成功 ack 了这条消息，并且没有连接报错，那么 HTTP/2 协议就算连接建立完成了。

# 一文读懂 HTTP/2 特性
HTTP/2 是 HTTP 协议自 1999 年 HTTP 1.1 发布后的首个更新，主要基于 SPDY 协议。由互联网工程任务组（IETF）的 Hypertext Transfer Protocol Bis（httpbis）工作小组进行开发。该组织于2014年12月将HTTP/2标准提议递交至IESG进行讨论，于2015年2月17日被批准。HTTP/2标准于2015年5月以RFC 7540正式发表。

那 HTTP/2 到底有哪些具体变化呢？

  

## **二进制分帧**

先来理解几个概念：

**帧：**HTTP/2 数据通信的最小单位消息：指 HTTP/2 中逻辑上的 HTTP 消息。例如请求和响应等，消息由一个或多个帧组成。

**流：**存在于连接中的一个虚拟通道。流可以承载双向消息，每个流都有一个唯一的整数ID。

HTTP/2 采用二进制格式传输数据，而非 HTTP 1.x 的文本格式，二进制协议解析起来更高效。 HTTP / 1 的请求和响应报文，都是由起始行，首部和实体正文（可选）组成，各部分之间以文本换行符分隔。HTTP/2 将请求和响应数据分割为更小的帧，并且它们采用二进制编码。

**HTTP/2 中，同域名下所有通信都在单个连接上完成，该连接可以承载任意数量的双向数据流。**每个数据流都以消息的形式发送，而消息又由一个或多个帧组成。多个帧之间可以乱序发送，根据帧首部的流标识可以重新组装。

  

## **多路复用**

多路复用，代替原来的序列和阻塞机制。所有就是请求的都是通过一个 TCP连接并发完成。 HTTP 1.x 中，如果想并发多个请求，必须使用多个 TCP 链接，且浏览器为了控制资源，还会对单个域名有 6-8个的TCP链接请求限制，如下图，红色圈出来的请求就因域名链接数已超过限制，而被挂起等待了一段时间：

![](https://pic1.zhimg.com/80/v2-b66be1a979ff666463a71c0eb4160c1c_720w.jpg)

在 HTTP/2 中，有了二进制分帧之后，HTTP /2 不再依赖 TCP 链接去实现多流并行了，在 HTTP/2中：

-   同域名下所有通信都在单个连接上完成。
-   单个连接可以承载任意数量的双向数据流。
-   数据流以消息的形式发送，而消息又由一个或多个帧组成，多个帧之间可以乱序发送，因为根据帧首部的流标识可以重新组装。

这一特性，使性能有了极大提升：

-   **同个域名只需要占用一个 TCP 连接**，消除了因多个 TCP 连接而带来的延时和内存消耗。
-   单个连接上可以并行交错的请求和响应，之间互不干扰。
-   在HTTP/2中，每个请求都可以带一个31bit的优先值，0表示最高优先级， 数值越大优先级越低。有了这个优先值，客户端和服务器就可以在处理不同的流时采取不同的策略，以最优的方式发送流、消息和帧。

  

## **服务器推送**

服务端可以在发送页面HTML时主动推送其它资源，而不用等到浏览器解析到相应位置，发起请求再响应。例如服务端可以主动把JS和CSS文件推送给客户端，而不需要客户端解析HTML时再发送这些请求。

服务端可以主动推送，客户端也有权利选择是否接收。如果服务端推送的资源已经被浏览器缓存过，浏览器可以通过发送RST_STREAM帧来拒收。主动推送也遵守同源策略，服务器不会随便推送第三方资源给客户端。

**[了解更多 Server Push 特性](https://link.zhihu.com/?target=https%3A//tech.upyun.com/article/294/1.html%3Futm_source%3Dzhihu%26utm_medium%3Dreferral%26utm_campaign%3D26559480%26utm_term%3Dhttp2)**

## **头部压缩**

HTTP 1.1请求的大小变得越来越大，有时甚至会大于TCP窗口的初始大小，因为它们需要等待带着ACK的响应回来以后才能继续被发送。HTTP/2对消息头采用HPACK（专为http/2头部设计的压缩格式）进行压缩传输，能够节省消息头占用的网络的流量。而HTTP/1.x每次请求，都会携带大量冗余头信息，浪费了很多带宽资源。

HTTP每一次通信都会携带一组头部，用于描述这次通信的的资源、浏览器属性、cookie等，例如

![](https://pic2.zhimg.com/80/v2-282eb9ed0d8dfd42f730784367dcf43d_720w.jpg)

为了减少这块的资源消耗并提升性能， **HTTP/2对这些首部采取了压缩策略**：

-   HTTP/2在客户端和服务器端使用“首部表”来跟踪和存储之前发送的键－值对，对于相同的数据，不再通过每次请求和响应发送；
-   首部表在HTTP/2的连接存续期内始终存在，由客户端和服务器共同渐进地更新;
-   每个新的首部键－值对要么被追加到当前表的末尾，要么替换表中之前的值。

例如：下图中的两个请求， 请求一发送了所有的头部字段，第二个请求则只需要发送差异数据，这样可以减少冗余数据，降低开销。

![](https://pic3.zhimg.com/80/v2-1573194744d005dd110bbeac3a9b5246_720w.jpg)

我们来看一个实际的例子，下面是用WireShark抓取的访问google首页的包：

![](https://pic1.zhimg.com/80/v2-1b19808193ed42238410e88aedbd44fc_720w.jpg)

上图是是访问[https://www.google.com/](https://link.zhihu.com/?target=https%3A//www.google.com/)抓到的第一个请求的头部，可以看到头部的内容，总共占用了437 bytes，我们选中头部的cookie，可以看到cookie总共占用了118 bytes。接下来我们看看第二个请求的头部：

![](https://pic2.zhimg.com/80/v2-0bc35f311f6cbc0110d81726ac71f98d_720w.jpg)

从上图可以看到，得益于头部压缩，第二个请求中cookie只占用了1个字节，我们来看看变化了的Accept字段：

![](https://pic3.zhimg.com/80/v2-8b7ba79ecf5b5f8c971e461a593e3dc2_720w.jpg)

由于Accept字段与请求一中的内容不同，需要发送给服务器，所以占用了29 bytes。


# 深入理解http2.0协议，看这篇就够了！

**导读**

http2.0是一种安全高效的下一代http传输协议。安全是因为http2.0建立在https协议的基础上，高效是因为它是通过二进制分帧来进行数据传输。正因为这些特性，http2.0协议也在被越来越多的网站支持。据统计，截止至2018年8月，已经有27.9%的网站支持http2.0。

本文将从**概述、原理、实战及检测**等方面来详细介绍http2.0，希望能够加深你的理解。

**什么是http2.0协议？**

在http2.0官网①的描述是：

http/2 is a replacement for how http is expressed “on the wire.” It is not a ground-up rewrite of the protocol; http methods, status codes and semantics are the same, and it should be possible to use the same APIs as http/1.x (possibly with some small additions) to represent the protocol.

The focus of the protocol is on performance; specifically, end-user perceived latency, network and server resource usage. One major goal is to allow the use of a single connection from browsers to a Web site.

The basis of the work was SPDY, but http/2 has evolved to take the community’s input into account, incorporating several improvements in the process.

中文总结一下就是：

**●对1.x协议语意的完全兼容**

2.0协议是在1.x基础上的升级而不是重写，1.x协议的方法，状态及api在2.0协议里是一样的。

**●性能的大幅提升**

2.0协议重点是对终端用户的感知延迟、网络及服务器资源的使用等性能的优化。

**http2.0优化内容**

**01**

**二进制分帧（Binary Format）- http2.0的基石**

http2.0之所以能够突破http1.X标准的性能限制，改进传输性能，实现低延迟和高吞吐量，就是因为其新增了二进制分帧层。

帧(frame)包含部分：类型Type, 长度Length, 标记Flags, 流标识Stream和frame payload有效载荷。

消息(message)：一个完整的请求或者响应，比如请求、响应等，由一个或多个 Frame 组成。

流是连接中的一个虚拟信道，可以承载双向消息传输。每个流有唯一整数标识符。为了防止两端流ID冲突，客户端发起的流具有奇数ID，服务器端发起的流具有偶数ID。

流标识是描述二进制frame的格式，使得每个frame能够基于http2发送，与流标识联系的是一个流，每个流是一个逻辑联系，一个独立的双向的frame存在于客户端和服务器端之间的http2连接中。一个http2连接上可包含多个并发打开的流，这个并发流的数量能够由客户端设置。

在二进制分帧层上，http2.0会将所有传输信息分割为更小的消息和帧，并对它们采用二进制格式的编码将其封装，新增的二进制分帧层同时也能够保证http的各种动词，方法，首部都不受影响，兼容上一代http标准。其中，http1.X中的首部信息header封装到Headers帧中，而request body将被封装到Data帧中。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee1d0caab8~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

**02**

**多路复用 (Multiplexing) / 连接共享**

在http1.1中，浏览器客户端在同一时间，针对同一域名下的请求有一定数量的限制，超过限制数目的请求会被阻塞。这也是为何一些站点会有多个静态资源 CDN 域名的原因之一。

而http2.0中的多路复用优化了这一性能。多路复用允许同时通过单一的http/2 连接发起多重的请求-响应消息。有了新的分帧机制后，http/2 不再依赖多个TCP连接去实现多流并行了。每个数据流都拆分成很多互不依赖的帧，而这些帧可以交错（乱序发送），还可以分优先级，最后再在另一端把它们重新组合起来。

http 2.0 连接都是持久化的，而且客户端与服务器之间也只需要一个连接（每个域名一个连接）即可。http2连接可以承载数十或数百个流的复用，多路复用意味着来自很多流的数据包能够混合在一起通过同样连接传输。当到达终点时，再根据不同帧首部的流标识符重新连接将不同的数据流进行组装。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee1cebfa44~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

上图展示了一个连接上的多个传输数据流：客户端向服务端传输数据帧stream5，同时服务端向客户端乱序发送stream1和stream3。这次连接上有三个响应请求乱序并行交换。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee1e5c87aa~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

上图就是http1.X和http2.0在传输数据时的区别。以货物运输为例再现http1.1与http2.0的场景：

http1.1过程：货轮1从A地到B地去取货物，取到货物后，从B地返回，然后货轮2在A返回并卸下货物后才开始再从A地出发取货返回，如此有序往返。

http2.0过程：货轮1、2、3、4、5从A地无序全部出发，取货后返回，然后根据货轮号牌卸载对应货物。

显然，第二种方式运输货物多，河道的利用率高。

03

**头部压缩（Header Compression）**

http1.x的头带有大量信息，而且每次都要重复发送。http/2使用encoder来减少需要传输的header大小，通讯双方各自缓存一份头部字段表，既避免了重复header的传输，又减小了需要传输的大小。

对于相同的数据，不再通过每次请求和响应发送，通信期间几乎不会改变通用键-值对(用户代理、可接受的媒体类型，等等)只需发送一次。

事实上,如果请求中不包含首部(例如对同一资源的轮询请求)，那么，首部开销就是零字节，此时所有首部都自动使用之前请求发送的首部。

如果首部发生了变化，则只需将变化的部分加入到header帧中，改变的部分会加入到头部字段表中，首部表在 http 2.0 的连接存续期内始终存在，由客户端和服务器共同渐进地更新。

需要注意的是，http 2.0关注的是首部压缩，而我们常用的gzip等是报文内容（body）的压缩，二者不仅不冲突，且能够一起达到更好的压缩效果。

http/2使用的是专门为首部压缩而设计的HPACK②算法。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee3331569a~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

从上图可以看到http1.X不支持首部压缩，而http2.0的压缩算法效果最好，发送和接受的数据量都是最少的。

**04**

**压缩原理**

用header字段表里的索引代替实际的header。

http/2的HPACK算法使用一份索引表来定义常用的http Header，把常用的 http Header 存放在表里，请求的时候便只需要发送在表里的索引位置即可。

例如 :method=GET 使用索引值 2 表示，:path=/index.html 使用索引值 5 表示，如下图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee3d18e3b3~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

完整的列表参考：HPACK Static Table③ 。

只要给服务端发送一个 Frame，该 Frame 的 Payload 部分存储 0x8285，Frame 的 Type 设置为 Header 类型，便可表示这个 Frame 属于 http Header，请求的内容是：

```
1GET /index.html复制代码
```

为什么是 0x8285，而不是 0x0205？这是因为高位设置为 1 表示这个字节是一个完全索引值（key 和 value 都在索引中）。

类似的，通过高位的标志位可以区分出这个字节是属于一个完全索引值，还是仅索引了 key，还是 key和value 都没有索引(参见：HTTP/2首部压缩的OkHttp3实现④)。

因为索引表的大小的是有限的，它仅保存了一些常用的 http Header，同时每次请求还可以在表的末尾动态追加新的 http Header 缓存，动态部分称之为 Dynamic Table。Static Table 和 Dynamic Table 在一起组合成了索引表：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee45d07de6~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

HPACK 不仅仅通过索引键值对来降低数据量，同时还会将字符串进行霍夫曼编码来压缩字符串大小。

以常用的 User-Agent 为例，它在静态表中的索引值是 58，它的值是不存在表中的，因为它的值是多变的。第一次请求的时候它的 key 用 58 表示，表示这是一个 User-Agent ，它的值部分会进行霍夫曼编码（如果编码后的字符串变更长了，则不采用霍夫曼编码）。

服务端收到请求后，会将这个 User-Agent 添加到 Dynamic Table 缓存起来，分配一个新的索引值。客户端下一次请求时，假设上次请求User-Agent的在表中的索引位置是 62， 此时只需要发送 0xBE（同样的，高位置 1），便可以代表：User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/33.0.1750.146 Safari/537.36。

其过程如下图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee50805e63~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

最终，相同的 Header 只需要发送索引值，新的 Header 会重新加入 Dynamic Table。

**05**

**请求优先级（Request Priorities）**

把http消息分为很多独立帧之后，就可以通过优化这些帧的交错和传输顺序进一步优化性能。每个流都可以带有一个31比特的优先值：0 表示最高优先级；2的31次方-1 表示最低优先级。

服务器可以根据流的优先级，控制资源分配（CPU、内存、带宽），而在响应数据准备好之后，优先将最高优先级的帧发送给客户端。高优先级的流都应该优先发送，但又不会绝对的。绝对地准守，可能又会引入首队阻塞的问题：高优先级的请求慢导致阻塞其他资源交付。

分配处理资源和客户端与服务器间的带宽，不同优先级的混合也是必须的。客户端会指定哪个流是最重要的，有一些依赖参数，这样一个流可以依赖另外一个流。优先级别可以在运行时动态改变，当用户滚动页面时，可以告诉浏览器哪个图像是最重要的，你也可以在一组流中进行优先筛选，能够突然抓住重点流。

●优先级最高：主要的html

●优先级高：CSS文件

●优先级中：js文件

●优先级低：图片

**06**

**服务端推送（Server Push）**

服务器可以对一个客户端请求发送多个响应，服务器向客户端推送资源无需客户端明确地请求。并且，服务端推送能把客户端所需要的资源伴随着index.html一起发送到客户端，省去了客户端重复请求的步骤。

正因为没有发起请求，建立连接等操作，所以静态资源通过服务端推送的方式可以极大地提升速度。Server Push 让 http1.x 时代使用内嵌资源的优化手段变得没有意义；如果一个请求是由你的主页发起的，服务器很可能会响应主页内容、logo 以及样式表，因为它知道客户端会用到这些东西，这相当于在一个 HTML 文档内集合了所有的资源。

不过与之相比，服务器推送还有一个很大的优势：可以缓存！也让在遵循同源的情况下，不同页面之间可以共享缓存资源成为可能。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/31/16e208ee5cbb8b0e~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

注意两点：

1、推送遵循同源策略；

2、这种服务端的推送是基于客户端的请求响应来确定的。

当服务端需要主动推送某个资源时，便会发送一个 Frame Type 为 PUSH_PROMISE 的 Frame，里面带了 PUSH 需要新建的 Stream ID。意思是告诉客户端：接下来我要用这个 ID 向你发送东西，客户端准备好接着。客户端解析 Frame 时，发现它是一个 PUSH_PROMISE 类型，便会准备接收服务端要推送的流。

**http2.0性能瓶颈**

启用http2.0后会给性能带来很大的提升，但同时也会带来新的性能瓶颈。因为现在所有的压力集中在底层一个TCP连接之上，TCP很可能就是下一个性能瓶颈，比如TCP分组的队首阻塞问题，单个TCP packet丢失导致整个连接阻塞，无法逃避，此时所有消息都会受到影响。未来，服务器端针对http 2.0下的TCP配置优化至关重要。

---

Reference：
https://halfrost.com/http2_begin/

https://zhuanlan.zhihu.com/p/26559480
https://juejin.cn/post/6844903984524705800
