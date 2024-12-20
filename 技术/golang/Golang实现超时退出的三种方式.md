前段时间发现线上有个服务接口，总是间歇性告警，有时候一天两三次，有时候一天都没有。

告警的逻辑是在一个接口中异步调用了另一个HTTP接口，这个HTTP接口调用出现超时。但是我去问了负责这个HTTP接口的同学，人家说他们的接口相应都是毫秒级别，还截图监控了，有图有真相，我还能说啥。

但是，超时是确实存在的，只是请求还可能没有到人家服务那边。

这种偶发性问题不好复现，偶尔来个告警也挺烦的，第一反应还是先解决问题，思路也简单，失败后重试。

**解决方法**

且不谈重试策略，先说说什么时候触发重试。

我们可以在接口请求出错抛出err的时候重试，但是这种不好控制，如果一个请求出去，十来秒都没有响应，则这个协程就要傻傻的等他报错才能重试，浪费生命啊~

所以结合上面同学给出的毫秒级响应指标，可以设定一个超时时间，如果在指定超时时间后没有返回结果，则重试（这篇重试不是重点）。
```go
func AsyncCall() {
 ctx, cancel := context.WithTimeout(context.Background(), time.Duration(time.Millisecond*800))
 defer cancel()
 go func(ctx context.Context) {
 // 发送HTTP请求
 }()

 select {
 case <-ctx.Done():
 fmt.Println("call successfully!!!")
 return
 case <-time.After(time.Duration(time.Millisecond * 900)):
 fmt.Println("timeout!!!")
 return
 }
}
```

**说明**

1、通过context的WithTimeout设置一个有效时间为800毫秒的context。

2、该context会在耗尽800毫秒后或者方法执行完成后结束，结束的时候会向通道ctx.Done发送信号。

3、有人可能要问，你这里已经设置了context的有效时间，为什么还要加上这个time.After呢？

这是因为该方法内的context是自己申明的，可以手动设置对应的超时时间，但是在大多数场景，这里的ctx是从上游一直传递过来的，对于上游传递过来的context还剩多少时间，我们是不知道的，所以这时候通过time.After设置一个自己预期的超时时间就很有必要了。

4、注意，这里要记得调用`cancel()`，不然即使提前执行完了，还要傻傻等到800毫秒后context才会被释放。

**总结**

上面的超时控制是搭配使用了`ctx.Done`和time.After。

Done通道负责监听context啥时候完事，如果在time.After设置的超时时间到了，你还没完事，那我就不等了，执行超时后的逻辑代码。

**举一反三**

那么，除了上面这种超时控制策略，还有其他的套路吗？

有，但是大同小异。

**第一种：使用time.NewTimer  **

```go
func AsyncCall() {
 ctx, cancel := context.WithTimeout(context.Background(), time.Duration(time.Millisecond * 800))
 defer cancel()
 timer := time.NewTimer(time.Duration(time.Millisecond * 900))

 go func(ctx context.Context) {
 // 发送HTTP请求
 }()

 select {
 case <-ctx.Done():
 timer.Stop()
 timer.Reset(time.Second)
 fmt.Println("call successfully!!!")
 return
 case <-timer.C:
 fmt.Println("timeout!!!")
 return
 }
}
```

这里的主要区别是将time.After换成了`time.NewTimer`，也是同样的思路如果接口调用提前完成，则监听到Done信号，然后关闭定时器。

否则的话，会在指定的timer即900毫秒后执行超时后的业务逻辑。

**第二种：使用通道 

```go
func AsyncCall() {
 ctx := context.Background()
 done := make(chan struct{}, 1)

 go func(ctx context.Context) {
 // 发送HTTP请求
 done <- struct{}{}
 }()

 select {
 case <-done:
 fmt.Println("call successfully!!!")
 return
 case <-time.After(time.Duration(800 * time.Millisecond)):
 fmt.Println("timeout!!!")
 return
 }
}
```

1、这里主要利用通道可以在协程之间通信的特点，当调用成功后，向done通道发送信号。

2、监听Done信号，如果在time.After超时时间之前接收到，则正常返回，否则走向time.After的超时逻辑，执行超时逻辑代码。

3、这里使用的是通道和time.After组合，也可以使用通道和time.NewTimer组合。

**总结**

本篇主要介绍如何实现超时控制，主要有三种

1、context.WithTimeout/context.WithDeadline + time.After

2、context.WithTimeout/context.WithDeadline + time.NewTimer

3、channel + time.After/time.NewTimer