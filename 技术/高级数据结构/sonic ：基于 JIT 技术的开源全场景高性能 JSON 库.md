## 2 快速试用sonic

### 2.1 较小侵入的使用方法

想要了解sonic会对自己的服务产生多大的性能提升，评估是否值得切换，可以下面的方式较小侵入地将当前使用的json库切换为sonic：使用[http://github.com/brahma-adshonor/gohook](https://link.zhihu.com/?target=http%3A//github.com/brahma-adshonor/gohook)，在main函数的入口处hook当前使用的json库函数为sonic中对等的函数。

```text
import "github.com/brahma-adshonor/gohook"

func main() {
    // 在main函数的入口hook当前使用的json库（如encoding/json）
    gohook.Hook(json.Marshal, sonic.Marshal, nil)
    gohook.Hook(json.Unmarshal, sonic.Unmarshal, nil)
}
```

从上面可以看到，hook是函数级的，因此可以具体验证具体函数的性能提升，也可以部分函数使用sonic（出于对某些函数的不信任、或者自己有性能更优异或更稳定的实现）。

**关于gohook**

[http://github.com/brahma-adshonor/gohook](https://link.zhihu.com/?target=http%3A//github.com/brahma-adshonor/gohook)的大概实现是向被hook的函数地址中写入跳转指令，直接跳转到新的函数地址。 \_需要注意的是，gohook未经过生产环境验证，建议仅测试使用。_
## 为什么要自研 JSON 库

JSON（JavaScript Object Notation） 以其简洁的语法和灵活的自描述能力，被广泛应用于各互联网业务。但是 JSON 由于本质是一种文本协议，且没有类似 Protobuf 的强制模型约束（schema），编解码效率往往十分低下。再加上有些业务开发者对 JSON 库的不恰当选型与使用，最终导致服务性能急剧劣化。

在字节跳动，我们也遇到了上述问题。根据此前统计的公司 CPU 占比 TOP 50 服务的性能分析数据，JSON 编解码开销总体接近 10%，单个业务占比甚至超过 40%，提升 JSON 库的性能至关重要。因此我们对业界现有 Go JSON 库进行了一番评估测试。

首先，根据主流 JSON 库 API，我们将它们的使用方式分为三种：

- **泛型（generic）编解码**：JSON 没有对应的 schema，只能依据自描述语义将读取到的 value 解释为对应语言的运行时对象，例如：JSON object 转化为 Go map[string]interface{}；
- **定型（binding）编解码**：JSON 有对应的 schema，可以同时结合模型定义（Go struct）与 JSON 语法，将读取到的 value 绑定到对应的模型字段上去，同时完成数据解析与校验；
- **查找（get）& 修改（set）**：指定某种规则的查找路径（一般是 key 与 index 的集合），获取需要的那部分 JSON value 并处理。

其次，我们根据样本 JSON 的 key 数量和深度分为三个量级：

- **小（small）**：400B，11 key，深度 3 层；
- **中（medium）**：110KB，300+ key，深度 4 层（实际业务数据，其中有大量的嵌套 JSON string)；
- **大（large）**：550KB，10000+ key，深度 6 层。

测试结果如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0gXCYzMFBsFFl3n4ESrRtJzQ2IYiaFpolZGapib9lUmZibOUqYJ7tXMb1g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

不同数据量级下 JSON 库性能表现

结果显示：**目前这些 JSON 库均无法在各场景下都保持最优性能，即使是当前使用最广泛的第三方库 json-iterator，在泛型编解码、大数据量级场景下的性能也满足不了我们的需要**。

JSON 库的基准编解码性能固然重要，但是对不同场景的最优匹配更关键 —— 于是我们走上了自研 JSON 库的道路。

首先，标准库使用的**基于模式(Schema)的处理机制**是值得称赞的，解析器可以在扫描时提前获取元信息，从而缩短分支选择的时间。然而，它的原始实现没有很好地利用这个机制，而是**花费了大量时间使用反射获取模式的元信息**。与此同时，json-iterator 的方法是：将结构解释为逐个字段的编码和解码函数，然后将它们组装和缓存起来，最小化反射带来的性能损失。但这种方法是否一劳永逸呢？实际测试中，我们发现**随着输入的 JSON 变深、变大，json-iterator 和其他库之间的差距逐渐缩小**——甚至最终被超越： [![Scalability](https://github.com/bytedance/sonic/raw/main/docs/imgs/introduction-1.png)](https://github.com/bytedance/sonic/blob/main/docs/imgs/introduction-1.png)

原因是**该实现转化为大量接口封装和函数调用**，导致了函数调用的性能损失：

1. **调用接口涉及到对 `itab` 的动态地址获取**
2. **组装的函数无法内联**，而 Golang 的函数调用性能较差（没有寄存器传参）

#### 有没有办法避免动态组装函数的调用开销？

我们首先考虑的是类似[easyjson](https://github.com/mailru/easyjson)的代码生成。但是这会带来**模式依赖和便利性下降**。为了实现对标准库的真正插拔式替换，我们转向了另一种技术- **[JIT](https://en.wikipedia.org/wiki/Jit) （即时编译）**。因为编译后的编解码函数是一个集成的函数，它可以大大减少函数调用，同时保证灵活性。

### 为什么 Simdjson-go 速度不够快？

[SIMD](https://en.wikipedia.org/wiki/SIMD) （单指令流多数据流）是一组特殊的 CPU 指令，用于并行处理矢量化数据。目前，大多数 CPU 都支持 SIMD ，并广泛用于图像处理和大数据计算。毫无疑问，SIMD在JSON处理中很有用（整形-字符串转换，字符搜索等都是合适的场景）。我们可以看到， simdjson-go 在大型 JSON 场景 (>100KB) 下非常有竞争力。然而，对于一些很小或不规则的字符字符串， **SIMD 所需的额外加载操作将导致性能下降**。因此，我们需要考虑不同的场景，并决定哪些场景应该使用 SIMD ，哪些不应该使用（例如，长度小于16字节的字符串）。

第二个问题来自 Go 编译器本身。为了保证编译速度， **Golang 在编译阶段几乎不进行任何优化工作**也无法直接使用编译器后端，如 [LLVM](https://en.wikipedia.org/wiki/LLVM) 等进行优化。

那么，**一些关键的计算函数能否用计算效率更高的其他语言编写吗**?

C/Clang 是一种理想的编译工具（内部集成了 LLVM ）。但关键是如何将优化后的汇编嵌入到 Golang 中。

### 如何更好地使用 `Gjson` ？

我们还发现在单键查找场景中， [gjson](https://github.com/tidwall/gjson)具有巨大的优势。这是因为它的查找是通过**惰性加载机制**实现的，巧妙地跳过了传递的值，并有效的减少了许多不必要的解析。实际应用证明，在产品中充分利用这个特性确实能带来收益。但是，当涉及到多键查找时，Gjson甚至比标准库还要差，这是其跳过机制的副作用——**搜索相同路径会导致重复解析**（跳过解析也是一种轻量的解析）因此，根据实际情况准确的做出调整是关键问题。


## 开源库 sonic 技术原理
基于以上问题，我们的设计很好实现：

1. 针对编解码动态汇编的函数调用开销，我们**使用 JIT 技术在运行时组装与模式对应的字节码（汇编指令）**，最终将其以 Golang 函数的形式缓存在堆外内存上。
2. 针对大数据和小数据共存的实际场景，我们**使用预处理判断**（字符串大小、浮点数精度等）**将 SIMD 与标量指令相结合**，从而实现对实际情况的最佳适应。
3. 对于 Golang 语言编译优化的不足，我们决定**使用 C/Clang 编写和编译核心计算函数**，并且**开发了一套 [asm2asm](https://github.com/chenzhuoyu/asm2asm) 工具，将经过充分优化的 x86 汇编代码转换为 Plan9 格式**，最终加载到 Golang 运行时中。
4. 考虑到解析和跳过解析之间的速度差异很大， **惰性加载机制**当然也在我们的 AST 解析器中使用了，但**以一种更具适应性和高效性的方式来降低多键查询的开销**。

在细节上，我们进行了一些进一步的优化：

1. 由于 Golang 中的原生汇编函数不能被内联，我们发现其成本甚至超过了 C 编译器的优化所带来的改善。所以我们在 JIT 中重新实现了一组轻量级的函数调用：
    - 全局函数表+静态偏移量，用于调用指令
    - **使用寄存器传递参数**
2. `Sync.Map` 一开始被用来缓存编解码器，但是对于我们的**准静态**（读远多于写），**元素较少**（通常不足几十个）的场景，它的性能并不理想，所以我们使用开放寻址哈希和 RCU 技术重新实现了一个高性能且并发安全的缓存。

由于 JSON 业务场景复杂，指望通过单一算法来优化并不现实。于是在设计 sonic 的过程中，我们借鉴了其他领域/语言的优化思想（不仅限于 JSON），将其融合到各个处理环节中。其中较为核心的技术有三块：**JIT、lazy-load** 与 **SIMD** 。
### 为什么不使用cgo

使用cgo，可以直接用golang编译并调用C代码。import虚拟package C，并在注释中include C代码文件、声明C中实现的函数。

```text
/*
#include "hello.c"
int SayHello();
double Sum();
*/
import "C"
```

即可在golang代码中通过C包名调用上面声明的C函数。编译命令与编译只包含go文件的编译命令相同（如`go build main.go`）。

cgo也可以对C代码进行O3级别的优化。

与sonic相比，cgo的实现更加简便，也对代码进行了深度优化，似乎是一个更好的方案。但是cgo在调用c代码的时候引入了调度、切换线程栈等开销，会造成较大（有的场景中高达20多倍）的性能损耗。

### JIT

对于有 schema 的**定型编解码**场景而言，很多运算其实不需要在“运行时”执行。这里的“运行时”是指程序真正开始解析 JSON 数据的时间段。

举个例子，如果业务模型中确定了某个 JSON key 的值一定是布尔类型，那么我们就可以在序列化阶段直接输出这个对象对应的 JSON 值（‘true’或‘false’），并不需要再检查这个对象的具体类型。

sonic-JIT 的核心思想就是：**将模型解释与数据处理逻辑分离，让前者在“编译期”固定下来**。

这种思想也存在于标准库和某些第三方 JSON 库，如 json-iterator 的函数组装模式：把 Go struct 拆分解释成一个个字段类型的编解码函数，然后组装并缓存为整个对象对应的编解码器（codec），运行时再加载出来处理 JSON。但是这种实现难以避免转化成大量 interface 和 function 调用栈，随着 JSON 数据量级的增长，function-call 开销也成倍放大。只有**将模型解释逻辑真正编译出来**，实现 stack-less 的执行体，才能最大化 schema 带来的性能收益。

业界实现方式目前主要有两种：**代码生成 code-gen**（或**模版 template**）和 **即时编译 JIT**。前者的优点是库开发者实现起来相对简单，缺点是增加业务代码的维护成本和局限性，无法做到秒级热更新——这也是代码生成方式的 JSON 库受众并不广泛的原因之一。JIT 则将编译过程移到了程序的加载（或首次解析）阶段，只需要提供 JSON schema 对应的结构体类型信息，就可以一次性编译生成对应的 codec 并高效执行。

sonic-JIT 大致过程如下：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0X6sJicMXefj3x9mE5GmnRb4KI4GZKiasuCU5q1h8MRvjWibAOhwPSR2Yg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

sonic-JIT 体系

1. 初次运行时，基于 Go 反射来获取需要编译的 schema 信息；
2. 结合 JSON 编解码算法生成一套自定义的中间代码 OP codes；
3. 将 OP codes 翻译为 Plan9 汇编；
4. 使用第三方库 golang-asm 将 Plan 9 转为机器码；
5. 将生成的二进制码注入到内存 cache 中并封装为 go function；
6. 后续解析，直接根据 type ID （rtype.hash）从 cache 中加载对应的 codec 处理 JSON。

从最终实现的结果来看，sonic-JIT 生成的 codec 性能不仅好于 json-iterator，甚至超过了代码生成方式的 easyjson（见后文“性能测试”章节）。这一方面跟底层文本处理算子的优化有关（见后文“SIMD & asm2asm”章节），另一方面来自于 sonic-JIT 能控制底层 CPU 指令，在运行时建立了一套独立高效的 ABI（Application Binary Interface）体系：

- 将使用频繁的变量放到固定的寄存器上（如 JSON buffer、结构体指针），尽量避免 memory load & store；
- 自己维护变量栈（内存池），避免 Go 函数栈扩展；
- 自动生成跳转表，加速 generic decoding 的分支跳转；
- 使用寄存器传递参数（当前 Go Assembly 并未支持，见“SIMD & asm2asm”章节）。

### Lazy-load

对于大部分 Go JSON 库，泛型编解码是它们性能表现最差的场景之一，然而由于业务本身需要或业务开发者的选型不当，它往往也是被应用得最频繁的场景。

泛型编解码性能差仅仅是因为没有 schema 吗？其实不然。我们可以对比一下 C++ 的 JSON 库，如 rappidjson、simdjson，它们的解析方式都是泛型的，但性能仍然很好（simdjson 可达 2GB/s 以上）。标准库泛型解析性能差的根本原因在于**它采用了 Go 原生泛型——interface（map[string]interface{}）作为 JSON 的编解码对象**。

这其实是一种糟糕的选择：首先是数据反序列化的过程中，map 插入的开销很高；其次在数据序列化过程中，map 遍历也远不如数组高效。

回过头来看，JSON 本身就具有完整的自描述能力，如果我们用一种与 JSON AST 更贴近的数据结构来描述，不但可以让转换过程更加简单，甚至可以实现按需加载（lazy-load）——这便是 sonic-ast 的核心逻辑：**它是一种 JSON 在 Go 中的编解码对象，用 node {type, length, pointer} 表示任意一个 JSON 数据节点，并结合树与数组结构描述节点之间的层级关系**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0FHyUaCytibObOKKqycXdMmysapbE8FbkeAxZpTI9mYDrbt6ibeTWXEZw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

sonic-ast 结构示意

sonic-ast 实现了一种有状态、可伸缩的 JSON 解析过程：当使用者 get 某个 key 时，sonic 采用 skip 计算来轻量化跳过要获取的 key 之前的 json 文本；对于该 key 之后的 JSON 节点，直接不做任何的解析处理；仅使用者真正需要的 key 才完全解析（转为某种 Go 原始类型）。由于节点转换相比解析 JSON 代价小得多，在并不需要完整数据的业务场景下收益相当可观。

虽然 skip 是一种轻量的文本解析（处理 JSON 控制字符“[”、“{”等），但是使用类似 gjson 这种纯粹的 JSON 查找库时，往往会有相同路径查找导致的重复开销。

针对该问题，sonic 在对于子节点 skip 处理过程增加了一个步骤，将跳过 JSON 的 key、起始位、结束位记录下来，分配一个 Raw-JSON 类型的节点保存下来，这样二次 skip 就可以直接基于节点的 offset 进行。同时 sonic-ast 支持了节点的更新、插入和序列化，甚至支持将任意 Go types 转为节点并保存下来。

换言之，sonic-ast 可以作为一种通用的泛型数据容器替代 Go interface，在**协议转换、动态代理**等服务场景有巨大潜力。

### SIMD & asm2asm

无论是定型编解码场景还是泛型编解码场景，核心都离不开 JSON 文本的处理与计算。其中一些问题在业界已经有比较成熟高效的解决方案，如浮点数转字符串算法 Ryu，整数转字符串的查表法等，这些都被实现到 sonic 的底层文本算子中。

还有一些问题逻辑相对简单，但是可能会面对较大数量级的文本，如 JSON string 的 unquote\quote 处理、空白字符的跳过等。此时我们就需要某种技术手段来提升处理能力。SIMD 就是这样一种用于并行处理大规模数据的技术，目前大部分 CPU 已具备 SIMD 指令集（例如 Intel AVX），并且在 simdjson 中有比较成功的实践。

下面是一段 sonic 中 skip 空白字符的算法代码：
```c
#if USE_AVX2  
    // 一次比较比较32个字符  
    while (likely(nb >= 32)) {  
        // vmovd 将单个字符转成YMM  
        __m256i x = _mm256_load_si256 ((const void *)sp);  
        // vpcmpeqb 比较字符，同时为了充分利用CPU 超标量特性使用4 倍循环  
        __m256i a = _mm256_cmpeq_epi8 (x, _mm256_set1_epi8(' '));  
        __m256i b = _mm256_cmpeq_epi8 (x, _mm256_set1_epi8('\t'));  
        __m256i c = _mm256_cmpeq_epi8 (x, _mm256_set1_epi8('\n'));  
        __m256i d = _mm256_cmpeq_epi8 (x, _mm256_set1_epi8('\r'));  
        // vpor 融合4次结果  
        __m256i u = _mm256_or_si256   (a, b);  
        __m256i v = _mm256_or_si256   (c, d);  
        __m256i w = _mm256_or_si256   (u, v);  
        // vpmovmskb  将比较结果按位展示  
        if ((ms = _mm256_movemask_epi8(w)) != -1) {  
            _mm256_zeroupper();  
            // tzcnt 计算末尾零的个数N  
            return sp - ss + __builtin_ctzll(~(uint64_t)ms);  
        }  
        /* move to next block */  
        sp += 32;  
        nb -= 32;  
    }  
    /* clear upper half to avoid AVX-SSE transition penalty */  
    _mm256_zeroupper();  
#endif
```

sonic 中 strnchr() 实现（SIMD 部分）

开发者们会发现这段代码其实是用 C 语言编写的 —— 其实 sonic 中绝大多数文本处理函数都是用 C 实现的：一方面 SIMD 指令集在 C 语言下有较好的封装，实现起来较为容易；另一方面这些 C 代码通过 clang 编译能充分享受其编译优化带来的提升。为此我们开发了一套 x86 汇编转 Plan9 汇编的工具 asm2asm，将 clang 输出的汇编通过 Go Assembly 机制静态嵌入到 sonic 中。同时在 JIT 生成的 codec 中我们利用 asm2asm 工具计算好的 C 函数 PC 值，直接调用 CALL 指令跳转，从而绕过 Go Assembly 不能寄存器传参的限制，压榨最后一丝 CPU 性能。

### 其它

除了上述提到的技术外，sonic 内部还有很多的细节优化，比如使用 RCU 替换 sync.Map 提升 codec cache 的加载速度，使用内存池减少 encode buffer 的内存分配，等等。这里限于篇幅便不详细展开介绍了，感兴趣的同学可以自行搜索阅读 sonic 源码进行了解。

## 性能测试

我们以前文中的不同测试场景进行测试，得到结果如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0gaCk2VdgAN3nvdlQuZSkrRtETiazdkd1GaAKsN2xEaaCqcBHU2khDTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

小数据（400B，11 个 key，深度 3 层）

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0SGPLjjBa016SS1nMN9dXoCILibLroR6cJX9WicgKsweb61wGMzl28oiaA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

中数据（110KB，300+ key，深度 4 层）

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0sEuykFSIIsEWWDGx7ibqCQHgGzX8aqzm68Wic72a22sWByKnLNuX9sPw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

大数据（550KB，10000+ key，深度 6 层）

可以看到 sonic 在几乎所有场景下都处于领先（sonic-ast 由于直接使用了 Go Assembly 导入的 C 函数导致小数据集下有一定性能折损）

- 平均编码性能较 json-iterator 提升 240% ，平均解码性能较 json-iterator 提升 110% ；
    
- 单 key 修改能力较 sjson 提升 75% 。
    

并且在生产环境中，sonic 中也验证了良好的收益，服务高峰期占用核数减少将近三分之一：

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOiaTEXRLjJ4sDdFQGTvfP2A0ZibGPQxV6yFavpQKg3ODEgZyKkMwPRAayC3z05RZibEOknDibjibx5Tdgw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

字节某服务在 sonic 上线前后的 CPU 占用（核数）对比

## 结语

由于底层基于汇编进行开发，sonic 当前仅支持 amd64 架构下的 darwin/linux 平台 ，后续会逐步扩展到其它操作系统及架构。除此之外，我们也考虑将 sonic 在 Go 语言上的成功经验移植到不同语言及序列化协议中。目前 sonic 的 C++ 版本正在开发中，其定位是基于 sonic 核心思想及底层算子实现一套通用的高性能 JSON 编解码接口。

近日，sonic 发布了第一个大版本 v1.0.0，标志着其除了可被企业灵活用于生产环境，也正在积极响应社区需求、拥抱开源生态。我们期待 sonic 未来在使用场景和性能方面可以有更多突破，欢迎开发者们加入进来贡献 PR，一起打造业界最佳的 JSON 库！

**相关链接**

项目地址：https://github.com/bytedance/sonic

## RCU
RCU(Read-Copy Update)，是 Linux 中比较重要的一种同步机制。顾名思义就是“读，拷贝更新”，再直白点是“随意读，但更新数据的时候，需要先复制一份副本，在副本上完成修改，再一次性地替换旧数据”。这是 Linux 内核实现的一种针对“读多写少”的共享数据的同步机制。
**RCU机制解决了什么**

在RCU的实现过程中，我们主要解决以下问题：

1、在读取过程中，另外一个线程删除了一个节点。删除线程可以把这个节点从链表中移除，但它不能直接销毁这个节点，必须等到所有的读取线程读取完成以后，才进行销毁操作。RCU中把这个过程称为宽限期（Grace period）。

2、在读取过程中，另外一个线程插入了一个新节点，而读线程读到了这个节点，那么需要保证读到的这个节点是完整的。这里涉及到了发布-订阅机制（Publish-Subscribe Mechanism）。

3、保证读取链表的完整性。新增或者删除一个节点，不至于导致遍历一个链表从中间断开。但是RCU并不保证一定能读到新增的节点或者不读到要被删除的节点。

RCU(Read-Copy Update)，顾名思义就是读-拷贝修改，它是基于其原理命名的。对于被RCU保护的共享数据结构，读者不需要获得任何锁就可以访问它，但写者在访问它时首先拷贝一个副本，然后对副本进行修改，最后使用一个回调（callback）机制在适当的时机把指向原来数据的指针重新指向新的被修改的数据。那么这个“适当的时机”是怎么确定的呢？这是由内核确定的，也是我们后面讨论的重点。

## RCU原理

RCU实际上是一种改进的rwlock，读者几乎没有什么同步开销，它不需要锁，不使用原子指令，而且在除alpha的所有架构上也不需要内存栅（Memory Barrier），因此不会导致锁竞争，内存延迟以及流水线停滞。不需要锁也使得使用更容易，因为死锁问题就不需要考虑了。写者的同步开销比较大，它需要延迟数据结构的释放，复制被修改的数据结构，它也必须使用某种锁机制同步并行的其它写者的修改操作。

读者必须提供一个信号给写者以便写者能够确定数据可以被安全地释放或修改的时机。有一个专门的垃圾收集器来探测读者的信号，一旦所有的读者都已经发送信号告知它们都不在使用被RCU保护的数据结构，垃圾收集器就调用回调函数完成最后的数据释放或修改操作。

RCU与rwlock的不同之处是：它既允许多个读者同时访问被保护的数据，又允许多个读者和多个写者同时访问被保护的数据（注意：是否可以有多个写者并行访问取决于写者之间使用的同步机制），读者没有任何同步开销，而写者的同步开销则取决于使用的写者间同步机制。但RCU不能替代rwlock，因为如果写比较多时，对读者的性能提高不能弥补写者导致的损失。

读者在访问被RCU保护的共享数据期间不能被阻塞，这是RCU机制得以实现的一个基本前提，也就说当读者在引用被RCU保护的共享数据期间，读者所在的CPU不能发生上下文切换，spinlock和rwlock都需要这样的前提。写者在访问被RCU保护的共享数据时不需要和读者竞争任何锁，只有在有多于一个写者的情况下需要获得某种锁以与其他写者同步。

写者修改数据前首先拷贝一个被修改元素的副本，然后在副本上进行修改，修改完毕后它向垃圾回收器注册一个回调函数以便在适当的时机执行真正的修改操作。等待适当时机的这一时期称为grace period，而CPU发生了上下文切换称为经历一个quiescent state，grace period就是所有CPU都经历一次quiescent state所需要的等待的时间。垃圾收集器就是在grace period之后调用写者注册的回调函数来完成真正的数据修改或数据释放操作的。

要想使用好RCU，就要知道RCU的实现原理。我们拿linux 2.6.21 kernel的实现开始分析，为什么选择这个版本的实现呢？因为这个版本的实现相对较为单纯，也比较简单。当然之后内核做了不少改进，如抢占RCU、可睡眠RCU、分层RCU。但是基本思想都是类似的。所以先从简单入手。

首先，上一节我们提到，写者在访问它时首先拷贝一个副本，然后对副本进行修改，最后使用一个回调（callback）机制在适当的时机把指向原来数据的指针重新指向新的被修改的数据。而这个“适当的时机”就是所有CPU经历了一次进程切换（也就是一个grace period）。为什么这么设计？因为RCU读者的实现就是关抢占执行读取，读完了当然就可以进程切换了，也就等于是写者可以操作临界区了。

那么就自然可以想到，内核会设计两个元素，来分别表示写者被挂起的起始点，以及每cpu变量，来表示该cpu是否经过了一次进程切换(quies state)。

就是说，当写者被挂起后，
1）重置每cpu变量，值为0。
2）当某个cpu经历一次进程切换后，就将自己的变量设为1。
3）当所有的cpu变量都为1后，就可以唤醒写者了。


# Reference
https://mp.weixin.qq.com/s?spm=a2c6h.12873639.article-detail.28.2210217cVimLy0&__biz=MzI1MzYzMjE0MQ==&mid=2247491325&idx=1&sn=e8799316d55c0951b0b54b404a3d87b8&scene=21#wechat_redirect