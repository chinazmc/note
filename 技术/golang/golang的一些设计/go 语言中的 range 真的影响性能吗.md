
> 是不是有人告诉你，range 每次循环都进行一次值拷贝，非常影响性能。今天来揭秘 range 到底有多”坑“

# range 的简单介绍

Go 语言中，range 可以用来很方便地遍历数组(array)、切片(slice)、字典(map)和信道(chan)

## array/slice
```go
words := []string{"Go", "语言", "很", "棒"}  
for i, s := range words {  
    words = append(words, "test")  
    fmt.Println(i, s)  
}  
```

输出结果如下：

```
0 Go  
1 语言  
2 很  
3 棒  
```

-   变量 words 在循环开始前，仅会计算一次，如果在循环中修改切片的长度不会改变本次循环的次数。
-   迭代过程中，每次迭代的下标和值被赋值给变量 i 和 s，第二个参数 s 是可选的。
-   针对 nil 切片，迭代次数为 0。

range 还有另一种只遍历下标的写法，这种写法与 for 几乎没什么差异了。
```go
for i := range words {  
	fmt.Println(i, words[i])  
}  
```

输出也是一样的：

```go
0 Go  
1 语言  
2 很  
3 棒  
```

## map

```go
m := map[string]int{  
    "one":   1,  
    "two":   2,  
    "three": 3,  
}  
for k, v := range m {  
    delete(m, "two")  
    m["four"] = 4  
    fmt.Printf("%v: %v\n", k, v)  
}  
```

输出结果为：

```
one: 1  
four: 4  
three: 3  
```

-   和切片不同的是，迭代过程中，删除还未迭代到的键值对，则该键值对不会被迭代。
-   在迭代过程中，如果创建新的键值对，那么新增键值对，可能被迭代，也可能不会被迭代。
-   针对 nil 字典，迭代次数为 0

## channel

```
ch := make(chan string)  
go func() {  
    ch <- "Go"  
    ch <- "语言"  
    close(ch)  
}()  
for n := range ch {  
    fmt.Println(n)  
}  
```

-   发送给信道(channel) 的值可以使用 for 循环迭代，直到信道被关闭。
-   如果是 nil 信道，循环将永远阻塞。

# for 和 range 的性能比较

## int

```go
func generateWithCap(n int) []int {  
	rand.Seed(time.Now().UnixNano())  
	nums := make([]int, 0, n)  
	for i := 0; i < n; i++ {  
		nums = append(nums, rand.Int())  
	}  
	return nums  
}  
  
func BenchmarkForIntSlice(b *testing.B) {  
	nums := generateWithCap(1024 * 1024)  
	for i := 0; i < b.N; i++ {  
		len := len(nums)  
		var tmp int  
		for k := 0; k < len; k++ {  
			tmp = nums[k]  
		}  
		_ = tmp  
	}  
}  
  
func BenchmarkRangeIntSlice(b *testing.B) {  
	nums := generateWithCap(1024 * 1024)  
	for i := 0; i < b.N; i++ {  
		var tmp int  
		for _, num := range nums {  
			tmp = num  
		}  
		_ = tmp  
	}  
}  
```

运行结果如下：

```go
$ go test -bench=IntSlice$ .  
goos: darwin  
goarch: amd64  
pkg: example/hpg-range  
BenchmarkForIntSlice-8              3603            324512 ns/op  
BenchmarkRangeIntSlice-8            3591            322744 ns/op  
```

-   `generateWithCap` 用于生成长度为 n 元素类型为 int 的切片。
-   从最终的结果可以看到，遍历 []int 类型的切片，for 与 range 性能几乎没有区别。

## struct

那如果是稍微复杂一点的 `[]struct` 类型呢？

```go
type Item struct {  
	id  int  
	val [4096]byte  
}  
  
func BenchmarkForStruct(b *testing.B) {  
	var items [1024]Item  
	for i := 0; i < b.N; i++ {  
		length := len(items)  
		var tmp int  
		for k := 0; k < length; k++ {  
			tmp = items[k].id  
		}  
		_ = tmp  
	}  
}  
  
func BenchmarkRangeIndexStruct(b *testing.B) {  
	var items [1024]Item  
	for i := 0; i < b.N; i++ {  
		var tmp int  
		for k := range items {  
			tmp = items[k].id  
		}  
		_ = tmp  
	}  
}  
  
func BenchmarkRangeStruct(b *testing.B) {  
	var items [1024]Item  
	for i := 0; i < b.N; i++ {  
		var tmp int  
		for _, item := range items {  
			tmp = item.id  
		}  
		_ = tmp  
	}  
}  
```

先看下 Benchmark 的结果：

```go
$ go test -bench=Struct$ .  
goos: darwin  
goarch: amd64  
pkg: example/hpg-range  
BenchmarkForStruct-8             3769580               324 ns/op  
BenchmarkRangeIndexStruct-8      3597555               330 ns/op  
BenchmarkRangeStruct-8              2194            467411 ns/op  
```

-   仅遍历下标的情况下，for 和 range 的性能几乎是一样的。
-   `items` 的每一个元素的类型是一个结构体类型 `Item`，`Item` 由两个字段构成，一个类型是 int，一个是类型是 `[4096]byte`，也就是说每个 `Item` 实例需要申请约 4KB 的内存。
-   在这个例子中，for 的性能大约是 range (同时遍历下标和值) 的 2000 倍。

## int 和 []struct{} 的性能差异

与 for 不同的是，`range` 对每个迭代值都创建了一个拷贝。因此如果每次迭代的值内存占用很小的情况下，for 和 range 的性能几乎没有差异，但是如果每个迭代值内存占用很大，例如上面的例子中，每个结构体需要占据 4KB 的内存，这种情况下差距就非常明显了。

我们可以用一个非常简单的例子来证明 range 迭代时，返回的是拷贝。

```go
persons := []struct{ no int }{{no: 1}, {no: 2}, {no: 3}}  
for _, s := range persons {  
    s.no += 10  
}  
for i := 0; i < len(persons); i++ {  
    persons[i].no += 100  
}  
fmt.Println(persons) // [{101} {102} {103}]  
```

-   `persons` 是一个长度为 3 的切片，每个元素是一个结构体。
-   使用 `range` 迭代时，试图将每个结构体的 no 字段增加 10，但修改无效，因为 range 返回的是拷贝。
-   使用 `for` 迭代时，将每个结构体的 no 字段增加 100，修改有效。

## \*struct{}

那如果切片中是指针，而不是结构体呢？

```go
func generateItems(n int) []*Item {  
	items := make([]*Item, 0, n)  
	for i := 0; i < n; i++ {  
		items = append(items, &Item{id: i})  
	}  
	return items  
}  
  
func BenchmarkForPointer(b *testing.B) {  
	items := generateItems(1024)  
	for i := 0; i < b.N; i++ {  
		length := len(items)  
		var tmp int  
		for k := 0; k < length; k++ {  
			tmp = items[k].id  
		}  
		_ = tmp  
	}  
}  
  
func BenchmarkRangePointer(b *testing.B) {  
	items := generateItems(1024)  
	for i := 0; i < b.N; i++ {  
		var tmp int  
		for _, item := range items {  
			tmp = item.id  
		}  
		_ = tmp  
	}  
}  
```

运行结果如下：

```go
goos: darwin  
goarch: amd64  
pkg: example/hpg-range  
BenchmarkForPointer-8             271279              4160 ns/op  
BenchmarkRangePointer-8           264068              4194 ns/op  
```
切片元素从结构体 `Item` 替换为指针 `*Item` 后，for 和 range 的性能几乎是一样的。而且使用指针还有另一个好处，可以直接修改指针对应的结构体的值。

# 总结

range 在迭代过程中返回的是迭代值的拷贝，如果每次迭代的元素的内存占用很低，那么 for 和 range 的性能几乎是一样，例如 `[]int`。但是如果迭代的元素内存占用较高，例如一个包含很多属性的 struct 结构体，那么 for 的性能将显著地高于 range，有时候甚至会有上千倍的性能差异。对于这种场景，建议使用 for，如果使用 range，建议只迭代下标，通过下标访问迭代值，这种使用方式和 for 就没有区别了。如果想使用 range 同时迭代下标和值，则需要将切片/数组的元素改为指针，才能不影响性能。

# 附加：值得注意的坑

在编程中我遇到过很多次循环的坑，我觉得可以趁着这次对比两种不同循环方式的契机聊一下：

range 的坑：把range出来的值取地址存起来，这个坑很简单就可以想到，只要你对range的机制比较了解。 range 机制就是新建一个地址，遍历取值，把每次遍历元素的值拷贝给这个地址。所以这个指针最终存储的值，其实是最后一个元素的值。其实 range 的本身就是让你使用值传递的，你非要取人家的指针那肯定是有问题的，这个很容易理解我就不写代码了。

另外一个就是使用循环将参数传入 go 程并发时，发生参数错乱。这其实不是range的问题。

看下面这个例子：

```go
import (  
	"fmt"  
	"time"  
)  
  
func main(){  
	tasks := []int{1,2,3,4,5,6,7,8,9,10}  
  
	for _,task := range tasks{  
	 func() {  
			fmt.Printf("任务：%d \n",task)  
		}()  
	}  
  
	time.Sleep(20 * time.Second)  
  
}  
```

你觉得会打出什么？

我加个并发呢？又会打印什么？
```go
import (  
	"fmt"  
	"time"  
)  
  
func main(){  
	tasks := []int{1,2,3,4,5,6,7,8,9,10}  
  
	for _,task := range tasks{  
		go func() {  
			fmt.Printf("任务：%d \n",task)  
		}()  
	}  
  
	time.Sleep(20 * time.Second)  
  
}  
```

Case 1:
任务：1   
任务：2   
任务：3   
任务：4   
任务：5   
任务：6   
任务：7   
任务：8   
任务：9   
任务：10   

Case 2:
任务：10   
任务：10   
任务：10   
任务：10   
任务：10   
任务：10   
任务：10   
任务：10   
任务：10   
任务：10   

你猜对了吗？

怎么一并发就乱了呢？而且都是最后一个值。

其实这是个面试题且比较高频，当时在网上找答案，大部分说是for循环执行完了以后goroutine才开始触发，触发goroutine时task的指针指向了最后一个值所以都打印了最后一个值。这个答案看起来是没啥问题的，但如果这样解释那么task传入函数岂不是一个指针，但go语言中的参数传递其实本质上没有指针传递（这个可以自行百度，不解释了在这里），那如果是值传递就不该出现这种问题。就算你触发goroutine晚传入的值也不受影响所以我认为是在凭自己的理解瞎扯。

针对网上的解释再看一个例子：

```go
func main(){  
	tasks := []int{1,2,3,4,5,6,7,8,9,10}  
  
	for i:=0;i<len(tasks);i++{  
		go func() {  
			fmt.Printf("第 %d 个任务：%d \n",i,tasks[i])  
		}()  
	}  
  
	time.Sleep(20 * time.Second)  
}  
```

你觉得会打印什么，这个每次可是数据真正的指针。仍然是先for循环，在执行goroutine，goroutine 给 task 复值，通过指针找到正确的值。没问题吧应该不会都是最后一个值吧。解释不通了。

但结果是：

```sh
panic: runtime error: index out of range [10] with length 10  
```

Panic …. 就很尴尬，这怎么会有下标10？

那么好我把下标减去一

```go
import (  
	"fmt"  
	"time"  
)  
  
func main(){  
	tasks := []int{1,2,3,4,5,6,7,8,9,10}  
  
	for i:=0;i<len(tasks)-1;i++{  
		go func() {  
			fmt.Printf("第 %d 个任务：%d \n",i,tasks[i])  
		}()  
	}  
  
	time.Sleep(20 * time.Second)  
}  
```

结果是：
第 3 个任务：4   
第 9 个任务：10   
第 9 个任务：10   
第 9 个任务：10   
第 9 个任务：10   
第 9 个任务：10   
第 9 个任务：10   
第 9 个任务：10   
第 9 个任务：10   

和 range一样的。

为什么会这样，其实不了解go的基础编译原理是很难解释的，函数的私有空间内部的变量是保存在独享的栈内存之内，而外部的公共变量则保存在共享的堆内存中，i 这个变量是在每个并发函数外部的，是共享堆内存中的。所以 i 是几取决于在函数执行时，for循环，循环到了几，一般for循环都是很快的，所以一般都会取到i的最后一个变量付值 这也就解释了为什么会Panic，for循环在i=10的时候退出，此时我们的goroutine函数才刚开始执行那么此时的i就是最后一个值10。  
这可以很明显的得出闭包包裹的其实是外部环境。再看一个例子:
```go
import (  
	"fmt"  
	"time"  
)  
  
func main(){  
	tasks := []int{1,2,3,4,5,6,7,8,9,10}  
  
	for i:=0;i<len(tasks)-1;i++{  
		time.Sleep(1 * time.Second)  
		go func() {  
			fmt.Printf("第 %d 个任务：%d \n",i,tasks[i])  
		}()  
	}  
  
	time.Sleep(12 * time.Second)  
  
}  
```

我每次让它睡一秒

ANGELSCRIPT

第 1 个任务：2   
第 2 个任务：3   
第 3 个任务：4   
第 4 个任务：5   
第 5 个任务：6   
第 6 个任务：7   
第 7 个任务：8   
第 8 个任务：9   
第 9 个任务：10   

发现没有，第0个任务木有，是因为第0个任务触发闭包函数的时候外部环境的i已经变成了1。在2-9任务执行时for循环在sleep ，所以拿到了理论上正确的i值

## 解决办法

### 将值传给函数的参数

解决方法其实很简单，一开始就把task做成参数进行值传递，这样task的值会保存在goroutine的栈中，就不会有问题了。这里要注意的是一定是值传递不要传一个共享的指针。

### 将值取出重新付给一个新的变量

将值直接在for循环中取出，这样一来虽然我们的并发函数闭包取值仍然存在共享的堆中，但它是独有的不会发生变化，在for循环结束后goroutine执行，函数从堆中获取该参数仍然是我们预期中的值。

```go
func TestFORRange(t *testing.T) {  
	tasks := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}  
  
	for i := 0; i < len(tasks)-1; i++ {  
		val := tasks[i]  
		index := i  
		go func() {  
			fmt.Printf("第 %d 个任务：%d \n", index, val)  
		}()  
	}  
  
	time.Sleep(12 * time.Second)  
}  
```
  

结果是符合预期的：
=== RUN   TestFORRange  
第 8 个任务：9   
第 3 个任务：4   
第 4 个任务：5   
第 5 个任务：6   
第 7 个任务：8   
第 2 个任务：3   
第 1 个任务：2   
第 0 个任务：1   
第 6 个任务：7   
--- PASS: TestFORRange (12.00s)  
PASS  

**这种解决方式的好处**：是你可以不用给goroutine执行的函数设置参数，可以满足一些需要通用的场景，把参数完全用闭包来传递就好了。

# Reference
https://marksuper.xyz/2021/09/15/range/