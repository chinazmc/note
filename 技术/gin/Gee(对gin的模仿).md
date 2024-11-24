#gin #golang 

# 第一天

```go
import (  
   "fmt"  
   "net/http")  
//HandlerFunc defines the request handler userd by geetype HandlerFunc func(http.ResponseWriter,*http.Request)  
  
//Engine implement the interface of ServeHTTP  
type Engine struct {  
   router map[string]HandlerFunc  
}  
  
//New is the constructor of gee.Engine  
func New() *Engine {  
   return &Engine{  
      router: make(map[string]HandlerFunc),  
   }  
}  
func (engine *Engine) addRoute(method string,pattern string,handler HandlerFunc){  
   key:=method+"-"+pattern  
   engine.router[key]=handler  
}  
  
func (engine *Engine) Get(pattern string,handler HandlerFunc)  {  
   engine.addRoute("GET",pattern,handler)  
}  
func (engine *Engine) POST(pattern string,handler HandlerFunc)  {  
   engine.addRoute("POST",pattern,handler)  
}  
func (engine *Engine) Run(addr string) (err error){  
   return http.ListenAndServe(addr,engine)  
}  
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request)()  {  
   key:=req.Method+"-"+req.URL.Path  
   if handler,ok:=engine.router[key];ok {  
      handler(w,req)  
   }else {  
      w.WriteHeader(http.StatusNotFound)  
      fmt.Fprintf(w,"404 NOT FOUND:%s\n",req.URL)  
   }  
}
```
调用案例：
```go
func main() {  
   r := gee.New()  
   r.Get("/", func(w http.ResponseWriter, req *http.Request) {  
      fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)  
   })  
  
   r.Get("/hello", func(w http.ResponseWriter, req *http.Request) {  
      for k, v := range req.Header {  
         fmt.Fprintf(w, "Header[%q] = %q\n", k, v)  
      }  
   })  
  
   r.Run(":9999")  
}
```
主要是对http的使用

# 第二天
## 设计Context

### 必要性
1、对Web服务来说，无非是根据请求`*http.Request`，构造响应`http.ResponseWriter`。但是这两个对象提供的接口粒度太细，比如我们要构造一个完整的响应，需要考虑消息头(Header)和消息体(Body)，而 Header 包含了状态码(StatusCode)，消息类型(ContentType)等几乎每次请求都需要设置的信息。因此，如果不进行有效的封装，那么框架的用户将需要写大量重复，繁杂的代码，而且容易出错。针对常用场景，能够高效地构造出 HTTP 响应是一个好的框架必须考虑的点。

用返回 JSON 数据作比较，感受下封装前后的差距。

封装前
```go
obj = map[string]interface{}{  
    "name": "geektutu",  
    "password": "1234",  
}  
w.Header().Set("Content-Type", "application/json")  
w.WriteHeader(http.StatusOK)  
encoder := json.NewEncoder(w)  
if err := encoder.Encode(obj); err != nil {  
    http.Error(w, err.Error(), 500)  
}  

```

VS 封装后：
```go
c.JSON(http.StatusOK, gee.H{  
    "username": c.PostForm("username"),  
    "password": c.PostForm("password"),  
})
```
2.、针对使用场景，封装`*http.Request`和`http.ResponseWriter`的方法，简化相关接口的调用，只是设计 Context 的原因之一。对于框架来说，还需要支撑额外的功能。例如，将来解析动态路由`/hello/:name`，参数`:name`的值放在哪呢？再比如，框架需要支持中间件，那中间件产生的信息放在哪呢？Context 随着每一个请求的出现而产生，请求的结束而销毁，和当前请求强相关的信息都应由 Context 承载。因此，设计 Context 结构，扩展性和复杂性留在了内部，而对外简化了接口。路由的处理函数，以及将要实现的中间件，参数都统一使用 Context 实例， Context 就像一次会话的百宝箱，可以找到任何东西。
gee.go
```go
//HandlerFunc defines the request handler userd by geetype HandlerFunc func(*Context)  
  
//Engine implement the interface of ServeHTTP  
type Engine struct {  
   router *router  
}  
  
//New is the constructor of gee.Engine  
func New() *Engine {  
   return &Engine{  
      router: newRouter(),  
   }  
}  
func (engine *Engine) addRoute(method string,pattern string,handler HandlerFunc){  
   engine.router.addRoute(method, pattern, handler)  
}  
  
func (engine *Engine) Get(pattern string,handler HandlerFunc)  {  
   engine.addRoute("GET",pattern,handler)  
}  
func (engine *Engine) POST(pattern string,handler HandlerFunc)  {  
   engine.addRoute("POST",pattern,handler)  
}  
func (engine *Engine) Run(addr string) (err error){  
   return http.ListenAndServe(addr,engine)  
}  
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request)()  {  
   c := newContext(w, req)  
   engine.router.handle(c)  
}
```
router.go
```go
type router struct {  
   handlers map[string]HandlerFunc  
}  
  
func newRouter() *router {  
   return &router{  
      handlers: make(map[string]HandlerFunc),  
   }  
}  
func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {  
   log.Printf("Route %4s - %s", method, pattern)  
   key := method + "-" + pattern  
   r.handlers[key] = handler  
}  
  
func (r *router) handle(c *Context) {  
   key := c.Method + "-" + c.Path  
   if handler, ok := r.handlers[key]; ok {  
      handler(c)  
   } else {  
      c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)  
   }  
}
```
context.go
```go
type H map[string]interface{}  
  
type Context struct {  
   //origin objects  
   Writer http.ResponseWriter  
   Req *http.Request  
   //request info  
   Path string  
   Method string  
   //response info  
   StatusCode int  
}  
  
func newContext(w http.ResponseWriter,req *http.Request) *Context{  
   return &Context{  
      Writer:w,  
      Req:req,  
      Path:req.URL.Path,  
      Method: req.Method,  
   }  
}  
func (context *Context) PostForm(key string)  string{  
   return context.Req.FormValue(key)  
}  
func (context *Context) Query(key string)  string{  
   return context.Req.URL.Query().Get(key)  
}  
func (context *Context) Status(code int)  {  
   context.StatusCode=code  
   context.Writer.WriteHeader(code)  
}  
func (context *Context) SetHeader(key string,value string)  {  
   context.Writer.Header().Set(key,value)  
}  
func (context *Context) String(code int,format string,values ...interface{}){  
   context.SetHeader("Content-Type","text/plain")  
   context.Status(code)  
   context.Writer.Write([]byte(fmt.Sprintf(format,values)))  
}  
func (context Context) JSON(code int,obj interface{})  {  
   //在 WriteHeader() 后调用 Header().Set 是不会生效的  
   context.SetHeader("Content-Type", "application/json")  
   context.Status(code)  
   encoder:=json.NewEncoder(context.Writer)  
   if err:=encoder.Encode(obj);err!=nil {  
      //如果err!=nil的话http.Error(c.Writer, err.Error(), 500)这里是不起作用的，  
      //因为前面已经执行了WriteHeader(code),那么返回码将不会再更改  
      //http.Error(c.Writer, err.Error(), 500)里面的w.WriteHeader(code)、w.Header().Set()不起作用，  
      //而且encoder.Encode(obj)相当于调用了Write()，  
      //http.Error(c.Writer, err.Error(), 500)里面的WriteHeader、Header().Set()操作都是无效的。  
      //我看了gin的代码，如果encoder.Encode(obj)这里报错的话是直接panic，感觉这里如果err!=nil的话确实不好处理  
  
  
      //参考 gin的render/json.go#L56  
      //http.Error(context.Writer,err.Error(),500)      panic(err)  
   }  
}  
func (c *Context) Data(code int, data []byte) {  
   c.Status(code)  
   c.Writer.Write(data)  
}  
  
func (c *Context) HTML(code int, html string) {  
   c.SetHeader("Content-Type", "text/html")  
   c.Status(code)  
   c.Writer.Write([]byte(html))  
}
```
调用方式：
```go
func main() {  
   r := gee.New()  
   r.Get("/", func(c *gee.Context) {  
      c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")  
   })  
   r.Get("/hello", func(c *gee.Context) {  
      // expect /hello?name=geektutu  
      c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)  
   })  
  
   r.POST("/login", func(c *gee.Context) {  
      c.JSON(http.StatusOK, gee.H{  
         "username": c.PostForm("username"),  
         "password": c.PostForm("password"),  
      })  
   })  
  
   r.Run(":9999")  
}
```

# 第三天
之前，我们用了一个非常简单的`map`结构存储了路由表，使用`map`存储键值对，索引非常高效，但是有一个弊端，键值对的存储的方式，只能用来索引静态路由。那如果我们想支持类似于`/hello/:name`这样的动态路由怎么办呢？所谓动态路由，即一条路由规则可以匹配某一类型而非某一条固定的路由。例如`/hello/:name`，可以匹配`/hello/geektutu`、`hello/jack`等。

动态路由有很多种实现方式，支持的规则、性能等有很大的差异。例如开源的路由实现`gorouter`支持在路由规则中嵌入正则表达式，例如`/p/[0-9A-Za-z]+`，即路径中的参数仅匹配数字和字母；另一个开源实现`httprouter`就不支持正则表达式。著名的Web开源框架`gin` 在早期的版本，并没有实现自己的路由，而是直接使用了`httprouter`，后来不知道什么原因，放弃了`httprouter`，自己实现了一个版本。
实现动态路由最常用的数据结构，被称为前缀树(Trie树)。看到名字你大概也能知道前缀树长啥样了：每一个节点的所有的子节点都拥有相同的前缀。这种结构非常适用于路由匹配，比如我们定义了如下路由规则：

-   /:lang/doc
-   /:lang/tutorial
-   /:lang/intro
-   /about
-   /p/blog
-   /p/related
![[Pasted image 20220425152555.png]]
HTTP请求的路径恰好是由`/`分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们通过树结构查询，如果中间某一层的节点都不满足条件，那么就说明没有匹配到的路由，查询结束。

接下来我们实现的动态路由具备以下两个功能。

-   参数匹配`:`。例如 `/p/:lang/doc`，可以匹配 `/p/c/doc` 和 `/p/go/doc`。
-   通配`*`。例如 `/static/*filepath`，可以匹配`/static/fav.ico`，也可以匹配`/static/js/jQuery.js`，这种模式常用于静态服务器，能够递归地匹配子路径。

router.go
```go
type router struct {  
   roots    map[string]*node  
   handlers map[string]HandlerFunc  
}  
  
func newRouter() *router {  
   return &router{  
      roots:    make(map[string]*node),  
      handlers: make(map[string]HandlerFunc),  
   }  
}  
  
// Only one * is allowed  
func parsePattern(pattern string) []string {  
   vs := strings.Split(pattern, "/")  
  
   parts := make([]string, 0)  
   for _, item := range vs {  
      if item != "" {  
         parts = append(parts, item)  
         if item[0] == '*' {  
            break  
         }  
      }  
   }  
   return parts  
}  
  
func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {  
   parts := parsePattern(pattern)  
  
   key := method + "-" + pattern  
   _, ok := r.roots[method]  
   if !ok {  
      r.roots[method] = &node{}  
   }  
   r.roots[method].insert(pattern, parts, 0)  
   r.handlers[key] = handler  
}  
  
func (r *router) getRoute(method string, path string) (*node, map[string]string) {  
   searchParts := parsePattern(path)  
   params := make(map[string]string)  
   root, ok := r.roots[method]  
  
   if !ok {  
      return nil, nil  
   }  
  
   n := root.search(searchParts, 0)  
  
   if n != nil {  
      parts := parsePattern(n.pattern)  
      for index, part := range parts {  
         if part[0] == ':' {  
            params[part[1:]] = searchParts[index]  
         }  
         if part[0] == '*' && len(part) > 1 {  
            params[part[1:]] = strings.Join(searchParts[index:], "/")  
            break  
         }  
      }  
      return n, params  
   }  
  
   return nil, nil  
}  
  
func (r *router) getRoutes(method string) []*node {  
   root, ok := r.roots[method]  
   if !ok {  
      return nil  
   }  
   nodes := make([]*node, 0)  
   root.travel(&nodes)  
   return nodes  
}  
  
func (r *router) handle(c *Context) {  
   n, params := r.getRoute(c.Method, c.Path)  
   if n != nil {  
      c.Params = params  
      key := c.Method + "-" + n.pattern  
      r.handlers[key](c)  
   } else {  
      c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)  
   }  
}
```
trie.go
```go
type node struct {  
   pattern  string// 待匹配路由，例如 /p/:lang   part     string// 路由中的一部分，例如 :lang   children []*node// 子节点，例如 [doc, tutorial, intro]   isWild   bool// 是否精确匹配，part 含有 : 或 * 时为true  
}  
  
func (n *node) String() string {  
   return fmt.Sprintf("node{pattern=%s, part=%s, isWild=%t}", n.pattern, n.part, n.isWild)  
}  
  
func (n *node) insert(pattern string, parts []string, height int) {  
   if len(parts) == height {  
      n.pattern = pattern  
      return  
   }  
  
   part := parts[height]  
   child := n.matchChild(part)  
   if child == nil {  
      child = &node{part: part, isWild: part[0] == ':' || part[0] == '*'}  
      n.children = append(n.children, child)  
   }  
   child.insert(pattern, parts, height+1)  
}  
  
func (n *node) search(parts []string, height int) *node {  
   if len(parts) == height || strings.HasPrefix(n.part, "*") {  
      if n.pattern == "" {  
         return nil  
      }  
      return n  
   }  
  
   part := parts[height]  
   children := n.matchChildren(part)  
  
   for _, child := range children {  
      result := child.search(parts, height+1)  
      if result != nil {  
         return result  
      }  
   }  
  
   return nil  
}  
  
func (n *node) travel(list *([]*node)) {  
   if n.pattern != "" {  
      *list = append(*list, n)  
   }  
   for _, child := range n.children {  
      child.travel(list)  
   }  
}  
// 第一个匹配成功的节点，用于插入  
func (n *node) matchChild(part string) *node {  
   for _, child := range n.children {  
      if child.part == part || child.isWild {  
         return child  
      }  
   }  
   return nil  
}  
// 所有匹配成功的节点，用于查找  
func (n *node) matchChildren(part string) []*node {  
   nodes := make([]*node, 0)  
   for _, child := range n.children {  
      if child.part == part || child.isWild {  
         nodes = append(nodes, child)  
      }  
   }  
   return nodes  
}
```
调用案例
```go
func main() {  
   r := gee.New()  
   r.GET("/", func(c *gee.Context) {  
      c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")  
   })  
  
   r.GET("/hello", func(c *gee.Context) {  
      // expect /hello?name=geektutu  
      c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)  
   })  
  
   r.GET("/hello/:name", func(c *gee.Context) {  
      // expect /hello/geektutu  
      c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)  
   })  
  
   r.GET("/assets/*filepath", func(c *gee.Context) {  
      c.JSON(http.StatusOK, gee.H{"filepath": c.Param("filepath")})  
   })  
  
   r.Run(":9999")  
}
```

# 第四天
## 分组的意义

分组控制(Group Control)是 Web 框架应提供的基础功能之一。所谓分组，是指路由的分组。如果没有路由分组，我们需要针对每一个路由进行控制。但是真实的业务场景中，往往某一组路由需要相似的处理。例如：

-   以`/post`开头的路由匿名可访问。
-   以`/admin`开头的路由需要鉴权。
-   以`/api`开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

大部分情况下的路由分组，是以相同的前缀来区分的。因此，我们今天实现的分组控制也是以前缀来区分，并且支持分组的嵌套。例如`/post`是一个分组，`/post/a`和`/post/b`可以是该分组下的子分组。作用在`/post`分组上的中间件(middleware)，也都会作用在子分组，子分组还可以应用自己特有的中间件。

中间件可以给框架提供无限的扩展能力，应用在分组上，可以使得分组控制的收益更为明显，而不是共享相同的路由前缀这么简单。例如`/admin`的分组，可以应用鉴权中间件；`/`分组应用日志中间件，`/`是默认的最顶层的分组，也就意味着给所有的路由，即整个框架增加了记录日志的能力。

提供扩展能力支持中间件的内容，我们将在下一节当中介绍。

##分组嵌套

一个 Group 对象需要具备哪些属性呢？首先是前缀(prefix)，比如`/`，或者`/api`；要支持分组嵌套，那么需要知道当前分组的父亲(parent)是谁；当然了，按照我们一开始的分析，中间件是应用在分组上的，那还需要存储应用在该分组上的中间件(middlewares)。还记得，我们之前调用函数`(*Engine).addRoute()`来映射所有的路由规则和 Handler 。如果Group对象需要直接映射路由规则的话，比如我们想在使用框架时，这么调用：
```go
r := gee.New()  
v1 := r.Group("/v1")  
v1.GET("/", func(c *gee.Context) {  
	c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")  
})  
```

那么Group对象，还需要有访问`Router`的能力，为了方便，我们可以在Group中，保存一个指针，指向`Engine`，整个框架的所有资源都是由`Engine`统一协调的，那么就可以通过`Engine`间接地访问各种接口了。
```go
Engine struct {  
	*RouterGroup  
	router *router  
	groups []*RouterGroup // store all groups  
}
```
```go
// New is the constructor of gee.Engine  
func New() *Engine {  
	engine := &Engine{router: newRouter()}  
	engine.RouterGroup = &RouterGroup{engine: engine}  
	engine.groups = []*RouterGroup{engine.RouterGroup}  
	return engine  
}  
  
// Group is defined to create a new RouterGroup  
// remember all groups share the same Engine instance  
func (group *RouterGroup) Group(prefix string) *RouterGroup {  
	engine := group.engine  
	newGroup := &RouterGroup{  
		prefix: group.prefix + prefix,  
		parent: group,  
		engine: engine,  
	}  
	engine.groups = append(engine.groups, newGroup)  
	return newGroup  
}  
  
func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {  
	pattern := group.prefix + comp  
	log.Printf("Route %4s - %s", method, pattern)  
	group.engine.router.addRoute(method, pattern, handler)  
}  
  
// GET defines the method to add GET request  
func (group *RouterGroup) GET(pattern string, handler HandlerFunc) {  
	group.addRoute("GET", pattern, handler)  
}  
  
// POST defines the method to add POST request  
func (group *RouterGroup) POST(pattern string, handler HandlerFunc) {  
	group.addRoute("POST", pattern, handler)  
}

```
使用案例：
```go
func main() {  
   r := gee.New()  
   r.GET("/index", func(c *gee.Context) {  
      c.HTML(http.StatusOK, "<h1>Index Page</h1>")  
   })  
   v1 := r.Group("/v1")  
   {  
      v1.GET("/", func(c *gee.Context) {  
         c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")  
      })  
  
      v1.GET("/hello", func(c *gee.Context) {  
         // expect /hello?name=geektutu  
         c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)  
      })  
   }  
   v2 := r.Group("/v2")  
   {  
      v2.GET("/hello/:name", func(c *gee.Context) {  
         // expect /hello/geektutu  
         c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)  
      })  
      v2.POST("/login", func(c *gee.Context) {  
         c.JSON(http.StatusOK, gee.H{  
            "username": c.PostForm("username"),  
            "password": c.PostForm("password"),  
         })  
      })  
  
   }  
  
   r.Run(":9999")  
}
```

# 第五天

# 第七天
