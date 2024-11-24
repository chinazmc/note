
![](https://sponsor.segmentfault.com/lg.php?bannerid=0&campaignid=0&zoneid=25&loc=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000023351535&cb=4e7f0a21d5)

> 导语 | Elasticsearch（下文简称ES） 是当前热门的开源全文搜索引擎，利用它我们可以方便快捷搭建出搜索平台，但通用的配置还需要根据平台内容的具体情况做进一步优化，才能产生令用户满意的搜索结果。下文将介绍对 ES 搜索排名的优化实践，希望与大家一同交流。文章作者：曹毅，腾讯应用开发工程师。

## **一、引言**

虽然使用 ES 可以非常方便快速地搭建出搜索平台，但搜出来的结果往往不符合预期。因为 ES 是一个通用的全文搜索引擎，它无法理解被搜索的内容，通用的配置也无法适合所有内容的搜索。所以 ES 在搜索中的应用需要针对具体的平台做很多的优化才可以达到良好的效果。

ES 搜索结果排序是**通过 query 关键字与文档内容计算相关性评分**来实现的。想掌握相关性评分并不容易。首先 ES 对中文并不是很友好，需要安装插件与做一些预处理，其次影响相关性评分的因素比较多，这些因素可控性高，灵活性高。

下文将为大家介绍 ES 搜索排名优化上的实践经验，本篇文章示例索引数据来自一份报告文档，如下图所示：

![](https://segmentfault.com/img/remote/1460000023351539)

## **二、优化 ES Query DSL**

构建完搜索平台后，我们首先要进行 ES Query DSL 的优化。针对 ES 的优化，关键点就在于优化 Query DSL，只要 DSL 使用恰当，搜索的最终效果就会很好。

### **1. 最初使用的 multi_match**

当我们构建好索引同步好数据以后，想**比较快实现全文索引的方式就是使用 multi_match 来查询**，例如:

![](https://segmentfault.com/img/remote/1460000023351538)

这样使用非常方便和快速，并且实现了全文搜索的需求。但是搜索的结果可能不太符合预期。但是不用担心，我们可以继续往下优化。

### **2. 使用 bool 查询的 filter 增加筛选**

在应用中，我们应该避免直接让用户针对所有内容进行查询，这样会返回大量的命中结果，如果结果的排序稍微有一点出入，用户将无法获取到更精准的内容。

针对这种情况，我们可以**给内容增加一些标签、分类等筛选项**提供给用户做选择，以达到更好的结果排名。这样搜索时被 ES 引擎评分的目标结果将会变少，评分的抖动影响会更小。

实现这个功能就使用到 bool 查询的过滤器。bool 查询中提供了4个语句，must / filter / should / must_not，其中 filter / must_not 属于过滤器，must / should 属于查询器。关于过滤器，你需要知道以下两点：

-   过滤器并不计算相关性评分，因为被过滤掉的内容不会影响返回内容的排序；
-   过滤器可以使用 ES 内部的缓存，所以过滤器可以提高查询速度。

这里需要注意：虽然 must 查询像是一种正向过滤器，但是它所查询的结果将会返回并会和其他的查询一起计算相关性评分，因此无法使用缓存，与过滤器并不一样。

一般一个文档拥有多个可以被筛选的属性，例如 id、时间、标签、分类等。为了搜索的质量我们应该认真地对文档进行打标签和分类处理，因为一旦选择了过滤，即使用户的搜索关键词再匹配文档也不会被返回了。示例 Query DSL 如下：

![](https://segmentfault.com/img/remote/1460000023351540)

上面的示例中，存在一个小技巧，即使用标签的 id 来进行筛选。因为 tags 字段是text 类型的，**term 查询是精确匹配，不要将其应用到 text 类型的字段上**，如果text字段要被过滤器使用，在 mappings 中应该要使用 string 类型(它将字段映射到两个类型上，text 和 keyword )或者 keyword 类型。

### **3. 使用 match_phrase 提高搜索短语的权重**

在这个阶段，搜索的时候经常会出现**搜索结果和搜索关键词不是连续匹配的**情况。例如搜索关键词为：“2020年微信用户研究报告”，而返回的结果大多数是匹配“微信”、“用户”、“研究”、“报告”这些零散的关键词，而用户想要匹配整个短语的结果却在后面。

这并不是 ES 的 bug，在了解这种行为之前，我们需要先弄清楚 ES 是如何处理match 的？

首先 multi_match 会把多个字段的匹配转换成多个 match 查询组合，挨个对字段进行 match 查询。match 执行查询时，先把查询关键词经过 search_analyzer 设置的分析器分析，再把分析器得到的结果挨个放进 bool 查询中的 should 语句，这些 should 没有权重与顺序的差别，并且只要命中一个should 语句的文档都会被返回。转换语句如下图所示，前面是原语句，后面是转换后的语句：

![](https://segmentfault.com/img/remote/1460000023351542)

![](https://segmentfault.com/img/remote/1460000023351541)

这样就导致了有的文档只拥有查询短语中若干个词，但评分却比可以匹配整个短语的文档高的情况。那我们如何考虑词的顺序呢？先别急，我们再来看看 ES 中的倒排索引。

我们都知道倒排索引中记录了一个词到包含词文档的 ID，但倒排索引当然不会这么简单。倒排列表中记录了单词对应的文档集合，由倒排索引项组成。倒排索引项中主要包含如下信息：

-   文档ID：用于获取文档；
-   单词词频(TF)：用于相关性计算(TF-IDF，BM25)；
-   位置：记录单词在文档中的分词位置，会有多个，用于短语查询；
-   偏移：记录在文档中的开始位置与结束位置，用于高亮。

这下我们就很清楚了，**ES 专门记录了词语的位置信息用于查询，在DSL中是使用 match_phrase 查询**。match_phrase 要求必须命中所有分词，并且返回的文档命中的词也要按照查询短语的顺序，**词的间距可以使用 slop 设置**。

match_phrase 虽然帮我们解决了顺序的问题，但是它要求比较苛刻，**需要命中所有分词**。如果单独使用它来进行搜索，会发现搜索出来的结果相比 match 会大大减少，这是因为匹配若干个词的文档和匹配顺序不对的文档都没被返回。

这时候可以采用 bool 查询的 should 语句，同时使用 match 与 match_phrase 查询语句，这样相当于 match_pharse 提高了搜索短语顺序的权重，使得能够顺序匹配到的文档相关性评分更高。如下是示例DSL：

![](https://segmentfault.com/img/remote/1460000023351547)

这里有一点需要注意，在倒排索引项中 **text  类型的数组里，每个元素记录的位置是连续的**。例如某文档数据中的 tags：["腾讯CDC","京东研究院”]，“CDC” 与“京东”的位置是连续的，如果搜索 “CDC京东”，那此文档的评分将会比较高。

这种情况是不应该发生的，我们应该在设置索引mappings时，**给 tags 字段设置上 position_increment_gap ，来增加数组元素之间的位置，此位置要超过查询所使用的 slop**，例如：

![](https://segmentfault.com/img/remote/1460000023351543)

### **4. 使用 boost 调整查询语句的权重**

前文提到的搜索实现，有一个显而易见的问题：所有字段都无权重之分。根据常识我们知道，title 的权重应该高于其他字段，显然不能和其他字段是一样的得分。

**查询时可以用 boost 配置来增加权重**，不过这里设置的对象并不是某个字段，而是查询语句。设置后，查询语句的得分等于**默认得分乘以 boost**。

设置 boost 有几个需要注意的地方：

-   数据质量高的字段可以相应提高权重；
-   match_phrase 语句的权重应该高于相应字段 match 查询的权重，因为文档中按顺序匹配的短语可能数量不会太多，但是查询关键词被分词后的词语将会很多，match的得分将会比较高，则 match 的得分将会冲淡 match_phrase 的影响；
-   在 mappings 设置中，可以针对字段设置权重，查询时不用再针对字段使用 boost 设置。

具体示例 DSL 如下：

![](https://segmentfault.com/img/remote/1460000023351545)

### **5. 使用 function_score 增加更多的评分因素**

影响文档评分的还有一些因素，例如我可能会经常考虑以下问题：

-   时间越近的文档信息比较及时，对用户更有用，应该排在前面；
-   平台中热门的文档，可能用户比较喜欢，应该比其他文档好；
-   文档质量比较高的，更希望让用户看到，那些缺失标签与摘要的文档并不希望用户总是看到；
-   运营人员有时候想让用户搜到正在推广的文档；
-   ……

我们可以通过增加更多的影响报告评分的因素来实现以上场景，这些因素包括：时间、热度、质量评分、运营权重等。

这些因素有一个特点，就是在数据搭建阶段我们就能确定其权重，并且和查询关键词没有什么关系。这些文档本身就具有的权重属性我们可以认为是静态评分，需要和查询关键词来计算出的相关性评分称为动态评分，所以一个文档的最终评分应该是动态评分与静态评分的结合。

静态评分相关的属性不应该随便设置。为了给用户一个更好的体验，静态评分的影响应该具有：

-   **稳定性：**不要经常有大幅度的变动，如果大幅度变化会导致用户搜索相同的关键词过段时间出来的结果会不同；
-   **连续性：**方便我们其他的优化也能影响总评分，例如对于热度 0.1 与 1000 的文档，即使用户搜索 0.1 热度文档的匹配度为 1000 热度文档的 100 倍，但结果排名依然比不过 1000 热度的文档；
-   **区分度**：在连续稳定的情况下，应该有一定的区分度，也即分值的间隔应该合理。如果有 1000 份文档，在 1.0 分到 1.001 分之间，这其实是没有实际意义的，因为对文档排名的影响太少了。

新增加的这些因素并没有太通用的查询语句，不过 ES 提供了 function_score 来让我们自定义评分计算公式，也提供了多种类型方便我们快速应用。function_score 提供了五种类型

-   script_score，这是最灵活的方式，可以自定义算法；
-   weight，乘以一个权重数值；
-   random_score，随机分数；
-   field_value_factor，使用某个字段来影响总分数；
-   decay fucntion，包括gauss、exp、linear三种衰减函数。

因为类型比较多，下面只介绍使用较多的 filed_value_factor 与 decay function 的实际案例。

#### **（1）filed_value_factor**

热度、推荐权重等对评分的影响可以按权重相乘，刚好适合 filed_value_factor 这种类型的函数，实现如下：

![](https://segmentfault.com/img/remote/1460000023351546)

上面这段 DSL 的含义是：sqrt (1.2 * doc['likes'].value)，如果缺失了此字段则为1。missing 这里的设置有个小技巧，比如假定缺失摘要的文档为质量低的文档，可以适当降低权重，那我们可以把 missing 设置为小于1的数值，factor 填 1 即可。

#### **（2）decay function**

衰减函数是一个非常有用的函数，**它可以帮我们实现平滑过渡，使距离某个点越近的文档分数越高，越远的分数越低**。使用衰减函数很容易实现时间越近的文档得分就越高的场景。

ES提供了三个衰减函数，我们先来看一下这三种衰减函数的差别，截取官方文档展示图如下：

![](https://segmentfault.com/img/remote/1460000023351544)

-   linear，是两条线性函数，从直线和横轴相交处以外，评分都为0；
-   exp，是指数函数，先剧烈的衰减，然后缓慢衰减；
-   guass，高斯衰减是最常用的，先缓慢再剧烈再缓慢，scale相交的点附近衰减比较剧烈。

当我们想选取一定范围内的结果，或者一定范围内的结果比较重要时，例如某个时间、地域（圆形）、价格范围内，都可以使用高斯衰减函数。高斯衰减函数有4个参数可以设置

-   origin：中心点，或字段可能的最佳值，落在原点 origin 上的文档评分 _score 为满分 1.0 ；
-   scale：衰减率，即一个文档从原点 origin 下落时，评分 _score 改变的速度；
-   decay：从原点 origin 衰减到 scale 所得的评分 _score ，默认值为 0.5 ；
-   offset：以原点 origin 为中心点，为其设置一个非零的偏移量 offset 覆盖一个范围，而不只是单个原点。在范围 -offset <= origin <= +offset 内的所有评分 _score 都是 1.0 。

假定搜索引擎中三年内的文档会比较重要，三年之前的信息价值降低，就可以选择 origin 为今天，scale 为三年，decay 为 0.5，offset 为三个月，DSL 如下：

![](https://segmentfault.com/img/remote/1460000023351549)

### **6. 最终结果**

到这里我们的 Query DSL 已经优化的差不多了，现在的搜索结果终于可以让人满意了。我们来看一下最终的DSL语句示例：(示例并非实际运行的代码)

![](https://segmentfault.com/img/remote/1460000023351548)

## **三、优化相关性算法**

上文我们讨论了相关性评分应该由动态评分和静态评分结合得出，静态评分我们已经有了优化的方法，下文我们再来讨论一下动态评分的计算。

所谓动态评分，就是用户每次查询都要计算用户查询关键词与文档的相关性，更细一点来说，就是**实时计算全文搜索字段的相关性**。

ES 提供了一种非常好的方式，实现了可插拔式的配置，允许我们控制每个字段的相关性算法。在 Mappings 设置阶段，我们**可以调整 similarity 的参数并给不同的字段设置不同的 similarity 来达到调整相关性算法的目的**。ES 提供了几种可用的 similarity，我们接下来主要讨论 BM 25。

ES **默认的相关性算法就是 BM25**，它是一个**基于概率模型的词与文档的相关性算法**，可以看做是基于向量空间模型的 TF-IDF 算法的升级。ES 在7.0.0版本已经废弃了 TF-IDF 算法，完全使用 BM25 替换，BM25 与 TF-IDF 算法的对比和细节本文不描述。

先来看看 wikipedia 上 BM25 的公式：

![](https://segmentfault.com/img/remote/1460000023351551)

![](https://segmentfault.com/img/remote/1460000023351550)

一眼看上去有这么多变量，是不是觉得难以理解？不过不用担心，我们需要调整的参数其实只有两个。这些变量里除了 k1 与 b ，其余都是可以直接从文档中算出的，所以在 ES 中 BM25 公式其实就是靠调整这两个参数来影响评分。

k1 这个参数控制着词频结果在词频饱和度中的上升速度。默认值为 1.2 。值越小饱和度变化越快，值越大饱和度变化越慢。词频饱和度可以参看下面官方文档的截图，图中反应了词频对应的得分曲线，k1 控制 tf of BM25 这条曲线。

![](https://segmentfault.com/img/remote/1460000023351553)

b 这个参数控制着字段长归一值所起的作用， 0.0 会禁用归一化， 1.0 会启用完全归一化。默认值为 0.75 。

在优化 BM25 的 k1 和 b 时，我们要**根据搜索内容的特点入手，仔细分析检索的需求**。

例如在示例的索引数据中 content 字段的质量参差不齐，甚至有些文档可能会缺失此字段，但此文档对应的真实数据（可能是某文件、某视频等）质量很高，因此放入 ES 中 content 字段的长度并不能反映文档真实的情况，更不希望 content 短的文档被突出，所以我们要**弱化文档长度对评分的影响**。

根据 k1 和 b 的描述，我们将 BM25 模型中的 b 值从默认的 0.75 降低，具体降低到多少才合适，还需要进一步的尝试。这里我以调整到 0.2 为例，写出对应的 settings 和 mappings ：

![](https://segmentfault.com/img/remote/1460000023351552)

k1 和 b 的默认值适用于绝大多数文档集合，但最优值还是会因为文档集不同而有所区别，为了找到文档集合的最优值，就必须对参数进行反复修改验证。

## **四、优化的建议**

对 ES 搜索的优化应该把大部分精力花在文档数据质量提升和查询 DSL 组合调优上，需要反复尝试各种查询的组合和调整权重，在 DSL 的优化已经达到较好程度之前，尽量不要调整 similarity。

更不要在初期就引入太多的插件，例如近义词，拼音等，这样会影响你的优化，它们只是提高搜索召回率的工具，并不一定会提高准确率。**更专业的平台应该做好更专业的搜索引导与建议，而不是让用户盲目的去尝试搜索**。

搜索的调优也不能一直关注技术方面，**还要关注用户**。搜索质量的好坏是一个比较主观的评价，想要了解用户是否满意搜索结果，只能通过**监测搜索结果和用户的行为**，例如用户重复搜索的频率，翻页的频次等。

如果搜索能返回相关性较高的文档，用户应该会在第一次搜索便得到想要的内容，如果返回相关性不太好的结果，用户可能会来回点击并尝试新的搜索条件。

## **五、使用 \_explain 做 bad case 分析**

到这里，我们的搜索排名优化就告一段落，但可能时不时还会有一些用户反馈搜索的结果不准确。虽然有可能是用户自身不会使用搜索引擎（这里应该从产品上引导用户写出更好的查询关键词），但更多时候应该还是排名优化没有做好。

所以如果发现了 bad case，我们应该记录这些 bad case ，从这些问题入手仔细分析问题出在哪里，然后慢慢优化。

分析 bad case 可以使用 ES 提供的 \_explain 查询 api，它会返回我们使用某 DSL 查到某文档的得分细节，通过这些细节，我们就能知道产生 bad case 的原因，然后有针对性的下手优化了。

## **六、结语**

ES 是一个通用型的全文检索引擎，如果想用 ES 搭建一个专业的搜索平台，必须要经过搜索调优才可以达到可用的状态。本文总结了基于 ES 的搜索优化，主要是优化 DSL 与相关性计算，希望读者可以从中学习到有用的知识。

![](https://zhuanlan.zhihu.com/p/565206413)

# Es搜索优化（五）-自动补齐/自动推荐/荐词

## 1. what

举例，我们使用Google的时候，Google会提示我们  

![](https://pic1.zhimg.com/80/v2-038f7afe9d857b9c2c2d6d681688c4ac_720w.webp)

image.png

  

-   可以看到这里Google自动提示分为两种
-   第一种：我之前有搜索过的历史记录
-   第二种：根据目前各种词的出现频率进行了展示（不排除里面有我被打上的tag进行了运算）

  

## 2. why

-   为什么要怎么做？
-   可以节省用户输入时间，用户体验++
-   为什么Google可以做到这样？
-   有记录我们的搜索历史记录
-   有记录其他用Google搜索引擎的人的搜索记录，进行综合分析
-   有针对性的进行了数据存储优化，搜索算法优化，使得可以接近即时的自动提示

## 3. how

  

### 3.1 基础分析

-   如果普通的公司做类似的自动补齐，我们可以怎么去做？
-   第一种：前端来做，跨域，借用[第三方的自动提示API](https://link.zhihu.com/?target=https%3A//www.iaspnetcore.com/blog/blogpost/5a8efc68f5eba4276034179d)
-   第二种：后端来做，利用业务数据库中的数据，提供实时提示
-   **第一种**

-   目前不考虑第一种，因为是用于搜索。因为提示和业务数据无关，而是来自第三方。所以会出现，用户输入A，我们自动补齐ABC这个关键词，但实际上啥也搜不出来

-   **第二种**：

-   mysql：直接排除，因为感觉用mysql做这个，数据量少，那做了没意思，数据量多，mysql性能也跟不上
-   redis：可以通过redis的zset数据结构来进行提示 -> 3.1
-   elasticsearch：可以通过elasticsearch来进行提示 - > 3.2

  

### 3.1 用redis实现

-   数据结构：用 zset 进行存储，member为热词，score统一为0，这样的话，默认排序就会按照member的顺序进行排序
-   插入数据如下

![](https://pic4.zhimg.com/80/v2-3568e853ab2c438d43dd12b1cbbd48f7_720w.webp)

image.png

  

-   现在触发一次自动补齐，后端需要做的操作

1.  接受到用户输入的 关键词 ，查看是否是否存在与此 zset 中
2.  如果关键词存在，则直接获取下标，如果不存在，则插入后，获取到下标
3.  根据所获得的下标，然后进行获取此下标后面N个关键词
4.  取出数据展示如下

![](https://pic2.zhimg.com/80/v2-265c56c47450e5439ec8de7139b0bb35_720w.webp)

image.png

  

-   优点：实现简单，同时redis可以提供较高的性能
-   缺点：

-   没有利用到业务数据库中的词频等数据
-   算法太过于简单粗暴，按道理可以通过树或者字符串匹配算法来提高查找效率和存储效率
-   自动补齐可能补齐不相关的数据，需要特殊处理
-   不支持热点数据，需要另外一个zset进行记录，然后对取出来的数据进行取交集
-   不支持纠错功能（因为不是存储的拼音）

### 3.2 用elasticsearch实现

Suggesters API 来实现这个功能

1.  Term Suggester（纠错补全，输入错误的情况下补全正确的单词）
2.  这个建议是基于编辑距离，基于已经在数据库中的数据进行分词，然后分割出来术语
3.  所有出来的结果有两种排序方式

1.  第一种（score）：首先按照分数，然后是文档出现的频率，然后是被搜索词本身
2.  第二种（frequency）： 首先按照文档出现的频率，然后是分数，然后是术语本身

  

1.  **Phrase Suggester**（自动补全短语，输入一个单词补全整个短语）
2.  这个建议是基于编辑距离，基于已经在数据库中的数据进行分词，然后分割出来术语，但是会根据字符串之间的距离再次提示短语
3.  同时具有纠错功能
4.  **Completion Suggester(**完成补全单词，输出如前半部分，补全整个单词）
5.  **Context Suggester（上下文补全，可以基于Completion Suggester的基础上做）**
6.  可以在插入数据的时候，数据的字段打上特定的标签
7.  然后做sugget搜索的时候，加一个搜索条件，这样的话，就会从特点的数据中进行搜索
8.  标签支持两种：category 和 geo

1.  category：就一个或者多个字符串进行标记，比如对于食物，我们可以给一个类型为category 的标签，字段名是place_type，然后字段值是resturant，如果命中了对应的一个或者多个place_type，则可以提升分数
2.  geo： 地理上下文可以在索引时将一个或多个地理点与文档字段相关联。在查询时，如果建议在特定地理位置的一定距离内，则可以过滤和提升分数

10.  **我们前期选择 Completion Suggester 因为这个性能最好，他将数据保存在内存中的有限状态转移机中（FST）**
11.  后期如果需要优化，可以结合 Term Suggester ，Phrase Suggester， Context Suggester进行优化

### 3.2.1 建立对应的index（completion 实现）

```sql
curl -XPUT "localhost:9200/test_index?pretty=true" -H 'content-type: application/json' --data '
{
    "settings": {
        "number_of_shards": 1
    },
    "mappings": {
        "properties": {
            "id": {
                "type": "keyword"
            },
            "title": {
                "type": "completion",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_smart"
            },
            "content": {
                "type": "completion",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_smart"
            },
            "status": {
                "type": "keyword"
            }
        }
    }
}
'

curl -XPOST "localhost:9200/test_index/_doc/?pretty=true" -H 'content-type: application/json' --data '
{
    "id": 1,
    "title": "春节碰到自己的前男友怎么办？",
    "content": "如题",
    "status": "1"
}
'
curl -XPOST "localhost:9200/test_index/_doc/?pretty=true" -H 'content-type: application/json' --data '
{
    "id": 1,
    "title": "春节碰到自己的前女友怎么办？",
    "content": "如题",
    "status": "1"
}
'
curl -XPOST "localhost:9200/test_index/_doc/?pretty=true" -H 'content-type: application/json' --data '
{
    "id": 1,
    "title": "春节碰到自己的老师怎么办？",
    "content": "如题",
    "status": "1"
}
'
curl -XPOST "localhost:9200/test_index/_doc/?pretty=true" -H 'content-type: application/json' --data '
{
    "id": 1,
    "title": "春节碰到自己的老师怎么办？我应该打招呼吗？",
    "content": "如题",
    "status": "1"
}
'
```

  

### 3.2.2 进行相应的查询

-   最简单的查询
-   skip_duplicates 跳过返回的重复数据

```sql
curl -XPOST "localhost:9200/test_index/_doc/_search?pretty=true" -H 'content-type: application/json' --data '
{
    "suggest": {
        "suggest_1": {
            "text": "春节碰",
            "completion": {
                "field": "title",
        "skip_duplicates": true
            }
        }
    }
}
'

## response

{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "suggest" : {
    "suggest_1" : [
      {
        "text" : "春节碰",
        "offset" : 0,
        "length" : 3,
        "options" : [
          {
            "text" : "春节碰到自己的前女友怎么办？",
            "_index" : "test_index",
            "_type" : "_doc",
            "_id" : "eOZ-bX8BVW9cjfXV7UG_",
            "_score" : 1.0,
            "_source" : {
              "id" : 1,
              "title" : "春节碰到自己的前女友怎么办？",
              "content" : "如题",
              "status" : "1"
            }
          },
          {
            "text" : "春节碰到自己的前男友怎么办？",
            "_index" : "test_index",
            "_type" : "_doc",
            "_id" : "d-Z-bX8BVW9cjfXV7UGS",
            "_score" : 1.0,
            "_source" : {
              "id" : 1,
              "title" : "春节碰到自己的前男友怎么办？",
              "content" : "如题",
              "status" : "1"
            }
          },
          {
            "text" : "春节碰到自己的老师怎么办？",
            "_index" : "test_index",
            "_type" : "_doc",
            "_id" : "eeZ-bX8BVW9cjfXV7UHr",
            "_score" : 1.0,
            "_source" : {
              "id" : 1,
              "title" : "春节碰到自己的老师怎么办？",
              "content" : "如题",
              "status" : "1"
            }
          },
          {
            "text" : "春节碰到自己的老师怎么办？我应该打招呼吗？",
            "_index" : "test_index",
            "_type" : "_doc",
            "_id" : "euZ-bX8BVW9cjfXV7kEg",
            "_score" : 1.0,
            "_source" : {
              "id" : 1,
              "title" : "春节碰到自己的老师怎么办？我应该打招呼吗？",
              "content" : "如题",
              "status" : "1"
            }
          }
        ]
      }
    ]
  }
}
```

  

### 3.3.2 match phrase prefix query 实现

  

![](https://pic1.zhimg.com/80/v2-e3cd054e44f93a8895445ab1ab57873c_720w.webp)

image.png

# 通过某瓣真实案例看Elasticsearch优化

### 前言

我瓣用户产品中有多个核心场景都在使用 Elasticsearch (以下简称 ES)，主要用在全文搜索、聚合计算等方面。ES 相关的功能开发者主要是我，我最早是 2016 年在 [选影视](https://movie.douban.com/tag/) 的功能上开始使用 ES 做结构化搜索，这个工具给运营同学和资深用户提供了按各种复杂条件找电影的，可以说是最好的途径。

这个项目为多个核心产品功能提供原始数据，这几年它历经多次迭代和功能完善，不过最近它出了一点问题，我觉得解决问题的过程很值得写篇文章分享一下，所以就有了此文。

为了让大家能更理解下面的内容，先介绍下「选影视」存储在 ES 里面的文档部分字段:

class Subject(DocType):
    title = Text()
    genres = Text()
    countries = Text()
    standard_tags = Text()
    ...

每个文档的`_id`是条目 ID，也包含很多条目属性字段 (如上述列出的标题、类型、地区、标签，以及未列出的一些字段)，其中标签 (standard_tags) 字段是非常重要的，正是由于这个字段的数据，可以基于 ES 搜索包含某个 (些) 标签条目 ID 列表。

### 问题

8 月 28 日 (下图中的最高峰那天) 平台同事 @我，说发现这个项目最近服务调用慢且有大量超时，由于项目是一个微服务，所以经常触发服务调用的熔断，希望解决。

这个项目由于最近改动并不大，我已经有一段时间没有特别关注了，赶紧打开对应的 Sentry 页面，通过一个 Issue 页面找到问题原因:

> 某个 API 请求 ES 很容易造成请求超时 (抛 ConnectionTimeout)

下面是按天统计的事件总数的趋势图:

![图一](https://user-images.githubusercontent.com/841395/65648971-3ba4f400-e037-11e9-9483-0b5116e76a43.png)

设置的超时间隔是 2 秒：这已经是 API 请求里面非常宽容的阈值了，事实上在之前的使用中大部分对 ES 的 API 请求在几毫秒到几十毫秒之间，鲜有超时问题。

PS: 这个图下面还会出现多次，会分别展示对应时间点下每天事件的总数范围。

很快和对应开发同事确认，造成问题的接口是给新版本 APP 里面的新功能提供的，我们发现问题时这个超时事件已经达到每天 4-6w+，虽然看起来量倒不算大，但是我依然觉得需要快速解决它:

-   已经影响了服务质量
-   功能比较隐蔽，如果不是主动用这个功能是不会触发 API 请求的，我体验了下对应功能确实很容易出现点开页面还没加载出来或者干脆窗口空白的情况，这太影响用户体验了
-   这个超时量会随着新版本 APP 的装机量不断提高
-   这些慢查询给 ES 带来压力，也影响了其他正常查询请求

回到问题，这个特别慢的请求是做一个聚合计算，看一下请求的 body:

```json
{
  'aggs': {
    'by_standard_tags': {
      'terms': {
        'field': 'standard_tags.keyword',
        'size': 100
      }
    }
  },
  'query': {
    'bool': {
      'must': [{'ids': {'values': IDS}}]
    }
  }
}
```

其中 IDS 是条目 ID 列表，这个请求是让 ES 聚合这些条目包含的标签总数。举个例子，ID 为 1 的条目有「剧情 / 犯罪 / 动作」三个标签；ID 为 2 的条目有「喜剧 / 剧情 / 爱情」三个标签，那么 IDS 为[1, 2] 时，返回的内容中「剧情」为 2 (2 个条目都有这个标签)，其他的标签都是 1 (只有一个条目包含)

OK，现在事情很明朗了，我们开始解决超时的问题吧。

这个接口的代码我当时一眼看去并没有发现问题，那马上想到的就是缓存和减少调用这 2 条路，但是很遗憾行不通:

-   缓存。IDS 是用户看过 / 想看电影的条目列表，每个人都不同。用户访问和用户兴趣相关，且频率不高，由于用户量庞大不值得为了这么个小功能就主动缓存并添加一个好的更新缓存的机制，成本太高甚至效果会更差
-   减少调用。目前的调用已经是按需请求，没有找到明显的可以优化调用量的地方

OK，既然「绕」不了了，就直面吧，我继续尝试其他解决超时问题的方法

### 优化聚合计算请求

这算是我的个人性格，接下来我第一个想法的思路就是优化这个聚合请求的写法。其实在 5 年前我的那个 [《Python 高级编程》](http://dongweiming.github.io/Expert-Python/#70) 的 PPT 里面就说过:

> 1.  在合适的地方用合适的技巧
> 2.  不是它不好，而是你没有用好

这一直算是我的技术格言吧，无论是不熟悉的还是熟知的内容，我都会保持敬畏。出了问题会首先考虑是不是我没有用对用好。就拿上面的这个 body 来说，其实做 2 件事:

1.  限定要查询的 `_id` 范围 (in IDS)
2.  聚合查找 `standard_tags` 字段中的标签数据，返回匹配标签数量最多的 100 个标签和包含的条目数量

那怎么优化呢？首先我试了一下改用`Term Query`代替`IDs Query`。`IDs Query`的写法是我开始用的，之后其他同学有这样的需求就按着我这种写法来了，我怀疑是这种查询语句的问题，改成下面这样的效果:

```json
{
  'aggs': {
    'by_standard_tags': {
      'terms': {                                                                                                 'field': 'standard_tags.keyword',
        'size': 100
      }
    }
  },
  'query': {
    'bool': {
      'must': [{'terms': {'_id': IDS}}]
    }
  }
}
```

上线后发现修改对超时没有帮助 😢，说明`IDs Query`的写法没有问题。

#### 优化点 1

这让我一时间没有了头绪，我开始搜索一些「优化 Elasticsearch」相关的技术博文和开源书籍，希望从中找找灵感。然后就在 Elastic 社区找到了一个 [query+aggs 查询性能问题](https://elasticsearch.cn/question/1008) 的帖子，其作者也遇到了使用聚合后查询非常慢的问题，@kennywu76 给出了一个方案: 「在每一层 terms aggregation 内部加一个 {"execution_hint":"map"}」。在评论区 @kennywu76 也给了详细的解释，我认为算是全网最好的解释了，转发一下:

> Terms aggregation 默认的计算方式并非直观感觉上的先查询，然后在查询结果上直接做聚合。
> 
> ES 假定用户需要聚合的数据集是海量的，如果将查询结果全部读取回来放到内存里计算，内存消耗会非常大。因此 ES 利用了一种叫做 global ordinals 的数据结构来对聚合的字段来做 bucket 分配，这个 ordinals 用有序的数值来代表字段里唯一的一个字符串，因此为每个 ordinals 值分配一个 bucket 就等同于为每个唯一的 term 分配了 bucket。 之后遍历查询结果的时候，可以将结果映射到各个 bucket 里，就可以很快的统计出每个 bucket 里的文档数了。
> 
> 这种计算方式主要开销在构建 global ordinals 和分配 bucket 上，如果索引包含的原始文档非常多，查询结果包含的文档也很多，那么默认的这种计算方式是内存消耗最小，速度最快的。
> 
> 如果指定 execution_hint:map 则会更改聚合执行的方式，这种方式不需要构造 global ordinals，而是直接将查询结果拿回来在内存里构造一个 map 来计算，因此在查询结果集很小的情况下会显著的比 global ordinals 快。
> 
> 要注意的是这中间有一个平衡点，当结果集大到一定程度的时候，map 的内存开销带来的代价可能就抵消了构造 global ordinals 的开销，从而比 global ordinals 更慢，所以需要根据实际情况测试对比一下才能找好平衡点。

对于我们这个场景，IDS 量级比较小所以查询结果集很小，可以改用`execution_hint:map`这种方式:

```json
{
  'aggs': {
    'by_standard_tags': {
      'terms': {
        'execution_hint': 'map',
        'field': 'standard_tags.keyword',
        'size': 100
      }
    }
  }
}
```

#### 优化点 2

优化过程里我突然想起了官网对于`Filter or Query`的优化意见 (详见延伸阅读链接 1)，文中说:

> 过滤查询（Filtering queries）只是简单的检查包含或者排除，这就使得计算起来非常快。考虑到至少有一个过滤查询（filtering query）的结果是 “稀少的”（很少匹配的文档），并且经常使用不评分查询（non-scoring queries），结果会被缓存到内存中以便快速读取，所以有各种各样的手段来优化查询结果。
> 
> 相反，评分查询（scoring queries）不仅仅要找出 匹配的文档，还要计算每个匹配文档的相关性，计算相关性使得它们比不评分查询费力的多。同时，查询结果并不缓存。
> 
> 多亏倒排索引（inverted index），一个简单的评分查询在匹配少量文档时可能与一个涵盖百万文档的 filter 表现的一样好，甚至会更好。但是在一般情况下，一个 filter 会比一个评分的 query 性能更优异，并且每次都表现的很稳定。
> 
> 过滤（filtering）的目标是减少那些需要通过评分查询（scoring queries）进行检查的文档。

这段话之前我也看过，但是这次再看发现了一个可优化的点，就是用`Filter`替代`Query`，因为我们想要的只是聚合结果，完全不需要做评分查询。请求 body 就改成了这样:

```json
{
  'aggs': {
    'by_standard_tags': {
      'terms': {
        'execution_hint': 'map',
        'field': 'standard_tags.keyword',
        'size': 100
      }
    }
  },
 'query': {
   'bool': {
     'filter': [{'ids': {'values': IDS}}]
    }
  }
}
```

#### 优化点 3

正是上面的这段对于`Filter or Query`的优化意见，让我也注意到一个词:「不评分查询」，前面的查询结果其实会返回传入的 IDS 列表的对应文档结果，但是我们根本不需要，所以可以不返回 source:

```json
{
  'aggs': {
    'by_standard_tags': {
      'terms': {
        'execution_hint': 'map',
        'field': 'standard_tags.keyword',
        'size': 100
      }
    }
  },
 'query': {
   'bool': {
     'filter': [{'ids': {'value': IDS}}]
    }
  },
  '_source': False
}
```

#### 优化结果

基于上面三点优化。上线后超时问题得到了很大缓解:

![图二](https://user-images.githubusercontent.com/841395/65648972-3ba4f400-e037-11e9-8498-5c92038f08fd.png)

优化效果比较显著:

1.  超时数降到了之前的约 1/4
2.  项目总超时 (还有其他查询引起的超时) 降到了约之前的 1/5
3.  找了一些 BadCase 对比效果，优化后请求耗时最差降到之前的一半，约 1/3 的请求的时间重回小于 100ms 级别

虽然已经不会触发平台的熔断了，但是超时事件总量依然很大，需要进一步优化。

### 优化分片数

我继续保持怀疑的态度寻找优化方案，在我的认知里面，像 Elasticsearch 这样成熟的、广受关注和欢迎的项目经过多年的发展，我不相信一个很常见的聚合查询就能引起集群负载的波动，也不相信它能引起这么大量的请求超时，所以我还在怀疑问题出在使用姿势上。由于我不熟悉 Java 语言，在短时间不具备阅读 ES 源码的能力，所以开始希望通过别人的文章中获得灵感。

在看到《Mastering Elasticsearch》中的「选择恰当的分片数量和分片副本数量」(延伸阅读链接 4) 这一节后，我赶紧看了下项目用的这个索引的分片情况 (通过`http http://ES_URL/_cat/shards/INDEX_NAME`)，并和萨 (sa) 确认了一下。我瓣一开始用的 ES 版本比较低 - 2.3，之后升级到当时最新的 6.4，ES 在 6.X 及之前的版本默认索引分片数为 5 (Primary Shards)、副本数为 1 (Replica，当一个节点的主分片丢失，ES 可以把任意一个可用的分片副本推举为主分片)，从 ES7.0 开始调整为默认索引分片数为 1、副本数为 1 (详见延伸阅读链接 2 和链接 3)。而我瓣为了数据安全，默认每个索引分片数为 5、副本数为 2，也就是这个索引一共有 15 个分片（总分片数 = 主分片数*(副分片数 + 1))。

ES 默认的分片配置并不适用于所有业务场景，那么分片数应该怎么安排呢？延伸阅读链接 5 是 ES 官方博客中的一篇叫做「Elasticsearch 究竟要设置多少分片数？」的文章，其中有这么 2 段内容:

> Elasticsearch 中的数据组织成索引。每一个索引由一个或多个分片组成。每个分片是 Luncene 索引的一个实例，你可以把实例理解成自管理的搜索引擎，用于在 Elasticsearch 集群中对一部分数据进行索引和处理查询。
> 
> 构建 Elasticsearch 集群的初期如果集群分片设置不合理，可能在项目的中后期就会出现性能问题。

通过分片，ES 把数据放在不同节点上，这样可以存储超过单节点容量的数据。而副本分片数的增加可以提高搜索的吞吐量 (主分片与副本都能处理查询请求，ES 会自动对搜索请求进行负载均衡)。在《Mastering Elasticsearch》里面还提到了路由 (routing) 功能 (延伸阅读链接 6)，ES 在写入文档时，文档会通过一个公式路由到一个索引中的一个分片上。默认的选择分片公式如下：

shard_num = hash(\_routing) % num_primary_shards

`_routing`字段的取值默`_id`字段。如果不指定路由，在查询 / 聚合时就需要由 ES 协调节点搜集到每个分片 (每个主分片或者其副本上查询结果，再将查询的结果进行排序然后返回：在我们这个场景，每次要从 5 个分片上聚合。

**如果能够确定文档会被映射到哪个 (些) 分片，可以只在对应的一 (多) 个分片上执行查询命令，不用全局的搜索**。我实际的试了下，用 routing 确实快了很多。所以看到这里我第一个感觉是分片多了会带来路由问题，前面说了，「每个分片是自管理的，对一部分数据进行索引和处理查询」，查询和聚合都需要在最后把结果搜集起来。那可不可以就把数据放在一个分片上，这样就没有路由的问题了？另外 ES7.0 默认的分片方案也是朝着这个方面走的，所以我觉得方案调整应该是经过一段时间的实践，考虑到大部分场景下`1 Shard + 1 Replica`这样的方案是更高的选择。

Ok, 现在优化的目的就是选择**合适的主分片数 + 合适的副本分片数**

要改善现在面临的问题，考虑本业务的数据量，分片坏掉的概率和数据安全等因素，我直观的感受就是我瓣的索引分片数太多了。但由于索引创建好后，主分片数量不可修改，只可以修改副本分片数量。而要修改主分片数量，只能重建或者建新的索引，代价比较大。

无论是官方还是一些大厂相关文章都没有对于分片数做出完美的公式，默认的不一定是最好的，但什么样的组合还是需要实际测试，所以我尝试了多个分片方案，如下:

![[Pasted image 20230424162536.png]]

> Note: 这里副本分片数没有大于 2 的方案是所用到的 ES 集群节点规模所限。

为了不影响现有业务，用的都是新索引，这么创建:

```json
curl -XPUT "http://ES_URL/INDEXNAME/" -H 'Content-Type: application/json' -d '{
    "settings" : {
        "index" : {
            "number_of_shards" : 1,
            "number_of_replicas" : 1
        }
    }
}'
```

在数据上我使用 [elasticsearch-dump](https://github.com/taskrabbit/elasticsearch-dump) 把旧索引的中数据灌到新索引，另外一方面在新索引的因业务逻辑上订阅 Kafka 消息同步对索引数据的修改。为了让写入更快，我还在灌数据过程中关闭了副本和索引刷新:

curl -XPUT "http://ES_URL/INDEXNAME/_settings" -H 'Content-Type: application/json' -d '{ "index" : { "refresh_interval" : "-1", "number_of_replicas": 0 } }'

#### 优化结果

上述分片方案的尝试并不是按表格顺序来做的，而且一开始我的理解是分片太多，所以最初的主要目的是要减少分片数。我一开始是在当前的主分片方案上尝试`5 Shards + 1 Replica`，也就是减少副本分片数:

![图三](https://user-images.githubusercontent.com/841395/65648970-3b0c5d80-e037-11e9-9cf9-341ea992ed1a.png)

就是图中 9 月 2 日这天，超时降得很明显，这给我带来了非常大的信心，说明我的方向是对的。当时由于数据的问题，后来迁回了原来的索引一直到 9 月 4 日。

刚才提到，官方从 7.0 开始改为默认`1 Shard + 1 Replica`这样的方案，所以接下来我新建了一个`1 Shard + 2 Replicas`的新索引，9 月 4 日上线，当时效果也非常好。

![图四](https://user-images.githubusercontent.com/841395/65648973-3c3d8a80-e037-11e9-911a-36f1a3a1616e.png)

但是通过上图可以看到当天超时量又涨起来了，其实这是我犯的一个错误，早上测试`1 Shard + 2 Replica`观察了一段时间效果非常好，我认为还可以继续降分片数以提高「路由效率」，所以调整成了`1 Shard + 1 Replica`，我当时觉得毫无疑问效果会更好，然后下午就请假了... 结果过了 2 个小时发现超时涨的非常厉害，就回滚了。

不过到这里，可以感受到官方默认的方案对于我们这个例子是不可取的：如果当时选择`1 Shard + 1 Replica`运行满一天，我相信超时量将远高于之前 8 月 28 日最高峰的超时量。我觉得造成这个问题是由于分片数太少了，2 个分片扛不住这样的吞吐量。

在接下来的一段时间里面在准备好数据后。我分别尝试了上述提到的主分片小于 5 的各种组合，结果非常反直觉:

> 超时情况在 `1 Shard + 2 Replicas` 和 `5 Shard + 1 Replica` 这 2 个方案下表现是最好的，其他的方案的效果都很差。

为什么说反直觉呢？我本来认为:

-   考虑请求 ES 集群带来的压力，在一定分片数范围内增加主分片能提高吞吐量，由于路由效率超过一定阈值应该会起反作用
-   副本分片数多的副作用只是硬盘空间的「浪费」，但是能对查询效率有帮助，所以可以在一定范围内增加副本

在《eBay 的 Elasticsearch 性能调优实践》(延伸阅读链接 7) 中有「搜索性能和副本数之间的关系」和「搜索性能和分片数量之间的关系」的 2 张图表，支持了我的自觉:

-   分片数增加的过程中，开始时搜索吞吐量增大 (响应时间减少)，但随着分片数量的增加，搜索吞吐量减小 (响应时间增加)
-   搜索吞吐量几乎与副本数量成线性关系

现在的测试结果和预想对不上，尤其是`5 Shard + 1 Replica`和`5 Shard + 2 Replicas`(最初的方案) 怎么效果差这么多？

我继续搜索，找到官方对副本数的建议和问题解释 (延伸阅读链接 7):

> Which setup is going to perform best in terms of search performance? Usually, the setup that has fewer shards per node in total will perform better. The reason for that is that it gives a greater share of the available filesystem cache to each shard, and the filesystem cache is probably Elasticsearch’s number 1 performance factor.
> 
> So what is the right number of replicas? If you have a cluster that has num_nodes nodes, num_primaries primary shards in total and if you want to be able to cope with max_failures node failures at once at most, then the right number of replicas for you is max(max_failures, ceil(num_nodes / num_primaries) - 1).

也就是说，当我能接受的`max_failures`为 1、`num_nodes`为 3:

```
In : from math import ceil

In : max(1, ceil(3 / 5) - 1)  # num_primaries = 5
Out: 1 # 5 Shard + 1 Replica

In : max(1, ceil(3 / 1) - 1)
Out: 2 # 1 Shard + 2 Replicas  # num_primaries = 1

In : max(1, ceil(3 / 6) - 1)  # num_primaries = 6
Out: 1

In : max(1, ceil(3 / 9) - 1)  # num_primaries = 9
Out: 1
```

大家可以看图示，从 9 月 4 日到 9 月 19 日每日的超时量大部分在 100 - 300，也有几天达到了 4000+，较之前的超时量也可说降到了之前的 1%。每天几百的量级已经很少了:

![图五](https://user-images.githubusercontent.com/841395/65648974-3cd62100-e037-11e9-9eb9-b734b1648c14.png)

那么是否可以继续优化呢？我找萨跑了下`Slow Log`想分析这些慢的请求 body 的特点，结果发现这些请求并没有什 么特殊性。我又想如果选「大于 5 个主分片」这种反直觉的方案，也就是让分片数变的更多会怎么样呢？所以，我试了`6 Shard`和`9 Shards`，可以看最近几天:

![图六](https://user-images.githubusercontent.com/841395/65648977-3cd62100-e037-11e9-9953-b38d460c3b36.png)

结论是大于 5 分片的全部方案效果都不错。最好的是`9 Shards + 1 Replica`，每天超时数小于 30 之间:

![图七](https://user-images.githubusercontent.com/841395/65648978-3cd62100-e037-11e9-9941-cf6a5278161c.png)

这些超时都发生下凌晨各服务定期任务对这个服务产生大量大量引起的，不会影响用户体验。找调用方确认了下，有重试机制，所以到现在，我们的优化任务告一个段落了。


还有另外的优化方案， term 语句的统计结果，本身就是 " 大概 "，而不是 “精确”。它包括 2 个参数：

size 参数规定了最后返回的 term 个数 (默认是 10 个)，此处你设置为 100 个。

聚合的字段可能存在一些频率很低的词条，如果这些词条数目比例很大，那么就会造成很多不必要的计算。 min_doc_count 可以规定最小的文档数目，只有满足这个参数要求的个数的词条才会被记录返回。

"terms": { "field": "standard_tags.keyword", "size": 100, "min_doc_count": 50}

可以调整 min_doc_count 大小，来调节 term 结果集的大小，从这个方面也可以优化 ES 查询的耗时。

我们最近遇到跟你类似的需求，是从 ES 查询查询每个 IP 访问的 url 条数，结果这个 IP 在一段时间内访问 3w 多个 uri，而多数 uri 是无效的，ES 查询结果浪费了大量时间。

而我们优化 term 查询后，调整合理的 size 参数与 min_doc_count 参数， 查询耗时得到了极大改善。

优化 size 参数与 min_doc_count 参数，也是一个不错的思路

### 后记

还是那句话:

> 不是它不好，而是你没有用好

通过这个带着问题做优化的案例，让我对 ES 有了更深入的了解。我获得的经验是:

-   默认的分片方案不一定合适，具体的分片方案应该根据业务场景具体实验
-   如官网所说「副本可能有助于提高吞吐量，但并不总是如此」，主要是由于节点数少儿带来的文件系统缓存性能问题，所以副本数不是越多越好，还是尽量按照官方推荐的来
-   主分片数在一定范围内越多越好。我之前只觉得路由效率的问题，但是另外一个角度，分片多那么每个分片上的数据就变少了，查询和聚合要更快。

这次优化是从开发者有权限的地方去找优化思路的，没有考虑服务器资源和 ES 配置方面的优化方向，相信也会有收获。

另外本来还准备了「 [使用 preference 优化缓存利用率](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html#request-body-search-preference) 」、「 [6.0 新增的 Index Sorting](https://www.elastic.co/cn/blog/index-sorting-elasticsearch-6-0) 」、「 [直接路由](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html) 」等多个思路来优化。之后有时间我准备调低现在超时的阈值，相信到时候都能用上。

# Reference
https://zhuanlan.zhihu.com/p/565206413
https://www.dongwm.com/post/elasticsearch-performance-tuning-practice-at-douban/#%E5%90%8E%E8%AE%B0