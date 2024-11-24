下单过程中的库存扣减的精确执行，这种数据一致性的实现效果会直接影响订单是否能够成功履约，而传统关系型数据库的并发更新存在显著瓶颈，因此需要专项优化。

**扣减库存问题**：性能瓶颈 – MySQL热点行扣减库存（行级锁）。

**技术策略**：扣减库存异步化，异步扣库存主要分3步（见下图）：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/kEeDgfCVf1fncomiaI2IibY12HtbuIMgbpWhk0gwFfhomOTNuP2hyO9arpAJyXfWceSicIAQsXc2ia9UcJ96Yr76Wg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1）初始化：秒杀商品设置好活动场次，将秒杀库存同步至Redis。

2）扣库存：活动开始时，先从Redis扣减库存，在通过消息通知异步扣减DB库存，削除DB更新同一行的峰值。

3）还库存：如果有退订，先判断DB中是否有扣减记录，如果有，则先退DB再退Redis；如果没有，重试多次。

扣还库存过程中也会存在超时等未知情况，此处细节过多不再展开。按照业务“可少买不超卖”的原则，即使在这个过程中数据可能存在短暂的延时，但能够确保最终一致性。

**优化效果**：库存扣减异步化，消除行级锁瓶颈。现在系统能够轻松支撑数十万单/分钟交易流量。

#### 说出以下输出结果？为什么？

`func TestSlicePrint(t *testing.T) {    a := []byte("AAAA/BBBBB")    index := bytes.IndexByte(a, '/')    b := a[:index]    c := a[index+1:]    b = append(b, "CCC"...)    fmt.Println(string(a))    fmt.Println(string(b))    fmt.Println(string(c))   }   `

结果：

`AAAACCC   CCBBB   AAAACCCBBB   `

**切片是对底层数组的引用，因此对切片的修改会影响原始的切片。**![图片](https://mmbiz.qpic.cn/mmbiz_png/UBiaA4ibmWk2bp3o99tqboeyTI07iaMZ9BhwNnP8Ktg1ajxaSOQohOm88Ya7E0pCrxGrJ0cWy3oNzAncveDu774bw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当一个切片被用做一个`append函数调用中`的基础切片时：

- 如果添加的元素数量大于`此（基础）切片的冗余元素槽位的数量`，则一个`新的底层内存片段`将被开辟出来并用来存放结果切片的元素。这时，基础切片和结果切片`不共享任何底层元素`。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/UBiaA4ibmWk2bp3o99tqboeyTI07iaMZ9BhlGey4pqGSibCuzxUO8U4hBlzpjGJ9SREicEXRZoZBKpFjJnYTedVLb5Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

插入的C大于原来的cap

- 否则，不会有底层内存片段被开辟出来。这时，基础切片中的所有元素也同时属于结果切片。两个切片的元素都`存放于同一个内存片段上`。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/UBiaA4ibmWk2bp3o99tqboeyTI07iaMZ9Bhr2DtbQYpcIn3vAkDPiaweGT6WzEwqfE3YxKUjbmahF6e3iaia5Ft6sOXQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

插入的C小于原来的cap

#### 这个锁是什么用的？这段代码有什么问题？

`func TestNumPrint(t *testing.T) {    wg := sync.WaitGroup{}    lock := new(sync.Mutex)    var a int32 = 0    var b int32 = 2    for i := 0; i < 5; i++ {     go func() {      if a > b {       fmt.Println("done")       return      }      lock.Lock()      defer lock.Unlock()      a++      fmt.Printf("i: %d a: %d \n", i, a)     }()    }    wg.Wait()   }   `

我个人觉得有三点：

1. wg没有add(1)导致主进程结束了，子进程还没开始。但这个被面试官否掉啦，“你就当主进程一直阻塞就好了”
    
2. 这个i的变量有问题，都将会是最后一个数也就是5的这个值。也被否掉了，“go最新版里面这个不是问题，你就当是最新版吧”
    
3. 还有就是这个锁有问题，虽然锁住了`a`的自增，但是没锁住a的读取，`int 类型`本身并不是并发安全的，我们必须加锁才能保证原子性，那么如果我们不加锁，还可以使用`atomic.AddInt32()` 进行原子操作。**但是这里锁的粒度出现了问题，锁应该是对变量的`读取和修改`都进行锁，上面只锁住了修改，锁应该要锁出一段逻辑操作，而不是一个变量，所以只需要将锁提到`if a>b`的前面**。

### RDB是怎么进行操作？

redis 一般会有两个命令生产RDB，一个是`save`，另一个是`bgsave`，区别在于`是否阻塞主线程`

- save 是和操作命令在同一个线程里面，如果有大key写入RDB文件，会造成阻塞。
    
- bgsave是创建一个子进程来写入RDB文件，不会造成阻塞。
    

**RDB和everysec模式的AOF区别就是一个是全量，一个是增量**

那么在bgsave中，通过`fork`函数创建的子进程会和父进程共享同一片内存数据，因为在`fork`的时候，**子进程会复制父进程的页表数据，而这两个进程的`页表数据一摸一样`也就表示`会指向同一片物理内存地址。`** 这样是为了减少创建子进程时的性能损耗，加快子进程的创建速度，`毕竟创建子进程的过程中，是会阻塞主线程的。`![图片](https://mmbiz.qpic.cn/mmbiz_png/UBiaA4ibmWk2bp3o99tqboeyTI07iaMZ9BhGROX7JtIzsHvq1Z5wCD3uu9CHcaPib772oqAsBWwO7I5eUvAJ2icLEAw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

golang http包服务端逻辑，是否连接复用了？
gin的context包含了什么，有什么逻辑

http是有状态的吗，长连接有什么好处，session是放到哪里的。

restful的优势？
session放到哪里了？

golang的引用类型有哪些
****golang**中分为值**类型**和**引用类型****

- 值**类型**分别**有**：int系列、float系列、bool、string、数组和结构体
- **引用类型有**：指针、slice切片、管道channel、接口interface、map、函数等


如何设计一个rpc？
输入url后的过程
http连接跟tcp的关系

![[Pasted image 20240309152257.png]]
redo log 三种状态：

- 存在 redo log buffer 中，物理上是在 MySQL 进程内存中  
    
- 写到磁盘（write），但是没有持久化（fsync），物理上是在文件系统的 page cache 里  
    
- 持久化磁盘，对应的是 hard disk  
    

日志写到 redo log buffer 是很快的，write 到 page cache 也差不多，但是持久化到磁盘的速度就慢多了。

InnoDB 提供了 innodb_flush_log_at_trx_commit 参数，取值如下：

1. 设置为 0 时，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中；  
    
2. 设置为 1 时，表示每次事务提交时都将 redo log 直接持久化到磁盘；  
    
3. 设置为 2 时，表示每次事务提交时都只是把 redo log 写到 page cache。  
    

InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。

## 未提交事务是否会写redolog

涉及事务提交流程和redolog的写入机制：

有binlog情况下，**commit动作开始时，会有一个Redo XID 的动作记录写到redo**，然后写data到binlog，binlog写成功后，会将binlog的filename，日志写的位置position再写到redo(position也会写到pos文件里)，此时才表示该事务完成（committed）。如果只有XID，没有后面的filename和position，则表示事务为prepare状态。

  

事务提交流程：

     **commit; --> write XID to redo. --> write data to Binlog. --> write filename,postsion of binlog to redo. --> commited.**

　　记录Binlog是在InnoDB引擎Prepare（即Redo Log写入磁盘）之后，这点至关重要。

![image](https://ucc.alicdn.com/pic/developer-ecology/tauolzunvmg6a_7212eca42ecd4d50a7f45382d331fc16.png "image")

  

crash发生在不同阶段时的事务状态和事务结果：

![image](https://ucc.alicdn.com/pic/developer-ecology/tauolzunvmg6a_281453feffd745bcab55c0f6e243ac27.png "image")

总结起来说就是如果一个事务在prepare阶段中落盘成功，并在MySQL Server层中的binlog也写入成功，那这个事务必定commit成功。但二者缺一不可。

不过上面描述还是没有明确，事务还没提交的时候，redolog 能不能被持久化到磁盘？

主要有三种可能的原因：

（1）InnoDB 有一个后台线程，每隔 1 秒轮询一次，具体的操作是这样的：调用 write 将 redolog buffer 中的日志写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。而在事务执行中间过程的redolog 都是直接写在 redolog buffer 中的，也就是说，**一个没有提交的事务的 redolog，也是有可能会被后台线程一起持久化到磁盘的**；

（2）innodb_flush_log_at_trx_commit 设置是 1，也就是每次事务提交时，都执行 fsync 将 redolog 直接持久化到磁盘（设置为0表示每次事务提交的时候，都只是把 redolog 留在 redolog buffer 中；设置为2表示每次事务提交的时候，都只执行 write 将 redolog 写到文件系统的 page cache ）。例如，假设事务 A 执行到一半，已经写了一些 redolog 到 redolog buffer 中，这时候有另外一个事务 B 提交，按照 innodb_flush_log_at_trx_commit = 1 的逻辑，事务 B 要把 redolog buffer 里的日志全部持久化到磁盘，这时候，就会带上事务 A 在 redolog buffer 里的日志一起持久化到磁盘；

（3）redolog buffer 占用的空间达到 innodb_log_buffer_size的一半大小时（默认是 8MB），后台线程会主动写盘。不过由于这个事务并没有提交，所以这个写盘动作只是 write 到了文件系统的 page cache，**仍然是在内存中，并没有调用 fsync 真正落盘。**

## 1-2 事务提交且当redolog文件满时，是否可以立即覆盖写入

   如果确定需要写redo log文件，这时要看checkpoint。redo log文件是循环写入的，覆盖写之前，总要保证对应的（即将被覆盖的）脏页已经刷到了磁盘。所以如果要覆盖的脏页已经被刷到磁盘，那么久直接覆盖；如果还没刷到磁盘，就需要阻塞等待脏页完成刷到磁盘后再执行覆盖。


![[Pasted image 20240109154207.png]]
![[Pasted image 20240109154415.png]]
![[Pasted image 20240109154544.png]]
![[Pasted image 20240109154604.png]]

![[Pasted image 20240109154907.png]]
![[Pasted image 20240109155018.png]]





### [2、请说出下面代码存在什么问题。](https://github.com/lifei6671/interview-go/blob/master/question/q007.md#2%E8%AF%B7%E8%AF%B4%E5%87%BA%E4%B8%8B%E9%9D%A2%E4%BB%A3%E7%A0%81%E5%AD%98%E5%9C%A8%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98)

```go
type student struct {
	Name string
}

func zhoujielun(v interface{}) {
	switch msg := v.(type) {
	case *student, student:
		msg.Name
	}
}
```

**解析：**
golang中有规定，`switch type`的`case T1`，类型列表只有一个，那么`v := m.(type)`中的`v`的类型就是T1类型。
如果是`case T1, T2`，类型列表中有多个，那`v`的类型还是多对应接口的类型，也就是`m`的类型。
所以这里`msg`的类型还是`interface{}`，所以他没有`Name`这个字段，编译阶段就会报错

### [以下代码有什么问题，说明原因](https://github.com/lifei6671/interview-go/blob/master/question/q008.md#2-%E4%BB%A5%E4%B8%8B%E4%BB%A3%E7%A0%81%E6%9C%89%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98%E8%AF%B4%E6%98%8E%E5%8E%9F%E5%9B%A0)

```go
type student struct {
	Name string
	Age  int
}

func pase_student() {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}
}
```

**解析：**

golang 的 `for ... range` 语法中，`stu` 变量会被复用，每次循环会将集合中的值复制给这个变量，因此，会导致最后`m`中的`map`中储存的都是`stus`最后一个`student`的值。

### [3、下面的代码会输出什么，并说明原因](https://github.com/lifei6671/interview-go/blob/master/question/q008.md#3%E4%B8%8B%E9%9D%A2%E7%9A%84%E4%BB%A3%E7%A0%81%E4%BC%9A%E8%BE%93%E5%87%BA%E4%BB%80%E4%B9%88%E5%B9%B6%E8%AF%B4%E6%98%8E%E5%8E%9F%E5%9B%A0)

```go
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("i: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("i: ", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

**解析：**

这个输出结果决定来自于调度器优先调度哪个G。从runtime的源码可以看到，当创建一个G时，会优先放入到下一个调度的`runnext`字段上作为下一次优先调度的G。因此，最先输出的是最后创建的G，也就是9.

### [下面代码会输出什么？](https://github.com/lifei6671/interview-go/blob/master/question/q008.md#4%E4%B8%8B%E9%9D%A2%E4%BB%A3%E7%A0%81%E4%BC%9A%E8%BE%93%E5%87%BA%E4%BB%80%E4%B9%88)

```go
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}

```

**解析：**

输出结果为`showA`、`showB`。golang 语言中没有继承概念，只有组合，也没有虚方法，更没有重载。因此，`*Teacher` 的 `ShowB` 不会覆写被组合的 `People` 的方法。

### [下面的代码有什么问题?](https://github.com/lifei6671/interview-go/blob/master/question/q008.md#8%E4%B8%8B%E9%9D%A2%E7%9A%84%E4%BB%A3%E7%A0%81%E6%9C%89%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98)

```go
type UserAges struct {
	ages map[string]int
	sync.Mutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}
```

**解析：**

在执行 Get方法时可能被thorw。

虽然有使用sync.Mutex做写锁，但是map是并发读写不安全的。map属于引用类型，并发读写时多个协程见是通过指针访问同一个地址，即访问共享变量，此时同时读写资源存在竞争关系。会报错误信息:“fatal error: concurrent map read and map write”。

因此，在 `Get` 中也需要加锁，因为这里只是读，建议使用读写锁 `sync.RWMutex`。

### [以下代码打印出来什么内容，说出为什么。。。](https://github.com/lifei6671/interview-go/blob/master/question/q008.md#11%E4%BB%A5%E4%B8%8B%E4%BB%A3%E7%A0%81%E6%89%93%E5%8D%B0%E5%87%BA%E6%9D%A5%E4%BB%80%E4%B9%88%E5%86%85%E5%AE%B9%E8%AF%B4%E5%87%BA%E4%B8%BA%E4%BB%80%E4%B9%88)

```go
package main

import (
	"fmt"
)

type People interface {
	Show()
}

type Student struct{}

func (stu *Student) Show() {

}

func live() People {
	var stu *Student
	return stu
}

func main() {
	if live() == nil {
		fmt.Println("AAAAAAA")
	} else {
		fmt.Println("BBBBBBB")
	}
}
```

**解析：**

跟上一题一样，不同的是`*Student` 的定义后本身没有初始化值，所以 `*Student` 是 `nil`的，但是`*Student` 实现了 `People` 接口，接口不为 `nil`。



select,waitgroup,singleFlight,chan，slice，map，内存，携程等数据结构都要复习以下

第一题:给个数组，每个数组上数字表示跳跃能力，从一跳到n最少跳次数 倒序贪心
第二题:给个string，求最大不重复元素字串如abc，string长度上面写250，我三个for循环段错误了。

1. 100 leetcode54 螺旋矩阵
2. 0 leetco52 n皇后 
自我介绍
介绍sql慢查询的情况？
如何定位到sql慢查询的？
sql慢查询日志
explain的语句分析
哪些操作可能会导致mysql的慢查询问题？
一道sql题，口述思路（找到所有学生中不及格课程数目最多的 学生名称 和 不及格课程数）
Java中是如何判断出垃圾的？
# 开发过程中，你觉得怎样的代码是写得比较好的代码？
1、有代码流程图和分支条件  =》符合简单理解的要求
2、关键路径有测试案例保证  =》健壮性，便于修改
3、代码分层规范，函数长度小  =》控制复杂性的要求
4、符合常见的代码规范  =》 便于团队的共识理解
5、经常review重构，才是保证代码好味道的标准。



开发过程中，使用过哪些项目管理的工具？
反问？

你在开发中碰到什么技术难点；

有个兔子的问题，一对兔子第三个月后就开始繁殖，不考虑死亡，求兔子数。
答：其实就是一个斐波那契数列,我是用循环解决的，当然你也可以用递归：
```go
func rabbitCount(m int) int { 
	sum:= 1 
	for m >=3 { 
		m-=3 
		sum+=m 
	}
 }；
```

百万级Mysql怎么处理。
答：1-主备 2-读写分离 3-集群 4-负载均衡 5-查询缓存 6-raid

怎么用go实现一个栈。
答：其实就是一个切片的问题，注意pop的时候一定要注意切片长度检查，否则会panic。

有个考察defer和panic执行顺序的问题。
答：defer是先注册后调用(最后注册的最先被执行)。所有defer都执行完后，才会显示panic里面的东西。

切片和数组差别、go多协程之间怎么同步；

通道；

sql的锁；

left join和inner join的区别；

怎么看goroutine之类的。

gorintine是怎么调度；

gc算法；

io模型；

考察切片比较，深拷贝的问题；

合并两个有序数组；

考察for ... range 陷阱。

对golang的测试问的很久，性能调优方面的；

redis库的场景问题还有些底层的实现问题；

问等待所有goroutine结束，怎么做？
答使用sync.WaitGroup。

map的底层实现。

redis集群？一致性hash？

go的测试？
go的调度
go struct能不能比较
go defer（for defer）
select可以用于什么
context包的用途
client如何实现长连接
主协程如何等其余协程完再操作
slice，len，cap，共享，扩容
map如何顺序读取
实现set
实现消息队列（多生产者，多消费者）
大文件排序
基本排序，哪些是稳定的
http get跟head
http 401,403
http keep-alive
http能不能一次连接多次请求，不等后端返回
tcp与udp区别，udp优点，适用场景
time-wait的作用
数据库如何建索引
孤儿进程，僵尸进程
死锁条件，如何避免
linux命令，查看端口占用，cpu负载，内存占用，如何发送信号给一个进程
git文件版本，使用顺序，merge跟rebase
技术二面 项目相关
通过腾讯会议，腾讯的两个大佬一起面试

项目是实现爬虫的流程
爬虫如何做的鉴权吗
怎么实现的分布式爬虫
# 电商系统图片过多会造成带宽过高，如何解决
cdn,nginx

micro服务发现
mysql底层有哪几种实现方式
channel底层实现
java nio和go 区别
读写锁底层是怎么实现的
go-micro 微服务架构怎么实现水平部署的，代码怎么实现的
micro怎么用
怎么做服务员的
mysql索引为什么要用B+树？
mysql语句性能评测？
服务员发现有哪些机制
raft算法是那种一致性算法
raft有什么特点
# 当go服务部署到线上了，发现有内存泄露，该怎么处理
kubectl 命令进行端口映射，然后导出pprof文件，然后进行pprof内存分析判断哪里泄露
pprof当时用的是临时加上去的，实际上可以通过配置文件进行这个采集的打开和关闭。


raft算法里面如果出现脑裂怎么处理？有没有了解过paxos和zookeeper的zab算法，他们之前有啥区别？

后面还问了一个问题定位的问题，服务器CPU 100%怎么定位？可能是由于平时定位业务问题的思维定势，加之处于蒙蔽状态，随口就是：先查看监控面板看有无突发流量异常，接着查看业务日志是否有异常，针对CPU100%那个时间段，取一个典型业务流程的日志查看。最后才提到使用top命令来监控看是哪个进程占用到100%。果然阵脚大乱，张口就来，捂脸。。。

本来正确的思路应该是先用top定位出问题的进程，再用top定位到出问题的线程，再打印线程堆栈查看运行情况，这个流程换平时肯定能答出来，但是，但是没有但是。还是得好好总结。

最后问了一个系统设计题目（朋友圈的设计），白板上面画出系统的架构图，主要的表结构和讲解主要的业务流程，如果用户变多流量变大，架构将怎么扩展，怎样应对？

这个答的也有点乱，直接上来自顾自的用了一个通用的架构，感觉毫无亮点。后面反思应该先定位业务的特点，这个业务明显是读多写少，然后和面试官沟通一期刚开始的方案的用户量，性能要求，单机目标qps是什么等等？在明确系统的特点和约束之后再来设计，而不是一开始就是用典型互联网的那种通用架构自顾自己搞自己的方案。

# 4 双检查实现单例
```go
package doublecheck
import (
	"sync"
)
type Once struct {
	m    sync.Mutex
	done uint32
}
func (o *Once) Do(f func()) {
	if o.done == 1 {
		return
	}
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		o.done = 1
		f()
	}
}
```
A: 不能编译
B: 可以编译，正确实现了单例
C: 可以编译，有并发问题，f函数可能会被执行多次
D: 可以编译，但是程序运行会panic
4. C
在多核CPU中，因为CPU缓存会导致多个核心中变量值不同步。

没有使用atomic是无法保证done这个字段对所有cpu可见。所以sync.Once使用了锁和atomic

7 channel
```go
package main
import (
	"fmt"
	"runtime"
	"time"
)
func main() {
	var ch chan int
	go func() {
		ch = make(chan int, 1)
		ch <- 1
	}()
	go func(ch chan int) {
		time.Sleep(time.Second)
		<-ch
	}(ch)
	c := time.Tick(1 * time.Second)
	for range c {
		fmt.Printf("#goroutines: %d\n", runtime.NumGoroutine())
	}
}
```
A: 不能编译
B: 一段时间后总是输出 #goroutines: 1
C: 一段时间后总是输出 #goroutines: 2
D: panic
7. C
因为 ch 未初始化，写和读都会阻塞，之后被第一个协程重新赋值，导致写的ch 都阻塞。

# 异步消息中废弃的方案：
1、热点表，需要单独发送（通过heavyKeeper判断）
2、应用重启会丢消息，因为分片只记录最大的offset
3、数据聚合一起处理
4、降级处理

# Java跟go有什么区别

如何评估并发量，带宽， 以及PV与并发，带宽的计算规则
以30w的PV为例，来计算需要买多少带宽合适？
　　对于这个问题，相信若从来没有想过这个问题，你很可能一下子一脸懵逼，怎么测算？
　　其实冷静下来，就会发现，不过就是一道小学数学题罢了。
不就是将30万PV的访问量，换算成带宽嘛,将问题拆分如下:
　　1. 一天30万的页面访问(PV),转换为每秒访问量(QPS): 30万/(24 × 60 × 60)秒 = 3.47个请求每秒
　　2. 知道了一秒大概是3.47个请求，将请求化整，就是4个请求每秒.
　　3. 再来看看我们整个网站平均一个页面有多大，怎么看？Chrome等浏览器打开调试窗口(如:按F12) ,访问自己的网站，看一下最右下角的访问统计，你就知道自己当前访问的这个页面有多少资源，这一次访问发起了多少个衍生连接，以及这一次压缩传输的数据大小，已经资源到客户端后，解压到大小，我们这里主要关心压缩传输大小，通过对整个网站的网页做访问，记录这些值，并求出平均页面大小，接着就可以继续下面的计算了。
4. 知道了一秒大概有4个并发请求，而平均一个页面假如是0.4M，那么4个并发请求需要占用的带宽不就是 4×0.4M=1.6MB，再换算成比特，就是1.6MByte × 8bit= 12.8Mbps 这就是你的网站大概需要多少带宽。
　 5. 引用道友的公式如下:
　　　　网站带宽= PV / 统计时间(换算到S)\*平均页面大小(单位KB)* 8


两个 interface {} 能不能比较
Go 语言 中的 空接口 在保存不同的值后，可以和其他 变量 一样使用 == 进行比较操作。

空接口的比较特性
类型 不同的空接口间的比较结果不相同
不能比较空接口中的动态值(比如interface{}中的数组)


# 1. raft 如何实现从 follower 读取？
Raft对只读操作的处理办法是

只读请求最终也必须依靠Leader来执行，如果是Follower接收请求的，那么必须转发。
记录下当前日志的commitIndex => readIndex
执行读操作前要向集群广播一次心跳，并得到majority的反馈
等待状态机的applyIndex移动过readIndex
通过查询状态机来执行读操作并返回客户端最终结果。
在步骤1到步骤2之间，如果Leader刚刚当选，还必须等待no-op操作同步完成。

上面的步骤看起来很复杂，其中最重要的就是心跳广播，这是为了确认当前集群没有被网络分区。

只读操作的进一步优化

标准的强一致只读操作是完全是在Leader端进行的。这里可以做一步改进让只读操作主要在Follower端进行。

Follower接收到只读指令后，向leader索要当前的readIndex值。
Follower端等待自身的状态机的applyIndex移动过readIndex。
Follower端通过查询自身的状态机来执行读操作并返回客户端最终结果。
leader向follower返回readIndex也不是简单的直接返回，而是需要重复前面的标准步骤1~步骤3，来确认网络没有分区。这还是一次RPC操作，无法省略。但是毫无疑问，它分担了leader的压力，可以让leader有更多的资源来处理自身的读写操作。


3.etcd相关

是什么？如何保持高可用性，选举机制，脑裂如何解决

4.k8s相关

哪些常用组件，发起一个pod的创建整个通路，service有哪些，

一个请求到达pod的过程、configmap、 dockerfile

为什么学go

Map是线程安全的吗？怎么解决并发安全问题？

sync.Map 怎么解决线程安全问题？看过源码吗？

copy是操作符还是内置函数
Go里面一个协程能保证绑定在一个内核线程上面的。
Go的协程可以不可以自己让出cpu

Go的协程可以只挂在一个线程上面吗

一个协程挂起换入另外一个协程是什么过程？

分页和分段内存管理有什么区别

内存泄漏和内存溢出的区别

寻找两个正序数组的中位数

# 智力题：有只猴子在树林里采了100根香蕉，堆成一堆。猴子家离香蕉堆50米，猴子打算把香蕉背回家，每次最多背50根。可是，猴子嘴馋，每走一米就要吃一根香蕉。**问猴子最多能背回几根香蕉？**
分析和答案：
答案1：25（返回走的时候没吃香蕉）

猴子从香蕉堆带50根香蕉走到离家25米处，吃完25根，放下剩下的25根香蕉，原路折返！再带50根香蕉回家，此时走到离家25米处一共有50根香蕉了，再走25米继续吃掉25根，所以，还剩下25根香蕉！

 

答案2：16 （返回走的时候也吃香蕉）

分析1：在剩余香蕉大于50根之前，猴子每走1米要吃3根香蕉，因为他走1米吃掉1根后，还得往回走1米抱剩下的香蕉，这又得吃1根，然后再回到原位置需要走1米，再吃1根，所以实际上猴子走1米需要消费3个香蕉。

当走到17米的时候，猴子一共吃了17\*3=51个香蕉，还剩49，这样猴子就可以一次性搬回家了，不用往回去搬香蕉，离家还剩下50-17=33米，需要吃33根香蕉，所以到家时还剩下49-33=16根。



插入排序的时间复杂度，快排呢？

算法题：快排思路找topK

场景题：两堆大数，100亿个数和10亿个数，找交集

场景题：直播房间，一个大V发了一条消息，如何让上千万的粉丝收到这条消息，如果只是纯粹的广

播会很耗资源

select 和 epoll 的差别

C 和 C++ 的字节对齐

问学习和生活等个人信息

（2）大量time_wait会有什么问题，time_wait的时候能不能有别的客户端来请求服务器的服务

（3）TCP全连接和半连接队列（不会）

（4）滑动窗口已经发送序号为1、2、3的包，此时接收端发送ACK=4，问滑动窗口是否会移动？

（1）孤儿进程、僵尸进程

（2）进程的通信方式

（3）进程的调度方式（最短作业优先的缺点）


（1）http状态码（304不会）

（2）301和302的区别

（3）cookie和session的区别


一道经典的面试题目：
从 URL 在浏览器被被输入到页面展现的过程中发生了什么？
上面问题，或多或少可以回答出来，但是

如果继续问：收到的 HTML 如果包含几十个图片标签，这些图片是以什么方式、什么顺序、建立了多少连接、使用什么协议被下载下来的呢？

HTTP基础知识
http1.0：单工。因为是短连接，客户端发起请求之后，服务端处理完请求并收到客户端的响应后即断开连接。
http1.1 是半双工,建立长连接,出现多路复用,可先后发送多个http请求,不用等待回复,但是回复按顺序一个一个回复.默认开启长连接keep-alive,支持请求的流水线（pipelining)。
http2.0是全双工,一个消息发送后不用等待接受,第二个消息可以直接发送.允许服务端主动向客户端发送数据。
tcp是全双工的，但它的上层可能支持半双工，比如http1.x，也有可能支持全双工，比如http2.0，也有可能是单工，比如http1.0。----下一层给上一层通过接口提供服务。
五个问题
现代浏览器在与服务器建立了一个 TCP 连接后是否会在一个 HTTP 请求完成后断开？什么情况下会断开？

一个 TCP 连接可以对应几个 HTTP 请求？

一个 TCP 连接中 HTTP 请求发送可以一起发送么（比如一起发三个请求，再三个响应一起接收）？

为什么有的时候刷新页面不需要重新建立 SSL 连接？

浏览器对同一 Host 建立 TCP 连接到数量有没有限制？

第一个问题
根据基础知道第一条：HTTP1.0为短连接。这样每次请求都会花费很长时间，紧接着HTTP1.1半连接加入Connection:keep-alive，长连接，可以对TCP连接进行复用。
如何验证前两次HTTP使用的是同一个TCP连接？
第一次连接，必须有初始化连接和SSL的开销。
第二次连接，初始化连接和SSL开销消失了，说明使用的是同一个TCP连接。

总结：HTTP1.0会直接断开，HTTP1.1 默认情况下建立 TCP 连接不会断开，只有在请求报头中声明 Connection: close 才会在请求完成后关闭连接。

SSL的引入：起初是因为HTTP在传输数据时使用的是明文（虽然说POST提交的数据时放在报体里看不到的，但是还是可以通过抓包工具窃取到）是不安全的，为了解决这一隐患网景公司推出了SSL安全套接字协议层，SSL是基于HTTP之下TCP之上的一个协议层，是基于HTTP标准并对TCP传输数据时进行加密，所以HPPTS是HTTP+SSL/TCP的简称。

第二个问题
一个 TCP 连接可以对应几个 HTTP 请求？

根据第一个问题，如果维持连接，一个 TCP 连接是可以发送多个 HTTP 请求的。

第三个问题
一个 TCP 连接中 HTTP 请求发送可以一起发送么（比如一起发三个请求，再三个响应一起接收）？

HTTP/1.1存在一个问题：单个TCP连接在同一个时刻只能处理一个请求。意思是说：两个请求的生命周期不能重叠，任意两个 HTTP 请求从开始到结束的时间在同一个 TCP 连接里不能重叠。
虽然 HTTP/1.1 规范中规定了 Pipelining 来试图解决这个问题，但是这个功能在浏览器中默认是关闭的。

原因：由于 HTTP/1.1 是个文本协议，同时返回的内容也并不能区分对应于哪个发送的请求，所以顺序必须维持一致。比如你向服务器发送了两个请求 GET/query?q=A 和 GET/query?q=B，服务器返回了两个结果，浏览器是没有办法根据响应结果来判断响应对应于哪一个请求的。

Pipelining存在的问题：HTTP1.1默认不开启的原因

一些代理服务器不能正确的处理 HTTP Pipelining。

正确的流水线实现是复杂的。

Head-of-line Blocking 连接头阻塞：在建立起一个 TCP 连接之后，假设客户端在这个连接连续向服务器发送了几个请求。按照标准，服务器应该按照收到请求的顺序返回结果，假设服务器在处理首个请求时花费了大量时间，那么后面所有的请求都需要等着首个请求结束才能响应。

HTTP/2.0应运而生，提供了Multiplexing多路传输特性，可以在一个 TCP 连接中同时完成多个 HTTP 请求。至于 Multiplexing 具体怎么实现的就是另一个问题了。同一个Connection，并行完成。

总结：在 HTTP/1.1 存在 Pipelining 技术可以完成这个多个请求同时发送，但是由于浏览器默认关闭，所以可以认为这是不可行的。在 HTTP2 中由于 Multiplexing 特点的存在，多个 HTTP 请求可以在同一个 TCP 连接中并行进行。

那么在 HTTP/1.1 时代，浏览器是如何提高页面加载效率的呢？主要有下面两点：

维持和服务器已经建立的 TCP 连接，在同一连接上顺序处理多个请求。

和服务器建立多个 TCP 连接。

第四个问题
为什么有的时候刷新页面不需要重新建立 SSL 连接？

第一道题目，比较是否是使用同一个TCP连接已有答案。TCP 连接有的时候会被浏览器和服务端维持一段时间。TCP 不需要重新建立，SSL 自然也会用之前的

第五个问题
浏览器对同一 Host 建立 TCP 连接到数量有没有限制？

假设我们还处在 HTTP/1.1 时代，那个时候没有多路传输，当浏览器拿到一个有几十张图片的网页该怎么办呢？如果对TCP连接数量没有限制，网页有1000张图片，意味着要开1000个TCP连接，这对于自己的电脑和服务器是不允许的。如果只开一个TCP连接的话，用户的等待时间会比较长。

Chrome 最多允许对同一个 Host 建立六个 TCP 连接。不同的浏览器有一些区别。

那么回到最开始的问题，收到的 HTML 如果包含几十个图片标签，这些图片是以什么方式、什么顺序、建立了多少连接、使用什么协议被下载下来的呢？

如果图片都是 HTTPS 连接并且在同一个域名下，那么浏览器在 SSL 握手之后会和服务器商量能不能用 HTTP2，如果能的话就使用 Multiplexing 功能在这个连接上进行多路传输。不过也未必会所有挂在这个域名的资源都会使用一个 TCP 连接去获取，但是可以确定的是 Multiplexing 很可能会被用到。

如果发现用不了 HTTP2 呢？或者用不了 HTTPS（现实中的 HTTP2 都是在 HTTPS 上实现的，所以也就是只能使用 HTTP/1.1）。那浏览器就会在一个 HOST 上建立多个 TCP 连接，连接数量的最大限制取决于浏览器设置，这些连接会在空闲的时候被浏览器用来发送新的请求，如果所有的连接都正在发送请求呢？那其他的请求就只能等等了。

other
TLS（Transport Layer Security 安全传输层协议）是SSL(Secure Socket Layer 安全套接层）的新版本3.1。
Pipelining:一个支持持久连接的客户端可以在一个连接中发送多个请求（不需要等待任意请求的响应）。收到请求的服务器必须按照请求收到的顺序发送响应。


线性分配内存的话就是内存压缩，或者分代回收这些比较需要耗时的操作，伴随着内存的移动。

golang基于空闲链表的tcmalloc算法，golang gc的四个阶段:
1、清扫完所有垃圾，stw来开启写屏障
2、开始标记栈上所有为黑色，新在栈上也为黑色，堆上被新增和删除的对象都为灰色，占用四分之一cpu和辅助标记来稳定整个标记过程
3、stw来删除写屏障，表示标记完成
4、标记完成就开始进行清扫，归还内存。
gc触发阈值是上次清扫完成的内存的翻倍，但是会随着cpu，内存分配速度之类的加快。
官方的解释是，如果当前使用了 4M 内存，那么下次 GC 将会在内存达到 8M 的时候。
gcTriggerTime 的触发时间是由 forcegcperiod 决定的，默认是2分钟。下面我们主要看看堆内存大小触发 GC 的情况。

精细的内存分配，和栈有限的分配思想，使得不需要使用分代回收和压缩。
三色标记法属于增量式GC算法，回收器首先将所有的对象着色成白色，然后从GC Root出发，逐步把所有“可达”的对象变成灰色再到黑色，最终所有的白色对象即是“不可达”对象。

具体的实现如下：

初始时所有对象都是白色对象
从GC Root对象出发，扫描所有可达对象并标记为灰色，放入待处理队列
从队列取出一个灰色对象并标记为黑色，将其引用对象标记为灰色放入队列
重复上一步骤，直到灰色对象队列为空
此时所有剩下的白色对象就是垃圾对象

强三色不变性：黑色对象永远不会指向白色对象
弱三色不变性：黑色对象指向的白色对象至少包含一条由灰色对象经过白色对象的可达路径

优化手段：
1、sync.pool
2、预分配一个很大的虚内存来减少调步算法的控制
3、gc=-1 && setLimitMemory=100m
4、新版本堆外分配内存

Go 提供了另外一个工具 go vet 来做补充, 用这个工具是能检测出来不可复制的类型是否被复制过.
go race检测竞争

Go 提供的不可复制的类型基本上就是 sync 包内的所有类型: atomic.Value, sync.Mutex, sync.Cond, sync.RWMutex, sync.Map, sync.Pool, sync.WaitGroup.

空结构体在结构体中的前面和中间时，是不占用空间的，但是当空结构体放到结构体中的最后时，会进行特殊填充，struct { } 作为最后一个字段，会被填充对齐到前一个字段的大小，地址偏移对齐规则不变；

一个为nil的索引，不可以进行索引，否则会引发panic，其他操作是可以。
根据运行结果我们可以得出关闭一个nil的channel会导致程序panic，在使用上我们要注意这个问题，还有有一个需要注意的问题：一个nil的channel读写数据都会造成永远阻塞。
根据运行结果我们可以看出，一个nil的map可以读数据，但是不可以写入数据，否则会发生panic，所以要使用map一定要使用make进行初始化。
nil的比较我们可以分为以下两种情况：

nil标识符的比较
nil的值比较

errgroup
type errgroup struct {    
	// context 的 cancel 方法	
	cancel func()    
	// 复用 WaitGroup	
	wg sync.WaitGroup	
	// 用来保证只会接受一次错误	
	errOnce sync.Once    
	// 保存第一个返回的错误	
	err     error
}

slice是一个底层数组指针，cap和len 所以可以做到随意[:],扩容是翻倍和1.25. 危险是可能一个小切片底部用着一个大数组。


arena区域就是我们所谓的堆区，Go动态分配的内存都是在这个区域，它把内存分割成8KB大小的页，一些页组合起来称为mspan。

bitmap区域标识arena区域哪些地址保存了对象，并且用4bit标志位表示对象是否包含指针、GC标记信息。bitmap中一个byte大小的内存对应arena区域中4个指针大小（指针大小为 8B ）的内存，所以bitmap区域的大小是512GB/(4*8B)=16GB。

从上图其实还可以看到bitmap的高地址部分指向arena区域的低地址部分，也就是说bitmap的地址是由高地址向低地址增长的。

spans区域存放mspan（也就是一些arena分割的页组合起来的内存管理基本单元，后文会再讲）的指针，每个指针对应一页，所以spans区域的大小就是512GB/8KB*8B=512MB。除以8KB是计算arena区域的页数，而最后乘以8是计算spans区域所有指针的大小。创建mspan的时候，按页填充对应的spans区域，在回收object时，根据地址很容易就能找到它所属的mspan。

type mspan struct {
    next *mspan     // 链表下一个span地址
    prev *mspan     // 链表前一个span地址
    list *mSpanList // 链表地址, 用于DEBUG
​
    startAddr uintptr // 该span在arena区域的起始地址
    npages    uintptr // 该span占用arena区域page的数量
​
    manualFreeList gclinkptr // 空闲对象列表
​
    // freeindex是0~nelems的位置索引, 标记当前span中下一个空对象索引
    freeindex uintptr 
    // 当前span中管理的对象数
    nelems uintptr 
​
    allocCache uint64  // 从freeindex开始的位标记 用于快速查找内存中未被使用的内存
    allocBits  *gcBits // 该mspan中对象的位图
    gcmarkBits *gcBits // 该mspan中标记的位图,用于垃圾回收
    
    sweepgen    uint32
    divMul      uint16        // for divide by elemsize - divMagic.mul
    baseMask    uint16        // if non-0, elemsize is a power of 2, & this will get object allocation base
    allocCount  uint16        // number of allocated objects
    spanclass   spanClass     // size class and noscan (uint8)
    state       mSpanStateBox // mSpanInUse etc; accessed atomically (get/set methods)
    needzero    uint8         // needs to be zeroed before allocation
    divShift    uint8         // for divide by elemsize - divMagic.shift
    divShift2   uint8         // for divide by elemsize - divMagic.shift2
    elemsize    uintptr       // computed from sizeclass or from npages 用于计算mspan管理了多少内存
    limit       uintptr       // end of data in span span的结束地址值
    speciallock mutex         // guards specials list
    specials    *special      // linked list of special records sorted by offset.