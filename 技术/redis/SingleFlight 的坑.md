
# SingleFlight 的坑
#### 问题 

SingleFlight 有坑吗？先看下面的例子，模拟了 singleflight 中定义的方法阻塞的场景（实际中肯定会遇到这种情况：如果函数执行遇到问题被阻塞了），由于 singleflight `Do` 方法是以阻塞的方式读取下游请求的并发量，在第一个下游请求没返回之前，所有的请求都会被阻塞。假设并发量较大的情况下，如果某一个下游并发请求出现了阻塞，那么可能会出现下述问题：

1. goroutines 暴增
2. 内存暴涨，主要是 `1` 引发的
3. 后续同类请求都会阻塞在 singleflight 的阻塞等待完成上，所以后续请求耗时必然会增加

```GOLANG
type Result string

func find(ctx context.Context, query string) (Result, error) {
	time.Sleep(30*time.Second)
	return "", errors.New("bad result")
}

func main() {
	var g singleflight.Group
	const n = 10
	waited := int32(n)
	done := make(chan struct{})
	key := "some_related_key"
	for i := 0; i < n; i++ {
		go func(j int) {
			v, err , shared := g.Do(key, func() (interface{}, error) {
				ret, err := find(context.Background(), key)
				return ret, err
			})
			if atomic.AddInt32(&waited, -1) == 0 {
				close(done)
			}
			fmt.Printf("index: %d, val: %v, shared: %v error:%v\n", j, v, shared,err)
		}(i)
	}

	select {
	case <-done:
	case <-time.After(120*time.Second):
		fmt.Println("Do hangs")
	}
}
```

#### 解决方案（1）：有效超时控制，避免一个阻塞，全员等待[](https://pandaychen.github.io/2023/03/25/A-GOLANG-SINGLEFLIGHT-PRACTICE/#%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%881%E6%9C%89%E6%95%88%E8%B6%85%E6%97%B6%E6%8E%A7%E5%88%B6%E9%81%BF%E5%85%8D%E4%B8%80%E4%B8%AA%E9%98%BB%E5%A1%9E%E5%85%A8%E5%91%98%E7%AD%89%E5%BE%85)

上述问题的根本原因是阻塞读导致缺少超时控制，难以快速失败；解决方式，就是在对阻塞过久的 singleflight 请求进行超时控制，刚好它提供了 `DoChan` 方法，`DoChan` 通过 channel 返回结果，可使用 `select` 实现超时控制，如下：

```GOLANG
func find(ctx context.Context, query string) (Result, error) {
        time.Sleep(50 * time.Second)
        return "", errors.New("bad result")
}

func main() {
        var (
                g       singleflight.Group
                timeout = time.After(5000 * time.Millisecond)
        )
        const n = 10
        key := "some_related_key"
        for i := 0; i < n; i++ {
                go func(j int) {
                        ch := g.DoChan(key, func() (interface{}, error) {
                                ret, err := find(context.Background(), key)
                                return ret, err
                        })
                        var ret singleflight.Result
                        select {
                        case <-timeout: // Timeout elapsed
                                fmt.Println("Timeout")
                                return
                        case ret = <-ch: // Received result from channel
                                fmt.Printf("index: %d, val: %v, shared: %v\n", j, ret.Val, ret.Shared)
                        }
                }(i)
        }
        fmt.Println("done")
        select {
        case <-time.After(100 * time.Second):
                fmt.Println("Do hangs")
        }
}
```

注意，当超时时，输出如下：

```
done
Timeout
```

一种更为优雅的封装是走 `context` 方式，如下。调用的时候传入一个含超时的 `context` 即可，执行时就会返回超时错误：

```
func singleflightChan(ctx context.Context, sg *singleflight.Group, key string) (string, error) {
        result := sg.DoChan(key, func() (interface{}, error) {
                // 模拟出现问题
                time.Sleep(30 * time.Second)
                return "", errors.New("bad result")
        })

        select {
        case r := <-result:
                return r.Val.(string), r.Err
        case <-ctx.Done():
                return "", ctx.Err()
        }
}
```

#### 解决方案（2）：频率控制，解决一个出错，全部出错[](https://pandaychen.github.io/2023/03/25/A-GOLANG-SINGLEFLIGHT-PRACTICE/#%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%882%E9%A2%91%E7%8E%87%E6%8E%A7%E5%88%B6%E8%A7%A3%E5%86%B3%E4%B8%80%E4%B8%AA%E5%87%BA%E9%94%99%E5%85%A8%E9%83%A8%E5%87%BA%E9%94%99)

降低请求数量 在一些对可用性要求极高的场景下，往往需要一定的请求饱和度来保证业务的最终成功率。一次请求还是多次请求，对于下游服务而言并没有太大区别，此时使用 singleflight 只是为了降低请求的数量级，那么使用 `Forget()` 提高下游请求的并发，如下代码：

```go
func find(ctx context.Context, query string) (Result, error) {
        time.Sleep(1 * time.Second)
        return "succ", nil
}

func main() {
        var g singleflight.Group
        const n = 10
        key := "some_related_key"
        for i := 0; i < n; i++ {
                go func(j int) {
                        v, err, shared := g.Do(key, func() (interface{}, error) {
                                go func() {
                                        time.Sleep(10 * time.Millisecond)
                                        fmt.Printf("Deleting key:%d, %v\n",j, key)
                                        g.Forget(key)
                                }()
                                ret, err := find(context.Background(), key)
                                return ret, err
                        })
                        fmt.Printf("index: %d, val: %v, shared: %v error:%v\n", j, v, shared,err)
                }(i)
        }

        select {
        case <-time.After(100 * time.Second):
                fmt.Println("Do hangs")
        }
}
```

当有一个并发请求超过 `10ms`，那么将会有第二个请求发起，此时只有 `10ms` 内的请求最多发起一次请求，即最大并发：`100 QPS`，单次请求失败的影响大大降低

## 0x03 总结 

本文介绍了 singleflight 机制的应用及避坑场景，另外注意几点

1. **项目中 singleflight 的变量尽量设置成全局**，如果定义为局部变量（如 `http.HandleFunc` 方法），导致没有发挥 singleflight 的作用
2. `DoChan` 方法配置 `context.WithTimeout` 使用效果更好（参考上面 `net.Resolver` 实现）
# Reference
https://pandaychen.github.io/2023/03/25/A-GOLANG-SINGLEFLIGHT-PRACTICE/