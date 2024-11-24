
## 最小堆
```go
type MinHeap struct {  
   Element []int  
}  
  
func NewMinHeap() *MinHeap {  
   h := &MinHeap{  
      Element: []int{math.MinInt64},  
   }  
   return h  
}  
func (h *MinHeap) Insert(v int)  {  
   h.Element = append(h.Element,v)  
   i := len(h.Element) -1  
   for ;h.Element[i/2] > v ;i /=2 {  
      h.Element[i] = h.Element[i/2]  
   }  
   h.Element[i] = v  
}  
func (h *MinHeap) DeleteMin() (int,error)  {  
   if len(h.Element) <= 1 {  
      return 0,fmt.Errorf("MinHeap is empty")  
   }  
   minElement := h.Element[1]  
   lastElement := h.Element[len(h.Element)-1]  
   var i,child int  
   for i := 1;i*2 < len(h.Element);i = child {  
      child = i*2  
      if child < len(h.Element)-1 && h.Element[child+1] < h.Element[child] {  
         child++  
      }  
      //下滤一层  
      if lastElement > h.Element[child] {  
         h.Element[i] = h.Element[child]  
      }else {  
         break    
}  
   }  
   h.Element[i] = lastElement  
   h.Element = h.Element[:len(h.Element) - 1]  
   return minElement,nil  
}
```

## 第五个问题

> 浏览器对同一 Host 建立 TCP 连接到数量有没有限制？

**答案：**  
有。Chrome 最多允许对同一个 Host 建立六个 TCP 连接，不同的浏览器有一些区别。

---

### 终章

> 若收到的 HTML 如果包含几十个图片标签，这些图片是以什么方式、什么顺序、建立了多少连接、使用什么协议被下载下来的呢？

1、 如果图片都是 **HTTPS 连接并且在同一个域名**下，那么浏览器在 SSL 握手之后会和服务器商量能不能用 **HTTP2**，如果能的话就使用 Multiplexing 功能在这个连接上进行**多路传输**。不过也未必会所有挂在这个域名的资源都会使用一个 TCP 连接去获取，但是可以确定的是 Multiplexing 很可能会被用到。

2、 如果发现用不了 HTTP2 呢？或者用不了 HTTPS（现实中的 **HTTP2** 都是**在 HTTPS 上实现**的，所以也就是只能使用 **HTTP/1.1**）。那浏览器就会在**一个 HOST 上建立多个 TCP 连接**，连接数量的**最大限制取决于浏览器设置**，这些连接会在空闲的时候被浏览器用来发送新的请求，若所有的连接都正在发送请求，那么其他的请求就只能等待。

## doc_values 的作用：

        倒排索引虽然可以提高搜索性能，但也存在缺陷，比如我们需要对数据做排序或聚合等操作时，lucene 会提取所有出现在文档集合的排序字段，然后构建一个排好序的文档集合，而这个步骤是基于内存的，如果排序数据量巨大的话，容易造成内存溢出和性能缓慢。

        doc_values 就是 es 在构建倒排索引的同时，会对开启 doc_values 的字段构建一个有序的 “document文档 ==> field value” 的列式存储映射，可以看作是以文档维度，实现了根据指定字段进行排序和聚合的功能，降低对内存的依赖。另外 doc_values 保存在操作系统的磁盘中，当 doc_values 大于节点的可用内存，ES可以从操作系统页缓存中加载或弹出，从而避免发生内存溢出的异常，但如果 docValues 远小于节点的可用内存，操作系统就自然将所有 doc_values 存于内存中（堆外内存），有助于快速访问。


## 为什么要使用SetMemoryLimit
因为对于瞬时的内存使用上升，gc频率太多，使用setmemoryLimit可以减少

## go reflect性能
Java中的reflect的使用对性能也有影响， 但是和Java reflect不同， Java中不区分`Type`和`Value`类型的， 所以至少Java中我们可以预先将相应的reflect对象缓存起来，减少反射对性能的影响， 但是Go没办法预先缓存reflect, 因为`Type`类型并不包含对象运行时的值，必须通过`ValueOf`和运行时实例对象才能获取`Value`对象。

对象的反射生成和获取都会增加额外的代码指令， 并且也会涉及`interface{}`装箱/拆箱操作，中间还可能增加临时对象的生成，所以性能下降是肯定的，但是具体能下降多少呢，还是得数据来说话。

reflect实现里面有大量的枚举，也就是for循环，比如类型之类的。

## **为何高版本Go要改用寄存器传参？**

至于为什么Go1.17.1函数调用的参数传递开始基于寄存器进行传递，原因无外乎：

1）CPU访问寄存器比访问栈要快的多，函数调用通过寄存器传参比栈传参，性能要高5%；

2）早期Go版本为了降低实现的复杂度，统一使用栈传递参数和返回值，不惜牺牲函数调用的性能；

3）Go从1.17.1版本，开始支持多ABI（application binary interface 应用程序二进制接口，规定了程序在机器层面的操作规范，主要包括调用规约calling convention），主要是两个ABI：一个是老版本Go采用的平台通用ABI0，一个是Go独特的ABIInternal，前者遵循平台通用的函数调用约定，实现简单，不用担心底层cpu架构寄存器的差异；后者可以指定特定的函数调用规范，可以针对特定性能瓶颈进行优化，在多个Go版本之间可以迭代，灵活性强，支持寄存器传参提升性能。

\_所谓“调用规约(calling convention)”是调用方和被调用方对于函数调用的一个明确的约定，包括：函数参数与返回值的传递方式、传递顺序。只有双方都遵守同样的约定，函数才能被正确地调用和执行。如果不遵守这个约定，函数将无法正确执行。_
1）Go1.17.1之前的函数调用，参数都在栈上传递；Go1.17.1以后，9个以内的参数在寄存器传递，9个以外的在栈上传递；

2）Go1.17.1之前版本，callee函数返回值通过caller栈传递；Go1.17.1以后，函数调用的返回值，9个以内通过寄存器传递回caller，9个以外在栈上传递；

在Go 1.17的版本发布说明文档中有提到：切换到基于寄存器的调用惯例后，一组有代表性的Go包和程序的基准测试显示，Go程序的运行性能提高了约5%，二进制文件大小减少约2%。

由于CPU访问寄存器的速度要远高于栈内存，参数在栈上传递会增加栈内存空间，并且影响栈的扩缩容和垃圾回收，改为寄存器传递，这些缺点都得到了优化，Go程序在从低版本升级到17版本后，性能有一定的提升。在业务允许的情况下，这里建议相关团队可以把自己程序的Go版本升级到17及以上。

## 关于 json-iterator 库

**json-iterator 库为什么快**  
标准库 json 库使用 reflect.Value 进行取值与赋值，但 reflect.Value 不是一个可复用的反射对象，每次都需要按照变量生成 reflect.Value 结构体，因此性能很差。  
json-iterator 实现原理是用 reflect.Type 得出的类型信息通过「对象指针地址+字段偏移」的方式直接进行取值与赋值，而不依赖于 reflect.Value，reflect.Type 是一个可复用的对象，同一类型的 reflect.Type 是相等的，因此可按照类型对 reflect.Type 进行 cache 复用。  
总的来说其作用是**减少内存分配**和**反射调用次数**，进而减少了内存分配带来的系统调用、锁和 GC 等代价，以及使用反射带来的开销。

echo 3 drop_caches 命令不能直接释放由mmap映射的内存。
mmap(内存映射)是一种将文件或设备映射到进程地址空间的机制，通过将文件或设备映射到虚拟内存，进程可以直接读写这些映射区域，而不需要显式的文件操作mmap映射的文件数据通常会被缓存在内存中，以提高访问速度。
当使用 echo 3 drop_caches 命令时，它主要清除的是页缓存、目录项缓存和inode缓存，而不是清除mmap映射的文件数据所占用的内存。因此， echo 3 drop_caches 命令无法直接释放mmap映射的内存。
如果你希望释放由mmap映射的内存，你可以通过以下方法之一来实现:
1.解除映射:使用 munmap 系统调用将映射区域从进程的地址空间解除映射。
2关闭文件句柄:如果映射是通过已打开的文件描述符进行的，可以通过关闭该文件句柄来解除映射。
需要注意的是，解除映射后的内存将被释放，但这并不意味着它会立即归还给操作系统。操作系统会根据需要自动管理和回收内存资源

# 别用float作为map的key
现在可以看到它其实是把浮点数的内存表示（IEEE 754 double encoding format) 当做一个普通的数字来计算的，先读4字节计算，再读剩下的4字节计算，再做其他计算。

两个浮点数最终hash值相同，其实就是浮点数精度导致的，写代码的时候，看上去我们定义了两个完全不同的浮点数，但是内存中按照IEEE 754 double的规范进行内存表示的时候，很可能就是一样的。

![IEEE 754 Double Format](https://www.hitzhangjie.pro/blog/assets/gomap/1558586071_3.png)

上面是IEEE 754 double的格式，1位符号位，11位阶码，52位尾数，像NaN、Inf的定义也都跟这些不同的组成部分有关。这里只关心尾数部分就好了，只有52位，当超过52 bits可以表示的精度之后，代码里面定义的数值就被截断了。

示例代码中，看似数值上有一两位末尾数字不同的浮点数，内存表示是相同的，hash值也相同，就会出现map赋值时值被覆盖的问题。

浮点数的更多细节如符号位、阶码（bias）、尾数，以及NaN、Inf的定义等，可以参考wikipedia了解更多细节。

## # 自定义结构体，能作为map的key吗
从上面`map`基本的模型介绍中，我们了解到，`map`中的Key需要支持哈希函数的计算，同时键的类型必须支持对比操作。

在`map`中，计算`key`的哈希值，是由默认哈希函数实现的，对于`map`中的`key`并没有额外的要求。

在`map`中，判断两个键是否相等是通过调用键类型的相等运算符（`==`或`!=`）来完成的，因此`key`必须确保该类型支持 `==` 操作。这个要求是由 `map` 的实现机制决定的。`map` 内部使用键的相等性来确定键的存储位置和检索值。如果键的类型不可比较，就无法进行相等性比较，从而导致无法准确地定位键和检索值。

在 Go 中，基本数据类型（如整数、浮点数、字符串）和一些内置类型都是可比较的，因此它们可以直接用作 `map` 的键。然而，自定义的结构体作为键时，需要确保结构体的所有字段都是可比较的类型。如果结构体包含引用类型的字段，那么该结构体就不能直接用作 `map` 的键，因为引用类型不具备简单的相等性比较。

因此，假如`map`中的键为自定义类型，同时包含引用字段，此时将无法作为`map`的键，会直接编译失败
## tcp如何应对syn攻击
SYN 攻击（SYN Flood Attack）是一种常见的网络攻击方式，利用 TCP 协议中的三次握手过程中的漏洞进行攻击。攻击者发送大量伪造的 TCP 连接请求（SYN 包），但不完成后续的握手过程，从而消耗服务器的资源，导致服务不可用。为了应对 SYN 攻击，可以考虑以下措施：

SYN Cookie：SYN Cookie 是一种防御 SYN 攻击的机制，在服务器端启用 SYN Cookie 可以有效地抵御大规模 SYN 攻击。它通过在接收到 SYN 请求时，根据请求计算一个 Cookie 值，并将该 Cookie 值作为序列号发送给客户端。只有在客户端发送 ACK 响应时，服务器才会解析该 Cookie 值并建立连接。这样可以避免在服务器端维护半开连接队列，从而减轻了服务器的负担。

调整系统参数：可以根据具体情况调整系统参数，以增加服务器对 SYN 攻击的抵抗能力。例如，调整操作系统的最大连接数、半开连接队列大小、超时时间等参数，以适应大规模 SYN 攻击的压力。

使用防火墙：通过配置防火墙规则，限制对服务器的连接请求。可以限制源 IP 地址、连接频率等，过滤掉恶意的 SYN 请求。

使用反向代理：使用反向代理服务器作为前置服务器，将请求分发给后端真实服务器。反向代理可以过滤掉一部分 SYN 攻击流量，并将合法的请求转发给后端服务器。

流量限制和流量清洗：可以使用流量限制和流量清洗技术来过滤掉异常或恶意的流量。这些技术可以根据流量的特征进行判断和过滤，有效地阻止 SYN 攻击流量。

使用专业的安全设备和软件：使用专门的网络安全设备和软件，如入侵检测系统（IDS）、入侵防御系统（IPS）等，能够对 SYN 攻击进行实时监测和防护。

综上所述，针对 SYN 攻击，可以采取多种措施来应对，包括使用 SYN Cookie 机制、调整系统参数、使用防火墙、使用反向代理、流量限制和流量清洗等。同时，使用专业的安全设备和软件可以增强网络的安全性和抵御 SYN 攻击的能力。

## **02 TF-IDF 和 BM25 是什么**

**2.1 词频 TF（Term Frequency）**

**检索词在文档中出现的频度是多少？出现频率越高，相关性也越高。**

关于TF的数学表达式，参考ES官网，如下：

> tf(t in d) = √frequency 词 t 在文档 d 的词频（ tf ）是该词在文档中出现次数的平方根。

**概念理解**：比如说我们检索关键字“es”，“es”在文档A中出现了10次，在文档B中只出现了1次。我们不会认为文档B与“es”的相关性更高，而是文档A。

  

**2.2 逆向⽂档频率 IDF（Inverse Document Frequency）**

**每个检索词在索引中出现的频率，频率越高，相关性越低。**

关于 IDF 的数学表达式，参考ES官网，如下：

> idf(t) = 1 + log ( numDocs / (docFreq + 1)) 词 t 的逆向文档频率（ idf ）是：索引中文档数量除以所有包含该词的文档数，然后求其对数。注意: 这里的log是**指以e为底的对数,不是以10为底的对数。**

**概念理解：**比如说检索词“学习ES”，按照Ik分词会得到两个Token【学习】【ES】，假设在当前索引下有100个文档包含Token“学习”，只有10个文档包含Token“ES”。那么对于【学习】【ES】这两个Token来说，**出现次数较少的 Token【ES】就可以帮助我们快速缩小范围找到我们想要的文档，**所以说此时“ES”的权重就比“学习”的权重要高。

  

**2.3 字段长度准则 field-length norm**

**字段的长度是多少？字段越短，字段的权重_越高_。**检索词出现在一个内容短的 title 要比同样的词出现在一个内容长的 content 字段权重更大。

关于 norm 的数学表达式，参考ES官网，如下：

> norm(d) = 1 / √numTerms 字段长度归一值（ norm ）是字段中词数平方根的倒数。

  

以上三个因素——词频（term frequency）、逆向文档频率（inverse document frequency）和字段长度归一值（field-length norm）——**是在索引时计算并存储的。最后将它们结合在一起计算单个词在特定文档中的权重。**
**2.5 BM25**

整体而言**BM25 就是对 TF-IDF 算法的改进**，对于 TF-IDF 算法，**TF(t) 部分的值越大，整个公式返回的值就会越大。**

BM25 就针对这点进行来优化，**随着TF(t) 的逐步加大，该算法的返回值会趋于一个数值。**

![](https://pic2.zhimg.com/80/v2-f6f6a139e373f4333ae29444f9674891_720w.webp)

BM25 有一个比较好的特性就是提供了**两个可调参数：**

![](https://pic3.zhimg.com/80/v2-a9f7932148c186b5e8f0123160f4e0de_720w.webp)

> k1：这个参数控制着词频结果在**词频饱和度中的上升速度。**默认值为1.2。值越小饱和度变化越快，值越大饱和度变化越慢。  
> b：这个参数**控制着字段长归一值所起的作用，**0.0会禁用归一化，1.0会启用完全归一化。默认值为0.75。

该公式"."的前部分就是 IDF 的算法，后部分就是 TF+Norm 的算法。

# nutsdb启动速度优化之旅

原创 陪计算机走过漫长岁月 陪计算机走过漫长岁月 _2022-09-24 13:32_ _发表于北京_

## 背景

有一天nutsdb学习交流群中有一位老哥说nutsdb重新恢复的速度太慢了，他那边大概有一个G左右的数据。我对这个问题还是比较感兴趣的，当场就接下了这个任务。开启了这段时间的优化之路，历时还蛮长时间的，因为平时上班，还有其他的一些兴趣爱好什么，所以开源上面的投入倒是有细水长流的感觉。好了，闲话不多说了，下面把这段时间在这个问题上面的研究以及一些心得体会分享给大家。

## 分析问题

俗语有云，能复现的问题都不是什么大问题。根据学习老哥的描述，我首先要做的就是准备1G的数据，然后benchmark+pprof测试重启恢复db的cpu和内存情况。最终问题定位到了这段代码上面：

```
// ReadAt returns entry at the given off(offset).
```

这段代码的作用是给定一个文件的偏移位置，从这个位置开始往后读取一条数据。在db重启恢复的过程中这个函数会被频繁的调用，为什么呢？我们知道nutsdb是基于bitcask模型实现的，本质上来说是hash索引。在重启恢复的过程中会加载所有db的数据来重新构建索引。所以理论上来说，重启的时候当前db有多少数据，这个函数就会被调用多少次，由于这个函数性能不大行，所以这里成为了瓶颈。为什么说这个函数性能不太行呢？且看我娓娓道来。

一条nutsdb中的数据是以这样的形式被存储在磁盘中的：

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBT8EXWBqdZEdP20JicoAAqGos5Hl1y1fAbBl8sWTWRhuibQWwZajreL0tw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

首先我们需要获取这条数据的元数据(header)，也就是描述真实数据的数据，这里会记录bucket，key，value的长度，我们拿到这些信息之后会再次发起系统调用去获取真实bucket，key，value。由这段代码我们可以看到，这个函数没执行一次，就会发起四次系统调用，分别在磁盘中获取meta，bucket，key，value。而且每次都需要申请与数据长度匹配字节数组去承接这些数据。这个现状有两个问题，其一是过多的系统调用会是性能的瓶颈，其次是过多的申请内存，在这些对象被使用完之后会在内存中堆积起来，触发gc之后才会被清除回收。但是如果db数据过多，在这个时候也会触发gc造成卡顿。

## 如何解决？

在定位到问题之后，就开始了解决问题的旅程，这段旅程异常的漫长，这段过程翻阅了很多资料，尝试了很多方法。且看我一一道来。

### 1. 减少系统调用

在上面的分析中，我们知道读一条数据需要发起四次系统调用。我觉得这里其实大可不必，两次就足够了。因为第一次读取meta之后，我们就知道bucket，key，value这些数据的长度和位置了加上这些数据都是连续存储的，直接一次全部拿出来然后分别解析就好。

所以在第一次优化中，将上面的四次系统调用优化成了2次，带来了一倍速度上的提升。不仅仅是重启速度上的，因为在运行中读取数据用的也是这个函数。所以在数据的读取上同样也是优化了一倍的读取速度。下图是我提交的pr的截图，我构造了大小分别是50B， 256B，1024B的数据，db总数据1G，做并发读取性能测试，实验结果表明此次优化在读取速度上提升了接近一倍，但是在内存的使用上并没有减少。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBTAvzLxKohqzgxPbsl9vJLzhPK5vfISA7iae0XsjBBtSGEMVnnzTcIDfw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### 2. 批量读取，然后解析数据

在做完系统调用的优化之后的一个星期的时间里，这项优化工作其实陷入了沉寂，有时候会脑爆一下这个该怎么做，翻越了bitcask论文和DDIA对hash索引恢复操作的描述，书中所说都是要将数据一条条的拿出来，没有可参考的资料感觉有点寸步难行。不过天无绝人之路，有一天打游戏的时候突然想到了，为什么不一次性读取文件的一部分，比如4KB，然后在这4KB中解析出里面的数据，解析到解析不下去的时候再拿下一个，直到文件解析完毕。下面让我们展开一下这个思路。

我们知道磁盘底层会把自己存储空间划分成一个个扇区，在磁盘之上的文件系统会把磁盘的扇区组成一个个的block，我们的应用程序每次去读取数据都是按照block去读取的。打个比方，一个block是4KB，如果我要读1KB的数据，那么会加载4KB的数据到pagecache，然后返回1KB给我们的应用程序。所以当我们一次次发起系统调用的去磁盘回去数据的时候，实际上数据已经从磁盘里拿出来并且放到pagecache中了。所以我们直接把整个block拿出来就完事了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBTwT497emmBjbqRuZdE2FpHNnM83tzEawLlPNQziahSZU5Q1icwPto1t5A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

一开始想的是批量获取一批完整的数据。这个时候其实不太好控制，因为我们不知道一条数据有多大，所以如果要要批量获取数据还需要额外的数据结构去描述，我把这个东西叫做断点，也就是说一个文件分成若干个断点，断点之间存储的是一批数据。如果朝这个方向改会衍生出很多问题，其一是在什么样的标准下面去写这个断点，比如按照数据量来记呢，还是按照一个固定的空间大小来记录。其二是再增加这个逻辑去额外记录还需要在数据写入过程中做添加。一来二去感觉太复杂了。

那么我们按照一个个固定的块来搞。由于我们不知道数据的大小是怎么样的，所以按照固定块读取会出现以下的情况。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBT5LHcGdDpKh9NQXuMBZbMYcEx2nsJaAVEFusfIMZW8z1BEwVHibjiboPw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

我们看上图，上图中的e1和e2代表着在一个存储文件中一条条连续着存储的数据，header是元数据，payload是对bucket，key，value这三个数据的整体描述，即有效数据部分。图中的箭头代表按块读取数据的时候有可能读到的块的边界。其实也很好理解，在读一个块的时候可能会出现的情况是：

1. 这个块就包含了若干条完完整整的数据。这个时候不需要额外的处理。
    
2. 这个块前面包含了若干条完整的数据，但是最后的一条是残缺的，缺失了header。这个时候我们需要读入header残余的数据，解析出一整个header。为什么不读一整个block然后在在block中获取解析header所需的剩余数据呢？因为我们不知道这条数据长什么样子，如果他的大小超过了一个block的size，那么还需要在后续做额外的处理。
    
3. 最后一种情况是缺失了payload，那么我们需要根据缺失数据的大小来判断要读多少个block进来。
    

经过这一分析之后，其实一整个解析数据的流程就是在这三个状态之间来回转换。解析的流程就是一个状态机的算法。等到文件读完，那么也就解析完了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBTA91ONBQ7TfUX6YYRB5xnNk0e3g9TlON0ujDpCRPlJubSd08HbFKFMA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

激动的心颤抖的手，代码写完之后做了一版benchmark。

BenchmarkRecovery-10           1 2288482750 ns/op 1156858488 B/op 14041579 allocs/op  
BenchmarkRecovery-10           2 701933396 ns/op 600666340 B/op 3947879 allocs/op

我们分析一下这个思路下的优化，理论上来说db中一条数据越小，优化效果越明显，因为这就意味着单次获取的block中蕴含着更多的数据，就可以减少更多的系统调用。但是我们可以看到的是，虽然启动的时间变短了，申请内存的次数变少了，但是申请的内存还是不少。

执行两次启动恢复1G左右的数据大概花了4G的内存。也就是说一次启动花了2G，里面有1G消耗不知道是在哪里来的。一开始我以为是读取数据的时候会在用户空间往内核空间拷贝，会存在读取放大的情况。不过转念一想觉得不大可能，因为pagecache是类似LRU的缓存机制，不会缓存你所有的数据，他也有数据的淘汰策略，不太可能1比1放大的。

所以问题出在哪里呢？我百思不得其解。不过源码之下无秘密，我反复一次又一次看我写下的代码，终于让我在一个地方发现了猫腻。一个我意想不到的地方，那就是append操作。

我的文件数据恢复恢复操作所代表的结构体是这样的：

```
FileRecovery struct {
```

我的解析流程是把文件读完，把里面的数据一个个append进entries字段里面，文件解析完了再把entries返回。要是在平时我不会觉得这个操作是有什么问题的，不过我准备的数据量太大了（两百万条），append应该会触发多次扩容操作，然后反复申请内存。去翻看了一下append操作的扩容机制，当数据量小于1024的时候，会扩成原来的2倍，数据量大于1024时，会变成原来的1.25倍。看到这里其实基本验证了我的猜想，不过谨慎起见还是计算了一下, nutsdb数据有segment设置，当一个文件大于这个参数的大小的时候会保存成只读文件。两百万条数据写了两个文件，这个结构体是解析单个文件的，我们按照每个文件100万条数据算。100万条数据大概要扩容几次呢：

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBTzJSHwBnicexHzPoMXKL3xTX07qicGdtpGQMibTib0ZqlBibPk2TxHJtrD8A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

1024前面的过程就忽略不计了，直接从1024开始，掏出计算器，这里x的值是30次，那么扩容30次总共需要多少内存呢？

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBTRBPdRlkNYH0BvvEoQF10qZAnXsYoQJicaX1EsD2ibSk56zr9Ctwqx4Pw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

为什么是乘8呢，因为我是64位机器，entries里面的元素是指针，那么指针的长度和字长是一样的，也就是8个byte。这里的结果大概是3000万，3000万个比特大概是220MB的内存，两个文件加起来大概就是440MB的内存了。惊不惊喜，意不意外？

那么如何优化这里呢，其实一个个往外传就可以了，不要等到解析完了堆在数组里再往外丢。如果要在原来的基础上改的话，我需要每次处理完一条数据的时候记录下当前这个block处理到哪里，也就是要把block改成ring buffer的形式，这个时候我隐隐约约想到了go的标准库里有一个东西做了类似的事情，没错，就是bufio.Reader.他不仅仅会帮我们看到记录ringbuffer的位置，还会预读block。整体上看是完美契合这个需求的。

### 3. bufio.Reader

bufio.Reader会维护一个默认大小4KB的ring buffer，当读取的内容大于它所能承载的缓存时，他会直接在读取而不使用缓存，如果读取的是比较小的数据，会在他的缓存中copy出来一份返回。这个就完美的契合了我们多次读取的场景。看一下改成这样子的性能提升吧：

BenchmarkRecovery-10           1 2288482750 ns/op 1156858488 B/op 14041579 allocs/op  
BenchmarkRecovery-10           4 265742781 ns/op 152531440 B/op 1089555 allocs/op

这下无论是速度上，还是内存的使用上，都达到了一个比较理想的状态。这次优化历程就到此结束了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BtqZzg1Lt0wbXVOD5bEjV6gAjLFMcsBTvYRSHTxCaF1F3ac9Viap95YSRxd5VQ6AYzFVrlXiaibQPBFibMqjXNulJA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结

做性能优化的感觉就像和计算机对话，依照自己现有的知识去想方案， 然后写出来之后做实验求证。做这个的思路是让他慢慢的变好，而不是上来就追求完美主义，完美主义是不靠谱的，反而会让你陷入到纠结之中，能优化一点是一点，我们要看到一个变好的趋势，然后在这个趋势上面不断的基于上一次的结果去猜想下一次怎么优化，也就是所谓的“小步快跑”。在做这个的过程中会往各个方向去脑爆，一些背景知识不是很清楚的时候需要翻越各种资料。整体来说是一次很不错的成长体验。

## 参考资料

1. 磁盘那些事：https://tech.meituan.com/2017/05/19/about-desk-io.html
    
2. bufio读取原理：https://studygolang.com/articles/35636?fr=sidebar
    
3. 一次系统调用的开销：https://zhuanlan.zhihu.com/p/80206565
    


### 内存分配方式

## 线性分配

> 线性分配（Bump Allocator）是一种高效的内存分配方法，但是有较大的局限性。当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，即移动下图中的指针：  
> ![图片](https://segmentfault.com/img/remote/1460000021948710 "图片")  
> 线性分配器虽然线性分配器实现为它带来了较快的执行速度以及较低的实现复杂度，但是线性分配器无法在内存被释放时重用内存。如下图所示，如果已经分配的内存被回收，线性分配器无法重新利用红色的内存：  
> ![图片](https://segmentfault.com/img/remote/1460000021948712 "图片")  
> 线性分配器回收内存因为线性分配器具有上述特性，所以需要与合适的垃圾回收算法配合使用，例如：标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法，它们可以通过拷贝的方式整理存活对象的碎片，将空闲内存定期合并，这样就能利用线性分配器的效率提升内存分配器的性能了。因为线性分配器需要与具有拷贝特性的垃圾回收算法配合，所以 C 和 C++ 等需要直接对外暴露指针的语言就无法使用该策略。
> 
> 引用自：[https://draveness.me/golang/d...](https://link.segmentfault.com/?enc=w%2B7838eKEj%2FlYUSR%2F8S7Lg%3D%3D.%2FZoW0mVHSLsK6Tuw%2FcT6EnIv%2Fi6G%2BNL%2FHJHRYrsUfynLbPeEBS4ECVgR8Oq50kJoUcx0UCStfIdANOzWI7ylp%2Bvo3LNOQglQqbnJEIf7VlFdbmJHEZuG%2BY24RTT9FQ8OjWcvtjQJJOKOqvLva%2Bv5s%2BiSx6exIzBjhBz3EKyxFTo%3D)

应用代表：Java（如果使用 Serial, ParNew 等带有 Compact 过程的收集器时，采用分配的方式为线性分配）  
问题：内存碎片  
解决方式：GC 算法中加入「复制/整理」阶段

## 空闲链表分配

> 空闲链表分配器（Free-List Allocator）可以重用已经被释放的内存，它在内部会维护一个类似链表的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表：  
> ![图片](https://segmentfault.com/img/remote/1460000021948713 "图片")  
> 空闲链表分配器因为不同的内存块通过指针构成了链表，所以使用这种方式的分配器可以重新利用回收的资源，但是因为分配内存时需要遍历链表，所以它的时间复杂度是 𝑂(𝑛)。空闲链表分配器可以选择不同的策略在链表中的内存块中进行选择，最常见的是以下四种：
> 
> - 首次适应（First-Fit）— 从链表头开始遍历，选择第一个大小大于申请内存的内存块；
> - 循环首次适应（Next-Fit）— 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块；
> - 最优适应（Best-Fit）— 从链表头遍历整个链表，选择最合适的内存块；
> - 隔离适应（Segregated-Fit）— 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；
> 
> 引用自：[https://draveness.me/golang/d...](https://link.segmentfault.com/?enc=fzlalBbfAprmlN0rxoN7vA%3D%3D.uWS7Ue12BP%2BoBRg8c6kJ%2B%2FV68jUhkPfb%2Fe%2FSTKoi43U9ITYUmSvBQG7WuAttmaqcesF4tKY5%2BZhwdKqrTkimCG3UkU68dk8CT2090Mu6n9bmN519mOhxaiRiNJOtoft23mFtiyjOEZ4%2F5XBlRu%2BFn9H5sCCRWkIf%2B7gpQzBDZ1U%3D)

应用代表：GO、Java（如果使用 CMS 这种基于标记-清除，采用分配的方式为空闲链表分配）  
问题：相比线性分配方式的 bump-pointer 分配操作（top += size），空闲链表的分配操作过重，例如在 GO 程序的 pprof 图中经常可以看到 mallocgc() 占用了比较多的 CPU；



# ElasticSearch搜索引擎：常用的存储mapping配置项 与 doc_values详细介绍
## 一、ES的数据存储结构：

ES底层使用 Lucene 存储数据，Lucene 的索引包含以下部分：

> A Lucene index is made of several components: an inverted index, a bkd tree, a column store (doc values), a document store (stored fields) and term vectors, and these components can communicate thanks to these doc ids.

 其中：

- inverted index：倒排索引。
- bkd tree: Block k-d tree，用于在高维空间内做索引，如地理坐标的索引。
- column store：doc values，列式存储，批量读取连续的数据以提高排序和聚合的效率。
- document store：Store Fileds，行式存储文档，用于控制 doc 原始数据的存储，其中占比最大的是 source 字段。
- term vectors：用于存储各个词在文档中出现的位置等信息。

## 二、ES常见配置项说明：

在很多场合下，我们并不需要存储上述全部信息，因此可以通过设置 mappings 里面的属性来控制哪些字段是我们需要存储的、哪些是不需要存储的。而 ES 的 mapping 中有很多设置选项，这些选项如果设置不当，有的可能浪费存储空间，有的可能导致无法使用 Aggregation，有的可能导致不能检索。下面就简单介绍下 ES 中常见的存储与检索的 mapping 配置项：

|   |   |   |   |
|---|---|---|---|
|配置项|作用|注意事项|默认值|
|_all|提供跨字段全文检索|（1）会占用额外空间，把 mapping 中的所有字段通过空格拼接起来做索引，在跨字段全文检索才需要打开；<br><br>（2）在 v6.0+已被弃用，v7.0会正式移除，可以使用 [copy_to] 来自定义组合字段|关闭|
|_source|存储 post 提交到ES的原始 json 内容|（1）会占用很多存储空间。数据压缩存储，读取会有额外解压开销。<br><br>（2）不需要读取原始字段内容可以考虑关闭，但关闭后无法 reindex|开启|
|store|是否单独存储该字段|（1）会占用额外存储空间，与 source 独立，同时开启 store 和 source 则会将该字段原始内容保存两份，不同字段单独存储，不同字段的数据在磁盘上不连续，若读取多个字段则需要查询多次，如需读取多个字段，需权衡比较 source 与 store 效率|关闭|
|doc_values|支持排序、聚合|会占用额外存储空间，与 source 独立，同时开启 doc_values 和 _source 则会将该字段原始内容保存两份。doc_values 数据在磁盘上采用列式存储，关闭后无法使用排序和聚合|开启|
|index|是否加入倒排索引|关闭后无法对其进行搜索，但字段仍会存储到 _source 和 doc_values，字段可以被排序和聚合|开启|
|enabled|是否对该字段进行处理|关闭后，只在 _source中存储，类似 index 与 doc_values 的总开关|开启|

在ES的 mapping 设置里，all，source 是 mapping 的元数据字段（Meta-Fields），store、doc_values、enabled、index 是 mapping 参数。

**1、_all：**

all 字段的作用是提供跨字段查询的支持，把 mapping 中的所有字段通过空格拼接起来做索引。ES在查询的过程中，需要指定在哪一个field里面查询。

```perl
{    “name”: “smith”,    “email”: "John@example.com"}
```

用户在查询时，想查询叫做 John 的人，但不知道 John 出现在 name 字段中还是在 email 字段中，由于ES是为每一个字段单独建立索引，所以用户需要以 John 为关键词发起两次查询，分别查询name字段和email字段。

如果开启了 all 字段，则ES会在索引过程中创建一个虚拟的字段 all，其值为文档中各个字段拼接起来所组成的一个很长的字符串（例如上面的例子，all 字段的内容为字符串 “smith John@example.com”）。随后，该字段将被分词打散，与其他字段一样被收入倒排索引中。由于 all 字段包含了所有字段的信息，因此可以实现跨字段的查询，用户不用关心要查询的关键词在哪个字段中。

由于该字段的内容都来自 source 字段，因此默认情况下，该字段的内容并不会被保存，可以通过设置 store 属性来强制保存 all 字段。开启 all 字段，会带来额外的CPU开销和存储，如果没有使用到，可以关闭 all 字段。

**2、_source：**

source 字段用于存储 post 到 ES 的原始 json 文档。为什么要存储原始文档呢？因为 ES 采用倒排索引对文本进行搜索，而倒排索引无法存储原始输入文本。一段文本交给ES后，首先会被分析器(analyzer)打散成单词，为了保证搜索的准确性，在打散的过程中，会去除文本中的标点符号，统一文本的大小写，甚至对于英文等主流语言，会把发生形式变化的单词恢复成原型或词根，然后再根据统一规整之后的单词建立倒排索引，经过如此一番处理，原文已经面目全非。因此需要有一个地方来存储原始的信息，以便在搜到这个文档时能够把原文返回给查询者。

那么一定要存储原始文档吗？不一定！如果没有取出整个原始 json 结构体的需求，可以在 mapping 中关闭 source 字段或者只在 source 中存储部分字段（使用store）。 但是这样做有些负面影响：

- （1）不能获取到原文
- （2）无法reindex：如果存储了 source，当 index 发生损坏，或需要改变 mapping 结构时，由于存在原始数据，ES可以通过原始数据自动重建index，如果不存 source 则无法实现
- （3）无法在查询中使用script：因为 script 需要访问 source 中的字段

**3、store：**

store 决定一个字段是否要被单独存储。大家可能会有疑问，source 里面不是已经存储了原始的文档嘛，为什么还需要一个额外的 store 属性呢？原因如下：

（1）如果禁用了 source 保存，可以通过指定 store 属性来单独保存某个或某几个字段，而不是将整个输入文档保存到 source 中。

（2）如果 source 中有长度很长的文本（如一篇文章）和较短的文本（如文章标题），当只需要取出标题时，如果使用 source 字段，ES需要读取整个 source 字段，然后返回其中的 title，由此会引来额外的IO开销，降低效率。此时可以选择将 title 的 store 设置为true，在 source 字段外单独存储一份。读取时不必在读取整 source 字段了。但是需要注意，应该避免使用 store 查询多个字段，因为 store 的存储在磁盘上不连续，ES在读取不同的 store 字段时，每个字段的读取均需要在磁盘上进行查询操作，而使用 source 字段可以一次性连续读取多个字段。

**4、doc_values：**

倒排索引可以提供全文检索能力，但是无法提供对排序和数据聚合的支持。doc_values 本质上是一个序列化的列式存储结构，适用于聚合（aggregations）、排序（Sorting）、脚本（scripts access to field）等操作。默认情况下，ES几乎会为所有类型的字段存储doc_value，但是 text 或 text_annotated 等可分词字段不支持 doc values 。如果不需要对某个字段进行排序或者聚合，则可以关闭该字段的doc_value存储。

**5、index：**

控制倒排索引，用于标识指定字段是否需要被索引。默认情况下是开启的，如果关闭了 index，则该字段的内容不会被 analyze 分词，也不会存入倒排索引，即意味着该字段无法被搜索。

**6、enabled：**

这是一个 index 和 doc_value 的总开关，如果 enabled 设置为false，则这个字段将会仅存在于 source 中，其对应的 index 和 doc_value 都不会被创建。这意味着，该字段将不可以被搜索、排序或者聚合，但可以通过 source 获取其原始值。

**7、term_vector：**

在对文本进行 analyze 的过程中，可以保留有关分词结果的相关信息，包括单词列表、单词之间的先后顺序、单词在原文中的位置等信息。查询结果返回的高亮信息就可以利用其中的数据来返回。默认情况下，term_vector是关闭的，如有需要（如加速highlight结果）可以开启该字段的存储。

## 三、doc_values 详细说明：

**1、doc_values 的作用：**

基于 lucene 的 solr 和 es 都是使用倒排索引实现快速检索的，也就是通过建立 "搜索关键词 ==>文档ID列表" 的关系映射实现快速检索，但是倒排索引也是有缺陷的，比如我们需要字段值做一些排序、分组、聚合操作，lucene 内部会遍历提取所有出现在文档集合的排序字段，然后再次构建一个最终的排好序的文档集合list，这个步骤的过程全部维持在内存中操作，而且如果排序数据量巨大的话，非常容易就造成solr内存溢出和性能缓慢。

doc values 就是在构建倒排索引时，会对开启 doc values 的字段额外构建一个有序的 "document文档 ==> field value“ 的列式存储映射，从而实现对指定字段进行排序和聚合时对内存的依赖，提升该过程的性能。默认情况下每个字段的 doc values 都是开启的，当然 doc values 也会耗费一定的磁盘空间。

另外 doc values 保存在操作系统的磁盘中，当 doc values 大于节点的可用内存，ES 可以从操作系统页缓存中加载或弹出，从而避免发生 JVM 内存溢出的异常，docValues 远小于节点的可用内存，操作系统自然将所有Doc Values存于内存中（堆外内存），有助于快速访问。

**2、doc_values 与 source 的区别？使用 docvalue_fields 检索指定的字段？**

post 提交到 ES 的原始 Json 文档都存储在 source 字段中，默认情况下，每次搜索的命中结果都包含文档 source，即使仅请求少量字段，也必须加载并解析整个 source 对象，而 source 每次使用时都必须加载和解析，所以使用 source 非常慢。为避免该问题，当我们只需要返回相当少的支持 doc_values 的字段时，可以使用 [docvalue_fields](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/search-fields.html#docvalue-fields) 参数获取选定字段的值。

doc values 存储与 \_source 相同的值，但在磁盘上基于列的结构中进行了优化，以进行排序和汇总。由于每个字段都是单独存储的，因此 Elasticsearch 仅读取请求的字段值，并且可以避免加载整个文档 \_source。通过 docvalue_fields 可以从建好的列式存储结果中直接返回字段值，毕竟 source 是从一大片物理磁盘去，理论上从 doc values 处拿这个字段值会比 source 要快一点，页面抖动少一点。

**3、如何在 ES 中使用 doc values？**

doc values 通过牺牲一定的磁盘空间带来的好处主要有两个：

- 节省内存
- 提升排序，分组等聚合操作的性能

那么我们如何使用 doc values 呢？

（1）我们首先关注如何激活 doc values，只要开启 doc values 后，排序，分组，聚合的时候会自动使用 doc values 提速。在 [ElasticSearch](https://so.csdn.net/so/search?q=ElasticSearch&spm=1001.2101.3001.7020) 中，doc values 默认是开启的，比较简单暴力，我们也可以酌情关闭一些不需要使用 doc values 的字段，以节省磁盘空间，只需要设置 doc_values 为 false 就可以了，如下：

```csharp
"session_id":{"type":"string","index":"not_analyzed","doc_values":false}
```

（2）使用 docvalue_fields 的检索指定的字段：

```cobol
GET my-index-000001/_search{  "query": {    "match": {      "user.id": "kimchy"    }  },  "docvalue_fields": [    "user.id",    "http.response.*",     {      "field": "date",      "format": "epoch_millis"     }  ]}
```

ES搜索指定字段的不同方式，详情请见官网：[https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-fields.html#search-fields](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-fields.html#search-fields)

