---
annotation-target: Go语言学习笔记.pdf
---




>%%
>```annotation-json
>{"text":"golang内存分配的大体流程\n","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":191042,"end":191265},{"type":"TextQuoteSelector","exact":"基本策略： 1．每次从操作系统申请一大块内存（比如1MB），以减少系统调用。 2．将申请到的大块内存按照特定大小预先切分成小块，构成链表。 3．为对象分配内存时，只须从大小合适的链表提取一个小块即可。 4．回收对象内存时，将该小块内存重新归还到原链表，以便复用。 5．如闲置内存过多，则尝试归还部分内存给操作系统，降低整体开销。 内存分配器只管理内存块，并不关心对象状态。且它不会主动回收内存，垃圾回收器在完成清理操作后，触发内存分配器的回收操作","prefix":"配算法细节前，我们需要了解一些基本概念，这有助于建立宏观认识。","suffix":"。 内存块 分配器将其管理的内存块分为两种。 Go语言学习笔记"}]}],"created":"2022-03-02T08:36:19.470Z","updated":"2022-03-02T08:36:19.470Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}
>```
>%%
>*%%PREFIX%%配算法细节前，我们需要了解一些基本概念，这有助于建立宏观认识。%%HIGHLIGHT%% ==基本策略： 1．每次从操作系统申请一大块内存（比如1MB），以减少系统调用。 2．将申请到的大块内存按照特定大小预先切分成小块，构成链表。 3．为对象分配内存时，只须从大小合适的链表提取一个小块即可。 4．回收对象内存时，将该小块内存重新归还到原链表，以便复用。 5．如闲置内存过多，则尝试归还部分内存给操作系统，降低整体开销。 内存分配器只管理内存块，并不关心对象状态。且它不会主动回收内存，垃圾回收器在完成清理操作后，触发内存分配器的回收操作== %%POSTFIX%%。 内存块 分配器将其管理的内存块分为两种。 Go语言学习笔记*
>%%LINK%%[[#^z3q31wx1j6|show annotation]]
>%%COMMENT%%
>golang内存分配的大体流程
>
>%%TAGS%%
>#内存分配
^z3q31wx1j6


>%%
>```annotation-json
>{"created":"2022-03-02T08:56:11.009Z","text":"分配器将其管理的内存块分为两种","updated":"2022-03-02T08:56:11.009Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":191301,"end":191406},{"type":"TextQuoteSelector","exact":" span：由多个地址连续的页（page）组成的大块内存。  object：将 span 按特定大小切分成多个小块，每个小块可存储一个对象。 按照其用途，span 面向内部管理，object 面向对象分配。","prefix":"块 分配器将其管理的内存块分为两种。 Go语言学习笔记 256 ","suffix":" 分配器按页数来区分不同大小的  span。比如，以页数为单位将"}]}]}
>```
>%%
>*%%PREFIX%%块 分配器将其管理的内存块分为两种。 Go语言学习笔记 256%%HIGHLIGHT%% == span：由多个地址连续的页（page）组成的大块内存。  object：将 span 按特定大小切分成多个小块，每个小块可存储一个对象。 按照其用途，span 面向内部管理，object 面向对象分配。== %%POSTFIX%%分配器按页数来区分不同大小的  span。比如，以页数为单位将*
>%%LINK%%[[#^tnlpixfr90d|show annotation]]
>%%COMMENT%%
>分配器将其管理的内存块分为两种
>%%TAGS%%
>#内存分配
^tnlpixfr90d


>%%
>```annotation-json
>{"created":"2022-03-02T08:57:02.836Z","text":"灵活的裁剪和合并，以此来减少分片","updated":"2022-03-02T08:57:02.836Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":191407,"end":191610},{"type":"TextQuoteSelector","exact":"分配器按页数来区分不同大小的  span。比如，以页数为单位将  span  存放到管理数组中，需要时就以页数为索引进行查找。当然，span  大小并非固定不变。在获取闲置  span 时，如果没找到大小合适的，那就返回页数更多的，此时会引发裁剪操作，多余部分将构成新的  span  被放回管理数组。分配器还会尝试将地址相邻的空闲  span  合并，以构建更大的内存块，减少碎片，提供更灵活的分配策略。","prefix":"照其用途，span 面向内部管理，object 面向对象分配。 ","suffix":" malloc.go _PageShift = 13 _Page"}]}]}
>```
>%%
>*%%PREFIX%%照其用途，span 面向内部管理，object 面向对象分配。%%HIGHLIGHT%% ==分配器按页数来区分不同大小的  span。比如，以页数为单位将  span  存放到管理数组中，需要时就以页数为索引进行查找。当然，span  大小并非固定不变。在获取闲置  span 时，如果没找到大小合适的，那就返回页数更多的，此时会引发裁剪操作，多余部分将构成新的  span  被放回管理数组。分配器还会尝试将地址相邻的空闲  span  合并，以构建更大的内存块，减少碎片，提供更灵活的分配策略。== %%POSTFIX%%malloc.go _PageShift = 13 _Page*
>%%LINK%%[[#^1m17nkx5xmv|show annotation]]
>%%COMMENT%%
>灵活的裁剪和合并，以此来减少分片
>%%TAGS%%
>#内存分配
^1m17nkx5xmv


>%%
>```annotation-json
>{"created":"2022-03-02T08:58:42.622Z","text":"对象好像是超出了32kb就是被当作了大对象","updated":"2022-03-02T08:58:42.622Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":192711,"end":192751},{"type":"TextQuoteSelector","exact":"若对象大小超出特定阈值限制，会被当作大对象（large object）特别对待。","prefix":"MaxSmallSize-1024)/128 + 1]int8 ","suffix":" malloc.go _MaxSmallSize = 32 <<"}]}]}
>```
>%%
>*%%PREFIX%%MaxSmallSize-1024)/128 + 1]int8%%HIGHLIGHT%% ==若对象大小超出特定阈值限制，会被当作大对象（large object）特别对待。== %%POSTFIX%%malloc.go _MaxSmallSize = 32 <<*
>%%LINK%%[[#^gz10oll4vef|show annotation]]
>%%COMMENT%%
>对象好像是超出了32kb就是被当作了大对象
>%%TAGS%%
>#内存分配
^gz10oll4vef


>%%
>```annotation-json
>{"created":"2022-03-02T08:59:56.552Z","text":"分配器由三种组件组成：cache,central,heap","updated":"2022-03-02T08:59:56.552Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":192973,"end":193100},{"type":"TextQuoteSelector","exact":"分配器由三种组件组成。  cache：每个运行期工作线程都会绑定一个 cache，用于无锁 object 分配。  central：为所有 cache 提供切分好的后备 span 资源。  heap：管理闲置 span，需要时向操作系统申请新内存。","prefix":"urceforge.net/doc/tcmalloc.html ","suffix":" mheap.go type mheap struct {  f"}]}]}
>```
>%%
>*%%PREFIX%%urceforge.net/doc/tcmalloc.html%%HIGHLIGHT%% ==分配器由三种组件组成。  cache：每个运行期工作线程都会绑定一个 cache，用于无锁 object 分配。  central：为所有 cache 提供切分好的后备 span 资源。  heap：管理闲置 span，需要时向操作系统申请新内存。== %%POSTFIX%%mheap.go type mheap struct {  f*
>%%LINK%%[[#^ge0zv677sg9|show annotation]]
>%%COMMENT%%
>分配器由三种组件组成：cache,central,heap
>%%TAGS%%
>#内存分配
^ge0zv677sg9


>%%
>```annotation-json
>{"created":"2022-03-02T09:10:17.725Z","text":"不包含大对象的分配和释放流程","updated":"2022-03-02T09:10:17.725Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":193621,"end":194054},{"type":"TextQuoteSelector","exact":"分配流程： 1．计算待分配对象对应的规格（size class）。 2．从 cache.alloc 数组找到规格相同的 span。 3．从 span.freelist 链表提取可用 object。 4．如 span.freelist 为空，从 central 获取新 span。 5．如 central.nonempty 为空，从 heap.free/freelarge 获取，并切分成 object 链表。 6．如 heap 没有大小合适的闲置 span，向操作系统申请新内存块。 释放流程： 1．将标记为可回收的 object 交还给所属 span.freelist。 2．该 span 被放回 central，可供任意 cache 重新获取使用。 3．如 span 已收回全部 object，则将其交还给 heap，以便重新切分复用。 4．定期扫描 heap 里长时间闲置的 span，释放其占用的内存。 注：以上不包括大对象，它直接从 heap 分配和回收。","prefix":"以 sizeclass 为索引管理多个用于分配的 span } ","suffix":"   第16章  内存分配 259 作为工作线程私有且不被共享的"}]}]}
>```
>%%
>*%%PREFIX%%以 sizeclass 为索引管理多个用于分配的 span }%%HIGHLIGHT%% ==分配流程： 1．计算待分配对象对应的规格（size class）。 2．从 cache.alloc 数组找到规格相同的 span。 3．从 span.freelist 链表提取可用 object。 4．如 span.freelist 为空，从 central 获取新 span。 5．如 central.nonempty 为空，从 heap.free/freelarge 获取，并切分成 object 链表。 6．如 heap 没有大小合适的闲置 span，向操作系统申请新内存块。 释放流程： 1．将标记为可回收的 object 交还给所属 span.freelist。 2．该 span 被放回 central，可供任意 cache 重新获取使用。 3．如 span 已收回全部 object，则将其交还给 heap，以便重新切分复用。 4．定期扫描 heap 里长时间闲置的 span，释放其占用的内存。 注：以上不包括大对象，它直接从 heap 分配和回收。== %%POSTFIX%%第16章  内存分配 259 作为工作线程私有且不被共享的*
>%%LINK%%[[#^4d77z2m7w6y|show annotation]]
>%%COMMENT%%
>不包含大对象的分配和释放流程
>%%TAGS%%
>#内存分配
^4d77z2m7w6y


>%%
>```annotation-json
>{"created":"2022-03-02T09:19:16.725Z","text":"span归还的好处","updated":"2022-03-02T09:19:16.725Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":194072,"end":194438},{"type":"TextQuoteSelector","exact":"作为工作线程私有且不被共享的  cache  是实现高性能无锁分配的核心，而  central  的作用是在多个 cache 间提高 object 利用率，避免内存浪费。 假如  cache1  获取一个  span  后，仅使用了一部分  object，那么剩余空间就可能会被浪费。而回收操作将该  span  交还给  central  后，该  span  完全可以被  cache2、cacheN  获取使用。此时，cache1 已不再持有该 span，完全不会造成问题。 将 span 归还给 heap，是为了在不同规格 object 需求间平衡。 某时段某种规格的  object  需求量可能激增，那么当需求过后，大量被切分成该规格的  span  就会被闲置浪费。将其归还给 heap，就可被其他需求获取，重新切分。","prefix":"接从 heap 分配和回收。   第16章  内存分配 259 ","suffix":" 16.2  初始化 因为内存分配器和垃圾回收算法都依赖连续地址"}]}]}
>```
>%%
>*%%PREFIX%%接从 heap 分配和回收。   第16章  内存分配 259%%HIGHLIGHT%% ==作为工作线程私有且不被共享的  cache  是实现高性能无锁分配的核心，而  central  的作用是在多个 cache 间提高 object 利用率，避免内存浪费。 假如  cache1  获取一个  span  后，仅使用了一部分  object，那么剩余空间就可能会被浪费。而回收操作将该  span  交还给  central  后，该  span  完全可以被  cache2、cacheN  获取使用。此时，cache1 已不再持有该 span，完全不会造成问题。 将 span 归还给 heap，是为了在不同规格 object 需求间平衡。 某时段某种规格的  object  需求量可能激增，那么当需求过后，大量被切分成该规格的  span  就会被闲置浪费。将其归还给 heap，就可被其他需求获取，重新切分。== %%POSTFIX%%16.2  初始化 因为内存分配器和垃圾回收算法都依赖连续地址*
>%%LINK%%[[#^jpwfwu6mhf|show annotation]]
>%%COMMENT%%
>span归还的好处
>%%TAGS%%
>
^jpwfwu6mhf


>%%
>```annotation-json
>{"created":"2022-03-02T09:21:32.024Z","text":"高性能内存管理结构","updated":"2022-03-02T09:21:32.024Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":194970,"end":195177},{"type":"TextQuoteSelector","exact":"简单点说，就是用三个数组组成一个高性能内存管理结构。 1．使用 arena 地址向操作系统申请内存，其大小决定了可分配用户内存的上限。 2．位图 bitmap 为每个对象提供 4 bit 标记位，用以保存指针、GC 标记等信息。 3．创建  span  时，按页填充对应  spans  空间。在回收  object  时，只须将其地址按页对齐后就可找到所属 span。分配器还用此访问相邻 span，做合并操作。","prefix":"区域从 Go 1.4 的 128 GB 提高到 512 GB。 ","suffix":" Go语言学习笔记 260 任何  arena  区域的地址，只"}]}]}
>```
>%%
>*%%PREFIX%%区域从 Go 1.4 的 128 GB 提高到 512 GB。%%HIGHLIGHT%% ==简单点说，就是用三个数组组成一个高性能内存管理结构。 1．使用 arena 地址向操作系统申请内存，其大小决定了可分配用户内存的上限。 2．位图 bitmap 为每个对象提供 4 bit 标记位，用以保存指针、GC 标记等信息。 3．创建  span  时，按页填充对应  spans  空间。在回收  object  时，只须将其地址按页对齐后就可找到所属 span。分配器还用此访问相邻 span，做合并操作。== %%POSTFIX%%Go语言学习笔记 260 任何  arena  区域的地址，只*
>%%LINK%%[[#^5vr19nvupd|show annotation]]
>%%COMMENT%%
>高性能内存管理结构
>%%TAGS%%
>
^5vr19nvupd


>%%
>```annotation-json
>{"created":"2022-03-02T09:22:16.886Z","text":"arena寻址spans,bitmap","updated":"2022-03-02T09:22:16.886Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":195191,"end":195287},{"type":"TextQuoteSelector","exact":"任何  arena  区域的地址，只要将其偏移量配以不同步幅和起始位置，就可快速访问与之对应的 spans、bitmap 数据。最关键的是，这三个数组可以按需同步线性扩张，无须预先分配内存。 ","prefix":"用此访问相邻 span，做合并操作。 Go语言学习笔记 260 ","suffix":"这些区域相关属性被保存在 heap 里，其中包括递进的分配位置 "}]}]}
>```
>%%
>*%%PREFIX%%用此访问相邻 span，做合并操作。 Go语言学习笔记 260%%HIGHLIGHT%% ==任何  arena  区域的地址，只要将其偏移量配以不同步幅和起始位置，就可快速访问与之对应的 spans、bitmap 数据。最关键的是，这三个数组可以按需同步线性扩张，无须预先分配内存。== %%POSTFIX%%这些区域相关属性被保存在 heap 里，其中包括递进的分配位置*
>%%LINK%%[[#^zaliqr4jkfg|show annotation]]
>%%COMMENT%%
>arena寻址spans,bitmap
>%%TAGS%%
>#内存分配
^zaliqr4jkfg


>%%
>```annotation-json
>{"created":"2022-03-02T09:24:20.294Z","text":"初始化工作","updated":"2022-03-02T09:24:20.294Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":195554,"end":195654},{"type":"TextQuoteSelector","exact":"初始化工作很简单： 1．创建对象规格大小对照表。 2．计算相关区域大小，并尝试从某个指定位置开始保留地址空间。 3．在 heap 里保存区域信息，包括起始位置和大小。 4．初始化 heap 其他属性。","prefix":" uintptr  arena_reserved bool } ","suffix":" malloc.go func mallocinit() {  "}]}]}
>```
>%%
>*%%PREFIX%%uintptr  arena_reserved bool }%%HIGHLIGHT%% ==初始化工作很简单： 1．创建对象规格大小对照表。 2．计算相关区域大小，并尝试从某个指定位置开始保留地址空间。 3．在 heap 里保存区域信息，包括起始位置和大小。 4．初始化 heap 其他属性。== %%POSTFIX%%malloc.go func mallocinit() {*
>%%LINK%%[[#^mrs1zfqd0e|show annotation]]
>%%COMMENT%%
>初始化工作
>%%TAGS%%
>#内存分配
^mrs1zfqd0e


>%%
>```annotation-json
>{"created":"2022-03-02T09:31:36.403Z","text":"内联优化和逃逸分析","updated":"2022-03-02T09:31:36.403Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":201036,"end":201293},{"type":"TextQuoteSelector","exact":"看不懂汇编没关系，但显然内联优化后的代码没有调用 newobject 在堆上分配内存。 编译器这么做，道理很简单。没有内联时，需要在两个栈帧间传递对象，因此会在堆上分配而不是返回一个失效栈帧里的数据。而当内联后，它实际上就成了  main  栈帧内的局部变量，无须去堆上操作。 Go  编译器支持逃逸分析（escape  analysis），它会在编译期通过构建调用图来分析局部变量是否会被外部引用，从而决定是否可直接分配在栈上。 编译参数 -gcflags \"-m\" 可输出编译优化信息，其中包括内联和逃逸分析。","prefix":"Q $0x18, SP     test.go:13  RET ","suffix":"   第16章  内存分配 267 你或许见过“Zero  Ga"}]}]}
>```
>%%
>*%%PREFIX%%Q $0x18, SP     test.go:13  RET%%HIGHLIGHT%% ==看不懂汇编没关系，但显然内联优化后的代码没有调用 newobject 在堆上分配内存。 编译器这么做，道理很简单。没有内联时，需要在两个栈帧间传递对象，因此会在堆上分配而不是返回一个失效栈帧里的数据。而当内联后，它实际上就成了  main  栈帧内的局部变量，无须去堆上操作。 Go  编译器支持逃逸分析（escape  analysis），它会在编译期通过构建调用图来分析局部变量是否会被外部引用，从而决定是否可直接分配在栈上。 编译参数 -gcflags "-m" 可输出编译优化信息，其中包括内联和逃逸分析。== %%POSTFIX%%第16章  内存分配 267 你或许见过“Zero  Ga*
>%%LINK%%[[#^77c75x10xh4|show annotation]]
>%%COMMENT%%
>内联优化和逃逸分析
>%%TAGS%%
>#内存分配
^77c75x10xh4


>%%
>```annotation-json
>{"created":"2022-03-02T09:37:34.391Z","text":"malloc.go中mallocgc的逻辑，分配的基本思路","updated":"2022-03-02T09:37:34.391Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":204240,"end":204560},{"type":"TextQuoteSelector","exact":"整理一下这段代码的基本思路：  大对象直接从 heap 获取 span。  小对象从 cache.alloc[sizeclass].freelist 获取 object。  微小对象组合使用 cache.tiny object。 对微小对象的处理很有意思。首先，它不能是指针，因为多个小对象被组合到一个 object  里，显然无法应对垃圾扫描。其次，它从  span.freelist  获取一个  16  字节的 object，然后利用偏移量来记录下一次分配的位置。 这里有个小细节，体现了作者的细心。当 tiny 因剩余空间不足而使用新 object 时，会比较新旧两个 tiny object 的剩余空间，而非粗暴地喜新厌旧。","prefix":" 检查触发条件，启动垃圾回收 ...   return x } ","suffix":" 分配算法本身并不复杂，没什么好说的，接下来要关注的自然是资源不"}]}]}
>```
>%%
>*%%PREFIX%%检查触发条件，启动垃圾回收 ...   return x }%%HIGHLIGHT%% ==整理一下这段代码的基本思路：  大对象直接从 heap 获取 span。  小对象从 cache.alloc[sizeclass].freelist 获取 object。  微小对象组合使用 cache.tiny object。 对微小对象的处理很有意思。首先，它不能是指针，因为多个小对象被组合到一个 object  里，显然无法应对垃圾扫描。其次，它从  span.freelist  获取一个  16  字节的 object，然后利用偏移量来记录下一次分配的位置。 这里有个小细节，体现了作者的细心。当 tiny 因剩余空间不足而使用新 object 时，会比较新旧两个 tiny object 的剩余空间，而非粗暴地喜新厌旧。== %%POSTFIX%%分配算法本身并不复杂，没什么好说的，接下来要关注的自然是资源不*
>%%LINK%%[[#^1qnxx8l2eu2|show annotation]]
>%%COMMENT%%
>malloc.go中mallocgc的逻辑，分配的基本思路
>%%TAGS%%
>#内存分配
^1qnxx8l2eu2


>%%
>```annotation-json
>{"created":"2022-03-02T09:50:18.318Z","text":"从heap获取span的算法逻辑","updated":"2022-03-02T09:50:18.318Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":209211,"end":209358},{"type":"TextQuoteSelector","exact":"从  heap  获取  span  的算法核心是找到大小最合适的块。首先从页数相同的链表查找，如没有结果，再从页数更多的链表提取，直至超大块或申请新块。 如返回更大的  span，为避免浪费，会将多余部分切出来重新放回  heap  链表。同时还尝试合并相邻的闲置 span 空间，减少碎片。","prefix":"ge, s)    }   }  }   return s } ","suffix":" mheap.go type mheap struct {  f"}]}]}
>```
>%%
>*%%PREFIX%%ge, s)    }   }  }   return s }%%HIGHLIGHT%% ==从  heap  获取  span  的算法核心是找到大小最合适的块。首先从页数相同的链表查找，如没有结果，再从页数更多的链表提取，直至超大块或申请新块。 如返回更大的  span，为避免浪费，会将多余部分切出来重新放回  heap  链表。同时还尝试合并相邻的闲置 span 空间，减少碎片。== %%POSTFIX%%mheap.go type mheap struct {  f*
>%%LINK%%[[#^n47arx9lv0l|show annotation]]
>%%COMMENT%%
>从heap获取span的算法逻辑
>%%TAGS%%
>#内存分配
^n47arx9lv0l


>%%
>```annotation-json
>{"created":"2022-03-02T09:53:12.919Z","text":"内存回收的思想","updated":"2022-03-02T09:53:12.919Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":212575,"end":212770},{"type":"TextQuoteSelector","exact":"内存回收的源头是垃圾清理操作。 之所以说回收而非释放，是因为整个内存分配器的核心是内存复用，不再使用的内存会被放回合适位置，等下次分配时再次使用。只有当空闲内存资源过多时，才会考虑释放。 基于效率考虑，回收操作自然不会直接盯着单个对象，而是以 span 为基本单位。通过比对 bitmap 里的扫描标记，逐步将 object 收归原 span，最终上交 central 或 heap 复用。","prefix":" 0) } 至此，内存分配操作流程正式结束。 16.4  回收 ","suffix":" 清理函数 sweepone 调用 mSpan_Sweep 来引"}]}]}
>```
>%%
>*%%PREFIX%%0) } 至此，内存分配操作流程正式结束。 16.4  回收%%HIGHLIGHT%% ==内存回收的源头是垃圾清理操作。 之所以说回收而非释放，是因为整个内存分配器的核心是内存复用，不再使用的内存会被放回合适位置，等下次分配时再次使用。只有当空闲内存资源过多时，才会考虑释放。 基于效率考虑，回收操作自然不会直接盯着单个对象，而是以 span 为基本单位。通过比对 bitmap 里的扫描标记，逐步将 object 收归原 span，最终上交 central 或 heap 复用。== %%POSTFIX%%清理函数 sweepone 调用 mSpan_Sweep 来引*
>%%LINK%%[[#^n5s465yuyok|show annotation]]
>%%COMMENT%%
>内存回收的思想
>%%TAGS%%
>#内存分配
^n5s465yuyok


>%%
>```annotation-json
>{"created":"2022-03-02T10:04:12.712Z","text":"释放逻辑","updated":"2022-03-02T10:04:12.712Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":216043,"end":216111},{"type":"TextQuoteSelector","exact":"在运行时入口函数  main.main  里，会专门启动一个监控任务  sysmon，它每隔一段时间就会检查 heap 里的闲置内存块。","prefix":"回的 span 并不会被释放，而是等待复用。 16.5  释放 ","suffix":" proc.go func sysmon() {  scaven"}]}]}
>```
>%%
>*%%PREFIX%%回的 span 并不会被释放，而是等待复用。 16.5  释放%%HIGHLIGHT%% ==在运行时入口函数  main.main  里，会专门启动一个监控任务  sysmon，它每隔一段时间就会检查 heap 里的闲置内存块。== %%POSTFIX%%proc.go func sysmon() {  scaven*
>%%LINK%%[[#^7hoe1c3rcwc|show annotation]]
>%%COMMENT%%
>释放逻辑
>%%TAGS%%
>#内存分配
^7hoe1c3rcwc


>%%
>```annotation-json
>{"created":"2022-03-02T10:04:37.995Z","text":"遍历free,freeLarge里的所有span，如果闲置时间超过阈值，则释放这个内存。","updated":"2022-03-02T10:04:37.995Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":216344,"end":216394},{"type":"TextQuoteSelector","exact":"遍历 free、freelarge 里的所有 span，如闲置时间超过阈值，则释放其关联的物理内存。","prefix":"    lastscavenge = now   }  } } ","suffix":" mheap.go func mHeap_Scavenge(k "}]}]}
>```
>%%
>*%%PREFIX%%lastscavenge = now   }  } }%%HIGHLIGHT%% ==遍历 free、freelarge 里的所有 span，如闲置时间超过阈值，则释放其关联的物理内存。== %%POSTFIX%%mheap.go func mHeap_Scavenge(k*
>%%LINK%%[[#^pk4nt2r7ls|show annotation]]
>%%COMMENT%%
>遍历free,freeLarge里的所有span，如果闲置时间超过阈值，则释放这个内存。
>%%TAGS%%
>#内存分配
^pk4nt2r7ls


>%%
>```annotation-json
>{"created":"2022-03-02T10:07:29.257Z","text":"runtime对象分为两类","updated":"2022-03-02T10:07:29.257Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":218187,"end":218300},{"type":"TextQuoteSelector","exact":"从运行时的角度，整个进程内的对象可分为两类：一种，自然是从  arena  区域分配的用户对象；另一种，则是运行时自身运行和管理所需的对象，比如管理  arena  内存片段的 mspan，提供无锁分配的 mcache 等等。","prefix":"g.FreeOSMemory 函数主动释放。 16.6  其他 ","suffix":" Go语言学习笔记 286 管理对象的生命周期并不像用户对象那样"}]}]}
>```
>%%
>*%%PREFIX%%g.FreeOSMemory 函数主动释放。 16.6  其他%%HIGHLIGHT%% ==从运行时的角度，整个进程内的对象可分为两类：一种，自然是从  arena  区域分配的用户对象；另一种，则是运行时自身运行和管理所需的对象，比如管理  arena  内存片段的 mspan，提供无锁分配的 mcache 等等。== %%POSTFIX%%Go语言学习笔记 286 管理对象的生命周期并不像用户对象那样*
>%%LINK%%[[#^ikohmrqdms|show annotation]]
>%%COMMENT%%
>runtime对象分为两类
>%%TAGS%%
>#内存分配
^ikohmrqdms


>%%
>```annotation-json
>{"created":"2022-03-02T10:08:03.315Z","text":"管理对象的分配使用了FixAlloc固定分配器","updated":"2022-03-02T10:08:03.315Z","document":{"title":"","link":[{"href":"urn:x-pdf:1c81eb70d4a5aef032dd3cac411fbcd2"},{"href":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf"}],"documentFingerprint":"1c81eb70d4a5aef032dd3cac411fbcd2"},"uri":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","target":[{"source":"vault:/%E5%9B%BE%E4%B9%A6%E9%A6%86/Golang/Go%E8%AF%AD%E8%A8%80%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.pdf","selector":[{"type":"TextPositionSelector","start":218314,"end":218468},{"type":"TextQuoteSelector","exact":"管理对象的生命周期并不像用户对象那样复杂，且类型和长度都相对固定，所以算法策略显然不用那么复杂。还有，它们相对较长的生命周期也不适合占用  arena  区域，否则会导致更多碎片。为此，运行时专门设计了  FixAlloc  固定分配器来为管理对象分配内存。 固定分配器使用相同的算法框架，只有相应参数不同。","prefix":"提供无锁分配的 mcache 等等。 Go语言学习笔记 286 ","suffix":" mfixalloc.go type fixalloc stru"}]}]}
>```
>%%
>*%%PREFIX%%提供无锁分配的 mcache 等等。 Go语言学习笔记 286%%HIGHLIGHT%% ==管理对象的生命周期并不像用户对象那样复杂，且类型和长度都相对固定，所以算法策略显然不用那么复杂。还有，它们相对较长的生命周期也不适合占用  arena  区域，否则会导致更多碎片。为此，运行时专门设计了  FixAlloc  固定分配器来为管理对象分配内存。 固定分配器使用相同的算法框架，只有相应参数不同。== %%POSTFIX%%mfixalloc.go type fixalloc stru*
>%%LINK%%[[#^btlteeagcrf|show annotation]]
>%%COMMENT%%
>管理对象的分配使用了FixAlloc固定分配器
>%%TAGS%%
>#内存分配
^btlteeagcrf
