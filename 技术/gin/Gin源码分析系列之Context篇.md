#gin 


## 一、结构体说明

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

> 在`Gin`框架中主要实现了以下的数据处理。

## 二、元数据管理

> 这类的数据管理专门用于为此上下文存储新的键值对。存储在Context中的Keys数据字段中，如果以前没有使用过，它会延迟初始化。

```go
// 存储数据
func (c *Context) Set(key string, value interface{}) {
	c.mu.Lock()
	if c.Keys == nil {
		c.Keys = make(map[string]interface{})
	}

	c.Keys[key] = value
	c.mu.Unlock()
}
// 获取数据，相应操作都是在这个函数的基础上，进行类型的返回
// MustGet(key string) interface{} 返回给定键的值（如果存在），否则会panic。
// GetString(key string) (s string)
// GetBool(key string) (b bool)
// GetInt(key string) (i int)
// GetInt64(key string) (i64 int64)
// GetUint(key string) (ui uint)
// GetUint64(key string) (ui64 uint64)
// GetFloat64(key string) (f64 float64)
// GetTime(key string) (t time.Time)
// GetDuration(key string) (d time.Duration)
// GetStringSlice(key string) (ss []string)
// GetStringMap(key string) (sm map[string]interface{})
// GetStringMapString(key string) (sms map[string]string)
// GetStringMapStringSlice(key string) (smss map[string][]string)
func (c *Context) Get(key string) (value interface{}, exists bool) {
	c.mu.RLock()
	value, exists = c.Keys[key]
	c.mu.RUnlock()
	return
}
```

## 三、获取请求数据

> 处理`HTTP`的`Request`请求数据，主要有以下几个方面的数据

### 3.1、Param数据

> 当路由是这种形式时`router.GET("/user/:id", func(c *gin.Context) {})`，可以通过 `id := c.Param("id")`来获取到`id`的参数值

```go
// 返回URL参数
func (c *Context) Param(key string) string {
	return c.Params.ByName(key)
}
```

### 3.2、Header数据

> 主要调用了`Context`中的`Request`来实现获取头部字段

```go
func (c *Context) GetHeader(key string) string {
	return c.requestHeader(key)
}
func (c *Context) requestHeader(key string) string {
	return c.Request.Header.Get(key)
} 
```

### 3.3、Get数据

> 在`Gin`框架中，`Get`请求的参数都保存在`Context`结构体中的`queryCache`字段中；获取`Get`请求参数的核心方法如下，具体含义可根据字面意思来解释。

```go
// Query(key string) string
// DefaultQuery(key, defaultValue string) string
// GetQuery(key string) (string, bool)
// QueryArray(key string) []string
// QueryMap(key string) map[string]string
// GetQueryMap(key string) (map[string]string, bool)
func (c *Context) GetQueryArray(key string) ([]string, bool) {
	c.initQueryCache()
	if values, ok := c.queryCache[key]; ok && len(values) > 0 {
		return values, true
	}
	return []string{}, false
}

func (c *Context) initQueryCache() {
	if c.queryCache == nil {
		if c.Request != nil {
			c.queryCache = c.Request.URL.Query()
		} else {
			c.queryCache = url.Values{}
		}
	}
}
```

### 3.4、Post数据

> 在`Gin`框架中，`POST`请求的参数都保存在`Context`结构体中的`formCache`字段中；获取`POST`请求参数的核心方法如下，具体含义可根据字面意思来解释。

```go
// PostForm(key string) string
// GetPostForm(key string) (string, bool)
// PostFormArray(key string) []string
// GetPostFormArray(key string) ([]string, bool)
// PostFormMap(key string) map[string]string
// GetPostFormMap(key string) (map[string]string, bool)
func (c *Context) GetPostFormArray(key string) ([]string, bool) {
	c.initFormCache()
	if values := c.formCache[key]; len(values) > 0 {
		return values, true
	}
	return []string{}, false
}
func (c *Context) initFormCache() {
	if c.formCache == nil {
		c.formCache = make(url.Values)
		req := c.Request
		if err := req.ParseMultipartForm(c.engine.MaxMultipartMemory); err != nil {
			if err != http.ErrNotMultipart {
				debugPrint("error on parse multipart form array: %v", err)
			}
		}
		c.formCache = req.PostForm
	}
}
```

### 3.5、File数据

> 在`Gin`框架中，在处理`File`请求时，分为单个文件上传和多个文件上传 ，处理完成后在来实现文件的移动保存。具体处理函数如下：

```go
// 单个文件上传
func (c *Context) FormFile(name string) (*multipart.FileHeader, error) {
	if c.Request.MultipartForm == nil {
		if err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory); err != nil {
			return nil, err
		}
	}
	f, fh, err := c.Request.FormFile(name)
	if err != nil {
		return nil, err
	}
	f.Close()
	return fh, err
}

// 多个文件上传
func (c *Context) MultipartForm() (*multipart.Form, error) {
	err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory)
	return c.Request.MultipartForm, err
}

// 将文件移动保存到目标地址
func (c *Context) SaveUploadedFile(file *multipart.FileHeader, dst string) error {
	src, err := file.Open()
	if err != nil {
		return err
	}
	defer src.Close()

	out, err := os.Create(dst)
	if err != nil {
		return err
	}
	defer out.Close()

	_, err = io.Copy(out, src)
	return err
} 
```

### 3.6、Bind数据

> 在`Gin`中，分别有`JSON`、`XML`、`Form`、`FormPost`、`FormMultipart`、`Uri`、`ProtoBuf`、`MsgPack`、`YAML`、`Header`几种 类型实现了`Binding`接口，可用于在请求中构造结构体（tag标签必须与上面的类型所匹配）来绑定请求数据。 `Gin`具体提供两类绑定方法：`MustBind`和`ShouldBind`，前者如果绑定发生了错误，则请求终止，并响应`400`状态码；后者如果发生了绑定错误，`Gin`会返回错误并由开发者处理错误和请求。 下面以`JSON`为例：

-   `MustBind`

```go
// Bind(obj interface{}) error
// BindJSON(obj interface{}) error
// BindXML(obj interface{}) error
// BindQuery(obj interface{}) error
// BindYAML(obj interface{}) error
// BindHeader(obj interface{}) error
// BindUri(obj interface{}) error
func (c *Context) BindJSON(obj interface{}) error {
	return c.MustBindWith(obj, binding.JSON)
}

func (c *Context) MustBindWith(obj interface{}, b binding.Binding) error {
	if err := c.ShouldBindWith(obj, b); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) // nolint: errcheck
		return err
	}
	return nil
}
```

-   `ShouldBind`

```go
// ShouldBind(obj interface{}) error 
// ShouldBindJSON(obj interface{}) error
// ShouldBindXML(obj interface{}) error
// ShouldBindQuery(obj interface{}) error
// ShouldBindYAML(obj interface{}) error
// ShouldBindHeader(obj interface{}) error
// ShouldBindUri(obj interface{}) error
// ShouldBindWith(obj interface{}, b binding.Binding) error
// ShouldBindBodyWith(obj interface{}, bb binding.BindingBody) (err error)
func (c *Context) ShouldBindJSON(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.JSON)
}

func (c *Context) ShouldBindWith(obj interface{}, b binding.Binding) error {
	return b.Bind(c.Request, obj)
}
```

-   `JSON Bind`

```go
 func (jsonBinding) Bind(req *http.Request, obj interface{}) error {
	if req == nil || req.Body == nil {
		return fmt.Errorf("invalid request")
	}
	return decodeJSON(req.Body, obj)
}

func decodeJSON(r io.Reader, obj interface{}) error {
	decoder := json.NewDecoder(r)
	if EnableDecoderUseNumber {
		decoder.UseNumber()
	}
	if EnableDecoderDisallowUnknownFields {
		decoder.DisallowUnknownFields()
	}
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)  //验证这块就不展开了
}
```

## 3.7、Cookie

```go
func (c *Context) Cookie(name string) (string, error) {
	cookie, err := c.Request.Cookie(name)
	if err != nil {
		return "", err
	}
	val, _ := url.QueryUnescape(cookie.Value)
	return val, nil
}
```

## 四、响应数据

> `Gin`支持这几种渲染：`JSON`,`IndentedJson`,`SecureJSON`,`XML`,`String`，`Data`，`Redirect`,`HTML`,`HTMLDebug`,`HTMLProduction`,`YAML`,`Reader`,  
> `ProtoBuf`,`AsciiJSON`它们都实现了`Render`接口中的`Render`；也实现了`Header`的写入。后期在研究`Render`接口的实现。

-   `Render`

```go
// HTML(code int, name string, obj interface{})
// IndentedJSON(code int, obj interface{})
// SecureJSON(code int, obj interface{})
// JSONP(code int, obj interface{})
// JSON(code int, obj interface{})
// AsciiJSON(code int, obj interface{})
// PureJSON(code int, obj interface{})
// XML(code int, obj interface{})
// YAML(code int, obj interface{})
// ProtoBuf(code int, obj interface{})
// String(code int, format string, values ...interface{})
// Redirect(code int, location string)
// Data(code int, contentType string, data []byte)
// DataFromReader(code int, contentLength int64, contentType string, reader io.Reader, extraHeaders map[string]string)
func (c *Context) JSON(code int, obj interface{}) {
	c.Render(code, render.JSON{Data: obj})
}

func (c *Context) Render(code int, r render.Render) {
	c.Status(code)

	if !bodyAllowedForStatus(code) {
		r.WriteContentType(c.Writer)
		c.Writer.WriteHeaderNow()
		return
	}

	if err := r.Render(c.Writer); err != nil {
		panic(err)
	}
}

func (r JSON) Render(w http.ResponseWriter) (err error) {
	if err = WriteJSON(w, r.Data); err != nil {
		panic(err)
	}
	return
}

func WriteJSON(w http.ResponseWriter, obj interface{}) error {
	writeContentType(w, jsonContentType)
	jsonBytes, err := json.Marshal(obj)
	if err != nil {
		return err
	}
	_, err = w.Write(jsonBytes)
	return err
}
```

-   `Header`

```go
func (c *Context) Status(code int) {
	c.Writer.WriteHeader(code)
}
func (c *Context) Header(key, value string) {
	if value == "" {
		c.Writer.Header().Del(key)
		return
	}
	c.Writer.Header().Set(key, value)
}
```

-   `Cookie`

```go
func (c *Context) SetSameSite(samesite http.SameSite) {
	c.sameSite = samesite
}
// SetCookie将Set-Cookie标头添加到ResponseWriter的标头中。
func (c *Context) SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool) {
	if path == "" {
		path = "/"
	}
	http.SetCookie(c.Writer, &http.Cookie{
		Name:     name,
		Value:    url.QueryEscape(value),
		MaxAge:   maxAge,
		Path:     path,
		Domain:   domain,
		SameSite: c.sameSite,
		Secure:   secure,
		HttpOnly: httpOnly,
	})
}
```

## 五、流程控制

> `Gin`中保存了路由处理信息，在流程控制中统一来执行所保存的信息。在实现的过程中主要调用了`Next`和`Abort`两个方法，如下所示：

```go
// 依次调用处理程序
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}

// AbortWithStatus(code int)
// AbortWithStatusJSON(code int, jsonObj interface{})
// AbortWithError(code int, err error) *Error
// 中止阻止挂起的处理程序被调用，请注意，这不会停止当前的处理程序
func (c *Context) Abort() {
	c.index = abortIndex
} 
```

# Reference
https://zhuanlan.zhihu.com/p/373578225