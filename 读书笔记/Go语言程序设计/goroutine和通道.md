#channel #goroutine #锁

# 协程与channel

go有两种并发编程的风格，goroutine和channel支持通信顺序进程（CSP），CSP是一个并发的模式，在不同执行体（goroutine）之间传递值，但是变量本身局限于单一的执行体。

如果说goruntine是go程序并发的执行体，通道就是它们之间的连接。通道是可以让一个goroutine发送特定值到另一个goroutine的通信机制。每一个通道是一个具体类型的导管，叫做通道的元素类型。一个有int类型元素的通道写为chan int。
![[Pasted image 20220211153140.png]]
通道支持第三个操作：关闭（close）,它设置一个标志位来指示值当前已经发送完毕，这个通道后面没有值了；关闭后的发送操作将导致宕机。在一个已经关闭的通道上进行接收操作，将获取所有已经发送的值，直到通道为空；这时接受操作会立即完成，同时获取到一个通道元素类型对应的零值。

# 互斥锁：sync.Mutex
![[Pasted image 20220211160321.png]]

# 读写互斥锁：sync.RWMutex

# 延迟初始化：sync.Once

# 竞态检测器
![[Pasted image 20220211162144.png]]

# goroutine与线程
## 可增长的栈
![[Pasted image 20220211162430.png]]

## goruntine调度
![[Pasted image 20220211162749.png]]

## goroutine没有标识
![[Pasted image 20220211162942.png]]

