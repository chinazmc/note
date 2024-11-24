# 磁盘使用量优化

优化磁盘使用量与建立索引时的映射参数和索引元数据字段密切相关，在介绍具体的优化措施之前，我们先介绍这两方面的基础知识。
## 预备知识
### 元数据字段
每个文档都有与其相关的元数据，比如_index、\_type 和_id。当创建映射类型时，可以定制其中一些元数据字段。下面列出了与本文相关的元数据字段，完整的介绍请参考官方手册：https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-fields.html。

· \_source：原始的JSON文档数据。
· \_all：索引所有其他字段值的一种通用字段，这个字段中包含了所有其他字段的值。

允许在搜索的时候不指定特定的字段名，意味着“从全部字段中搜索”，例如：
http://localhost:9200/website/_search?q=keyword

\_all字段是一个全文字段，有自己的分析器。从ES6.0开始该字段被禁用。之前的版本默认启用，但字段的store属性为false，因此它不能被查询后取回显示。

###  索引映射参数
索引创建时可以设置很多映射参数，各种映射参数的详细说明可参考官方手册：https://www.elastic.co/guide/en/elasticsearch/reference/master/mapping-params.html，这里只介绍与本文相关的参数。
· index：控制字段值是否被索引。它可以设置为true或false，默认为true。未被索引的字段不会被查询到，但是可以聚合。除非禁用doc_values。
· doc values：默认情况下，大多数字段都被索引，这使得它们可以搜索。倒排索引根据term找到文档列表，然后获取文档原始内容。但是排序和聚合，以及从脚本中访问某个字段值，需要不同的数据访问模式，它们不仅需要根据term找到文档，还要获取文档中字段的值。这些值需要单独存储。doc_values 就是用来存储这些字段值的。它是一种存储在磁盘上的列式存储，在文档索引时构建，这使得上述数据访问模式成为可能。它们以面向列的方式存储与_source相同的值，这使得排序和聚合效率更高。几乎所有字段类型都支持doc_values，但被分析（analyzed）的字符串字段除外（即text类型字符串）。doc_values默认启用。
· store：默认情况下，字段值会被索引使它们能搜索，但它们不会被存储（stored）。意味着可以通过这个字段查询，但不能取回它的原始值。
但这没有关系。因为字段值已经是_source字段的一部分，它是被默认存储的。如果只想取回一个字段或少部分字段的值，而不是整个_source，则可以通过source filtering达到目的。
在某些情况下，存储字段是有意义的。例如，如果有一个包含标题、日期和非常多的内容字段的文档，则可能希望只检索标题和日期，而不需要从大型_source字段中提取这些字段：例如，我们创建一个索引：
![[Pasted image 20220417200216.png]]
然后写入一条数据：
```json
PUT my_index/_doc/1
{
＂title＂: ＂Some short title＂，
＂content＂: ＂A very long content field...＂
}
下面的搜索将返回 title 字段的值：
GET my_index/_search
{
＂stored_fields＂: [ ＂title＂ ]
}
```
还有一种情况可能用到存储字段，就是不在_source中出现的字段（例如，copy_to字段）。

doc_values和存储字段（＂stored＂:ture）都属于正排内容，两者的设计初衷不同。

stored fields被设计为优化存储，doc_values被设计为快速访问字段值。搜索可能会访问很多docvalues中的字段，所以必须能够快速访问，我们将doc_values用于聚合、排序，以及脚本中。现在，ES中的许多特性都会自动使用doc_values。
另一方面，存储字段仅用于返回前几个最匹配文档的字段值，默认情况下 ES 只将其用于这种情况，解压存储字段，将其发送给客户端。为少量文档获取存储字段还好。它不能在查询的时候使用，否则会让查询变得非常慢。脚本中可以访问存储字段，但最好不要那么做。

## 优化措施
### 禁用对你来说不需要的特性
默认情况下，ES为大多数的字段建立索引，并添加到doc_values，以便使之可以被搜索和聚合。但是有时候不需要通过某些字段过滤，例如，有一个名为 foo 的数值类型字段，需要运行直方图，但不需要在这个字段上过滤，那么可以不索引这个字段：
![[Pasted image 20220417200950.png]]
text 类型的字段会在索引中存储归一因子（normalization factors），以便对文档进行评分，如果只需要在文本字段上进行匹配，而不关心生成的得分，则可以配置 ES 不将 norms 写入索引：
![[Pasted image 20220417201004.png]]
text类型的字段默认情况下也在索引中存储频率和位置。频率用于计算得分，位置用于执行短语（phrase）查询。如果不需要运行短语查询，则可以告诉ES不索引位置：
![[Pasted image 20220417201026.png]]
在text类型的字段上，index_options的默认值为positions。index_options参数用于控制添加到倒排索引中的信息。
freqs 文档编号和词频被索引，词频用于为搜索评分，重复出现的词条比只出现一次的词条评分更高。positions 文档编号、词频和位置被索引。位置被用于邻近查询（proximity queries）和短语查询（phrase queries）。
完整的 index_options 选项请参考官方手册：https://www.elastic.co/guide/en/elasticsearch/reference/master/index-options.html。

此外，如果也不关心评分，则可以将ES配置为只为每个term索引匹配的文档。仍然可以在这个字段上搜索，但是短语查询会出现错误，评分将假定在每个文档中只出现一次词汇。
![[Pasted image 20220417201059.png]]
### 禁用doc values
所有支持doc value的字段都默认启用了docvalue。如果确定不需要对字段进行排序或聚合，或者从脚本访问字段值，则可以禁用doc value以节省磁盘空间：
![[Pasted image 20220417201119.png]]
### 不要使用默认的动态字符串映射
默认的动态字符串映射会把字符串类型的字段同时索引为 text 和 keyword。如果只需要其中之一，则显然是一种浪费。通常，id字段只需作为keyword类型进行索引，而body字段只需作为text类型进行索引。
要禁用默认的动态字符串映射，则可以显式地指定字段类型，或者在动态模板中指定将字符串映射为text或keyword。下例将字符串字段映射为keyword：
![[Pasted image 20220417201149.png]]
### 观察分片大小
较大的分片可以更有效地存储数据。为了增加分片大小，可以在创建索引的时候设置较少的主分片数量，或者使用shrink API来修改现有索引的主分片数量。但是较大的分片也有缺点，例如，较长的索引恢复时间。
### 禁用_source
\_source 字段存储文档的原始内容。如果不需要访问它，则可以将其禁用。但是，需要访问_source的API将无法使用，至少包括下列情况：
· update、update_by_query、reindex；
· 高亮搜索；
· 重建索引（包括更新mapping、分词器，或者集群跨大版本升级可能会用到）；
· 调试聚合查询功能，需要对比原始数据。
### 使用best_compression
\_source和设置为＂store＂: true的字段占用磁盘空间都比较多。默认情况下，它们都是被压缩存储的。默认的压缩算法为LZ4，可以通过使用best_compression来执行压缩比更高的算法：DEFLATE。但这会占用更多的CPU资源。
PUT index{＂settings＂: {＂index＂: {＂codec＂: ＂best_compression＂}}}

### Fource Merge
一个ES索引由若干分片组成，一个分片有若干Lucene分段，较大的Lucene分段可以更有效地存储数据。使用_forcemerge API来对分段执行合并操作，通常，我们将分段合并为一个单个的分段：max_num_segments=1。
### Shrink Index
Shrink API允许减少索引的分片数量，结合上面的Force Merge API，可以显著减少索引的分片和Lucene分段数量。
### 数值类型长度够用就好
为数值类型选择的字段类型也可能会对磁盘使用空间产生较大影响，整型可以选择 byte、short、integer或 long，浮点型可以选择 scaled_float、float、double、half_float，每个数据类型的字节长度是不同的，为业务选择够用的最小数据类型，可以节省磁盘空间。
### 使用索引排序来排列类似的文档
当 ES 存储_source 时，它同时压缩多个文档以提高整体压缩比。例如，文档共享相同的字段名，或者它们共享一些字段值，特别是在具有低基数或zipfian 分布（参考https://en.wikipedia.org/wiki/Zipf%27s_law）的字段上。
默认情况下，文档按照添加到索引中的顺序压缩在一起。如果启用了索引排序，那么它们将按排序顺序压缩。对具有相似结构、字段和值的文档进行排序可以提高压缩比。关于索引排序的详细内容请参考官方手册：https://www.elastic.co/guide/en/elasticsearch/reference/master/index-modules-index-sorting.html。
### 在文档中以相同的顺序放置字段
由于多个文档被压缩成块，如果字段总是以相同的顺序出现，那么在这些_source文档中可以找到更长的重复字符串的可能性更大。

## 20.3 测试数据
下面是在笔者的环境中，使用测试数据调整不同索引方式的测试结论。测试数据为单个文档十几个字段，大小为800字节左右。数据样本如下：
![[Pasted image 20220417201435.png]]

在其他条件不变的情况下，调整单个参数，测试结果如下：
· 禁用_source，空间占用量下降30%左右；
· 禁用doc values，空间占用量下降10%左右；
· 压缩算法将LZ4改为Deflate，空间占用量可以下降15%～25%。
对于相似结构的数据，本文的测试结果可作一定参考，实际业务最好使用自己的样本数据进行压力测试以获取准确的结果。


# 搜索速度的优化

本章讨论搜索速度的优化、搜索速度与系统资源、数据索引方式、查询方式等多个方面，下面我们逐一讨论如何优化搜索速度。

## 为文件系统cache预留足够的内存
在一般情况下，应用程序的读写都会被操作系统“cache”（除了direct方式）,cache保存在系统物理内存中（线上应该禁用swap），命中cache可以降低对磁盘的直接访问频率。搜索很依赖对系统 cache 的命中，如果某个请求需要从磁盘读取数据，则一定会产生相对较高的延迟。应该至少为系统cache预留一半的可用物理内存，更大的内存有更高的cache命中率。

## 使用更快的硬件
写入性能对CPU的性能更敏感，而搜索性能在一般情况下更多的是在于I/O能力，使用SSD会比旋转类存储介质好得多。尽量避免使用 NFS 等远程文件系统，如果 NFS 比本地存储慢3倍，则在搜索场景下响应速度可能会慢10倍左右。这可能是因为搜索请求有更多的随机访问。
如果搜索类型属于计算比较多，则可以考虑使用更快的CPU。

## 文档模型
为了让搜索时的成本更低，文档应该合理建模。特别是应该避免join操作，嵌套（nested）会使查询慢几倍，父子（parent-child）关系可能使查询慢数百倍，因此，如果可以通过非规范化（denormalizing）文档来回答相同的问题，则可以显著地提高搜索速度。

## 预索引数据
还可以针对某些查询的模式来优化数据的索引方式。例如，如果所有文档都有一个 price字段，并且大多数查询在一个固定的范围上运行range聚合，那么可以通过将范围“pre-indexing”到索引中并使用terms聚合来加快聚合速度。
例如，文档起初是这样的：
```json
PUT index/type/1
{
＂designation＂: ＂spoon＂，
＂price＂: 13
}
```
采用如下的搜索方式：
![[Pasted image 20220417185739.png]]
那么我们的优化是，在建立索引时对文档进行富化，增加 price_range 字段，mapping 为keyword类型：
![[Pasted image 20220417185846.png]]
接下来，搜索请求可以聚合这个新字段，而不是在price字段上运行range聚合。
![[Pasted image 20220417185901.png]]

## 字段映射
有些字段的内容是数值，但并不意味着其总是应该被映射为数值类型，例如，一些标识符，将它们映射为keyword可能会比integer或long更好。

##  避免使用脚本
一般来说，应该避免使用脚本。如果一定要用，则应该优先考虑painless和expressions。

## 优化日期搜索
在使用日期范围检索时，使用now的查询通常不能缓存，因为匹配到的范围一直在变化。但是，从用户体验的角度来看，切换到一个完整的日期通常是可以接受的，这样可以更好地利用查询缓存。例如，有下列查询：
![[Pasted image 20220417190201.png]]
可以替换成下面的查询方式：
![[Pasted image 20220417190219.png]]
在这个例子中，我们将日期四舍五入到分钟，因此如果当前时间是16:31:29，那么 range查询将匹配my_date字段的值在15:31～16:31之间的所有内容。如果几个用户同时运行一个包含此范围的查询，则查询缓存可以加快查询速度。用于舍入的时间间隔越长，查询缓存就越有帮助，但要注意，太高的舍入也可能损害用户体验。
为了能够利用查询缓存，可以很容易将范围分割成一个大的可缓存部分和一个小的不可缓存部分，如下所示。
![[Pasted image 20220417190246.png]]
然而，这种做法可能会使查询在某些情况下运行得更慢，因为bool查询引入的开销可能会抵销利用查询缓存所节省的开销。

## 为只读索引执行force-merge
为不再更新的只读索引执行force merge，将Lucene索引合并为单个分段，可以提升查询速度。当一个Lucene索引存在多个分段时，每个分段会单独执行搜索再将结果合并，将只读索引强制合并为一个Lucene分段不仅可以优化搜索过程，对索引恢复速度也有好处。
基于日期进行轮询的索引的旧数据一般都不会再更新。此前的章节中说过，应该避免持续地写一个固定的索引，直到它巨大无比，而应该按一定的策略，例如，每天生成一个新的索引，然后用别名关联，或者使用索引通配符。这样，可以每天选一个时间点对昨天的索引执行force-merge、Shrink等操作。

## 预热全局序号（global ordinals）
全局序号是一种数据结构，用于在keyword字段上运行terms聚合。它用一个数值来代表字段中的字符串值，然后为每一数值分配一个 bucket。这需要一个对 global ordinals 和 bucket的构建过程。默认情况下，它们被延迟构建，因为ES不知道哪些字段将用于 terms聚合，哪些字段不会。可以通过配置映射在刷新（refresh）时告诉ES预先加载全局序数：
![[Pasted image 20220417190629.png]]
## execution hin
tterms聚合有两种不同的机制：
· 通过直接使用字段值来聚合每个桶的数据（map）。
· 通过使用字段的全局序号并为每个全局序号分配一个bucket（global_ordinals）。
ES 使用 global_ordinals 作为 keyword 字段的默认选项，它使用全局序号动态地分配bucket，因此内存使用与聚合结果中的字段数量是线性关系。在大部分情况下，这种方式的速度很快。
当查询只会匹配少量文档时，可以考虑使用map。默认情况下，map 只在脚本上运行聚合时使用，因为它们没有序数。
![[Pasted image 20220417191540.png]]
## 预热文件系统cache
如果ES主机重启，则文件系统缓存将为空，此时搜索会比较慢。可以使用index.store.preload设置，通过指定文件扩展名，显式地告诉操作系统应该将哪些文件加载到内存中，例如，配置到elasticsearch.yml文件中：
index.store.preload: [＂nvd＂, ＂dvd＂]
或者在索引创建时设置：
```json
PUT /my_index
{
	＂settings＂: {
	＂index.store.preload＂: [＂nvd＂, ＂dvd＂]
	}
}
```
如果文件系统缓存不够大，则无法保存所有数据，那么为太多文件预加载数据到文件系统缓存中会使搜索速度变慢，应谨慎使用。
##  转换查询表达式
在组合查询中可以通过bool过滤器进行and、or和not的多个逻辑组合检索，这种组合查询中的表达式在下面的情况下可以做等价转换：
（A|B） & （C|D） ==> （A&C） | （A&D） |（B&C） | （B&D ）
## 调节搜索请求中的batched_reduce_size
该字段是搜索请求中的一个参数。默认情况下，聚合操作在协调节点需要等所有的分片都取回结果后才执行，使用 batched_reduce_size 参数可以不等待全部分片返回结果，而是在指定数量的分片返回结果之后就可以先处理一部分（reduce）。这样可以避免协调节点在等待全部结果的过程中占用大量内存，避免极端情况下可能导致的OOM。该字段的默认值为512，从ES 5.4开始支持。
## 使用近似聚合
近似聚合以牺牲少量的精确度为代价，大幅提高了执行效率，降低了内存使用。近似聚合的使用方式可以参考官方手册：
· Percentiles Aggregation（https://www.elastic.co/guide/en/elasticsearch/ reference/current/search-aggregations-metrics-percentile-aggregation.html）。
· Cardinality Aggregation（https://www.elastic.co/guide/en/elasticsearch/reference/current/search- aggregations-metrics-cardinality-aggregation.html）。

## 深度优先还是广度优先
ES有两种不同的聚合方式：深度优先和广度优先。深度优先是默认设置，先构建完整的树，然后修剪无用节点。大多数情况下深度聚合都能正常工作，但是有些特殊的场景更适合广度优先，先执行第一层聚合，再继续下一层聚合之前会先做修剪，官方有一个例子可以参考：https://www.elastic.co/guide/cn/elasticsearch/guide/current/_preventing_combinatorial_explosions. html。

## 限制搜索请求的分片数
一个搜索请求涉及的分片数量越多，协调节点的CPU和内存压力就越大。默认情况下，ES会拒绝超过1000个分片的搜索请求。我们应该更好地组织数据，让搜索请求的分片数更少。如果想调节这个值，则可以通过action.search.shard_count配置项进行修改。虽然限制搜索的分片数并不能直接提升单个搜索请求的速度，但协调节点的压力会间接影响搜索速度，例如，占用更多内存会产生更多的GC压力，可能导致更多的stop-the-world时间等，因此间接影响了协调节点的性能，所以我们仍把它列作本章的一部分。
##  利用自适应副本选择（ARS）提升ES响应速度
为了充分利用计算资源和负载均衡，协调节点将搜索请求轮询转发到分片的每个副本，轮询策略是负载均衡过程中最简单的策略，任何一个负载均衡器都具备这种基础的策略，缺点是不考虑后端实际系统压力和健康水平。例如，一个分片的三个副本分布在三个节点上，其中Node2可能因为长时间GC、磁盘I/O过高、网络带宽跑满等原因处于忙碌状态，如下图所示。
![[Pasted image 20220417192532.png]]
如果搜索请求被转发到副本2，则会看到相对于其他分片来说，副本2有更高的延迟。· 分片副本1:100ms。· 分片副本2（degraded）:1350ms。· 分片副本3:150ms。由于副本2的高延迟，使得整个搜索请求产生长尾效应。ES希望这个过程足够智能，能够将请求路由到其他数据副本，直到该节点恢复到足以处理更多搜索请求的程度。在ES中，此过程称为“自适应副本选择”。

在实现过程中，ES参考一篇名为C3的论文：Cutting Tail Latency in Cloud Data Stores viaAdaptive Replica Selection（https://www.usenix.org/conference/nsdi15/technical-sessions/presentation/suresh）。这篇论文是为Cassandra写的，ES基于这篇论文的思想做了调整以适合自己的场景。ES的ARS实现基于这样一个公式：对每个搜索请求，将分片的每个副本进行排序，以确定哪个最可能是转发请求的“最佳”副本。与轮询方式向分片的每个副本发送请求不同，ES选择“最佳”副本并将请求路由到那里。ARS公式为：
![[Pasted image 20220417192953.png]]
每项含义如下：
· os（s），节点未完成的搜索请求数；
· n，系统中数据节点的数量；
· R（s），响应时间的EWMA（从协调节点上可以看到），单位为毫秒；
· q（s），搜索线程池队列中等待任务数量的EWMA；
· µ（s），数据节点上的搜索服务时间的EWMA，单位为毫秒。
关于EWMA的解释可参考https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average。通过这些信息我们大致可以评估出分片副本所在节点的压力和健康程度，这就可以让我们选出一个能够更快返回搜索请求的节点。在上面的例子中，请求将被转发到分片副本1或分片副本3。ARS从6.1版本开始支持，但是默认关闭，可以通过下面的命令动态开启：
```json
PUT /_cluster/settings
{＂transient＂: 
{＂cluster.routing.use_adaptive_replica_selection＂: true
}
}
```
从ES 7.0开始，ARS将默认开启。官方进行了多种场景的基准测试，包括某个数据节点处于高负载状态和非负载状态，测试使用5节点的集群，单个索引，5个主分片，每个主分片有一个副分片。将搜索请求发送到单个协调节点。没有模拟某个节点高负载的情况下（测试前节点都处于空闲），指标如下表所示。
![[Pasted image 20220417193112.png]]
可见，即使集群的负载是均匀的，ARS仍然可以改善吞吐量和响应延迟。模拟单个节点处于高负载下的情况，指标如下表所示。
![[Pasted image 20220417193129.png]]
使用ARS，在某个数据节点处于高负载的情况下，吞吐量有了很大的提高。延迟中位数有所增加是预料之中的，为了绕开高负载的节点，稍微增加了无压力节点的负载，从而增加了延迟。

# 写入速度优化


在 ES 的默认设置下，是综合考虑数据可靠性、搜索实时性、写入速度等因素的。当离开默认设置、追求极致的写入速度时，很多是以牺牲可靠性和搜索实时性为代价的。有时候，业务上对数据可靠性和搜索实时性要求并不高，反而对写入速度要求很高，此时可以调整一些策略，最大化写入速度。接下来的优化基于集群正常运行的前提下，如果是集群首次批量导入数据，则可以将副本数设置为0，导入完毕再将副本数调整回去，这样副分片只需要复制，节省了构建索引过程。综合来说，提升写入速度从以下几方面入手：
· 加大translog flush间隔，目的是降低iops、writeblock。
· 加大index refresh间隔，除了降低I/O，更重要的是降低了segment merge频率。
· 调整bulk请求。
· 优化磁盘间的任务均匀情况，将shard尽量均匀分布到物理主机的各个磁盘。
· 优化节点间的任务分布，将任务尽量均匀地发到各节点。
· 优化Lucene层建立索引的过程，目的是降低CPU占用率及I/O，例如，禁用_all字段。

## translog flush间隔调整
从ES 2.x开始，在默认设置下，translog的持久化策略为：每个请求都“flush”。对应配置项如下：index.translog.durability: request
这是影响 ES 写入速度的最大因素。但是只有这样，写操作才有可能是可靠的。如果系统可以接受一定概率的数据丢失（例如，数据写入主分片成功，尚未复制到副分片时，主机断电。由于数据既没有刷到Lucene,translog也没有刷盘，恢复时translog中没有这个数据，数据丢失），则调整translog持久化策略为周期性和一定大小的时候“flush”，例如：index.translog.durability: async

设置为async表示translog的刷盘策略按sync_interval配置指定的时间周期进行。index.translog.sync_interval: 120s加大translog刷盘间隔时间。默认为5s，不可低于100ms。index.translog.flush_threshold_size: 1024mb超过这个大小会导致refresh操作，产生新的Lucene分段。默认值为512MB。

## 索引刷新间隔refresh_interval
默认情况下索引的refresh_interval为1秒，这意味着数据写1秒后就可以被搜索到，每次索引的refresh会产生一个新的Lucene段，这会导致频繁的segment merge行为，如果不需要这么高的搜索实时性，应该降低索引refresh周期，例如：index.refresh_interval: 120s

## 段合并优化
segment merge操作对系统I/O和内存占用都比较高，从ES 2.0开始，merge行为不再由ES控制，而是由Lucene控制，因此以下配置已被删除：
indices.store.throttle.type
indices.store.throttle.max_bytes_per_sec
index.store.throttle.type
index.store.throttle.max_bytes_per_sec
改为以下调整开关：
index.merge.scheduler.max_thread_count
index.merge.policy.*
最大线程数
max_thread_count的默认值如下：
Math.max（1, Math.min（4,Runtime.getRuntime（）.availableProcessors（） / 2））

以上是一个比较理想的值，如果只有一块硬盘并且非 SSD，则应该把它设置为1，因为在旋转存储介质上并发写，由于寻址的原因，只会降低写入速度。

merge策略index.merge.policy有三种：
· tiered（默认策略）；
· log_byete_size；
· log_doc。
每个策略的具体描述可以参考 MasteringElasticsearch:https://doc.yonyoucloud.com/doc/mastering-elasticsearch/chapter-3/36_README.html。目前我们使用默认策略，但是对策略的参数进行了一些调整。

索引创建时合并策略就已确定，不能更改，但是可以动态更新策略参数，可以不做此项调整。如果堆栈经常有很多merge，则可以尝试调整以下策略配置：index.merge.policy.segments_per_tier
该属性指定了每层分段的数量，取值越小则最终segment越少，因此需要merge的操作更多，可以考虑适当增加此值。默认为10，其应该大于等于index.merge.policy.max_merge_at_once。index.merge.policy.max_merged_segment指定了单个segment的最大容量，默认为5GB，可以考虑适当降低此值。

## indexing buffer
indexing buffer在为doc建立索引时使用，当缓冲满时会刷入磁盘，生成一个新的segment，这是除refresh_interval刷新索引外，另一个生成新segment的机会。每个shard有自己的indexing buffer，下面的这个buffer大小的配置需要除以这个节点上所有shard的数量：

indices.memory.index_buffer_size默认为整个堆空间的10%。indices.memory.min_index_buffer_size默认为48MB。indices.memory.max_index_buffer_size默认为无限制。
在执行大量的索引操作时，indices.memory.index_buffer_size的默认设置可能不够，这和可用堆内存、单节点上的shard数量相关，可以考虑适当增大该值。

## 使用bulk请求
批量写比一个索引请求只写单个文档的效率高得多，但是要注意bulk请求的整体字节数不要太大，太大的请求可能会给集群带来内存压力，因此每个请求最好避免超过几十兆字节，即使较大的请求看上去执行得更好。
### bulk线程池和队列
建立索引的过程属于计算密集型任务，应该使用固定大小的线程池配置，来不及处理的任务放入队列。线程池最大线程数量应配置为CPU核心数+1，这也是bulk线程池的默认设置，可以避免过多的上下文切换。队列大小可以适当增加，但一定要严格控制大小，过大的队列导致较高的GC压力，并可能导致FGC频繁发生。
### 并发执行bulk请求
bulk写请求是个长任务，为了给系统增加足够的写入压力，写入过程应该多个客户端、多线程地并行执行，如果要验证系统的极限写入能力，那么目标就是把CPU压满。磁盘util、内存等一般都不是瓶颈。如果 CPU 没有压满，则应该提高写入端的并发数量。但是要注意 bulk线程池队列的reject情况，出现reject代表ES的bulk队列已满，客户端请求被拒绝，此时客户端会收到429错误（TOO_MANY_REQUESTS），客户端对此的处理策略应该是延迟重试。不可忽略这个异常，否则写入系统的数据会少于预期。即使客户端正确处理了429错误，我们仍然应该尽量避免产生reject。因此，在评估极限的写入能力时，客户端的极限写入并发量应该控制在不产生reject前提下的最大值为宜。

## 磁盘间的任务均衡
如果部署方案是为path.data配置多个路径来使用多块磁盘，则ES在分配shard时，落到各磁盘上的shard 可能并不均匀，这种不均匀可能会导致某些磁盘繁忙，利用率在较长时间内持续达到100%。这种不均匀达到一定程度会对写入性能产生负面影响。ES在处理多路径时，优先将shard分配到可用空间百分比最多的磁盘上，因此短时间内创建的shard可能被集中分配到这个磁盘上，即使可用空间是99%和98%的差别。后来ES在2.x版本中开始解决这个问题：预估一下 shard 会使用的空间，从磁盘可用空间中减去这部分，直到现在6.x版也是这种处理方式。但是实现也存在一些问题：从可用空间减去预估大小
这种机制只存在于一次索引创建的过程中，下一次的索引创建，磁盘可用空间并不是上次做完减法以后的结果。这也可以理解，毕竟预估是不准的，一直减下去空间很快就减没了。但是最终的效果是，这种机制并没有从根本上解决问题，即使没有完美的解决方案，这种机制的效果也不够好。如果单一的机制不能解决所有的场景，那么至少应该为不同场景准备多种选择。为此，我们为ES增加了两种策略。· 简单轮询：在系统初始阶段，简单轮询的效果是最均匀的。· 基于可用空间的动态加权轮询：以可用空间作为权重，在磁盘之间加权轮询。

## 节点间的任务均衡
为了节点间的任务尽量均衡，数据写入客户端应该把bulk请求轮询发送到各个节点。当使用Java API或REST API的bulk接口发送数据时，客户端将会轮询发送到集群节点，节点列表取决于：· 使用Java API时，当设置client.transport.sniff为true（默认为false）时，列表为所有数据节点，否则节点列表为构建客户端对象时传入的节点列表。· 使用REST API时，列表为构建对象时添加进去的节点。Java API的TransportClient和REST API的RestClient都是线程安全的，如果写入程序自己创建线程池控制并发，则应该使用同一个Client对象。在此建议使用REST API,Java API会在未来的版本中废弃，REST API有良好的版本兼容性好。理论上，Java API在序列化上有性能优势，但是只有在吞吐量非常大时才值得考虑序列化的开销带来的影响，通常搜索并不是高吞吐量的业务。要观察bulk请求在不同节点间的均衡性，可以通过cat接口观察bulk线程池和队列情况：\_cat/thread_pool

## 索引过程调整和优化
### 自动生成doc ID
通过ES写入流程可以看出，写入doc时如果外部指定了id，则ES会先尝试读取原来doc的版本号，以判断是否需要更新。这会涉及一次读取磁盘的操作，通过自动生成doc ID可以避免这个环节。
### 调整字段Mappings
（1）减少字段数量，对于不需要建立索引的字段，不写入ES。
（2）将不需要建立索引的字段index属性设置为not_analyzed或no。对字段不分词，或者不索引，可以减少很多运算操作，降低CPU占用。尤其是binary类型，默认情况下占用CPU非常高，而这种类型进行分词通常没有什么意义。
（3）减少字段内容长度，如果原始数据的大段内容无须全部建立索引，则可以尽量减少不必要的内容。
（4）使用不同的分析器（analyzer），不同的分析器在索引过程中运算复杂度也有较大的差异。

### 调整_source字段
\_source 字段用于存储 doc 原始数据，对于部分不需要存储的字段，可以通过 includes excludes过滤，或者将_source禁用，一般用于索引和数据分离。这样可以降低 I/O 的压力，不过实际场景中大多不会禁用_source，而即使过滤掉某些字段，对于写入速度的提升作用也不大，满负荷写入情况下，基本是 CPU 先跑满了，瓶颈在于CPU。

### 禁用_all字段
从ES 6.0开始，\_all字段默认为不启用，而在此前的版本中，\_all字段默认是开启的。\_all字段中包含所有字段分词后的关键词，作用是可以在搜索的时候不指定特定字段，从所有字段中检索。ES 6.0默认禁用_all字段主要有以下几点原因：
· 由于需要从其他的全部字段复制所有字段值，导致_all字段占用非常大的空间。
· \_all 字段有自己的分析器，在进行某些查询时（例如，同义词），结果不符合预期，因为没有匹配同一个分析器。
· 由于数据重复引起的额外建立索引的开销。
· 想要调试时，其内容不容易检查。· 有些用户甚至不知道存在这个字段，导致了查询混乱。
· 有更好的替代方法。

关于这个问题的更多讨论可以参考https://github.com/elastic/elasticsearch/issues/19784。
禁用_all字段可以明显降低对CPU和I/O的压力。

### 对Analyzed的字段禁用Norms
Norms用于在搜索时计算doc的评分，如果不需要评分，则可以将其禁用：＂title＂: {＂type＂: ＂string＂,＂norms＂: {＂enabled＂: false}}

### index_options 设置
index_options用于控制在建立倒排索引过程中，哪些内容会被添加到倒排索引，例如，doc数量、词频、positions、offsets等信息，优化这些设置可以一定程度降低索引过程中的运算任务，节省CPU占用率。不过在实际场景中，通常很难确定业务将来会不会用到这些信息，除非一开始方案就明确是这样设计的。

## 参考配置
下面是笔者的线上环境使用的全局模板和配置文件的部分内容，省略掉了节点名称、节点列表等基础配置字段，仅列出与本文相关内容。从ES 5.x开始，索引级设置需要写在模板中，或者在创建索引时指定，我们把各个索引通用的配置写到了模板中，这个模板匹配全部的索引，并且具有最低的优先级，让用户定义的模板有更高的优先级，以覆盖这个模板中的配置。
```json
{＂template＂: ＂*＂，
 ＂order＂ : 0，
 ＂settings＂: {
	 ＂index.merge.policy.max_merged_segment＂: ＂2gb＂，
	 ＂index.merge.policy.segments_per_tier＂ : ＂24＂，
	 ＂index.number_of_replicas＂ : ＂1＂，
	 ＂index.number_of_shards＂ : ＂24＂，
	 ＂index.optimize_auto_generated_id＂ : ＂true＂，
	 ＂index.refresh_interval＂ : ＂120s＂，
	 ＂index.translog.durability＂ : ＂async＂，
	 ＂index.translog.flush_threshold_size＂ : ＂1000mb＂，
	 ＂index.translog.sync_interval＂ : ＂120s＂，
	 ＂index.unassigned.node_left.delayed_timeout＂ : ＂5d＂
  }
}
```
elasticsearch.yml中的配置：
indices.memory.index_buffer_size: 30%

##  思考与总结
（1）方法比结论重要。一个系统性问题往往是多种因素造成的，在处理集群的写入性能问题上，先将问题分解，在单台上进行压测，观察哪种系统资源达到极限，例如，CPU或磁盘利用率、I/Oblock、线程切换、堆栈状态等。然后分析并调整参数，优化单台上的能力，先解决局部问题，在此基础上解决整体问题会容易得多。
（2）可以使用更好的 CPU，或者使用 SSD，对写入性能提升明显。在我们的测试中，在相同条件下，E5 2650V4比E5 2430 v2的写入速度高60%左右。
（3）在我们的压测环境中，写入速度稳定在平均单机每秒3万条以上，使用的测试数据：每个文档的字段数量为10个左右，文档大小约100字节，CPU使用E5 2430 v2。


# 硬件配置优化

> 升级硬件设备配置一直都是提高服务能力最快速有效的手段，在系统层面能够影响应用性能的一般包括三个因素：CPU、内存和 IO，可以从这三方面进行 ES 的性能优化工作。

## CPU 配置

一般说来，CPU 繁忙的原因有以下几个：

-   线程中有无限空循环、无阻塞、正则匹配或者单纯的计算；
-   发生了频繁的 GC；
-   多线程的上下文切换；

大多数 Elasticsearch 部署往往对 CPU 要求不高。因此，相对其它资源，具体配置多少个（CPU）不是那么关键。你应该选择具有多个内核的现代处理器，常见的集群使用 2 到 8 个核的机器。如果你要在更快的 CPUs 和更多的核数之间选择，选择更多的核数更好。多个内核提供的额外并发远胜过稍微快一点点的时钟频率。

## 内存配置

如果有一种资源是最先被耗尽的，它可能是内存。排序和聚合都很耗内存，所以有足够的堆空间来应付它们是很重要的。即使堆空间是比较小的时候，也能为操作系统文件缓存提供额外的内存。因为 Lucene 使用的许多数据结构是基于磁盘的格式，Elasticsearch 利用操作系统缓存能产生很大效果。

64 GB 内存的机器是非常理想的，但是 32 GB 和 16 GB 机器也是很常见的。少于8 GB 会适得其反（你最终需要很多很多的小机器），大于 64 GB 的机器也会有问题。

由于 ES 构建基于 lucene，而 lucene 设计强大之处在于 lucene 能够很好的利用操作系统内存来缓存索引数据，以提供快速的查询性能。lucene 的索引文件 segements 是存储在单文件中的，并且不可变，对于 OS 来说，能够很友好地将索引文件保持在 cache 中，以便快速访问；因此，我们很有必要将一半的物理内存留给 lucene；另**一半的物理内存留给 ES**（JVM heap）。

### 内存分配

当机器内存小于 64G 时，遵循通用的原则，50% 给 ES，50% 留给 lucene。

当机器内存大于 64G 时，遵循以下原则：

-   如果主要的使用场景是全文检索，那么建议给 ES Heap 分配 4~32G 的内存即可；其它内存留给操作系统，供 lucene 使用（segments cache），以提供更快的查询性能。
-   如果主要的使用场景是聚合或排序，并且大多数是 numerics，dates，geo_points 以及 not_analyzed 的字符类型，建议分配给 ES Heap 分配 4~32G 的内存即可，其它内存留给操作系统，供 lucene 使用，提供快速的基于文档的聚类、排序性能。
-   如果使用场景是聚合或排序，并且都是基于 analyzed 字符数据，这时需要更多的 heap size，建议机器上运行多 ES 实例，每个实例保持不超过 50% 的 ES heap 设置（但不超过 32 G，堆内存设置 32 G 以下时，JVM 使用对象指标压缩技巧节省空间），50% 以上留给 lucene。

### 禁止 swap

禁止 swap，一旦允许内存与磁盘的交换，会引起致命的性能问题。可以通过在 elasticsearch.yml 中 bootstrap.memory_lock: true，以保持 JVM 锁定内存，保证 ES 的性能。

### GC 设置

[老的版本中官方文档在新窗口打开](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dont-touch-these-settings.html)中推荐默认设置为：Concurrent-Mark and Sweep（CMS），给的理由是当时G1 还有很多 BUG。

原因是：已知JDK 8附带的HotSpot JVM的早期版本存在一些问题，当启用G1GC收集器时，这些问题可能导致索引损坏。受影响的版本早于JDK 8u40随附的HotSpot版本。来源于[官方说明在新窗口打开](https://www.elastic.co/guide/en/elasticsearch/reference/current/_g1gc_check.html)

实际上如果你使用的JDK8较高版本，或者JDK9+，我推荐你使用G1 GC； 因为我们目前的项目使用的就是G1 GC，运行效果良好，对Heap大对象优化尤为明显。修改jvm.options文件，将下面几行:

```
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
```

更改为

```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=50
```

其中 -XX:MaxGCPauseMillis是控制预期的最高GC时长，默认值为200ms，如果线上业务特性对于GC停顿非常敏感，可以适当设置低一些。但是 这个值如果设置过小，可能会带来比较高的cpu消耗。

G1对于集群正常运作的情况下减轻G1停顿对服务时延的影响还是很有效的，但是如果是你描述的GC导致集群卡死，那么很有可能换G1也无法根本上解决问题。 通常都是集群的数据模型或者Query需要优化。

## 磁盘

硬盘对所有的集群都很重要，对大量写入的集群更是加倍重要（例如那些存储日志数据的）。硬盘是服务器上最慢的子系统，这意味着那些写入量很大的集群很容易让硬盘饱和，使得它成为集群的瓶颈。

**在经济压力能承受的范围下，尽量使用固态硬盘（SSD）**。固态硬盘相比于任何旋转介质（机械硬盘，磁带等），无论随机写还是顺序写，都会对 IO 有较大的提升。

> 1.  如果你正在使用 SSDs，确保你的系统 I/O 调度程序是配置正确的。当你向硬盘写数据，I/O 调度程序决定何时把数据实际发送到硬盘。大多数默认 *nix 发行版下的调度程序都叫做 cfq（完全公平队列）。
> 2.  调度程序分配时间片到每个进程。并且优化这些到硬盘的众多队列的传递。但它是为旋转介质优化的：机械硬盘的固有特性意味着它写入数据到基于物理布局的硬盘会更高效。
> 3.  这对 SSD 来说是低效的，尽管这里没有涉及到机械硬盘。但是，deadline 或者 noop 应该被使用。deadline 调度程序基于写入等待时间进行优化，noop 只是一个简单的 FIFO 队列。

**这个简单的更改可以带来显著的影响。仅仅是使用正确的调度程序，我们看到了 500 倍的写入能力提升**。

如果你使用旋转介质（如机械硬盘），尝试获取尽可能快的硬盘（高性能服务器硬盘，15k RPM 驱动器）。

**使用 RAID0 是提高硬盘速度的有效途径，对机械硬盘和 SSD 来说都是如此**。没有必要使用镜像或其它 RAID 变体，因为 Elasticsearch 在自身层面通过副本，已经提供了备份的功能，所以不需要利用磁盘的备份功能，同时如果使用磁盘备份功能的话，对写入速度有较大的影响。

最后，**避免使用网络附加存储（NAS）**。人们常声称他们的 NAS 解决方案比本地驱动器更快更可靠。除却这些声称，我们从没看到 NAS 能配得上它的大肆宣传。NAS 常常很慢，显露出更大的延时和更宽的平均延时方差，而且它是单点故障的。


# 哈啰：记录一次ElasticSearch的查询性能优化
## 问题: 慢查询

搜索平台的公共集群，由于业务众多，对业务的es查询语法缺少约束，导致问题频发。业务可能写了一个巨大的查询直接把集群打挂掉，但是我们平台人力投入有限，也不可能一条条去审核业务的es查询语法，只能通过后置的手段去保证整个集群的稳定性，通过slowlog分析等，下图中cpu已经100%了。
![[Pasted image 20230406160958.png]]
昨天刚好手头有一点点时间，就想着能不能针对这些情况，把影响最坏的业务抓出来，进行一些改善，于是昨天花了2小时分析了一下，找到了一些共性的问题，可以通过平台来很好的改善这些情况。
![[Pasted image 20230406161005.png]]
首先通过slowlog抓到一些耗时比较长的查询，例如下面这个索引的查询耗时基本都在300ms以上：

```json
{
  "from": 0,
  "size": 200,
  "timeout": "60s",
  "query": {
    "bool": {
      "must": \[
        {
          "match": {
            "source": {
              "query": "5",
              "operator": "OR",
              "prefix\_length": 0,
              "fuzzy\_transpositions": true,
              "lenient": false,
              "zero\_terms\_query": "NONE",
              "auto\_generate\_synonyms\_phrase\_query": "false",
              "boost": 1
            }
          }
        },
        {
          "terms": {
            "type": \[
              "21"
            \],
            "boost": 1
          }
        },
        {
          "match": {
            "creator": {
              "query": "0d754a8af3104e978c95eb955f6331be",
              "operator": "OR",
              "prefix\_length": 0,
              "fuzzy\_transpositions": "true",
              "lenient": false,
              "zero\_terms\_query": "NONE",
              "auto\_generate\_synonyms\_phrase\_query": "false",
              "boost": 1
            }
          }
        },
        {
          "terms": {
            "status": \[
              "0",
              "3"
            \],
            "boost": 1
          }
        },
        {
          "match": {
            "isDeleted": {
              "query": "0",
              "operator": "OR",
              "prefix\_length": 0,
              "fuzzy\_transpositions": "true",
              "lenient": false,
              "zero\_terms\_query": "NONE",
              "auto\_generate\_synonyms\_phrase\_query": "false",
              "boost": 1
            }
          }
        }
      \],
      "adjust\_pure\_negative": true,
      "boost": 1
    }
  },
  "\_source": {
    "includes": \[
    \],
    "excludes": \[\]
  }
}

这个查询比较简单，翻译一下就是：

SELECT guid FROM xxx WHERE source=5 AND type=21 AND creator='0d754a8af3104e978c95eb955f6331be' AND status in (0,3) AND isDeleted=0;
```

## 慢查询分析

这个查询问题还挺多的，不过不是今天的重点。比如这里面不好的一点是还用了模糊查询fuzzy_transpositions,也就是查询ab的时候，ba也会被命中，其中的语法不是今天的重点，可以自行查询，我估计这个是业务用了SDK自动生成的，里面很多都是默认值。

第一反应是当然是用filter来代替match查询，一来filter可以缓存，另外避免这种无意义的模糊匹配查询，但是这个优化是有限的，并不是今天讲解的关键点，先忽略。

###  错用的数据类型

我们通过kibana的profile来进行分析，耗时到底在什么地方？es有一点就是开源社区很活跃，文档齐全，配套的工具也非常的方便和齐全。
![[Pasted image 20230406161038.png]]
可以看到大部分的时间都花在了PointRangQuery里面去了，这个是什么查询呢？为什么这么耗时呢？这里就涉及到一个es的知识点，那就是对于integer这种数字类型的处理。在es2.x的时代，所有的数字都是按keyword处理的，每个数字都会建一个倒排索引，这样查询虽然快了，但是一旦做范围查询的时候。比如 `type>1 and type<5`就需要转成 type in (1,2,3,4,5)来进行，大大的增加了范围查询的难度和耗时。

之后es做了一个优化，在integer的时候设计了一种类似于b-tree的数据结构，加速范围的查询，详细可以参考(https://elasticsearch.cn/article/446)

所以在这之后，所有的integer查询都会被转成范围查询，这就导致了上面看到的isDeleted的查询的解释。那么为什么范围查询在我们这个场景下，就这么慢呢？能不能优化。

明明我们这个场景是不需要走范围查询的，因为如果走倒排索引查询就是O(1)的时间复杂度，将大大提升查询效率。由于业务在创建索引的时候，isDeleted这种字段建成了Integer类型，导致最后走了范围查询，那么只需要我们将isDeleted类型改成keyword走term查询，就能用上倒排索引了。

实际上这里还涉及到了es的一个查询优化。类似于isDeleted这种字段，毫无区分度的倒排索引的时候，在查询的时候，es是怎么优化的呢？

### 多个Term查询的顺序问题

实际上，如果有多个term查询并列的时候，他的执行顺序，既不是你查询的时候，写进去的顺序。
![[Pasted image 20230406161048.png]]
例如上面这个查询，他既不是先执行source=5再执行type=21按照你代码的顺序执行过滤，也不是同时并发执行所有的过滤条件，然后再取交集。es很聪明，他会评估每个filter的条件的区分度，把高区分度的filter先执行，以此可以加速后面的filter循环速度。比如creator=0d754a8af3104e978c95eb955f6331be查出来之后10条记录，他就会优先执行这一条。

怎么做到的呢？其实也很简单，term建的时候，每一个term在写入的时候都会记录一个词频，也就是这个term在全部文档里出现的次数，这样我们就能判断当前的这个term他的区分度高低了。

### 为什么PointRangeQuery在这个场景下非常慢

上面提到了这种查询的数据结构类似于b-tree,他在做范围查询的时候，非常有优势，Lucene将这颗B-tree的非叶子结点部分放在内存里，而叶子结点紧紧相邻存放在磁盘上。当作range查询的时候，内存里的B-tree可以帮助快速定位到满足查询条件的叶子结点块在磁盘上的位置，之后对叶子结点块的读取几乎都是顺序的。
![[Pasted image 20230406161055.png]]
总结就是这种结构适合范围查询，且磁盘的读取是顺序读取的。但是在我们这种场景之下，term查询可就麻烦了，数值型字段的TermQuery被转换为了PointRangeQuery。这个Query利用Block k-d tree进行范围查找速度非常快，但是满足查询条件的docid集合在磁盘上并非向Postlings list那样按照docid顺序存放，也就无法实现postings list上借助跳表做蛙跳的操作。

要实现对docid集合的快速advance操作，只能将docid集合拿出来，做一些再处理。这个处理过程在org.apache.lucene.search.PointRangeQuery#createWeight这个方法里可以读取到。这里就不贴冗长的代码了，主要逻辑就是在创建scorer对象的时候，顺带先将满足查询条件的docid都选出来，然后构造成一个代表docid集合的bitset，这个过程和构造Query cache的过程非常类似。之后advance操作，就是在这个bitset上完成的。所有的耗时都在构建bitset上，因此可以看到耗时主要在build_scorer上了。

## 验证

找到原因之后，就可以开始验证了。将原来的integer类型全部改成keyword类型，如果业务真的有用到范围查询，应该会报错。通过搜索平台的平台直接修改配置，修改完成之后，重建索引就生效了。
![[Pasted image 20230406161105.png]]
索引切换之后的效果也非常的明显，通过kibana的profile分析可以看到，之前需要接近100ms的PointRangQuery现在走倒排索引，只需要0.5ms的时间。
![[Pasted image 20230406161120.png]]
之前这个索引的平均latency在100ms+，这个是es分片处理的耗时,从搜索行为开始，到搜索行为结束的打点，不包含网络传输时间和连接建立时间，单纯的分片内的函数的处理时间的平均值，正常情况在10ms左右。
![[Pasted image 20230406161128.png]]
经过调整之后的耗时降到了10ms内。
![[Pasted image 20230406161137.png]]
通过监控查看慢查询的数量，立即减少到了0。
![[Pasted image 20230406161144.png]]


