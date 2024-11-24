#golang 

### 坑

interface 最著名的坑的，应该就是下面这个了。

```go
func main() {
    var a interface{} = nil
    var b *int = nil
    
    isNil(a)
    isNil(b)
}

func isNil(x interface{}) {
    if x == nil {
      fmt.Println("empty interface")
      return
    }
    fmt.Println("non-empty interface")
}
复制代码
```

output:

```
empty interface
non-empty interface
复制代码
```

为什么会这样呢？这就涉及到 interface == nil 的判断方式了。一般情况只有 eface 的 type 和 data 都为 nil 时，interface == nil 才是 true。

当我们把 b 复制给 interface 时，x.\_type.Kind = kindPtr。虽说 x.data = nil，但是不符合 interface == nil 的判断条件了。
	