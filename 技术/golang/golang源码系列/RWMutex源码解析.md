
`RWMutex`是一个支持**并行读串行写**的读写锁。`RWMutex`具有**写操作优先**的特点，写操作发生时，仅允许正在执行的读操作执行，后续的读操作都会被阻塞。

## 使用场景

`RWMutex`常用于大量并发读，少量并发写的场景；比如微服务配置更新、交易路由缓存等场景。相对于`Mutex`互斥锁，`RWMutex`读写锁具有更好的读性能。

下面以 “多个协程并行读取str变量，一个协程每100毫秒定时更新str变量” 场景为例，进行`RWMutex`读写锁和`Mutex`互斥锁的性能对比。

```go
// 基于RWMutex的实现
var rwLock sync.RWMutex
var str1 = "hello"

func readWithRWLock() string {
    rwLock.RLock()
    defer rwLock.RUnlock()
    return str1
}

func writeWithRWLock() {
    rwLock.Lock()
    str1 = time.Now().Format("20060102150405")
    rwLock.Unlock()
}

// 多个协程并行读取string变量，同时每100ms对string变量进行1次更新
func BenchmarkRWMutex(b *testing.B) {
    ticker := time.NewTicker(100 * time.Millisecond)
    go func() {
        for range ticker.C {
            writeWithRWLock()
        }
    }()
    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            readWithRWLock()
        }
    })
}
// 基于Mutex实现
var lock sync.Mutex
var str2 = "hello"

func readWithMutex() string {
    lock.Lock()
    defer lock.Unlock()
    return str2
}

func writeWithMutex() {
    lock.Lock()
    str2 = time.Now().Format("20060102150405")
    lock.Unlock()
}

// 多个协程并行读取string变量，同时每100ms对string变量进行1次更新
func BenchmarkMutex(b *testing.B) {
    ticker := time.NewTicker(100 * time.Millisecond)
    go func() {
        for range ticker.C {
            writeWithMutex()
        }
    }()
    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            readWithMutex()
        }
    })
}
复制代码
```

`RWMutex`读写锁和`Mutex`互斥锁的性能对比，结果如下：

```shell
# go test 结果
go test -bench . -benchtime=10s
BenchmarkRWMutex-8      227611413               49.5 ns/op
BenchmarkMutex-8        135363408               87.8 ns/op
PASS
ok      demo    37.800s
复制代码
```

## 源码解析

`RWMutex`是一个**写操作优先**的读写锁，如下图所示：

1.  写操作C发生时，读操作A和读操作B正在执行，因此写操作C被挂起；
2.  当读操作D发生时，由于存在写操作C等待锁，所以读操作D被挂起；
3.  读操作A和读操作B执行完成，由于没有读操作和写操作正在执行，写操作C被唤醒执行；
4.  当读操作E发生时，由于写操作C正在执行，所以读操作E被挂起；
5.  当写操作C执行完成后，读操作D和读操作E被唤醒；

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d16135e786e84f8b900dcbbb6eb08e9a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### RWMutex结构体

`RWMutex`由如下变量组成：

1.  `rwmutexMaxReaders`：表示`RWMutex`能接受的最大读协程数量，超过`rwmutexMaxReaders`后会发生panic；
2.  `w`：`Mutex`互斥锁，用于实现写操作之间的互斥
3.  `writerSem`：写操作操作信号量；当存在读操作时，写操作会被挂起；读操作全部完成后，通过`writerSem`信号量唤醒写操作；
4.  `readerSem`：读操作信号量；当存在写操作时，读操作会被挂起；写操作完成后，通过`readerSem`信号量唤醒读操作；
5.  `readerCount`：正在执行中的读操作数量；当不存在写操作时从0开始计数，为正数；当存在写操作时从负的rwmutexMaxReaders开始计数，为负数；
6.  `readerWait`：写操作等待读操作的数量；当执行`Lock()`方法时，如果当前存在读操作，会将读操作的数量记录在`readerWait`中，并挂起写操作；读操作执行完成后，会更新`readerWait`，当`readerWait`为0时，唤醒写操作；

```go
const rwmutexMaxReaders = 1 << 30

type RWMutex struct {
    w           Mutex  // Mutex互斥锁，用于实现写操作之间的互斥

    writerSem   uint32 // 写操作信号量，用于读操作唤醒写操作
    readerSem   uint32 // 读操作信号量，用于写操作唤醒读操作

    readerCount int32  // 读操作的数量，不存在写操作时从0开始计数，存在写操作时从-rwmutexMaxReaders开始计数
    readerWait  int32  // 写操作等待读操作的数量
}
复制代码
```

### Lock()方法

`Lock`方法用于写操作获取锁，其操作如下：

1.  获取`w`互斥锁，保证同一时刻只有一个写操作执行；
2.  将`readerCount`更新为负数，使后续发生的读操作被阻塞；
3.  如果当前存在活跃的读操作`r != 0`，写操作进入阻塞状态`runtime_SemacquireMutex`；

```go
func (rw *RWMutex) Lock() {
    // 写操作之间通过w互斥锁实现互斥
    rw.w.Lock()
    // 1.将readerCount更新为负值，表示当前有写操作；当readerCount为负数时，新的读操作会被挂起
    // 2.r表示当前正在执行的读操作数量
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // r != 0表示当前存在正在执行的读操作；写操作需要等待所有读操作执行完，才能被执行；
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        // 将写操作挂起
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
复制代码
```

### Unlock()方法

`Unlock`方法用于写操作释放锁，其操作如下：

1.  将`readerCount`更新为正数，表示当前不存在活跃的写操作；
    1.  如果更新后的`readerCount`大于0，表示当前写操作阻塞了`readerCount`个读操作，需要将所有被阻塞的读操作都唤醒；
2.  将`w`互斥锁释放，允许其他写操作执行；

```go
func (rw *RWMutex) Unlock() {
    // 将readerCount更新为正数，从0开始计数
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    if r >= rwmutexMaxReaders {
        throw("sync: Unlock of unlocked RWMutex")
    }
    // 唤醒所有等待写操作的读操作 
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // 释放w互斥锁，允许其他写操作进入
    rw.w.Unlock()
}
复制代码
```

### RLock()方法

`RLock`方法用于读操作获取锁，其操作如下：

1.  原子更新`readerCount+1`；
2.  如果当前存在写操作`atomic.AddInt32(&rw.readerCount, 1) < 0`，读操作进入阻塞状态；

```go
func (rw *RWMutex) RLock() {
    // 原子更新readerCount+1
    // 1. readerCount+1为负数时，表示当前存在写操作；读操作需要等待写操作执行完，才能被执行
    // 2. readerCount+1不为负数时，表示当前不存在写操作，读操作可以执行
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // 将读操作挂起
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
复制代码
```

### RUnlock()方法

`RUnlock`方法用于读操作释放锁，其操作如下：

1.  原子更新`readerCount-1`；
2.  如果当前读操作阻塞了写操作`atomic.AddInt32(&rw.readerCount, -1)<0`，原子更新`readerWait-1`；
    \1.  当`readerWait`为0时，表示阻塞写操作的所有读操作都执行完了，唤醒写操作；


```go
func (rw *RWMutex) RUnlock() {
    // 原子更新readerCount-1
    // 当readerCount-1为负时，表示当前读操作阻塞了写操作，需要进行readerWait的更新
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow(r)
    }
}

func (rw *RWMutex) rUnlockSlow(r int32) {
    if r+1 == 0 || r+1 == -rwmutexMaxReaders {
        throw("sync: RUnlock of unlocked RWMutex")
    }
    // 原子操作readerWait-1
    // 当readerWait-1为0时，表示导致写操作阻塞的所有读操作都执行完，将写操作唤醒
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 唤醒读操作
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```


# 为什么很多场景要使用互斥锁而不是读写锁
单位读写时间时间增加之后，频率低的情况下，写次数多的话，互斥锁性能就会跟读写锁差不多。所以不管我怎么测试，读写锁都比互斥锁速度快。
读写锁可以说是跟互斥锁相比是空间换时间吧！！！！

但是呢，读写锁要进行的操作比互斥锁复杂，读写锁的对象也比互斥锁大，那么在读写数量大致相同的时候，读写速度其实是相同的，但是呢，内存分配的次数和大小都是读写锁要多很多，而且读写锁的操作要考虑锁定类型，滥用的话会容易出错
1.  Attempting to lock the write lock again when the write lock has been locked will block the current goroutine  
    在写锁已经被锁定的情况下再次尝试锁定写锁会阻塞当前goroutine
2.  Attempting to lock the read lock again when the write lock is locked will also block the current goroutine  
    在写锁上锁的情况下尝试再次上锁读锁也会阻塞当前goroutine
3.  Attempting to lock the write lock while the read lock is locked will also block the current goroutine  
    在读锁被锁定的情况下试图锁定写锁也会阻塞当前的goroutine
4.  Attempting to lock the read lock after the read lock has been locked will not block the current goroutine  
    在读锁被锁定后尝试锁定读锁不会阻塞当前goroutine


## 附 互斥锁如何实现公平

如果多个 goroutine 都在请求同一个锁，sync.Mutex 是如何实现分配公平的呢？[sync.mutex 源代码分析](https://colobu.com/2018/12/18/dive-into-sync-mutex/) 这篇文章介绍了 sync.Mutex 的演进历史和当前的实现机制。重要的部分引用如下：

根据Mutex的注释，当前的 Mutex 有如下的性质。这些注释将极大的帮助我们理解Mutex的实现。

> 互斥锁有两种状态：正常状态和饥饿状态。
> 
> 在正常状态下，所有等待锁的 goroutine 按照FIFO顺序等待。唤醒的 goroutine 不会直接拥有锁，而是会和新请求锁的 goroutine 竞争锁的拥有。新请求锁的 goroutine 具有优势：它正在 CPU 上执行，而且可能有好几个，所以刚刚唤醒的 goroutine 有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的 goroutine 会加入到等待队列的前面。 如果一个等待的 goroutine 超过 1ms 没有获取锁，那么它将会把锁转变为饥饿模式。
> 
> 在饥饿模式下，锁的所有权将从 unlock 的 goroutine 直接交给交给等待队列中的第一个。新来的 goroutine 将不会尝试去获得锁，即使锁看起来是 unlock 状态, 也不会去尝试自旋操作，而是放在等待队列的尾部。
> 
> 如果一个等待的 goroutine 获取了锁，并且满足一以下其中的任何一个条件：(1)它是队列中的最后一个；(2)它等待的时候小于1ms。它会将锁的状态转换为正常状态。
> 
> 正常状态有很好的性能表现，饥饿模式也是非常重要的，因为它能阻止尾部延迟的现象。


