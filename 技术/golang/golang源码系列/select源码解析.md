
具体源码在 runtime/select.go下面，只有500+行代码

**select的几大特点**  
1.可以实现两种收发操作，阻塞收发和非阻塞收发  
2.当多个case ready的情况下会随机选择一个执行，不是顺序执行  
3.没有ready的case时，有default语句，执行default语句;没有default语句，阻塞直到某个case ready  
4.select中的case必须是channel操作  
5.default语句总是可运行的

**select 的几大用处**  
1.超时处理  
一旦超时就返回，不再等待

```
func main() {     
	c := boring("Joe")     
	timeout := time.After(5 * time.Second)     
	for {         
		select {         
			case s := <-c:             
				fmt.Println(s)         
			case <-timeout:             
				fmt.Println("You talk too much.")             
				return         
		}     
	} 
}
```

2.生产，消费者通信  
消费者消费达到一定条件不再需要数据的时候，发送一个quit信号；生产者用select语句，一个case生产数据，一个case监听quit消息，收到消费者的停止信号，就会跳出select语句，不再生产数据  
消费者
```go
// 创建 quit channel 
quit := make(chan string) 
// 启动生产者 goroutine 
c := boring("Joe", quit) 
// 从生产者 channel 读取结果 
for i := rand.Intn(10); i >= 0; i-- { 
	fmt.Println(<-c) 
} 
// 通过 quit channel 通知生产者停止生产 
quit <- "Bye!" 
fmt.Printf("Joe says: %q\n", <-quit)
```

生产者
```go
select { 
	case c <- fmt.Sprintf("%s: %d", msg, i):     // do nothing 
	case <-quit:     
		cleanup()     
		quit <- "See you!"     
		return 
}
```

3.非阻塞读写  
有时候只是希望看一下当前有没有数据，如果没有也不希望继续阻塞下去，加default语句，如果有数据就返回，没有就走default语句,比如有错误的话会往C通道里面塞数据，如果执行select的时候有的话就捕捉到，没有就走default逻辑。这里只是想看有没有err，多少个都无所谓

```go
select { 
	case m <- c:     // print something 
	case default:     // print something else 
}
```

**具体源码分析**

select 中的case用runtime.scase结构体来表示，具体如下
```go
type scase struct { 	
	c           *hchan         // 除了default其他都是channel操作，所以需要一个channel变量存储通道信息 	
	elem        unsafe.Pointer // data element 	
	kind        uint16//case类型 default语句是caseDefault类型，接收通道是caseRecv，发送通道是caseSend 	
	pc          uintptr // race pc (for race detector / msan) 	
	releasetime int64 
}
```

通道类型具体代码定义如下
```go
const ( 	
		caseNil = iota 	
		caseRecv 	
		caseSend 	
		caseDefault 
)
```

编译器在中间代码生成期间会根据 select 中 case 的不同对控制语句进行优化，这一过程都发生在 cmd/compile/internal/gc.walkselectcases 函数中，比如select没有case的情况会把当前goroutine挂起，把处理器的使用权让出去，导致Goroutine 进入无法被唤醒的永久休眠状态，这种具体特殊处理目前不关注，下面重点看常规流程 常规编译之后，调用方法runtime.selectgo，具体操作都在这个方法里面，传入的参数是scase数组，传出的参数是随机选择的ready的scase下标，如下
```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) { 	
	if debugSelect { 		
		print("select: cas0=", cas0, "\n") 	
	} 
	...
```

方法中具体分为三个部分

**一.初始化**  
确定case的轮询顺序 pollOrder 和加锁顺序 lockOrder

通过 runtime.fastrandn 函数，打乱case的访问顺序
```go
// generate permuted order 	
for i := 1; i < ncases; i++ { 		
	j := fastrandn(uint32(i + 1)) 		
	pollorder[i] = pollorder[j] 		
	pollorder[j] = uint16(i) 	
}
```

下面步骤2访问的时候就是按照乱序访问的，这也就是为什么多个case ready随机执行一个的原因
```go
loop: 
	// pass 1 - look for something already waiting 
	var dfli int 
	var dfl *scase 
	var casi int 
	var cas *scase 
	var recvOK bool 
	for i := 0; i < ncases; i++ { 
		casi = int(pollorder[i])//pollorder返回的是case数组下标，已经打乱，所以随机找某一个case，如果ready直接执行这一个case了 
		cas = &scases[casi] 
		c = cas.c 
		switch cas.kind { 
			case caseNil: 
				continue 
			case caseRecv: 
				sg = c.sendq.dequeue() 
				if sg != nil {
```

加锁顺序lockOrder具体是用一个简单的堆排序对channel地址排序实现，为了在读数据时能顺序给channel加锁，和去重防止对同一个channel多次加锁
```go
for i := 0; i < ncases; i++ { 		
	j := i 		
	// Start with the pollorder to permute cases on the same channel. 		
	c := scases[pollorder[i]].c 		
	for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() { 			
		k := (j - 1) / 2 			
		lockorder[j] = lockorder[k] 			
		j = k 		
	} 		
	lockorder[j] = pollorder[i] 	
} 	
for i := ncases - 1; i >= 0; i-- { 		
	o := lockorder[i] 		
	c := scases[o].c 		
	lockorder[i] = lockorder[0] 		
	j := 0 		
	for { 			
		k := j*2 + 1 			
		if k >= i { 				
			break 			
		} 			
		if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
		 				k++ 			
		 } 			
		 if c.sortkey() < scases[lockorder[k]].c.sortkey() { 				
			 lockorder[j] = lockorder[k] 				
			 j = k 				
			 continue 			
		} 			
		break 		
		} 		
		lockorder[j] = o 	
}
```

**二.主循环**

1.首先在for循环中对case进行遍历，查看是否ready，已经ready就直接跳到处理部分，流程就结束
```go
// lock all the channels involved in the select 	
sellock(scases, lockorder)//读写channel前会对所有的channel加锁 	
var ( 		
gp     *g 		
sg     *sudog 		
c      *hchan 		
k      *scase 		
sglist *sudog 		
sgnext *sudog 		
qp     unsafe.Pointer 		
nextp  **sudog 	) 
loop: 	
	// pass 1 - look for something already waiting 	
	var dfli int 	
	var dfl *scase 	
	var casi int 	
	var cas *scase 	
	var recvOK bool 	
	for i := 0; i < ncases; i++ { 		
		casi = int(pollorder[i])//随机找到一个case 		
		cas = &scases[casi] 		
		c = cas.c 		
		switch cas.kind { 		
			case caseNil: 			
				continue//空的就跳过 		
			case caseRecv: 			
				sg = c.sendq.dequeue() 			
				if sg != nil { 				
				goto recv//接收通道的发送队列里有等待的goroutine直接跳到recv处理，其实和channel之前的实现一样，把通道buff的取出来，把队列的goroutine内容复制到buff当前位置，然后释放队列的goroutine，具体看channel源码剖析的文章 			
				} 			
				if c.qcount > 0 { 				
				goto bufrecv//如果没有等待对列，buff有值，直接从buff复制 			
				} 			
				if c.closed != 0 {//没有数据，如果关闭了，做一下清除数据的工作 				
				goto rclose 			
				} 		
			case caseSend: 			
				if raceenabled { 				
				racereadpc(c.raceaddr(), cas.pc, chansendpc) 			
				} 			
				if c.closed != 0 {//发送通道，关闭直接panic 				
				goto sclose 			
				} 			
				sg = c.recvq.dequeue()//直接复制给等待的接收通道，并唤醒 			
				if sg != nil { 				
				goto send 			
				} 			
				if c.qcount < c.dataqsiz {//没有等待通道，buff还有空间，放在buff里面 				
					goto bufsend 			
				} 		
			case caseDefault: 			
				dfli = casi 			
				dfl = cas 		
		} 
	} 	
	if dfl != nil {//如果default有，直接执行default语句，不再阻塞 		
		selunlock(scases, lockorder)//解锁所有channel 		
		casi = dfli 		
		cas = dfl 		
		goto retc 	
	}
```

上面的操作就解释了，如果有ready的case就随机执行一个，没有的基础上如果有default，执行default语句

2.挂起  
上面没有结束，说明通道都没有ready，且没有default语句，所以把当前goroutine挂到每一个通道的等待队列中等待唤醒
```go
// pass 2 - enqueue on all chans 	
gp = getg() 	
if gp.waiting != nil { 		
throw("gp.waiting != nil") 	
} 	
nextp = &gp.waiting 	
for _, casei := range lockorder { 		
	casi = int(casei) 		
	cas = &scases[casi] 		
	if cas.kind == caseNil { 			
	continue 		
	} 		
	c = cas.c 		
	sg := acquireSudog()//把goroutine封装成sugod形式，放在channel等待队列 		
	sg.g = gp 		
	sg.isSelect = true 		// No stack splits between assigning elem and enqueuing 		
	// sg on gp.waiting where copystack can find it. 		
	sg.elem = cas.elem 		
	sg.releasetime = 0 		
	if t0 != 0 { 			
		sg.releasetime = -1 		
	} 		
	sg.c = c 		// Construct waiting list in lock order. 		
	*nextp = sg 		
	nextp = &sg.waitlink 		
	switch cas.kind { 		
	case caseRecv: 			
		c.recvq.enqueue(sg)//放在等待队列中 		
	case caseSend: 			
		c.sendq.enqueue(sg) 		
	} 	
} 	
// wait for someone to wake us up 	
gp.param = nil 	
gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)//goroutine挂起，等待唤醒 	
gp.activeStackChans = false 	
sellock(scases, lockorder) 	
gp.selectDone = 0 	
sg = (*sudog)(gp.param) 	
gp.param = nil
```

3.唤醒 有channel准备好了，当前 Goroutine 就会被调度器唤醒，返回当前case，其他case中通道队列移除该goroutine，不再等待
```go
// pass 3 - dequeue from unsuccessful chans 	
// otherwise they stack up on quiet channels 	
// record the successful case, if any. 	
// We singly-linked up the SudoGs in lock order. 	
casi = -1 	
cas = nil 	
sglist = gp.waiting 	
// Clear all elem before unlinking from gp.waiting. 	
for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink { 		
	sg1.isSelect = false 		
	sg1.elem = nil 		
	sg1.c = nil 	
} 	
gp.waiting = nil 	
for _, casei := range lockorder { 		
	k = &scases[casei] 		
	if k.kind == caseNil { 			continue 		} 		
	if sglist.releasetime > 0 { 			
		k.releasetime = sglist.releasetime 		
	} 		
	if sg == sglist { 			
	// sg has already been dequeued by the G that woke us up. 			
		casi = int(casei)//找到唤醒的case 			
		cas = k 		
	} else { 			
		c = k.c 			
		if k.kind == caseSend {//其他case从通道的等待队列剔除，这个goroutine已经在上面的case等到数据，不用等其他case了 				
		c.sendq.dequeueSudoG(sglist) 			
	} else { 				
		c.recvq.dequeueSudoG(sglist) 			
	} 		
} 		
sgnext = sglist.waitlink 		
sglist.waitlink = nil 		
releaseSudog(sglist) 		
sglist = sgnext 	
} 	
if cas == nil { 		// We can wake up with gp.param == nil (so cas == nil) 		
// when a channel involved in the select has been closed. 		
// It is easiest to loop and re-run the operation; 		
// we'll see that it's now closed. 		
// Maybe some day we can signal the close explicitly, 		
// but we'd have to distinguish close-on-reader from close-on-writer. 		
// It's easiest not to duplicate the code and just recheck above. 		
// We know that something closed, and things never un-close, 		
// so we won't block again. 		
goto loop 	
} 	
c= cas.c ...
```

select 关键字是 Go 语言特有的控制结构，它的实现原理比较复杂，需要编译器和运行时函数的通力合作

##### 如何随机选择 case

**`selectgo` 是通过循环 `scases` 来挑选可以收发的 channel** **然而循环时并不是按照 `scases`的顺序，而是 `pollorder` 中记录的顺序, 这样可以避免 channel 的饥饿问题**

**为了保证 `select` 随机选择 case，所以使用 `fastrandn` 来生成随机数**

```go
for i := 1; i < ncases; i++ {        
	j := fastrandn(uint32(i+1))        
	pollorder[i] = pollorder[j]        
	pollorder[j] = uint16(i)    
}
```

`pollorder` 在开始的时候值都是 0，循环结束后值便是随机顺序的 scases 索引

##### 避免相同 channel 重复加锁，以及死锁问题

**`selectgo` 在查找 scases 中已经可以进行收发操作的 channel 前会先对所有的 channel 进行加锁操作**

###### 死锁问题

> 如果多个 goroutine 都需要锁定 ch1 ch2，而他们加锁的顺序不固定，那么很可能会出现死锁问题这个时候，对加锁的顺序就有要求了，按照同样的顺序的话，没有竞争到 ch1.lock 的 goroutine,会等待加锁 ch1.lcok，而不会直接去加锁 ch2.lock

**加锁前首先会对 `lockorder` 进行堆排序，生成由 case.c(*hchan) 来排序的 scases 索引顺序**

```go
func selectgo(cas0 *scase, order0 *uint16, ncase int)(int, bool) {    
...    
// ... 对 looporder 堆排序     
// selectgo 在查找 scases 前，先对所有 channel 加锁    
sellock(scases, lockorder)    
...}
```

**`sellock` 对地址相同的 channel 只会加锁一次**

```go
func sellock(scases []scases, lockorder []int16) {    
var c *hchan    
for _, o := range lockorder {        
c0 := scases[0].c // 根据加锁顺序获取 case         
// c 记录了上次加锁的 hchan 地址，如果和当前 *hchan 相同，那么就不会再次加锁        
if c0 != nil && c0 != c {            
c = c0            
lock(&c.lock)        
}    }}
```

加锁完成后，可以进入 `selectgo` 主循环逻辑了 主逻辑会分为三部分：

1. 首先根据 `pollorder` 的顺序查找 scases 是否有可以立即收发的 channel
2. channel 都没有准备好，并且不存在 default，那么就将当前 `goroutine` 加入到 channel 相应的等待队列，然后等待收其他 `goroutine` 唤醒
3. 被唤醒后，再次找到满足条件的 channel

## 总结

到这一节的最后我们需要总结一下，`select` 结构的执行过程与实现原理，首先在编译期间，Go 语言会对 `select` 语句进行优化，以下是根据 `select` 中语句的不同选择了不同的优化路径：

1. 空的 `select` 语句会被直接转换成 `block` 函数的调用，直接挂起当前 Goroutine；
2. 如果 `select` 语句中只包含一个 `case`，就会被转换成 `if ch == nil { block }; n;` 表达式；
    - 首先判断操作的 Channel 是不是空的；
    - 然后执行 `case` 结构中的内容；
3. 如果 `select` 语句中只包含两个 `case` 并且其中一个是 `default`，那么 Channel 和接收和发送操作都会使用 `selectnbrecv` 和 `selectnbsend` 非阻塞地执行接收和发送操作；
4. 在默认情况下会通过 `selectgo` 函数选择需要执行的 `case` 并通过多个 `if` 语句执行 `case` 中的表达式；

在编译器已经对 `select` 语句进行优化之后，Go 语言会在运行时执行编译期间展开的 `selectgo` 函数，这个函数会按照以下的过程执行：

1. 随机生成一个遍历的轮询顺序 `pollOrder` 并根据 Channel 地址生成一个用于遍历的锁定顺序 `lockOrder`；
2. 根据 `pollOrder` 遍历所有的 `case` 查看是否有可以立刻处理的 Channel 消息；
    1. 如果有消息就直接获取 `case` 对应的索引并返回；
3. 如果没有消息就会创建 `sudog` 结构体，将当前 Goroutine 加入到所有相关 Channel 的 `sendq` 和 `recvq` 队列中并调用 `gopark` 触发调度器的调度；
4. 当调度器唤醒当前 Goroutine 时就会再次按照 `lockOrder` 遍历所有的 `case`，从中查找需要被处理的 `sudog` 结构并返回对应的索引；

然而并不是所有的 `select` 控制结构都会走到 `selectgo` 上，很多情况都会被直接优化掉，没有机会调用 `selectgo` 函数。

Go 语言中的 `select` 关键字与 IO 多路复用中的 `select`、`epoll` 等函数非常相似，不但 Channel 的收发操作与等待 IO 的读写能找到这种一一对应的关系，这两者的作用也非常相似；总的来说，`select` 关键字的实现原理稍显复杂，与 [Channel](https://link.juejin.cn?target=https%3A%2F%2Fdraveness.me%2Fgolang-channel "https://draveness.me/golang-channel") 的关系非常紧密，这里省略了很多 Channel 操作的细节，数据结构一章其实就介绍了 [Channel](https://link.juejin.cn?target=https%3A%2F%2Fdraveness.me%2Fgolang-channel "https://draveness.me/golang-channel") 收发的相关细节。
