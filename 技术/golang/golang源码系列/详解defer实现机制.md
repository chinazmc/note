# 基本使用
我们首先来看一看defer关键字是怎么使用的，一个经典的场景就是我们在使用事务时，发生错误需要回滚，这时我们就可以用使用defer来保证程序退出时保证事务回滚，示例代码如下：

// 代码摘自之前写的 Leaf-segment数据库获取ID方案：https://github.com/asong2020/go-algorithm/blob/master/leaf/dao/leaf_dao.go
```go
func (l *LeafDao) NextSegment(ctx context.Context, bizTag string) (*model.Leaf, error) {
 // 开启事务
 tx, err := l.sql.Begin()
 defer func() {
  if err != nil {
   l.rollback(tx)
  }
 }()
 if err = l.checkError(err); err != nil {
  return nil, err
 }
 err = l.db.UpdateMaxID(ctx, bizTag, tx)
 if err = l.checkError(err); err != nil {
  return nil, err
 }
 leaf, err := l.db.Get(ctx, bizTag, tx)
 if err = l.checkError(err); err != nil {
  return nil, err
 }
 // 提交事务
 err = tx.Commit()
 if err = l.checkError(err); err != nil {
  return nil, err
 }
 return leaf, nil
}
```
上面只是一个简单的应用，defer还有一些特性，如果你不知道，使用起来可能会踩到一些坑，尤其是跟带命名的返回参数一起使用时。下面我们我先来带大家踩踩坑。

# defer的注意事项和细节
## defer调用顺序
我们先来看一道题，你能说他的答案吗？

```go
func main() {
    fmt.Println("reciprocal")

    for i := 0; i < 10; i++ {
        defer fmt.Println(i)
    }
}
```
答案：

```sh
reciprocal
9
8
7
6
5
4
3
2
1
0
```
看到答案，你是不是产生了疑问？这就对了，我最开始学golang时也有这个疑问，这个跟栈一样，即"先进后出"特性，越后面的defer表达式越先被调用。所以这里大家关闭依赖资源时一定要注意defer调用顺序。

## defer拷贝
我们先来看这样一段代码，你能说出defer中num1和num2的值是多少吗？

```go
func main() {
 fmt.Println(Sum(1, 2))
}

func Sum(num1, num2 int) int {
 defer fmt.Println("num1:", num1)
 defer fmt.Println("num2:", num2)
 num1++
 num2++
 return num1 + num2
}
```

聪明的你一定会说："这也太简单了，答案就是num1等于2，num2等于3"。很遗憾的告诉你，错了，正确的答案是num1为1,num2为2，这两个变量并不受num1++、num2++的影响，因为defer将语句放入到栈中时，也会将相关的值拷贝同时入栈。

## defer与return
的返回时机
这里我先说结论，总结一下就是，函数的整个返回过程应该是：
- return 对返回变量赋值，如果是匿名返回值就先声明再赋值；
- 执行 defer 函数；
- return 携带返回值返回。
下面我们来看两道题，你知道他们的返回值是多少吗？

匿名返回值函数
```go
// 匿名函数
func Anonymous() int {
 var i int
 defer func() {
  i++
  fmt.Println("defer2 value is ", i)
 }()

 defer func() {
  i++
  fmt.Println("defer1 in value is ", i)
 }()

 return i
}
```
命名返回值的函数
```go
func HasName() (j int) {
 defer func() {
  j++
  fmt.Println("defer2 in value", j)
 }()

 defer func() {
  j++
  fmt.Println("defer1 in value", j)
 }()

 return j
}
```
先来公布一下答案吧：

1. Anonymous()的返回值为0
2. HasName()的返回值为2
从这我们可以看出命名返回值的函数的返回值被 defer 修改了。这里想必大家跟我一样，都很疑惑，带着疑惑我查阅了一下go官方文档，文档指出，defer的执行顺序有以下三个规则：

A deferred function’s arguments are evaluated when the defer statement is evaluated.
Deferred function calls are executed in Last In First Out order after the surrounding function returns.
Deferred functions may read and assign to the returning function’s named return values.
规则3就可以印证为什么命名返回值的函数的返回值被更改了，其实在函数最终返回前，defer 函数就已经执行了，在命名返回值的函数 中，由于返回值已经被提前声明，所以 defer 函数能够在 return 语句对返回值赋值之后，继续对返回值进行操作，操作的是同一个变量，而匿名返回值函数中return先返回，已经进行了一次值拷贝r=i，defer函数中再次对变量i的操作并不会影响返回值。

这里可能有些小伙伴还不是很懂，我在讲一下return返回步骤，相信你们会豁然开朗。

函数在返回时，首先函数返回时会自动创建一个返回变量假设为ret(如果是命名返回值的函数则不会创建)，函数返回时要将变量i赋值给ret，即有ret = i。
然后检查函数中是否有defer存在，若有则执行defer中部分。
最后返回ret
现在你们应该知道上面是什么原因了吧～。

# 解密defer源码
写在开头：go版本1.15.3

我们先来写一段代码，查看一下汇编代码：

```go
func main() {
 defer func() {
  fmt.Println("asong 真帅")
 }()
}
```
执行如下指令：go tool compile -N -l -S main.go，截取部分汇编指令如下：

![[Pasted image 20230306194830.png]]

我们可以看出来，从执行流程来看首先会调用deferproc来创建defer，然后在函数返回时插入了指令CALL runtime.deferreturn。知道了defer在流程中是通过这两个方法是调用的，接下来我们来看一看defer的结构：

```go
// go/src/runtime/runtime2.go
type _defer struct {
 siz     int32 // includes both arguments and results
 started bool
 heap    bool
 // openDefer indicates that this _defer is for a frame with open-coded
 // defers. We have only one defer record for the entire frame (which may
 // currently have 0, 1, or more defers active).
 openDefer bool
 sp        uintptr  // sp at time of defer
 pc        uintptr  // pc at time of defer
 fn        *funcval // can be nil for open-coded defers
 _panic    *_panic  // panic that is running defer
 link      *_defer

 // If openDefer is true, the fields below record values about the stack
 // frame and associated function that has the open-coded defer(s). sp
 // above will be the sp for the frame, and pc will be address of the
 // deferreturn call in the function.
 fd   unsafe.Pointer // funcdata for the function associated with the frame
 varp uintptr        // value of varp for the stack frame
 // framepc is the current pc associated with the stack frame. Together,
 // with sp above (which is the sp associated with the stack frame),
 // framepc/sp can be used as pc/sp pair to continue a stack trace via
 // gentraceback().
 framepc uintptr
}
```
这里简单介绍一下runtime._defer结构体中的几个字段：

>siz代表的是参数和结果的内存大小
sp和pc分别代表栈指针和调用方的程序计数器
fn代表的是defer关键字中传入的函数
\_panic是触发延迟调用的结构体，可能为空
openDefer表示的是当前defer是否已经开放编码优化(1.14版本新增)
link所有runtime.\_defer结构体都通过该字段串联成链表

先来我们也知道了defer关键字的数据结构了，下面我们就来重点分析一下deferproc和deferreturn函数是如何调用。

## deferproc函数
deferproc函数也不长，我先贴出来代码；

```go
// proc/panic.go
// Create a new deferred function fn with siz bytes of arguments.
// The compiler turns a defer statement into a call to this.
//go:nosplit
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
	 gp := getg()
	 if gp.m.curg != gp {
	  // go code on the system stack can't defer
	  throw("defer on system stack")
	 }
	
	 // the arguments of fn are in a perilous state. The stack map
	 // for deferproc does not describe them. So we can't let garbage
	 // collection or stack copying trigger until we've copied them out
	 // to somewhere safe. The memmove below does that.
	 // Until the copy completes, we can only call nosplit routines.
	 sp := getcallersp()
	 argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	 callerpc := getcallerpc()
	
	 d := newdefer(siz)
	 if d._panic != nil {
	  throw("deferproc: d.panic != nil after newdefer")
	 }
	 d.link = gp._defer
	 gp._defer = d
	 d.fn = fn
	 d.pc = callerpc
	 d.sp = sp
	 switch siz {
		 case 0:
		  // Do nothing.
		 case sys.PtrSize:
		  *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
		 default:
		  memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	 }
	
	 // deferproc returns 0 normally.
	 // a deferred func that stops a panic
	 // makes the deferproc return 1.
	 // the code the compiler generates always
	 // checks the return value and jumps to the
	 // end of the function if deferproc returns != 0.
	 return0()
	 // No code can go here - the C return register has
	 // been set and must not be clobbered.
}
```
上面介绍了rumtiem.\_defer结构想必这里的入参是什么意思就不用我介绍了吧。

deferproc的函数流程很清晰，首先他会通过newdefer函数分配一个_defer结构对象，然后把需要延迟执行的函数以及该函数需要用到的参数、调用deferproc函数时的rps寄存器的值以及deferproc函数的返回地址保存在_defer结构体对象中，最后通过return0()设置rax寄存器的值为0隐性的给调用者返回一个0值。deferproc主要是靠newdefer来分配_defer结构体对象的，下面我们一起来看看newdefer实现，代码有点长：

```go
// proc/panic.go
// Allocate a Defer, usually using per-P pool.
// Each defer must be released with freedefer.  The defer is not
// added to any defer chain yet.
//
// This must not grow the stack because there may be a frame without
// stack map information when this is called.
//
//go:nosplit
func newdefer(siz int32) *_defer {
	 var d *_defer
	 sc := deferclass(uintptr(siz))
	 gp := getg()//获取当前goroutine的g结构体对象
	 if sc < uintptr(len(p{}.deferpool)) {
	  pp := gp.m.p.ptr() //与当前工作线程绑定的p
	  if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
	   // Take the slow path on the system stack so
	   // we don't grow newdefer's stack.
	   systemstack(func() {
	    lock(&sched.deferlock) 
	         //把新分配出来的d放入当前goroutine的_defer链表头
	    for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
	     d := sched.deferpool[sc]
	     sched.deferpool[sc] = d.link
	     d.link = nil
	     pp.deferpool[sc] = append(pp.deferpool[sc], d)
	    }
	    unlock(&sched.deferlock)
	   })
	  }
	  if n := len(pp.deferpool[sc]); n > 0 {
	   d = pp.deferpool[sc][n-1]
	   pp.deferpool[sc][n-1] = nil
	   pp.deferpool[sc] = pp.deferpool[sc][:n-1]
	  }
	 }
	 if d == nil {
	    //如果p的缓存中没有可用的_defer结构体对象则从堆上分配
	         // Allocate new defer+args.
	         //因为roundupsize以及mallocgc函数都不会处理扩栈，所以需要切换到系统栈执行
	  // Allocate new defer+args.
	  systemstack(func() {
	   total := roundupsize(totaldefersize(uintptr(siz)))
	   d = (*_defer)(mallocgc(total, deferType, true))
	  })
	  if debugCachedWork {
	   // Duplicate the tail below so if there's a
	   // crash in checkPut we can tell if d was just
	   // allocated or came from the pool.
	   d.siz = siz
	       //把新分配出来的d放入当前goroutine的_defer链表头
	   d.link = gp._defer
	   gp._defer = d
	   return d
	  }
	 }
	 d.siz = siz
	 d.heap = true
	 return d
}
```
newdefer函数首先会尝试从当前工作线程绑定的p的_defer对象池和全局对象池中获取一个满足大小要求(sizeof(\_defer) + siz向上取整至16的倍数)的_defer 结构体对象，如果没有能够满足要求的空闲 \_defer对象则从堆上分一个，最后把分配到的对象链入当前 goroutine的_defer 链表的表头。

到此deferproc函数就分析完了，你们懂了吗? 没懂不要紧，我们再来总结一下这个过程：

首先编译器把defer语句翻译成对应的deferproc函数的调用
然后deferproc函数通过newdefer函数分配一个_defer结构体对象并放入当前的goroutine的_defer链表的表头；
在 \_defer 结构体对象中保存被延迟执行的函数 fn 的地址以及 fn 所需的参数
返回到调用 deferproc 的函数继续执行后面的代码。
## deferreturn函数
```go
// Run a deferred function if there is one.
// The compiler inserts a call to this at the end of any
// function which calls defer.
// If there is a deferred function, this will call runtime·jmpdefer,
// which will jump to the deferred function such that it appears
// to have been called by the caller of deferreturn at the point
// just before deferreturn was called. The effect is that deferreturn
// is called again and again until there are no more deferred functions.
//
// Declared as nosplit, because the function should not be preempted once we start
// modifying the caller's frame in order to reuse the frame to call the deferred
// function.
//
// The single argument isn't actually used - it just has its address
// taken so it can be matched against pending defers.
//go:nosplit
func deferreturn(arg0 uintptr) {
 gp := getg() //获取当前goroutine对应的g结构体对象
 d := gp._defer //获取当前goroutine对应的g结构体对象
 if d == nil {
    //没有需要执行的函数直接返回，deferreturn和deferproc是配对使用的
         //为什么这里d可能为nil？因为deferreturn其实是一个递归调用，这个是递归结束条件之一
  return
 }
 sp := getcallersp() //获取调用deferreturn时的栈顶位置
 if d.sp != sp { // 递归结束条件
    //如果保存在_defer对象中的sp值与调用deferretuen时的栈顶位置不一样，直接返回
        //因为sp不一样表示d代表的是在其他函数中通过defer注册的延迟调用函数，比如:
        //a()->b()->c()它们都通过defer注册了延迟函数，那么当c()执行完时只能执行在c中注册的函数
  return
 }
 if d.openDefer {
  done := runOpenDeferFrame(gp, d)
  if !done {
   throw("unfinished open-coded defers in deferreturn")
  }
  gp._defer = d.link
  freedefer(d)
  return
 }

 // Moving arguments around.
 //
 // Everything called after this point must be recursively
 // nosplit because the garbage collector won't know the form
 // of the arguments until the jmpdefer can flip the PC over to
 // fn.
      //把保存在_defer对象中的fn函数需要用到的参数拷贝到栈上，准备调用fn
    //注意fn的参数放在了调用调用者的栈帧中，而不是此函数的栈帧中
 switch d.siz {
 case 0:
  // Do nothing.
 case sys.PtrSize:
  *(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
 default:
  memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
 }
 fn := d.fn
 d.fn = nil
 gp._defer = d.link // 使gp._defer指向下一个_defer结构体对象
    //因为需要调用的函数d.fn已经保存在了fn变量中，它的参数也已经拷贝到了栈上，所以释放_defer结构体对象
 freedefer(d)
 // If the defer function pointer is nil, force the seg fault to happen
 // here rather than in jmpdefer. gentraceback() throws an error if it is
 // called with a callback on an LR architecture and jmpdefer is on the
 // stack, because the stack trace can be incorrect in that case - see
 // issue #8153).
 _ = fn.fn
 jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```
deferreturn函数主要流程还是简单一些的，我们来分析一下：

首先我们通过当前goroutine对应的g结构体对象的_defer链表判断是否有需要执行的defered函数，如果没有则返回；这里的没有是指g.\_defer== nil 或者defered函数不是在deferteturn的caller函数中注册的函数。
然后我们在从_defer对象中把defered函数需要的参数拷贝到栈上，并释放_defer的结构体对象。
最红调用jmpderfer函数调用defered函数，也就是defer关键字中传入的函数.
jmpdefer函数实现挺优雅的，我们一起来看看他是如何实现的：

```
// runtime/asm_amd64.s : 581
// func jmpdefer(fv *funcval, argp uintptr)
// argp is a caller SP.
// called from deferreturn.
// 1. pop the caller
// 2. sub 5 bytes from the callers return
// 3. jmp to the argument
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-16
 MOVQ fv+0(FP), DX // fn
 MOVQ argp+8(FP), BX // caller sp
 LEAQ -8(BX), SP // caller sp after CALL
 MOVQ -8(SP), BP // restore BP as if deferreturn returned (harmless if framepointers not in use)
 SUBQ $5, (SP) // return to CALL again
 MOVQ 0(DX), BX
 JMP BX // but first run the deferred function
```
这里都是汇编，大家可能看不懂，没关系，我只是简单介绍一下这里，有兴趣的同学可以去查阅一下相关知识再来深入了解。

MOVQ fv+0(FP), DX这条指令会把jmpdefer的第一个参数也就是结构体对象fn的地址存放入DX寄存器，之后的代码就可以通过寄存器访问到fn，fn就可以拿到defer关键字中传入的函数，对应上面的例子就是匿名函数func(){}().
MOVQ argp+8(FP), BX这条指令就是把jmpdefer的第二个参数放入BX寄存器，该参数是一个指针，他指向defer关键字中传入的函数的第一个参数.
LEAQ -8(BX), SP这条指令的作用是让 SP 寄存器指向 deferreturn 函数的返回地址所在的栈内存单元.
MOVQ -8(SP), BP这条指令的作用是调整 BP 寄存器的值，此时SP_8的位置存放的是defer关键字当前所在的函数的rbp寄存器的值，所以这条指令在调整rbp寄存器的值使其指向当前所在函数的栈帧的适当位置.
SUBQ $5, (SP)这里的作用是完成defer函数的参数以及执行完函数后返回地址在栈上的构造.因为在执行这条指令时，rsp寄存器指向的是deferreturn函数的返回地址.
MOVQ 0(DX), BX和JMP BX放到一起说吧，目的是跳转到对应defer函数去执行，完成defer函数的调用.
总结
大概分析了一下defer的实现机制，但还是有点蒙圈，最后在总结一下这里：

>
1、首先编译器会把defer语句翻译成对deferproc函数的调用。
2、然后deferproc函数会负责调用newdefer函数分配一个_defer结构体对象并放入当前的goroutine的_defer链表的表头；
3、然后编译起会在defer所在函数的结尾处插入对deferreturn的调用，deferreturn负责递归的调用某函数(defer语句所在函数)通过defer语句注册的函数。

总体来说就是这三个步骤，go语言对defer的实现机制就是这样啦，你明白了吗？

# 小试牛刀
上面我们也细致学习了一下defer，下面出几道题吧，看看你们真的学会了吗？

问题1
```go
// 测试1
func Test1() (r int) {
 i := 1
 defer func() {
  i = i + 1
 }()
 return i
}
```
返回结果是什么？
1
问题2
```
func Test2() (r int) {
 defer func(r int) {
  r = r + 2
 }(r)
 return 2
}
```
返回结果是什么？
2
如果改成这样呢？
```go
func Test3() (r int) {
 defer func(r *int) {
  *r = *r + 2
 }(&r)
 return 2
}
```
4
问题3
```go

func main(){
  e1()
  e2()
  e3()
}
func e1() {
 var err error
 defer fmt.Println(err)
 err = errors.New("e1 defer err")
}

func e2() {
 var err error
 defer func() {
  fmt.Println(err)
 }()
 err = errors.New("e2 defer err")
}

func e3() {
 var err error
 defer func(err error) {
  fmt.Println(err)
 }(err)
 err = errors.New("e3 defer err")
}
```
这个返回结果又是什么呢？
nil
e2 defer err
nil
