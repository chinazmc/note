
##  背景

关于`Gin`中间件`Context.Next()`的用途我之前一直认为就是用在当前中间件的最后，用来把控制权还给`Context`让其它的中间件能执行，反过来说就是如果没有这一句其它的中间件就不能执行，翻阅[这篇文章](https://segmentfault.com/q/1010000020256918)发现不止我有同样的想法，直到今天遇到跨域问题需要为此写一个中间件，于是找到了[cors](https://github.com/gin-contrib/cors)这个项目，在翻阅源码的过程中意外发现代码中竟然没有`c.Next()`环节，顿时有了疑虑，其它中间件和`handler`怎么执行？经过仔细确认发现确实没有后，感觉事情并不简单可能超出认知了于是就有了这篇文章。

还是一样先实践验证后分析原理，既然[cors](https://github.com/gin-contrib/cors)可以不用`c.Next()`,那我们就来写个小demo验证下如果没有调用`Next()`后续中间件到底能不能执行？
```go

package main

import (
	"github.com/gin-gonic/gin"
	"log"
)

func midOne() gin.HandlerFunc {
	return func(c *gin.Context) {
		log.Println("midOne")
	}
}


func midTwo() gin.HandlerFunc {
	return func(c *gin.Context) {
		log.Println("midTwo")
	}
}

func main() {
	r := gin.New()
	r.Use(midOne())
	r.Use(midTwo())

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200,"ok")
	})
	
	r.Run(":8080")
}
```

[![image-20201016111403262](https://blog.dianduidian.com/images/image-20201016111403262.png)](https://blog.dianduidian.com/images/image-20201016111403262.png)

通过上边demo可以看到中间件中最后虽然没有调用`Next()`但其它中间件及`handlers`也都执行了。接下来我们来结合源码分析下原理。

##  原理

`Gin`中最终处理请求的逻辑是在`engine.handleHTTPRequest()` 这个函数
```go
func (engine *Engine) handleHTTPRequest(c *Context) {
	// ...

	// Find root of the tree for the given HTTP method
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
		value := root.getValue(rPath, c.params, unescape)
		if value.params != nil {
			c.Params = *value.params
		}
		if value.handlers != nil {
			c.handlers = value.handlers
			c.fullPath = value.fullPath
			c.Next() //执行handlers
			c.writermem.WriteHeaderNow()
			return
		}
		// ...
		}
		break
	}

// ... 
}
```

其中`c.Next()` 是关键
```go
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in GitHub.
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c) //执行handler
		c.index++
	}
}
```
从`Next()`方法我们可以看到它会遍历执行全部`handlers`（中间件也是`handler`），所以中间件中调不调用`Next()`方法并不会影响后续中间件的执行。

既然中间件中没有`Next()`不影响后续中间件的执行，那么在当前中间件中调用`c.Next()`的作用又是什么呢？

通过`Next()`函数的逻辑也能很清晰的得出结论：**在当前中间件中调用`c.Next()`时会中断当前中间件中后续的逻辑，转而执行后续的中间件和handlers，等他们全部执行完以后再回来执行当前中间件的后续代码。**

##  结论

1. **中间件代码最后即使没有调用`Next()`方法，后续中间件及`handlers`也会执行；**
2. **如果在中间件函数的非结尾调用`Next()`方法当前中间件剩余代码会被暂停执行，会先去执行后续中间件及`handlers`，等这些`handlers`全部执行完以后程序控制权会回到当前中间件继续执行剩余代码；**
3. **如果想提前中止当前中间件的执行应该使用`return`退出而不是`Next()`方法；**
4. **如果想中断剩余中间件及handlers应该使用`Abort`方法，但需要注意当前中间件的剩余代码会继续执行。**
# Reference
https://blog.dianduidian.com/post/gin-%E4%B8%AD%E9%97%B4%E4%BB%B6next%E6%96%B9%E6%B3%95%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/