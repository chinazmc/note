#golang 


什么是`slice`？官方解析:`slice`是一个经过包装的数组(array)，其可为数据序列提供更通用，更强大和更方便的接口，除了具有明确维数的项（例如转换矩阵）外，Go中的大多数数组编程都是使用切片而不是简单数组完成的。接下来从源码(`runtime/slice.go`)角度来解析下`slice`相关结构体以及操作。

### **Slice结构体**

本篇文章主要基于`GO1.15+`版本编写，如下结构体所示，切片的内部实现的数据结构是通过指针引用底层数组，设定相关属性将数据读写操作限定在指定的区域内。其工作机制类似数组指针的一种封装。

```go
type slice struct {
  array unsafe.Pointer  // 一个指向底层数组的指针
  len   int // 切片当前元素的个数 
  cap   int // 切片的容量,cap永远要大于len
}
```

### **切片初始化**

`slice`初始化主要有以下三种方式，初始化方式不同，所调用的底层实现函数也不同，如下代码所示:

```go
package main
​
import (
  "fmt"
)
​
func main() {
  slice1 := []int{1, 2, 3, 4}
  slice2 := make([]int, 0, 5)
  slice3 := new([]int)
  fmt.Printf("切片slice1，分配大小:%d,实际长度： %d,指针地址：%p \n", cap(slice1), len(slice1), slice1)
  fmt.Printf("切片slice2，分配大小:%d,实际长度： %d,指针地址：%p \n", cap(slice2), len(slice2), slice2)
  fmt.Printf("切片slice3，分配大小:%d,实际长度： %d,指针地址：%p \n", cap(*slice3), len(*slice3), slice3)
}
// 输出如下
// 切片slice1，分配大小:4,实际长度： 4,指针地址：0xc00000e200
// 切片slice2，分配大小:5,实际长度： 0,指针地址：0xc00000a540 
// 切片slice3，分配大小:0,实际长度： 0,指针地址：0xc0000040d8 
```

下面我们通过`go tool compile -S main.go`来查看下这三种方式调用的底层函数

-   `[]Type`

![](https://pic1.zhimg.com/80/v2-e12d23892b9cb0a533f2d7cb6b2dcda0_720w.jpg)

如图所示，该初始化方式主要调用了`runtime.newobject`函数, 源码位置为`runtime.malloc.go`，详细的对象内存分配将放在`golang内存分配`一文中。

```go
func newobject(typ *_type) unsafe.Pointer {
 return mallocgc(typ.size, typ, true)
}
```

-   `make`

![](https://pic4.zhimg.com/80/v2-0bf31d13371698c0bcb269b025b1d043_720w.jpg)

这种方式的初始化`slice`，主要借助了`builtin`包中的`make`函数，该方法主要调用了`runtime.makeslice`函数，源码位置为`runtime/slice.go`

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
 // 判断是否溢出，以及len与cap是否越界
 mem, overflow := math.MulUintptr(et.size, uintptr(cap))
 if overflow || mem > maxAlloc || len < 0 || len > cap {
  mem, overflow := math.MulUintptr(et.size, uintptr(len))
  if overflow || mem > maxAlloc || len < 0 {
   panicmakeslicelen()
  }
  panicmakeslicecap()
 }
 // 如上面方式一样调用系统分配对象的方法
 return mallocgc(mem, et, true)
}
```

-   `new`

![](https://pic1.zhimg.com/80/v2-3c7f1b8fd22b4cb6528c3d5b8b4df380_720w.jpg)

如图所示，该初始化方式主要调用了`runtime.newobject`函数, 源码位置为`runtime.malloc.go`

```go
func newobject(typ *_type) unsafe.Pointer {
  return mallocgc(typ.size, typ, true)
}
```

### **访问元素**

`slice`的访问元素是通过访问对应`slice`的索引来返回对应的值。

```go
package main
​
import (
  "fmt"
)
​
func main() {
  src := []int{1, 2, 3}
  fmt.Println(src[1])  
}
// 输出结果：2
```

### **追加操作**

`slice`相比于`array`的一个有点就是可以根据使用情况动态的进行扩容，来适应随时增加的数据，在追加时，通过调用`append`函数来针对`slice`进行尾部追加，如果此时`slice`的`cap`值小于当前`len`加上`append`中传入的数量时，那么就会触发扩容，此时则会调用`runtime.growslice`方法来进行扩容

```go
package main
​
import "fmt"
​
func main() {
  slice1 := make([]int, 0, 2)
  slice1 = append(slice1, 1)
  slice1 = append(slice1, 1)
  slice1 = append(slice1, 1)
  fmt.Println(slice1)
}
```

下面我们通过`go tool compile -S main.go`来查看下`append`出发扩容的编译期间的操作，可以看到前面两次`append`并没有出发扩容,第三次`append`出发了扩容

![](https://pic1.zhimg.com/80/v2-8abaf1707c9cb4c6841c6808b30e1630_720w.jpg)

### **扩容机制**

如上所示，`slice`的扩容机制主要是通过`runtime.growslice`函数来实现的，源代码在`runtime/slice.go`当中，下面着重看下该函数的源码。

```go
func growslice(et *_type, old slice, cap int) slice {
  // 是否启用数据竞争监测
  if raceenabled {
    callerpc := getcallerpc()
    racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
  }
  if msanenabled {
    msanread(old.array, uintptr(old.len*int(et.size)))
  }
  // 如果扩容后的cap还小于扩容前的cap 直接panic
  if cap < old.cap {
    panic(errorString("growslice: cap out of range"))
  }
  // 如果当前slice大小为0还调用了扩容，则直接返回一个新slice
  if et.size == 0 {
    // append 没法创建一个nil指针的但是len不为0的切片
    return slice{unsafe.Pointer(&zerobase), old.len, cap}
  }
  // 新容量的计算
  newcap := old.cap
  doublecap := newcap + newcap
  if cap > doublecap { // 如果指定容量>2倍扩容前slice容量，则直接使用指定的容量
    newcap = cap
  } else { // 如果指定容量小于或者等于2倍当前slice容量
    if old.len < 1024 { // 当扩容前slice长度<1024时，则直接扩容2倍
      newcap = doublecap
    } else { // 当扩容前的长度>=1024
      // 检查是否存在越界，防止死循环；
      for 0 < newcap && newcap < cap { // 元素个数>=1024，则扩容为1.25倍,但<cap
        newcap += newcap / 4
      }
      if newcap <= 0 {
        newcap = cap
      }
    }
  }
  // 计算对应的内存块大小（span）
  var overflow bool
  var lenmem, newlenmem, capmem uintptr
  switch {
  case et.size == 1:
    lenmem = uintptr(old.len)
    newlenmem = uintptr(cap)
    capmem = roundupsize(uintptr(newcap))
    overflow = uintptr(newcap) > maxAlloc
    newcap = int(capmem)
  case et.size == sys.PtrSize:
    lenmem = uintptr(old.len) * sys.PtrSize
    newlenmem = uintptr(cap) * sys.PtrSize
    capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
    overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
    newcap = int(capmem / sys.PtrSize)
  case isPowerOfTwo(et.size):
    var shift uintptr
    if sys.PtrSize == 8 {
      shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
    } else {
      shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
    }
    lenmem = uintptr(old.len) << shift
    newlenmem = uintptr(cap) << shift
    capmem = roundupsize(uintptr(newcap) << shift)
    overflow = uintptr(newcap) > (maxAlloc >> shift)
    newcap = int(capmem >> shift)
  default:
    lenmem = uintptr(old.len) * et.size
    newlenmem = uintptr(cap) * et.size
    capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
    capmem = roundupsize(capmem)
    newcap = int(capmem / et.size)
  }
  // 如果所需要的内存超过最大可分配内存则panic
  if overflow || capmem > maxAlloc {
    panic(errorString("growslice: cap out of range"))
  }
  var p unsafe.Pointer
  if et.ptrdata == 0 {
    //分配一段新内存
    p = mallocgc(capmem, nil, false)
    memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
  } else {
    //如果元素是指针类型，申请一段新内存时需要置0，不然会被GC扫描，指针类型要考虑GC写屏障
    p = mallocgc(capmem, et, true)
    if lenmem > 0 && writeBarrier.enabled {
      bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
    }
  }
  // 把slice中的数据拷贝到新申请内存的低地址
  memmove(p, old.array, lenmem)
  // 返回新创建slice并设置cap为newcap
  return slice{p, old.len, newcap}
}
```

### **拷贝操作**

`slice`的拷贝操作也是针对切片提供的接口，可以通过调用`copy`函数来将源切片中的值拷贝到目标切片中的，拷贝成功后，针对目标切片进行的操作不会对源切片产生任何影响.

```go
package main
​
func main() {
  src := []int{ 1, 2, 3 }
  tar := make([]int, 2)
  copy(tar, src)
}
```

![](https://pic1.zhimg.com/80/v2-4380951da90950e05e83542efccafd18_720w.jpg)

其拷贝的操作主要是调用了`runtime.makeslicecopy`函数来实现，源代码在`runtime/slice.go`当中，下面来着重分析下源代码

```go
func makeslicecopy(et *_type, tolen int, fromlen int, from unsafe.Pointer) unsafe.Pointer {
  // 计算需要复制的内存块大小
  // 目标slice长度 > 来源slice长度，以来源长度为准
  // 目标slice长度 <= 来源slice长度，以目标长度为准
  var tomem, copymem uintptr
  if uintptr(tolen) > uintptr(fromlen) { //目标slice长度>来源slice长度，已目标
    // 目标slice内存，如果大于最大可分配内存，则panic
    var overflow bool
    tomem, overflow = math.MulUintptr(et.size, uintptr(tolen))  
    if overflow || tomem > maxAlloc || tolen < 0 { 
      panicmakeslicelen()
    }
    // 需要拷贝的内存为源目标内存
    copymem = et.size * uintptr(fromlen) 
  } else {
    // 需要拷贝的内存为目标内存
    tomem = et.size * uintptr(tolen)
    copymem = tomem
  }
  // 申请目标slice内存块
  var to unsafe.Pointer
  if et.ptrdata == 0 {
    to = mallocgc(tomem, nil, false)
    if copymem < tomem {
      memclrNoHeapPointers(add(to, copymem), tomem-copymem)
    }
  } else {
    to = mallocgc(tomem, et, true)
    if copymem > 0 && writeBarrier.enabled {
      bulkBarrierPreWriteSrcOnly(uintptr(to), uintptr(from), copymem)
    }
  }
  // 数据竞争检测和支持与内存清理程序的互操作
  if raceenabled {
    callerpc := getcallerpc()
    pc := funcPC(makeslicecopy)
    racereadrangepc(from, copymem, callerpc, pc)
  }
  if msanenabled {
    msanread(from, copymem)
  }
  // 复制内存块
  memmove(to, from, copymem)
  return to
}
```

### **删除操作**

Golang中并未内置删除`slice`指定元素的函数，需要开发者自己去实现。

```go
func removeElem(src []int, toDel int) []int {
  for i := 0; i < len(src); i++ {
    if src[i] == toDel {
      src = append(src[:i], src[i+1:]...)
      i--
    }
  }
  return src
}
​
func removeElem(src []int, toDel int) []int {
  tar := src[:0]
  for _, num := range src {
    if num != toDel {
      tar = append(tar, num)
    }
  }
  return tar
}
​
func removeElem(src []int, toDel int) []int {
  tar := make([]int, 0, len(src))
  for _, num := range src {
    if num != toDel {
      tar = append(tar, num)
    }
  }
  return tar
}
```

# 你真的懂 golang reslice 吗


```go
package main

func a() []int {
	a1 := []int{3}
	a2 := a1[1:]
	return a2
}

func main() {
	a()
}
复制代码
```

看到这个题, 你的第一反应是啥?

```
(A) 编译失败
(B) panic: runtime error: index out of range [1] with length 1
(C) []
(D) 其他
复制代码
```

第一感觉: 肯定能编译过, 但是运行时一定会panic的. 但事与愿违竟然能够正常运行, **结果是:[]**

## 疑问

```golang
a1 := []int{3}
a2 := a1[1:]
fmt.Println("a[1:]", a2)
复制代码
```

a1 和 a2 共享同样的底层数组, len(a1) = 1, a1[1]绝对会panic, 但是a[1:]却能正常输出, 这是为何?

## 从表面入手

整体上看下整体的情况

```golang
a1 := []int{3}
fmt.Printf("len:%d, cap:%d", len(a1), cap(a1))
fmt.Println("a[0:]", a1[0:])
fmt.Println("a[1:]", a1[1:])
fmt.Println("a[2:]", a1[2:])
复制代码
```

结果:

```
len:1, cap:1
a[0:]: [1]
a[1:] []
panic: runtime error: slice bounds out of range [2:1]
复制代码
```

从表面来看, 从a[2:]才开始panic, 到底是谁一手造成这样的结果呢?

## 汇编上看一目了然

```asm
"".a STEXT size=87 args=0x18 locals=0x18
	// 省略...
	0x0028 00040 (main.go:6)	CALL	runtime.newobject(SB)
	0x002d 00045 (main.go:6)	MOVQ	8(SP), AX  // 将slice的数据首地址加载到AX寄存器
	0x0032 00050 (main.go:6)	MOVQ	$3, (AX)   // 把3放入到AX寄存器中, 也就是a1[0]
	0x0039 00057 (main.go:8)	MOVQ	AX, "".~r0+32(SP)
	0x003e 00062 (main.go:8)	XORPS	X0, X0     // 初始化 X0 寄存器
	0x0041 00065 (main.go:8)	MOVUPS	X0, "".~r0+40(SP) // 将X0放入返回值
	0x0046 00070 (main.go:8)	MOVQ	16(SP), BP
	0x004b 00075 (main.go:8)	ADDQ	$24, SP
	0x004f 00079 (main.go:8)	RET
	// 省略....
```

其实主要关心这两行即可.

```
0x003e 00062 (main.go:8)	XORPS	X0, X0     // 初始化 X0 寄存器
0x0041 00065 (main.go:8)	MOVUPS	X0, "".~r0+40(SP) // 将X0放入返回值
复制代码
```

是不是很神奇, a[1:] 没有调用`runtime.panicSliceB(SB)`, 而是返回的是一个空的slice. 这是为何呢?

持着怀疑态度, 去 github 提上一枚 issue. [github.com/golang/go/i…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fissues%2F42069 "https://github.com/golang/go/issues/42069")

![reslice](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6b94b444fdb4f3db96f9af805431da1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

结论: 这是故意的, 单纯为了保持reslice对称而已. 这也就解释了返回一个空的slice的原因.
为了方便起见，可以省略任何索引。缺少的`low`索引默认为零；缺少的`high`索引默认为 slice 操作数的长度。长度是允许的 slice 索引。 `a1[s:]`缺少`high`索引，因此默认为`len(a1)`，并且`a1[1:len(a1)]`有效，但是当`s == len(a1)`时它将是空片。

## reslice 原理

上面的问题已经解释清楚了, 回过头来看正常 reslice 的例子

```go
func a() []int {
	a1 := []int{3, 4, 5, 6, 7, 8}
	a2 := a1[2:]
	return a2
}
```

用简单的图来描述这段代码里, a1 和 a2 之间的reslice 关系. 可以看到 a1 和 a2 是共享底层数组的.

![reslice](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eda8d5f75b6247c7840a4c64e6c2f722~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

如果你知道这些, 那么 slice 的使用基本上不会出现问题.

下面这些问题你考虑过吗 ?

1.  a1, a2 是如何共享底层数组的?
2.  a1[low:high]是如何实现的?

继续来看这段代码的汇编:

```asm
"".a STEXT size=117 args=0x18 locals=0x18
	// 省略...
	0x0028 00040 (main.go:4)	CALL	runtime.newobject(SB)
	0x002d 00045 (main.go:4)	MOVQ	8(SP), AX
	0x0032 00050 (main.go:4)	MOVQ	$3, (AX)
	0x0039 00057 (main.go:4)	MOVQ	$4, 8(AX)
	0x0041 00065 (main.go:4)	MOVQ	$5, 16(AX)
	0x0049 00073 (main.go:4)	MOVQ	$6, 24(AX)
	0x0051 00081 (main.go:4)	MOVQ	$7, 32(AX)
	0x0059 00089 (main.go:4)	MOVQ	$8, 40(AX)
	0x0061 00097 (main.go:5)	ADDQ	$16, AX
	0x0065 00101 (main.go:6)	MOVQ	AX, "".~r0+32(SP)
	0x006a 00106 (main.go:6)	MOVQ	$4, "".~r0+40(SP)
	0x0073 00115 (main.go:6)	MOVQ	$4, "".~r0+48(SP)
	0x007c 00124 (main.go:6)	MOVQ	16(SP), BP
	0x0081 00129 (main.go:6)	ADDQ	$24, SP
	0x0085 00133 (main.go:6)	RET
	// 省略....
复制代码
```

-   第4行: 将 AX 栈顶指针下移 8 字节, 指向了 a1 的 data 指向的地址空间里
-   第5-10行: 将 [3,4,5,6,7,8] 放入到 a1 的 data 指向的地址空间里
-   第11行: AX 指针后移 16 个字节. 也就是指向元素 5 的位置
-   第12行: 将 SP 指针下移 32 字节指向即将返回的 slice (其实就是 a2 ), 同时将 AX 放入到 SP. 注意 AX 放入 SP 里的是一个指针, 也就造成了a1, a2是共享同一块内存空间的
-   第13行: 将 SP 指针下移 40 字节指向了 a2 的 len, 同时 把 4 放入到 SP, 也就是 len(a2) = 4
-   第14行: 将 SP 指针下移 48 字节指向了 a2 的 cap, 同时 把 4 放入到 SP, 也就是 cap(a2) = 4

下图是 slice 的 栈图, 可以配合着上面的汇编一块看.

![slice stack](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e79f202909e489e93ecd019b68653bb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

看到这里是不是一目了然了. 于是有了下面的这些结论:

1.  reslice 完全是利用汇编实现的
2.  reslice 时, slice 的 data 通过指针的移动完成, 造成了共享相同的底层数据, 同时将新的 len, cap 放入对应的位置

至此, golang reslice的原理基本已经阐述清楚了.

## reslice 性能陷阱

### 3.1 大量内存得不到释放

在已有切片的基础上进行切片，不会创建新的底层数组。因为原来的底层数组没有发生变化，内存会一直占用，直到没有变量引用该数组。因此很可能出现这么一种情况，原切片由大量的元素构成，但是我们在原切片的基础上切片，虽然只使用了很小一段，但底层数组在内存中仍然占据了大量空间，得不到释放。比较推荐的做法，使用 `copy` 替代 `re-slice`。

```go
func lastNumsBySlice(origin []int) []int {  
	return origin[len(origin)-2:]  
}  
  
func lastNumsByCopy(origin []int) []int {  
	result := make([]int, 2)  
	copy(result, origin[len(origin)-2:])  
	return result  
}  
```

上述两个函数的作用是一样的，取 origin 切片的最后 2 个元素。

-   第一个函数直接在原切片基础上进行切片。
-   第二个函数创建了一个新的切片，将 origin 的最后两个元素拷贝到新切片上，然后返回新切片。

我们可以写两个测试用例来比较这两种方式的性能差异：

在此之前呢，我们先实现 2 个辅助函数：

```go
func generateWithCap(n int) []int {  
	rand.Seed(time.Now().UnixNano())  
	nums := make([]int, 0, n)  
	for i := 0; i < n; i++ {  
		nums = append(nums, rand.Int())  
	}  
	return nums  
}  
  
func printMem(t *testing.T) {  
	t.Helper()  
	var rtm runtime.MemStats  
	runtime.ReadMemStats(&rtm)  
	t.Logf("%.2f MB", float64(rtm.Alloc)/1024./1024.)  
}  
```

-   `generateWithCap` 用于随机生成 n 个 int 整数，64位机器上，一个 int 占 8 Byte，128 * 1024 个整数恰好占据 1 MB 的空间。
-   `printMem` 用于打印程序运行时占用的内存大小。

接下来分别为 `lastNumsBySlice` 和 `lastNumsByCopy` 实现测试用例：

```go
func testLastChars(t *testing.T, f func([]int) []int) {  
	t.Helper()  
	ans := make([][]int, 0)  
	for k := 0; k < 100; k++ {  
		origin := generateWithCap(128 * 1024) // 1M  
		ans = append(ans, f(origin))  
	}  
	printMem(t)  
	_ = ans  
}  
  
func TestLastCharsBySlice(t *testing.T) { testLastChars(t, lastNumsBySlice) }  
func TestLastCharsByCopy(t *testing.T)  { testLastChars(t, lastNumsByCopy) }  
```

-   测试用例内容非常简单，随机生成一个大小为 1 MB 的切片( 128*1024 个 int 整型，恰好为 1 MB)。
-   分别调用 `lastNumsBySlice` 和 `lastNumsByCopy` 取切片的最后两个元素。
-   最后然后打印程序所占用的内存。

运行结果如下：

```sh
$ go test -run=^TestLastChars  -v  
=== RUN   TestLastCharsBySlice  
--- PASS: TestLastCharsBySlice (0.31s)  
    slice_test.go:73: 100.14 MB  
=== RUN   TestLastCharsByCopy  
--- PASS: TestLastCharsByCopy (0.28s)  
    slice_test.go:74: 3.14 MB  
PASS  
ok      example 0.601s  
```

结果差异非常明显，`lastNumsBySlice` 耗费了 100.14 MB 内存，也就是说，申请的 100 个 1 MB 大小的内存没有被回收。因为切片虽然只使用了最后 2 个元素，但是因为与原来 1M 的切片引用了相同的底层数组，底层数组得不到释放，因此，最终 100 MB 的内存始终得不到释放。而 `lastNumsByCopy` 仅消耗了 3.14 MB 的内存。这是因为，通过 `copy`，指向了一个新的底层数组，当 origin 不再被引用后，内存会被垃圾回收(garbage collector, GC)。

如果我们在循环中，显示地调用 `runtime.GC()`，效果会更加地明显：

```go
func testLastChars(t *testing.T, f func([]int) []int) {  
	t.Helper()  
	ans := make([][]int, 0)  
	for k := 0; k < 100; k++ {  
		origin := generateWithCap(128 * 1024) // 1M  
		ans = append(ans, f(origin))  
		runtime.GC()  
	}  
	printMem(t)  
	_ = ans  
}  
```

`lastNumsByCopy` 内存占用直接下降到 0.15 MB。

```bash
$ go test -run=^TestLastChars  -v  
=== RUN   TestLastCharsBySlice  
--- PASS: TestLastCharsBySlice (0.37s)  
    slice_test.go:75: 100.14 MB  
=== RUN   TestLastCharsByCopy  
--- PASS: TestLastCharsByCopy (0.34s)  
    slice_test.go:76: 0.15 MB  
PASS  
ok      example 0.723s
```

# 见微知著 带你透过内存看 Slice 和 Array的异同

Go 语言中，数组变量属于值类型(value type)，因此当一个数组变量被赋值或者传递时，实际上会复制整个数组。例如，将 a 赋值给 b，修改 a 中的元素并不会改变 b 中的元素
## Array

```go
func main() {
  as := [4]int{10, 5, 8, 7}
  
  fmt.Println("as[0]：", as[0])
  fmt.Println("as[1]：", as[1])
  fmt.Println("as[2]：", as[2])
  fmt.Println("as[3]：", as[3])
}
复制代码
```

这段很简单的代码，声明了一个 array。当然输出结果也足够简单。

我们现在玩点花活，如何通过非正常的手段访问数组里面的元素呢？在做这个事情之前是需要先知道 array 的底层结构的。其实很简单，Go array 就是一块连续的内存空间。如下图所示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25a907fde8204187b2baf95439bc5ee5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

写一段简单的代码，我们不通过下标访问的方式去获取元素。通过移动指针的方式去获取对应位置的指针。

```go
func main() {
    as := [4]int{10, 5, 8, 7}

    p1 := *(*int)(unsafe.Pointer(&as))
    fmt.Println("as[0]:", p1)

    p2 := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + unsafe.Sizeof(as[0])))
    fmt.Println("as[1]:", p2)

    p3 := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + unsafe.Sizeof(as[0])*2))
    fmt.Println("as[2]:", p3)

    p4 := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + unsafe.Sizeof(as[0])*3))
    fmt.Println("as[3]:", p4)
}
复制代码
```

结果：

```
as[0]: 10
as[1]: 5
as[2]: 8
as[3]: 7
```

下图演示下获取对应位置的值的过程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97f09417d2ad42ed8caeeaaf249e9f7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## Slice

同样对于 slice 这段简单的代码：

```go
func main() {
  as := []int{10, 5, 8, 7}
  
  fmt.Println("as[0]：", as[0])
  fmt.Println("as[1]：", as[1])
  fmt.Println("as[2]：", as[2])
  fmt.Println("as[3]：", as[3])
}
复制代码
```

想要通过移动指针的方式获取 slice 对应位置的值，仍然需要知道 slice 的底层结构。如图： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1824df747e64864a8fa94729fe8c713~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

```go
func main() {
    as := []int{10, 5, 8, 7}

    p := *(*unsafe.Pointer)(unsafe.Pointer(&as))
    fmt.Println("as[0]:", *(*int)(unsafe.Pointer(uintptr(p))))
    fmt.Println("as[1]:", *(*int)(unsafe.Pointer(uintptr(p) + unsafe.Sizeof(&as[0]))))
    fmt.Println("as[2]:", *(*int)(unsafe.Pointer(uintptr(p) + unsafe.Sizeof(&as[0])*2)))
    fmt.Println("as[3]:", *(*int)(unsafe.Pointer(uintptr(p) + unsafe.Sizeof(&as[0])*3)))

    var Len = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + uintptr(8)))
    fmt.Println("len", Len) 

    var Cap = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + uintptr(16)))
    fmt.Println("cap", Cap) 
}
复制代码
```

结果：

```
as[0]: 10
as[1]: 5
as[2]: 8
as[3]: 7
len 4
cap 4
复制代码
```

用指针取 slice 的底层 Data 里面的元素跟 array 稍微有点不同：

-   对 slice 变量 as 取地址后，拿到的是 SiceHeader 的地址，对这个指针进行移动，得到是 slice 的 Data, Len, Cap。
-   所以当拿到 Data 的值时，我们拿到的是 Data 所指向的 array 的首地址的值。
-   由于这个值是个指针，需要对这个值 *Data， 取到 array 真正的首地址的指针值
-   然后对这个值 &(*Data)，获取到真正的首地址，然后对这个值进行指针的移动，才能获取到 slice 的数组里的值

获取 slice cap 和 len:

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/875c2349d6cc4b71b43b5b876a10c939~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

获取 slice 的 Data：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c59d2efa68640e4bba4de0c7c653f65~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

# Reference
https://zhuanlan.zhihu.com/p/506575298
https://juejin.cn/post/6887946473632333838
https://geektutu.com/post/hpg-slice.html
https://juejin.cn/post/6999843912760328205