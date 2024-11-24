#go 

# 什么是defer？

`defer`是Go语言提供的一种用于注册延迟调用的机制：让函数或语句可以在当前函数执行完毕后（包括通过return正常结束或者panic导致的异常结束）执行。

`defer`语句通常用于一些成对操作的场景：打开连接/关闭连接；加锁/释放锁；打开文件/关闭文件等。

`defer`在一些需要回收资源的场景非常有用，可以很方便地在函数结束前做一些清理操作。在打开资源语句的下一行，直接一句defer就可以在函数返回前关闭资源，可谓相当优雅。

```go
f, _ := os.Open("defer.txt")
defer f.Close()
```

注意：以上代码，忽略了err, 实际上应该先判断是否出错，如果出错了，直接return. 接着再判断`f`是否为空，如果`f`为空，就不能调用`f.Close()`函数了，会直接panic的。

# 为什么需要defer？

程序员在编程的时候，经常需要打开一些资源，比如数据库连接、文件、锁等，这些资源需要在用完之后释放掉，否则会造成内存泄漏。

但是程序员都是人，是人就会犯错。因此经常有程序员忘记关闭这些资源。go直接在语言层面提供`defer`关键字，在打开资源语句的下一行，就可以直接用`defer`语句来注册函数结束后执行关闭资源的操作。因为这样一颗“小小”的语法糖，程序员忘写关闭资源语句的情况就大大地减少了。

# 怎样合理使用defer?

defer的使用其实非常简单：

```go
f,err := os.Open(filename)
if err != nil {
    panic(err)
}

if f != nil {
    defer f.Close()
}
```

在打开文件的语句附近，用defer语句关闭文件。这样，在函数结束之前，会自动执行defer后面的语句来关闭文件。

当然，defer会有小小地延迟，对时间要求特别特别特别高的程序，可以避免使用它，其他一般忽略它带来的延迟。

# defer进阶

## defer的底层原理是什么？

我们先看一下官方对`defer`的解释：

> Each time a “defer” statement executes, the function value and parameters to the call are evaluated as usual and saved anew but the actual function is not invoked. Instead, deferred functions are invoked immediately before the surrounding function returns, in the reverse order they were deferred. If a deferred function value evaluates to nil, execution panics when the function is invoked, not when the “defer” statement is executed.

翻译一下：每次defer语句执行的时候，会把函数“压栈”，函数参数会被拷贝下来；当外层函数（非代码块，如一个for循环）退出时，defer函数按照定义的逆序执行；如果defer执行的函数为nil, 那么会在最终调用函数的产生panic.

defer语句并不会马上执行，而是会进入一个栈，函数return前，会按先进后出的顺序执行。也说是说最先被定义的defer语句最后执行。先进后出的原因是后面定义的函数可能会依赖前面的资源，自然要先执行；否则，如果前面先执行，那后面函数的依赖就没有了。

`在defer函数定义时，对外部变量的引用是有两种方式的，分别是作为函数参数和作为闭包引用。作为函数参数，则在defer定义时就把值传递给defer，并被cache起来；作为闭包引用的话，则会在defer函数真正调用时根据整个上下文确定当前的值。`

defer后面的语句在执行的时候，函数调用的参数会被保存起来，也就是复制了一份。真正执行的时候，实际上用到的是这个复制的变量，因此如果此变量是一个“值”，那么就和定义的时候是一致的。如果此变量是一个“引用”，那么就可能和定义的时候不一致。

举个例子：

```go
func main() {
	var whatever [3]struct{}
	
	for i := range whatever {
		defer func() { 
			fmt.Println(i) 
		}()
	}
}
```

执行结果：

```shell
2
2
2
```

defer后面跟的是一个闭包（后面会讲到），i是“引用”类型的变量，最后i的值为2, 因此最后打印了三个2.

有了上面的基础，我们来检验一下成果：

```go
type number int

func (n number) print()   { fmt.Println(n) }
func (n *number) pprint() { fmt.Println(*n) }

func main() {
	var n number

	defer n.print()
	defer n.pprint()
	defer func() { n.print() }()
	defer func() { n.pprint() }()

	n = 3
}
```

执行结果是：

```go
3
3
3
0
```

第四个defer语句是闭包，引用外部函数的n, 最终结果是3;  
第三个defer语句同第四个；  
第二个defer语句，n是引用，最终求值是3.  
第一个defer语句，对n直接求值，开始的时候n=0, 所以最后是0;

## 利用defer原理

有些情况下，我们会故意用到defer的先求值，再延迟调用的性质。想象这样的场景：在一个函数里，需要打开两个文件进行合并操作，合并完后，在函数执行完后关闭打开的文件句柄。

```go
func mergeFile() error {
	f, _ := os.Open("file1.txt")
	if f != nil {
		defer func(f io.Closer) {
			if err := f.Close(); err != nil {
				fmt.Printf("defer close file1.txt err %v\n", err)
			}
		}(f)
	}

	// ……

	f, _ = os.Open("file2.txt")
	if f != nil {
		defer func(f io.Closer) {
			if err := f.Close(); err != nil {
				fmt.Printf("defer close file2.txt err %v\n", err)
			}
		}(f)
	}

	return nil
}
```

`上面的代码中就用到了defer的原理，defer函数定义的时候，参数就已经复制进去了，之后，真正执行close()函数的时候就刚好关闭的是正确的“文件”了，妙哉！可以想像一下如果不这样将f当成函数参数传递进去的话，最后两个语句关闭的就是同一个文件了，都是最后一个打开的文件。`

不过在调用close()函数的时候，要注意一点：先判断调用主体是否为空，否则会panic. 比如上面的代码片段里，先判断`f`不为空，才会调用`Close()`函数，这样最安全。

## defer命令的拆解

如果defer像上面介绍地那样简单（其实也不简单啦），这个世界就完美了。事情总是没这么简单，defer用得不好，是会跳进很多坑的。

理解这些坑的关键是这条语句：

```go
return xxx
```

上面这条语句经过编译之后，变成了三条指令：

```asm
1. 返回值 = xxx
2. 调用defer函数
3. 空的return
```

1,3步才是Return 语句真正的命令，第2步是defer定义的语句，这里可能会操作返回值。

下面我们来看两个例子，试着将return语句和defer语句拆解到正确的顺序。

第一个例子：

```go
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```

拆解后：

```go
func f() (r int) {
     t := 5
     
     // 1. 赋值指令
     r = t
     
     // 2. defer被插入到赋值与返回之间执行，这个例子中返回值r没被修改过
     func() {        
         t = t + 5
     }
     
     // 3. 空的return指令
     return
}
```

这里第二步没有操作返回值r, 因此，main函数中调用f()得到5.

第二个例子：

```go
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}

```

拆解后：

```go
func f() (r int) {
     // 1. 赋值
     r = 1
     
     // 2. 这里改的r是之前传值传进去的r，不会改变要返回的那个r值
     func(r int) { 
          r = r + 5
     }(r)
     
     // 3. 空的return
     return
}
```

因此，main函数中调用f()得到1.

## defer语句的参数

defer语句表达式的值在定义时就已经确定了。下面展示三个函数：

```go
func f1() {
	var err error
	
	defer fmt.Println(err)

	err = errors.New("defer error")
	return
}

func f2() {
	var err error
	
	defer func() {
		fmt.Println(err)
	}()

	err = errors.New("defer error")
	return
}

func f3() {
	var err error
	
	defer func(err error) {
		fmt.Println(err)
	}(err)

	err = errors.New("defer error")
	return
}

func main() {
	f1()
	f2()
	f3()
}
```

运行结果：

```shell
<nil>
defer error
<nil>
```

第1，3个函数是因为作为函数参数，定义的时候就会求值，定义的时候err变量的值都是nil, 所以最后打印的时候都是nil. 第2个函数的参数其实也是会在定义的时候求值，只不过，第2个例子中是一个闭包，它引用的变量err在执行的时候最终变成`defer error`了。关于闭包在本文后面有介绍。

第3个函数的错误还比较容易犯，在生产环境中，很容易写出这样的错误代码。最后defer语句没有起到作用。

## 闭包是什么？

闭包是由函数及其相关引用环境组合而成的实体,即：

```shell
闭包=函数+引用环境
```

一般的函数都有函数名，但是匿名函数就没有。匿名函数不能独立存在，但可以直接调用或者赋值于某个变量。匿名函数也被称为闭包，一个闭包继承了函数声明时的作用域。在go中，所有的匿名函数都是闭包。

有个不太恰当的例子，可以把闭包看成是一个类，一个闭包函数调用就是实例化一个类。`闭包在运行时可以有多个实例，它会将同一个作用域里的变量和常量捕获下来，无论闭包在什么地方被调用（实例化）时，都可以使用这些变量和常量。而且，闭包捕获的变量和常量是引用传递，不是值传递。

举个简单的例子：

```go
func main() {
	var a = Accumulator()

	fmt.Printf("%d\n", a(1))
	fmt.Printf("%d\n", a(10))
	fmt.Printf("%d\n", a(100))

	fmt.Println("------------------------")
	var b = Accumulator()

	fmt.Printf("%d\n", b(1))
	fmt.Printf("%d\n", b(10))
	fmt.Printf("%d\n", b(100))


}

func Accumulator() func(int) int {
	var x int

	return func(delta int) int {
		fmt.Printf("(%+v, %+v) - ", &x, x)
		x += delta
		return x
	}
}
```

执行结果：

```shell
(0xc420014070, 0) - 1
(0xc420014070, 1) - 11
(0xc420014070, 11) - 111
------------------------
(0xc4200140b8, 0) - 1
(0xc4200140b8, 1) - 11
(0xc4200140b8, 11) - 111
```

`闭包引用了x变量，a,b可看作2个不同的实例，实例之间互不影响。实例内部，x变量是同一个地址，因此具有“累加效应”。

## defer配合recover

go被诟病比较多的就是它的error, 经常是各种error满天飞。编程的时候总是会返回一个error, 留给调用者处理。如果是那种致命的错误，比如程序执行初始化的时候出问题，直接panic掉，省得上线运行后出更大的问题。

但是有些时候，我们需要从异常中恢复。比如服务器程序遇到严重问题，产生了panic, 这时我们至少可以在程序崩溃前做一些“扫尾工作”，如关闭客户端的连接，防止客户端一直等待等等。

panic会停掉当前正在执行的程序，不只是当前协程。在这之前，它会有序地执行完当前协程defer列表里的语句，其它协程里挂的defer语句不作保证。因此，我们经常在defer里挂一个recover语句，防止程序直接挂掉，这起到了`try...catch`的效果。

注意，recover()函数只在defer的上下文中才有效（且只有通过在defer中用匿名函数调用才有效），直接调用的话，只会返回`nil`.

```go
func main() {
	defer fmt.Println("defer main")
	var user = os.Getenv("USER_")
	
	go func() {
		defer func() {
			fmt.Println("defer caller")
			if err := recover(); err != nil {
				fmt.Println("recover success. err: ", err)
			}
		}()

		func() {
			defer func() {
				fmt.Println("defer here")
			}()

			if user == "" {
				panic("should set user env.")
			}

			// 此处不会执行
			fmt.Println("after panic")
		}()
	}()

	time.Sleep(100)
	fmt.Println("end of main function")
}
```

上面的panic最终会被recover捕获到。这样的处理方式在一个http server的主流程常常会被用到。一次偶然的请求可能会触发某个bug, 这时用recover捕获panic, 稳住主流程，不影响其他请求。

程序员通过监控获知此次panic的发生，按时间点定位到日志相应位置，找到发生panic的原因，三下五除二，修复上线。一看四周，大家都埋头干自己的事，简直完美：偷偷修复了一个bug, 没有发现！嘿嘿！



# go 中 defer Close() 的潜在风险
作为一名 Gopher，我们很容易形成一个编程惯例：每当有一个实现了 `io.Closer` 接口的对象 `x` 时，在得到对象并检查错误之后，会立即使用 `defer x.Close()` 以保证函数返回时 `x` 对象的关闭 。以下给出两个惯用写法例子。

-   HTTP 请求

```go
resp, err := http.Get("https://go.google.cn/")
if err != nil {
    return err
}
defer resp.Body.Close()
// The following code: handle resp
```

-   访问文件

```go
f, err := os.Open("/home/goshare/gopher.txt")
if err != nil {
    return err
}
defer f.Close()
// The following code: handle f
```

## 存在问题

实际上，这种写法是存在潜在问题的。`defer x.Close()` 会忽略它的返回值，但在执行 `x.Close()` 时，我们并不能保证 `x` 一定能正常关闭，万一它返回错误应该怎么办？这种写法，会让程序有可能出现非常难以排查的错误。

那么，`Close()` 方法会返回什么错误呢？在 POSIX 操作系统中，例如 Linux 或者 maxOS，关闭文件的 `Close()` 函数最终是调用了系统方法 `close()`，我们可以通过 `man close` 手册，查看 `close()` 可能会返回什么错误

```text
ERRORS
     The close() system call will fail if:

     [EBADF]            fildes is not a valid, active file descriptor.

     [EINTR]            Its execution was interrupted by a signal.

     [EIO]              A previously-uncommitted write(2) encountered an
                        input/output error.
```

错误 `EBADF` 表示无效文件描述符 fd，与本文中的情况无关；`EINTR` 是指的 Unix 信号打断；那么本文中可能存在的错误是 `EIO`。

`EIO` 的错误是指**未提交读**，这是什么错误呢？

![](https://pic4.zhimg.com/80/v2-a2ed62094e1fc4c49338e4aa08e26347_720w.jpg)

`EIO` 错误是指文件的 `write()` 的读还未提交时就调用了 `close()` 方法。

上图是一个经典的计算机存储器层级结构，在这个层次结构中，从上至下，设备的访问速度越来越慢，容量越来越大。存储器层级结构的主要思想是上一层的存储器作为低一层存储器的高速缓存。

CPU 访问寄存器会非常之快，相比之下，访问 RAM 就会很慢，而访问磁盘或者网络，那意味着就是蹉跎光阴。如果每个 `write()` 调用都将数据同步地提交到磁盘，那么系统的整体性能将会极度降低，而我们的计算机是不会这样工作的。当我们调用 `write()` 时，数据并没有立即被写到目标载体上，计算机存储器每层载体都在缓存数据，在合适的时机下，将数据刷到下一层载体，这将写入调用的同步、缓慢、阻塞的同步转为了快速、异步的过程。

这样看来，`EIO` 错误的确是我们需要提防的错误。这意味着如果我们尝试将数据保存到磁盘，在 `defer x.Close()` 执行时，操作系统还并未将数据刷到磁盘，这时我们应该获取到该错误提示（**只要数据还未落盘，那数据就没有持久化成功，它就是有可能丢失的，例如出现停电事故，这部分数据就永久消失了，且我们会毫不知情**）。但是按照上文的惯例写法，我们程序得到的是 `nil` 错误。

## 解决方案

我们针对关闭文件的情况，来探讨几种可行性改造方案

-   第一种方案，那就是不使用 defer

```go
func solution01() error {
    f, err := os.Create("/home/goshare/gopher.txt")
    if err != nil {
        return err
    }

    if _, err = io.WriteString(f, "hello gopher"); err != nil {
        f.Close()
        return err
    }

    return f.Close()
}
```

这种写法就需要我们在 `io.WriteString` 执行失败时，明确调用 `f.Close()` 进行关闭。但是这种方案，需要在每个发生错误的地方都要加上关闭语句 `f.Close()`，如果对 `f` 的写操作 case 较多，容易存在遗漏关闭文件的风险。

-   第二种方案是，通过命名返回值 err 和闭包来处理

```go
func solution02() (err error) {
    f, err := os.Create("/home/goshare/gopher.txt")
    if err != nil {
        return
    }

    defer func() {
        closeErr := f.Close()
        if err == nil {
            err = closeErr
        }
    }()

    _, err = io.WriteString(f, "hello gopher")
    return
}
```

这种方案解决了方案一中忘记关闭文件的风险，如果有更多 `if err !=nil` 的条件分支，这种模式可以有效降低代码行数。

-   第三种方案是，在函数最后 return 语句之前，显示调用一次 f.Close()

```go
func solution03() error {
    f, err := os.Create("/home/goshare/gopher.txt")
    if err != nil {
        return err
    }
    defer f.Close()

    if _, err := io.WriteString(f, "hello gopher"); err != nil {
        return err
    }

    if err := f.Close(); err != nil {
        return err
    }
    return nil
}
```

这种解决方案能在 `io.WriteString` 发生错误时，由于 `defer f.Close()` 的存在能得到 `close` 调用。也能在 `io.WriteString` 未发生错误，但缓存未刷新到磁盘时，得到 `err := f.Close()` 的错误，而且由于 `defer f.Close()` 并不会返回错误，所以并不担心两次 `Close()` 调用会将错误覆盖。

-   最后一种方案是，函数 return 时执行 f.Sync()

```go
func solution04() error {
    f, err := os.Create("/home/goshare/gopher.txt")
    if err != nil {
        return err
    }
    defer f.Close()

    if _, err = io.WriteString(f, "hello world"); err != nil {
        return err
    }

    return f.Sync()
}
```

由于调用 `close()` 是最后一次获取操作系统返回错误的机会，但是在我们关闭文件时，缓存不一定被会刷到磁盘上。那么，我们可以调用 `f.Sync()` （其内部调用系统函数 `fsync` ）强制性让内核将缓存持久到磁盘上去。

```go
// Sync commits the current contents of the file to stable storage.
// Typically, this means flushing the file system's in-memory copy
// of recently written data to disk.
func (f *File) Sync() error {
    if err := f.checkValid("sync"); err != nil {
        return err
    }
    if e := f.pfd.Fsync(); e != nil {
        return f.wrapErr("sync", e)
    }
    return nil
}
```

由于 `fsync` 的调用，这种模式能很好地避免 `close` 出现的 `EIO`。可以预见的是，由于强制性刷盘，这种方案虽然能很好地保证数据安全性，但是在执行效率上却会大打折扣。

