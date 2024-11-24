# [Gin中间件中获取request.body](http://razil.cc/post/2019/04/go_gin_middleware_request_body/ "[go] Gin中间件中获取request.body")

线上有个中间件开发需求，需要对请求内容进行部分过滤。由于默认http.Request.Body类型为`io.ReadCloser`类型,即只能读一次，读完后直接close掉,后续流程无法继续读取。这对于在http handler之前需要对body进行处理就带来了麻烦。实际上针对这种需求，其他框架有自己的解决方案。 在beego中开启 `beego.BConfig.CopyRequestBody = true`后,Controller对应gin的Context中即包含了request.body的拷贝值，可以直接在中间件和Handler中传递。  
但是Gin实际上并未提供该功能，issue中的GetRawBody本质也未能解决这个问题。  
最后在stackoverflow找到问题解决方案,在这里做个记录,[how-to-get-the-json-from-the-body-of-a-request-on-go](https://stackoverflow.com/questions/47186741/how-to-get-the-json-from-the-body-of-a-request-on-go/47295689#47295689)  
> ps:该解决方案同样适用于http原生路由http.ServeMux处理方法。

## 解决方案

解决思路: 由于 Request.Body 为公共变量,我们在对原有的buffer读取完成后,只要手动创建一个新的buffer然后以同样接口形式替换掉原有的Request.Body即可。

```go
var bodyBytes []byte // 我们需要的body内容

// 从原有Request.Body读取
bodyBytes, err := ioutil.ReadAll(c.Request.Body)
if err != nil {
	return 0, nil, fmt.Errorf("Invalid request body")
}

// 新建缓冲区并替换原有Request.body
c.Request.Body = ioutil.NopCloser(bytes.NewBuffer(bodyBytes))

// 当前函数可以使用body内容
_ := bodyBytes
```

# Gin 中间件Next()方法作用解析

## [](https://blog.dianduidian.com/post/gin-%E4%B8%AD%E9%97%B4%E4%BB%B6next%E6%96%B9%E6%B3%95%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/#%E8%83%8C%E6%99%AF)背景

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

## [](https://blog.dianduidian.com/post/gin-%E4%B8%AD%E9%97%B4%E4%BB%B6next%E6%96%B9%E6%B3%95%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/#%E5%8E%9F%E7%90%86)原理

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

## [](https://blog.dianduidian.com/post/gin-%E4%B8%AD%E9%97%B4%E4%BB%B6next%E6%96%B9%E6%B3%95%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/#%E7%BB%93%E8%AE%BA)结论

1. **中间件代码最后即使没有调用`Next()`方法，后续中间件及`handlers`也会执行；**
2. **如果在中间件函数的非结尾调用`Next()`方法当前中间件剩余代码会被暂停执行，会先去执行后续中间件及`handlers`，等这些`handlers`全部执行完以后程序控制权会回到当前中间件继续执行剩余代码；**
3. **如果想提前中止当前中间件的执行应该使用`return`退出而不是`Next()`方法；**
4. **如果想中断剩余中间件及handlers应该使用`Abort`方法，但需要注意当前中间件的剩余代码会继续执行。**
我们在编写gin的中间件时，如果需要后置处理，是需要执行context.Next()的,很显然，这是一个递归调用，只是通过串联context,使中间件可以主动把握递归调用下一层的时机，甚至中止处理链的继续执行，如果没有调用next()，则在本次handler执行结束后直接执行下一个。中间件的一般流程是，先调用前置动作，使函数往下执行，往上返回，然后执行后置动作。  
  
下面是GinContext的两个执行模型。分别对应中间件中使用Next()和不使用Next()。  
  
![[Pasted image 20231108144134.png]]
通过对gin源码的分析，可以看出gin.Context的设计还是比较巧妙的，context在递归调用过程中很好的发挥了上下文的作用，穿针引线，携带着全局的请求信息，暴露向下层调用的api。

  