#gin 

## `一、Gin框架说明

> `Gin`是一个用`Golang`编写的`Web`框架， 是一个基于`Radix`树的路由，小内存占用，没有反射，可预测的`API`框架，速度挺高了近40倍，支持中间件、路由组处理、JSON等多方式验证、内置了`JSON`、`XML`、`HTML`等渲染。是一个易于使用的`API`框架；本系列文章讲来具体揭秘`Gin`的源码。 下面请看一个实例：

```go
func main() {
	// 新建一个没有任何默认中间件的路由
	r := gin.New()

	// 全局中间件
	// Logger 中间件将日志写入 gin.DefaultWriter，即使你将 GIN_MODE 设置为 release。
	// By default gin.DefaultWriter = os.Stdout
	r.Use(gin.Logger())

	// Recovery 中间件会 recover 任何 panic。如果有 panic 的话，会写入 500。
	r.Use(gin.Recovery())

	// 你可以为每个路由添加任意数量的中间件。
	r.GET("/benchmark", MyBenchLogger(), benchEndpoint)

	// 认证路由组
	// authorized := r.Group("/", AuthRequired())
	// 和使用以下两行代码的效果完全一样:
	authorized := r.Group("/")
	// 路由组中间件! 在此例中，我们在 "authorized" 路由组中使用自定义创建的 
        // AuthRequired() 中间件
	authorized.Use(AuthRequired())
	{
		authorized.POST("/login", loginEndpoint)
		authorized.POST("/submit", submitEndpoint)
		authorized.POST("/read", readEndpoint)

		// 嵌套路由组
		testing := authorized.Group("testing")
		testing.GET("/analytics", analyticsEndpoint)
	}

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

## 二、Engine相关结构体

> 下图描述了整个`Engine`相关的结构体组合；`gin.Engine`是一个容器对象，为整个框架的基础；`gin.RouterGroup`负责存储所有的中间件，包括请求路径；`gin.trees`负责存储路由和`handle`方法的映射，采用的是`Radix`树的结构； 下面重点对`Engine`参数做一个简单说明；其他部分在后续系列文章中在做解析。

![](https://pic2.zhimg.com/80/v2-10c2859cae686284f524c602e26c2b5d_720w.jpg)

`RouterGroup`

> 负责存储所有的中间件，包括请求路径。

`RedirectTrailingSlash`

> 对于后缀为/的路由是否开启重定向请求，如请求/ foo / 但仅存在/ foo的路由，则客户端使用http状态吗307重定向到/ foo进行请求

`RedirectFixedPath`

> 如果开启该参数，没有handler注册时，路由会尝试自己去修复当前的请求地址。  
> 1、首位多余元素会被删除(../ or //);  
> 2、然后路由会对新的路径进行不区分大小写的查找;  
> 3、如果能正常找到对应的handler，路由就会重定向到正确的handler上并返回301或者307.(比如: 用户访问/FOO 和 /..//Foo可能会被重定向到/foo这个路由上)

`HandleMethodNotAllowed`

> 检查当前路径是否允许使用其他请求方法来路由。

`ForwardedByClientIP`

> 是否转发客户端ip。

`RemoteIPHeaders`

> 在以下情况下获取客户端IP。 ForwardedByClientIP为true；context.Request.RemoteAddr 有值

`TrustedProxies`

> 网络来源列表（IPv4地址，IPv4 CIDR，IPv6地址或信任包含以下内容的请求标头的IPv6 CIDR：当`（* gin.Engine）.ForwardedByClientIP`为`true`。

`AppEngine`

> 如果开启将会在请求中增加一个以"X-AppEngine..."开头的header。

`UseRawPath`

> 如果启用，则将使用url.RawPath查找参数

`UnescapePathValues`

> 如果为true，则将不转义路径值。如果UseRawPath为false（默认情况下），则UnescapePathValues实际上为true，作为url.Path将被使用，它已经被转义了。

`MaxMultipartMemory`

> 赋予http.Request的ParseMultipartForm的'maxMemory'参数的值。

`RemoveExtraSlash`

> 是否删除额外的反斜线(开始时可解析有额外斜线的请求)

`delims`

> Delims代表用于HTML模板呈现的一组左右定界符。

`secureJSONPrefix`

> 设置在Context.SecureJSON中国的json前缀

`HTMLRender`

> 返回HTMLRender接口(用于渲染HTMLProduction和HTMLDebug两个结构体类型的模板)

`FuncMap`

> html/template包中的FuncMap map[string]interface{} ,用来定义从名称到函数的映射。

`allNoRoute`

> 复制一份全局的`handlers`，加上`NoRoute`处理方法

`allNoMethod`

> 复制一份全局的`handlers`，加上`NoMethod`处理方法

`noRoute`

> noRoute为NoRoute添加处理程序。它默认返回404代码

`noMethod`

> noMethod为NoMethod添加处理程序，它默认返回405代码

`pool`

> 主要用来存储`context`上下文对象，用来优化处理`http`请求时的性能

`trees`

> 负责存储路由和handle方法的映射，采用的是Radix树的结构。

`maxParams`

> 分配`context`时，初始化参数`Params`的切片长度

`trustedCIDRs`

> IP网络地址指针切片

## 三、请求处理流程

### 3.1、初始化Gin实例

> `Gin`框架暴露了两个方法用来初始化`Gin`实例，`New`和`Default`方法，两者的区别是`Default`方法在调用`New`的基础上增加了使用`Logger`和 `Recovery`自带中间件的处理。如果需要自己手写这两个中间件可以在调用`New`的基础上，增加`Use`方法，用来注册中间件。 初始化`Gin`实例中主要实现了两个核心功能：初始化路由组`RouterGroup`和初始化`pool`主要用来存储`context`上下文对象，用来优化处理`http`请求时的性能。

```go
// 返回Engine实例，具体字段见上面结构体说明
func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{ //初始化RouterGroup
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
		ForwardedByClientIP:    true,
		RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
		TrustedProxies:         []string{"0.0.0.0/0"},
		AppEngine:              defaultAppEngine,
		UseRawPath:             false,
		RemoveExtraSlash:       false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
		trees:                  make(methodTrees, 0, 9), //初始化路由树
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJSONPrefix:       "while(1);",
	}
	engine.RouterGroup.engine = engine
	engine.pool.New = func() interface{} {  //初始化Pool
		return engine.allocateContext()
	}
	return engine
}
// 默认值返回已注册Logger和Recovery中间件的Engine实例
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

### 3.2、注册中间件

> 在`Gin`框架中主要包括两种中间件：全局中间件和路由组中间件。

-   `全局中间件`：通过调用`Gin.Use()`来实现往`RouterGroup.Handlers`写入记录

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```

-   `路由组中间件` ：通过`RouterGroup.Group`返回一个新生成的`RouterGroup`指针，用来分开每个路由组加载不一样的中间件。 注意`group.combineHandlers(handlers)`会复制一份全局中间件到新生成的`RouterGroup`中。

```go
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
}
```

### 3.3、注册路由

> `Gin`框架中指定了注册路由的若干方法：`GET`、`POST`、`DELETE`、`PUT`、`HEAD`、`OPTIONS`、`ANY` 、`PATCH`。不管哪种请求方式都会调用`RouterGroup.handler`来添加路由，在该方法中同样会通过`group.combineHandlers(handlers)`会复制一份全局中间件并且加上这次相应路由的`handlers`，组成一个`List`放入树节点中，最后调用`tree.addRoute`来增加节点。在涉及路由树部分会有文章详细分析。

```go
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPost, relativePath, handlers)
}
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}
func (group *RouterGroup) DELETE(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodDelete, relativePath, handlers)
}
func (group *RouterGroup) PATCH(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPatch, relativePath, handlers)
}
func (group *RouterGroup) PUT(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPut, relativePath, handlers)
}
func (group *RouterGroup) OPTIONS(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodOptions, relativePath, handlers)
}
func (group *RouterGroup) HEAD(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodHead, relativePath, handlers)
}
func (group *RouterGroup) Any(relativePath string, handlers ...HandlerFunc) IRoutes {
	group.handle(http.MethodGet, relativePath, handlers)
	group.handle(http.MethodPost, relativePath, handlers)
	group.handle(http.MethodPut, relativePath, handlers)
	group.handle(http.MethodPatch, relativePath, handlers)
	group.handle(http.MethodHead, relativePath, handlers)
	group.handle(http.MethodOptions, relativePath, handlers)
	group.handle(http.MethodDelete, relativePath, handlers)
	group.handle(http.MethodConnect, relativePath, handlers)
	group.handle(http.MethodTrace, relativePath, handlers)
	return group.returnObj()
}
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)

	root := engine.trees.get(method)
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	root.addRoute(path, handlers)

	if paramsCount := countParams(path); paramsCount > engine.maxParams {
		engine.maxParams = paramsCount
	}
}
```

### 3.4、上下文处理

> `Gin`在初始化的过程中，会初始化`Context`池即为`engine.pool`，用来优化处理http请求时的性能。在后续处理请求的过程中取去用于存储`Context`的内存池。该`Context`主要实现了对`Request`和`Response`的封装以及一些参数的传递。其数据结构如下：

```go
type Context struct {
	writermem responseWriter //响应处理
	Request   *http.Request //HTTP请求
	Writer    ResponseWriter //HTTP响应
	Params   Params //url参数，由key和value组成
	handlers HandlersChain //定义一个HandlerFunc数组
	index    int8  //定义一个HandlerFunc数组的索引
	fullPath string //请求地址
	engine *Engine  //gin框架对象
	params *Params  //URL参数的切片
	mu sync.RWMutex //读写锁，用来保护Keys
	Keys map[string]interface{} //键是专门针对每个请求的上下文的键/值对
        Errors errorMsgs //使用此上下文的所有处理程序或者中间件附带的错误列表
	Accepted []string //自定义请求接收的内容类型格式
	queryCache url.Values //使用url.ParseQuery从c.Request.URL.Query（）缓存了参数查询结果
	formCache url.Values // 使用url.ParseQuery缓存的PostForm包含来自POST，PATCH或者PUT请求参数
	sameSite http.SameSite //允许服务器定义cookie属性。用来防止 CSRF 攻击和用户追踪
}
```

### 3.5、启动

> 通过调用`net/http`来启动服务，由于`engine`实现了`ServeHttp`方法，只需要直接在`http.ListenAndServe`方法第二个参数传入`engine`即可，这样就可以初始化并完成启动。 在`Gin`框架中，一共实现了四种类型的启动，如下：

-   `HTTP`

```go
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	trustedCIDRs, err := engine.prepareTrustedCIDRs()
	if err != nil {
		return err
	}
	engine.trustedCIDRs = trustedCIDRs
	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine)
	return
}
```

-   `HTTPS`

```go
func (engine *Engine) RunTLS(addr, certFile, keyFile string) (err error) {
	debugPrint("Listening and serving HTTPS on %s\n", addr)
	defer func() { debugPrintError(err) }()
	err = http.ListenAndServeTLS(addr, certFile, keyFile, engine)
	return
}
```

-   `Unix Socket`

```go
func (engine *Engine) RunUnix(file string) (err error) {
	debugPrint("Listening and serving HTTP on unix:/%s", file)
	defer func() { debugPrintError(err) }()

	listener, err := net.Listen("unix", file)
	if err != nil {
		return
	}
	defer listener.Close()
	defer os.Remove(file)

	err = http.Serve(listener, engine)
	return
}
```

-   `File`

```go
func (engine *Engine) RunFd(fd int) (err error) {
	debugPrint("Listening and serving HTTP on fd@%d", fd)
	defer func() { debugPrintError(err) }()

	f := os.NewFile(uintptr(fd), fmt.Sprintf("fd@%d", fd))
	listener, err := net.FileListener(f)
	if err != nil {
		return
	}
	defer listener.Close()
	err = engine.RunListener(listener)
	return
}
```

### 3.6、处理请求

> 在启动处理过程中，传入了实现`ServeHttp`方法的`engine`来处理`HTTP`请求。

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}

func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method //http请求方法，get、post
	rPath := c.Request.URL.Path //请求路径
	unescape := false
	if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
		rPath = c.Request.URL.RawPath
		unescape = engine.UnescapePathValues
	}

	if engine.RemoveExtraSlash {
		rPath = cleanPath(rPath)
	}

	// 遍历路由树中对应的httpMethod的节点
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod { 
			continue
		}
		root := t[i].root
		// 找到rPath对应的路由节点，后续详细分析
		value := root.getValue(rPath, c.params, unescape)
		if value.params != nil {
			c.Params = *value.params //Params赋值
		}
		if value.handlers != nil {
			c.handlers = value.handlers //handlers赋值
			c.fullPath = value.fullPath //fullPath赋值
			c.Next() //执行路由对应的处理方法
			c.writermem.WriteHeaderNow() //响应头
			return
		}
                //出现不规则的自定义路由时并且有类似的路由存在。重定向处理
		if httpMethod != "CONNECT" && rPath != "/" {
			if value.tsr && engine.RedirectTrailingSlash {
				redirectTrailingSlash(c) 
				return
			}
			if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
				return
			}
		}
		break
	}
        // 检查是否允许使用其他方法（get、post等）
	if engine.HandleMethodNotAllowed {
		for _, tree := range engine.trees {
			if tree.method == httpMethod {
				continue
			}
                        // 检查到该请求方式没有请求被路由时，返回不允许的方法（http code = 405）
			if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
				c.handlers = engine.allNoMethod
				serveError(c, http.StatusMethodNotAllowed, default405Body)
				return
			}
		}
	}
        // 没有路由返回http code = 404
	c.handlers = engine.allNoRoute
	serveError(c, http.StatusNotFound, default404Body)
}
```

