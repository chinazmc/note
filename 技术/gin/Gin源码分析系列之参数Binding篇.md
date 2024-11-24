#gin 


## 一、说明

> 在Gin框架中内置了几种数据的绑定例如JSON, XML等。简单来说, 即根据Body数据类型, 将数据赋值到指定的结构体变量中. (类似于序列化和反序列化)，下面一一说明。

## 二、Binding

> Gin主要提供了两类绑定方法：Must Bind 和 Should Bind。

### 2.1、Must Bind

> Must Bind 包含了Bind、BindJson、BindXML、BindQuery、BindYaml，这些方法都属于MustBindWith的调用。如果发生绑定错误，则请求将终止，并触发 c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind)。

```go
func (c *Context) Bind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.MustBindWith(obj, b)
}
func (c *Context) BindJSON(obj interface{}) error {
	return c.MustBindWith(obj, binding.JSON)
}
func (c *Context) BindXML(obj interface{}) error {
	return c.MustBindWith(obj, binding.XML)
}
func (c *Context) BindQuery(obj interface{}) error {
	return c.MustBindWith(obj, binding.Query)
}
func (c *Context) BindYAML(obj interface{}) error {
	return c.MustBindWith(obj, binding.YAML)
}
func (c *Context) BindHeader(obj interface{}) error {
	return c.MustBindWith(obj, binding.Header)
}
func (c *Context) BindUri(obj interface{}) error {
	if err := c.ShouldBindUri(obj); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) // nolint: errcheck
		return err
	}
	return nil
}
func (c *Context) MustBindWith(obj interface{}, b binding.Binding) error {
	if err := c.ShouldBindWith(obj, b); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) // nolint: errcheck
		return err
	}
	return nil
}
```

### 2.2、Should Bind

> Should Bind包含了ShouldBind、ShouldBindJson、ShouldBindXML、ShouldBindQuery、ShouldBindYAML。这些方法属于ShouldBindWith的具体调用，如果发生绑定错误，Gin会返回错误并由开发者自行处理。

```go
func (c *Context) ShouldBind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.ShouldBindWith(obj, b)
}
func (c *Context) ShouldBindJSON(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.JSON)
}
func (c *Context) ShouldBindXML(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.XML)
}
func (c *Context) ShouldBindQuery(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.Query)
}
func (c *Context) ShouldBindYAML(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.YAML)
}
func (c *Context) ShouldBindHeader(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.Header)
}
func (c *Context) ShouldBindUri(obj interface{}) error {
	m := make(map[string][]string)
	for _, v := range c.Params {
		m[v.Key] = []string{v.Value}
	}
	return binding.Uri.BindUri(m, obj)
}
func (c *Context) ShouldBindWith(obj interface{}, b binding.Binding) error {
	return b.Bind(c.Request, obj)
}
```

### 2.3、内置绑定类型

> Gin提供了json、xml、protobuf等的内置绑定类型，它们都是实现了Bind方法。并且统一调用Binding接口中的validate方法来实现对参数的校验。

```go
type Binding interface {
	Name() string  // 各内置绑定类型的校验标签
	Bind(*http.Request, interface{}) error
}
type StructValidator interface {
	ValidateStruct(interface{}) error
	Engine() interface{}
}
```

-   `json`

```go
func (jsonBinding) Name() string {
	return "json"
}
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
	return validate(obj) //校验
}
```

-   `xml`

```go
func (xmlBinding) Name() string {
	return "xml"
}
func (xmlBinding) Bind(req *http.Request, obj interface{}) error {
	return decodeXML(req.Body, obj)
}
func decodeXML(r io.Reader, obj interface{}) error {
	decoder := xml.NewDecoder(r)
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)
}
```

-   `protobuf`

```go
func (protobufBinding) Name() string {
	return "protobuf"
}
func (b protobufBinding) Bind(req *http.Request, obj interface{}) error {
	buf, err := ioutil.ReadAll(req.Body)
	if err != nil {
		return err
	}
	return b.BindBody(buf, obj)
}

func (protobufBinding) BindBody(body []byte, obj interface{}) error {
	if err := proto.Unmarshal(body, obj.(proto.Message)); err != nil {
		return err
	}
	return nil
}
```

-   `msgpack`

```go
func (msgpackBinding) Name() string {
	return "msgpack"
}
func (msgpackBinding) Bind(req *http.Request, obj interface{}) error {
	return decodeMsgPack(req.Body, obj)
}
func decodeMsgPack(r io.Reader, obj interface{}) error {
	cdc := new(codec.MsgpackHandle)
	if err := codec.NewDecoder(r, cdc).Decode(&obj); err != nil {
		return err
	}
	return validate(obj)
} 
```

-   `yaml`

```go
func (yamlBinding) Name() string {
	return "yaml"
}
func (yamlBinding) Bind(req *http.Request, obj interface{}) error {
	return decodeYAML(req.Body, obj)
}
func decodeYAML(r io.Reader, obj interface{}) error {
	decoder := yaml.NewDecoder(r)
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)
}
```

-   `form-multipart`

```go
func (formBinding) Name() string {
	return "form"
}
func (formBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := req.ParseMultipartForm(defaultMemory); err != nil {
		if err != http.ErrNotMultipart {
			return err
		}
	}
	if err := mapForm(obj, req.Form); err != nil {
		return err
	}
	return validate(obj)
}

func (formPostBinding) Name() string {
	return "form-urlencoded"
}
func (formPostBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := mapForm(obj, req.PostForm); err != nil {
		return err
	}
	return validate(obj)
}

func (formMultipartBinding) Name() string {
	return "multipart/form-data"
}
func (formMultipartBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseMultipartForm(defaultMemory); err != nil {
		return err
	}
	if err := mappingByPtr(obj, (*multipartRequest)(req), "form"); err != nil {
		return err
	}

	return validate(obj)
}
```

-   `form`

```go
func (formBinding) Name() string {
	return "form"
}
func (formBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := req.ParseMultipartForm(defaultMemory); err != nil {
		if err != http.ErrNotMultipart {
			return err
		}
	}
	if err := mapForm(obj, req.Form); err != nil {
		return err
	}
	return validate(obj)
}

func (formPostBinding) Name() string {
	return "form-urlencoded"
}
func (formPostBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := mapForm(obj, req.PostForm); err != nil {
		return err
	}
	return validate(obj)
}

func (formMultipartBinding) Name() string {
	return "multipart/form-data"
}
func (formMultipartBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseMultipartForm(defaultMemory); err != nil {
		return err
	}
	if err := mappingByPtr(obj, (*multipartRequest)(req), "form"); err != nil {
		return err
	}

	return validate(obj)
}
```

-   `query`

```go
func (queryBinding) Name() string {
	return "query"
}

func (queryBinding) Bind(req *http.Request, obj interface{}) error {
	values := req.URL.Query()
	if err := mapForm(obj, values); err != nil {
		return err
	}
	return validate(obj)
}
```

-   `form-post`

```go
func (formBinding) Name() string {
	return "form"
}

func (formBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := req.ParseMultipartForm(defaultMemory); err != nil {
		if err != http.ErrNotMultipart {
			return err
		}
	}
	if err := mapForm(obj, req.Form); err != nil {
		return err
	}
	return validate(obj)
}

func (formPostBinding) Name() string {
	return "form-urlencoded"
}

func (formPostBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := mapForm(obj, req.PostForm); err != nil {
		return err
	}
	return validate(obj)
}

func (formMultipartBinding) Name() string {
	return "multipart/form-data"
}

func (formMultipartBinding) Bind(req *http.Request, obj interface{}) error {
	if err := req.ParseMultipartForm(defaultMemory); err != nil {
		return err
	}
	if err := mappingByPtr(obj, (*multipartRequest)(req), "form"); err != nil {
		return err
	}

	return validate(obj)
} 
```

-   `uri`

```go
func (uriBinding) Name() string {
	return "uri"
}

func (uriBinding) BindUri(m map[string][]string, obj interface{}) error {
	if err := mapUri(obj, m); err != nil {
		return err
	}
	return validate(obj)
} 
```

-   `header`

```go
func (headerBinding) Name() string {
	return "header"
}
func (headerBinding) Bind(req *http.Request, obj interface{}) error {

	if err := mapHeader(obj, req.Header); err != nil {
		return err
	}

	return validate(obj)
}
func mapHeader(ptr interface{}, h map[string][]string) error {
	return mappingByPtr(ptr, headerSource(h), "header")
} 
```

# Reference
https://zhuanlan.zhihu.com/p/373885393