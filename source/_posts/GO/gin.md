---
title: go-gin
tags: [go]      #添加的标签
categories: 
  - GO
description:
toc: true
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00732-993264224.png
---

## gin路由



### 基本路由





### API参数（param参数）

可以通过Context的Param方法获取API参数

```go
func main() {
	// 1、创建一个默认的路由引擎
	r := gin.Default()
	// 2、绑定路由规则，执行的函数
	r.GET("/hello/:name/*action", func(c *gin.Context) {
		name := c.Param("name")
		action := c.Param("action")
		c.String(http.StatusOK, name+" is "+action)
	})
	// 3、监听端口启动HTTP服务，默认在127.0.0.0:8080启动服务
	r.Run(":9090")
}
```

访问：

```text
http://127.0.0.1:9090/hello/lihua/sing
```

即lihua就是name，sing就是action



### URL参数（query参数）

URL参数可以通过DefaultQuery()或者Query()方法获取，参数指的是URL中`?`后面携带的参数，例如：`/user/search?username=lihua&address=beijing`

DefaultQuery()：返回默认值

```go
func main() {
	// 1、创建一个默认的路由引擎
	r := gin.Default()
	// 2、绑定路由规则，执行的函数
	r.GET("/welcom", func(c *gin.Context) {
		name := c.DefaultQuery("name", "jack")	//默认值，没有传name，那就用jack
		c.String(http.StatusOK, fmt.Sprintf("hello %s", name))
	})
	// 3、监听端口启动HTTP服务，默认在127.0.0.0:8080启动服务
	r.Run(":9090")
}
```

访问：

```text
http://127.0.0.1:9090/welcom?name=lihua
```



### 表单参数（form）

表单传输为post请求，http常见的传输格式为4种：

- json
- x-www-form-urlencoded
- xml
- form-data

表单参数可以通过PostForm()方法获取，或者直接使用ShouldBindWith()方法绑定结构体，该方法默认解析的是x-www-form-urlencoded或form-data格式的数据。

```go
func Post() {
	r := gin.Default()

	r.POST("/form", func(c *gin.Context) {

		//表单参数设置默认值
		type1 := c.DefaultPostForm("type", "alert")
		//接收其他的
		username := c.PostForm("username")
		password := c.PostForm("password")
		hobbys := c.PostFormArray("hobby")
		c.String(http.StatusOK,
			fmt.Sprintf("type is %s, username is %s, password is %s, hobbys is %v",
				type1, username, password, hobbys))
	})
	r.Run(":9090")
}
```

前端请求界面：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
    <form action="http://127.0.0.1:9090/form" method="post" enctype="application/x-www-form-urlencoded">
        用户名:<input type="text" name="username">
        <br>
        密&nbsp&nbsp码:<input type="password" name="password">
        兴&nbsp&nbsp趣:
        <input type="checkbox" value="run" name="hobby">跑步
        <input type="checkbox" value="game" name="hobby">游戏
        <input type="checkbox" value="money" name="hobby">金钱
        <br>
        <input type="submit" value="登录">
    </form>
</body> 
</html>
```



### 上传文件

`c.FormFile("file")`方法获取上传的文件，其中`"file"`是前端表单中文件字段的名称

```go
func main() {
	r := gin.Default()

	// 设置上传文件的最大大小为 8MB
	r.MaxMultipartMemory = 8 << 20 // 8MB

	r.POST("/upload", func(c *gin.Context) {
		// 单个文件
		file, err := c.FormFile("file")
		if err != nil {
			c.String(http.StatusBadRequest, fmt.Sprintf("获取上传文件失败: %s", err.Error()))
			return
		}

		// 将文件保存到指定路径
		dst := "uploads/" + file.Filename
		err = c.SaveUploadedFile(file, dst)
		if err != nil {
			c.String(http.StatusInternalServerError, fmt.Sprintf("保存上传文件失败: %s", err.Error()))
			return
		}

		c.String(http.StatusOK, fmt.Sprintf("文件 '%s' 上传成功!", file.Filename))
	})

	r.Run(":9090")
}
```





### 路由组（routes group）

路由组是位了管理一些相同的URL

```go
func rout() {
	r := gin.Default()
	// 路由组1，处理get请求
	v1 := r.Group("/user")
	{
		v1.GET("/login", login)
		v1.GET("submit", submit)
	}
    // 路由组2，处理post请求
	v2 := r.Group("/shop")
	{
		v2.POST("/login", login)
		v2.POST("submit", submit)
	}
	r.Run(":9090")
}

func login(c *gin.Context) {
	name := c.DefaultQuery("name", "jack")
	c.String(200, fmt.Sprintf("hello %s\n", name))
}

func submit(c *gin.Context) {
	name := c.DefaultQuery("name", "lily")
	c.String(200, fmt.Sprintf("hello %s\n", name))
}
```

请求：

```text
http://127.0.0.1:9090/v1/login?name=xiaozhang
http://127.0.0.1:9090/v1/submit?name=xiaozhang
```







### 路由原理

httproter会将所有路由规则构造一颗前缀树，树方便查询。





## gin数据解析与绑定



### json数据解析和绑定

客户端传参，后端接收并解析到结构体

```go
// 定义接收数据的结构体
type Login struct {
	// binding:"required"修饰的字段，若接收为空置，则报错，是必须字段
	User     string `form:"username" json:"user" uri:"user" xml:"user" binding:"required"`
	Password string `form:"password" json:"password" uri:"password" xml:"password" binding:"required"`
}

func analysisData() {
	r := gin.Default()
	r.POST("/login_json", func(c *gin.Context) {
		var login Login
		// 将request的body中的数据，自动按照json格式解析到结构体
		err := c.ShouldBindJSON(&login)
		if err != nil {
			// gin.H封装了生成json数据的工具
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		if login.User != "root" || login.Password != "123456" {
			c.JSON(http.StatusBadRequest, gin.H{"status": "304"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "200"})
	})

	r.Run(":9090")
}
```

windows下的请求方法（json中需要加反斜杠）：

```text
curl http://127.0.0.1:9090/login_json -H 'content-type:application/json' -d "{\"user\":\"root\", \"password\":\"123456\"}" -X POST
```





### 表单数据解析和绑定

```go
type Login struct {
	// binding:"required"修饰的字段，若接收为空置，则报错，是必须字段
	User     string `form:"username" json:"user" uri:"user" xml:"user" binding:"required"`
	Password string `form:"password" json:"password" uri:"password" xml:"password" binding:"required"`
}

func analysisData() {
	r := gin.Default()
	r.POST("/form", func(c *gin.Context) {
		var login_form Login
		// Bind()默认解析并绑定form格式
		// 根据请求头中的content-type自动推断
		err := c.Bind(&login_form)
		if err != nil {
			// gin.H封装了生成json数据的工具
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		if login_form.User != "root" || login_form.Password != "123456" {
			c.JSON(http.StatusBadRequest, gin.H{"status": "304"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "200"})
	})

	r.Run(":9090")
}
```

前端请求界面：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
    <form action="http://127.0.0.1:9090/form" method="post" enctype="application/x-www-form-urlencoded">
        用户名:<input type="text" name="username">
        <br>
        密&nbsp&nbsp码:<input type="password" name="password">
        兴&nbsp&nbsp趣:
        <br>
        <input type="submit" value="登录">
    </form>
</body> 
</html>
```



### URI数据解析和绑定

```go
type Login struct {
	// binding:"required"修饰的字段，若接收为空置，则报错，是必须字段
	User     string `form:"username" json:"user" uri:"user" xml:"user" binding:"required"`
	Password string `form:"password" json:"password" uri:"password" xml:"password" binding:"required"`
}

func analysisURI() {
	r := gin.Default()
	r.GET("/:user/:password", func(c *gin.Context) {
		var login_uri Login
		// 根据请求头中的content-type自动推断
		err := c.ShouldBindUri(&login_uri)
		if err != nil {
			// gin.H封装了生成json数据的工具
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		if login_uri.User != "root" || login_uri.Password != "123456" {
			c.JSON(http.StatusBadRequest, gin.H{"status": "304"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "200"})
	})

	r.Run(":9090")
}
```

请求方法：

```text
curl http://127.0.0.1:9090/root/123456
```



## gin渲染

### 各种数据格式的响应

```go
func resp() {
	r := gin.Default()

	// 1、json响应
	r.GET("/someJSON", func(c *gin.Context) {
		c.JSON(200, gin.H{"message": "someJSON", "status": 200})
	})

	// 2、结构体响应（转换成json）
	r.GET("/someStruct", func(c *gin.Context) {
		var msg struct {
			Name    string `json:"name"`
			Message string `json:"message"`
			Number  int
		}
		msg.Name = "root"
		msg.Message = "message"
		msg.Number = 123
        c.JSON(200, msg)
	})

	// 3、xml响应
	r.GET("/someXML", func(c *gin.Context) {
		c.XML(200, gin.H{"message": "abc"})
	})

	// 4、YAML响应
	r.GET("/someYAML", func(c *gin.Context) {
		c.YAML(200, gin.H{"name": "zhangsan"})
	})

	// 5、protobuf
	r.GET("someProtoBuf", func(c *gin.Context) {
		reps := []int64{0, 1}
		// 定义数据
		label := "label"
		// 传protobuf格式数据
		data := &protoexample.Text{
			Label: &label,
			Reps:  reps,
		}
		c.ProtoBuf(200, data)
	})
}
```



### HTML模板渲染

gin支持加载HTML模板，然后根据模板参数进行配置并返回相应的数据，**本质上就是字符串替换**；LoadHTMLGlob()方法可以加载模板文件

```go
// HTML渲染
func HTML_render() {
	r := gin.Default()
	r.LoadHTMLGlob("templates/*")
	r.GET("/index", func(c *gin.Context) {
		// 根据文件名渲染
		// 最终json讲title替换
		c.HTML(200, "index.tmpl", gin.H{"title": "我的标题"})
	})
	r.Run()
}
```

html的模板文件，index.tmpl

```text
<html>
    <h1>
        {{.title}}
    </h1>
</html>
```



### 重定向

```go
func gin_edirect() {
	r := gin.Default()
	r.GET("/redirect", func(c *gin.Context) {
		// 支持内部和外部重定向
		c.Redirect(200, "https://www.runoob.com/")
	})
	r.Run()
}
```



### 同步异步

goroutine机制可以方便地实现异步处理，另外，在启动新的goroutine时，**不应该使用原始上下文，必须使用它的只读副本**

```go
func async_sync() {
	r := gin.Default()
	// 异步
	r.GET("async", func(c *gin.Context) {
		// 需要使用副本
		copyContext := c.Copy()
		// 异步处理
		go func() {
			time.Sleep(3 * time.Second)
			log.Println("异步执行:" + copyContext.Request.URL.Path)
		}()
	})

	// 同步
	r.GET("sync", func(c *gin.Context) {
		// 同步处理
		time.Sleep(3 * time.Second)
		log.Println("异步执行:" + c.Request.URL.Path)
	})
	r.Run()
}
```



## gin中间件

Gin框架允许开发者在处理请求的过程中，加入用户自己的钩子（Hook）函数。这个钩子函数就叫中间件，中间件适合处理一些公共的业务逻辑，比如登录认证、权限校验、数据分页、记录日志、耗时统计等。

- gin可以构建中间件，但它只对注册过的路由函数起作用
- 对于分组路由，嵌套使用中间件，可以限定中间件的作用范围
- 中间件分为全局中间件，单个路由中间件和群组中间件
- gin中间件必须是一个gin.HandlerFunc类型

### 全局中间件

- 所有请求都经过此中间件

```go
func InitRouter() {
	gin.SetMode(utils.AppNode)
	// r := gin.Default()
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(middleware.Logger())

	auth := r.Group("api/v1")
	auth.Use(middleware.JwtToken()) // 加入中间件
	{
		auth.PUT("user/:id", v1.EditUser)
    }
}
```



### Next()方法

```go

func main(){
	router := gin.New()
 
	mid1 := func(c * gin.Context){
		fmt.Println("mid1 start")
		c.Next()
		fmt.Println("mid1 end")
	}
	mid2 := func(c * gin.Context){
		fmt.Println("mid2 start")
		//c.Abort()
		c.Next()
		fmt.Println("mid2 end")
	}
	mid3 := func(c * gin.Context){
		fmt.Println("mid3 start")
		c.Next()
		fmt.Println("mid3 end")
	}
	router.Use(mid1,mid2,mid3)
	router.GET("/",func(c * gin.Context){
		fmt.Println("process get request")
		c.JSON(http.StatusOK,"hello")
	})
	router.Run()
```

上述代码中使用了3个中间件（mid1,mid2,mid3），加上最后的路由处理即返回hello部分，共4个handles。

如果注释掉3个中间件中的c.Next()，则执行情况如下：

```text
mid1 start
mid1 end
mid2 start
mid2 end
mid3 start
mid3 end
process get request
```

![go中间件_next1](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E9%97%B4%E4%BB%B6_next1.png)



如果仅在mid1中间件中使用c.Next()，则执行流程如下：

```text
mid1 start
mid2 start
mid2 end
mid3 start
mid3 end
process get request
mid1 end
```

![go中间件_next2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E9%97%B4%E4%BB%B6_next2.png)

总结：

最后的get路由处理函数可以理解为最后的中间件，在不是调用c.Abort()的情况下，所有的中间件都会被执行到。当某个中间件调用了c.Next(),则整个过程会产生嵌套关系。如果某个中间件调用了c.Abort()，则此中间件结束后会直接返回，后面的中间件均不会调用。



### 局部中间件

```go
func InitRouter() {
	gin.SetMode(utils.AppNode)
	// r := gin.Default()
	r := gin.New()
	r.Use(gin.Recovery())
	auth := r.Group("api/v1")
	{
		auth.PUT("user/:id", middleware.Logger(), v1.EditUser)	//局部中间件
    }
}
```

## 优雅关闭

优雅关机就是服务端关机命令发出后不是立即关机，而是等待当前还在处理的请求全部处理完毕后再退出程序，是一种对客户端友好的关机方式。而执行`Ctrl+C`关闭服务端时，会强制结束进程导致正在访问的请求出现问题。

Go 1.8版本之后， http.Server 内置的 [Shutdown()](https://golang.org/pkg/net/http/#Server.Shutdown) 方法就支持优雅地关机，具体示例如下：

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router, //把gin的路由传给http服务的handler
	}

	go func() {
		// 开启一个goroutine启动服务
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()

	// 等待中断信号来优雅地关闭服务器，为关闭服务器操作设置一个5秒的超时
	quit := make(chan os.Signal, 1) // 创建一个接收信号的通道
	// kill 默认会发送 syscall.SIGTERM 信号
	// kill -2 发送 syscall.SIGINT 信号，我们常用的Ctrl+C就是触发系统SIGINT信号
	// kill -9 发送 syscall.SIGKILL 信号，但是不能被捕获，所以不需要添加它
	// signal.Notify把收到的 syscall.SIGINT或syscall.SIGTERM 信号转发给quit
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)  // 此处不会阻塞
	<-quit  // 阻塞在此，当接收到上述两种信号时才会往下执行
	log.Println("Shutdown Server ...")
	// 创建一个5秒超时的context
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	// 5秒内优雅关闭服务（将未处理完的请求处理完再关闭服务），超过5秒就超时退出
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server Shutdown: ", err)
	}

	log.Println("Server exiting")
}
```



使用gin本身代码实现优化关闭

```go
func main() {
	r := gin.Default()

	r.GET("/ping", func(context *gin.Context) {
        time.Sleep(5 * time.Second)
		context.JSON(http.StatusOK, gin.H{
			"msg": "/pong",
		})
	})
	go func() {
		_ = r.Run(":8081")
	}()

	//等待中断信号，以优雅地关闭服务器
	quit := make(chan os.Signal)
	// 可以捕捉除了kill-9的所有中断信号
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    // 阻塞
	<-quit
	fmt.Println("收到中断信号;优雅的退出...")
	fmt.Println("退出完成")
```
