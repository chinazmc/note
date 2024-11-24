#golang  #note 

# go pprof发现存在内存问题

> 有情提醒：如果对pprof不了解，可以先看[go pprof基本知识](#go pprof基本知识)，这是下一节，看完再倒回来看。

如果你Google或者百度，Go程序内存泄露的文章，它总会告诉你使用**pprof heap**，能够生成漂亮的调用路径图，火焰图等等，然后你根据调用路径就能定位内存泄露问题，我最初也是对此深信不疑，尝试了若干天后，只是发现内存泄露跟某种场景有关，根本找不到内存泄露的根源，**如果哪位朋友用heap就能定位内存泄露的线上问题，麻烦介绍下**。

后来读了Dave的[《High Performance Go Workshop》](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html#using_more_than_one_cpu)，刷新了对heap的认识，内存pprof的简要内容如下：

[![image-20190512114048868](https://camo.githubusercontent.com/87d7c0e6876775ce6dc9d2bf1138825a35b35f80cce1454cf63ab478f15648e9/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531323131343034383836382d373633323434382e706e67)]

Dave讲了以下几点：

1.  **内存profiling记录的是堆内存分配的情况，以及调用栈信息**，并不是进程完整的内存情况，猜测这也是在go pprof中称为heap而不是memory的原因。
2.  **栈内存的分配是在调用栈结束后会被释放的内存，所以并不在内存profile中**。
3.  内存profiling是基于抽样的，默认是每1000次堆内存分配，执行1次profile记录。
4.  因为内存profiling是基于抽样和它跟踪的是已分配的内存，而不是使用中的内存，（比如有些内存已经分配，看似使用，但实际以及不使用的内存，比如内存泄露的那部分），所以**不能使用内存profiling衡量程序总体的内存使用情况**。
5.  **Dave个人观点：使用内存profiling不能够发现内存泄露**。

基于目前对heap的认知，我有2个观点：

1.  **heap能帮助我们发现内存问题，但不一定能发现内存泄露问题**，这个看法与Dave是类似的。heap记录了内存分配的情况，我们能通过heap观察内存的变化，增长与减少，内存主要被哪些代码占用了，程序存在内存问题，这只能说明内存有使用不合理的地方，但并不能说明这是内存泄露。
2.  **heap在帮助定位内存泄露原因上贡献的力量微乎其微**。如第一条所言，能通过heap找到占用内存多的位置，但这个位置通常不一定是内存泄露，就算是内存泄露，也只是内存泄露的结果，并不是真正导致内存泄露的根源。

接下来，我介绍怎么用heap发现问题，然后再解释为什么heap几乎不能定位内存泄露的根因。

## 怎么用heap发现内存问题

使用pprof的heap能够获取程序运行时的内存信息，在程序平稳运行的情况下，每个一段时间使用heap获取内存的profile，**然后使用`base`能够对比两个profile文件的差别，就像`diff`命令一样显示出增加和减少的变化**，使用一个简单的demo来说明heap和base的使用，依然使用demo2进行展示。
```go
// 展示内存增长和pprof，并不是泄露
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
	"os"
	"time"
)

// 运行一段时间：fatal error: runtime: out of memory
func main() {
	// 开启pprof
	go func() {
		ip := "0.0.0.0:6060"
		if err := http.ListenAndServe(ip, nil); err != nil {
			fmt.Printf("start pprof failed on %s\n", ip)
			os.Exit(1)
		}
	}()

	tick := time.Tick(time.Second / 100)
	var buf []byte
	for range tick {
		buf = append(buf, make([]byte, 1024*1024)...)
	}
}
```

将上面代码运行起来，执行以下命令获取profile文件，Ctrl-D退出，1分钟后再获取1次。
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

我已经获取到了两个profile文件：
```bash
$ ls
pprof.demo2.alloc_objects.alloc_space.inuse_objects.inuse_space.001.pb.gz
pprof.demo2.alloc_objects.alloc_space.inuse_objects.inuse_space.002.pb.gz
```

使用`base`把001文件作为基准，然后用002和001对比，先执行`top`看`top`的对比，然后执行`list main`列出`main`函数的内存对比，结果如下：
```bash
$ go tool pprof -base pprof.demo2.alloc_objects.alloc_space.inuse_objects.inuse_space.001.pb.gz pprof.demo2.alloc_objects.alloc_space.inuse_objects.inuse_space.002.pb.gz

File: demo2
Type: inuse_space
Time: May 14, 2019 at 2:33pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
(pprof)
(pprof) top
Showing nodes accounting for 970.34MB, 32.30% of 3003.99MB total
      flat  flat%   sum%        cum   cum%
  970.34MB 32.30% 32.30%   970.34MB 32.30%  main.main   // 看这
         0     0% 32.30%   970.34MB 32.30%  runtime.main
(pprof)
(pprof)
(pprof) list main.main
Total: 2.93GB
ROUTINE ======================== main.main in /home/ubuntu/heap/demo2.go
  970.34MB   970.34MB (flat, cum) 32.30% of Total
         .          .     20:	}()
         .          .     21:
         .          .     22:	tick := time.Tick(time.Second / 100)
         .          .     23:	var buf []byte
         .          .     24:	for range tick {
  970.34MB   970.34MB     25:		buf = append(buf, make([]byte, 1024*1024)...) // 看这
         .          .     26:	}
         .          .     27:}
         .          .     28:
```

`top`列出了`main.main`和`runtime.main`，`main.main`就是我们编写的main函数，`runtime.main`是runtime包中的main函数，也就是所有main函数的入口，这里不多介绍了，有兴趣可以看之前的调度器文章[《Go调度器系列（2）宏观看调度器》](http://lessisbetter.site/2019/03/26/golang-scheduler-2-macro-view/)。

`top`显示`main.main` 第2次内存占用，比第1次内存占用多了970.34MB。

`list main.main`告诉了我们增长的内存都在这一行：

buf = append(buf, make([]byte, 1024*1024)...)

001和002 profile的文件不进去看了，你本地测试下计算差值，绝对是刚才对比出的970.34MB。

## heap“不能”定位内存泄露

heap能显示内存的分配情况，以及哪行代码占用了多少内存，我们能轻易的找到占用内存最多的地方，如果这个地方的数值还在不断怎大，基本可以认定这里就是内存泄露的位置。

曾想按图索骥，从内存泄露的位置，根据调用栈向上查找，总能找到内存泄露的原因，这种方案看起来是不错的，但实施起来却找不到内存泄露的原因，结果是事半功倍。

原因在于一个Go程序，其中有大量的goroutine，这其中的调用关系也许有点复杂，也许内存泄露是在某个三方包里。举个栗子，比如下面这幅图，每个椭圆代表1个goroutine，其中的数字为编号，箭头代表调用关系。heap profile显示g111（最下方标红节点）这个协程的代码出现了泄露，任何一个从g101到g111的调用路径都可能造成了g111的内存泄露，有2类可能：

1.  该goroutine只调用了少数几次，但消耗了大量的内存，说明每个goroutine调用都消耗了不少内存，**内存泄露的原因基本就在该协程内部**。
2.  该goroutine的调用次数非常多，虽然每个协程调用过程中消耗的内存不多，但该调用路径上，协程数量巨大，造成消耗大量的内存，并且这些goroutine由于某种原因无法退出，占用的内存不会释放，**内存泄露的原因在到g111调用路径上某段代码实现有问题，造成创建了大量的g111**。

**第2种情况，就是goroutine泄露，这是通过heap无法发现的，所以heap在定位内存泄露这件事上，发挥的作用不大**。

[![image-20190512144150064](https://camo.githubusercontent.com/95dce54f86e60429f9a5e5a28ed299364e023af81c4aa12482cfed7d85b2c2b6/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531323134343135303036342d373634333331302e706e67)](https://camo.githubusercontent.com/95dce54f86e60429f9a5e5a28ed299364e023af81c4aa12482cfed7d85b2c2b6/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531323134343135303036342d373634333331302e706e67)

---

# goroutine泄露怎么导致内存泄露

## 什么是goroutine泄露

如果你启动了1个goroutine，但并没有符合预期的退出，直到程序结束，此goroutine才退出，这种情况就是goroutine泄露。

> 提前思考：什么会导致goroutine无法退出/阻塞？

## goroutine泄露怎么导致内存泄露

每个goroutine占用2KB内存，泄露1百万goroutine至少泄露`2KB * 1000000 = 2GB`内存，为什么说至少呢？

goroutine执行过程中还存在一些变量，如果这些变量指向堆内存中的内存，GC会认为这些内存仍在使用，不会对其进行回收，这些内存谁都无法使用，造成了内存泄露。

所以goroutine泄露有2种方式造成内存泄露：

1.  goroutine本身的栈所占用的空间造成内存泄露。
2.  goroutine中的变量所占用的堆内存导致堆内存泄露，这一部分是能通过heap profile体现出来的。

Dave在文章中也提到了，如果不知道何时停止一个goroutine，这个goroutine就是潜在的内存泄露：

> [7.1.1 Know when to stop a goroutine](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html#know_when_to_stop_a_goroutine)
> 
> If you don’t know the answer, that’s a potential memory leak as the goroutine will pin its stack’s memory on the heap, as well as any heap allocated variables reachable from the stack.

## 怎么确定是goroutine泄露引发的内存泄露

掌握了前面的pprof命令行的基本用法，很快就可以确认是否是goroutine泄露导致内存泄露，如果你不记得了，马上回去看一下[go pprof基本知识](#go pprof基本知识)。

**判断依据：在节点正常运行的情况下，隔一段时间获取goroutine的数量，如果后面获取的那次，某些goroutine比前一次多，如果多获取几次，是持续增长的，就极有可能是goroutine泄露**。

goroutine导致内存泄露的demo：
```go
// goroutine泄露导致内存泄露
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
	"os"
	"time"
)

func main() {
	// 开启pprof
	go func() {
		ip := "0.0.0.0:6060"
		if err := http.ListenAndServe(ip, nil); err != nil {
			fmt.Printf("start pprof failed on %s\n", ip)
			os.Exit(1)
		}
	}()

	outCh := make(chan int)
	// 死代码，永不读取
	go func() {
		if false {
			<-outCh
		}
		select {}
	}()

	// 每s起100个goroutine，goroutine会阻塞，不释放内存
	tick := time.Tick(time.Second / 100)
	i := 0
	for range tick {
		i++
		fmt.Println(i)
		alloc1(outCh)
	}
}

func alloc1(outCh chan<- int) {
	go alloc2(outCh)
}

func alloc2(outCh chan<- int) {
	func() {
		defer fmt.Println("alloc-fm exit")
		// 分配内存，假用一下
		buf := make([]byte, 1024*1024*10)
		_ = len(buf)
		fmt.Println("alloc done")

		outCh <- 0 // 53行
	}()
}
```

编译并运行以上代码，然后使用`go tool pprof`获取gorourine的profile文件。

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

已经通过pprof命令获取了2个goroutine的profile文件:
```bash

$ ls
/home/ubuntu/pprof/pprof.leak_demo.goroutine.001.pb.gz
/home/ubuntu/pprof/pprof.leak_demo.goroutine.002.pb.gz

同heap一样，我们可以使用`base`对比2个goroutine profile文件：

$go tool pprof -base pprof.leak_demo.goroutine.001.pb.gz pprof.leak_demo.goroutine.002.pb.gz

File: leak_demo
Type: goroutine
Time: May 16, 2019 at 2:44pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
(pprof) top
Showing nodes accounting for 20312, 100% of 20312 total
      flat  flat%   sum%        cum   cum%
     20312   100%   100%      20312   100%  runtime.gopark
         0     0%   100%      20312   100%  main.alloc2
         0     0%   100%      20312   100%  main.alloc2.func1
         0     0%   100%      20312   100%  runtime.chansend
         0     0%   100%      20312   100%  runtime.chansend1
         0     0%   100%      20312   100%  runtime.goparkunlock
(pprof)
```

可以看到运行到`runtime.gopark`的goroutine数量增加了20312个。再通过002文件，看一眼执行到`gopark`的goroutine数量，即挂起的goroutine数量：
```bash
go tool pprof pprof.leak_demo.goroutine.002.pb.gz
File: leak_demo
Type: goroutine
Time: May 16, 2019 at 2:47pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 24330, 100% of 24331 total
Dropped 32 nodes (cum <= 121)
      flat  flat%   sum%        cum   cum%
     24330   100%   100%      24330   100%  runtime.gopark
         0     0%   100%      24326   100%  main.alloc2
         0     0%   100%      24326   100%  main.alloc2.func1
         0     0%   100%      24326   100%  runtime.chansend
         0     0%   100%      24326   100%  runtime.chansend1
         0     0%   100%      24327   100%  runtime.goparkunlock
```

显示有24330个goroutine被挂起，这不是goroutine泄露这是啥？已经能确定八九成goroutine泄露了。

是什么导致如此多的goroutine被挂起而无法退出？接下来就看怎么定位goroutine泄露。

---

# 定位goroutine泄露的2种方法

使用pprof有2种方式，一种是web网页，一种是`go tool pprof`命令行交互，这两种方法查看goroutine都支持，但有轻微不同，也有各自的优缺点。

我们先看Web的方式，再看命令行交互的方式，这两种都很好使用，结合起来用也不错。

## Web可视化查看

Web方式适合web服务器的端口能访问的情况，使用起来方便，有2种方式：

1.  **查看某条调用路径上，当前阻塞在此goroutine的数量**
2.  查看所有goroutine的运行栈（调用路径），可以**显示阻塞在此的时间**

### 方式一

url请求中设置debug=1：
```bash
http://ip:port/debug/pprof/goroutine?debug=1
```

效果如下：[![](https://camo.githubusercontent.com/23940cec786d682113976a6958d6664a78c3ede72129ca99112acb4bcf307a7d/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531363134333734303536372d373938383636302e706e67)](https://camo.githubusercontent.com/23940cec786d682113976a6958d6664a78c3ede72129ca99112acb4bcf307a7d/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531363134333734303536372d373938383636302e706e67)

看起来密密麻麻的，其实简单又十分有用，看上图标出来的部分，手机上图看起来可能不方便，那就放大图片，或直接看下面各字段的含义：

1.  `goroutine profile: total 32023`：32023是**goroutine的总数量**，
2.  `32015 @ 0x42e15a 0x42e20e 0x40534b 0x4050e5 ...`：32015代表当前有32015个goroutine运行这个调用栈，并且停在相同位置，@后面的十六进制，现在用不到这个数据，所以暂不深究了。
3.  下面是当前goroutine的**调用栈**，列出了**函数和所在文件的行数，这个行数对定位很有帮助**，如下：

```
32015 @ 0x42e15a 0x42e20e 0x40534b 0x4050e5 0x6d8559 0x6d831b 0x45abe1
#	0x6d8558	main.alloc2.func1+0xf8	/home/ubuntu/heap/leak_demo.go:53
#	0x6d831a	main.alloc2+0x2a	/home/ubuntu/heap/leak_demo.go:54
```

根据上面的提示，就能判断32015个goroutine运行到`leak_demo.go`的53行：
```go

func alloc2(outCh chan<- int) {
	func() {
		defer fmt.Println("alloc-fm exit")
		// 分配内存，假用一下
		buf := make([]byte, 1024*1024*10)
		_ = len(buf)
		fmt.Println("alloc done")

		outCh <- 0 // 53行
	}()
}
```

阻塞的原因是outCh这个写操作无法完成，outCh是无缓冲的通道，并且由于以下代码是死代码，所以goroutine始终没有从outCh读数据，造成outCh阻塞，进而造成无数个alloc2的goroutine阻塞，形成内存泄露：
```bash
if false {
    <-outCh
}
```

### 方式二

url请求中设置debug=2：
```bash
http://ip:port/debug/pprof/goroutine?debug=2
```

[![](https://camo.githubusercontent.com/3ad69b4ccf1ec37a05e2d264edb7f05765b3d04f6e311b3d8430239a0c36ce23/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531363134333533373333392d373938383533372e706e67)](https://camo.githubusercontent.com/3ad69b4ccf1ec37a05e2d264edb7f05765b3d04f6e311b3d8430239a0c36ce23/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30352d696d6167652d32303139303531363134333533373333392d373938383533372e706e67)

第2种方式和第1种方式是互补的，它可以看到每个goroutine的信息：

1.  `goroutine 20 [chan send, 2 minutes]`：20是goroutine id，`[]`中是当前goroutine的状态，阻塞在写channel，并且阻塞了2分钟，长时间运行的系统，你能看到阻塞时间更长的情况。
2.  同时，也可以看到调用栈，看当前执行停到哪了：`leak_demo.go`的53行，
```bash
goroutine 20 [chan send, 2 minutes]:
main.alloc2.func1(0xc42015e060)
	/home/ubuntu/heap/leak_demo.go:53 +0xf9  // 这
main.alloc2(0xc42015e060)
	/home/ubuntu/heap/leak_demo.go:54 +0x2b
created by main.alloc1
	/home/ubuntu/heap/leak_demo.go:42 +0x3f
```

## 命令行交互式方法

Web的方法是简单粗暴，无需登录服务器，浏览器打开看看就行了。但就像前面提的，没有浏览器可访问时，命令行交互式才是最佳的方式，并且也是手到擒来，感觉比Web一样方便。

命令行交互式只有1种获取goroutine profile的方法，不像Web网页分`debug=1`和`debug=2`2中方式，并将profile文件保存到本地：
```bash
// 注意命令没有`debug=1`，debug=1，加debug有些版本的go不支持
$ go tool pprof http://0.0.0.0:6060/debug/pprof/goroutine
Fetching profile over HTTP from http://localhost:6061/debug/pprof/goroutine
Saved profile in /home/ubuntu/pprof/pprof.leak_demo.goroutine.001.pb.gz  // profile文件保存位置
File: leak_demo
Type: goroutine
Time: May 16, 2019 at 2:44pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
```

命令行只需要掌握3个命令就好了，上面介绍过了，详细的倒回去看[top](https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/go-goroutine-leak.md#top), [list](https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/go-goroutine-leak.md#list), [traces](https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/go-goroutine-leak.md#traces)：

1.  **top**：显示正运行到某个函数goroutine的数量
2.  **traces**：显示所有goroutine的调用栈
3.  **list**：列出代码详细的信息。

我们依然使用`leak_demo.go`这个demo，
```bash
$  go tool pprof -base pprof.leak_demo.goroutine.001.pb.gz pprof.leak_demo.goroutine.002.pb.gz
File: leak_demo
Type: goroutine
Time: May 16, 2019 at 2:44pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
(pprof)
(pprof) top
Showing nodes accounting for 20312, 100% of 20312 total
      flat  flat%   sum%        cum   cum%
     20312   100%   100%      20312   100%  runtime.gopark
         0     0%   100%      20312   100%  main.alloc2
         0     0%   100%      20312   100%  main.alloc2.func1
         0     0%   100%      20312   100%  runtime.chansend
         0     0%   100%      20312   100%  runtime.chansend1
         0     0%   100%      20312   100%  runtime.goparkunlock
(pprof)
(pprof) traces
File: leak_demo
Type: goroutine
Time: May 16, 2019 at 2:44pm (CST)
-----------+-------------------------------------------------------
     20312   runtime.gopark
             runtime.goparkunlock
             runtime.chansend
             runtime.chansend1 // channel发送
             main.alloc2.func1 // alloc2中的匿名函数
             main.alloc2
-----------+-------------------------------------------------------
```

top命令在[怎么确定是goroutine泄露引发的内存泄露](https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/go-goroutine-leak.md#%E6%80%8E%E4%B9%88%E7%A1%AE%E5%AE%9A%E6%98%AFgoroutine%E6%B3%84%E9%9C%B2%E5%BC%95%E5%8F%91%E7%9A%84%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2)介绍过了，直接看traces命令，traces能列出002中比001中多的那些goroutine的调用栈，这里只有1个调用栈，有20312个goroutine都执行这个调用路径，可以看到alloc2中的匿名函数`alloc2.func1`调用了写channel的操作，然后阻塞挂起了goroutine，使用list列出`alloc2.func1`的代码，显示有20312个goroutine阻塞在53行：
```bash
(pprof) list main.alloc2.func1
Total: 20312
ROUTINE ======================== main.alloc2.func1 in /home/ubuntu/heap/leak_demo.go
         0      20312 (flat, cum)   100% of Total
         .          .     48:		// 分配内存，假用一下
         .          .     49:		buf := make([]byte, 1024*1024*10)
         .          .     50:		_ = len(buf)
         .          .     51:		fmt.Println("alloc done")
         .          .     52:
         .      20312     53:		outCh <- 0  // 看这
         .          .     54:	}()
         .          .     55:}
         .          .     56:
```

**友情提醒：使用list命令的前提是程序的源码在当前机器，不然可没法列出源码。**服务器上，通常没有源码，那我们咋办呢？刚才介绍了Web查看的方式，那里会列出代码行数，我们可以使用`wget`下载网页：

```bash
$ wget http://localhost:6060/debug/pprof/goroutine?debug=1
```

下载网页后，使用编辑器打开文件，使用关键字`main.alloc2.func1`进行搜索，找到与当前相同的调用栈，就可以看到该goroutine阻塞在哪一行了，不要忘记使用`debug=2`还可以看到阻塞了多久和原因，Web方式中已经介绍了，此处省略代码几十行。

---

# 总结

文章略长，但全是干货，感谢阅读到这。然读到着了，跟定很想掌握pprof，建议实践一把，现在和大家温习一把本文的主要内容。

## goroutine泄露的本质

goroutine泄露的本质是channel阻塞，无法继续向下执行，导致此goroutine关联的内存都无法释放，进一步造成内存泄露。

## goroutine泄露的发现和定位

利用好go pprof获取goroutine profile文件，然后利用3个命令top、traces、list定位内存泄露的原因。

##goroutine泄露的场景

泄露的场景不仅限于以下两类，但因channel相关的泄露是最多的。

1.  channel的读或者写：
    1.  无缓冲channel的阻塞通常是写操作因为没有读而阻塞
    2.  有缓冲的channel因为缓冲区满了，写操作阻塞
    3.  期待从channel读数据，结果没有goroutine写
2.  select操作，select里也是channel操作，如果所有case上的操作阻塞，goroutine也无法继续执行。

## 编码goroutine泄露的建议

为避免goroutine泄露造成内存泄露，启动goroutine前要思考清楚：

1.  goroutine如何退出？
2.  是否会有阻塞造成无法退出？如果有，那么这个路径是否会创建大量的goroutine？


---

# 实战pprof前言

如果要说在 golang 开发过程进行性能调优，pprof 一定是一个大杀器般的工具。但在网上找到的教程都偏向简略，难寻真的能应用于实战的教程。这也无可厚非，毕竟 pprof 是当程序占用资源异常时才需要启用的工具，而我相信大家的编码水平和排场问题的能力是足够高的，一般不会写出性能极度堪忧的程序，且即使发现有一些资源异常占用，也会通过排查代码快速定位，这也导致 pprof 需要上战场的机会少之又少。即使大家有心想学习使用 pprof，却也常常相忘于江湖。

**既然如此，那我就送大家一个性能极度堪忧的“炸弹”程序吧！**

这程序没啥正经用途缺极度占用资源，基本覆盖了常见的性能问题。本文就是希望读者能一步一步按照提示，使用 pprof 定位这个程序的的性能瓶颈所在，借此学习 pprof 工具的使用方法。

因此，本文是一场“实验课”而非“理论课”，请读者腾出时间，脚踏实地，一步一步随实验步骤进行操作，这会是一个很有趣的冒险，不会很无聊，希望你能喜欢。

# 实验准备

这里假设你有基本的 golang 开发功底，拥有 golang 开发环境并配置了 $GOPATH，能熟练阅读简单的代码或进行简单的修改，且知道如何编译运行 golang 程序。此外，需要你大致知道 pprof 是干什么的，有一个基本印象即可，你可以花几分钟时间读一下[《Golang 大杀器之性能剖析 PProf》](https://blog.wolfogre.com/redirect/v3/A0uO_9v0ECGYMC6hksfOB8YSAwM8Cv46xcU7LxImWv3FQhhTHP5qU8VaFgY7xVoWBlrFrU0bxXVQYcU8Bk0KxTsGxRQGFgrF_wQyMDE4zP8CMDnM_wIxNcw7BswUBhbMPDxzLG4tGDESAwM8Cv46xcVaFgY7bkEGFtw7If3FPAZNCsU7Bsw8PAXMPIIcSojF)的开头部分，这不会耽误太久。

此外由于你需要运行一个“炸弹”程序，请务必确保你用于做实验的机器有充足的资源，你的机器至少需要：

-   2 核 CPU；
-   2G 内存。

注意，以上只是最低需求，你的机器配置能高于上述要求自然最好。实际运行“炸弹”时，你可以关闭电脑上其他不必要的程序，甚至 IDE 都不用开，我们的实验操作基本上是在命令行里进行的。

此外，务必确保你是在个人机器上运行“炸弹”的，能接受机器死机重启的后果（虽然这发生的概率很低）。请你务必不要在危险的边缘试探，比如在线上服务器运行这个程序。

可能说得你都有点害怕了，为打消你顾虑，我可以剧透一下“炸弹”的情况，让你安心：

-   程序会占用约 2G 内存；
-   程序占用 CPU 最高约 100%（总量按“核数 * 100%”来算）；
-   程序不涉及网络或文件读写；
-   程序除了吃资源之外没有其他危险操作。

且程序所占用的各类资源，均不会随着运行时间的增长而增长，换句话说，只要你把“炸弹”启动并正常运行了一分钟，就基本确认安全了，之后即使运行几天也不会有更多的资源占用，除了有点费电之外。

# 获取“炸弹”

炸弹程序的代码我已经放到了 [GitHub](https://blog.wolfogre.com/redirect/v3/A_4-v86v-9Btg9a9FuRKCcgSAwM8Cv46xcU7LxImWv3FQQYW3DshxTsGzDw8cyzMPIIcSogxEgMDPAr-OsXFWhYGO25BBhbcOyH9xTwGTQrFOwbMPDwFzDyCHEqIxQ) 上，你只需要在终端里运行 `go get` 便可获取，注意加上 `-d` 参数，避免下载后自动安装：

```bash
go get -d github.com/wolfogre/go-pprof-practice
cd $GOPATH/src/github.com/wolfogre/go-pprof-practice
```

我们可以简单看一下 `main.go` 文件，里面有几个帮助排除性能调问题的关键的的点，我加上了些注释方便你理解，如下：

```go
package main

import (
	// 略
	_ "net/http/pprof" // 会自动注册 handler 到 http server，方便通过 http 接口获取程序运行采样报告
	// 略
)

func main() {
	// 略

	runtime.GOMAXPROCS(1) // 限制 CPU 使用数，避免过载
	runtime.SetMutexProfileFraction(1) // 开启对锁调用的跟踪
	runtime.SetBlockProfileRate(1) // 开启对阻塞操作的跟踪

	go func() {
		// 启动一个 http server，注意 pprof 相关的 handler 已经自动注册过了
		if err := http.ListenAndServe(":6060", nil); err != nil {
			log.Fatal(err)
		}
		os.Exit(0)
	}()

	// 略
}
```

除此之外的其他代码你一律不用看，那些都是我为了模拟一个“逻辑复杂”的程序而编造的，其中大多数的问题很容易通过肉眼发现，但我们需要做的是通过 pprof 来定位代码的问题，所以为了保证实验的趣味性请不要提前阅读代码，可以实验完成后再看。

接着我们需要编译一下这个程序并运行，你不用担心依赖问题，这个程序没有任何外部依赖。

```bash
go build
./go-pprof-practice
```

运行后注意查看一下资源是否吃紧，机器是否还能扛得住，坚持一分钟，如果确认没问题，咱们再进行下一步。

控制台里应该会不停的打印日志，都是一些“猫狗虎鼠在不停地吃喝拉撒”的屁话，没有意义，不用细看。
![[Pasted image 20220711174257.png]]

## 使用 pprof

保持程序运行，打开浏览器访问 `http://localhost:6060/debug/pprof/`，可以看到如下页面：
![[Pasted image 20220711174322.png]]
页面上展示了可用的程序运行采样数据，分别有：

![[Pasted image 20220711174340.png]]

因为 cmdline 没有什么实验价值，trace 与本文主题关系不大，threadcreate 涉及的情况偏复杂，所以这三个类型的采样信息这里暂且不提。除此之外，其他所有类型的采样信息本文都会涉及到，且炸弹程序已经为每一种类型的采样信息埋藏了一个对应的性能问题，等待你的发现。

由于直接阅读采样信息缺乏直观性，我们需要借助 `go tool pprof` 命令来排查问题，这个命令是 go 原生自带的，所以不用额外安装。

我们先不用完整地学习如何使用这个命令，毕竟那太枯燥了，我们一边实战一边学习。

以下正式开始。

## 排查 CPU 占用过高

我们首先通过活动监视器（或任务管理器、top 命令，取决于你的操作系统和你的喜好），查看一下炸弹程序的 CPU 占用
可以看到 CPU 占用相当高，这显然是有问题的，我们使用 `go tool pprof` 来排场一下：

```bash
go tool pprof http://localhost:6060/debug/pprof/profile
```

等待一会儿后，进入一个交互式终端：
![[Pasted image 20220711174426.png]]
 输入 top 命令，查看 CPU 占用较高的调用：
![[Pasted image 20220711174445.png]]
很明显，CPU 占用过高是 `github.com/wolfogre/go-pprof-practice/animal/felidae/tiger.(*Tiger).Eat` 造成的。

> 注：为了保证实验节奏，关于图中 `flat`、`flat%`、`sum%`、`cum`、`cum%` 等参数的含义这里就不展开讲了，你可以先简单理解为数字越大占用情况越严重，之后可以在[《Golang 大杀器之性能剖析 PProf》](https://blog.wolfogre.com/redirect/v3/A3jsjsv0r3pCsn4x_qmqFKwSAwM8Cv46xcU7LxImWv3F_wdFRERZQ0pZxVoWBjvFWhYGWsWtTRvFOwaJVMX_BDIwMTjM_wIwOcz_AjE1zP5HBolU_1AlMjAlRTUlQTQlQTclRTYlOUQlODAlRTUlOTklQTglRTQlQjklOEIlRTYlODAlQTclRTglODMlQkQlRTUlODklOTYlRTYlOUUlOTAlMjBQUHMsbi0YMRIDAzwK_jrFxVoWBjtuQQYW3Dsh_cU8Bk0KxTsGzDw8Bcw8ghxKiMU)等其他资料中深入学习。

输入 `list Eat`，查看问题具体在代码的哪一个位置：
![[Pasted image 20220711174544.png]]
可以看到，是第 24 行那个一百亿次空循环占用了大量 CPU 时间，至此，问题定位成功！

接下来有一个扩展操作：图形化显示调用栈信息，这很酷，但是需要你事先在机器上安装 `graphviz`，大多数系统上可以轻松安装它：

```bash
brew install graphviz # for macos
apt install graphviz # for ubuntu
yum install graphviz # for centos
```

或者你也可以访问 [graphviz 官网](https://blog.wolfogre.com/redirect/v3/A421Yoc_xEV4GG_UO8tV1nMSAwM8Cv46xcU7gjwSbQjbbjsviVpukMUYBkEJFgboxTESAwM8Cv46xcVaFgY7bkEGFtw7If3FPAZNCsU7Bsw8PAXMPIIcSojF)寻找适合自己操作系统的安装方法。

安装完成后，我们继续在上文的交互式终端里输入 `web`，注意，虽然这个命令的名字叫“web”，但它的实际行为是产生一个 .svg 文件，并调用你的系统里设置的默认打开 .svg 的程序打开它。如果你的系统里打开 .svg 的默认程序并不是浏览器（比如可能是你的代码编辑器），这时候你需要设置一下默认使用浏览器打开 .svg 文件，相信这难不倒你。

你应该可以看到：
![[Pasted image 20220711174616.png]]
图中，`tiger.(*Tiger).Eat` 函数的框特别大，箭头特别粗，pprof 生怕你不知道这个函数的 CPU 占用很高，这张图还包含了很多有趣且有价值的信息，你可以多看一会儿再继续。

至此，这一小节使用 pprof 定位 CPU 占用的实验就结束了，你需要输入 `exit` 退出 pprof 的交互式终端。

为了方便进行后面的实验，我们修复一下这个问题，不用太麻烦，注释掉相关代码即可：

```go
func (t *Tiger) Eat() {
	log.Println(t.Name(), "eat")
	//loop := 10000000000
	//for i := 0; i < loop; i++ {
	//	// do nothing
	//}
}
```

之后修复问题的的方法都是注释掉相关的代码，不再赘述。你可能觉得这很粗暴，但要知道，这个实验的重点是如何使用 pprof 定位问题，我们不需要花太多时间在改代码上。

## 排查内存占用过高

重新编译炸弹程序，再次运行，可以看到 CPU 占用率已经下来了，但是内存的占用率仍然很高
我们再次运行使用 pprof 命令，注意这次使用的 URL 的结尾是 heap：

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

再一次使用 `top`、`list` 来定问问题代码：
![[Pasted image 20220711174643.png]]
可以看到这次出问题的地方在 `github.com/wolfogre/go-pprof-practice/animal/muridae/mouse.(*Mouse).Steal`，函数内容如下：

```go
func (m *Mouse) Steal() {
	log.Println(m.Name(), "steal")
	max := constant.Gi
	for len(m.buffer) * constant.Mi < max {
		m.buffer = append(m.buffer, [constant.Mi]byte{})
	}
}
```

可以看到，这里有个循环会一直向 m.buffer 里追加长度为 1 MiB 的数组，直到总容量到达 1 GiB 为止，且一直不释放这些内存，这就难怪会有这么高的内存占用了。

使用 `web` 来查看图形化展示，可以再次确认问题确实出在这里：
![[Pasted image 20220711174704.png]]
现在我们同样是注释掉相关代码来解决这个问题。

再次编译运行，查看内存占用
可以看到内存占用已经将到了 35 MB，似乎内存的使用已经恢复正常，一片祥和。

但是，内存相关的性能问题真的已经全部解决了吗？

## 排查频繁内存回收

你应该知道，频繁的 GC 对 golang 程序性能的影响也是非常严重的。虽然现在这个炸弹程序内存使用量并不高，但这会不会是频繁 GC 之后的假象呢？

为了获取程序运行过程中 GC 日志，我们需要先退出炸弹程序，再在重新启动前赋予一个环境变量，同时为了避免其他日志的干扰，使用 grep 筛选出 GC 日志查看：

```bash
GODEBUG=gctrace=1 ./go-pprof-practice | grep gc
```

日志输出如下：
![[Pasted image 20220711174731.png]]
可以看到，GC 差不多每 3 秒就发生一次，且每次 GC 都会从 16MB 清理到几乎 0MB，说明程序在不断的申请内存再释放，这是高性能 golang 程序所不允许的。

如果你希望进一步了解 golang 的 GC 日志可以查看[《如何监控 golang 程序的垃圾回收》](https://blog.wolfogre.com/redirect/v3/A9DNc05mRFLA-ZPsjfPhLuZDu-oKbuLF_wQyMDE2xf8CMDfF_wIwMcUtHy8qzDsGiVTMOxzFMRIDAzwK_jrFxVoWBjtuQQYW3Dsh_cU8Bk0KxTsGzDw8Bcw8ghxKiMU),为保证实验节奏，这里不做展开。

所以接下来使用 pprof 排查时，我们在乎的不是什么地方在占用大量内存，而是什么地方在不停地申请内存，这两者是有区别的。

由于内存的申请与释放频度是需要一段时间来统计的，所有我们保证炸弹程序已经运行了几分钟之后，再运行命令：

```bash
go tool pprof http://localhost:6060/debug/pprof/allocs
```

同样使用 top、list、web 大法：
![[Pasted image 20220711174756.png]]

![[Pasted image 20220711174818.png]]
可以看到 `github.com/wolfogre/go-pprof-practice/animal/canidae/dog.(*Dog).Run` 会进行无意义的内存申请，而这个函数又会被频繁调用，这才导致程序不停地进行 GC:

```go
func (d *Dog) Run() {
	log.Println(d.Name(), "run")
	_ = make([]byte, 16 * constant.Mi)
}
```

这里有个小插曲，你可尝试一下将 `16 * constant.Mi` 修改成一个较小的值，重新编译运行，会发现并不会引起频繁 GC，原因是在 golang 里，对象是使用堆内存还是栈内存，由编译器进行逃逸分析并决定，如果对象不会逃逸，便可在使用栈内存，但总有意外，就是对象的尺寸过大时，便不得不使用堆内存。所以这里设置申请 16 MiB 的内存就是为了避免编译器直接在栈上分配，如果那样得话就不会涉及到 GC 了。

我们同样注释掉问题代码，重新编译执行，可以看到这一次，程序的 GC 频度要低很多，以至于短时间内都看不到 GC 日志了：

## 排查协程泄露

由于 golang 自带内存回收，所以一般不会发生内存泄露。但凡事都有例外，在 golang 中，协程本身是可能泄露的，或者叫协程失控，进而导致内存泄露。

我们在浏览器里可以看到，此时程序的协程数已经多达 106 条：
![[Pasted image 20220711174848.png]]
虽然 106 条并不算多，但对于这样一个小程序来说，似乎还是不正常的。为求安心，我们再次是用 pprof 来排查一下：

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

同样是 top、list、web 大法：
![[Pasted image 20220711174908.png]]
![[Pasted image 20220711174929.png]]
可能这次问题藏得比较隐晦，但仔细观察还是不难发现，问题在于 `github.com/wolfogre/go-pprof-practice/animal/canidae/wolf.(*Wolf).Drink` 在不停地创建没有实际作用的协程：

```go
func (w *Wolf) Drink() {
	log.Println(w.Name(), "drink")
	for i := 0; i < 10; i++ {
		go func() {
			time.Sleep(30 * time.Second)
		}()
	}
}
```

可以看到，Drink 函数每次会释放 10 个协程出去，每个协程会睡眠 30 秒再退出，而 Drink 函数又会被反复调用，这才导致大量协程泄露，试想一下，如果释放出的协程会永久阻塞，那么泄露的协程数便会持续增加，内存的占用也会持续增加，那迟早是会被操作系统杀死的。

我们注释掉问题代码，重新编译运行可以看到，协程数已经降到 4 条了：

## 排查锁的争用

到目前为止，我们已经解决这个炸弹程序的所有资源占用问题，但是事情还没有完，我们需要进一步排查那些会导致程序运行慢的性能问题，这些问题可能并不会导致资源占用，但会让程序效率低下，这同样是高性能程序所忌讳的。

我们首先想到的就是程序中是否有不合理的锁的争用，我们倒一倒，回头看看上一张图，虽然协程数已经降到 4 条，但还显示有一个 mutex 存在争用问题。

相信到这里，你已经触类旁通了，无需多言，开整。

```bash
go tool pprof http://localhost:6060/debug/pprof/mutex
```

同样是 top、list、web 大法：
![[Pasted image 20220711174955.png]]
![[Pasted image 20220711175005.png]]
可以看出来这问题出在 `github.com/wolfogre/go-pprof-practice/animal/canidae/wolf.(*Wolf).Howl`。但要知道，在代码中使用锁是无可非议的，并不是所有的锁都会被标记有问题，我们看看这个有问题的锁那儿触雷了。

```go
func (w *Wolf) Howl() {
	log.Println(w.Name(), "howl")

	m := &sync.Mutex{}
	m.Lock()
	go func() {
		time.Sleep(time.Second)
		m.Unlock()
	}()
	m.Lock()
}
```

可以看到，这个锁由主协程 Lock，并启动子协程去 Unlock，主协程会阻塞在第二次 Lock 这儿等待子协程完成任务，但由于子协程足足睡眠了一秒，导致主协程等待这个锁释放足足等了一秒钟。虽然这可能是实际的业务需要，逻辑上说得通，并不一定真的是性能瓶颈，但既然它出现在我写的“炸弹”里，就肯定不是什么“业务需要”啦。

所以，我们注释掉这段问题代码，重新编译执行，继续。

## 排查阻塞操作

好了，我们开始排查最后一个问题。

在程序中，除了锁的争用会导致阻塞之外，很多逻辑都会导致阻塞。
![[Pasted image 20220711175023.png]]
可以看到，这里仍有 2 个阻塞操作，虽然不一定是有问题的，但我们保证程序性能，我们还是要老老实实排查确认一下才对。

```bash
go tool pprof http://localhost:6060/debug/pprof/block
```

top、list、web，你懂得。
![[Pasted image 20220711175043.png]]
![[Pasted image 20220711175054.png]]
可以看到，阻塞操作位于 `github.com/wolfogre/go-pprof-practice/animal/felidae/cat.(*Cat).Pee`：

```go
func (c *Cat) Pee() {
	log.Println(c.Name(), "pee")

	<-time.After(time.Second)
}
```

你应该可以看懂，不同于睡眠一秒，这里是从一个 channel 里读数据时，发生了阻塞，直到这个 channel 在一秒后才有数据读出，这就导致程序阻塞了一秒而非睡眠了一秒。

这里有个疑点，就是上文中是可以看到有两个阻塞操作的，但这里只排查出了一个，我没有找到其准确原因，但怀疑另一个阻塞操作是程序监听端口提供 porof 查询时，涉及到 IO 操作发生了阻塞，即阻塞在对 HTTP 端口的监听上，但我没有进一步考证。

不管怎样，恭喜你完整地完成了这个实验。

## 思考题

另有一些问题，虽然比较重要，但碍于篇幅（其实是我偷懒），就以思考题的形式留给大家了。

1.  每次进入交互式终端，都会提示“type ‘help’ for commands, ‘o’ for options”，你有尝试过查看有哪些命令和哪些选项吗？有重点了解一下“sample_index”这个选项吗？
2.  关于内存的指标，有申请对象数、使用对象数、申请空间大小、使用空间大小，本文用的是什么指标？如何查看不同的指标？（提示：在内存实验中，试试查看、修改“sample_index”选项。）
3.  你有听说过火焰图吗？要不要在试验中生成一下火焰图？
4.  main 函数中的 `runtime.SetMutexProfileFraction` 和 `runtime.SetBlockProfileRate` 是如何影响指标采样的？它们的参数的含义是什么？
5.  评论区有很多很有价值的提问，你有注意到吗？

## 最后

碍于我的水平有限，实验中还有很多奇怪的细节我只能暂时熟视无睹（比如“排查内存占用过高”一节中，为什么实际申请的是 1.5 GiB 内存），如果这些奇怪的细节你也发现了，并痛斥我假装睁眼瞎，那么我的目的就达到了……

——还记得吗，本文的目的是让你熟悉使用 pprof，消除对它的陌生感，并能借用它进一步得了解 golang。而你通过这次试验，发现了程序的很多行为不同于你以往的认知或假设，并抱着好奇心，开始向比深处更深处迈进，那么，我何尝不觉得这是本文的功德呢？



# 高德Go生态的服务稳定性建设｜性能优化的实战总结
# 一、性能调优-理论篇

## 1.1 衡量指标

优化的第一步是先衡量一个应用性能的好坏，性能良好的应用自然不必费心优化，性能较差的应用，则需要从多个方面来考察，找到木桶里的短板，才能对症下药。那么如何衡量一个应用的性能好坏呢？最主要的还是通过观察应用对核心资源的占用情况以及应用的稳定性指标来衡量。所谓核心资源，就是相对稀缺的，并且可能会导致应用无法正常运行的资源，常见的核心资源如下：

-   cpu：对于偏计算型的应用，cpu往往是影响性能好坏的关键，如果代码中存在无限循环，或是频繁的线程上下文切换，亦或是糟糕的垃圾回收策略，都将导致cpu被大量占用，使得应用程序无法获取到足够的cpu资源，从而响应缓慢，性能变差。  
-   内存：内存的读写速度非常快，往往不是性能的瓶颈，但是内存相对来说容量有限切价格昂贵，如果应用大量分配内存而不及时回收，就会造成内存溢出或泄漏，应用无法分配新的内存，便无法正常运行，这将导致很严重的事故。  
-   带宽：对于偏网络I/O型的应用，例如网关服务，带宽的大小也决定了应用的性能好坏，如果带宽太小，当系统遇到大量并发请求时，带宽不够用，网络延迟就会变高，这个虽然对服务端可能无感知，但是对客户端则是影响甚大。  
-   磁盘：相对内存来说，磁盘价格低廉，容量很大，但是读写速度较慢，如果应用频繁的进行磁盘I/O，那性能可想而知也不会太好。

以上这些都是系统资源层面用于衡量性能的指标，除此之外还有应用本身的稳定性指标：

-   异常率：也叫错误率，一般分两种，执行超时和应用panic。panic会导致应用不可用，虽然服务通常都会配置相应的重启机制，确保偶然的应用挂掉后能重启再次提供服务，但是经常性的panic，会导致应用频繁的重启，减少了应用正常提供服务的时间，整体性能也就变差了。异常率是非常重要的指标，服务的稳定和可用是一切的前提，如果服务都不可用了，还谈何性能优化。  

-   响应时间(RT)：包括平均响应时间，百分位(top percentile)响应时间。响应时间是指应用从收到请求到返回结果后的耗时，反应的是应用处理请求的快慢。通常平均响应时间无法反应服务的整体响应情况，响应慢的请求会被响应快的请求平均掉，而响应慢的请求往往会给用户带来糟糕的体验，即所谓的长尾请求，所以我们需要百分位响应时间，例如tp99响应时间，即99%的请求都会在这个时间内返回。  

-   吞吐量：主要指应用在一定时间内处理请求/事务的数量，反应的是应用的负载能力。我们当然希望在应用稳定的情况下，能承接的流量越大越好，主要指标包括QPS(每秒处理请求数)和QPM(每分钟处理请求数)。

## 1.2 制定优化方案

明确了性能指标以后，我们就可以评估一个应用的性能好坏，同时也能发现其中的短板并对其进行优化。但是做性能优化，有几个点需要提前注意：

第一，不要反向优化。比如我们的应用整体占用内存资源较少，但是rt偏高，那我们就针对rt做优化，优化完后，rt下降了30%，但是cpu使用率上升了50%，导致一台机器负载能力下降30%，这便是反向优化。性能优化要从整体考虑，尽量在优化一个方面时，不影响其他方面，或是其他方面略微下降。

第二，不要过度优化。如果应用性能已经很好了，优化的空间很小，比如rt的tp99在2ms内，继续尝试优化可能投入产出比就很低了，不如将这些精力放在其他需要优化的地方上。

由此可见，在优化之前，明确想要优化的指标，并制定合理的优化方案是很重要的。

常见的优化方案有以下几种：

1.  优化代码

有经验的程序员在编写代码时，会时刻注意减少代码中不必要的性能消耗，比如使用strconv而不是fmt.Sprint进行数字到字符串的转化，在初始化map或slice时指定合理的容量以减少内存分配等。良好的编程习惯不仅能使应用性能良好，同时也能减少故障发生的几率。总结下来，常用的代码优化方向有以下几种：

1）提高复用性，将通用的代码抽象出来，减少重复开发。

2）池化，对象可以池化，减少内存分配；协程可以池化，避免无限制创建协程打满内存。

3）并行化，在合理创建协程数量的前提下，把互不依赖的部分并行处理，减少整体的耗时。

4）异步化，把不需要关心实时结果的请求，用异步的方式处理，不用一直等待结果返回。

5）算法优化，使用时间复杂度更低的算法。

2.  使用设计模式

设计模式是对代码组织形式的抽象和总结，代码的结构对应用的性能有着重要的影响，结构清晰，层次分明的代码不仅可读性好，扩展性高，还能避免许多潜在的性能问题，帮助开发人员快速找到性能瓶颈，进行专项优化，为服务的稳定性提供保障。常见的对性能有所提升的设计模式例如单例模式，我们可以在应用启动时将需要的外部依赖服务用单例模式先初始化，避免创建太多重复的连接。

3.  空间换时间或时间换空间

在优化的前期，可能一个小的优化就能达到很好的效果。但是优化的尽头，往往要面临抉择，鱼和熊掌不可兼得。性能优秀的应用往往是多项资源的综合利用最优。为了达到综合平衡，在某些场景下，就需要做出一些调整和牺牲，常用的方法就是空间换时间或时间换空间。比如在响应时间优先的场景下，把需要耗费大量计算时间或是网络i/o时间的中间结果缓存起来，以提升后续相似请求的响应速度，便是空间换时间的一种体现。

4.  使用更好的三方库
在我们的应用中往往会用到很多开源的第三方库，目前在github上的go开源项目就有173万+。有很多go官方库的性能表现并不佳，比如go官方的日志库性能就一般，下面是zap发布的基准测试信息（记录一条消息和10个字段的性能表现）。
从上面可以看出zap的性能比同类结构化日志包更好，也比标准库更快，那我们就可以选择更好的三方库。

# 二、性能调优-工具篇

当我们找到应用的性能短板，并针对短板制定相应优化方案，最后按照方案对代码进行优化之后，我们怎么知道优化是有效的呢？直接将代码上线，观察性能指标的变化，风险太大了。此时我们需要有好用的性能分析工具，帮助我们检验优化的效果，下面将为大家介绍几款go语言中性能分析的利器。

## 2.1 benchmark

Go语言标准库内置的 testing 测试框架提供了基准测试(benchmark)的能力，benchmark可以帮助我们评估代码的性能表现，主要方式是通过在一定时间(默认1秒)内重复运行测试代码，然后输出执行次数和内存分配结果。下面我们用一个简单的例子来验证一下，strconv是否真的比fmt.Sprint快。首先我们来编写一段基准测试的代码，如下：

```go
package main

import (
    "fmt"
    "strconv"
    "testing"
)

func BenchmarkStrconv(b *testing.B) {
    for n := 0; n < b.N; n++ {
      strconv.Itoa(n)
  }
}

func BenchmarkFmtSprint(b *testing.B) {
    for n := 0; n < b.N; n++ {
      fmt.Sprint(n)
  }
}
```

我们可以用命令行go test -bench . 来运行基准测试，输出结果如下：

```sh
goos: darwin
goarch: amd64
pkg: main
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkStrconv-12             41988014                27.41 ns/op
BenchmarkFmtSprint-12           13738172                81.19 ns/op
ok      main  7.039s
```

可以看到strconv每次执行只用了27.41纳秒，而fmt.Sprint则是81.19纳秒，strconv的性能是fmt.Sprint的三倍，那为什么strconv要更快呢？会不会是这次运行时间太短呢？为了公平起见，我们决定让他们再比赛一轮，这次我们延长比赛时间，看看结果如何。

通过go test -bench . -benchtime=5s 命令，我们可以把测试时间延长到5秒，结果如下:

```sh
goos: darwin
goarch: amd64
pkg: main
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkStrconv-12             211533207               31.60 ns/op
BenchmarkFmtSprint-12           69481287                89.58 ns/op
PASS
ok      main  18.891s
```

结果有些变化，strconv每次执行的时间上涨了4ns，但变化不大，差距仍有2.9倍。但是我们仍然不死心，我们决定让他们一次跑三轮，每轮5秒，三局两胜。

通过go test -bench . -benchtime=5s -count=3 命令，我们可以把测试进行3轮，结果如下:

```sh
goos: darwin
goarch: amd64
pkg: main
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkStrconv-12             217894554               31.76 ns/op
BenchmarkStrconv-12             217140132               31.45 ns/op
BenchmarkStrconv-12             219136828               31.79 ns/op
BenchmarkFmtSprint-12           70683580                89.53 ns/op
BenchmarkFmtSprint-12           63881758                82.51 ns/op
BenchmarkFmtSprint-12           64984329                82.04 ns/op
PASS
ok      main  54.296s
```

结果变化也不大，看来strconv是真的比fmt.Sprint快很多。那快是快，会不会内存分配上情况就相反呢？

通过go test -bench . -benchmem 这个命令我们可以看到两个方法的内存分配情况，结果如下：

```
goos: darwin
goarch: amd64
pkg: main
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkStrconv-12         43700922    27.46 ns/op    7 B/op   0 allocs/op
BenchmarkFmtSprint-12       143412      80.88 ns/op   16 B/op   2 allocs/op
PASS
ok      main  7.031s
```

可以看到strconv在内存分配上是0次，每次运行使用的内存是7字节，只是fmt.Sprint的43.8%，简直是全方面的优于fmt.Sprint啊。那究竟是为什么strconv比fmt.Sprint好这么多呢？

通过查看strconv的代码，我们发现，对于小于100的数字，strconv是直接通过digits和smallsString这两个常量进行转换的，而大于等于100的数字，则是通过不断除以100取余，然后再找到余数对应的字符串，把这些余数的结果拼起来进行转换的。

```go
const digits = "0123456789abcdefghijklmnopqrstuvwxyz"
const smallsString = "00010203040506070809" +
  "10111213141516171819" +
  "20212223242526272829" +
  "30313233343536373839" +
  "40414243444546474849" +
  "50515253545556575859" +
  "60616263646566676869" +
  "70717273747576777879" +
  "80818283848586878889" +
  "90919293949596979899"
// small returns the string for an i with 0 <= i < nSmalls.
func small(i int) string {
  if i < 10 {
    return digits[i : i+1]
  }
  return smallsString[i*2 : i*2+2]
}
func formatBits(dst []byte, u uint64, base int, neg, append_ bool) (d []byte, s string) {
    ...
    for j := 4; j > 0; j-- {
        is := us % 100 * 2
        us /= 100
        i -= 2
        a[i+1] = smallsString[is+1]
        a[i+0] = smallsString[is+0]
    }
    ...
}
```

而fmt.Sprint则是通过反射来实现这一目的的，fmt.Sprint得先判断入参的类型，在知道参数是int型后，再调用fmt.fmtInteger方法把int转换成string，这多出来的步骤肯定没有直接把int转成string来的高效。

```go
// fmtInteger formats signed and unsigned integers.
func (f *fmt) fmtInteger(u uint64, base int, isSigned bool, verb rune, digits string) {
    ...
    switch base {
  case 10:
    for u >= 10 {
      i--
      next := u / 10
      buf[i] = byte('0' + u - next*10)
      u = next
    }
    ...
}
```

benchmark还有很多实用的函数，比如ResetTimer可以重置启动时耗费的准备时间，StopTimer和StartTimer则可以暂停和启动计时，让测试结果更集中在核心逻辑上。

## 2.2 pprof

### 2.2.1 使用介绍

pprof是go语言官方提供的profile工具，支持可视化查看性能报告，功能十分强大。pprof基于定时器(10ms/次)对运行的go程序进行采样，搜集程序运行时的堆栈信息，包括CPU时间、内存分配等，最终生成性能报告。

pprof有两个标准库，使用的场景不同：

-   runtime/pprof 通过在代码中显式的增加触发和结束埋点来收集指定代码块运行时数据生成性能报告。  
-   net/http/pprof 是对runtime/pprof的二次封装，基于web服务运行，通过访问链接触发，采集服务运行时的数据生成性能报告。

runtime/pprof的使用方法如下：

```go
package main

import (
  "os"
  "runtime/pprof"
  "time"
)

func main() {
  w, _ := os.OpenFile("test_cpu", os.O_RDWR | os.O_CREATE | os.O_APPEND, 0644)
  pprof.StartCPUProfile(w)
  time.Sleep(time.Second)
  pprof.StopCPUProfile()
}
```

我们也可以使用另外一种方法，net/http/pprof：

```go
package main

import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    err := http.ListenAndServe(":6060", nil)
    if err != nil {
        panic(err)
    }
}
```

将程序run起来后，我们通过访问http://127.0.0.1:6060/debug/pprof/就可以看到如下页面：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4YzXbjFQPpJzc3icVkBrL7x1pC0nicnnCADzjzoaSAT6oMC3jGlC3nOfA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

点击profile就可以下载cpu profile文件。那我们如何查看我们的性能报告呢？  
pprof支持两种查看模式，终端和web界面，注意: 想要查看可视化界面需要提前安装graphviz。

这里我们以web界面为例，在终端内我们输入如下命令：

```
go tool pprof -http :6060 test_cpu
```

就会在浏览器里打开一个页面，内容如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4UNib5SjO1TzdBiaIxJZLnVcjUZJuyBX72pfSNfWqXfdEvFsvv7leyfrg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从界面左上方VIEW栏下，我们可以看到，pprof支持Flame Graph，dot Graph和Top等多种视图，下面我们将一一介绍如何阅读这些视图。

### 2.2.1 火焰图 Flame Graph如何阅读

首先，推荐直接阅读火焰图，在查函数耗时场景，这个比较直观；

最简单的：横条越长，资源消耗、占用越多； 

注意：每一个function 的横条虽然很长，但可能是他的下层“子调用”耗时产生的，所以一定要关注“下一层子调用”各自的耗时分布；

每个横条支持点击下钻能力，可以更详细的分析子层的耗时占比。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4S2Q1g8icN4XfktE171nt4hcMEb0r4GJp7m44ic0DjhUtWJOaqN4KDGrA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.2.2 dot Graph 图如何阅读

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4UGwgHh8jxCIYnjjO8iaOxWEfAgiarKEticNYBTswPY0zL3wVrcKJcUqmw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

-   节点颜色:
-   红色表示耗时多的节点；
-   绿色表示耗时少的节点；
-   灰色表示耗时几乎可以忽略不计（接近零）；  

-   节点字体大小 :
-   字体越大，表示占“上层函数调用”比例越大；（其实上层函数自身也有耗时，没包含在此）
-   字体越小，表示占“上层函数调用”比例越小；  

-   线条（边）粗细:
-   线条越粗，表示消耗了更多的资源；
-   反之，则越少；  

-   线条（边）颜色:
-   颜色越红，表示性能消耗占比越高；
-   颜色越绿，表示性能消耗占比越低；
-   灰色，表示性能消耗几乎可以忽略不计；  

-   虚线：表示中间有一些节点被“移除”或者忽略了；(一般是因为耗时较少所以忽略了)       
-   实线：表示节点之间直接调用  
-   内联边标记：被调用函数已经被内联到调用函数中
    （对于一些代码行比较少的函数，编译器倾向于将它们在编译期展开从而消除函数调用，这种行为就是内联。）  

### 2.2.3 TOP 表如何阅读

flat：当前函数，运行耗时（不包含内部调用其他函数的耗时）
flat%：当前函数，占用的 CPU 运行耗时总比例（不包含外部调用函数）
sum%：当前行的 flat% 与上面所有行的flat%总和。
cum：当前函数加上它内部的调用的运行总耗时（包含内部调用其他函数的耗时）
cum%：同上的 CPU 运行耗时总比例


![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD46RN4sDqJZ50qnxrDiah3MKqSic25ETI8IPTw8YBeKUCVZiaeyxwiboONRg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.3 trace

pprof已经有了对内存和CPU的分析能力，那trace工具有什么不同呢？虽然pprof的CPU分析器，可以告诉你什么函数占用了最多的CPU时间，但它并不能帮助你定位到是什么阻止了goroutine运行，或者在可用的OS线程上如何调度goroutines。这正是trace真正起作用的地方。

我们需要更多关于Go应用中各个goroutine的执行情况的更为详细的信息，可以从P（goroutine调度器概念中的processor)和G（goroutine调度器概念中的goroutine）的视角完整的看到每个P和每个G在Tracer开启期间的全部“所作所为”，对Tracer输出数据中的每个P和G的行为分析并结合详细的event数据来辅助问题诊断的。

Tracer可以帮助我们记录的详细事件包含有：

-   与goroutine调度有关的事件信息：goroutine的创建、启动和结束；goroutine在同步原语（包括mutex、channel收发操作）上的阻塞与解锁。
-   与网络有关的事件：goroutine在网络I/O上的阻塞和解锁；
-   与系统调用有关的事件：goroutine进入系统调用与从系统调用返回；
-   与垃圾回收器有关的事件：GC的开始/停止，并发标记、清扫的开始/停止。  

Tracer主要也是用于辅助诊断这三个场景下的具体问题的：
-   并行执行程度不足的问题：比如没有充分利用多核资源等；
-   因GC导致的延迟较大的问题；
-   Goroutine执行情况分析，尝试发现goroutine因各种阻塞（锁竞争、系统调用、调度、辅助GC）而导致的有效运行时间较短或延迟的问题。  

### 2.3.1 trace性能报告

打开trace性能报告，首页信息包含了多维度数据，如下图：  
  
![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4sUx08dG2K1VLMdJ0HLEtf93FURicb8WSJnrRuL81HWlicmbou5BQfrpA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

-   View trace：以图形页面的形式渲染和展示tracer的数据，这也是我们最为关注/最常用的功能
-   Goroutine analysis：以表的形式记录执行同一个函数的多个goroutine的各项trace数据
-   Network blocking profile：用pprof profile形式的调用关系图展示网络I/O阻塞的情况
-   Synchronization blocking profile：用pprof profile形式的调用关系图展示同步阻塞耗时情况
-   Syscall blocking profile：用pprof profile形式的调用关系图展示系统调用阻塞耗时情况
-   Scheduler latency profile：用pprof profile形式的调用关系图展示调度器延迟情况
-   User-defined tasks和User-defined regions：用户自定义trace的task和region
-   Minimum mutator utilization：分析GC对应用延迟和吞吐影响情况的曲线图  

通常我们最为关注的是View trace和Goroutine analysis，下面将详细说说这两项的用法。

### 2.3.2 view trace

如果Tracer跟踪时间较长,trace会将View trace按时间段进行划分，避免触碰到trace-viewer的限制：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD43ccw0sOg5cZz5ESLWpdpx292kicn6r7FEGGhiaDWkhEibXPfVIsvhqqyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4gZNleYAVfyj00npebn5EXAmxQomxs77QNfhdIDCiaiaDW2Z2ZWfs4d9g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

View trace使用快捷键来缩放时间线标尺：w键用于放大（从秒向纳秒缩放），s键用于缩小标尺（从纳秒向秒缩放）。我们同样可以通过快捷键在时间线上左右移动：s键用于左移，d键用于右移。(游戏快捷键WASD)  
  
采样状态

这个区内展示了三个指标：Goroutines、Heap和Threads，某个时间点上的这三个指标的数据是这个时间点上的状态快照采样：Goroutines：某一时间点上应用中启动的goroutine的数量，当我们点击某个时间点上的goroutines采样状态区域时（我们可以用快捷键m来准确标记出那个时间点），事件详情区会显示当前的goroutines指标采样状态：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4KtDK8hfmPiaxvurWQJicWwCQNpnqMIyPy6qs4FrcMdLu1kV1RXia6XPPA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Heap指标则显示了某个时间点上Go应用heap分配情况（包括已经分配的Allocated和下一次GC的目标值NextGC）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD45Tlu60lvDmZ6RichiaUSZ91ibULiaH22Kjo8NAnQIhGN8ib0zLaiaC2Ux8LQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Threads指标显示了某个时间点上Go应用启动的线程数量情况，事件详情区将显示处于InSyscall（整阻塞在系统调用上）和Running两个状态的线程数量情况：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4SZ2AN78HzXI04Tuj5rUIePiawTX2n1bhIPpFibVBzFDuCDFIPp5SAl8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

P视角区

这里将View trace视图中最大的一块区域称为“P视角区”。这是因为在这个区域，我们能看到Go应用中每个P（Goroutine调度概念中的P）上发生的所有事件，包括：EventProcStart、EventProcStop、EventGoStart、EventGoStop、EventGoPreempt、Goroutine辅助GC的各种事件以及Goroutine的GC阻塞(STW)、系统调用阻塞、网络阻塞以及同步原语阻塞(mutex)等事件。除了每个P上发生的事件，我们还可以看到以单独行显示的GC过程中的所有事件。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4xvJVBhQCeoO9ZsSqbr4NsfONDXeUvx3cOoxqBxCibFcjm6q3J06ITicg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

事件详情区

点选某个事件后，关于该事件的详细信息便会在这个区域显示出来，事件详情区可以看到关于该事件的详细信息：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4WicKT6twpzKcrToLcfTNrIGPlCFNYzrUUiaVDZq5ic6KWBqyp7PjrPicDg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

-   Title：事件的可读名称；
-   Start：事件的开始时间，相对于时间线上的起始时间；
-   Wall Duration：这个事件的持续时间，这里表示的是G1在P4上此次持续执行的时间；
-   Start Stack Trace：当P4开始执行G1时G1的调用栈；
-   End Stack Trace：当P4结束执行G1时G1的调用栈；从上面End Stack Trace栈顶的函数为runtime.asyncPreempt来看，该Goroutine G1是被强行抢占了，这样P4才结束了其运行；
-   Incoming flow：触发P4执行G1的事件；
-   Outgoing flow：触发G1结束在P4上执行的事件；
-   Preceding events：与G1这个goroutine相关的之前的所有的事件；
-   Follwing events：与G1这个goroutine相关的之后的所有的事件
-   All connected：与G1这个goroutine相关的所有事件。  

### 2.3.3 Goroutine analysis

Goroutine analysis提供了从G视角看Go应用执行的图景。与View trace不同，这次页面中最广阔的区域提供的G视角视图，而不再是P视角视图。在这个视图中，每个G都会对应一个单独的条带（和P视角视图一样，每个条带都有两行），通过这一条带可以按时间线看到这个G的全部执行情况。通常仅需在goroutine analysis的表格页面找出执行最快和最慢的两个goroutine，在Go视角视图中沿着时间线对它们进行对比，以试图找出执行慢的goroutine究竟出了什么问题。
 ![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4UeGIMhkV6lkRS3JS6fKPE9GQejgKouAbQzDvr4o2JJfpRZyodhrZ6w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.4 后记

虽然pprof和trace有着非常强大的profile能力，但在使用过程中，仍存在以下痛点：

-   获取性能报告麻烦：一般大家做压测，为了更接近真实环境性能态，都使用生产环境/pre环境进行。而出于安全考虑，生产环境内网一般和PC办公内网是隔离不通的，需要单独配置通路才可以获得生产环境内网的profile 文件下载到PC办公电脑中，这也有一些额外的成本；  
-   查看profile分析报告麻烦：之前大家在本地查看profile 分析报告，一般 
    go tool pprof -http=":8083" profile
     命令在本地PC开启一个web service 查看，并且需要至少安装graphviz 等库。
-   查看trace分析同样麻烦：查看go trace 的profile 信息来分析routine 锁和生命周期时，也需要类似的方式在本地PC执行命令 
    go tool trace mytrace.profile
-   分享麻烦：如果我想把自己压测的性能结果内容，分享个另一位同学，那只能把1中获取的性能报告“profile文件”通过钉钉发给被分享人。然而有时候本地profile文件比较多，一不小心就发错了，还不如截图，但是截图又没有了交互放大、缩小、下钻等能力。处处不给力！  
-   留存复盘麻烦：系统的性能分析就像一份病历，每每看到阶段性的压测报告，总结或者对照时，不禁要询问，做过了哪些优化和改造，病因病灶是什么，有没有共性，值不值得总结归纳，现在是不是又面临相似的性能问题？  

那么能不能开发一个平台工具，解决以上的这些痛点呢？目前在阿里集团内部，高德的研发同学已经通过对go官方库的定制开发，实现了go语言性能平台，解决了以上这些痛点，并在内部进行了开源。该平台已面向阿里集团，累计实现性能场景快照数万条的获取和分析，解决了很多的线上服务性能调试和优化问题，这里暂时不展开，后续有机会可以单独分享。

# 三、性能调优-技巧篇

除了前面提到的尽量用strconv而不是fmt.Sprint进行数字到字符串的转化以外，我们还将介绍一些在实际开发中经常会用到的技巧，供各位参考。

3.1 字符串拼接

拼接字符串为了书写方便快捷，最常用的两个方法是运算符 + 和 fmt.Sprintf()

运算符 + 只能简单地完成字符串之间的拼接，fmt.Sprintf() 其底层实现使用了反射，性能上会有所损耗。

从性能出发，兼顾易用可读，如果待拼接的变量不涉及类型转换且数量较少（<=5），拼接字符串推荐使用运算符 +，反之使用 fmt.Sprintf()。

3.2 提前指定容器容量

3.3 遍历 []struct{} 使用下标而不是 range

3.4 利用unsafe包避开内存copy

unsafe包提供了任何类型的指针和 unsafe.Pointer 的相互转换及uintptr 类型和 unsafe.Pointer 可以相互转换，如下图

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4dRAbb8HRebjt5unedx8TzMwMt1Cjwb4tT7WXD17TCW1RFhwicZmvmrQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

unsafe包指针转换关系

依据上述转换关系，其实除了string和[]byte的转换，也可以用于slice、map等的求长度及一些结构体的偏移量获取等，但是这种黑科技在一些情况下会带来一些匪夷所思的诡异问题，官方也不建议用，所以还是慎用，除非你确实很理解各种机制了，这里给出项目中实际用到的常规string和[]byte之间的转换，如下：

```go
func Str2bytes(s string) []byte {
   x := (*[2]uintptr)(unsafe.Pointer(&s))
   h := [3]uintptr{x[0], x[1], x[1]}
   return *(*[]byte)(unsafe.Pointer(&h))
}

func Bytes2str(b []byte) string {
   return *(*string)(unsafe.Pointer(&b))
}
```

我们通过benchmark来验证一下是否性能更优：

```go
// 推荐：用unsafe.Pointer实现string到bytes
func BenchmarkStr2bytes(b *testing.B) {
  s := "testString"
  var bs []byte
  for n := 0; n < b.N; n++ {
    bs = Str2bytes(s)
  }
  _ = bs
}
// 不推荐：用类型转换实现string到bytes
func BenchmarkStr2bytes2(b *testing.B) {
  s := "testString"
  var bs []byte
  for n := 0; n < b.N; n++ {
    bs = []byte(s)
  }
  _ = bs
}
// 推荐：用unsafe.Pointer实现bytes到string
func BenchmarkBytes2str(b *testing.B) {
  bs := Str2bytes("testString")
  var s string
  b.ResetTimer()
  for n := 0; n < b.N; n++ {
    s = Bytes2str(bs)
  }
  _ = s
}
// 不推荐：用类型转换实现bytes到string
func BenchmarkBytes2str2(b *testing.B) {
  bs := Str2bytes("testString")
  var s string
  b.ResetTimer()
  for n := 0; n < b.N; n++ {
    s = string(bs)
  }
  _ = s
}

goos: darwin
goarch: amd64
pkg: demo/test
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkStr2bytes-12           1000000000               0.2938 ns/op          0 B/op          0 allocs/op
BenchmarkStr2bytes2-12          38193139                28.39 ns/op           16 B/op          1 allocs/op
BenchmarkBytes2str-12           1000000000               0.2552 ns/op          0 B/op          0 allocs/op
BenchmarkBytes2str2-12          60836140                19.60 ns/op           16 B/op          1 allocs/op
PASS
ok      demo/test       3.301s
```

可以看到使用unsafe.Pointer比强制类型转换性能是要高不少的，从内存分配上也可以看到完全没有新的内存被分配。

## 3.5 协程池

go语言最大的特色就是很容易的创建协程，同时go语言的协程调度策略也让go程序可以最大化的利用cpu资源，减少线程切换。但是无限度的创建goroutine，仍然会带来问题。我们知道，一个go协程占用内存大小在2KB左右，无限度的创建协程除了会占用大量的内存空间，同时协程的切换也有不少开销，一次协程切换大概需要100ns，虽然相较于线程毫秒级的切换要优秀很多，但依然存在开销，而且这些协程最后还是需要GC来回收，过多的创建协程，对GC也是很大的压力。所以我们在使用协程时，可以通过协程池来限制goroutine数量，避免无限制的增长。


## 3.6 sync.Pool 对象复用

## 3.7 避免系统调用

系统调用是一个很耗时的操作，在各种语言中都是，go也不例外，在go的GPM模型中，异步系统调用G会和MP分离，同步系统调用GM会和P分离，不管何种形式除了状态切换及内核态中执行操作耗时外，调度器本身的调度也耗时。所以在可以避免系统调用的地方尽量去避免。

```go
// 推荐：不使用系统调用
func BenchmarkNoSytemcall(b *testing.B) {
   b.RunParallel(func(pb *testing.PB) {
      for pb.Next() {
         if configs.PUBLIC_KEY != nil {
         }
      }
   })
}
// 不推荐：使用系统调用
func BenchmarkSytemcall(b *testing.B) {
   b.RunParallel(func(pb *testing.PB) {
      for pb.Next() {
         if os.Getenv("PUBLIC_KEY") != "" {
         }
      }
   })
}
goos: darwin
goarch: amd64
pkg: demo/test
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkNoSytemcall-12         1000000000              0.1495 ns/op          0 B/op          0 allocs/op
BenchmarkSytemcall-12           37224988                31.10 ns/op           0 B/op          0 allocs/op
PASS
ok      demo/test       1.877s
```

# 四、性能调优-实战篇

案例1: go协程创建数据库连接不释放导致内存暴涨

应用背景

感谢@路现提供的案例。

遇到的问题及表象特征

线上机器偶尔出现内存使用率超过百分之九十报警。

分析思路及排查方向

在报警触发时，通过直接拉取线上应用的profile文件，查看内存分配情况，我们看到内存分配主要产生在本地缓存的组件上。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD467O1Te1fqRnqoJczew6pgjMv89aZWehmic4WBYdJeQkZNetsa4COskQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

但是分析代码并没有发现存在内存泄露的情况，看着像是资源一直没有被释放，进一步分析goroutine的profile文件。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4ibY7Msy3QqlXktCsljTfib1wPGm0RW3xpbPVzPlLZwD8n21eFArkXwXw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

发现存在大量的goroutine未释放，表现在本地缓存击穿后回源数据库，对数据库的查询访问一直不释放。

调优手段与效果

最终通过排查，发现使用的数据库组件存在bug，在极端情况下会出现死锁的情况，导致数据库访问请求无法返回也无法释放。最终bug修复后升级数据库组件版本解决了问题。

案例2: 优惠索引内存分配大，gc 耗时高

应用背景

感谢@梅东提供的案例。

遇到的问题及表象特征

接口tp99高，偶尔会有一些特别耗时的请求，导致用户的优惠信息展示不出来。

分析思路及排查方向

通过直接在平台上抓包观察，我们发现使用的分配索引这个方法占用的堆内存特别高，通过 top 可以看到是排在第一位的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4eAojebDsVSlW7xY6YRXkWPsOc3s6F5CSPHVNCmBUeFhve5uYHEhUcg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4oXyzrGx6o7j7EaIibviapXwPMtYwAQbD3a6kY4iaKCFJbUibI4pNzicd20A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们分析代码，可以看到，获取城市索引的地方，每次都是重新申请了内存的，通过改动为返回指针，就不需要每次都单独申请内存了，核心代码改动：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD47CEW1GB8YdObZzmvYyC3JcJsgRGxpxkGXug6P5ssWaOt0ibj21yuxqg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

调优手段与效果

修改后，上线观察，可以看到使用中的内存以及gc耗时都有了明显降低

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4q3YnyTUmnxyxmfW18ia0L4dl5SgGbEV0rvD3bJicV1g3qhibTFjewBH1g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

案例3：流量上涨导致cpu异常飙升

应用背景

感谢@君度提供的案例。

遇到的问题及表象特征

能量站v2接口和task-home-page接口流量较大时，会造成ab实验策略匹配时cpu飙升

分析思路及排查方向

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4ywmysBJhAyrStD0hg0d2b6H1R1RXvc9uxc6mMaaVTrZlWIogeDuZGw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

调优手段与效果

主要优化点如下：

1、优化toEntity方法，简化为单独的ID()方法

2、优化数组、map初始化结构

3、优化adCode转换为string过程

4、关闭过多的match log打印

优化后profile：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4xUe4n0XlTdEYZiaicnxgo5q1ABqbiax1w25Pmiakgtsc0qL1aV1jrcxNfg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

优化上线前后CPU的对比

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4Jh23pHq54n4XtKibjdrQlon1lBpO31IwXicq3FOYldp3ia3hkHqmgOdHQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

案例4：内存对象未释放导致内存泄漏

应用背景

感谢@淳深提供的案例，提供案例的服务，日常流量峰值在百万qps左右，是高德内部十分重要的服务。此前该服务是由java实现的，后来用go语言进行重构，在重构完成切全量后，有许多性能优化的优秀案例，这里选取内存泄漏相关的一个案例分享给大家，希望对大家在自己服务进行内存泄漏问题排查时能提供参考和帮助。

遇到的问题及表象特征

go语言版本全量切流后，每天会对服务各项指标进行详细review，发现每日内存上涨约0.4%，如下图

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4nKkMkGvE8TCNguas6jGEQqJWcozGvRVRWJejD4bJnfHsqGbjdurjXA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4otrEv3fNWia85dBFvsiae93jnTCicSPLsxOIM2E2xmxdodL9M86aY9Jwg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在go版本服务切全量前，从第一张图可以看到整个内存使用是平稳的，无上涨趋势，但是切go版本后，从第二张图可以看到，整个内存曲线呈上升趋势，遂认定内存泄漏，开始排查内存泄漏的“罪魁祸首”。

分析思路及排查方向

我们先到线上机器抓取当前时间的heap文件，间隔一天后再次抓取heap文件，通过pprof diff对比，我们发现time.NewTicker的内存占用增长了几十MB(由于未保留当时的heap文件，此处没有截图)，通过调用栈信息，我们找到了问题的源头，来自中间件vipserver client的SrvHost方法，通过深扒vipserver client代码，我们发现，每个vipserver域名都会有一个对应的协程，这个协程每隔三秒钟就会新建一个ticker对象，且用过的ticker对象没有stop，也就不会释放相应的内存资源。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4GUSl2yCwTHntverOgYsCWkq9PHUh64hGLBF4RIicakk3M9ibicHsVZ3zw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

而这个time.NewTicker会创建一个timer对象，这个对象会占用72字节内存。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD43bJZpKYeJPC0wSp6LZ3SdeUBeBRTERucBc9vzgWmESZKo5eIDa6DZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在服务运行一天的情况下，进过计算，该对象累计会在内存中占用约35.6MB，和上述内存每日增长0.4%正好能对上，我们就能断定这个内存泄漏来自这里。

调优手段与效果

知道是timer对象重复创建的问题后，只需要修改这部分的代码就好了，最新的vipserver client修改了此处的逻辑，如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4MpQztNTp0Dv0LAvZEyQNEwm56RygGfXiak7q0iauBrLRDMWfYq2Vibw4Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

修改完后，运行一段时间，内存运行状态平稳，已无内存泄漏问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIfHgB9U9AAjUtvaS7ooWD4RvB7awtJ4euApDTPF0TlRM8c3sbCMn6JIHxTcULpp9NjnJEZl7aMPA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

# 结语

目前go语言不仅在阿里集团内部，在整个互联网行业内也越来越流行，希望本文能为正在使用go语言的同学在性能优化方面带来一些参考价值。在阿里集团内部，高德也是最早规模化使用go语言的团队之一，目前高德线上运行的go服务已经达到近百个，整体qps已突破百万量级。在使用go语言的同时，高德也为集团内go语言生态建设做出了许多贡献，包括开发支持阿里集团常见的中间件（比如配置中心-Diamond、分布式RPC服务框架-HSF、服务发现-Vipserver、消息队列-MetaQ、流量控制-Sentinel、日志追踪-Eagleeye等）go语言版本，并被阿里中间件团队官方收录。但是go语言生态建设仍然有很长的道路要走，希望能有更多对go感兴趣的同学能够加入我们，一起参与阿里的go生态建设，乃至为互联网业界的go生态发展添砖加瓦。

# Reference
https://blog.wolfogre.com/posts/go-ppof-practice/

https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/go-goroutine-leak.md
https://mp.weixin.qq.com/s/UHaCLhiIyLYVrba-nEUONA
## 示例源码

**本文所有示例源码，及历史文章、代码都存储在Github，阅读原文可直接跳转**，Github：[https://github.com/Shitaibin/golang_step_by_step/tree/master/pprof](https://github.com/Shitaibin/golang_step_by_step/tree/master/pprof) 。