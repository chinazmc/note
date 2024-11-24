#golang 

本篇文章会从使用方式和源码角度剖析 sync.Map。不过不管是日常开发还是开源项目中，好像 sync.Map 并没有得到很好的利用，大家还是习惯使用 Mutex + Map 来使用。

下面这段代码，看起来很有道理，其实是用错了（背景：并发场景中获取注册信息）。

```go
instance, ok := instanceMap[name]
if ok {
    return instance, nil
}

initLock.Lock()
defer initLock.Unlock()

// double check
instance, ok = instanceMap[name]
if ok {
    return instance, nil
}

```

这里使用使用 sync.Map 会更合理些，因为 sync.Map 底层完全包含了这个逻辑。可能写 Java 的同学看着上面这段代码很眼熟，但确实是用错了，关于为什么用错了以及会造成什么影响，请大家关注后续的文章。

我大概分析了下大家宁愿使用 Mutex + Map，也不愿使用 sync.Map 的原因：

1.  sync.Map 本身就很难用，使用起来并不像一个 Map。失去了 map 应有的特权语法，如：make, map[1] 等
2.  sync.Map 方法较多。让一个简单的 Map 使用起来有了较高的学习成本。

不管什么样的原因吧，当你读过这篇文章后，在某些特定的并发场景下，建议使用 sync.Map 代替 Map + Mutex 的。

## 用法全解

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var syncMap sync.Map
	syncMap.Store("11", 11)
	syncMap.Store("22", 22)

	fmt.Println(syncMap.Load("11")) // 11
	fmt.Println(syncMap.Load("33")) // 空

	fmt.Println(syncMap.LoadOrStore("33", 33)) // 33
	fmt.Println(syncMap.Load("33")) // 33
	fmt.Println(syncMap.LoadAndDelete("33")) // 33
	fmt.Println(syncMap.Load("33")) // 空

	syncMap.Range(func(key, value interface{}) bool {
		fmt.Printf("key:%v value:%v\n", key, value)
		return true
	})
    // key:22 value:22
	// key:11 value:11
}

```

其实 sync.Map 并不复杂，只是将普通 map 的相关操作转成对应函数而已。

![[Pasted image 20220512115824.png]]

sync.Map 两个特有的函数，不过从字面就能理解是什么意思了。 LoadOrStore：sync.Map 存在就返回，不存在就插入 LoadAndDelete：sync.Map 获取某个 key，如果存在的话，同时删除这个 key

## 源码解析

```go
type Map struct {
	mu Mutex
	read atomic.Value // readOnly  read map
	dirty map[interface{}]*entry  // dirty map
	misses int
}

```

![sync map 全景图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e43e8e6d4d5c45e4bd583828ebe90c9a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![Load](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0b2ce0bc920486684e9c8be65bd046c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![Store](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/372873ca13694fa88db13b17f01f05e3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![Delete](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2eb152ca82e3448ba00dde977fab7ca4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### read map 的值是什么时间更新的 ？

1.  Load/LoadOrStore/LoadAndDelete 时，当 misses 数量大于等于 dirty map 的元素个数时，会整体复制 dirty map 到 read map
2.  Store/LoadOrStore 时，当 read map 中存在这个key，则更新
3.  Delete/LoadAndDelete 时，如果 read map 中存在这个key，则设置这个值为 nil

### dirty map 的值是什么时间更新的 ？

1.  完全是一个新 key， 第一次插入 sync.Map，必先插入 dirty map
2.  Store/LoadOrStore 时，当 read map 中不存在这个key，在 dirty map 存在这个key，则更新
3.  Delete/LoadAndDelete 时，如果 read map 中不存在这个key，在 dirty map 存在这个key，则从 dirty map 中删除这个key
4.  当 misses 数量大于等于 dirty map 的元素个数时，会整体复制 dirty map 到 read map，同时设置 dirty map 为 nil

**疑问：当 dirty map 复制到 read map 后，将 dirty map 设置为 nil，也就是 dirty map 中就不存在这个 key 了。如果又新插入某个 key，多次访问后达到了 dirty map 往 read map 复制的条件，如果直接用 read map 覆盖 dirty map，那岂不是就丢了之前在 read map 但不在 dirty map 的 key ?**

答：其实并不会。当 dirty map 向 read map 复制后，readOnly.amended 等于了 false。当新插入了一个值时，会将 read map 中的值，重新给 dirty map 赋值一遍，也就是 read map 也会向 dirty map 中复制。

```go
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

```

### read map 和 dirty map 是什么时间删除的？

-   当 read map 中存在某个 key 的时候，这个时候只会删除 read map， 并不会删除 dirty map（因为 dirty map 不存在这个值）
-   当 read map 中不存在时，才会去删除 dirty map 里面的值

**疑问：如果按照这个删除方式，那岂不是 dirty map 中会有残余的 key，导致没删除掉？**

答：其实并不会。当 misses 数量大于等于 dirty map 的元素个数时，会整体复制 dirty map 到 read map。这个过程中还附带了另外一个操作：将 dirty map 置为 nil。

```go
func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	m.read.Store(readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}

```

### read map 与 dirty map 的关系 ？

1.  在 read map 中存在的值，在 dirty map 中可能不存在。
2.  在 dirty map 中存在的值，在 read map 中也可能存在。
3.  当访问多次，发现 dirty map 中存在，read map 中不存在，导致 misses 数量大于等于 dirty map 的元素个数时，会整体复制 dirty map 到 read map。
4.  当出现 dirty map 向 read map 复制后，dirty map 会被置成 nil。
5.  当出现 dirty map 向 read map 复制后，readOnly.amended 等于了 false。当新插入了一个值时，会将 read map 中的值，重新给 dirty map 赋值一遍

### read/dirty map 中的值一定是有效的吗？

并不一定。放入到 read/dirty map 中的值总共有 3 种类型：

-   nil：如果获取到的 value 是 nil，那说明这个 key 是已经删除过的。既不在 read map，也不在 dirty map
-   expunged：这个 key 在 dirty map 中是不存在的
-   valid：其实就正常的情况，要么这个值存在在 read map 中，要么存在在 dirty map 中

## sync.Map 是如何提高性能的？

通过源码解析，我们知道 sync.Map 里面有两个普通 map，read map主要是负责读，dirty map 是负责读和写（加锁）。在读多写少的场景下，read map 的值基本不发生变化，可以让 read map 做到无锁操作，就减少了使用 Mutex + Map 必须的加锁/解锁环节，因此也就提高了性能。

不过也能够看出来，read map 也是会发生变化的，如果某些 key 写操作特别频繁的话，sync.Map 基本也就退化成了 Mutex + Map（有可能性能还不如 Mutex + Map）。

所以，不是说使用了 sync.Map 就一定能提高程序性能，我们日常使用中尽量注意拆分粒度来使用 sync.Map。

关于如何分析 sync.Map 是否优化了程序性能，同样可以使用 pprof。具体过程可以参考 《这可能是最容易理解的 Go Mutex 源码剖析》

## sync.Map 应用场景

1.  读多写少
2.  写操作也多，但是修改的 key 和读取的 key 特别不重合。

关于第二点我觉得挺扯的，毕竟我们很难把控这一点，不过由于是官方的注释还是放在这里。

实际开发中我们要注意使用场景和擅用 pprof 来分析程序性能。

## sync.Map 使用注意点

和 Mutex 一样， sync.Map 也同样不能被复制，因为 atomic.Value 是不能被复制的。

## 二、什么是sync.Map

> sync.Map就像Go map[interface{}]interface{}，但是它可以线程安全的对其进行读写操作，而无需其他锁定或者协调。

### sync.Map结构体说明

![](https://pic3.zhimg.com/80/v2-b363167074b865429ec321287bc8810a_720w.jpg)

![[Pasted image 20220512142740.png]]

## 三、源码实现

### Load

```go
// 返回Map中对应的key的值，如果不存在则返回nil
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly) // 因read只读，线程安全，优先读取
	e, ok := read.m[key] // 读取key对应的值
        // 如果read没有，并且dirty有新数据，那么去dirty中查找
	if !ok && read.amended {
		m.mu.Lock() //加互斥锁
                // 再次读取数据，避免加锁过程中m.dirty提升为m.read,这个时候m.read被替换了
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
                // 如果read中还是不存在，并且dirty中有新数据
		if !ok && read.amended {
			e, ok = m.dirty[key] //读取数据
                        // 不管m.dirty中是否有数据，都将misses计数+1
                        // 然后在 m.misses >= len(m.dirty)下将dirty赋值给read
			m.missLocked()
		}
		m.mu.Unlock() //解锁
	}
        // read中没有，dirty中也没有 返回nil
	if !ok {
		return nil, false
	}
        // 中read或者dirty中加载数据
	return e.load()
}
// 返回key对应的值
func (e *entry) load() (value interface{}, ok bool) {
	p := atomic.LoadPointer(&e.p)
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*interface{})(p), true
}

func (m *Map) missLocked() {
	m.misses++ //将misses计数+1
	if m.misses < len(m.dirty) {  // ？
		return
	}
        // 将dirty置给read。
 	m.read.Store(readOnly{m: m.dirty})
	m.dirty = nil 
	m.misses = 0
}
```

![](https://pic1.zhimg.com/80/v2-5b6bc01313820d9a7b0a2b9b00152964_720w.jpg)

### Store

```go
// 设置或者更新数据
func (m *Map) Store(key, value interface{}) {
        // 往只读的entry获得数据
	read, _ := m.read.Load().(readOnly) 
        // 如果m.read存在这个键，并且没有被标记删除，则尝试更新
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}
        // 如果read不存在或者已经被标记删除
	m.mu.Lock() //加锁
	read, _ = m.read.Load().(readOnly) //再次读取read中的数据
	if e, ok := read.m[key]; ok { // read中存在key对应的数据
                // 确保key值已经删除
		if e.unexpungeLocked() {
			m.dirty[key] = e //加入dirty中，这儿是指针
		}
                // 更新value值
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok { // dirty中存在key
		// 更新value值
                e.storeLocked(&value)
	} else { // read中没有，dirty中也没有

                // 如果read与dirty相同，则触发一次dirty刷新
                // 因为当read重置的时候，dirty已置为nil了
		if !read.amended { // amended=true时不一致，fasle则是一致 
                        // 将read中未删除的数据加入到dirty中
			m.dirtyLocked()
                        // amended标记为read与dirty不相同，因为后面即将加入新数据。
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value) //将这个entry加入到m.dirty中
	}
	m.mu.Unlock() //解锁
}
// 初始化entry
func newEntry(i interface{}) *entry {
	return &entry{p: unsafe.Pointer(&i)}
}
// 对entry尝试更新 （原子cas操作）
func (e *entry) tryStore(i *interface{}) bool {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}
// unexpungeLocked确保该条目未标记为已删除
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}
// 原子更新数据
func (e *entry) storeLocked(i *interface{}) {
	atomic.StorePointer(&e.p, unsafe.Pointer(i))
}
// 将read中未删除的数据加入到dirty中
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
        //  遍历read
	for k, e := range read.m {
                // 通过此次操作，dirty中的元素都是未被删除的，
                // 可见标记为expunged的元素不在dirty中！！！
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}
// 判断entry是否被标记删除，并且将标记为nil的entry更新标记为expunge
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
                // 将已经删除标记为nil的数据标记为expunged
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
                // 重新设置p的值，然后进入for循环
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}
```

![](https://pic4.zhimg.com/80/v2-1b7fbba5388eef2a68561cb758c16a37_720w.jpg)

### LoadOrStore

> 参考Load和Store函数

### LoadAndDelete、Delete

```go
// 删除数据
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
	read, _ := m.read.Load().(readOnly) //往只读read总获得数据
	e, ok := read.m[key]
        // 如果read没有，并且dirty有新数据，那么去dirty中查找
        // read.amended=true说明dirty和read中的数据不一致，有新数据
	if !ok && read.amended { 
		m.mu.Lock() //加锁
                //二次读取，以防read在加锁过程中发生变化
		read, _ = m.read.Load().(readOnly) 
		e, ok = read.m[key]
                // 如果read没有，并且dirty有新数据，那么去dirty中查找
		if !ok && read.amended {
                        // 直接删除数据
			e, ok = m.dirty[key]
			delete(m.dirty, key)
			m.missLocked() // 不管m.dirty中是否有数据，都将misses计数+1
		}
		m.mu.Unlock() //解锁
	}
        // 如果read中有数据
	if ok {
		return e.delete() //尝试删除数据
	}
	return nil, false
}
// 尝试删除数据
func (e *entry) delete() (value interface{}, ok bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return nil, false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return *(*interface{})(p), true
		}
	}
}
// 调用了LoadAndDelete来删除
func (m *Map) Delete(key interface{}) {
	m.LoadAndDelete(key)
}
```

![](https://pic2.zhimg.com/80/v2-0ed70c18cde74efe97a4b99bb31c5f9d_720w.jpg)

### Range

```go
// 遍历调用时刻 map 中的所有 k-v 对，将它们传给 f 函数，如果 f 返回 false，将停止遍历
func (m *Map) Range(f func(key, value interface{}) bool) {
	read, _ := m.read.Load().(readOnly)
        // read和dirty数据不一致时为true 
	if read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		if read.amended {
                        // 将dirty赋值给read
			read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}
        // 遍历, for range是安全的
	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) { //按照传入的func退出循环
			break
		}
	}
}
```

![](https://pic4.zhimg.com/80/v2-e4828d41a1517c3489332dbbf308a9bb_720w.jpg)
## Delete

```golang
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 如果 read 中没有这个 key，且 dirty map 不为空
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			delete(m.dirty, key) // 直接从 dirty 中删除这个 key
		}
		m.mu.Unlock()
	}
	if ok {
		e.delete() // 如果在 read 中找到了这个 key，将 p 置为 nil
	}
}
```

可以看到，基本套路还是和 Load，Store 类似，都是先从 read 里查是否有这个 key，如果有则执行 `entry.delete` 方法，将 p 置为 nil，这样 read 和 dirty 都能看到这个变化。

如果没在 read 中找到这个 key，并且 dirty 不为空，那么就要操作 dirty 了，操作之前，还是要先上锁。然后进行 double check，如果仍然没有在 read 里找到此 key，则从 dirty 中删掉这个 key。但不是真正地从 dirty 中删除，而是更新 entry 的状态。

来看下 `entry.delete` 方法：

```golang
func (e *entry) delete() (hadValue bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return true
		}
	}
}
```

它真正做的事情是将正常状态（指向一个 interface{}）的 p 设置成 nil。没有设置成 expunged 的原因是，当 p 为 expunged 时，表示它已经不在 dirty 中了。这是 p 的状态机决定的，在 `tryExpungeLocked` 函数中，会将 nil 原子地设置成 expunged。

`tryExpungeLocked` 是在新创建 dirty 时调用的，会将已被删除的 entry.p 从 nil 改成 expunged，这个 entry 就不会写入 dirty 了。

```golang
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		// 如果原来是 nil，说明原 key 已被删除，则将其转为 expunged。
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}
```

注意到如果 key 同时存在于 read 和 dirty 中时，删除只是做了一个标记，将 p 置为 nil；而如果仅在 dirty 中含有这个 key 时，会直接删除这个 key。原因在于，若两者都存在这个 key，仅做标记删除，可以在下次查找这个 key 时，命中 read，提升效率。若只有在 dirty 中存在时，read 起不到“缓存”的作用，直接删除。

# Reference
https://juejin.cn/post/6950850928354263077
https://zhuanlan.zhihu.com/p/366986038