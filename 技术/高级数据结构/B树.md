
# BTree实现原理

发布于2022-08-15 15:00:30阅读 5240

小编在看etcd存储（store）模块的时候，发现它在进行key和keyIndex转换的时候，用到了btree包(http://godoc.org/github.com/google/btree)。btree是Google开源的一个Go语言的BTree实现，整个代码不到1000行，实现的非常简练，组织分层也做的很好，并对gc和并发读写做了很多优化，值得一读。小编打算用两篇文章讲解BTree内容，本文上篇主要介绍实现原理，下篇主要介绍btree源码实现。

## **BTree定义**

BTree和B-Tree都是指B树，不要把B-Tree理解成了B-树。关于BTree在《计算机程序设计与艺术》和《算法导论》对其描述是不一样的。一种是通过"阶"的概念来定义的，另一种是通过"度"的概念定义的，两者描述的本质是一样的。

我们先来看通过阶的概念的定义，阶指一个节点的最大子树的个数，定义如下：

> ❝树中每个节点至多有m颗子树 若根节点不是叶子节点，则至少有两颗子树 除根节点之外的所有非终端节点至少有「m/2」颗子树 所有的非终端节点中包含关键字和指向子树根节点的指针 所有叶子节点在同一个层次上 ❞

下面是算法导论中通过度来定义BTree, 度是一个节点子树的个数，对于一颗最小度为t(t>=2)的BTree,节点中的关键字为key,有如下限制：

> ❝除了根节点之外，其他节点的t-1<=len(key)<=2t-1 非叶子节点的子树个数t<=len(children)<=2t ❞

两者有点细微的差别，根据《计算机程序设计艺术》的定义，最小的树为3阶BTree,即树的非叶子节点的子树的个数为2或3，根据《算法导论》的定义，最小的树为2度数，对应的前者的阶数为4，即树的非叶子节点个数可以为2、3或者4. etcd中用到的btree包是按这里的度的定义实现的。下图是一个3阶的BTree，除了叶子节点，每个节点的子树个数不是2个就是3个，0004节点的子树有2个，0047|0051节点的子树有3个。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/04cf126e8e4aeef6c3128cefb3292ea8.jpeg?imageView2/2/w/2560/h/7000)

## **BTree使用场景**

BTree常用于实现[数据库](https://cloud.tencent.com/solution/database?from=20065&from_column=20065)索引，例如在[MongoDB](https://cloud.tencent.com/product/mongodb?from=20065&from_column=20065)中的索引是用BTree实现的，[MySQL](https://cloud.tencent.com/product/cdb?from=20065&from_column=20065)中的innodb存储引擎用B+树存储索引信息。通过前面的定义可以看到，BTree是一种平衡多路查找树，与AVL树和红黑树等二叉树比较起来，BTree通过多叉，降低了树的高度，从而减少了查询的次数。为啥数据库的索引采用BTree实现呢？因为数据库的索引信息以树形结构存放在磁盘上，对于高度为h的树，最多需要进行h次查找，对于存放在磁盘上的文件来说，需要读取磁盘h次，而读取磁盘的操作与操作内存相比是很慢的，一次磁盘读取耗时为寻道时间+旋转磁头时间+数据读取传输耗时。普通的机械硬盘，寻道时间大概在10毫秒左后，旋转磁头时间就是磁头移动到对应磁道后，旋转到对应扇区所需要的时间，可以看到整个磁盘IO操作是个很耗时的过程。而BTree降低了树的高度，减少了磁盘读取次数，所以数据库的索引采用BTree或B+树实现。

## **BTree实现原理**

BTree的核心操作包含树的创建，树中节点的删除，元素的查找。下面分别分析每个操作的具体实现。

### **插入**

插入操作是向BTree中插入一条记录，即向里面添加一个key-value键值对。如果要插入的key在BTree中已存在，则用当前的value更新之前的旧value.如果BTree中不存在这个key,则在它的叶子节点中进行插入操作。具体算法流程如下：

1.  根据要插入的key的值，在BTree查找key存不存，如果已存在，用新的value覆盖旧的value，操作结束
2.  如果流程1中，没有找到key，则定位到要插入的叶子节点并插入
3.  判断2中刚插入节点key的个数是否满足BTree的性质，如果不满足，则执行下面的第4步操作
4.  以插入节点中间的key为中心，分裂成左右两部分，然后将中间的key插入到它的父节点中，这个key的左子树指向分裂后的左半部分，右子树指向分裂后的右半部分。然后继续判断父节点满不满足BTree性质，如果不满足，继续对父节点进行分裂，否则流程执行结束

下面以`3阶段BTree`的创建过程为例，介绍插入过程。

向BTree中插入4，插入后只有一个key，因为这是首次插入。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/34879510b222a259c6657bc62fef614f.jpeg?imageView2/2/w/2560/h/7000)

向BTree中插入51，直接将51加入与4同节点中，此时该节点有2个key，满足每个节点不超过2个key的性质.

![](https://ask.qcloudimg.com/http-save/yehe-9955853/0da16b5dd9ebf0a27bc44d2ea99e860e.jpeg?imageView2/2/w/2560/h/7000)

向BTree中插入38, 度为3的BTree，每个节点最多有2个key, 此时节点有3个key，不满足BTree性质，将中间的key提升到父节点中,调整之后符合BTree定义，插入操作结束。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/a2aa0ee38a2cea96d8960718763bd213.jpeg?imageView2/2/w/2560/h/7000)

向BTree中插入43，添加到叶子节点51所在的位置。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/443a488468217f30ffc6d74e35802756.jpeg?imageView2/2/w/2560/h/7000)

向BTree中插入48，添加48到43|51所在的节点后，此时该节点不满足BTree性质，对其进行拆分，将中间的48加入到父节点（38所在的节点），43|48|51节点中的key被分成43和51两部分，48的左右子树分别指向它。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/37a7dbffa22faa37e277910028cf40ee.jpeg?imageView2/2/w/2560/h/7000)

向BTree中插入1

![](https://ask.qcloudimg.com/http-save/yehe-9955853/c8dcc548eca3814879dbd0108162f801.jpeg?imageView2/2/w/2560/h/7000)

向BTree中插入10，此时1|4|10节点不满足BTree性质，需要进行分裂，将4插入到父节点中，插入之后，父节点4|30|48也不满足BTree性质，继续对其进行分裂。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/8bc2592c4941226803f47f0c13696602.jpeg?imageView2/2/w/2560/h/7000)

插入的核心就是当节点的key的数量不满足BTree性质时，向上进行分裂，即将当前节点的中间key插入到父节点中，这样能够确保分裂之后节点的层次深度保持不变。如果父节点也不满足BTree性质，继续对其分裂，这样一路向上，直到根节点，保证整颗树🌲层次一致，即多路平衡性。

### **删除**

从BTree中删除一个元素，根据删除元素的所在的节点是叶子节点还是非叶子节点分为两种情况。下面对这两种情况做一个简单的分析：

1.  删除元素在非叶子节点：将下面BTree中的元素38删除，如果直接删除，这时候root节点只要一个元素了，但它有3个子节点，不满足BTree性质。那怎么做呢？可以将以38为根节点的子树的最右侧叶子节点的最后一个元素放入到38的位置，然后从叶子节点中删除放入的元素，这时候完美的符合BTree性质，整个数又是平衡的。对应到这里就是将元素4覆盖掉38，然后叶子节点1|4中的4元素删除。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/d4a007debe6b20eea88f9372c176bd68.jpeg?imageView2/2/w/2560/h/7000)

删除元素38后，得到新的BTree树如下

![](https://ask.qcloudimg.com/http-save/yehe-9955853/bf1b982aa38686cb74ff15144d3fcc8a.jpeg?imageView2/2/w/2560/h/7000)

但是这里有一种特殊情况，如果将要删除的元素的子树的最右侧的叶子节点的元素移走之后，可能会导致叶子节点为空，此时叶子节点不满足BTree性质。这种情况只做上面的处理还不够，还要做其他处理。这种情况在2中删除叶子节点的元素导致叶子节点为空也存在，所以放到下面一起分析说明。

1.  删除元素在叶子节点：如果要删除的元素所在的叶子节点，在删除该元素之后不为空，则可以直接将其删除，如果删除之后，叶子节点变为了空，这时还要做其他处理，上面1中的特殊情况下也可能存在该问题。

处理方法是，如果该叶子节点的兄弟节点有富余的元素，可以借一个过来，如果兄弟节点也没有富余的元素，这时候要对叶子节点进行合并。

下面的BTree删除叶子节点中的元素z, 删除z之后节点C不满足BTree性质。假定该BTree是m阶BTree,根据BTree性质，非根节点元素的个数n, 则n满足ceil(m/2)-1<=n<=m-1. 那说明删除z之后，节点C的元素个数为 ceile(m/2）-2. 其兄弟节点A的元素个数 > ceil(m/2)-1,即节点A最少元素个数为 ceil(m/2）,将元素x加入到节点C元素的最左边，然后将节点A最后一个元素y放入到x位置，最后将节点A中的元素y删除，此时又是BTree了。

删除元素z之前

![](https://ask.qcloudimg.com/http-save/yehe-9955853/bcaa089ed73baaf5cba55de272278a72.jpeg?imageView2/2/w/2560/h/7000)

删除元素z之后

![](https://ask.qcloudimg.com/http-save/yehe-9955853/3cd184499ddde69bd1e0d65b078d647f.jpeg?imageView2/2/w/2560/h/7000)

上面讨论的是叶子节点的兄弟节点有富余的元素，可以从兄弟节点借。如果兄弟节点的元素也不富余的情况怎么处理呢？嗯，方法是将叶子节点进行合并。下面的m阶BTree中删除叶子节点C中的元素z之后，不满足BTree性质，说明此时节点C在删除z之后剩余的节点数为ceil(m/2)-2, 其兄弟节点A没有富余元素，说明节点A的元素个数为ceil(m/2)-1.处理方法是将元素x下移，与节点A和节点C合并，合并之后的节点我们记为节点AC,则节点AC的元素个数为 ceil(m/2)-1+ceil(m/2)-2+1 = ceil(m/2)+ceil(m/2)-2, 可以证明合并之后的节点数是小等于m-1的，即ceil(m/2)+ceil(m/2)-2<=m-1. 这样合并之后，又是BTree了。如果合并之后节点B不满足BTree性质，因为从节点B移走了元素x, 则处理方式就是这里介绍的两种方法，从节点B的兄弟节点借，如果兄弟节点没有富余元素，则继续合并，最后直到根节点，会将BTree的高度降低。

删除元素z之前

![](https://ask.qcloudimg.com/http-save/yehe-9955853/6c46db57ba10ad28b4191adc8cff5ae1.jpeg?imageView2/2/w/2560/h/7000)

删除元素z之后

![](https://ask.qcloudimg.com/http-save/yehe-9955853/4c1787d4ccbe53b1f879e70d8691ba35.jpeg?imageView2/2/w/2560/h/7000)

下面是一颗3阶的BTree， 现在删除元素90

![](https://ask.qcloudimg.com/http-save/yehe-9955853/c413657f34b4b24e7efe9ade0aa3b313.jpeg?imageView2/2/w/2560/h/7000)

删除90之后，叶子节点变为空，此时不满足BTree性质，根据前面介绍的处理方法，尝试从它的兄弟节点借一个元素，但是它的兄弟节点也只有一个元素44，只能进行合并处理，将父节点的元素45和44合并。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/03c3e4e3be154a545d02e2b106d84ad0.jpeg?imageView2/2/w/2560/h/7000)

但此时父节点中的元素为空了，不满足BTree性质，于是对父节点采用从它的兄弟节点借或者合并的方法，而此时它的兄弟节点中也只有一个元素22，所以只能进行合并，将根节点的中的元素41和21合并，BTree的高度减少一层，得到如下的BTree.

![](https://ask.qcloudimg.com/http-save/yehe-9955853/8590424d0383d4f1ebd51edf9f21ce07.jpeg?imageView2/2/w/2560/h/7000)

### **查找**

BTree是一种多路平衡树，同时也满足有序性，对于每个节点，它左边子树的所有元素都小于该节点中最小的元素，它右边子树的所有元素都大于该节点中最大的元素。每个节点的内部元素也是有序的。所以采用类似二叉树的中序遍历，得到元素一定是有序的。所以BTree中查找元素的过程很简单，从根节点开始，每次可以定位可能所在的1个子节点，这样一路向下查询，如果在内部节点中没有找到，最后达到叶子节点，如果叶子节点也没有，则说明要查询的元素不在BTree中，整个查找过程很简单，这里就不在画图举例说明了。

# 2 B+树

#### 2.1 B+树概述

B+树其实和B树是非常相似的，我们首先看看**相同点**。

-   根节点至少一个元素
-   非根节点元素范围：m/2 <= k <= m-1

**不同点**。

-   B+树有两种类型的节点：内部结点（也称索引结点）和叶子结点。内部节点就是非叶子节点，内部节点不存储数据，只存储索引，数据都存储在叶子节点。
-   内部结点中的key都按照从小到大的顺序排列，对于内部结点中的一个key，左树中的所有key都小于它，右子树中的key都大于等于它。叶子结点中的记录也按照key的大小排列。
-   每个叶子结点都存有相邻叶子结点的指针，叶子结点本身依关键字的大小自小而大顺序链接。
-   父节点存有右孩子的第一个元素的索引。

下面我们看一个B+树的例子，感受感受它吧！

![](https://segmentfault.com/img/remote/1460000020416595)

#### 2.2 插入操作

对于插入操作很简单，只需要记住一个技巧即可：**当节点元素数量大于m-1的时候，按中间元素分裂成左右两部分，中间元素分裂到父节点当做索引存储，但是，本身中间元素还是分裂右边这一部分的**。

下面以一颗5阶B+树的插入过程为例，5阶B+树的节点最少2个元素，最多4个元素。

-   插入5，10，15，20

![](https://segmentfault.com/img/remote/1460000020416596)

-   插入25，此时元素数量大于4个了，分裂

![](https://segmentfault.com/img/remote/1460000020416597)

-   接着插入26，30，继续分裂

![](https://segmentfault.com/img/remote/1460000020416598)

![](https://segmentfault.com/img/remote/1460000020416599)

有了这几个例子，相信插入操作没什么问题了，下面接着看看删除操作。

#### 2.3 删除操作

对于删除操作是比B树简单一些的，因为**叶子节点有指针的存在，向兄弟节点借元素时，不需要通过父节点了，而是可以直接通过兄弟节移动即可（前提是兄弟节点的元素大于m/2），然后更新父节点的索引；如果兄弟节点的元素不大于m/2（兄弟节点也没有多余的元素），则将当前节点和兄弟节点合并，并且删除父节点中的key**，下面我们看看具体的实例。

-   初始状态

![](https://segmentfault.com/img/remote/1460000020416600)

-   删除10，删除后，不满足要求，发现左边兄弟节点有多余的元素，所以去借元素，最后，修改父节点索引

![](https://segmentfault.com/img/remote/1460000020416601)

-   删除元素5，发现不满足要求，并且发现左右兄弟节点都没有多余的元素，所以，可以选择和兄弟节点合并，最后修改父节点索引

![](https://segmentfault.com/img/remote/1460000020416602)

-   发现父节点索引也不满足条件，所以，需要做跟上面一步一样的操作

![](https://segmentfault.com/img/remote/1460000020416603)

这样，B+树的删除操作也就完成了，是不是看完之后，觉得非常简单！

### 3 B树和B+树总结

B+树相对于B树有一些自己的优势，可以归结为下面几点。

-   单一节点存储的元素更多，使得查询的IO次数更少，所以也就使得它更适合做为数据库MySQL的底层数据结构了。
-   所有的查询都要查找到叶子节点，查询性能是稳定的，而B树，每个节点都可以查找到数据，所以不稳定。
-   所有的叶子节点形成了一个有序链表，更加便于查找。


# BTree源码分析

发布于2022-08-15 15:01:12阅读 2490

本文是介绍BTree文章的下篇，在[BTree实现原理](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488409&idx=1&sn=7417c5e5f959ee96a7a221aa41a7dcdb&chksm=873a6e58b04de74e4fbb92a1e630876b8124b410047702c3f80f87cc66f682b4428324e05426&scene=21#wechat_redirect)上篇主要介绍实现原理，下篇主要介绍btree源码实现。

**t度 的[B树](https://www.zhihu.com/search?q=B%E6%A0%91&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)就是 2t阶 的B树(这也是B树的分裂机制决定的)**

**因为t度的B树节点最多有2t个孩子，2t-1个关键字；[m阶](https://www.zhihu.com/search?q=m%E9%98%B6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)的B树最多有m个孩子，**

**其实通过度定义的B树和通过阶数定义的B树，区别就是一个是用的这个B树节点的[最小度数](https://www.zhihu.com/search?q=%E6%9C%80%E5%B0%8F%E5%BA%A6%E6%95%B0&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)一个是用的这个树节点的最大度数。**

  

一棵m阶的B树满足下列条件：

1.树中每个结点至多有m个孩子。

2.除根结点和[叶子结点](https://www.zhihu.com/search?q=%E5%8F%B6%E5%AD%90%E7%BB%93%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)外，其它每个结点至少有m/2个孩子。

3.根结点至少有2个孩子（如果B树只有一个结点除外）,这条性质是由B树的插入分裂策略决定的。

4.所有[叶结点](https://www.zhihu.com/search?q=%E5%8F%B6%E7%BB%93%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)在同一层，B树的叶结点可以看成一种外部节点，不包含任何信息。

4.有k个关键字(关键字按递增次序排列)的[非叶结点](https://www.zhihu.com/search?q=%E9%9D%9E%E5%8F%B6%E7%BB%93%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)恰好有k+1个孩子。

5.一个节点如果由n个关键字，则节点内数据结构为P0，K1，P1，K2，P2.........Pn-1 Kn Pn 其中 p为指向其子节点的指针，因为父子的大小 关系和节点内大小关系，满足Kj 大于Pj指针所指向的子树上的所有关键字小雨Pj+1指针所指向子树上的所有关键字

  

  

------------------------------------------------

一个t度的B树满足:

1) All leaves are at same level.

所有叶子节点均在同一层

2) A B-Tree is defined by the term minimum degree ‘t’. The value of t depends upon disk block size.

B树可以用最小的度t定义，其中度t的值由磁盘块的大小来决定

3) Every node except root must contain at least t-1 keys. Root may contain minimum 1 key.

所有的节点除了根节点外都至少有t-1个关键字，根节点可以有最少的节点树1

4) All nodes (including root) may contain at most 2t – 1 keys.

所有的[节点数](https://www.zhihu.com/search?q=%E8%8A%82%E7%82%B9%E6%95%B0&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D)(包括根节点)都包含至多2t-1个关键字

5) Number of children of a node is equal to the number of keys in it plus 1.

节点的孩子数等于关键字数目+1

6) All keys of a node are sorted in increasing order. The child between two [keys](https://www.zhihu.com/search?q=keys&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A223588685%7D) k1 and k2 contains all keys in range from k1 and k2.

所有节点内的数据有序的且递增。在k1关键字和k2关键字之间的孩子树上的所有关键字都大于K1且小于K2

7) B-Tree grows and shrinks from root which is unlike Binary Search Tree. Binary Search Trees grow downward and also shrink from downward.

8) Like other balanced Binary Search Trees, time complexity to search, insert and delete is O(Logn)

## **BTree数据结构**

BTree是一种多路平衡树，多路通过度(degree)来体现，树通过记录根节点来表达。所以BTree结构必须包含描述树的度和根节点信息。下面是BTree结构体定义，结构体中的degree和root字段，正是我们这里分析的BTree必须有的基本信息,此外还定义了一个length字段，它表示BTree中有多少个元素，cow字段起辅助优化作用，用于在多颗BTree中达到节点node复用的作用，减少gc开销。
BTree结构体定义

```go
type BTree struct {
 // BTree的阶
 degree int
 // BTree中元素的数量
 length int
 root   *node
 cow    *copyOnWriteContext
}
```



node描述BTree中节点的信息，因为每个节点包含元素信息和子节点的信息，这是节点的最基本的信息，下面的node结构体定义中，items表示元素信息，children表示子节点的信息。

items是一个别名类型，它的实际类型为Item的切片。类似的，children是\*node的切片。cow是优化使用的，包含写时的上下文信息。详细的作用在分析BTree的Clone时介绍。items和children都是切片的别名，这样做是为了代码的封装处理。

节点node结构

![](https://ask.qcloudimg.com/http-save/yehe-9955853/f84aadddeaea9911c3da63c58f11e70f.jpeg?imageView2/2/w/2560/h/7000)

node结构体定义

```go
type node struct {
 items    items
 children children
 cow      *copyOnWriteContext
}

type items []Item

type children []*node
```



Item描述的是BTree中node中的元素信息，对于btree库的实现者来说，要容纳任意的元素类型，如果是我们来定义Item类型，因为interface{}能包罗万象，容纳任意类型的类型，所以会将Item定义为空接口类型。但btree库中将Item定义为含有Less方法的interface类型，所以Item不仅是interface类型，而是一个实现Less方法的接口类型。这样在BTree的创建、查找和删除等操作时，对使用者来说也更方便。

BTree中的元素Item定义

```go
type Item interface {
 Less(than Item) bool
}
```



## **BTree库函数**

BTree库函数是btree包提供给外部使用的API接口，也就是上面BTree结构体提供给外部的方法以及BTree的构造函数。这里做了一个汇总说明。
![[Pasted image 20230413162434.png]]
![[Pasted image 20230413162452.png]]
![[Pasted image 20230413162506.png]]

## **BTree基本操作实现**

插入、删除和查询是BTree最基本的操作，在[BTree实现原理](http://mp.weixin.qq.com/s?__biz=MzA3MzIxMjY1NA==&mid=2648488409&idx=1&sn=7417c5e5f959ee96a7a221aa41a7dcdb&chksm=873a6e58b04de74e4fbb92a1e630876b8124b410047702c3f80f87cc66f682b4428324e05426&scene=21#wechat_redirect)文章中介绍了它们的实现原理，此处将从源码的角度分析这几种操作的实现。

### **插入**

向BTree中插入元素的逻辑实现方法为ReplaceOrInsert，该方法只有一个入参，即待插入的元素，返回值为一个Item类型，如果待插入的元素已在BTree中，则返回的值为旧元素，如果不存在，返回的Item为nil.

整个插入逻辑有点复杂，可以分成3部分来理解。第一部分逻辑处理的是首次向BTree中插入元素的情况，这个时候BTree的根节点还不存在，先创建一个根节点，然后将待插入的元素item添加到根节点中，更新下BTree中的元素的数量t.length,整个处理已结束可以返回了。

第二部分，也就是下面代码part 2处的逻辑块。检查根节点的元素数量是否达到最大值(2\*degree-1)，如果达到最大值，则需要对根节点进行分裂成两部分，同时需要创建一个新的节点作为根节点，树会长高一层。待分裂节点的中间元素移动到新的根节点中，最后将分裂的两个节点添加到新的根节点中，作为它的左右子节点。

最后的第三部分是插入的核心逻辑，先说明一点，**「向BTree插入元素都是在叶子节点插入」**.一路从BTree的根节点向下定位到要插入的叶子的节点位置，在定位的过程中，遇到节点的元素个数达到最大值(2\*degree-1)，则要对该节点进行拆分，并将中间的元素加入到父节点中，即产生上溢操作. 插入实现在insert方法中，见下面的详细分析。

```go
func (t *BTree) ReplaceOrInsert(item Item) Item {
 // 检查待插入元素，不能为nil
 if item == nil {
  panic("nil item being added to BTree")
 }
 // part 1
 // 向BTree中首次插入元素，此时BTree中根节点为空，创建一个根节点
 // 并将插入元素加入到根节点
 if t.root == nil {
  t.root = t.cow.newNode()
  t.root.items = append(t.root.items, item)
  t.length++
  return nil
 } else {
  // part 2
  // 检查根节点中元素的数量是否达到最大数量(2*degree-1),如果达到最大数量
  // 则对根节点从中间元素的位置分裂成2个节点，然后创建一个新的根节点，将中间
  // 元素添加到新的根节点中，最后将分裂的两个节点添加到新的根的子节点中，
  // 构成它的左右子节点，此时BTree会长高一层
  t.root = t.root.mutableFor(t.cow)
  // 检查根节点中元素的数量是否达到分裂条件
  if len(t.root.items) >= t.maxItems() {
   // 将根节点分裂成两部分，即将根节点 [first][item2][second]分裂
   // 子节点1 [first]
   // 子节点2 [second]
   item2, second := t.root.split(t.maxItems() / 2)
   // 保存旧根节点到oldroot，此时的oldroot即为上面的first节点的根节点
   oldroot := t.root
   // 新创建一个节点作为根节点
   t.root = t.cow.newNode()
   // 将元素item2加入到新的根节点中
   t.root.items = append(t.root.items, item2)
   // 将first节点和second加入的新的根节点的子节点中
   t.root.children = append(t.root.children, oldroot, second)
  }
 }
 // part 3
 // 真正得进行将元素item加入到BTree中，所有元素的插入都是在叶子节点插入。一路从
 // BTree的根节点向下定位到要插入的叶子的节点位置，在定位的过程中，遇到节点的元素个数
 // 达到最大值(2*degree-1)，则要对该节点进行拆分，并将中间的元素加入到父节点中，即产生
 // 上溢操作
 out := t.root.insert(item, t.maxItems())
 // out为nil,表示当前插入的元素之前不在BTree中
 if out == nil {
  // BTree中元素的数量+1
  t.length++
 }
 // out如果不为空，记录的是之前的旧元素item
 return out
}

```

插入操作insert函数是一个递归调用，所以一开始逻辑处理递归结束的情况。这里的结束情况为待插入的元素在当前的节点n中或当前的节点n已是叶子节点。如果待插入的元素已在节点n中，更新item并返回旧item. 如果当前的节点n是叶子节点，则直接将item加入到n中。剩下的就是递归调用的逻辑，从内部节点不断递归到叶子节点。递归的过程中要处理一种情况，就是当前节点的孩子节点i，也就是item将要插入的子树，存在着节点元素大于等于2\*degree-1的情况，这种情况下插入item后会导致节点的元素个数超标，所以提前判断当节点的元素个数>=2\*degreee-1时，对该节点进行分裂，将分裂节点的中间元素移动到父节点中。

也许有种担心，将中间元素添加节点n中，有可能导致节点n中的元素超过2\*degree-1个，即导致节点n不满足BTree性质。这种情况是不会出现的，因为 整个插入过程是从根节点向下查找并判断节点中元素是否超限的，所以当在处理到节点n的时候，它的元素个数最大为2\*degree-2个，现在向它里面添加一个元素，它能达到的最大值为2\*degree-1。

![](https://ask.qcloudimg.com/http-save/yehe-9955853/2ccd4ff7b9348374e2e155a1fbd8a056.jpeg?imageView2/2/w/2560/h/7000)

```go
func (n *node) insert(item Item, maxItems int) Item {
 // 检查节点n中是否已存在待插入的元素item
 i, found := n.items.find(item)
 // 如果元素已存在，更新item，并返回旧item
 if found {
  out := n.items[i]
  n.items[i] = item
  return out
 }
 // 如果当前的节点n是叶子节点，直接添加item到n中，然后返回
 if len(n.children) == 0 {
  n.items.insertAt(i, item)
  return nil
 }
 // 检查当前节点n的第i个子节点是否需要进行分裂，即检查节点n的child(i)中的元素是否
 // 达到2*degree-1个，如果达到，则对节点child(i)进行分裂，并将child(i)中的中间元素
 // 添加到的它的父节点(即节点n)中。也许有种担心，将中间元素添加节点n中，有可能导致节点n
 // 中的元素超过2*degree-1个，即导致节点n不满足BTree性质。这种情况是不会出现的，因为
 // 整个插入过程是从根节点向下查找并判断节点中元素是否超限的，所以当在处理到节点n的时候，它的
 // 元素个数最大为2*degree-2个，现在向它里面添加一个元素，它能达到的最大值为2*degree-1
 if n.maybeSplitChild(i, maxItems) {
  inTree := n.items[i]
  switch {
  case item.Less(inTree):
   // no change, we want first split node
   // item大小在inTree的左子树中
  case inTree.Less(item):
   i++ // we want second split node
   // item大小在inTree的右子树中
   // 因为child(i)进行了分裂，所以这里i值要加1，表示在分裂的右节点中进行查询
  default:
   // 待插入的元素item在child(i)节点中存在，更新它，并返回旧
   // 元素的值
   out := n.items[i]
   n.items[i] = item
   return out
  }
 }
 // 递归在child(i)节点所在的子树中插入元素item
 return n.mutableChild(i).insert(item, maxItems)
}
```



maybeSplitChild探测是否要对节点n的孩子节点i进行分裂

```go
// 探测是否要对节点n的孩子节点i进行分裂
func (n *node) maybeSplitChild(i, maxItems int) bool {
 // 检查孩子节点i中的元素数量是否小于2*degree-1,如果小于，不用进行分裂
 if len(n.children[i].items) < maxItems {
  return false
 }
 // 获取节点n的真正孩子节点i
 first := n.mutableChild(i)
 // 对节点first从中间进行分裂成两个节点，节点first和节点second
 item, second := first.split(maxItems / 2)
 // 将孩子节点i中的中间元素添加到它的父节点（节点n）中
 n.items.insertAt(i, item)
 // 将节点second添加到节点n,作为节点n的第i+1个孩子节点
 n.children.insertAt(i+1, second)
 return true
}
```

### **删除**

删除操作对外的接口是Delete方法，传入一个待删除的元素item, 返回值也是一个Item类型，如果待删除的元素在BTree中存在，则返回它，否则返回值为nil.

删除的内部实现是deleteItem方法，它包含了删除BTree中最小元素、最大元素和任意元素的逻辑。整个deleteItem主要处理3部分逻辑。

1.  第一部分是特殊情况检查，BTree为空或没有元素，这种情况直接返回。
2.  第二部分也是最核心的部分，执行真正的删除操作，调用的是node的remove方法，remove在下面详细分析，删除操作之后要检查根节点中的元素是否存在，因为在删除的过程中，可能有节点合并，导致树降低一层。
3.  第三部分更新树中元素的个数。

```go
// 从BTree中删除给定的元素
func (t *BTree) Delete(item Item) Item {
 return t.deleteItem(item, removeItem)
}

func (t *BTree) deleteItem(item Item, typ toRemove) Item {
 // 如果树为空或者根节点中没有元素，直接返回，因为这种情况待删除的元素肯定不存在
 if t.root == nil || len(t.root.items) == 0 {
  return nil
 }
 t.root = t.root.mutableFor(t.cow)
 // 从根节点开始查找待删除的元素并删除它，这个过程是一个递归查找删除的过程
 out := t.root.remove(item, t.minItems(), typ)
 // 因为删除的过程可能进行节点的合并，会导致树的高度减少一层，这里处理的就是这种情况
 // 根节点已经没有元素了，但还有孩子节点，此时t.root.children[0]会成为根节点，
 // 并且t.root.children中也只要一个节点
 if len(t.root.items) == 0 && len(t.root.children) > 0 {
  oldroot := t.root
  t.root = t.root.children[0]
  // 释放掉旧的根节点
  t.cow.freeNode(oldroot)
 }
 if out != nil {
  // out不为空，说明待删除的元素在BTree中存在并删除了，元素的数量减1
  t.length--
 }
 return out
}

```



remove处理删除最小元素、最大元素和任意元素的三种情况的逻辑，处理过程比较复杂。先来说删除最小元素和最大元素这两种简单的情况。

最小的元素在最左边的叶子节点中，并且是节点中第一个元素，所以一路向左下发，定位到最左边的叶子节点，将其第一个元素删除，对应的是下面的case removeMin逻辑。

类似的，最大的元素在最右边叶子节点中，并且是它的最后一个元素。定位到最右边的叶子节点之后将其删除即可，对应到下面的case removeMax逻辑。

看了下面的删除最小和最大元素操作，也许有同学有疑问？怎么没有考虑删除叶子节点中元素被删除之后可能导致节点数量不满足节点最小数量限制，甚至出现叶子节点中元素为空的情况。不用担心，这种情况是不存在的，remove在访问节点的过程，会判断当前节点n的孩子child(i)节点中元素是否不够的情况，如果不够，则会进行处理。具体处理在下面的growChildAndRemove中讲。所以这里将叶子节点删除一个元素之后，它里面还是有元素的，删除之后最小的数量为degree-1，还是满足BTree定义的。

remove操作是一个递归调用，递归的出口只有两种情况，一是在递归访问节点的过程中查到了待删除的元素，将其从节点中删除。二是待删除的元素在BTree中不存在，这种情况会一直查找到叶子节点，返回nil.

从节点中删除元素，可能会打破BTree性质，要进行调整处理。根据节点n的孩子节点i中的元素个数情况，存在着两种情况：

1.  情况1 孩子节点i的元素个数<=minItems=degree-1,此时需要对孩子节点i中的元素进行补充，补充方式是从它的兄弟节点借，或者进行节点合并，这部分处理在growChildAndRemove函数，后面会详细分析该逻辑。
2.  情况2 孩子节点i中的元素个数还是够的，这时的处理逻辑根据元素item是否在节点n中分两种情况：2.1 item在节点n中找到了，即found为true,将孩子节点中右子树中最右边的叶子节点中的最后一个元素覆盖掉节点n中的i号元素，并将叶子节点的元素删除 2.2 在孩子节点child中继续递归查找&删除

```go
func (n *node) remove(item Item, minItems int, typ toRemove) Item {
 var i int
 var found bool
 switch typ {
 case removeMax:
  // n为叶子节点，直接删除节点n的items里面的最后一个元素
  // 不用担心删除n的item会导致该节点中元素为空或者不满足BTree最少元素个数
  // ,不存在这种情况，删完之后节点n的最少元素个数为degree-1
  if len(n.children) == 0 {
   return n.items.pop()
  }
  // i为要进行处理的孩子节点下标，最右边的子节点
  i = len(n.items)
 case removeMin:
  // n为叶子节点，直接删除节点n的items里面的第一个元素
  if len(n.children) == 0 {
   return n.items.removeAt(0)
  }
  // 继续向下，最左边的子节点
  i = 0
 case removeItem:
  // 查找当前节点n的元素中是否存在item
  i, found = n.items.find(item)
  // 递归出口之一：已经查询到了叶子节点
  if len(n.children) == 0 {
   // 查询到了叶子节点，并且元素item在叶子节点中，直接删除
   if found {
    return n.items.removeAt(i)
   }
   // 查询到了叶子节点，但是没有查到，递归也结束，返回nil
   return nil
  }
 default:
  panic("invalid type")
 }
 
 // 根据节点n的孩子节点i中的元素个数情况，有两种处理逻辑：
 // 逻辑1，孩子节点i的元素个数<=minItems=degree-1,此时需要对孩子节点i中的元素进行
 // 补充，补充方式是从它的兄弟节点借，或者进行节点合并
 if len(n.children[i].items) <= minItems {
  return n.growChildAndRemove(i, item, minItems, typ)
 }
 
 // 逻辑2：孩子节点i中的元素个数还是够的，这时的处理逻辑根据元素item是否在节点n中分两种情况
 // 情况1，item在节点n中找到了，即found为true,将孩子节点中右子树中最右边的叶子节点中的最后一个
 // 元素覆盖掉节点n中的i号元素，并将叶子节点的元素删除
 child := n.mutableChild(i)
 if found {
  out := n.items[i]
  // 即找到节点n的最大前驱节点中的最大元素赋值给n.items[i]，并将叶子节点中的元素删除
  n.items[i] = child.remove(nil, minItems, removeMax)
  return out
 }
 // 情况2 在孩子节点child中继续递归查找&删除
 return child.remove(item, minItems, typ)
}

```



growChildAndRemove方法处理节点n的child(i)中元素不够的情况，处理的思路有两个，从它兄弟节点借或者对节点进行合并。下面实现中有3个if分支，前两个分支是从兄弟节点借，分为从左兄弟借或右兄弟借两种情况，最后一个是对节点进行合并，因为左右兄弟节点都没有足够的元素。

1.  从左兄弟节点借：孩子节点i有左兄弟，并且孩子节点i的左边的一个兄弟节点元素是够的，从它里面将最大的元素借走。将孩子节点i-1中的最大的元素赋值给节点n中的元素items[i-1]，将节点n中的元素items[i-1]插入到孩子节点i的最左边
2.  从右兄弟节点借：孩子节点i有右兄弟，并且第一个右兄弟中的元素是够的，从它里面借一个元素。将节点n中的元素i添加到孩子节点i中，将从右兄弟借走的元素赋值给节点n中元素i的位置
3.  对节点进行合并：节点n的child(i)节点的左右兄弟节点都没有足够的元素可以借，合并的是两个节点是child(i)和child(i+1)。处理特殊情况，当前的child已经是节点n的最右的孩子节点，所以child是没有右兄弟了。详细的合并逻辑见下面的源码注解，已经说的很详细了

从右兄弟节点借

对节点进行合并

```go
func (n *node) growChildAndRemove(i int, item Item, minItems int, typ toRemove) Item {
 // 孩子节点i有左兄弟，并且孩子节点i的左边的一个兄弟节点元素是够的，从它里面将最大的元素借走
 if i > 0 && len(n.children[i-1].items) > minItems {
  // 真正的孩子节点i
  child := n.mutableChild(i)
  // 真正的孩子节点i-1
  stealFrom := n.mutableChild(i - 1)
  // 孩子节点i-1中的最大元素（也是items切片中最后一个元素）被移除
  stolenItem := stealFrom.items.pop()
  // 将节点n中的元素items[i-1]插入到孩子节点i的最左边
  child.items.insertAt(0, n.items[i-1])
  // 将孩子节点i-1中的最大的元素赋值给节点n中的元素items[i-1]
  n.items[i-1] = stolenItem
  // 处理stealFrom的孩子节点，因为stealFrom节点中的最后一个元素pop出去了
  // 此时它的孩子节点数量比它的元素多2个，不满足BTree性质，所以需要将它的最后一个
  // 孩子节点pop出去，然后放入到child节点中，并作为它最左边的孩子节点。因为child节点
  // 刚好插入了一个元素，它缺少1个孩子节点。这样处理之后，完美的构成了BTree
  if len(stealFrom.children) > 0 {
   child.children.insertAt(0, stealFrom.children.pop())
  }
 } else if i < len(n.items) && len(n.children[i+1].items) > minItems {
  // 孩子节点i有右兄弟，并且第一个右兄弟中的元素是够的，从它里面借一个元素
  // 真正的孩子节点i
  child := n.mutableChild(i)
  // 真正的孩子节点i+1
  stealFrom := n.mutableChild(i + 1)
  // 孩子节点i+1中的最小元素（也是items切片中第一个元素）被移除
  stolenItem := stealFrom.items.removeAt(0)
  // 将节点n中的元素i添加到孩子节点i中
  child.items = append(child.items, n.items[i])
  // 将从右兄弟借走的元素赋值给节点n中元素i的位置
  n.items[i] = stolenItem
  // 处理stealFrom的孩子节点，因为stealFrom节点中的第一个元素pop出去了
  // 此时它的孩子节点数量比它的元素多2个，不满足BTree性质，所以需要将它的第一个
  // 孩子节点pop出去，然后放入到child节点中，并作为它最右边的孩子节点。因为child节点
  // 刚好插入了一个元素，它缺少1个孩子节点。
  if len(stealFrom.children) > 0 {
   child.children = append(child.children, stealFrom.children.removeAt(0))
  }
 } else {
  // 走到这里，说明前面两种情况都不满足，即child(i)节点的左右兄弟节点都没有足够的元素可以借
  // 这个时候的处理办法是对孩子节点进行合并，不用担心，合并之后节点的个数不会超过2*degree-1

  // 合并的是两个节点是child(i)和child(i+1)，也就是将child和它的右兄弟进行合并

  // 处理特殊情况，当前的child已经是节点n的最右的孩子节点，所以child是没有右兄弟了
  // 这时候将i-1, 实际变成了将child和它的左兄弟合并。这里将i-1使得后面的处理变得统一
  if i >= len(n.items) {
   i--
  }
  // 真正的孩子节点
  child := n.mutableChild(i)
  // 将节点n中位置i的元素移走
  mergeItem := n.items.removeAt(i)
  // 因为节点n中位置i中的元素移走了，所以节点n的孩子节点也要减少1个，
  // 将节点n中位置i+1中的孩子节点移走
  mergeChild := n.children.removeAt(i + 1)
  // 将节点n中移走的元素添加到孩子节点child中
  child.items = append(child.items, mergeItem)
  // 将mergeChild中的元素添加到child中，因为是mergeChild合并到child
  child.items = append(child.items, mergeChild.items...)
  // 将mergeChild中的子节点添加到child.children中
  child.children = append(child.children, mergeChild.children...)
  // mergeChild的元素和子节点信息已合并到了child中，此时mergeChild节点可以释放了
  n.cow.freeNode(mergeChild)
 }
 // 上面处理的是对child节点的合并，还没有做删除item的操作，下面才进行删除操作
 // 注意这里一定是从节点n开始remove操作，因为上面的合并操作会导致节点n中的元素发生变化
 // 从它的子节点进行remove会漏掉元素
 return n.remove(item, minItems, typ)
}
```



### **查询**

查询的操作Get方法处理逻辑很简单，查询的过程是一个递归的过程，直到走到叶子节点，还没有查到，返回nil, 如果在内部节点中查询到了元素，则直接返回。

```go
func (t *BTree) Get(key Item) Item {
 if t.root == nil {
  return nil
 }
 return t.root.get(key)
}

// 查找给定的元素是否在node中存在，会递归查询node的孩子节点
func (n *node) get(key Item) Item {
 i, found := n.items.find(key)
 if found {
       // 找到了 直接返回
  return n.items[i]
 } else if len(n.children) > 0 {
        // 没有找到 递归查询子节点
  return n.children[i].get(key)
 }
    // 走到这里，已经走到了叶子节点，并且没有查到，返回nil
 return nil
}
```



## **BTree优化技巧**

btree是一个生产包，它对GC和并发读写操作方面做了很多优化，我们可以从中学习它的优化思路。总结起来，btree的优化技巧有两点：

1.  针对GC问题,通过缓存，能够对节点进行复用，降低GC开销，这里的缓存就是代码中的freelist
2.  对于并发读写，通过写时，降低内存的开销，多颗BTree可以共享节点node,在写时才进行，保证互相不干扰

下面结合代码来分析优化是怎么做的，前面分析过BTree结构定义，这里在拿出来分析, 我们现在主要关注它的cow字段，这个字段的类型是一个copyOnWriteContext类型的指针。copyOnWriteContext结构中只有一个字段freelist, 它的类型是FreeList指针。FreeList中的freeelist是一个node指针的切片，mu是用来在并发情况下保护freelist的。

```go
type BTree struct {
 // BTree的度
 degree int
 // BTree中元素的数量
 length int
 root   *node
 // 默认的空闲node数量为32个
 cow    *copyOnWriteContext
}

type copyOnWriteContext struct {
 freelist *FreeList
}

type FreeList struct {
 mu       sync.Mutex
 // freelist 默认缓存空闲node的切片大小为32
 freelist []*node
}
```



上述的copyOnWriteContext结构图形结构是长下面这样。

btree包提供了两种BTree对象的两种构造方法，其中一种方式需要传入给FreeList类型的对象，这样可以在多颗BTree上共享FreeList对象。

通过New函数产生的BTree的cow中的freelist是不共享的，内部会new一个FreeList对象，通过NewWithFreeList产生的BTree,需要外部传入一个FreeList, 这样传入相同的FreeList得到的BTree是共享的。

```go
func New(degree int) *BTree {
 return NewWithFreeList(degree, NewFreeList(DefaultFreeListSize))
}
```



```go
func NewWithFreeList(degree int, f *FreeList) *BTree {
 if degree <= 1 {
  panic("bad degree")
 }
 return &BTree{
  degree: degree,
  cow:    &copyOnWriteContext{freelist: f},
 }
}
```



还有一点需要说明，无论是通过New还是NewWithFreeList产生的BTree,BTree中的cow值都是不同的，也就是每个BTree中根节点的cow都是不同的。这样可以通过根节点的cow来区分是不是同一颗BTree.

除了BTree结构体，node结构定义中也有一个cow字段。对应BTree中的node节点来说，它的cow值与BTree中的cow是一致的，都指向同一个copyOnWriteContext对象，表示它们都属于同一颗BTree。

FreeList结构中的freelist用于缓存释放的node节点，下次申请node的时候，优先从freelist中获取。

申请node时优先从freelist中获取

```go
func (c *copyOnWriteContext) newNode() (n *node) {
 n = c.freelist.newNode()
 n.cow = c
 return
}

func (f *FreeList) newNode() (n *node) {
 ...
 // index<0 表示空闲缓存中没有node了，这个时候new一个
 if index < 0 {
  f.mu.Unlock()
  return new(node)
 }
 ...
    // 从freelist中获取
 return
}
```



删除的时候将node返回freelist

```go
func (c *copyOnWriteContext) freeNode(n *node) freeType {
 if n.cow == c {
  ...
        // 将node n放回freelist
  if c.freelist.freeNode(n) {
   return ftStored
  } else {
   return ftFreelistFull
  }
 } else {
  return ftNotOwned
 }
}

func (f *FreeList) freeNode(n *node) (out bool) {
 f.mu.Lock()
 if len(f.freelist) < cap(f.freelist) {
  f.freelist = append(f.freelist, n)
  out = true
 }
 f.mu.Unlock()
 return
}
```



通过cow实现写时，多个BTree可以共享节点，有修改或删除操作才对节点进行拷贝，在副本上进行修改或删除。

通过Clone操作，可以获取两颗一模一样的BTree,这两颗BTree共享节点信息，实际只保存了一份节点，节省空间。注意下面的源码，更新了旧BTree t的cow，这里不更新可以吗？不可以，因为后续对t的修改或删除操作，会影响t2,所以这里给它赋值一个新的cow，用于区分。结合下面这张图理解起来很容易理解。

克隆前

克隆后

```go
func (t *BTree) Clone() (t2 *BTree) {
 // 产生两个cow对象，cow1和cow2的对象地址是不同的
 cow1, cow2 := *t.cow, *t.cow
 // 拷贝对象t，也就是第二颗BTree对象
 out := *t
 // 更新旧BTree的cow1
 t.cow = &cow1
 // 更新新BTree的cow2
 out.cow = &cow2
 return &out
}
```



# Reference
https://cloud.tencent.com/developer/article/2072926?areaSource=&traceId=
https://segmentfault.com/a/1190000020416577

https://cloud.tencent.com/developer/article/2072927?areaSource=&traceId=

https://www.zhihu.com/question/64704576
https://github.com/google/btree