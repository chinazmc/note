
_struct能不能比较？_ 很显然这句话包含了两种情况：

- 同一个struct的两个实例能不能比较？
- 两个不同的struct的实例能不能比较？

### 划重点

在分析上面两个问题前，先跟大家梳理一下golang中，哪些数据类型是可比较的，哪些是不可比较的：

- 可比较：_Integer_，_Floating-point_，_String_，_Boolean_，_Complex(复数型)_，_Pointer_，_Channel_，_Interface_，_Array_
- 不可比较：_Slice_，_Map_，_Function_

下面就跟大家分别分析一下上面两种情况吧

## 同一个struct的两个实例能不能比较

首先，我们构造一个struct结构体来玩玩吧
```go
type S struct {     
	Name    string     
	Age     int     
	Address *int 
}  
func main() {     
	a := S{         
		Name:    "aa",         
		Age:     1,         
		Address: new(int),     
	}     
	b := S{         
		Name:    "aa",         
		Age:     1,         
		Address: new(int),     
	}     
	fmt.Println(a == b) 
}
```

运行上面的代码发现会打印false。既然能正常打印输出，说明是可以个比较的，接下来让我们来个**「死亡两问」**

_什么可以比较？_

回到上面的划重点部分，在总结中我们可以知道，golang中 _Slice_，_Map_，_Function_ 这三种数据类型是不可以直接比较的。我们再看看S结构体，该结构体并没有包含不可比较的成员变量，所以该结构体是可以直接比较的。

_为什么打印输出false？_

a 和 b 虽然是同一个struct 的两个实例，但是因为其中的指针变量 Address 的值不同，所以 a != b，如果a b 在初始化时把 Address 去掉（不给 Address 初始化），那么这时 a == b 为true, 因为ptr变量默认值是nil，又或者给 Address 成员变量赋上同一个指针变量的值，也是成立的。

如果给结构体S增加一个Slice类型的成员变量后又是什么情况呢？
```go
type S struct {     
	Name    string     
	Age     int     
	Address *int    
	Data    []int 
}  
func main() {     
	a := S{         
		Name:    "aa",         
		Age:     1,         
		Address: new(int),       
		Data:    []int{1, 2, 3},     
	}     
	b := S{         
		Name:    "aa",         
		Age:     1,         
		Address: new(int),       
		Data:    []int{1, 2, 3},     
	}     
	fmt.Println(a == b) 
}
```

这时候会打印输出什么呢？true？false？实际上运行上面的代码会报下面的错误：

复制代码

`# command-line-arguments ./test.go:37:16: invalid operation: a == b (struct containing []int cannot be compared)`

a, b 虽然是同一个struct两个赋值相同的实例，因为结构体成员变量中带有了不能比较的成员（slice)，是不可以直接用 == 比较的，所以只要写 == 就报错

**「总结」**

同一个struct的两个实例可比较也不可比较，当结构不包含不可直接比较成员变量时可直接比较，否则不可直接比较

---

但在平时的实践过程中，当我们需要对含有不可直接比较的数据类型的结构体实例进行比较时，是不是就没法比较了呢？事实上并非如此，golang还是友好滴，我们可以借助 _reflect.DeepEqual 函数_ 来对两个变量进行比较。所以上面代码我们可以这样写：
```go
type S struct {     
	Name    string     
	Age     int     
	Address *int    
	Data    []int 
}  
func main() {     
	a := S{         
		Name:    "aa",         
		Age:     1,         
		Address: new(int),       
		Data:    []int{1, 2, 3},     
	}     
	b := S{         
		Name:    "aa",         
		Age:     1,         
		Address: new(int),       
		Data:    []int{1, 2, 3},     
	}     
	fmt.Println(reflect.DeepEqual(a, b)) 
}
```

打印输出：

复制代码

`true`

那么 reflect.DeepEqual 是如何对变量进行比较的呢？

### reflect.DeepEqual

DeepEqual函数用来判断两个值是否深度一致。具体比较规则如下：

- 不同类型的值永远不会深度相等
- 当两个数组的元素对应深度相等时，两个数组深度相等
- 当两个相同结构体的所有字段对应深度相等的时候，两个结构体深度相等
- 当两个函数都为nil时，两个函数深度相等，其他情况不相等（相同函数也不相等）
- 当两个interface的真实值深度相等时，两个interface深度相等
- map的比较需要同时满足以下几个
    - 两个map都为nil或者都不为nil，并且长度要相等
    - 相同的map对象或者所有key要对应相同
    - map对应的value也要深度相等
- 指针，满足以下其一即是深度相等
    - 两个指针满足go的\==操作符
    - 两个指针指向的值是深度相等的
- 切片，需要同时满足以下几点才是深度相等
    - 两个切片都为nil或者都不为nil，并且长度要相等
    - 两个切片底层数据指向的第一个位置要相同或者底层的元素要深度相等
    - 注意：空的切片跟nil切片是不深度相等的
- 其他类型的值（numbers, bools, strings, channels）如果满足go的\==操作符，则是深度相等的。要注意不是所有的值都深度相等于自己，例如函数，以及嵌套包含这些值的结构体，数组等

## 两个不同的struct的实例能不能比较

**「结论」**：可以比较，也不可以比较

可通过强制转换来比较：
```go
type S struct {     Name    string     Age     int     Address *int    Data    []int }  func main() {     a := S{         Name:    "aa",         Age:     1,         Address: new(int),       Data:    []int{1, 2, 3},     }     b := S{         Name:    "aa",         Age:     1,         Address: new(int),       Data:    []int{1, 2, 3},     }     fmt.Println(reflect.DeepEqual(a, b)) }
```

如果成员变量中含有不可比较成员变量，即使可以强制转换，也不可以比较

复制代码

`type T2 struct {     Name  string     Age   int     Arr   [2]bool     ptr   *int     map1  map[string]string }  type T3 struct {     Name  string     Age   int     Arr   [2]bool     ptr   *int     map1  map[string]string }  func main() {     var ss1 T2     var ss2 T3          ss3 := T2(ss2)     fmt.Println(ss3==ss1)   // 含有不可比较成员变量 }`

编译报错：

复制代码

`# command-line-arguments ./test.go:28:18: invalid operation: ss3 == ss1 (struct containing map[string]string cannot be compared)`

## 问：struct可以作为map的key么

struct必须是可比较的，才能作为key，否则编译时报错

复制代码

`type T1 struct {     Name  string     Age   int     Arr   [2]bool     ptr   *int     slice []int     map1  map[string]string }  type T2 struct {     Name string     Age  int     Arr  [2]bool     ptr  *int }  func main() {     // n := make(map[T2]string, 0) // 无报错     // fmt.Print(n)                // map[]      m := make(map[T1]string, 0)     fmt.Println(m) // invalid map key type T1 }`

上面就是今天要跟大家分享的内容，有什么问题欢迎大家在后台给大叔留言，关注[大叔说码](https://link.juejin.cn/?target=https%3A%2F%2Fmmbiz.qpic.cn%2Fmmbiz_jpg%2FbrLcpWouoGs8VKCwnJISI12ym8llEWL8Z1ZXAdGOaCXibS1IsUybW3ZHibXibl2YWRfPia48e2s83ictDDdafOiappfg%2F640%3Fwx_fmt%3Djpeg%26tp%3Dwebp%26wxfrom%3D5%26wx_lazy%3D1%26wx_co%3D1 "https://mmbiz.qpic.cn/mmbiz_jpg/brLcpWouoGs8VKCwnJISI12ym8llEWL8Z1ZXAdGOaCXibS1IsUybW3ZHibXibl2YWRfPia48e2s83ictDDdafOiappfg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1")，我们下期见～


# [Go 面试官：Go 结构体是否可以比较，为什么？](https://segmentfault.com/a/1190000040099215)

## 结构体是什么

在 Go 语言中有个基本类型，开发者们称之为结构体（struct）。是 Go 语言中非常常用的，基本定义：

type struct_variable_type struct {
    member definition
    member definition
    ...
    member definition
}

简单示例：

package main

import "fmt"

type Vertex struct {
    Name1 string
    Name2 string
}

func main() {
    v := Vertex{"脑子进了", "煎鱼"}
    v.Name2 = "蒸鱼"
    fmt.Println(v.Name2)
}

输出结果：

蒸鱼

这部分属于基础知识，因此不再过多解释。如果看不懂，建议重学 Go 语言语法基础。

## 比较两下

### 例子一

接下来正式开始研讨 Go 结构体比较的问题，第一个例子如下：

type Value struct {
    Name   string
    Gender string
}

func main() {
    v1 := Value{Name: "煎鱼", Gender: "男"}
    v2 := Value{Name: "煎鱼", Gender: "男"}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

我们声明了两个变量，分别是 v1 和 v2。其都是 `Value` 结构体的实例化，是同一个结构体的两个实例。

他们的比较结果是什么呢，是输出 ”脑子进煎鱼了“，还是 ”脑子没进煎鱼“？

输出结果：

脑子进煎鱼了

最终输出结果是 ”脑子进煎鱼了“，初步的结论是可以结构体间比较的。皆大欢喜，那这篇文章是不是就要结束了？

当然不是...很多人都会踩到这个 Go 语言的坑，**真实情况是结构体是可比较，也不可比较的**，不要误入歧途了，这是一个非常 "有趣" 的现象。

### 例子二

接下来继续改造上面的例子，我们在原本的结构体中增加了指针类型的引用。

第二个例子如下：

type Value struct {
    Name   string
    Gender *string
}

func main() {
    v1 := Value{Name: "煎鱼", Gender: new(string)}
    v2 := Value{Name: "煎鱼", Gender: new(string)}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

这段程序输出结果是什么呢，我们猜测一下，变量依然是同一结构体的两个实例，值的赋值方式和内容都是一样的，是否应当输出 “脑子进煎鱼了”？

答案是：脑子没进煎鱼。

### 例子三

我们继续不信邪，试试另外的基本类型，看看结果是不是还是相等的。

第三个例子如下：

type Value struct {
    Name   string
    GoodAt []string
}

func main() {
    v1 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    v2 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

这段程序输出结果是什么呢？

答案是：

# command-line-arguments
./main.go:15:8: invalid operation: v1 == v2 (struct containing []string cannot be compared)

程序运行就直接报错，IDE 也提示错误，一只煎鱼都没能输出出来。

### 例子四

那不同结构体，相同的值内容呢，能否进行比较？

第四个例子：

type Value1 struct {
    Name string
}

type Value2 struct {
    Name string
}

func main() {
    v1 := Value1{Name: "煎鱼"}
    v2 := Value2{Name: "煎鱼"}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

显然，会直接报错：

# command-line-arguments
./main.go:18:8: invalid operation: v1 == v2 (mismatched types Value1 and Value2)

那是不是就完全没法比较了呢？并不，我们可以借助强制转换来实现：

    if v1 == Value1(v2) {
        fmt.Println("脑子进煎鱼了")
        return
    }

这样程序就会正常运行，且输出 “脑子进煎鱼了”。当然，若是不可比较类型，依然是不行的。

## 为什么

为什么 Go 结构体有的比较就是正常，有的就不行，甚至还直接报错了。难道是有什么 “潜规则” 吗？

在 Go 语言中，Go 结构体有时候并不能直接比较，当其基本类型包含：slice、map、function 时，是不能比较的。若强行比较，就会导致出现例子中的直接报错的情况。

而指针引用，其虽然都是 `new(string)`，从表象来看是一个东西，但其具体返回的地址是不一样的。

因此若要比较，则需改为：

func main() {
    gender := new(string)
    v1 := Value{Name: "煎鱼", Gender: gender}
    v2 := Value{Name: "煎鱼", Gender: gender}
    ...
}

这样就可以保证两者的比较。如果我们被迫无奈，被要求一定要用结构体比较怎么办？

这时候可以使用反射方法 `reflect.DeepEqual`，如下：

func main() {
    v1 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    v2 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    if reflect.DeepEqual(v1, v2) {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

这样子就能够正确的比较，输出结果为 “脑子进煎鱼了”。

例子中所用到的反射比较方法 `reflect.DeepEqual` 常用于判定两个值是否深度一致，其规则如下：

- 相同类型的值是深度相等的，不同类型的值永远不会深度相等。
- 当数组值（array）的对应元素深度相等时，数组值是深度相等的。
- 当结构体（struct）值如果其对应的字段（包括导出和未导出的字段）都是深度相等的，则该值是深度相等的。
- 当函数（func）值如果都是零，则是深度相等；否则就不是深度相等。
- 当接口（interface）值如果持有深度相等的具体值，则深度相等。
- ...

更具体的大家可到 `golang.org/pkg/reflect/#DeepEqual` 进行详细查看：

![reflect.DeepEqual 完整说明](https://segmentfault.com/img/remote/1460000040099217 "reflect.DeepEqual 完整说明")

该方法对 Go 语言中的各种类型都进行了兼容处理和判别，由于这不是本文的重点，因此就不进一步展开了。

## 总结

在本文中，我们针对 Go 语言的结构体（struct）是否能够比较进行了具体例子的展开和说明。

其本质上还是对 Go 语言基本数据类型的理解问题，算是变形到结构体中的具体进一步拓展。

不知道你有没有在 Go 结构体吃过什么亏呢，欢迎在下方评论区留言和我们一起交流和讨论。

# Reference
https://segmentfault.com/a/1190000040099215#item-2-1
https://juejin.cn/post/6881912621616857102

