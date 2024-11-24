#golang  #note 

# golang调度器的场景
本文总结了12个主要的场景，覆盖了以下内容：

1.  G的创建和分配。
2.  P的本地队列和全局队列的负载均衡。
3.  M如何寻找G。
4.  M如何从G1切换到G2。
5.  work stealing，M如何去偷G。
6.  为何需要自旋线程。
7.  G进行系统调用，如何保证P的其他G'可以被执行，而不是饿死。
8.  Go调度器的抢占。

## 12场景

> 提示：图在前，场景描述在后。

[![](https://camo.githubusercontent.com/316c45d36035d8cec99f4ae587ba9d7d284c36ac583a598f1871569153813aa3/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313139303830393634392d343033303438392e706e67)]

> 上图中三角形、正方形、圆形分别代表了M、P、G，正方形连接的绿色长方形代表了P的本地队列。

**场景1**：p1拥有g1，m1获取p1后开始运行g1，g1使用`go func()`创建了g2，为了局部性g2优先加入到p1的本地队列。

[![](https://camo.githubusercontent.com/c634437984925ac21ea2209602808b3ead1233866ea8d93b0b434a06f31a0516/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313139303832363833382d343033303530362e706e67)]

**场景2**：**g1运行完成后(函数：`goexit`)，m上运行的goroutine切换为g0，g0负责调度时协程的切换（函数：`schedule`）**。从p1的本地队列取g2，从g0切换到g2，并开始运行g2(函数：`execute`)。实现了**线程m1的复用**。

[![](https://camo.githubusercontent.com/6a7feb43d0361798378f4a970c48fe1db6d7ad8f3888a14774cf47703b226020/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313136303731383634362d343031393633382e706e67)]

**场景3**：假设每个p的本地队列只能存4个g。g2要创建了6个g，前4个g（g3, g4, g5, g6）已经加入p1的本地队列，p1本地队列满了。

[![](https://camo.githubusercontent.com/b10ec32f295b0beb6ed9ded17a5cd14324812f95a8ba111297087599c7cfa7bb/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313136303732383032342d343031393634382e706e67)]

> 蓝色长方形代表全局队列。

**场景4**：g2在创建g7的时候，发现p1的本地队列已满，需要执行**负载均衡**，把p1中本地队列中前一半的g，还有新创建的g**转移**到全局队列（实现中并不一定是新的g，如果g是g2之后就执行的，会被保存在本地队列，利用某个老的g替换新g加入全局队列），这些g被转移到全局队列时，会被打乱顺序。所以g3,g4,g7被转移到全局队列。

[![](https://camo.githubusercontent.com/29223dd90f80c6986ac801bccb751060d7dd545ed740d107beeab7a4a7439c1e/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313136313133383335332d343031393839382e706e67)]

**场景5**：g2创建g8时，p1的本地队列未满，所以g8会被加入到p1的本地队列。

[![](https://camo.githubusercontent.com/e2c2566910234f77a3e883adddfbf214b2d1f539af0dccbc53a7f26610a2f7a1/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313136323733343833302d343032303835342e706e67)]

**场景6**：**在创建g时，运行的g会尝试唤醒其他空闲的p和m执行**。假定g2唤醒了m2，m2绑定了p2，并运行g0，但p2本地队列没有g，m2此时为自旋线程（没有G但为运行状态的线程，不断寻找g，后续场景会有介绍）。

[![](https://camo.githubusercontent.com/e19dcc653f07b02856cd81bd47af3cf44d58acaf7dc72004a4ed7cbc9db7880a/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313136323731373438362d343032303833372e706e67)]

**场景7**：m2尝试从全局队列(GQ)取一批g放到p2的本地队列（函数：`findrunnable`）。m2从全局队列取的g数量符合下面的公式：

```
n = min(len(GQ)/GOMAXPROCS + 1, len(GQ/2))
```

公式的含义是，至少从全局队列取1个g，但每次不要从全局队列移动太多的g到p本地队列，给其他p留点。这是**从全局队列到P本地队列的负载均衡**。

假定我们场景中一共有4个P，所以m2只从能从全局队列取1个g（即g3）移动p2本地队列，然后完成从g0到g3的切换，运行g3。

[![](https://camo.githubusercontent.com/62b51875acba3c106d6f95da231b644c1b8945db241b1d25c5fe379c652e36a1/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323032302d30392d676f2d7363686564756c65722d70382e706e67)]

**场景8**：假设g2一直在m1上运行，经过2轮后，m2已经把g7、g4也挪到了p2的本地队列并完成运行，全局队列和p2的本地队列都空了，如上图左边。

**全局队列已经没有g，那m就要执行work stealing：从其他有g的p哪里偷取一半g过来，放到自己的P本地队列**。p2从p1的本地队列尾部取一半的g，本例中一半则只有1个g8，放到p2的本地队列，情况如上图右边。

[![](https://camo.githubusercontent.com/265e98d2449c0902913bec2ce1a909302cc670017e7458409ae3977bee066084/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313137303131333435372d343032323837332e706e67)]

**场景9**：p1本地队列g5、g6已经被其他m偷走并运行完成，当前m1和m2分别在运行g2和g8，m3和m4没有goroutine可以运行，m3和m4处于**自旋状态**，它们不断寻找goroutine。为什么要让m3和m4自旋，自旋本质是在运行，线程在运行却没有执行g，就变成了浪费CPU？销毁线程不是更好吗？可以节约CPU资源。创建和销毁CPU都是浪费时间的，我们**希望当有新goroutine创建时，立刻能有m运行它**，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费CPU，所以系统中最多有GOMAXPROCS个自旋的线程，多余的没事做线程会让他们休眠（见函数：`notesleep()`）。

[![](https://camo.githubusercontent.com/c7eea6f0fe8d8702dbb96a1f742071e4b6147946b29f2614ec2e0a60c6fed6df/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d696d6167652d32303139303333313138323933393331382d343032383137392e706e67)]

**场景10**：假定当前除了m3和m4为自旋线程，还有m5和m6为自旋线程，g8创建了g9，g8进行了**阻塞的系统调用**，m2和p2立即解绑，p2会执行以下判断：如果p2本地队列有g、全局队列有g或有空闲的m，p2都会立马唤醒1个m和它绑定，否则p2则会加入到空闲P列表，等待m来获取可用的p。本场景中，p2本地队列有g，可以和其他自旋线程m5绑定。

**场景11**：（无图场景）g8创建了g9，假如g8进行了**非阻塞系统调用**（CGO会是这种方式，见`cgocall()`），m2和p2会解绑，但m2会记住p，然后g8和m2进入系统调用状态。当g8和m2退出系统调用时，会尝试获取p2，如果无法获取，则获取空闲的p，如果依然没有，g8会被记为可运行状态，并加入到全局队列。

**场景12**：（无图场景）Go调度在go1.12实现了抢占，应该更精确的称为**请求式抢占**，那是因为go调度器的抢占和OS的线程抢占比起来很柔和，不暴力，不会说线程时间片到了，或者更高优先级的任务到了，执行抢占调度。**go的抢占调度柔和到只给goroutine发送1个抢占请求，至于goroutine何时停下来，那就管不到了**。抢占请求需要满足2个条件中的1个：1）G进行系统调用超过20us，2）G运行超过10ms。调度器在启动的时候会启动一个单独的线程sysmon，它负责所有的监控工作，其中1项就是抢占，发现满足抢占条件的G时，就发出抢占请求。

## 场景融合

如果把上面所有的场景都融合起来，就能构成下面这幅图了，它从整体的角度描述了Go调度器各部分的关系。图的上半部分是G的创建、负债均衡和work stealing，下半部分是M不停寻找和执行G的迭代过程。

如果你看这幅图还有些似懂非懂，建议赶紧开始看[雨痕大神的Golang源码剖析](https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/github.com/qyuhen/book)，章节：并发调度。

[![](https://camo.githubusercontent.com/eccc0ec78ddba90fedc4494a6381adfe148a9e737062c83ba07fe7910a47e0ca/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d617263682e706e67)]
首先是 Processor（简称 P）， 其作用类似 CPU 核，用来控制可同时并发执行的任务数  
量。每个工作线程都必须绑定一个有效 P 才被允许执行任务，否则只能休眠，直到有空  
闲 P 时被唤醒。P 还为线程提供执行资源，比如对象分配内存、本地任务队列等。线程  
独享所绑定的 P 资源，可在无锁状态下执行高效操作。  
基本上，进程内的一切都在以 goroutine（简称 G）方式运行，包括运行时相关服务，以  
及 main.main 入口函数。需要指出，G 并非执行体，它仅仅保存并发任务状态，为任务  
执行提供所需栈内存空间。G 任务创建后被放置在 P 本地队列或全局队列，等待工作线  
程调度执行。  
实际执行体是系统线程（简称 M），它和 P 绑定，以调度循环方式不停执行 G 并发任  
务。M 通过修改寄存器，将执行栈指向 G 自带的栈内存，并在此空间内分配堆栈帧，执  
行任务函数。当需要中途切换时，只要将相关寄存器值保存回 G 空间即可维持状态，任  
何 M 都可据此恢复执行。线程仅负责执行，不再持有状态，这是并发任务跨线程调度，  
实现多路复用的根本所在。  
尽管 P/M 构成执行组合体，但两者数量并非一一对应。通常情况下，P 的数量相对恒  
定，默认与 CPU 核数量相同，但也可能更多或更少，而 M 则是由调度器按需创建的。  
举例来说，当 M 因陷入系统调用而长时间阻塞时，P 就会被监控线程抢回，去新建（或  
唤醒）一个 M 执行其他任务，这样M 的数量就会增长。  
因为 G 初始栈仅有 2 KB，且创建操作只是在用户空间简单地分配对象，远比进入内核  
态分配线程要简单得多。调度器让多个 M 进入调度循环，不停获取并执行任务，所以我  
们才能创建成千上万个并发任务。


总结，Go调度器和OS调度器相比，是相当的轻量与简单了，但它已经足以撑起goroutine的调度工作了，并且让Go具有了原生（强大）并发的能力，这是伟大的。如果你记住的不多，你一定要记住这一点：**Go调度本质是把大量的goroutine分配到少量线程上去执行，并利用多核并行，实现更强大的并发。**
![[Pasted image 20220711164801.png]]
![[Pasted image 20220711164832.png]]
[![](https://camo.githubusercontent.com/a2f479426e458a057bc4fcb1d6d2a05575a4f09344713ec2d702f586ef72797e/68747470733a2f2f6c65737369736265747465722e736974652f696d616765732f323031392d30342d7368636564756c652d666c6f772e706e67)
阅读 Go 调度器的源码，需要先从整体结构上对其有个把握，Go 程序启动后的调度器主逻辑如图 4.1 所示：

![](https://pic3.zhimg.com/80/v2-d74187a86c8c0d6e14cbb9c117908e72_720w.webp)

_图4.1 调度器主逻辑_

# 初始化
调度器初始化函数 schedinit 在前面已多次提及，除去内存分配、垃圾回收等操作外，针  
对自身的初始化无非是 MaxMcount、GOMAXPROCS。
proc1.go
```go
func schedinit() {  
	// 设置最大 M 数量  
	sched.maxmcount = 10000
	  
	// 初始化栈空间复用管理链表  
	stackinit()  
	  
	// 初始化当前 M  
	mcommoninit(_g_.m)  
	  
	// 默认值总算从 1 调整为 CPU Core 数量了  
	procs := int(ncpu)  
	if n := atoi(gogetenv("GOMAXPROCS")); n > 0 {  
		if n > _MaxGomaxprocs {  
			n = _MaxGomaxprocs  
		}  
		procs = n  
	}  
	  
	// 调整 P 数量  
	// 注意：此刻所有 P 都是新建的，所以不可能返回有本地任务的 P  
	if procresize(int32(procs)) != nil {  
		throw("unknown runnable goroutine during bootstrap")  
	}  
}
```
GOMAXPROCS 默认值总算从 1 改为 CPU Cores 了。
因为 P 的数量有最大限制，所以用一个足够大的数组存储才是最正确的做法。虽然浪费  
点空间，但省去很多内存增减的麻烦。
runtime2.go
```go
var allp [_MaxGomaxprocs + 1]*p  
  
type schedt struct {  
	pidle puintptr // 空闲 P 链表  
	npidle uint32 // 空闲 P 数量  
}
```
调整 P 的数量并不意味着全部分配新对象，仅仅做去余补缺即可。  
proc1.go
```go
func procresize(nprocs int32) *p {  
	old := gomaxprocs
	// 新增  
	for i := int32(0); i < nprocs; i++ {  
		pp := allp[i]  
		// 申请新 P 对象  
		if pp == nil {  
			pp = new(p)  
			pp.id = i  
			pp.status = _Pgcstop  
			  
			// 保存到 allp  
			atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))  
		}  
		// 为 P 分配 cache 对象  
		if pp.mcache == nil {  
			if old == 0 && i == 0 {  
				// bootstrap  
				pp.mcache = getg().m.mcache  
			} else {  
				// 创建 cache  
				pp.mcache = allocmcache()  
			}  
		}  
	}  
  
	// 释放多余的 P  
	for i := nprocs; i < old; i++ {  
		p := allp[i]  
	  
		// 将本地任务转移到全局队列  
		for p.runqhead != p.runqtail {  
			p.runqtail--  
			gp := p.runq[p.runqtail%uint32(len(p.runq))]  
			globrunqputhead(gp)  
		}  
		if p.runnext != 0 {  
			globrunqputhead(p.runnext.ptr())  
			p.runnext = 0  
		}  
  
		// 释放当前 P 绑定的 cache  
		freemcache(p.mcache)
		p.mcache = nil  
  
		// 将当前 P 的 G 复用链转移到全局  
		gfpurge(p)  
		  
		// 似乎就丢在那里不管了，反正也没剩下啥  
		p.status = _Pdead  
		// can't free P itself because it can be referenced by an M in syscall  
	}  
  
	_g_ := getg()  
  
	// 如果当前正在用的 P 属于被释放的那拨，那就换成 allp[0]  
	// 调度器初始化阶段，根本没有 P，那就绑定 allp[0]  
	if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {  
		// 继续使用当前 P  
		_g_.m.p.ptr().status = _Prunning  
	} else {  
		// 释放当前 P，因为它已经失效  
		if _g_.m.p != 0 {  
			_g_.m.p.ptr().m = 0  
		}  
		_g_.m.p = 0  
		_g_.m.mcache = nil  
  
		// 换成 allp[0]  
		p := allp[0]  
		p.m = 0  
		p.status = _Pidle  
		acquirep(p)  
	}  
  
	// 将没有本地任务的 P 放到空闲链表  
	var runnablePs *p  
	for i := nprocs - 1; i >= 0; i-- {  
		p := allp[i]  
		  
		// 确保不是当前正在用的 P  
		if _g_.m.p.ptr() == p {  
			continue  
		}  
	  
		p.status = _Pidle
		if runqempty(p) {  
			// 放入空闲链表  
			pidleput(p)  
		} else {  
			// 有本地任务，构建链表  
			p.m.set(mget())  
			p.link.set(runnablePs)  
			runnablePs = p  
		}  
	}  
  
	// 返回有本地任务的 P（链表）  
	return runnablePs  
}  
  
// 将 P 放入空闲链表  
func pidleput(_p_ *p) {  
	_p_.link = sched.pidle  
	sched.pidle.set(_p_)  
	xadd(&sched.npidle, 1)  
}
```
`默认只有 schedinit 和 startTheWorld 会调用 procresize 函数`。在调度器初始化阶段，所有  
P 对象都是新建的。除分配给当前主线程的外，其他都被放入空闲链表。而  
startTheWorld 会激活全部有本地任务的 P 对象（详见后文）。  

在完成调度器初始化后，引导过程才创建并运行 main goroutine。  

asm_amd64.s
```s
TEXT runtime·rt0_go(SB),NOSPLIT,$0  
	// save m->g0 = g0  
	MOVQ CX, m_g0(AX)  //初始化g0
	// save m0 to g0->m  
	MOVQ AX, g_m(CX)  //初始化m0
	  
	CALL runtime·schedinit(SB)   //调度器初始化
	  
	// 创建 main goroutine，并将其放入当前 P 本地队列  
	MOVQ $runtime·mainPC(SB), AX  
	PUSHQ AX  
	PUSHQ $0  
	CALL runtime·newproc(SB)
	POPQ AX  
	POPQ AX  
	  
	// 让当前 M0 进入调度，执行 main goroutine  
	CALL runtime·mstart(SB)  
	  
	// M0 永远不会执行这条崩溃测试指令  
	MOVL $0xf1, 0xf1 // crash  
	RET
```
		虽然可在运行期用 runtime.GOMAXPROCS 函数修改 P 的数量，但须付出极大代价。  
debug.go
```go
func GOMAXPROCS(n int) int {  
	if n > _MaxGomaxprocs {  
		n = _MaxGomaxprocs  
	}  
	  
	// 返回当前值（这个才是最常用的做法）  
	ret := int(gomaxprocs)  
	if n <= 0 || n == ret {  
		return ret  
	}  
	  
	// STW !!!  
	stopTheWorld("GOMAXPROCS")  
	  
	newprocs = int32(n)  
	  
	// 调用 procresize，并激活有任务的 P  
	startTheWorld()  
	  
	return ret  
}
```
# 任务
我们已经知道编译器会将“go func(...)”语句翻译成 newproc 调用，但这中间究竟有什  
么不为人知的秘密？
test.go
```go
package main  
  
import ()  
  
func add(x, y int) int {  
	z := x + y  
	return z  
}  
  
func main() {  
	x := 0x100  
	y := 0x200  
	go add(x, y)  
}
```
尽管这个示例有些简陋，但这不重要，重要的是编译器要做什么。
```bash
$ go build -o test test.go  
  
$ go tool objdump -s "main\.main" test  
  
TEXT main.main(SB) test.go  
	test.go:10 SUBQ $0x28, SP  
	test.go:11 MOVQ $0x100, CX  
	test.go:12 MOVQ $0x200, AX  
	test.go:13 MOVQ CX, 0x10(SP) // 实参 x 入栈  
	test.go:13 MOVQ AX, 0x18(SP) // 实参 y 入栈  
	test.go:13 MOVL $0x18, 0(SP) // 参数长度入栈  
	test.go:13 LEAQ 0x879ff(IP), AX // 将函数 add 地址存入 AX 寄存器  
	test.go:13 MOVQ AX, 0x8(SP) // 地址入栈  
	test.go:13 CALL runtime.newproc(SP)  
	test.go:14 ADDQ $0x28, SP  
	test.go:14 RET
```
从反汇编代码可以看出，Go 采用了类似 C/cdecl 的调用约定。由调用方负责提供参数空  
间，并从右往左入栈。  
proc1.go
```go
func newproc(siz int32, fn *funcval) {  
	// 获取第一参数地址  
	argp := add(unsafe.Pointer(&fn), ptrSize)
	  
	// 获取调用方 PC/IP 寄存器值  
	pc := getcallerpc(unsafe.Pointer(&siz))  
	  
	// 用 g0 栈创建 G/goroutine 对象  
	systemstack(func() {  
		newproc1(fn, (*uint8)(argp), siz, 0, pc)  
	})  
}
```
目标函数 newproc 只有两个参数，但 main 却向栈压入了四个值。按照顺序，后三个值  
应该会被合并成 funcval。还有，add 返回值会被忽略。  
runtime2.go  
```go
type funcval struct {  
	fn uintptr  
	// variable-size, fn-specific data here  
}  
```
果然是变长结构类型（目标函数参数不定），此处其补全状态应该是：  
```go
type struct {  
	fn uintptr  
	x int  
	y int  
}  
```
如此一来，关于“go 语句会复制参数值”的规则就很好理解了。站在 newproc 角度，我  
们可以画出执行栈的状态示意图。  
![[Pasted image 20220712121443.png]]
用“fn + ptrsize”跳过 add 获得第一个参数 x 的地址，getcallerpc 用“siz - 8”读取
CALL 指令压入的 main PC/IP 寄存器值，这就是 newproc 为 newproc1 准备的相关参数  
值。  
asm_amd64.s  
```bash
TEXT runtime·getcallerpc(SB),NOSPLIT,$8-16  
	MOVQ argp+0(FP),AX                     // addr of first arg  
	MOVQ -8(AX),AX                         // get calling pc  
	CMPQ AX, runtime·stackBarrierPC(SB)  
	JNE nobar  
	CALL runtime·nextBarrierPC(SB)         // Get original return PC.  
	MOVQ 0(SP), AX  
	nobar:  
	MOVQ AX, ret+8(FP)  
	RET  
```
至此，我们大概知道 go 语句编译后的真实模样。接下来，就转到 newproc1 看看如何创  
建并发任务单元 G。
runtime2.go  
```go
type g struct {  
	gopc uintptr // 调用者 PC/IP  
	startpc uintptr // 任务函数  
 stack       stack       // 描述了当前 Goroutine 的栈内存范围 [stack.lo, stack.hi) 执行栈
 stackguard0 uintptr     // 用于调度器抢占式调度
 _panic      *_panic     // 最内侧的 panic 结构体
 _defer      *_defer     // 最内侧的 defer 延迟函数结构体
 m           *m          // 当前 G 占用的线程，可能为空
 sched       gobuf       //  存储 G 的调度相关的数据 用于保存执行现场
 atomicstatus uint32     // G 的状态
 goid         int64      //  G 的 ID 唯一序号
 waitreason   waitReason //当状态status==Gwaiting时等待的原因
 preempt       bool      // 抢占信号
 preemptStop   bool      // 抢占时将状态修改成 `_Gpreempted`
 preemptShrink bool      // 在同步安全点收缩栈
 lockedm        muintptr   //G 被锁定只能在这个 m 上运行
 waiting        *sudog     // 这个 g 当前正在阻塞的 sudog 结构体
 ......
}  
```
```go
type gobuf struct {
 sp   uintptr      // 栈指针
 pc   uintptr      // 程序计数器，记录G要执行的下一条指令位置
 g    guintptr     // 持有 runtime.gobuf 的 G
 ret  uintptr      // 系统调用的返回值
 ......
}
```
这些字段会在调度器将当前 G 切换离开 M 和调度进入 M 执行程序时用到，栈指针 sp 和程序计数器 pc 用来存放或恢复寄存器中的值，改变程序执行的指令。

结构体 runtime.g 的 atomicstatus 字段存储了当前 G 的状态，G 可能处于以下状态：
```go
const (
 // _Gidle 表示 G 刚刚被分配并且还没有被初始化
 _Gidle = iota // 0
 // _Grunnable 表示 G  没有执行代码，没有栈的所有权，存储在运行队列中
 _Grunnable // 1
 // _Grunning 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P
 _Grunning // 2
 // _Gsyscall 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上
 _Gsyscall // 3
 // _Gwaiting 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上
 _Gwaiting // 4
 // _Gdead 没有被使用，没有执行代码，可能有分配的栈
 _Gdead // 6
 // _Gcopystack 栈正在被拷贝，没有执行代码，不在运行队列上
 _Gcopystack // 8
 // _Gpreempted 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒
 _Gpreempted // 9
 // _Gscan GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在
 _Gscan          = 0x1000
 ......
)
```
图3.1描述了G从创建到结束的生命周期中经历的各种状态变化过程：

![](https://pic4.zhimg.com/80/v2-c029b3b5e000b4b972d37adef1ce1e47_720w.webp)
_图3.1 G的状态变化_
虽然 G 在运行时中定义的状态较多且复杂，但是我们可以将这些不同的状态聚合成三种：等待中、可运行、运行中，分别由_Gwaiting、\_Grunnable、\_Grunning 三种状态表示，运行期间大部分情况是在这三种状态来回切换：

​ 等待中：G 正在等待某些条件满足，例如：系统调用结束等，包括 \_Gwaiting、\_Gsyscall 几个状态； ​ 可运行：G 已经准备就绪，可以在线程 M 上运行，如果当前程序中有非常多的 G，每个 G 就可能会等待更多的时间，即 \_Grunnable； ​ 运行中：G 正在某个线程 M 上运行，即 \_Grunning。

proc1.go  
```go
func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g {  
	_g_ := getg()  
	  
	//“参数 + 返回值”所需空间（对齐）  
	siz := narg + nret  
	siz = (siz + 7) &^ 7  
	  
	// 从当前 P 复用链表获取空闲的 G 对象  
	_p_ := _g_.m.p.ptr()  
	newg := gfget(_p_)  
	  
	// 获取失败，新建
	if newg == nil {  
		newg = malg(_StackMin)  
		casgstatus(newg, _Gidle, _Gdead)  
		allgadd(newg)  
	}  
	// 测试 G stack  
	if newg.stack.hi == 0 {  
		throw("newproc1: newg missing stack")  
	}  
	// 测试 G status  
	if readgstatus(newg) != _Gdead {  
		throw("newproc1: new g is not Gdead")  
	}  
	// 计算所需空间大小，并对齐  
	totalSize := 4*regSize + uintptr(siz)  
	totalSize += -totalSize & (spAlign - 1)  
  
	// 确定 SP 和参数入栈位置  
	sp := newg.stack.hi - totalSize  
	spArg := sp  
	  
	// 将执行参数拷贝入栈  
	memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))  
	  
	// 初始化用于保存执行现场的区域  
	memclr(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))  
	newg.sched.sp = sp  
	newg.sched.pc = funcPC(goexit) + _PCQuantum  
	newg.sched.g = guintptr(unsafe.Pointer(newg))  
	gostartcallfn(&newg.sched, fn)  
	  
	// 初始化基本状态  
	newg.gopc = callerpc  
	newg.startpc = fn.fn  
	casgstatus(newg, _Gdead, _Grunnable)  
	  
	// 设置唯一 id  
	if _p_.goidcache == _p_.goidcacheend {  
		// sched.goidgen 是一个全局计数器  
		// 每次取回一段有效区间，然后在该区间分配，避免频繁地去全局操作
		// [sched.goidgen+1, sched.goidgen+GoidCacheBatch]  
		_p_.goidcache = xadd64(&sched.goidgen, _GoidCacheBatch)  
		_p_.goidcache -= _GoidCacheBatch - 1  
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch  
	}  
	newg.goid = int64(_p_.goidcache)  
	_p_.goidcache++  
	  
	// 将 G 放入待运行队列  
	runqput(_p_, newg, true)  
	  
	// 如果有其他空闲 P，则尝试唤醒某个 M 出来执行任务  
	// 如果有 M 处于自旋等待 P 或 G 状态，放弃  
	// 如果当前创建的是 main goroutine (runtime.main)，那么还没有其他任务需要执行，放弃  
	if atomicload(&sched.npidle) != 0 &&  
	atomicload(&sched.nmspinning) == 0 &&  
	unsafe.Pointer(fn.fn) != unsafe.Pointer(funcPC(main)) {  
		wakep()  
	}  
	  
	return newg  
}
```
整个创建过程中，有一系列问题需要分开细说。  
首先，G 对象默认会复用，这看上去有点像 cache/object 做法。除 P 本地的复用链表  
外，还有全局链表在多个 P 之间共享。  
runtime2.go  
```go
type p struct {  
	gfree *g  
	gfreecnt int32  
	status      uint32      // p 的状态 pidle/prunning/...
 schedtick   uint32      // 每次执行调度器调度 +1
 syscalltick uint32      // 每次执行系统调用 +1
 m           muintptr    // 关联的 m 
 mcache      *mcache     // 用于 P 所在的线程 M 的内存分配的 mcache
 deferpool    []*_defer  // 本地 P 队列的 defer 结构体池
 // 可运行的 Goroutine 队列，可无锁访问
 runqhead uint32
 runqtail uint32
 runq     [256]guintptr
 // 线程下一个需要执行的 G
 runnext guintptr
 // 空闲的 G 队列，G 状态 status 为 _Gdead，可重新初始化使用
 gFree struct {
  gList
  n int32
 }
        ......
}  
  
type schedt struct {  
	gfree *g  
	ngfree int32  
	lock mutex            // schedt的锁
 midle        muintptr // 空闲的M列表
 nmidle       int32    // 空闲的M列表的数量
 nmidlelocked int32    // 被锁定正在工作的M数
 mnext        int64    // 下一个被创建的 M 的 ID
 maxmcount    int32    // 能拥有的最大数量的 M
 pidle      puintptr   // 空闲的 P 链表
 npidle     uint32     // 空闲 P 数量
 nmspinning uint32     // 处于自旋状态的 M 的数量
 // 全局可执行的 G 列表
 runq     gQueue
 runqsize int32        // 全局可执行 G 列表的大小
 // 全局 _Gdead 状态的空闲 G 列表
 gFree struct {
  lock    mutex
  stack   gList // Gs with stacks
  noStack gList // Gs without stacks
  n       int32
 }
 // sudog结构的集中存储
 sudoglock  mutex
 sudogcache *sudog
 // 有效的 defer 结构池
 deferlock mutex
 deferpool *_defer
        ......
}  
```
除了上面的四个结构体，还有一些全局变量：

```go
// src/runtime/runtime2.go
var (
 allm       *m         // 所有的 M
 gomaxprocs int32      // P 的个数，默认为 ncpu 核数
 ncpu       int32
 ......
 sched      schedt     // schedt 全局结构体
 newprocs   int32

 allpLock mutex       // 全局 P 队列的锁
 allp []*p            // 全局 P 队列，个数为 gomaxprocs
        ......
}
```

此外，src/runtime/proc.go 文件有两个全局变量：

```go
// src/runtime/proc.go 
var (
 m0           m       //  进程启动后的初始线程
 g0            g      //  代表着初始线程的stack
 ......
)
```
最主要的数据结构是 status 表示 P 的不同的状态，而 runqhead、runqtail 和 runq 三个字段表示处理器持有的运行队列，是一个长度为256的环形队列，其中存储着待执行的 G 列表，runnext 中是线程下一个需要执行的 G；gFree 存储 P 本地的状态为_Gdead 的空闲的 G，可重新初始化使用。

P 结构体中的状态 status 字段会是以下五种中的一种：

​ \_Pidle：P 没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空；
​ \_Prunning：被线程 M 持有，并且正在执行用户代码或者调度器；
​ \_Psyscall：没有执行用户代码，当前线程陷入系统调用；
​ \_Pgcstop：被线程 M 持有，当前处理器由于垃圾回收被停止；
​ \_Pdead：当前 P 已经不被使用；
P 的五种状态之间的转化关系如图 3.2 所示：

![](https://pic2.zhimg.com/80/v2-1760ebb88afba084731e4cf8b182f319_720w.webp)

_图3.2 P的状态变化_
![[Pasted image 20230321164420.png]]

proc1.go  
```go
func gfget(_p_ *p) *g {  
retry:
		// 从 P 本地队列提取复用对象  
	gp := _p_.gfree  
	  
	// 如果提取失败，尝试从全局链表转移一批到 P 本地  
	if gp == nil && sched.gfree != nil {  
		// 最多转移 32 个  
		for _p_.gfreecnt < 32 && sched.gfree != nil {  
			_p_.gfreecnt++  
			gp = sched.gfree  
			sched.gfree = gp.schedlink.ptr()  
			sched.ngfree--  
			gp.schedlink.set(_p_.gfree)  
			_p_.gfree = gp  
		}  
		  
		// 再试  
		goto retry  
	}  
	  
	// 如果成功获取复用对象  
	if gp != nil {  
		// 调整 P 复用链表  
		_p_.gfree = gp.schedlink.ptr()  
		_p_.gfreecnt--  
		  
		// 检查 G stack  
		if gp.stack.lo == 0 {  
			// 分配新栈  
			systemstack(func() {  
			gp.stack, gp.stkbar = stackalloc(_FixedStack)  
			})  
			gp.stackguard0 = gp.stack.lo + _StackGuard  
			gp.stackAlloc = _FixedStack  
		} else {  
		}  
	}  
	  
	return gp  
}
```
而当 goroutine 执行完毕，调度器相关函数会将 G 对象放回 P 复用链表。
proc1.go  
```go
func gfput(_p_ *p, gp *g) {  
	// 如果栈发生过扩张，则释放  
	stksize := gp.stackAlloc  
	if stksize != _FixedStack {  
		// non-standard stack size - free it.  
		stackfree(gp.stack, gp.stackAlloc)  
		gp.stack.lo = 0  
		gp.stack.hi = 0  
		gp.stackguard0 = 0  
		gp.stkbar = nil  
		gp.stkbarPos = 0  
	} else {  
		// Reset stack barriers.  
		gp.stkbar = gp.stkbar[:0]  
		gp.stkbarPos = 0  
	}  
	  
	// 放回 P 本地复用链表  
	gp.schedlink.set(_p_.gfree)  
	_p_.gfree = gp  
	_p_.gfreecnt++  
	  
	// 如果本地复用对象过多，则转移一批到全局链表  
	if _p_.gfreecnt >= 64 {  
		// 本地仅保留 32 个  
		for _p_.gfreecnt >= 32 {  
			_p_.gfreecnt--  
			gp = _p_.gfree  
			_p_.gfree = gp.schedlink.ptr()  
			gp.schedlink.set(sched.gfree)  
			sched.gfree = gp  
			sched.ngfree++  
		}  
	}  
}  
```
最初，G 对象都是由 malg 创建的。  
stack2.go  
```go
_StackMin = 2048
```
proc1.go  
```go
func malg(stacksize int32) *g {  
	newg := new(g)  
	if stacksize >= 0 {  
		stacksize = round2(_StackSystem + stacksize)  
		systemstack(func() {  
			newg.stack, newg.stkbar = stackalloc(uint32(stacksize))  
		})  
		newg.stackguard0 = newg.stack.lo + _StackGuard  
		newg.stackguard1 = ^uintptr(0)  
		newg.stackAlloc = uintptr(stacksize)  
	}  
	return newg  
}  
```
默认采用 2KB 栈空间，并且都被 allg 引用。这是垃圾回收遍历扫描的需要，以便获取  
指针引用，收缩栈空间。  
proc1.go  
```go
var (  
	allg **g  
	allglen uintptr  
	allgs []*g  
)  
  
func allgadd(gp *g) {  
	allgs = append(allgs, gp)  
	allg = &allgs[0]  
	allglen = uintptr(len(allgs))  
}  
```
>现在我们知道 G 的由来，以及复用方式。只是有个小问题，G 似乎从来不被释放，会不会有存  
留过多的问题？不过好在垃圾回收会调用 shrinkstack 将其栈空间回收。有关栈的相关细节，留待后文再说。

在获取 G 对象后，newproc1 会进行一系列初始化操作，毕竟不管新建还是复用，这些  
参数都必须正确地设置。同时，相关执行参数会被拷贝到 G 的栈空间，因为它和当前任  
务不再有任何关系，各自使用独立的栈空间。毕竟，“go func(...)”语句仅创建并发任  
务，当前流程会继续自己的逻辑。
创建完毕的 G 任务被优先放入 P 本地队列等待执行，这属于无锁操作。
proc1.go  
```go
func runqput(_p_ *p, gp *g, next bool) {  
	if randomizeScheduler && next && fastrand1()%2 == 0 {  
		next = false  
	}  
  
	// 如果可能，将 G 直接保存在 P.runnext，作为下一个优先执行任务  
	if next {  
	retryNext:  
			oldnext := _p_.runnext  
			if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {  
				goto retryNext  
			}  
			if oldnext == 0 {  
				return  
			}  
		  
			// 原本的 next G 会被放回本地队列  
			gp = oldnext.ptr()  
	}  
  
	retry:  
		// runqhead 是一个数组实现的循环队列  
		// head、tail 累加，通过取模即可获得索引位置，很典型的算法  
		h := atomicload(&_p_.runqhead)  
		t := _p_.runqtail  
		  
		// 如果本地队列未满，直接放到尾部  
		if t-h < uint32(len(_p_.runq)) {  
			_p_.runq[t%uint32(len(_p_.runq))] = gp  
			atomicstore(&_p_.runqtail, t+1)  
			return  
		}  
		  
		// 放入全局队列  
		// 因为需要加锁，所以 slow  
		if runqputslow(_p_, gp, h, t) {  
			return  
		}
		goto retry  
}
```
任务队列分为三级，按优先级从高到低分别是 P.runnext、P.runq、Sched.runq，很有些  
CPU 多级缓存的意思。  
runtime2.go  
```go
type schedt struct {  
	runqhead guintptr  
	runqtail guintptr  
	runqsize int32  
}  
  
type p struct {  
	runqhead uint32  
	runqtail uint32  
	runq [256]*g // 本地队列，访问时无须加锁  
	runnext guintptr // 优先执行  
}  
  
type g struct {  
	schedlink guintptr // 链表  
}  
```
往全局队列添加任务，显然需要加锁，只是专门取名为 runqputslow 就很有说法了。去  
看看到底怎么个慢法。  
proc1.go  
```go
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {  
	// 这意思显然是要从 P 本地转移一半任务到全局队列  
	// "+1" 是别忘了当前这个 gp  
	var batch [len(_p_.runq)/2 + 1]*g  
	  
	// 计算一半的实际数量  
	n := t - h  
	n = n / 2  
  
	// 从队列头部提取  
	for i := uint32(0); i < n; i++ {  
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))]  
	}
	// 调整 P 队列头部位置  
	if !cas(&_p_.runqhead, h, h+n) {  
		return false  
	}  
  
	// 加上当前 gp 这家伙  
	batch[n] = gp  
	  
	// 对顺序进行洗牌  
	if randomizeScheduler {  
		for i := uint32(1); i <= n; i++ {  
			j := fastrand1() % (i + 1)  
			batch[i], batch[j] = batch[j], batch[i]  
		}  
	}  
  
	// 串成链表  
	for i := uint32(0); i < n; i++ {  
		batch[i].schedlink.set(batch[i+1])  
	}  
	  
	// 添加到全局队列尾部  
	globrunqputbatch(batch[0], batch[n], int32(n+1))  
	return true  
}  
  
func globrunqputbatch(ghead *g, gtail *g, n int32) {  
	gtail.schedlink = 0  
	if sched.runqtail != 0 {  
		sched.runqtail.ptr().schedlink.set(ghead)  
	} else {  
		sched.runqhead.set(ghead)  
	}  
	sched.runqtail.set(gtail)  
	sched.runqsize += n  
}
```
若本地队列已满，一次性转移半数到全局队列。这个好理解，因为其他 P 可能正饿着  
呢。这也正好解释了 newproc1 最后尝试用 wakep 唤醒其他 M/P 去执行任务的意图，毕  
竟充分发挥多核优势才是正途。
最后标记一下 G 的状态切换过程。
![[Pasted image 20220712142210.png]]

# 线程
当 newproc1 成功创建 G 任务后，会尝试用 wakep 唤醒 M 执行任务。  
proc1.go  
```go
func wakep() {  
	// 被唤醒的线程需要绑定 P，累加自旋计数，避免 newproc1 唤醒过多线程  
	if !cas(&sched.nmspinning, 0, 1) {  
		return  
	}  
	startm(nil, true)  
}  
  
func startm(_p_ *p, spinning bool) {  
	// 如果没有指定 P，尝试获取空闲 P  
	if _p_ == nil {  
		_p_ = pidleget()  
		  
		// 获取失败，终止  
		if _p_ == nil {  
			// 递减自旋计数  
			if spinning {  
				xadd(&sched.nmspinning, -1)  
			}  
			return  
		}  
	}  
	  
	// 获取休眠的闲置 M  
	mp := mget()  
	  
	// 如没有闲置 M，新建  
	if mp == nil {
		// 默认启动函数  
		// 主要是判断 M.nextp 是否有暂存的 P，以此调整自旋计数  
		var fn func()  
		if spinning {  
			fn = mspinning  
		}  
		newm(fn, _p_)  
		return  
	}  
	  
	// 设置自旋状态和暂存 P  
	mp.spinning = spinning  
	mp.nextp.set(_p_)  
	  
	// 唤醒 M  
	notewakeup(&mp.park)  
}
```
>notewakeup/notesleep 实现细节参见后文。

和前文 G 对象复用类似，这个过程同样有闲置获取和新建两种方式。先不去理会闲置列  
表，看看 M 究竟如何创建，如何包装系统线程。  
runtime2.go  
```go
type m struct {  
	mstartfn func()   // 启动函数  
	curg *g           // 当前运行 G  
	park note         // 休眠锁  
	schedlink muintptr// 链表  
 g0            *g          // 提供系统栈空间 持有调度栈的 G
 gsignal       *g                // 处理 signal 的 g
 tls           [tlsSlots]uintptr // 线程本地存储
        mstartfn      func()      // M的起始函数，go语句携带的那个函数
 curg          *g          // 在当前线程上运行的 G
 p             puintptr    // 执行 go 代码时持有的 p (如果没有执行则为 nil)
 nextp         puintptr    // 用于暂存与当前 M 有潜在关联的 P
 oldp          puintptr    // 执行系统调用前绑定的 P
 spinning      bool        // 表示当前 M 是否正在寻找 G，在寻找过程中 M 处于自旋状态
 lockedg       guintptr    // 表示与当前 M 锁定的那个 G
 .....
}  
```
M 并没有像 G 和 P 一样的状态标记, 但可以认为一个 M 有以下的状态:

​ **自旋中(spinning)**: M 正在从运行队列获取 G, 这时候 M 会拥有一个 P；

​ **执行go代码中**: M 正在执行go代码, 这时候 M 会拥有一个 P；

​ **执行原生代码中**: M 正在执行原生代码或者阻塞的syscall, 这时M并不拥有P；

​ **休眠中**: M 发现无待运行的 G 时会进入休眠, 并添加到空闲 M 链表中, 这时 M 并不拥有 P。

proc1.go  
```go
func newm(fn func(), _p_ *p) {  
	// 创建 M 对象  
	mp := allocm(_p_, fn)  
	  
	// 暂存 P
	mp.nextp.set(_p_)  
	  
	// 创建系统线程  
	newosproc(mp, unsafe.Pointer(mp.g0.stack.hi))  
}  
  
func allocm(_p_ *p, fn func()) *m {  
	mp := new(m)  
	mp.mstartfn = fn // 启动函数  
	mcommoninit(mp) // 初始化  
	  
	// 创建 g0  
	// In case of cgo or Solaris, pthread_create will make us a stack.  
	// Windows and Plan 9 will layout sched stack on OS stack.  
	if iscgo || GOOS == "solaris" || GOOS == "windows" || GOOS == "plan9" {  
			mp.g0 = malg(-1)  
	} else {  
			mp.g0 = malg(8192 * stackGuardMultiplier)  
	}  
	mp.g0.m = mp  
	  
	return mp  
}
```
M 最特别的就是自带一个名为 g0，默认 8 KB 栈内存的 G 对象属性。它的栈内存地址被  
传给 newosproc 函数，作为系统线程的默认堆栈空间（并非所有系统都支持）。  
os1_linux.go  
```go
const cloneFlags = _CLONE_VM | /* share memory */  
					_CLONE_FS | /* share cwd, etc */  
					_CLONE_FILES | /* share fd table */  
					_CLONE_SIGHAND | /* share sig handler table */  
					_CLONE_THREAD /* revisit - okay for now */  
  
func newosproc(mp *m, stk unsafe.Pointer) {  
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0),  
				unsafe.Pointer(funcPC(mstart)))  
}
```
>关于系统调用 clone 的更多信息，请参考 man 2 手册。

os1_windows.go  
```go
func newosproc(mp *m, stk unsafe.Pointer) {  
	const _STACK_SIZE_PARAM_IS_A_RESERVATION = 0x00010000  
	thandle := stdcall6(_CreateThread, 0, 0x20000,  
				funcPC(tstart_stdcall), uintptr(unsafe.Pointer(mp)),  
				_STACK_SIZE_PARAM_IS_A_RESERVATION, 0)  
	if thandle == 0 {  
		print("runtime: failed to create new OS thread (have ",  
		mcount(), " already; errno=", getlasterror(), ")\n")  
		throw("runtime.newosproc")  
	}  
}  
```
>Windows API CreateThread 不支持自定义线程堆栈。  

在进程执行过程中，有两类代码需要运行。一类自然是用户逻辑，直接使用 G 栈内存；  
另一类是运行时管理指令，它并不便于直接在用户栈上执行，因为这需要处理与用户逻  
辑现场有关的一大堆事务。  
举例来说，G 任务可在中途暂停，放回队列后由其他 M 获取执行。如果不更改执行栈，  
可能会造成多个线程共享内存，从而引发混乱。另外，在执行垃圾回收操作时，如何收  
缩依旧被线程持有的 G 栈空间？因此，当需要执行管理指令时，会将线程栈临时切换到  
g0，与用户逻辑彻底隔离。  
其实，在前文就经常看到 systemstack 这种执行方式，它就是切换到 g0 栈后再执行运行  
时相关管理操作的。  
proc1.go  
```go
func newproc(siz int32, fn *funcval) {  
	systemstack(func() {  
		newproc1(fn, (*uint8)(argp), siz, 0, pc)  
	})  
}  
```
asm_amd64.s  
```go
TEXT runtime·systemstack(SB), NOSPLIT, $0-8  
	MOVQ fn+0(FP), DI // DI = fn  
	MOVQ g(CX), AX // AX = g  
	MOVQ g_m(AX), BX // BX = m
	MOVQ m_g0(BX), DX // DX = g0  
	CMPQ AX, DX // 如果当前g 已经是 g0，那么无须切换  
	JEQ  noswitch  
	  
	MOVQ m_curg(BX), R8 // 当前 g  
	CMPQ AX, R8 // 如果是用户逻辑 g，切换  
	JEQ  switch  
	  
	// Bad: g is not gsignal, not g0, not curg. What is it?  
	MOVQ $runtime·badsystemstack(SB), AX  
	CALL AX  
  
switch:  
	// 将 G 状态保存到 sched  
	MOVQ $runtime·systemstack_switch(SB), SI  
	MOVQ SI, (g_sched+gobuf_pc)(AX)  
	MOVQ SP, (g_sched+gobuf_sp)(AX)  
	MOVQ AX, (g_sched+gobuf_g)(AX)  
	MOVQ BP, (g_sched+gobuf_bp)(AX)  
	  
	// 切换到 g0.stack  
	MOVQ DX, g(CX) // DX = g0  
	MOVQ (g_sched+gobuf_sp)(DX), BX // 从 g0.sched 获取 SP  
	SUBQ $8, BX // 调整 SP  
	MOVQ $runtime·mstart(SB), DX  
	MOVQ DX, 0(BX)  
	MOVQ BX, SP // 通过调整 SP 寄存器值来切换栈内存  
	  
	// 执行系统管理函数  
	MOVQ DI, DX // DI = fn  
	MOVQ 0(DI), DI  
	CALL DI  
	  
	// 切换回 G，恢复执行现场  
	MOVQ g(CX), AX  
	MOVQ g_m(AX), BX  
	MOVQ m_curg(BX), AX  
	MOVQ AX, g(CX)  
	MOVQ (g_sched+gobuf_sp)(AX), SP  
	MOVQ $0, (g_sched+gobuf_sp)(AX)  
	RET
noswitch:  
	// already on m stack, just call directly  
	MOVQ DI, DX  
	MOVQ 0(DI), DI  
	CALL DI  
	RET
```
>从这段代码，我们可以看出 g0 为什么同样是 G 对象，而不直接用 stack 的原因。  

M 初始化操作会检查已有的 M 数量，如超出最大限制（默认为 10,000）会导致进程崩  
溃。所有 M 被添加到 allm 链表，且不被释放。  
runtime2.go  
```go
var allm *m  
```
proc1.go  
```go
func mcommoninit(mp *m) {  
	mp.id = sched.mcount  
	sched.mcount++  
	checkmcount()  
	mpreinit(mp)  
	  
	mp.alllink = allm  
	atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))  
}  
  
func checkmcount() {  
	if sched.mcount > sched.maxmcount {  
		throw("thread exhaustion")  
	}  
}  
```
可用 runtime/debug.SetMaxThreads 修改最大线程数量限制，但仅建议在测试阶段通过设置较小  
值作为错误触发条件。  
回到 wakep/startm 流程，默认优先选用闲置 M，只是这个闲置的M 从何而来？  
runtime2.go  
```go
type schedt struct {  
	midle muintptr  // 闲置 M 链表
	nmidle int32    // 闲置的 M 数量  
	mcount int32    // 已创建的 M 总数  
	maxmcount int32 // M 最大闲置数  
}
```
proc1.go  
```go
// 从空闲链表获取 M  
func mget() *m {  
	mp := sched.midle.ptr()  
	if mp != nil {  
		sched.midle = mp.schedlink  
		sched.nmidle--  
	}  
	return mp  
}  
```
被唤醒而进入工作状态的 M，会陷入调度循环，从各种可能场所获取并执行 G 任务。只  
有当彻底找不到可执行任务，或因任务用时过长、系统调用阻塞等原因被剥夺 P 时，M  
才会进入休眠状态。  
proc1.go  
```go
// 停止 M，使其休眠  
func stopm() {  
	_g_ := getg()  
	  
	// 取消自旋状态  
	if _g_.m.spinning {  
		_g_.m.spinning = false  
		xadd(&sched.nmspinning, -1)  
	}  
  
retry:  
	// 放回闲置队列  
	mput(_g_.m)  
	  
	// 休眠，等待被唤醒  
	notesleep(&_g_.m.park)  
	noteclear(&_g_.m.park)  
	  
	// 绑定 P  
	acquirep(_g_.m.nextp.ptr())
	_g_.m.nextp = 0  
}  
  
// 将 M 放入闲置链表  
func mput(mp *m) {  
	mp.schedlink = sched.midle  
	sched.midle.set(mp)  
	sched.nmidle++  
}
```
我们允许进程里有成千上万的并发任务 G，但最好不要有太多的 M。且不说通过系统调  
用创建线程本身就有很大的性能损耗，大量闲置且不被回收的线程、M 对象、g0 栈空间  
都是资源浪费。好在这种情形极少出现，不过还是建议在生产部署前进行严格的测试。  
下面是利用 cgo 调用 sleep syscall 来生成大量 M 的示例。  
test.go
```go
package main  
  
import (  
	"sync"  
	"time"  
)  
  
// #include <unistd.h>  
import "C"  
  
func main() {  
	var wg sync.WaitGroup  
	wg.Add(1000)  
	  
	for i := 0; i < 1000; i++ {  
		go func() {  
			C.sleep(1)  
			wg.Done()  
		}()  
	}  
	  
	wg.Wait()  
	println("done!")  
	time.Sleep(time.Second * 5)  
}
```
利用 GODEBUG 输出调度器状态，你会看到大量闲置线程。
```bash
$ go build -o test test  
  
$ GODEBUG="schedtrace=1000" ./test  
  
SCHED 0ms: gomaxprocs=2 idleprocs=1 threads=3 spinningthreads=0 idlethreads=0 runqueue=0 [0 0]  
SCHED 1006ms: gomaxprocs=2 idleprocs=0 threads=728 spinningthreads=0 idlethreads=0 runqueue=125 [113 33]  
SCHED 2009ms: gomaxprocs=2 idleprocs=2 threads=858 spinningthreads=0 idlethreads=590 runqueue=0 [0 0]  
done!  
SCHED 3019ms: gomaxprocs=2 idleprocs=2 threads=858 spinningthreads=0 idlethreads=855 runqueue=0 [0 0]  
SCHED 4029ms: gomaxprocs=2 idleprocs=2 threads=858 spinningthreads=0 idlethreads=855 runqueue=0 [0 0]  
SCHED 5038ms: gomaxprocs=2 idleprocs=2 threads=858 spinningthreads=0 idlethreads=855 runqueue=0 [0 0]  
SCHED 6048ms: gomaxprocs=2 idleprocs=2 threads=858 spinningthreads=0 idlethreads=855 runqueue=0 [0 0]

```
>runqueue 输出全局队列，以及 P 本地队列的 G 任务数量。

可将 done 后的等待时间修改得更长（比如 10 分钟），用来观察垃圾回收和系统监控等  
机制是否会影响 idlethreads 数量。  
```bash
$ GODEBUG="gctrace=1,schedtrace=1000" ./test  
```

除线程数量外，程序执行时间（user、sys）也有很大差别，可以简单对比一下。  
```go
func main() {  
var wg sync.WaitGroup  
wg.Add(1000)  
  
for i := 0; i < 1000; i++ {  
	go func() {  
		C.sleep(1) // 测试 1  
		// time.Sleep(time.Second) // 测试 2  
		  
		wg.Done()  
		}()  
	}  
	  
	wg.Wait()  
}  
```

```bash
$ go build -o test1 test.go && time ./test1  
  
real 0m1.159s
user 0m0.056s  
sys  0m0.105s  
  
$ go build -o test2 test.go && time ./test2  
  
real 0m1.022s  
user 0m0.006s  
sys  0m0.006s
```
>输出结果中 user 和 sys 分别表示用户态和内核态执行时间，多核累加

标准库封装的 time.Sleep 针对 goroutine 进行了改进，并未使用 syscall。当然，这个示例  
和测试结果也仅用于演示，具体问题还须具体分析。

# 执行
M 执行 G 并发任务有两个起点：线程启动函数 mstart，还有就是 stopm 休眠唤醒后再度  
恢复调度循环。  
让我们从头开始。  
proc1.go  
```go
func mstart() {  
		_g_ := getg()  
		  
		// 确定栈边界  
		if _g_.stack.lo == 0 {  
			// 对于无法使用 g0 stack 的系统，直接在系统堆栈上划出所需空间  
			size := _g_.stack.hi  
			if size == 0 {  
				size = 8192 * stackGuardMultiplier  
			}  
			// 通过取 size 变量指针来确定高位地址  
			_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))  
			_g_.stack.lo = _g_.stack.hi - size + 1024  
		}  
		_g_.stackguard0 = _g_.stack.lo + _StackGuard  
		_g_.stackguard1 = _g_.stackguard0
		  
		mstart1()  
}  
  
func mstart1() {  
	_g_ := getg()  
	  
	if _g_ != _g_.m.g0 {  
		throw("bad runtime·mstart")  
	}  
  
	// 初始化 g0 执行现场  
	gosave(&_g_.m.g0.sched)  
	_g_.m.g0.sched.pc = ^uintptr(0) // make sure it is never used  
	  
	// 执行启动函数  
	if fn := _g_.m.mstartfn; fn != nil {  
		fn()  
	}  
  
	// 在 GC startTheWorld 时，会检查闲置 M 是否少于并发标记需求（needaddgcproc）  
	// 新建 M，设置 m.helpgc = -1，加入闲置队列等待唤醒  
	if _g_.m.helpgc != 0 {  
		_g_.m.helpgc = 0  
		stopm()  
	} else if _g_.m != &m0 {  
		// 绑定 P  
		acquirep(_g_.m.nextp.ptr())  
		_g_.m.nextp = 0  
	}  
	  
	// 进入任务调度循环（不再返回）  
	schedule()  
}
```
准备进入工作状态的 M 必须绑定一个有效 P，nextp 临时持有待绑定的 P 对象。因为在  
未正式执行前，并不适合直接设置相关属性。P 为 M 提供 cache，以便为工作线程提供  
对象内存分配。  
proc1.go  
```go
func acquirep(_p_ *p) {  
	acquirep1(_p_)
	// 绑定 mcache  
	_g_ := getg()  
	_g_.m.mcache = _p_.mcache  
}  
  
func acquirep1(_p_ *p) {  
	_g_ := getg()  
	_g_.m.p.set(_p_)  
	_p_.m.set(_g_.m)  
	_p_.status = _Prunning  
}
```
一切就绪后，M 进入核心调度循环，这是一个由 schedule、execute、goroutine fn、  
goexit 函数构成的逻辑循环。就算 M 在休眠唤醒后，也只是从“断点”恢复。  
proc1.go  
```go
func schedule() {  
	_g_ := getg()  
  
top:  
	// 准备进入 GC STW，休眠  
	if sched.gcwaiting != 0 {  
		gcstopm()  
		goto top  
	}  
  
	var gp *g  
	  
	// 当从 P.next 提取 G 时，inheritTime = true  
	// 不累加 P.schedtick 计数，使得它延长本地队列处理时间  
	var inheritTime bool  
	  
	// 进入 GC MarkWorker 工作模式  
	if gp == nil && gcBlackenEnabled != 0 {  
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())  
		if gp != nil {  
			resetspinning()  
		}  
	}  
	  
	// 每处理 n 个任务后就去全局队列获取 G 任务，以确保公平  
	if gp == nil {
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {  
			lock(&sched.lock)  
			gp = globrunqget(_g_.m.p.ptr(), 1)  
			unlock(&sched.lock)  
			if gp != nil {  
				resetspinning()  
			}  
		}  
	}  
	  
	// 从 P 本地队列获取 G 任务  
	if gp == nil {  
		gp, inheritTime = runqget(_g_.m.p.ptr())  
		if gp != nil && _g_.m.spinning {  
			throw("schedule: spinning with local work")  
		}  
	}  
	  
	// 从其他可能的地方获取 G 任务  
	// 如果获取失败，会让 M 进入休眠状态，被唤醒后重试  
	if gp == nil {  
		gp, inheritTime = findrunnable() // blocks until work is available  
		resetspinning()  
	}  
	  
	// 执行 goroutine 任务函数  
	execute(gp, inheritTime)  
}
```
>有关 lockedg 的细节，参见后文

调度函数获取可用的 G 后，交由 execute 去执行。同时，它还检查环境开关来决定是否  
参与垃圾回收。  
把相关细节放下，先走完整个调度循环再说。  
proc1.go  
```go
func execute(gp *g, inheritTime bool) {  
	_g_ := getg()  
	  
	casgstatus(gp, _Grunnable, _Grunning)  
	gp.waitsince = 0
	gp.preempt = false  
	gp.stackguard0 = gp.stack.lo + _StackGuard  
	  
	_g_.m.curg = gp  
	gp.m = _g_.m  
	  
	gogo(&gp.sched)  
}
```
真正关键的就是汇编实现的 gogo 函数。它从 g0 栈切换到 G 栈，然后用一个 JMP 指令  
进入 G 任务函数代码。  
asm_amd64.s  
```
TEXT runtime·gogo(SB), NOSPLIT, $0-8  
	MOVQ buf+0(FP), BX // gobuf  
	MOVQ gobuf_g(BX), DX // G  
	MOVQ 0(DX), CX // make sure g != nil  
	get_tls(CX)  
	MOVQ DX, g(CX) // g = G  
	MOVQ gobuf_sp(BX), SP // 通过恢复 SP 寄存器值切换到 G 栈  
	MOVQ gobuf_ret(BX), AX  
	MOVQ gobuf_ctxt(BX), DX  
	MOVQ gobuf_bp(BX), BP  
	MOVQ $0, gobuf_sp(BX) // clear to help garbage collector  
	MOVQ $0, gobuf_ret(BX)  
	MOVQ $0, gobuf_ctxt(BX)  
	MOVQ $0, gobuf_bp(BX)  
	MOVQ gobuf_pc(BX), BX // 获取 G 任务函数地址  
	JMP BX // 执行  
```
这里有个细节，JMP 并不是 CALL，也就是说不会将 PC/IP 入栈，那么执行完任务函数  
后，RET 指令恢复的 PC/IP 值是什么？我们在 schedule、execute 里也没看到 goexit 调  
用，究竟如何再次进入调度循环呢？  
在 newproc1 创建 G 任务时，我们曾忽略了一个细节。  
proc1.go  
```go
func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g {  
	newg.sched.sp = sp  
	  
	// 此处保存的是 goexit 地址
	newg.sched.pc = funcPC(goexit) + _PCQuantum  
	newg.sched.g = guintptr(unsafe.Pointer(newg))  
	  
	// 此处的调用是关键所在  
	gostartcallfn(&newg.sched, fn)  
	  
	newg.gopc = callerpc  
	newg.startpc = fn.fn  
}
```
在初始化 G.sched 时，pc 保存的是 goexit 而非 fn 。关键秘密就是随后调用的  
gostartcallfn 函数。  
stack1.go  
```go
func gostartcallfn(gobuf *gobuf, fv *funcval) {  
	gostartcall(gobuf, fn, (unsafe.Pointer)(fv))  
}  
```
sys_x86.go  
```go
func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {  
	// 调整 sp  
	sp := buf.sp  
	if regSize > ptrSize {  
		sp -= ptrSize  
		*(*uintptr)(unsafe.Pointer(sp)) = 0  
	}  
	sp -= ptrSize  
	  
	// 将 buf.pc 也就是 goexit 入栈  
	*(*uintptr)(unsafe.Pointer(sp)) = buf.pc  
	  
	// 然后再次设置 sp 和 pc，此时 pc 才是 G 任务函数  
	buf.sp = sp  
	buf.pc = uintptr(fn)  
	buf.ctxt = ctxt  
}  
```
ARM 使用 LR 寄存器存储 PC 值，而非保存在栈上。  
很显然，在初始化完成后，G 栈顶端被压入了 goexit 地址。汇编函数 gogo JMP 跳转执  
行 G 任务，那么函数尾部的 RET 指令必然是将 goexit 地址恢复到 PC/IP，从而实现任
务结束清理操作和再次进入调度循环。
asm_amd64.s  
```bash
TEXT runtime·goexit(SB),NOSPLIT,$0-0  
	CALL runtime·goexit1(SB) // does not return  
```
proc1.go  
```go
func goexit1() {  
	// 切换到 g0 执行 goexit0  
	mcall(goexit0)  
}  
  
// goexit continuation on g0  
func goexit0(gp *g) {  
	_g_ := getg()  
	  
	// 清理 G 状态  
	casgstatus(gp, _Grunning, _Gdead)  
	gp.m = nil  
	gp.lockedm = nil  
	_g_.m.lockedg = nil  
	gp.paniconfault = false  
	gp._defer = nil  
	gp._panic = nil  
	gp.writebuf = nil  
	gp.waitreason = ""  
	gp.param = nil  
	  
	dropg()  
	  
	_g_.m.locked = 0  
	  
	// 将 G 放回复用链表  
	gfput(_g_.m.p.ptr(), gp)  
	  
	// 重新进入调度循环  
	schedule()  
}  
```
无论是 mcall、systemstack，还是 gogo 都不会更新 g0.sched 栈现场。需要切换到 g0 栈  
时，直接从“g_sched+gobuf_sp”读取地址恢复 SP。所以调用 goexit0/schedule 时，g0
栈又从头开始，原调用堆栈全部失效，就算不返回也无所谓。  
在 mstart1 里调用 gosave 初始化了 g0.sched.sp 等数据，  
proc1.go  
```go
func mstart1() {  
	// Record top of stack for use by mcall.  
	// Once we call schedule we're never coming back,  
	// so other calls can reuse this stack space.  
	gosave(&_g_.m.g0.sched)  
	_g_.m.g0.sched.pc = ^uintptr(0) // make sure it is never used  
}  
```
asm_amd64.s  
```
// save state in Gobuf; setjmp  
TEXT runtime·gosave(SB), NOSPLIT, $0-8  
	MOVQ buf+0(FP), AX               // gobuf  
	LEAQ buf+0(FP), BX               // caller's SP  
	MOVQ BX, gobuf_sp(AX)  
	MOVQ 0(SP), BX                   // caller's PC  
	MOVQ BX, gobuf_pc(AX)  
	MOVQ $0, gobuf_ret(AX)  
	MOVQ $0, gobuf_ctxt(AX)  
	MOVQ BP, gobuf_bp(AX)  
	MOVQ g(CX), BX  
	MOVQ BX, gobuf_g(AX)  
	RET  
```
至此，单次任务完整结束，又回到查找待运行G 任务的状态，循环往复。  
## findrunnable  
为了找到可以运行的 G 任务，findrunnable 可谓费尽心机。本地队列、全局队列、网络  
任务（netpoll）， 甚至是从其他 P 任务队列偷窃。所有的目的都是为了尽快完成所有任  
务，充分发挥多核并行能力。  
proc1.go  
```go
func findrunnable() (gp *g, inheritTime bool) {  
	_g_ := getg()
top:  
	// 垃圾回收  
	if sched.gcwaiting != 0 {  
		gcstopm()  
		goto top  
	}  
  
	// fing 是用来执行 finalizer 的 goroutine  
	if fingwait && fingwake {  
		if gp := wakefing(); gp != nil {  
			ready(gp, 0)  
		}  
	}  
  
	// 从本地队列获取  
	if gp, inheritTime := runqget(_g_.m.p.ptr()); gp != nil {  
		return gp, inheritTime  
	}  
	  
	// 从全局队列获取  
	if sched.runqsize != 0 {  
		gp := globrunqget(_g_.m.p.ptr(), 0)  
		if gp != nil {  
			return gp, false  
		}  
	}  
	  
	// 检查 netpoll 任务  
	if netpollinited() && sched.lastpoll != 0 {  
		if gp := netpoll(false); gp != nil { // non-blocking  
			// 返回的是多任务链表，将其他任务放回全局队列  
			// gp.schedlink 链表结构  
			injectglist(gp.schedlink.ptr())  
			casgstatus(gp, _Gwaiting, _Grunnable)  
			return gp, false  
		}  
	}  
	  
	// 随机挑一个 P，偷些任务  
	for i := 0; i < int(4*gomaxprocs); i++ {  
		if sched.gcwaiting != 0 {  
			goto top  
		}
		// 随机数取模确定目标 P  
		_p_ := allp[fastrand1()%uint32(gomaxprocs)]  
		var gp *g  
		if _p_ == _g_.m.p.ptr() {  
			// 本地队列  
			gp, _ = runqget(_p_)  
		} else {  
			// 如果尝试次数太多，连目标 P.runnext 都偷，这是饿得狠了  
			stealRunNextG := i > 2*int(gomaxprocs)  
			gp = runqsteal(_g_.m.p.ptr(), _p_, stealRunNextG)  
		}  
		if gp != nil {  
			return gp, false  
		}  
	}  
  
stop:  
  
	// 检查 GC MarkWorker  
	if _p_ := _g_.m.p.ptr(); gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != nil && gcMarkWorkAvailable(_p_) {  
		_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode  
		gp := _p_.gcBgMarkWorker  
		casgstatus(gp, _Gwaiting, _Grunnable)  
		return gp, false  
	}  
	  
	// 再次检查垃圾回收状态  
	if sched.gcwaiting != 0 || _g_.m.p.ptr().runSafePointFn != 0 {  
		goto top  
	}  
	  
	// 再次尝试全局队列  
	if sched.runqsize != 0 {  
		gp := globrunqget(_g_.m.p.ptr(), 0)  
		return gp, false  
	}  
	  
	// 释放当前 P，取消自旋状态  
	_p_ := releasep()  
	pidleput(_p_)  
	if _g_.m.spinning {  
		_g_.m.spinning = false
		xadd(&sched.nmspinning, -1)  
	}  
	  
	// 再次检查所有 P 任务队列  
	for i := 0; i < int(gomaxprocs); i++ {  
		_p_ := allp[i]  
		if _p_ != nil && !runqempty(_p_) {  
			// 绑定一个空闲 P，回到头部尝试偷取任务  
			_p_ = pidleget()  
			if _p_ != nil {  
				acquirep(_p_)  
				goto top  
			}  
			break  
		}  
	}  
	  
	// 再次检查 netpoll  
	if netpollinited() && xchg64(&sched.lastpoll, 0) != 0 {  
		gp := netpoll(true) // block until new work is available  
		atomicstore64(&sched.lastpoll, uint64(nanotime()))  
		if gp != nil {  
			_p_ = pidleget()  
			if _p_ != nil {  
				acquirep(_p_)  
				injectglist(gp.schedlink.ptr())  
				casgstatus(gp, _Gwaiting, _Grunnable)  
				return gp, false  
			}  
			injectglist(gp)  
		}  
	}  
	  
	// 一无所得，休眠  
	stopm()  
	goto top  
}
```
按照查找流程，我们依次查看不同优先级的获取方式。首先是本地队列，其中 P.runnext  
优先级最高。
proc1.go  
```go
func runqget(_p_ *p) (gp *g, inheritTime bool) {  
	// 优先从 runnext 获取  
	// 循环尝试 cas。为什么用同步操作？因为可能有其他 P 从本地队列偷任务  
	for {  
		next := _p_.runnext  
		if next == 0 {  
			break  
		}  
		if _p_.runnext.cas(next, 0) {  
			return next.ptr(), true  
		}  
	}  
	  
	// 本地队列  
	for {  
		h := atomicload(&_p_.runqhead)  
		t := _p_.runqtail  
		if t == h {  
			return nil, false  
		}  
		  
		// 从头部提取  
		gp := _p_.runq[h%uint32(len(_p_.runq))]  
		if cas(&_p_.runqhead, h, h+1) { // cas-release, commits consume  
			return gp, false  
		}  
	}  
}  
```
runnext 不会影响 schedtick 计数，也就是说让 schedule 执行更多的任务才会去检查全局  
队列，所以才会有 inheritTime = true 的说法。  
在检查全局队列时，除返回一个可用 G 外，还会批量转移一批到 P 本地队列，毕竟不能  
每次加锁去操作全局队列。  
proc1.go  
```go
func globrunqget(_p_ *p, max int32) *g {  
	if sched.runqsize == 0 {  
		return nil  
	}
	// 将全局队列任务等分，计算最多能批量获取的任务数量  
	n := sched.runqsize/gomaxprocs + 1  
	if n > sched.runqsize {  
		n = sched.runqsize  
	}  
	if max > 0 && n > max {  
		n = max  
	}  
	  
	// 不能超过 runq 数组长度的一半（128）  
	if n > int32(len(_p_.runq))/2 {  
		n = int32(len(_p_.runq)) / 2  
	}  
	  
	// 调整计数  
	sched.runqsize -= n  
	if sched.runqsize == 0 {  
		sched.runqtail = 0  
	}  
	  
	// 返回第一个 G 任务，随后的才是要批量转移到本地的任务  
	gp := sched.runqhead.ptr()  
	sched.runqhead = gp.schedlink  
	n--  
	for ; n > 0; n-- {  
		gp1 := sched.runqhead.ptr()  
		sched.runqhead = gp1.schedlink  
		runqput(_p_, gp1, false)  
	}  
	  
	return gp  
}
```
只有当本地和全局队列都为空时，才会考虑去检查其他 P 任务队列。这个优先级最低，  
因为会影响目标 P 的执行（必须使用原子操作）。  
proc1.go  
```go
func runqsteal(_p_, p2 *p, stealRunNextG bool) *g {  
	t := _p_.runqtail  
	  
	// 尝试从 p2 偷取一半任务存入 p 本地队列
	n := runqgrab(p2, &_p_.runq, t, stealRunNextG)  
	if n == 0 {  
		return nil  
	}  
	  
	// 返回尾部的 G 任务  
	n--  
	gp := _p_.runq[(t+n)%uint32(len(_p_.runq))]  
	if n == 0 {  
		return gp  
	}  
	  
	// 调整目标队列尾部状态  
	atomicstore(&_p_.runqtail, t+n)  
	  
	return gp  
}  
	  
func runqgrab(_p_ *p, batch *[256]*g, batchHead uint32, stealRunNextG bool) uint32 {  
	for {  
		// 计算批量转移任务数量  
		h := atomicload(&_p_.runqhead)  
		t := atomicload(&_p_.runqtail)  
		n := t - h  
		n = n - n/2  
		  
		// 如果没有，那就尝试偷 runnext 吧  
		if n == 0 {  
			if stealRunNextG {  
				if next := _p_.runnext; next != 0 {  
					usleep(100)  
					if !_p_.runnext.cas(next, 0) {  
						continue  
					}  
					batch[batchHead%uint32(len(batch))] = next.ptr()  
					return 1  
				}  
			}  
			return 0  
		}  
		  
		// 数据异常，不可能超过一半值重试  
		if n > uint32(len(_p_.runq)/2) { // read inconsistent h and t
			continue  
		}  
		  
		// 转移任务  
		for i := uint32(0); i < n; i++ {  
			g := _p_.runq[(h+i)%uint32(len(_p_.runq))]  
			batch[(batchHead+i)%uint32(len(batch))] = g  
		}  
		  
		// 修改源 P 队列状态  
		// 失败重试。因为没有修改源和目标队列位置状态，所以没有影响  
		if cas(&_p_.runqhead, h, h+n) { // cas-release, commits consume  
			return n  
		}  
	}  
}
```
这就是某份官方文档里提及的 Work -Stealing 算法。

## lockedg
在执行 cgo 调用时，会用 lockOSThread 将 G 锁定在当前线程。  
cgocall.go  
```go
func cgocall(fn, arg unsafe.Pointer) int32 {  
	/*  
	* Lock g to m to ensure we stay on the same stack if we do a  
	* cgo callback. Add entry to defer stack in case of panic.  
	*/  
	lockOSThread()  
	mp := getg().m  
	mp.ncgocall++  
	mp.ncgo++  
	defer endcgo(mp)  
}  
  
func endcgo(mp *m) {  
	mp.ncgo--  
	unlockOSThread() // invalidates mp  
}  
```
锁定操作很简单，只须设置 G.lockedm 和 M.lockedg 即可。
proc.go  
```go
func lockOSThread() {  
	getg().m.locked += _LockInternal  
	dolockOSThread()  
}  
  
func dolockOSThread() {  
	_g_ := getg()  
	_g_.m.lockedg = _g_  
	_g_.lockedm = _g_.m  
}  
```
当调度函数 schedule 检查到 locked 属性时，会适时移交，让正确的 M 去完成任务。  
简单点说，就是 lockedm 会休眠，直到某人将 lockedg 交给它。而不幸拿到 lockedg 的  
M，则要将 lockedg 连同 P 一起传递给 lockedm，还负责将其唤醒。至于它自己，则因  
失去 P 而被迫休眠，直到 wakep 带着新的 P 唤醒它。  
proc1.go  
```go
func schedule() {  
	_g_ := getg()  
	  
	// 如果当前 M 是 lockedm，那么休眠  
	// 没有立即 execute(lockedg)，是因为该 lockedg 此时可能被其他 M 获取  
	// 兴许是中途用 gosched 暂时让出 P，进入待运行队列  
	if _g_.m.lockedg != nil {  
		stoplockedm()  
		execute(_g_.m.lockedg, false) // Never returns.  
	}  
	  
	top:  
		...  
		  
		// 如果获取到的 G 是 lockedg，那么将其连同 P 交给 lockedm 去执行  
		// 休眠，等待唤醒后重新获取可用 G  
		if gp.lockedm != nil {  
			startlockedm(gp)  
			goto top  
		}  
		  
		// 执行 goroutine 任务函数
		execute(gp, inheritTime)  
}  
  
func startlockedm(gp *g) {  
	_g_ := getg()  
	mp := gp.lockedm  
	  
	// 移交 P，并唤醒 lockedm  
	_p_ := releasep()  
	mp.nextp.set(_p_)  
	notewakeup(&mp.park)  
	  
	// 当前 M 休眠  
	stopm()  
}
```
从中可以看出，除 lockedg 只能由 lockedm 执行外，lockedm 在完成任务或主动解除锁  
定前也不会执行其他任务。这也是在前面章节我们用 cgo 生成大量 M 实例的原因。  
proc1.go  
```go
func goexit0(gp *g) {  
	_g_ := getg()  
	  
	// 解除锁定设置  
	gp.m = nil  
	gp.lockedm = nil  
	_g_.m.lockedg = nil  
}  
```
可调用 UnlockOSThread 主动解除锁定，以便允许其他 M 完成当前任务。  
proc1.go  
```go
func unlockOSThread() {  
	_g_ := getg()  
	if _g_.m.locked < _LockInternal {  
		systemstack(badunlockosthread)  
	}  
	_g_.m.locked -= _LockInternal  
	dounlockOSThread()  
}  
func dounlockOSThread() {
	_g_ := getg()  
	if _g_.m.locked != 0 {  
		return  
	}  
	_g_.m.lockedg = nil  
	_g_.lockedm = nil
}
```

# 连续栈
历经 Go 1.3、1.4 两个版本的过渡，连续栈（Contiguous Stack）的地位已经稳固。而且  
Go 1.5 和 1.4 比起来，似乎也没太多的变化，这是个好现象。  
连续栈将调用堆栈（call stack）所有栈帧分配在一个连续内存空间。当空间不足时，另  
分配 2x 内存块，并拷贝当前栈全部数据，以避免分段栈（Segmented Stack）链表结构  
在函数调用频繁时可能引发的切分热点（hot split）问题。  
结构示意图：  
![[Pasted image 20220713112730.png]]

runtime2.go  
```go
type stack struct {  
	lo uintptr  
	hi uintptr  
}  
  
type g struct {  
	// Stack parameters.  
	// stack describes the actual stack memory: [stack.lo, stack.hi).  
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.  
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.  
	stack stack  
	stackguard0 uintptr  
}
```
其中 stackguard0 是个非常重要的指针。在函数头部，编译器会插入一段指令，用它和  
SP 寄存器进行比较，从而决定是否需要对栈空间扩容。另外，它还被用作抢占调度的标  
志。  
栈空间的初始分配发生在 newproc1 创建新 G 对象时。  
stack2.go  
```go
// 操作系统需要保留的区域，比如用来处理信号等等  
_StackSystem = goos_windows*512*ptrSize + goos_plan9*512 + goos_darwin*goarch_arm*1024  
  
// 默认栈大小  
_StackMin = 2048  
  
// StackGuard 是一个警戒指针，用来判断栈容量是否需要扩张  
_StackGuard = 640*stackGuardMultiplier + _StackSystem  
```
>有几个相关常量值，以 Linux 系统为例，\_StackSystem = 0，\_StackGuard = 640。  

proc1.go  
```go
func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g {  
	newg := gfget(_p_)  
	if newg == nil {  
		newg = malg(_StackMin)  
	}  
}  
  
func malg(stacksize int32) *g {  
	newg := new(g)  
	if stacksize >= 0 {  
		stacksize = round2(_StackSystem + stacksize)  
		systemstack(func() {  
		newg.stack, newg.stkbar = stackalloc(uint32(stacksize))  
		})  
		newg.stackguard0 = newg.stack.lo + _StackGuard  
		newg.stackguard1 = ^uintptr(0)  
		newg.stackAlloc = uintptr(stacksize)  
	}  
	return newg  
}  
```
在获取栈空间后，会立即设置 stackguard0 指针。
## stackcache  

因栈空间使用频繁，所以采取了和 cache/object 类似的做法，就是按大小分成几个等级  
进行缓存复用，当然也包括回收过多的闲置块。  
以 Linux 为例，\_FixedStack 大小和 \_StackMin 相同，\_NumStackOrders 等于 4。  
stack2.go  
```go
// The minimum stack size to allocate.  
// The hackery here rounds FixedStack0 up to a power of 2.  
_FixedStack0 = _StackMin + _StackSystem  
_FixedStack1 = _FixedStack0 - 1  
_FixedStack2 = _FixedStack1 | (_FixedStack1 >> 1)  
_FixedStack3 = _FixedStack2 | (_FixedStack2 >> 2)  
_FixedStack4 = _FixedStack3 | (_FixedStack3 >> 4)  
_FixedStack5 = _FixedStack4 | (_FixedStack4 >> 8)  
_FixedStack6 = _FixedStack5 | (_FixedStack5 >> 16)  
_FixedStack = _FixedStack6 + 1  
```
malloc.go  
```go
// Number of orders that get caching. Order 0 is FixedStack  
// and each successive order is twice as large.  
// We want to cache 2KB, 4KB, 8KB, and 16KB stacks. Larger stacks  
// will be allocated directly.  
// Since FixedStack is different on different systems, we  
// must vary NumStackOrders to keep the same maximum cached size.  
// OS               | FixedStack | NumStackOrders  
// -----------------+------------+---------------  
// linux/darwin/bsd | 2KB        | 4  
// windows/32       | 4KB        | 3  
// windows/64       | 8KB        | 2  
// plan9            | 4KB        | 3  
_NumStackOrders = 4 - ptrSize/4*goos_windows - 1*goos_plan9  
```
基于同样的性能考虑（无锁分配）， 栈空间被缓存在 Cache.stackcache 数组，且使用方法  
和 object 基本相同。  
mcache.go  
```go
type mcache struct {  
	stackcache [_NumStackOrders]stackfreelist  
}
type stackfreelist struct {  
	list gclinkptr // linked list of free stacks  
	size uintptr // total size of stacks in list  
}
```
在获取栈空间时，优先检查缓存链表。大空间直接从 heap 分配。  
malloc.go  
```go
// Per-P, per order stack segment cache size.  
_StackCacheSize = 32 * 1024  
```
stack1.go  
```go
func stackalloc(n uint32) (stack, []stkbar) {  
	var v unsafe.Pointer  
	  
	// 检查是否从缓存分配  
	if stackCache != 0 && n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {  
		// 计算 order 等级  
		order := uint8(0)  
		n2 := n  
		for n2 > _FixedStack {  
			order++  
			n2 >>= 1  
		}  
		  
		var x gclinkptr  
		c := thisg.m.mcache  
		  
		// 从对应链表提取复用空间  
		x = c.stackcache[order].list  
		  
		// 提取失败，扩容后重试  
		if x.ptr() == nil {  
			stackcacherefill(c, order)  
			x = c.stackcache[order].list  
		}  
		  
		// 调整缓存链表  
		c.stackcache[order].list = x.ptr().next  
		c.stackcache[order].size -= uintptr(n)
		v = (unsafe.Pointer)(x)  
	} else {  
		// 大空间直接从 heap 分配  
		s := mHeap_AllocStack(&mheap_, round(uintptr(n), _PageSize)>>_PageShift)  
		v = (unsafe.Pointer)(s.start << _PageShift)  
	}  
	  
	top := uintptr(n) - nstkbar  
	stkbarSlice := slice{add(v, top), 0, maxstkbar}  
	return stack{uintptr(v), uintptr(v) + top}, *(*[]stkbar)(unsafe.Pointer(&stkbarSlice))  
}
```
>这个函数代码删除较多，主要是为了不影响阅读。对stackpoolalloc 在下面也会介绍。

和前文内存分配器的做法很像，不是吗？我们继续看看如何扩容。  
stack1.go  
```go
func stackcacherefill(c *mcache, order uint8) {  
	var list gclinkptr  
	var size uintptr  
	  
	// 提取一批复用空间  
	for size < _StackCacheSize/2 {  
		// 每次提取一个  
		x := stackpoolalloc(order)  
		x.ptr().next = list  
		list = x  
		size += _FixedStack << order  
	}  
	  
	// 保存到 cache.stackcache 数组  
	c.stackcache[order].list = list  
	c.stackcache[order].size = size  
}  
```
有个全局缓存 stackpool 似乎在充当 central 的角色。  
stack1.go  
```go
var stackpool [_NumStackOrders]mspan  
  
func stackpoolalloc(order uint8) gclinkptr {  
	// 尝试从全局缓存获取
	list := &stackpool[order]  
	s := list.next  
	  
	// 重新从 heap 获取 span 切分  
	if s == list {  
		s = mHeap_AllocStack(&mheap_, _StackCacheSize>>_PageShift)  
		for i := uintptr(0); i < _StackCacheSize; i += _FixedStack << order {  
			x := gclinkptr(uintptr(s.start)<<_PageShift + i)  
			x.ptr().next = s.freelist  
			s.freelist = x  
		}  
		mSpanList_Insert(list, s)  
	}  
	  
	// 从链表返回一个空间  
	x := s.freelist  
	s.freelist = x.ptr().next  
	s.ref++  
	  
	// 如果当前链表已空，则移除 span  
	if s.freelist.ptr() == nil {  
		// all stacks in s are allocated.  
		mSpanList_Remove(s)  
	}  
	  
	return x  
}
```
从 heap 获取 span 的过程没有任何惊喜。  
mheap.go  
```go
func mHeap_AllocStack(h *mheap, npage uintptr) *mspan {  
	s := mHeap_AllocSpanLocked(h, npage)  
	if s != nil {  
		s.state = _MSpanStack  
		s.freelist = 0  
		s.ref = 0  
	}  
	return s  
}  
```
简单总结一下：栈内存也从 arena 区域分配，使用和对象分配相同的策略和算法。只是
我有些不明白，这东西是不是可以做到内存分配器里面？还是说为了以后修改方便才独  
立出来的？  
## morestack  
执行函数前，需要为其准备好所需栈帧空间，此时是检查连续栈是否需要扩容的最佳时
机。为此，编译器会在函数头部插入几条特殊指令，通过比较 stackguard0 和 SP 来决定  
是否进行扩容操作。  
test.go  
```go
package main  
  
func test() {  
	println("hello")  
}  
  
func main() {  
	test()  
}  
```
编译（禁用内联），反汇编。  
```bash
$ go build -gcflags "-l" -o test test.go  
  
$ go tool objdump -s "main\.test" test  
  
TEXT main.test(SB) test.go  
	test.go:3 0x2040 GS MOVQ GS:0x8a0, CX // 当前 G  
	test.go:3 0x2049 CMPQ 0x10(CX), SP // G+0x10 指向 g.stackguard0，和 SP 比较  
	test.go:3 0x204d JBE 0x2080 // 如果 SP <= stackguard0，则跳转到 0x2080  
	test.go:3 0x204f SUBQ $0x10, SP // 预留当前栈帧空间  
	test.go:4 0x2053 CALL runtime.printlock(SB)  
	test.go:4 0x2058 LEAQ 0x6b4f9(IP), BX  
	test.go:4 0x205f MOVQ BX, 0(SP)  
	test.go:4 0x2063 MOVQ $0x5, 0x8(SP)  
	test.go:4 0x206c CALL runtime.printstring(SB)  
	test.go:4 0x2071 CALL runtime.printnl(SB)  
	test.go:4 0x2076 CALL runtime.printunlock(SB)  
	test.go:5 0x207b ADDQ $0x10, SP  
	test.go:5 0x207f RET  
	test.go:3 0x2080 CALL runtime.morestack_noctxt(SB) // 执行 morestack 扩容
	test.go:3 0x2085 JMP main.test(SB) // 扩容结束后，重新执行当前函数
```
这几条指令很简单。如果 SP 指针地址小于 stackguard0（栈从高位地址向低位地址分  
配），那么显然已经溢出，这就需要扩容，否则当前和后续函数就无从分配栈帧内存。  
细心一点，你会发现 CMP 指令并没将当前栈帧所需空间算上。假如 SP 大于  
stackguard0，但与其差值又小于当前栈帧大小呢？这显然不会跳转执行扩容操作，但又  
不能满足当前函数需求，难道只能眼看着堆栈溢出？  
我们知道在 stack.lo 和 stackguard0 之间尚有部分保留空间，所以适当“溢出”是允许  
的。  
stack2.go  
```go
// After a stack split check the SP is allowed to be this many bytes below the stack guard.  
// This saves an instruction in the checking sequence for tiny frames.  
_StackSmall = 128  
```
修改一下测试代码，看看效果。  
test.go  
```go
package main  
  
func test1() {  
	var x [128]byte  
	x[1] = 1  
}  
  
func test2() {  
	var x [129]byte  
	x[1] = 1  
}  
  
func main() {  
	test1()  
	test2()  
}  
```

```bash
$ go build -gcflags "-l" -o test test.go  
  
$ go tool objdump -s "main\.test" test
TEXT main.test1(SB) test.go  
	test.go:3 0x2040 GS MOVQ GS:0x8a0, CX  
	test.go:3 0x2049 CMPQ 0x10(CX), SP // 当前栈帧 0x80 正好是 128  
	test.go:3 0x204d JBE 0x206e  
	test.go:3 0x204f SUBQ $0x80, SP  
  
TEXT main.test2(SB) test.go  
	test.go:8 0x2080 GS MOVQ GS:0x8a0, CX  
	test.go:8 0x2089 LEAQ -0x8(SP), AX // 当前栈帧 0x88 - 128 = 0x8，适当调整 SP 后再比较  
	test.go:8 0x208e CMPQ 0x10(CX), AX  
	test.go:8 0x2092 JBE 0x20b5  
	test.go:8 0x2094 SUBQ $0x88, SP
```
很显然，如果当前栈帧是 SmallStack（0x80）， 那么就允许在 [lo, stackguard0] 之间分  
配。  
对栈进行扩容并不是件容易的事情，其中涉及很多内容。不过，在这里我们只须了解其  
基本过程和算法意图，无须深入到所有细节。  
asm_amd64.s  
```go
TEXT runtime·morestack_noctxt(SB),NOSPLIT,$0  
	MOVL $0, DX  
	JMP runtime·morestack(SB)  
  
TEXT runtime·morestack(SB),NOSPLIT,$0-0  
	// Call newstack on m->g0's stack.  
	MOVQ m_g0(BX), BX  
	MOVQ BX, g(CX)  
	MOVQ (g_sched+gobuf_sp)(BX), SP  
	CALL runtime·newstack(SB)  
	MOVQ $0, 0x1003 // crash if newstack returns  
	RET  
```
基本过程就是分配一个 2x 大小的新栈，然后将数据拷贝过去，替换掉旧栈。当然，这  
期间需要对指针等内容做些调整。  
stack1.go  
```go
func newstack() {  
	thisg := getg()  
	gp := thisg.m.curg
	// 调整执行现场记录  
	rewindmorestack(&gp.sched)  
	  
	casgstatus(gp, _Grunning, _Gwaiting)  
	gp.waitreason = "stack growth"  
	  
	sp := gp.sched.sp  
	  
	// 扩张 2 倍  
	oldsize := int(gp.stackAlloc)  
	newsize := oldsize * 2  
	  
	casgstatus(gp, _Gwaiting, _Gcopystack)  
	  
	// 拷贝栈数据后切换到新栈  
	copystack(gp, uintptr(newsize))  
	  
	// 恢复执行  
	casgstatus(gp, _Gcopystack, _Grunning)  
	gogo(&gp.sched)  
}  
	  
func copystack(gp *g, newsize uintptr) {  
	old := gp.stack  
	used := old.hi - gp.sched.sp  
	  
	// 从缓存或堆分配新栈空间  
	new, newstkbar := stackalloc(uint32(newsize))  
	  
	// 清零  
	if stackPoisonCopy != 0 {  
		fillstack(new, 0xfd)  
	}  
	  
	// 调整指针等操作 ...  
	  
	// 拷贝数据到新栈空间  
	memmove(unsafe.Pointer(new.hi-used), unsafe.Pointer(old.hi-used), used)  
	  
	// 切换到新栈  
	gp.stack = new  
	gp.stackguard0 = new.lo + _StackGuard
	gp.sched.sp = new.hi - used  
	oldsize := gp.stackAlloc  
	gp.stackAlloc = newsize  
	gp.stkbar = newstkbar  
	  
	// 将旧栈清零后释放  
	if stackPoisonCopy != 0 {  
		fillstack(old, 0xfc)  
	}  
	stackfree(old, oldsize)  
}
```

## stackfree
释放栈空间的操作，依旧与回收 object 类似。  
stack1.go  
```go
func stackfree(stk stack, n uintptr) {  
	gp := getg()  
	v := (unsafe.Pointer)(stk.lo)  
  
	// 放回缓存链表  
	if stackCache != 0 && n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {  
		// 计算 order 等级  
		order := uint8(0)  
		n2 := n  
		for n2 > _FixedStack {  
			order++  
			n2 >>= 1  
		}  
		x := gclinkptr(v)  
		c := gp.m.mcache  
		  
		// 如果缓存大小超出限制，则释放一些  
		if c.stackcache[order].size >= _StackCacheSize {  
			stackcacherelease(c, order)  
		}  
		  
		// 放回缓存链表  
		x.ptr().next = c.stackcache[order].list  
		c.stackcache[order].list = x  
		c.stackcache[order].size += n 
	} else {  
		s := mHeap_Lookup(&mheap_, v)  
		if gcphase == _GCoff {  
			// 归还给 heap  
			mHeap_FreeStack(&mheap_, s)  
		} else {  
			// 如果正在垃圾回收期间，那么放到一个待处理队列，由垃圾回收器处理  
			mSpanList_Insert(&stackFreeQueue, s)  
		}  
	}  
}  
```
回收的栈空间被放回对应复用链表。如缓存过多，则转移一批到全局链表，或直接将自  
由的 span 归还给 heap。  
```go
func stackcacherelease(c *mcache, order uint8) {  
	x := c.stackcache[order].list  
	size := c.stackcache[order].size  
	  
	// 如果当前链表过大，则释放一半  
	for size > _StackCacheSize/2 {  
		y := x.ptr().next  
		  
		// 每次释放一个，它们可能属于不同的 span  
		stackpoolfree(x, order)  
		  
		x = y  
		size -= _FixedStack << order  
	}  
	  
	c.stackcache[order].list = x  
	c.stackcache[order].size = size  
}  
  
func stackpoolfree(x gclinkptr, order uint8) {  
	// 找到所属 span  
	s := mHeap_Lookup(&mheap_, (unsafe.Pointer)(x))  
	if s.freelist.ptr() == nil {  
		mSpanList_Insert(&stackpool[order], s)  
	}  
	  
	// 添加到 span.freelist  
	x.ptr().next = s.freelist
	s.freelist = x  
	s.ref--  
	  
	// 如果该 span 已收回全部空间，那么将其归还给 heap  
	if gcphase == _GCoff && s.ref == 0 {  
		mSpanList_Remove(s)  
		s.freelist = 0  
		mHeap_FreeStack(&mheap_, s)  
	}  
}
```
除了 morestack 调用导致 stackfree 操作外，垃圾回收对栈空间的处理也会发生此操作。  
mgcmark.go  
```go
func markroot(desc *parfor, i uint32) {  
	switch i {  
	case _RootFlushCaches:  
		if gcphase != _GCscan {  
			flushallmcaches()  
		}  
	default:  
		if gcphase == _GCmarktermination {  
			shrinkstack(gp)  
		}  
	}  
}  
```
mstats.go  
```go
func flushallmcaches() {  
	for i := 0; ; i++ {  
		p := allp[i]  
		c := p.mcache  
		mCache_ReleaseAll(c)  
		stackcache_clear(c)  
	}  
}  
```
mgc.go  
```go
func gcMark(start_time int64) {  
	freeStackSpans()  
}
```
因垃圾回收需要，stackcache_clear 会将所有cache 缓存的栈空间归还给全局或 heap。  
stack1.go  
```go
func stackcache_clear(c *mcache) {  
	for order := uint8(0); order < _NumStackOrders; order++ {  
		x := c.stackcache[order].list  
		for x.ptr() != nil {  
			y := x.ptr().next  
			stackpoolfree(x, order)  
			x = y  
		}  
		c.stackcache[order].list = 0  
		c.stackcache[order].size = 0  
	}  
}  
```
而 shrinkstack 除收回闲置栈空间外，还会收缩正在使用但被扩容过的栈以节约内存。  
```go
func shrinkstack(gp *g) {  
	if readgstatus(gp) == _Gdead {  
		if gp.stack.lo != 0 {  
			// 回收闲置 G 的栈空间，重新使用前会为其补上  
			stackfree(gp.stack, gp.stackAlloc)  
			gp.stack.lo = 0  
			gp.stack.hi = 0  
			gp.stkbar = nil  
			gp.stkbarPos = 0  
		}  
		return  
	}  
  
	// 收缩目标是一半大小  
	oldsize := gp.stackAlloc  
	newsize := oldsize / 2  
	if newsize < _FixedStack {  
		return  
	}  
	  
	// 如果使用的空间超过 1/4，则不收缩  
	avail := gp.stack.hi - gp.stack.lo  
	if used := gp.stack.hi - gp.sched.sp + _StackLimit; used >= avail/4 {  
		return  
	}
	// 用较小的栈替换  
	oldstatus := casgcopystack(gp)  
	copystack(gp, newsize)  
	casgstatus(gp, _Gcopystack, oldstatus)  
}
```
最后就是 freeStackSpans，它扫描全局队列 stackpool 和暂存队列 stackFreeQueue，将那  
些空间已完全收回的 span 交还给 heap。  
stack1.go  
```go
func freeStackSpans() {  
	for order := range stackpool {  
		list := &stackpool[order]  
		for s := list.next; s != list; {  
			next := s.next  
			if s.ref == 0 {  
				mSpanList_Remove(s)  
				s.freelist = 0  
				mHeap_FreeStack(&mheap_, s)  
			}  
			s = next  
		}  
	}  
	  
	for stackFreeQueue.next != &stackFreeQueue {  
		s := stackFreeQueue.next  
		mSpanList_Remove(s)  
		mHeap_FreeStack(&mheap_, s)  
	}  
}  
```
另外，调整 P 数量的 procresize，将任务完成的 G 对象放回复用链表的gfput，同样会引  
发栈空间释放操作。只是流程和上述基本类似，不再赘述。  
>运行时三大核心组件之间，相互纠缠得太多太细，已无从划分边界，我个人觉得这并不是什么  
好主意。诚然，为了性能，很多地方直接植入代码，而非通过消息或接口隔离等方式封装，但随  
着各部件复杂度和规模的提升，其可维护性必然也会降低。不知道开发团队对此有什么具体的想  
法。

# 系统调用
为支持并发调度，Go 专门对 syscall、cgo 进行了包装，以便在长时间阻塞时能切换执行  
其他任务。在标准库 syscall 包里，将系统调用函数分为 Syscall 和 RawSyscall 两类。  
src/syscall/zsyscall_linux_amd64.s  
```go
func Getcwd(buf []byte) (n int, err error) {  
	r0, _, e1 := Syscall(SYS_GETCWD, uintptr(_p0), uintptr(len(buf)), 0)  
}  
  
func EpollCreate(size int) (fd int, err error) {  
	r0, _, e1 := RawSyscall(SYS_EPOLL_CREATE, uintptr(size), 0, 0)  
}  
```
让我们看看这两者有什么区别。  
src/syscall/asm_linux_amd64.s  
```go
TEXT ·Syscall(SB),NOSPLIT,$0-56  
	CALL runtime·entersyscall(SB)  
	MOVQ trap+0(FP), AX // syscall entry  
	SYSCALL  
	JLS ok  
	CALL runtime·exitsyscall(SB)  
	RET  
ok:  
	CALL runtime·exitsyscall(SB)  
	RET  
  
  
TEXT ·RawSyscall(SB),NOSPLIT,$0-56  
	MOVQ trap+0(FP), AX // syscall entry  
	SYSCALL  
	JLS ok1  
	RET  
ok1:  
	RET  
```
最大的不同在于 Syscall 增加了 entrysyscall/exitsyscall，这就是允许调度的关键所在。
proc1.go  
```go
func entersyscall(dummy int32) {  
	reentersyscall(getcallerpc(unsafe.Pointer(&dummy)), getcallersp(unsafe.Pointer(&dummy)))  
}  
  
func reentersyscall(pc, sp uintptr) {  
	_g_ := getg()  
	  
	// 保存执行现场  
	save(pc, sp)  
	  
	_g_.syscallsp = sp  
	_g_.syscallpc = pc  
	casgstatus(_g_, _Grunning, _Gsyscall)  
	  
	// 确保 sysmon 运行  
	if atomicload(&sched.sysmonwait) != 0 {  
		systemstack(entersyscall_sysmon)  
		save(pc, sp)  
	}  
	  
	// 设置相关状态  
	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick  
	_g_.sysblocktraced = true  
	_g_.m.mcache = nil  
	_g_.m.p.ptr().m = 0  
	atomicstore(&_g_.m.p.ptr().status, _Psyscall)  
}  
```
监控线程 sysmon 对 syscall 非常重要，因为它负责将因系统调用而长时间阻塞的 P 抢  
回，用于执行其他任务。否则，整体性能会严重下降，甚至整个进程都会被冻结。  
proc1.go  
```go
func entersyscall_sysmon() {  
	if atomicload(&sched.sysmonwait) != 0 {  
		atomicstore(&sched.sysmonwait, 0)  
		notewakeup(&sched.sysmonnote)  
	}  
}  
```
某些系统调用本身就可以确定长时间阻塞（比如锁）， 那么它会选择执行entersyscallblock 主动交出所关联的P。
proc1.go  
```go
func entersyscallblock(dummy int32) {  
	casgstatus(_g_, _Grunning, _Gsyscall)  
	systemstack(entersyscallblock_handoff)  
}  
  
func entersyscallblock_handoff() {  
	// 释放 P，让它去执行其他任务  
	handoffp(releasep())  
}  
  
func handoffp(_p_ *p) {  
	// 如果 P 本地或全局有任务，直接唤醒某个 M 开始工作  
	if !runqempty(_p_) || sched.runqsize != 0 {  
		startm(_p_, false)  
		return  
	}  
	  
	...  
	  
	// 没有任务就放回空闲队列  
	pidleput(_p_)  
}  
```
从系统调用返回时，必须检查 P 是否依然可用，因为可能已被 sysmon 抢走。  
proc1.go  
```go
func exitsyscall(dummy int32) {  
	_g_ := getg()  
	oldp := _g_.m.p.ptr()  
	  
	if exitsyscallfast() {  
		casgstatus(_g_, _Gsyscall, _Grunning)  
		return  
	}  
	  
	mcall(exitsyscall0)  
}  
```
快速退出 exitsyscallfast 是指能重新绑定原有或空闲的 P，以继续当前 G 任务的执行。
proc1.go  
```go
func exitsyscallfast() bool {  
	_g_ := getg()  
	  
	// STW 状态，就不要继续了  
	if sched.stopwait == freezeStopWait {  
		_g_.m.mcache = nil  
		_g_.m.p = 0  
		return false  
	}  
	  
	// 尝试关联原本的 P  
	if _g_.m.p != 0 && _g_.m.p.ptr().status == _Psyscall &&  
	cas(&_g_.m.p.ptr().status, _Psyscall, _Prunning) {  
			_g_.m.mcache = _g_.m.p.ptr().mcache  
			_g_.m.p.ptr().m.set(_g_.m)  
			return true  
	}  
	  
	// 获取其他空闲 P  
	oldp := _g_.m.p.ptr()  
	_g_.m.mcache = nil  
	_g_.m.p = 0  
	if sched.pidle != 0 {  
		var ok bool  
		systemstack(func() {  
			ok = exitsyscallfast_pidle()  
		})  
		if ok {  
			return true  
		}  
	}  
	return false  
}  
	  
func exitsyscallfast_pidle() bool {  
	_p_ := pidleget()  
	  
	// 唤醒 sysmon  
	if _p_ != nil && atomicload(&sched.sysmonwait) != 0 {  
		atomicstore(&sched.sysmonwait, 0)  
		notewakeup(&sched.sysmonnote)
	}  
  
	// 重新关联  
	if _p_ != nil {  
		acquirep(_p_)  
		return true  
	}  
	return false  
}  
```
如果多次尝试绑定 P 却失败，那么只能将当前任务放入待运行队列。  
proc1.go  
```go
func exitsyscall0(gp *g) {  
	_g_ := getg()  
	  
	// 修改状态，解除和 M 的关联  
	casgstatus(gp, _Gsyscall, _Grunnable)  
	dropg()  
	  
	// 再次获取空闲 P  
	_p_ := pidleget()  
	if _p_ == nil {  
		// 获取失败，放回全局任务队列  
		globrunqput(gp)  
	} else if atomicload(&sched.sysmonwait) != 0 {  
		atomicstore(&sched.sysmonwait, 0)  
		notewakeup(&sched.sysmonnote)  
	}  
	  
	// 再次检查 P，以便执行当前任务  
	if _p_ != nil {  
		acquirep(_p_)  
		execute(gp, false) // Never returns.  
	}  
	  
	// 关联 P 失败，休眠当前 M  
	stopm()  
	schedule() // Never returns.  
}  
```
需要注意，cgo 使用了相同的封装方式，因为它同样不受调度器管理。
cgocall.go  
```go
func cgocall(fn, arg unsafe.Pointer) int32 {  
	/*  
	* Announce we are entering a system call  
	* so that the scheduler knows to create another M to run goroutines while we are in  
	* the foreign code.  
	*  
	* The call to asmcgocall is guaranteed not to split the stack and does not allocate  
	* memory, so it is safe to call while "in a system call", outside the $GOMAXPROCS  
	* accounting.  
	*/  
	entersyscall(0)  
	errno := asmcgocall(fn, arg)  
	exitsyscall(0)  
}
```

# 监控
我们在前面已经介绍过好几回系统监控线程了，现在对它做个总结。  
 释放闲置超过 5 分钟的 span 物理内存。  
 如果超过 2 分钟没有垃圾回收，则强制执行。  
 将长时间未处理的 netpoll 结果添加到任务队列。  
 向长时间运行的 G 任务发出抢占调度。  
 收回因 syscall 而长时间阻塞的 P。  
在进入垃圾回收状态时，sysmon 会自动进入休眠，所以我们才会在 syscall 里看到很多  
唤醒指令。另外，startTheWorld 也会做唤醒处理。保证监控线程正常运行，对内存分  
配、垃圾回收和并发调度都非常重要。  
proc1.go  
```go
func startTheWorldWithSema() {  
	sched.gcwaiting = 0  
	if sched.sysmonwait != 0 {  
		sched.sysmonwait = 0  
		notewakeup(&sched.sysmonnote)  
	}
}
```
现在，让我们忽略其他任务，看看对 syscall 和 preempt 的处理。
proc1.go  
```go
func sysmon() {  
	for {  
		usleep(delay)  
		  
		// STW 时休眠 sysmon  
		if debug.schedtrace <= 0 &&  
		(sched.gcwaiting != 0 || atomicload(&sched.npidle) == uint32(gomaxprocs)) {  
			if atomicload(&sched.gcwaiting) != 0 ||  
			atomicload(&sched.npidle) == uint32(gomaxprocs) {  
				// 设置休眠标志，休眠（有个超时，苏醒保障）  
				atomicstore(&sched.sysmonwait, 1)  
				notetsleep(&sched.sysmonnote, maxsleep)  
				  
				// 唤醒后重置状态标志，继续执行  
				atomicstore(&sched.sysmonwait, 0)  
				noteclear(&sched.sysmonnote)  
			}  
		}  
		  
		lastpoll := int64(atomicload64(&sched.lastpoll))  
		now := nanotime()  
		unixnow := unixnanotime()  
		  
		// 获取超过 10ms 的 netpoll 结果  
		if lastpoll != 0 && lastpoll+10*1000*1000 < now {  
			cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))  
			gp := netpoll(false) // non-blocking - returns list of goroutines  
			if gp != nil {  
				injectglist(gp)  
			}  
		}  
		  
		// 抢夺 syscall 长时间阻塞的 P  
		// 向长时间运行的 G 发出抢占调度  
		if retake(now) != 0 {  
			idle = 0  
		} else {  
			idle++
		}  
	}  
}
```
专门有个 pdesc 的全局变量用于保存 sysmon 运行统计信息，据此来判断 syscall 和 G 是  
否超时。  
proc1.go  
```go
var pdesc [_MaxGomaxprocs]struct {  
	schedtick uint32  
	schedwhen int64  
	syscalltick uint32  
	syscallwhen int64  
}  
  
const forcePreemptNS = 10 * 1000 * 1000 // 10ms  
  
func retake(now int64) uint32 {  
// 遍历 P  
	for i := int32(0); i < gomaxprocs; i++ {  
		_p_ := allp[i]  
		pd := &pdesc[i]  
		s := _p_.status  
		  
		// P 处于 syscall 模式  
		if s == _Psyscall {  
			// 更新 syscall 统计信息  
			t := int64(_p_.syscalltick)  
			if int64(pd.syscalltick) != t {  
				pd.syscalltick = uint32(t)  
				pd.syscallwhen = now  
				continue  
			}  
		  
			// 检查是否有其他任务需要 P，是否超出时间限制，是否有必要抢夺 P  
			if runqempty(_p_) &&  
				atomicload(&sched.nmspinning)+atomicload(&sched.npidle) > 0 &&  
				pd.syscallwhen+10*1000*1000 > now {  
				continue  
			}  
		  
			// 抢夺 P
			if cas(&_p_.status, s, _Pidle) {  
				_p_.syscalltick++  
				handoffp(_p_)  
			}  
		} else if s == _Prunning {  
			// 更新 G 运行统计信息  
			t := int64(_p_.schedtick)  
			if int64(pd.schedtick) != t {  
				pd.schedtick = uint32(t)  
				pd.schedwhen = now  
				continue  
			}  
		  
			// 如果没超过 10ms，则忽略  
			if pd.schedwhen+forcePreemptNS > now {  
				continue  
			}  
		  
			// 发出抢占调度  
			preemptone(_p_)  
		}  
	}  
}
```
##  抢占调度
所谓抢占调度要比你想象的简单许多，远不是你以为的“抢占式多任务操作系统”那种  
样子。因为 Go 调度器并没有真正意义上的时间片概念，只是在目标 G 上设置一个抢占  
标志，当该任务调用某个函数时，被编译器安插的指令就会检查这个标志，从而决定是  
否暂停当前任务。  
proc1.go  
```go
// Tell the goroutine running on processor P to stop.  
// This function is purely best-effort. It can incorrectly fail to inform the  
// goroutine. It can send inform the wrong goroutine. Even if it informs the  
// correct goroutine, that goroutine might ignore the request if it is  
// simultaneously executing newstack.  
// No lock needs to be held.  
// Returns true if preemption request was issued.  
// The actual preemption will happen at some point in the future  
// and will be indicated by the gp->status no longer being
// Grunning  
func preemptone(_p_ *p) bool {  
	mp := _p_.m.ptr()  
	gp := mp.curg  
	gp.preempt = true  
	  
	// Every call in a go routine checks for stack overflow by  
	// comparing the current stack pointer to gp->stackguard0.  
	// Setting gp->stackguard0 to StackPreempt folds  
	// preemption into the normal stack overflow check.  
	gp.stackguard0 = stackPreempt  
	return true  
}
```
>保留这段代码里的注释，是想告诉你，preempt 真的有些不靠谱。  

有两个标志，实际起作用的是 G.stackguard0。G.preempt 只是后备，以便在 stackguard0  
做回溢出检查标志时，依然可用 preempt 恢复抢占状态。  
编译器插入的指令？没错，就是那个 morestack。当它调用 newstack 扩容时会检查抢占  
标志，并决定是否暂停当前任务，当然这发生在实际扩容之前。  
stack1.go  
```go
func newstack() {  
	preempt := atomicloaduintptr(&gp.stackguard0) == stackPreempt  
	  
	if preempt {  
		// 如果 M 持有锁，或者正在进行内存分配、垃圾回收等操作，不抢占，留待下次  
		if thisg.m.locks != 0 || thisg.m.mallocing != 0 ||  
		thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {  
			// stackguard0 恢复溢出检查用途，下次用 G.preempt 恢复  
			gp.stackguard0 = gp.stack.lo + _StackGuard  
			gogo(&gp.sched) // never return  
		}  
	}  
  
	if preempt {  
			// 垃圾回收本身也算一次抢占，忽略本次抢占调度  
			if gp.preemptscan {  
				for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {  
					// Likely to be racing with the GC as  
					// it sees a _Gwaiting and does the
					// stack scan. If so, gcworkdone will  
					// be set and gcphasework will simply  
					// return.  
				}  
				if !gp.gcscandone {  
					scanstack(gp)  
					gp.gcscandone = true  
				}  
				gp.preemptscan = false  
				gp.preempt = false  
				casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)  
				casgstatus(gp, _Gwaiting, _Grunning)  
				gp.stackguard0 = gp.stack.lo + _StackGuard  
				gogo(&gp.sched) // never return  
			}  
			  
			// 开始抢占调度，将当前 G 放回队列，让 M 执行其他任务  
			casgstatus(gp, _Gwaiting, _Grunning)  
			gopreempt_m(gp) // never return  
	}  
  
	// Allocate a bigger segment and move the stack.  
	copystack(gp, uintptr(newsize))  
	gogo(&gp.sched)  
}  
```
proc1.go  
```go
func gopreempt_m(gp *g) {  
	goschedImpl(gp)  
}  
  
func goschedImpl(gp *g) {  
	status := readgstatus(gp)  
	casgstatus(gp, _Grunning, _Grunnable)  
	dropg()  
	globrunqput(gp)  
	  
	schedule()  
}
```
>这个抢占调度机制给我的感觉是越来越弱，毕竟垃圾回收和栈扩容这个时机都不是很“确定”  
和“实时”，更何况还有函数内联和纯算法循环等造成 morestack 不会执行等因素。不知道对此Go  的后续版本会有何改进。

# 其他
本节介绍与任务执行有关的几种暂停操作。  
## Gosched  
可被用户调用的 runtime.Gosched 将当前 G 任务暂停，重新放回全局队列，让出当前 M  
去执行其他任务。我们无须对 G 做唤醒操作，因为它总归会被某个 M 重新拿到，并从  
“断点”恢复。  
proc.go  
```go
func Gosched() {  
	mcall(gosched_m)  
}  
```
proc1.go  
```go
func gosched_m(gp *g) {  
	goschedImpl(gp)  
}  
  
func goschedImpl(gp *g) {  
	// 重置属性  
	casgstatus(gp, _Grunning, _Grunnable)  
	dropg()  
	  
	// 将当前 G 放回全局队列  
	globrunqput(gp)  
	  
	// 重新调度执行其他任务  
	schedule()  
}  
	  
func dropg() {
	_g_ := getg()  
	  
	if _g_.m.lockedg == nil {  
		_g_.m.curg.m = nil  
		_g_.m.curg = nil  
	}  
}
```
实现“断点恢复”的关键由 mcall 实现，它将当前执行状态，包括 SP、PC 寄存器等值  
保存到 G.sched 区域。  
asm_amd64.s  
```go
TEXT runtime·mcall(SB), NOSPLIT, $0-8  
	MOVQ fn+0(FP), DI  
	  
	get_tls(CX)  
	MOVQ g(CX), AX // save state in g->sched  
	MOVQ 0(SP), BX // caller's PC  
	MOVQ BX, (g_sched+gobuf_pc)(AX)  
	LEAQ fn+0(FP), BX // caller's SP  
	MOVQ BX, (g_sched+gobuf_sp)(AX)  
	MOVQ AX, (g_sched+gobuf_g)(AX)  
	MOVQ BP, (g_sched+gobuf_bp)(AX)  
	  
	// switch to m->g0 & its stack, call fn  
	...  
```
当 execute/gogo 再次执行该任务时，自然可从中恢复状态。反正执行栈是 G 自带的，不  
用担心执行数据丢失。  
## gopark  
与 Gosched 最大的区别在于，gopark 并没将 G 放回待运行队列。也就是说，必须主动  
恢复，否则该任务会遗失。  
proc.go  
```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason string, ...) {  
	mp := acquirem()  
	gp := mp.curg
	mp.waitlock = lock  
	mp.waitunlockf = *(*unsafe.Pointer)(unsafe.Pointer(&unlockf))  
	gp.waitreason = reason  
	mp.waittraceev = traceEv  
	mp.waittraceskip = traceskip  
	releasem(mp)  
	  
	mcall(park_m)  
}
```
可看到gopark 同样是由 mcall 保存执行状态，还有个 unlockf 作为暂停判断条件。  
proc1.go  
```go
func park_m(gp *g) {  
	_g_ := getg()  
	  
	// 重置属性  
	casgstatus(gp, _Grunning, _Gwaiting)  
	dropg()  
	  
	// 执行解锁函数。如果返回 false，则恢复执行  
	if _g_.m.waitunlockf != nil {  
		fn := *(*func(*g, unsafe.Pointer) bool)(unsafe.Pointer(&_g_.m.waitunlockf))  
		ok := fn(gp, _g_.m.waitlock)  
		_g_.m.waitunlockf = nil  
		_g_.m.waitlock = nil  
		if !ok {  
			casgstatus(gp, _Gwaiting, _Grunnable)  
			execute(gp, true) // Schedule it back, never returns  
		}  
	}  
	  
	// 调度执行其他任务  
	schedule()  
}  
```
与之配套，goready 用于恢复执行，G 被放回优先级最高的 P.runnext。  
proc.go  
```go
func goready(gp *g, traceskip int) {  
	systemstack(func() {  
		ready(gp, traceskip)
	})  
}
```


# 用 GODEBUG 看调度跟踪


## GODEBUG

GODEBUG 变量可以控制运行时内的调试变量，参数以逗号分隔，格式为：`name=val`。本文着重点在调度器观察上，将会使用如下两个参数：

-   schedtrace：设置 `schedtrace=X` 参数可以使运行时在每 X 毫秒发出一行调度器的摘要信息到标准 err 输出中。
-   scheddetail：设置 `schedtrace=X` 和 `scheddetail=1` 可以使运行时在每 X 毫秒发出一次详细的多行信息，信息内容主要包括调度程序、处理器、OS 线程 和 Goroutine 的状态。

### 6.4.2.1 示例代码

创建一个 main.go 文件，写入示例代码，如下：

```go
func main() {
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(wg *sync.WaitGroup) {
			var counter int
			for i := 0; i < 1e10; i++ {
				counter++
			}
			wg.Done()
		}(&wg)
	}

	wg.Wait()
}
```

### 6.4.2.2 schedtrace

```shell
$ GODEBUG=schedtrace=1000 go run main.go 
SCHED 0ms: gomaxprocs=4 idleprocs=1 threads=5 spinningthreads=1 idlethreads=0 runqueue=0 [0 0 0 0]
SCHED 1000ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 2000ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 3001ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
...
SCHED 14052ms: gomaxprocs=4 idleprocs=2 threads=5 
...
```

-   sched：每一行都代表调度器的调试信息，后面提示的毫秒数表示启动到现在的运行时间，输出的时间间隔受 `schedtrace` 的值影响。
-   gomaxprocs：当前的 CPU 核心数（GOMAXPROCS 的当前值）。
-   idleprocs：空闲的处理器数量，后面的数字表示当前的空闲数量。
-   threads：OS 线程数量，后面的数字表示当前正在运行的线程数量。
-   spinningthreads：自旋状态的 OS 线程数量。
-   idlethreads：空闲的线程数量。
-   runqueue：全局队列中中的 Goroutine 数量，而后面的 [0 0 1 1] 则分别代表这 4 个 P 的本地队列正在运行的 Goroutine 数量。

在上面我们有提到 “自旋线程” 这个概念，如果你之前没有了解过相关概念，一听 “自旋” 肯定会比较懵，我们引用 《Head First of Golang Scheduler》 的内容来说明：

> 自旋线程的这个说法，是因为 Go Scheduler 的设计者在考虑了 “OS 的资源利用率” 以及 “频繁的线程抢占给 OS 带来的负载” 之后，提出了 “Spinning Thread” 的概念。也就是当 “自旋线程” 没有找到可供其调度执行的 Goroutine 时，并不会销毁该线程 ，而是采取 “自旋” 的操作保存了下来。虽然看起来这是浪费了一些资源，但是考虑一下 syscall 的情景就可以知道，比起 “自旋”，线程间频繁的抢占以及频繁的创建和销毁操作可能带来的危害会更大。

### 6.4.2.3 scheddetail

如果我们想要更详细的看到调度器的完整信息时，我们可以增加 `scheddetail` 参数，就能够更进一步的查看调度的细节逻辑，如下：

```shell
$ GODEBUG=scheddetail=1,schedtrace=1000 go run main.go
SCHED 1000ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
  P0: status=1 schedtick=2 syscalltick=0 m=3 runqsize=3 gfreecnt=0
  P1: status=1 schedtick=2 syscalltick=0 m=4 runqsize=1 gfreecnt=0
  ...
  M4: p=1 curg=18 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  M3: p=0 curg=22 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  ...
  G1: status=4(semacquire) m=-1 lockedm=-1
  G2: status=4(force gc (idle)) m=-1 lockedm=-1
  G3: status=4(GC sweep wait) m=-1 lockedm=-1
  G17: status=1() m=-1 lockedm=-1
  G18: status=2() m=4 lockedm=-1
  ...
```

在这里我们抽取了 1000ms 时的调试信息来查看，信息量比较大，涉及到了核心的 GMP，我们先从每一个字段开始了解。

#### 6.4.2.3.1 G

-   status：G 的运行状态。
-   m：隶属哪一个 M。
-   lockedm：是否有锁定 M。

在第一点中我们有提到 G 的运行状态，这对于分析内部流转非常的有用，共涉及如下 9 种状态：

```
状态                          值                                     含义

_Gidle      0               刚刚被分配，还没有进行初始化。

_Grunnable  1              已经在运行队列中，还没有执行用户代码。

_Grunning   2       不在运行队列里中，已经可以执行用户代码，此时已经分配了 M 和 P。

_Gsyscall  3     正在执行系统调用，此时分配了 M。

_Gwaiting   4   在运行时被阻止，没有执行用户代码，也不在运行队列中，此时它正在某处阻塞等待中。

_Gmoribund_unused   5    尚未使用，但是在 gdb 中进行了硬编码。

_Gdead   6  尚未使用，这个状态可能是刚退出或是刚被初始化，此时它并没有执行用户代码，有可能有也有可能没有分配堆栈。

_Genqueue_unused  7   尚未使用。

_Gcopystack    8   正在复制堆栈，并没有执行用户代码，也不在运行队列中。
```

在理解了各类的状态的意思后，我们结合上述案例看看，如下：

```shell
G1: status=4(semacquire) m=-1 lockedm=-1
G2: status=4(force gc (idle)) m=-1 lockedm=-1
G3: status=4(GC sweep wait) m=-1 lockedm=-1
G17: status=1() m=-1 lockedm=-1
G18: status=2() m=4 lockedm=-1
```

在这个片段中，G1 的运行状态为 `_Gwaiting`，并没有分配 M 和锁定。这时候你可能好奇在片段中括号里的是什么东西呢，其实是因为该 `status=4` 是表示 `Goroutine` 在**运行时时被阻止**，而阻止它的事件就是 `semacquire` 事件，是因为 `semacquire` 会检查信号量的情况，在合适的时机就调用 `goparkunlock` 函数，把当前 `Goroutine` 放进等待队列，并把它设为 `_Gwaiting` 状态。

那么在实际运行中还有什么原因会导致这种现象呢，我们一起看看，如下：

```go
	waitReasonZero                                    // ""
	waitReasonGCAssistMarking                         // "GC assist marking"
	waitReasonIOWait                                  // "IO wait"
	waitReasonChanReceiveNilChan                      // "chan receive (nil chan)"
	waitReasonChanSendNilChan                         // "chan send (nil chan)"
	waitReasonDumpingHeap                             // "dumping heap"
	waitReasonGarbageCollection                       // "garbage collection"
	waitReasonGarbageCollectionScan                   // "garbage collection scan"
	waitReasonPanicWait                               // "panicwait"
	waitReasonSelect                                  // "select"
	waitReasonSelectNoCases                           // "select (no cases)"
	waitReasonGCAssistWait                            // "GC assist wait"
	waitReasonGCSweepWait                             // "GC sweep wait"
	waitReasonChanReceive                             // "chan receive"
	waitReasonChanSend                                // "chan send"
	waitReasonFinalizerWait                           // "finalizer wait"
	waitReasonForceGGIdle                             // "force gc (idle)"
	waitReasonSemacquire                              // "semacquire"
	waitReasonSleep                                   // "sleep"
	waitReasonSyncCondWait                            // "sync.Cond.Wait"
	waitReasonTimerGoroutineIdle                      // "timer goroutine (idle)"
	waitReasonTraceReaderBlocked                      // "trace reader (blocked)"
	waitReasonWaitForGCCycle                          // "wait for GC cycle"
	waitReasonGCWorkerIdle                            // "GC worker (idle)"
```

我们通过以上 `waitReason` 可以了解到 `Goroutine` 会被暂停运行的原因要素，也就是会出现在括号中的事件。

#### 6.4.2.3.2 M

-   p：隶属哪一个 P。
-   curg：当前正在使用哪个 G。
-   runqsize：运行队列中的 G 数量。
-   gfreecnt：可用的 G（状态为 Gdead）。
-   mallocing：是否正在分配内存。
-   throwing：是否抛出异常。
-   preemptoff：不等于空字符串的话，保持 curg 在这个 m 上运行。

#### 6.4.2.3.3 P

-   status：P 的运行状态。
-   schedtick：P 的调度次数。
-   syscalltick：P 的系统调用次数。
-   m：隶属哪一个 M。
-   runqsize：运行队列中的 G 数量。
-   gfreecnt：可用的 G（状态为 Gdead）。

```
状态        值         含义

_Pidle   0  刚刚被分配，还没有进行进行初始化。

_Prunning  1  当 M 与 P 绑定调用 acquirep 时，P 的状态会改变为 _Prunning。

_Psyscall  2   正在执行系统调用。

_Pgcstop  3   暂停运行，此时系统正在进行 GC，直至 GC 结束后才会转变到下一个状态阶段。

_Pdead   4  废弃，不再使用。
```

## 6.4.3 小结

通过本文我们学习到了调度的一些基础知识，再通过神奇的 GODEBUG 掌握了观察调度器的方式方法，你想想，是不是可以和我前面章节中的 `go tool trace` 工具来结合使用呢，在实际的使用中，类似的办法有很多，组合巧用是重点。



https://github.com/Shitaibin/shitaibin.github.io/blob/hexo_resource/source/_posts/golang-scheduler-3-principle-with-graph.md
