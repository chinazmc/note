---
sr-due: 2023-06-10
sr-interval: 263
sr-ease: 250
---

#golang  #note 

我在看一些优秀的开源库的时候看到一个有意思的缓存库 fastcache，在它的介绍主要有以下几点特点：

1.  读写数据要快，即使在并发下；
2.  即使在数 GB 的缓存中，也要保持很好的性能，以及尽可能减少 GC 次数；
3.  设计尽可能简单；

本文会通过模仿它写一个简单的缓存库，从而研究其内核是如何实现这样的目标的。希望各位能有所收获。

# 设计思想

在项目中，我们经常会用到 Go 缓存库比如说 `patrickmn/go-cache`库。但很多缓存库其实都是用一个简单的 Map 来存放数据，这些库在使用的时候，当并发低，数据量少的时候是没有问题的，但是在数据量比较大并发比较高的时候会延长 GC 时间，增加内存分配次数。

比如，我们使用一个简单的例子：
```go
func main() { 	
	a := make(map[string]string, 1e9)  	
	for i := 0; i < 10; i++ { 		
		runtime.GC() 	
	}  	
	runtime.KeepAlive(a) 
}
```

在这个例子中，预分配了大小是`10亿（1e9)` 的 map，然后我们通过 `gctrace` 输出一下 GC 情况：

> 做实验的环境是 Linux，机器配置是 16C 8G ，想要更深入理解 GC，可以看这篇：《 Go语言GC实现原理及源码分析 [https://www.luozhiyun.com/archives/475](https://www.luozhiyun.com/archives/475) 》
![[Pasted image 20220628141601.png]]

上面展示了最后 5 次 GC 的情况，下面我们看看具体的含义是什么：
![[Pasted image 20220628141706.png]]

可以从上面的输出看到每次 GC 处理的时间非常的长，占用的 CPU 资源也非常多。那么造成这样的原因是什么呢？

`string` 实际上底层数据结构是由两部分组成，其中包含指向字节数组的指针和数组的大小：
```go
type StringHeader struct { 	
		Data uintptr 	
		Len  int 
}
```

由于 `StringHeader`中包含指针，所以每次 GC 的时候都会扫描每个指针，那么在这个巨大的 `map`中是包含了非常多的指针的，所以造成了巨大的资源消耗。

在上面的例子 `map a` 中数据大概是这样存储：

![maps](https://img.luozhiyun.com/20210516203637.png)

一个 map 中里面有多个 `bucket` ，`bucket` 里面有一个 `bmap` 数组用来存放数据，但是由于 `key` 和 `value` 都是 `string` 类型的，所以在 GC 的时候还需要根据 `StringHeader`中的 `Data`指针扫描 `string` 数据。

对于这种情况，如果所有的string字节都在一个单一内存片段中，我们就可以通过偏移来追踪某个字符串在这段内存中的开始和结束位置。通过追踪偏移，我们不在需要在我们大数组中存储指针，GC也不在会被困扰。如下：

![maps2](https://img.luozhiyun.com/20210516203645.png)

如同上面所示，如果我们将字符串中的字节数据拷贝到一个连续的字节数组 `chunks` 中，并为这个字节数组提前分配好内存，并且仅存储字符串在数组中的偏移而不是指针。

除了上面所说的优化内容以外，还有其他的方法吗？

其实我们还可以直接从系统 OS 中调用 `mmap syscall` 进行内存分配，这样 GC 就永远不会对这块内存进行内存管理，因此也就不会扫描到它。如下：
```go
func main() {      
		test := "hello syscall" 	
		data, _ := syscall.Mmap(-1, 0, 13, 
								syscall.PROT_READ|syscall.PROT_WRITE, 
								syscall.MAP_ANON|syscall.MAP_PRIVATE) 	
		p := (*[13]byte)(unsafe.Pointer(&data[0]))  	
		for i := 0; i < 13; i++ { 		
				p[i] = test[i] 	
		} 	
		fmt.Println(string(p[:])) 
}
```

通过系统调用直接向 OS 申请了 13bytes 的内存，然后将一个字符串写入到申请的内存数组中。

所以我们也可以通过提前向 OS 申请一块内存，而不是用的时候才申请内存，减少频繁的内存分配从而达到提高性能的目的。

# 源码实战

## API

我们在开发前先把这个库的 API 定义一下：

### func New
```go
func New(maxBytes int) *Cache
```

创建一个 Cache 结构体，传入预设的缓存大小，单位是字节。

### func (*Cache) Get
```go
func (c *Cache) Get(k []byte) []byte
```

获取 Cache 中的值，传入的参数是 byte 数组。

### func (*Cache) Set
```go
func (c *Cache) Set(k, v []byte)
```

设置键值对到缓存中，k 是键，v 是值，参数都是 byte 数组。

## 结构体
```go
const bucketsCount = 512  
type Cache struct {  	
		buckets [bucketsCount]bucket 
}  
type bucket struct { 	
	// 读写锁 	
	mu sync.RWMutex  	
	// 二维数组，存放数据的地方，是一个环形链表 	
	chunks [][]byte  	
	// 索引字典 	
	m map[uint64]uint64  	
	// 索引值 	
	idx uint64  	
	// chunks 被重写的次数，用来校验环形链表中数据有效性 	
	gen uint64 
}
```

通过我们上面的分析，可以看到，实际上真正存放数据的地方是 chunks 二维数组，在实现上是通过 m 字段来映射索引路径，根据 chunks 和 gen 两个字段来构建一个环形链表，环形链表每转一圈 gen 就会加一。

![mycache](https://img.luozhiyun.com/20210516203651.png)

## 初始化
```go
func New(maxBytes int) *Cache {  
   if maxBytes <= 0 {  
      panic(fmt.Errorf("maxBytes must be greater than 0; got %d", maxBytes))  
   }  
   var c Cache  
   // 算出每个桶的大小，加上一个 bucketsCount 相当于有  
   maxBucketBytes := uint64((maxBytes + bucketsCount - 1) / bucketsCount)  
   for i := range c.buckets[:] {  
      c.buckets[i].Init(maxBucketBytes)  
   }  
   return &c  
}
```

我们会设置一个 New 函数来初始化我们 Cache 结构体，在 Cache 结构体中会将缓存的数据大小平均分配到每个桶中，然后对每个桶进行初始化。
```go
const bucketSizeBits = 40 
const maxBucketSize uint64 = 1 << bucketSizeBits 
const chunkSize = 64 * 1024  

func (b *bucket) Init(maxBytes uint64) {  
   if maxBytes == 0 {  
      panic(fmt.Errorf("maxBytes cannot be zero"))  
   }  
   if maxBytes >= maxBucketSize {  
      panic(fmt.Errorf("too big maxBytes=%d; should be smaller than %d", maxBytes, maxBucketSize))  
   }  
   // 初始化 Chunks 大小  
   maxChunks := (maxBytes + chunkSize - 1) / chunkSize  
   b.chunks = make([][]byte, maxChunks)  
   b.m = make(map[uint64]uint64)  
   // 初始化 chunk   
   b.Reset()  
}
```

在这里会将桶里面的内存按 chunk 进行分配，每个 chunk 占用内存约为 64 KB。在最后会调用 bucket 的 Reset 方法对 bucket 结构体进行初始化。
```go
func (b *bucket) Reset() {  
   b.mu.Lock()  
   chunks := b.chunks  
   // 遍历 chunks   
   for i := range chunks {  
      // 将 chunk 中的内存归还到缓存中  
      putChunk(chunks[i])  
      chunks[i] = nil  
   }  
   // 删除索引字典中所有的数据  
   bm := b.m  
   for k := range bm {  
      delete(bm, k)  
   }  
   b.idx = 0  
   b.gen = 1  
   b.mu.Unlock()  
}
```

Reset 方法十分简单，主要就是清空 chunks 数组、删除索引字典中所有的数据以及重置索引 idx 和 gen 的值。

在上面这个方法中有一个 putChunk ，其实这个就是直接操作我们提前向 OS 申请好的内存，相应的还有一个 getChunk 方法。下面我们具体看看 Chunk 的操作。

## Chunk 操作

### getChunk
```go
const chunksPerAlloc = 1024 
const chunkSize = 64 * 1024  
var ( 	
	freeChunks     []*[chunkSize]byte 	
	freeChunksLock sync.Mutex 
)  
func getChunk() []byte {  
   freeChunksLock.Lock()  
   if len(freeChunks) == 0 {  
      // 分配  64 * 1024 * 1024 = 64 MB 内存  
      data, err := syscall.Mmap(-1, 0, chunkSize*chunksPerAlloc, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_ANON|syscall.MAP_PRIVATE)  
      if err != nil {  
         panic(fmt.Errorf("cannot allocate %d bytes via mmap: %s", chunkSize*chunksPerAlloc, err))  
      }  
      // 循环遍历 data 数据  
      for len(data) > 0 {  
         // 将从系统分配的内存分为 64 * 1024 = 64 KB 大小，存放到 freeChunks中  
         p := (*[chunkSize]byte)(unsafe.Pointer(&data[0]))  
         freeChunks = append(freeChunks, p)  
         data = data[chunkSize:]  
      }  
   }  
   //从 freeChunks 获取最后一个元素  
   n := len(freeChunks) - 1  
   p := freeChunks[n]  
   freeChunks[n] = nil  
   freeChunks = freeChunks[:n]  
   freeChunksLock.Unlock()  
   return p[:]  
}
```

初次调用 getChunk 函数时会使用系统调用分配 64MB 的内存，然后循环将内存切成 1024 份，每份 64KB 放入到 freeChunks 空闲列表中。然后获取每次都获取 freeChunks 空闲列表最后一个元素 64KB 内存返回。需要注意的是 getChunk 会下下面将要介绍到的 Cache 的 set 方法中使用到，所以需要考虑到并发问题，所以在这里加了锁。

### putChunk
```go
func putChunk(chunk []byte) {  
   if chunk == nil {  
      return  
   }  
   chunk = chunk[:chunkSize]  
   p := (*[chunkSize]byte)(unsafe.Pointer(&chunk[0]))  
  
   freeChunksLock.Lock()  
   freeChunks = append(freeChunks, p)  
   freeChunksLock.Unlock()  
}
```

putChunk 函数就是将内存数据还回到 freeChunks 空闲列表中，会在 bucket 的 Reset 方法中被调用。

## Set
```go
const bucketsCount = 512  
func (c *Cache) Set(k, v []byte) { 	
	h := xxhash.Sum64(k) 	
	idx := h % bucketsCount 	
	c.buckets[idx].Set(k, v, h) 
}
```
Set 方法里面会根据 k 的值做一个 hash，然后取模映射到 buckets 桶中，这里用的 hash 库是 `cespare/xxhash`。

最主要的还是 buckets 里面的 Set 方法：
```go
func (b *bucket) Set(k, v []byte, h uint64) {  
   // 限定 k v 大小不能超过 2bytes   
	  if len(k) >= (1<<16) || len(v) >= (1<<16) {  
		  return  
	  }  
   // 4个byte 设置每条数据的数据头  
   var kvLenBuf [4]byte  
   kvLenBuf[0] = byte(uint16(len(k)) >> 8)  
   kvLenBuf[1] = byte(len(k))  
   kvLenBuf[2] = byte(uint16(len(v)) >> 8)  
   kvLenBuf[3] = byte(len(v))  
   kvLen := uint64(len(kvLenBuf) + len(k) + len(v))  
   // 校验一下大小  
   if kvLen >= chunkSize {  
      return  
   }  
  
   b.mu.Lock()  
   // 当前索引位置  
   idx := b.idx  
   // 存放完数据后索引的位置  
   idxNew := idx + kvLen  
   // 根据索引找到在 chunks 的位置  
   chunkIdx := idx / chunkSize  
   chunkIdxNew := idxNew / chunkSize  
   // 新的索引是否超过当前索引  
   // 因为还有chunkIdx等于chunkIdxNew情况，所以需要先判断一下  
   if chunkIdxNew > chunkIdx {  
      // 校验是否新索引已到chunks数组的边界  
      // 已到边界，那么循环链表从头开始  
      if chunkIdxNew >= uint64(len(b.chunks)) {  
         idx = 0  
         idxNew = kvLen  
         chunkIdx = 0  
         b.gen++  
         // 当 gen 等于 1<<genSizeBits时，才会等于0  
         // 也就是用来限定 gen 的边界为1<<genSizeBits  
         if b.gen&((1<<genSizeBits)-1) == 0 {  
            b.gen++  
         }  
      } else {  
         // 未到 chunks数组的边界,从下一个chunk开始  
         idx = chunkIdxNew * chunkSize  
         idxNew = idx + kvLen  
         chunkIdx = chunkIdxNew  
      }  
      // 重置 chunks[chunkIdx]      
      b.chunks[chunkIdx] = b.chunks[chunkIdx][:0]  
   }  
   chunk := b.chunks[chunkIdx]  
   if chunk == nil {  
      chunk = getChunk()  
      // 清空切片  
      chunk = chunk[:0]  
   }  
   // 将数据 append 到 chunk 中  
   chunk = append(chunk, kvLenBuf[:]...)  
   chunk = append(chunk, k...)  
   chunk = append(chunk, v...)  
   b.chunks[chunkIdx] = chunk  
   // 因为 idx 不能超过bucketSizeBits，所以用一个 uint64 同时表示gen和idx  
   // 所以高于bucketSizeBits位置表示gen  
   // 低于bucketSizeBits位置表示idx  
   b.m[h] = idx | (b.gen << bucketSizeBits)  
   b.idx = idxNew  
   b.mu.Unlock()  
}
```

1.  在这段代码开头实际上我会限制键值的大小不能超过 2bytes；
2.  然后将 2bytes 大小长度的键值封装到 4bytes 的 kvLenBuf 作为数据头，数据头和键值的总长度是不能超过一个 chunk 长度，也就是 `64 * 1024`；
3.  然后计算出原索引 chunkIdx 和新索引 chunkIdxNew，用来判断这次添加的数据加上原来的数据有没有超过一个 chunk 长度；
4.  根据新的索引找到对应的 chunks 中的位置，然后将键值以及 kvLenBuf 追加到 chunk 后面；
5.  设置新的 idx 以及 m 字典对应的值，m 字典中存放的是 gen 和 idx 通过取与的放置存放。

在 Set 一个键值对会有 4bytes 的 kvLenBuf 作为数据头，后面的数据会接着 key 和 value ，在 kvLenBuf 中，前两个 byte 分别代表了 key 长度的低位和高位；后两个 byte 分别代表了 value 长度的低位和高位，数据图大致如下：

![mycache2](https://img.luozhiyun.com/20210516203659.png)

下面举个例子来看看是是如何利用 chunks 这个二维数组来实现环形链表的。

我们在 bucket 的 Init 方法中会根据传入 maxBytes 桶字节数来设置 chunks 的长度大小，由于每个 chunk 大小都是 `64 * 1024`bytes，那么我们设置 `3 * 64 * 1024`bytes 大小的桶，那么 chunks 数组长度就为 3。

如果当前算出 chunkIdx 在 chunks 数组为 1 的位置，并且在 chunks[1] 的位置中，还剩下 6bytes 未被使用，那么有如下几种情况：

1.  现在假设放入的键值长度都是 1byte，那么在 chunks[1] 的位置中剩下的 6bytes 刚好可以放下；

![mycache3](https://img.luozhiyun.com/20210516203704.png)

2.  现在假设放入的键值长度超过了 1byte，那么在 chunks[1] 的位置中剩下的位置就放不下，只能放入到 chunks[2] 的位置中。

![mycache4](https://img.luozhiyun.com/20210516203710.png)

如果当前算出 chunkIdx 在 chunks 数组为 2 的位置，并且现在 Set 一个键值，经过计算 chunkIdxNew 为 3，已经超过了 chunks 数组长度，那么会将索引重置，重新将数据从 chunks[0] 开始放置，并将 gen 加一，表示已经跑完一圈了。

![mycache5](https://img.luozhiyun.com/20210516203717.png)

## Get
```go
func (c *Cache) Get(dst, k []byte) []byte {    
	h := xxhash.Sum64(k)    
	idx := h % bucketsCount    
	dst, _ = c.buckets[idx].Get(dst, k, h, true)    
	return dst 
}
```

这里和 Set 方法是一样的，首先是要找到对应的桶的位置，然后才去桶里面拿数据。需要注意的是，这里的 dst 可以从外部传入一个切片，以达到减少重复分配返回值。
```go
func (b *bucket) Get(dst, k []byte, h uint64, returnDst bool) ([]byte, bool) {  
   found := false  
   b.mu.RLock()  
   v := b.m[h]  
   bGen := b.gen & ((1 << genSizeBits) - 1)  
   if v > 0 {  
      // 高于bucketSizeBits位置表示gen  
      gen := v >> bucketSizeBits  
      // 低于bucketSizeBits位置表示idx  
      idx := v & ((1 << bucketSizeBits) - 1)  
      // 这里说明chunks还没被写满  
      if gen == bGen && idx < b.idx ||  
         // 这里说明chunks已被写满，并且当前数据没有被覆盖  
         gen+1 == bGen && idx >= b.idx ||  
         // 这里是边界条件gen已是最大，并且chunks已被写满bGen从1开始，，并且当前数据没有被覆盖  
         gen == maxGen && bGen == 1 && idx >= b.idx {  
         chunkIdx := idx / chunkSize  
         // chunk 索引位置不能超过 chunks 数组长度  
         if chunkIdx >= uint64(len(b.chunks)) {  
            goto end  
         }  
         // 找到数据所在的 chunk         
         chunk := b.chunks[chunkIdx]  
         // 通过取模找到该key 对应的数据在 chunk 中的位置  
         idx %= chunkSize  
         if idx+4 >= chunkSize {  
            goto end  
         }  
         // 前 4bytes 是数据头  
         kvLenBuf := chunk[idx : idx+4]  
         // 通过数据头算出键值的长度  
         keyLen := (uint64(kvLenBuf[0]) << 8) | uint64(kvLenBuf[1])  
         valLen := (uint64(kvLenBuf[2]) << 8) | uint64(kvLenBuf[3])  
         idx += 4  
         if idx+keyLen+valLen >= chunkSize {  
            goto end  
         }  
         // 如果键值是一致的，表示找到该数据  
         if string(k) == string(chunk[idx:idx+keyLen]) {  
            idx += keyLen  
            // 返回该键对应的值  
            if returnDst {  
               dst = append(dst, chunk[idx:idx+valLen]...)  
            }  
            found = true  
         }  
      }  
   }  
end:  
   b.mu.RUnlock()  
   return dst, found  
}
```

Get 方法主要是考虑环形链表的边界问题。我们在 Set 方法中会将每一个 key 对应的 gen 和 idx 索引存放到 m 字典中，所以我们通过 hash 获取 m 字典的值之后通过位运算就可以获取到 gen 和 idx 索引。

找到 gen 和 idx 索引之后就是边界条件的判断了，用一个 if 条件来进行判断：
`gen == bGen && idx < b.idx`

这里是判断如果是在环形链表的同一次循环中，那么 key 对应的索引应该小于当前桶的索引；
`gen+1 == bGen && idx >= b.idx`

这里表示当前桶已经进入到下一个循环中，所以需要判断 key 对应的索引是不是大于当前索引，以表示当前 key 对应的值没有被覆盖；
`gen == maxGen && bGen == 1 && idx >= b.idx`

因为 gen 和 idx 索引要塞到 uint64 类型的字段中，所以留给 gen 的最大值只有 `maxGen = 1<< 24 -1`，超过了 maxGen 会让 gen 从1开始。所以这里如果 key 对应 gen 等于 maxGen ，那么当前的 bGen 应该等于1，并且 key 对应的索引还应该大于当前 idx，这样才这个键值对才不会被覆盖。

判断完边界条件之后就会找到对应的 chunk ，然后取模后找到数据位置，通过偏移量找到并取出值。

![mycache6](https://img.luozhiyun.com/20210516203722.png)

## Benchmark

下面我上一下过后的 Benchmark：

> 代码位置： [https://github.com/devYun/mycache/blob/main/cache_timing_test.go](https://github.com/devYun/mycache/blob/main/cache_timing_test.go)
![[Pasted image 20220628150041.png]]

上面的测试是 GoCache、BigCache、Map、sync.Map 的情况。下面是本篇文章中所开发的缓存库的测试：
![[Pasted image 20220628150049.png]]

可以看到内存分配是几乎就不存在，操作速度在上面的库中也是佼佼者的存在。

# 总结

在本文中根据其他缓存库，并分析了如果用 Map 作为缓存所存在的问题，然后引出存在这个问题的原因，并提出解决方案；在我们的缓存库中，第一是通过使用索引加内存块的方式来存放缓存数据，再来是通过 OS 系统调用来进行内存分配让我们的缓存数据块脱离了 GC 的控制，从而做到降低 GC 频率提高并发的目的。

其实不只是缓存库，在我们的项目中当遇到需要使用大量的带指针的数据结构并需要长时间保持引用的时候，也是需要注意这样做可能会引发 GC 问题，从而给系统带来隐患。

# Reference

[https://github.com/VictoriaMetrics/fastcache](https://github.com/VictoriaMetrics/fastcache)

Further Dangers of Large Heaps in Go [https://syslog.ravelin.com/further-dangers-of-large-heaps-in-go-7a267b57d487](https://syslog.ravelin.com/further-dangers-of-large-heaps-in-go-7a267b57d487)

Avoiding high GC overhead with large heaps [https://blog.gopheracademy.com/advent-2018/avoid-gc-overhead-large-heaps/](https://blog.gopheracademy.com/advent-2018/avoid-gc-overhead-large-heaps/)

Go 的 GC 如何调优？[https://www.bookstack.cn/read/qcrao-Go-Questions/spilt.14.GC-GC.md](https://www.bookstack.cn/read/qcrao-Go-Questions/spilt.14.GC-GC.md)

https://www.cnblogs.com/luozhiyun/p/14869125.html
https://github.com/devYun/mycache