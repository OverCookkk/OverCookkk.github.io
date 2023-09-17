---
title: gin源码解读
tags: [go]      #添加的标签
categories: 
  - GO
description:
toc: true
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00733-3981599421.png
---



## 从运行服务看底层原理

启动监听8080端口的服务

```go
func main() {
	router := gin.New()
    router.GET("users", func(c *gin.Context){
        ...
    })
    
    router.Run(":8080")
}
```



在router.Run()中可以看到，它本质上是调用go原生http包的ListenAndServe方法，gin的Engine本质上是一个路由处理器，其内部定义了路由处理的各种规则。

```go
// gin.go
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	if engine.isUnsafeTrustedProxies() {
		debugPrint("[WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.\n" +
			"Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.")
	}

	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine.Handler())
	return
}
```



而Engine结构体实现了http.Handler接口的ServeHTTP方法，请求会首先进入到ServeHTTP函数进行处理，具体处理流程如下：

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 1、创建上下文对象，这里用sync.pool来复用内存，进行性能优化
   c := engine.pool.Get().(*Context)
    // 2、初始化上下文对象
   c.writermem.reset(w)
   c.Request = req
   c.reset()

    // 3、处理请求
   engine.handleHTTPRequest(c)

    // 4、回收上下文对象
   engine.pool.Put(c)
}
```



## 处理请求

在上面的第三步中，`engine.handleHTTPRequest(c)`开始处理请求，请求过程如下：

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
   httpMethod := c.Request.Method
   rPath := c.Request.URL.Path
   unescape := false
   if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
      rPath = c.Request.URL.RawPath
      unescape = engine.UnescapePathValues
   }

   if engine.RemoveExtraSlash {
      rPath = cleanPath(rPath)
   }

   // Find root of the tree for the given HTTP method
   // t为压缩前缀树（radix树）数组，每个请求方法有一棵radix树
   t := engine.trees
   for i, tl := 0, len(t); i < tl; i++ {
      if t[i].method != httpMethod {
         continue
      }
      // 1、获取到对应方法的radix树的根节点
      root := t[i].root
      // Find route in tree
      // 2、根据请求路径获取匹配的radix树节点，也就是获取到对应的路由处理器
      value := root.getValue(rPath, c.params, c.skippedNodes, unescape)
      if value.params != nil {
         c.Params = *value.params
      }
      if value.handlers != nil {
         c.handlers = value.handlers
         c.fullPath = value.fullPath
         // 3、执行上下文的Next方法，调用第一个处理器处理请求
         c.Next()
         c.writermem.WriteHeaderNow()
         return
      }
      // 4、后续处理
      if httpMethod != http.MethodConnect && rPath != "/" {
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

   if engine.HandleMethodNotAllowed {
      for _, tree := range engine.trees {
         if tree.method == httpMethod {
            continue
         }
         if value := tree.root.getValue(rPath, nil, c.skippedNodes, unescape); value.handlers != nil {
            c.handlers = engine.allNoMethod
            serveError(c, http.StatusMethodNotAllowed, default405Body)
            return
         }
      }
   }
   c.handlers = engine.allNoRoute
   serveError(c, http.StatusNotFound, default404Body)
}
```



## 中间件

以上第三步执行的`c.Next()`实际上是逐个调用处理器处理请求，**而中间件也是处理器类型**。

```go
func (c *Context) Next() {
   // index指向要执行的中间件
   c.index++
    // 遍历所有处理器（中间件），对请求依次使用中间件进行处理
   for c.index < int8(len(c.handlers)) {
      c.handlers[c.index](c)
      c.index++
   }
}
```



### 注册中间件

可以创建多个路由组，不同的路由组可以有不同的中间件

```go
// 使用全局中间件
r.Use(gin.Recovery())
r.Use(middleware.Logger())
r.Use(middleware.Cors())

// 创建路由组
auth := r.Group("api/v1")
auth.Use(middleware.JwtToken())
```

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
    // Engine继承了RouterGroup，给当前engine注册中间件
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}
```



中间件HandlerFunc是一个函数指针`type HandlerFunc func(*Context)`



## Engine对象初始化

```go
router := gin.New()或者router := gin.Default()
```

`gin.New()`和 `gin.Default()`区别是后者在前者的基础上添加了默认中间件。

`gin.New()`的逻辑如下：

1、初始化Engine对象

2、设置sync.pool的新建上下文对象的函数

```go
func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{	// 初始化路由组
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
        // 以下3项为代理相关配置
		ForwardedByClientIP:    true,
		RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
		TrustedPlatform:        defaultPlatform,
		UseRawPath:             false,
		RemoveExtraSlash:       false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
        // 创建容量为9的radix树切片，对应9种请求方法
		trees:                  make(methodTrees, 0, 9),
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJSONPrefix:       "while(1);",
		trustedProxies:         []string{"0.0.0.0/0", "::/0"},
		trustedCIDRs:           defaultTrustedCIDRs,
	}
	engine.RouterGroup.engine = engine
    // 设置sync.pool的新建上下文对象的函数
	engine.pool.New = func() any {
		return engine.allocateContext()
	}
	return engine
}
```



## 创建路由组和注册路由

```go
auth := r.Group("api/v1")
auth.GET("users", v1.GetUsers)
```

`r.Group()`内部逻辑：

```go
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
    // 返回一个路由组对象
	return &RouterGroup{
        // 新路由组继承父路由组的所有处理器
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
}
```

所以创建路由组，仅是返回路由组对象，省去用户填写相同路径前缀和中间件。



`auth.GET()`内部逻辑：

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    // 计算绝对路径
	absolutePath := group.calculateAbsolutePath(relativePath)
    // 合并处理器（实际上就是将handlers追加到原有的处理器切片中）
	handlers = group.combineHandlers(handlers)
    // 添加路由到radix树中
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}
```





## 上下文

gin的上下文和go原生的上下文不是同一个context

gin上下文方法分为：创建、流程控制、错误管理、元数据管理、请求数据、响应渲染、内容协商。

### Context结构

```go
type Context struct {
   writermem responseWriter
   // 请求对象
   Request   *http.Request
   // 响应对象
   Writer    ResponseWriter
   // 路由参数
   Params   Params
   // 中间件数组
   handlers HandlersChain
   // 当前执行的中间件下标
   index    int8
   fullPath string

   engine       *Engine // 引擎对象
   params       *Params
   skippedNodes *[]skippedNode

   // This mutex protects Keys map.
   mu sync.RWMutex

   // Keys is a key/value pair exclusively for the context of each request.
   Keys map[string]any

   // Errors is a list of errors attached to all the handlers/middlewares who used this context.
   Errors errorMsgs

   // Accepted defines a list of manually accepted formats for content negotiation.
   Accepted []string

   // queryCache caches the query result from c.Request.URL.Query().
   queryCache url.Values

   // formCache caches c.Request.PostForm, which contains the parsed form data from POST, PATCH,
   // or PUT body parameters.
   formCache url.Values

   // SameSite allows a server to define a cookie attribute making it impossible for
   // the browser to send this cookie along with cross-site requests.
   sameSite http.SameSite
}
```



### 注意点

如果上下文Context被传递给协程，必须使用拷贝函数`func (c *Context) Copy() *Context{}`，因为当一个请求被所有处理器处理完成后，Context会被存到内存池里，等待下一个请求复用该上下文，所以协程中的上下文必须是深拷贝的上下文。



## 路由树（radix树）



结构如下：

```go
type node struct {
	path      string
	indices   string
	wildChild bool
	nType     nodeType
	priority  uint32
	children  []*node // child nodes, at most 1 :param style node at the end of the array
	handlers  HandlersChain
	fullPath  string
}
```

