虽然 WaitGroup 已经帮我们做了很好的封装，但是仍然存在一些问题，例如如果需要返回错误，或者只要一个 goroutine 出错我们就不再等其他 goroutine 了，减少资源浪费，这些 WaitGroup 都不能很好的解决，这时候就派出本文的选手 errgroup 出场了。

# 函数签名

```go
type Group  

func WithContext(ctx context.Context) (*Group, context.Context)    
func (g *Group) Go(f func() error)    
func (g *Group) Wait() error
```

整个包就一个 Group 结构体

-   通过 `WithContext`  可以创建一个带取消的 `Group`
-   当然除此之外也可以零值的 Group 也可以直接使用，但是出错之后就不会取消其他的 goroutine 了
-   `Go`  方法传入一个 `func() error`  内部会启动一个 goroutine 去处理
-   `Wait`  类似 WaitGroup 的 Wait 方法，等待所有的 goroutine 结束后退出，返回的错误是一个出错的 err

# 源码

# Group

```
type Group struct {    
	// context 的 cancel 方法	
	cancel func()    
	// 复用 WaitGroup	
	wg sync.WaitGroup	
	// 用来保证只会接受一次错误	
	errOnce sync.Once    
	// 保存第一个返回的错误	
	err     error
}
```

## WithContext

```go
func WithContext(ctx context.Context) (*Group, context.Context) {	
	ctx, cancel := context.WithCancel(ctx)	
	return &Group{cancel: cancel}, ctx
}
```

`WithContext`  就是使用 `WithCancel`  创建一个可以取消的 context 将 cancel 赋值给 Group 保存起来，然后再将 context 返回回去

注意这里有一个坑，在后面的代码中不要把这个 ctx 当做父 context 又传给下游，因为 errgroup 取消了，这个 context 就没用了，会导致下游复用的时候出错

## Go

```go
func (g *Group) Go(f func() error) {	
	g.wg.Add(1)	
	go func() {		
		defer g.wg.Done()		
		if err := f(); err != nil {			
			g.errOnce.Do(func() {				
				g.err = err				
				if g.cancel != nil {					
					g.cancel()				
				}			
			})		
		}	
	}()
}
```

`Go`  方法其实就类似于 `go`  关键字，会启动一个携程，然后利用 `waitgroup`  来控制是否结束，如果有一个非 `nil`  的 error 出现就会保存起来并且如果有 `cancel`  就会调用 `cancel`  取消掉，使 `ctx`  返回

## Wait

```
func (g *Group) Wait() error {	
	g.wg.Wait()	
	if g.cancel != nil {		
		g.cancel()	
	}	
	return g.err
}
```

`Wait`  方法其实就是调用 `WaitGroup`  等待，如果有 `cancel`  就调用一下

# 案例

> 这个其实是 week03 的作业

基于  errgroup  实现一个  http server  的启动和关闭  ，以及  linux signal  信号的注册和处理，要保证能够   一个退出，全部注销退出。
```go
func main() {  
   g, ctx := errgroup.WithContext(context.Background())  
  
   mux := http.NewServeMux()  
   mux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {  
      w.Write([]byte("pong"))  
   })  
  
   // 模拟单个服务错误退出  
   serverOut := make(chan struct{})  
   mux.HandleFunc("/shutdown", func(w http.ResponseWriter, r *http.Request) {  
      serverOut <- struct{}{}  
   })  
  
   server := http.Server{  
      Handler: mux,  
      Addr:    ":8080",  
   }  
  
   // g1  
   // g1 退出了所有的协程都能退出么？  
   // g1 退出后, context 将不再阻塞，g2, g3 都会随之退出  
   // 然后 main 函数中的 g.Wait() 退出，所有协程都会退出  
   g.Go(func() error {  
      return server.ListenAndServe()  
   })  
  
   // g2  
   // g2 退出了所有的协程都能退出么？  
   // g2 退出时，调用了 shutdown，g1 会退出  
   // g2 退出后, context 将不再阻塞，g3 会随之退出  
   // 然后 main 函数中的 g.Wait() 退出，所有协程都会退出  
   g.Go(func() error {  
      select {  
      case <-ctx.Done():  
         log.Println("errgroup exit...")  
      case <-serverOut:  
         log.Println("server will out...")  
      }  
  
      timeoutCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)  
      // 这里不是必须的，但是如果使用 _ 的话静态扫描工具会报错，加上也无伤大雅  
      defer cancel()  
  
      log.Println("shutting down server...")  
      return server.Shutdown(timeoutCtx)  
   })  
  
   // g3  
   // g3 捕获到 os 退出信号将会退出  
   // g3 退出了所有的协程都能退出么？  
   // g3 退出后, context 将不再阻塞，g2 会随之退出  
   // g2 退出时，调用了 shutdown，g1 会退出  
   // 然后 main 函数中的 g.Wait() 退出，所有协程都会退出  
   g.Go(func() error {  
      quit := make(chan os.Signal, 0)  
      signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)  
  
      select {  
      case <-ctx.Done():  
         return ctx.Err()  
      case sig := <-quit:  
         return errors.Errorf("get os signal: %v", sig)  
      }  
   })  
  
   fmt.Printf("errgroup exiting: %+v\n", g.Wait())  
}
```

这里主要用到了 errgroup 一个出错，其余取消的能力
# 补充
当然，上面这个 case 是我们正好只有一个报错，但是，如果实际有多个任务都挂了呢？

从完备性来考虑，有没有什么办法能够将多个任务的错误都返回呢？

这一点其实 errgroup 库并没有提供非常好的支持，需要开发者自行做一些改造。因为 Group 中只有一个 err 变量，我们不可能基于 Group 来实现这一点。通常情况下，我们会创建一个 slice 来存储 `f()` 执行的 err。

```go

package main

import (
    "errors"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    var g errgroup.Group
    var result = make([]error, 3)

    // 启动第一个子任务,它执行成功
    g.Go(func() error {
        time.Sleep(5 * time.Second)
        fmt.Println("exec #1")
        result[0] = nil // 保存成功或者失败的结果
        return nil
    })


    // 启动第二个子任务，它执行失败
    g.Go(func() error {
        time.Sleep(10 * time.Second)
        fmt.Println("exec #2")

        result[1] = errors.New("failed to exec #2") // 保存成功或者失败的结果
        return result[1]
    })

    // 启动第三个子任务，它执行成功
    g.Go(func() error {
        time.Sleep(15 * time.Second)
        fmt.Println("exec #3")
        result[2] = nil // 保存成功或者失败的结果
        return nil
    })

    if err := g.Wait(); err == nil {
        fmt.Printf("Successfully exec all. result: %v\n", result)
    } else {
        fmt.Printf("failed: %v\n", result)
    }
}
复制代码
```

可以看到，我们声明了一个 result slice，长度为 3。这里不用担心并发问题，因为每个 goroutine 读写的位置是确定唯一的。

本质上可以理解为，我们把 `f()` 返回的 err 不仅仅给了 Group 一份，还自己存了一份，当出错的时候，Wait 返回的错误我们不一定真的用，而是直接用自己错的这一个 error slice。

补充一个超时代码：
```go
package main  
  
import (  
   "context"  
   "errors"   "fmt"   "golang.org/x/sync/errgroup"   "golang.org/x/sync/semaphore"   "time")  
  
func main() {  
   args := make([]interface{}, 20)  
   for i := range args {  
      args[i] = fmt.Sprintf("arg:%d", i)  
   }  
   results, err := run(args, 5, func(i int, arg interface{}) (result interface{}, err error) {  
      if i == 10 {  
         return "", errors.New("test error")  
      }  
      result = mockTask(arg)  
      return  
   })  
   if err != nil {  
      fmt.Printf("catch error:%v\n", err)  
   } else {  
      fmt.Println(results)  
   }  
}  
  
// 接收一组参数，通过n个并发去执行  
func run(args []interface{}, n int64, doWork func(i int, arg interface{}) (result interface{}, err error)) (results []interface{}, err error) {  
   //ctx := context.Background()  
   ctx,cancel := context.WithTimeout(context.Background(),6*time.Second)  
   defer cancel()  
   eg, ctx := errgroup.WithContext(ctx)  
   sem := semaphore.NewWeighted(n)  
   results = make([]interface{}, len(args))  
   for i := range args {  
      i := i  
      if err = sem.Acquire(ctx, 1); err != nil {  
         break  
      }  
      eg.Go(func() error {  
         defer sem.Release(1)  
         fmt.Printf("任务%d开始\n", i)  
         resultCh := make(chan result)  
         go func() {  
            ret, err := doWork(i, args[i])  
            resultCh <- result{  
               value: ret,  
               err:   err,  
            }  
         }()  
         select {  
         case result := <-resultCh:  
            if result.err != nil {  
               return result.err  
            }  
            results[i] = result.value  
            break  
         case <-ctx.Done():  
            break  
         }  
         return nil  
      })  
   }  
   err = eg.Wait()  
   return  
}  
  
type result struct {  
   value interface{}  
   err   error  
}  
  
func mockTask(arg interface{}) interface{} {  
   // 模拟执行任务  
   time.Sleep(time.Second * 3)  
   return fmt.Sprintf("result-%v", arg)  
}
```

# 参考文献
https://lailin.xyz/post/go-training-week3-errgroup.html