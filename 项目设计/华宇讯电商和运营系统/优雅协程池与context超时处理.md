```go
func main() {  
   args := make([]interface{}, 20)  
   for i := range args {  
      args[i] = fmt.Sprintf("arg:%d", i)  
   }  
   results, err := run(args, 5, func(i int, arg interface{}) (result interface{}, err error) {  
      if i == 6 {  
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
   ctx,cancel := context.WithTimeout(context.Background(),16*time.Second)  
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


# 优雅的高并发 生产消费者模型
```go
package handle_million_requests  
  
import (  
   "fmt"  
   "sync"  
   "github.com/sirupsen/logrus")  
  
type W1 struct {  
   WgSend       *sync.WaitGroup  
   Wg           *sync.WaitGroup  
   MaxNum       int  
   Ch           chan string  
   DispatchStop chan struct{}  
}  
  
func (w *W1) Dispatch(job string) {  
   w.WgSend.Add(10 * w.MaxNum)  
   for i := 0; i < 10*w.MaxNum; i++ {  
      go func(i int) {  
         defer w.WgSend.Done()  
  
         select {  
         case w.Ch <- fmt.Sprintf("%d", i):  
            return  
         case <-w.DispatchStop:  
            logrus.Debugln("退出发送 job: ", fmt.Sprintf("%d", i))  
            return  
         }  
      }(i)  
   }  
}  
  
func (w *W1) StartPool() {  
   if w.Ch == nil {  
      w.Ch = make(chan string, w.MaxNum)  
   }  
  
   if w.WgSend == nil {  
      w.WgSend = &sync.WaitGroup{}  
   }  
  
   if w.Wg == nil {  
      w.Wg = &sync.WaitGroup{}  
   }  
  
   if w.DispatchStop == nil {  
      w.DispatchStop = make(chan struct{})  
   }  
  
   w.Wg.Add(w.MaxNum)  
   for i := 0; i < w.MaxNum; i++ {  
      go func() {  
         defer w.Wg.Done()  
         for v := range w.Ch {  
            logrus.Debugf("完成工作: %s \n", v)  
         }  
      }()  
   }  
}  
  
func (w *W1) Stop() {  
   close(w.DispatchStop)  
   w.WgSend.Wait()  
  
   close(w.Ch)  
   w.Wg.Wait()  
}  
  
func DealW1(max int) {  
   w := NewWorker(w1, max)  
   w.StartPool()  
   w.Dispatch("")  
  
   w.Stop()  
}
```

# Reference
https://marksuper.xyz/2021/10/08/handle_million_req/