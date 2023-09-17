---
title: go设计模式的实现
tags: [go]      #添加的标签
categories: 
  - GO
description: 使用go语言实现单例、观察者等常用的设计模式。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00738-2959395181.png
---



## 单例模式

单例模式是用来控制类型实例的数量的，当需要确保一个类型只有一个实例时，就需要使用单例模式。



### 饿汉模式

该模式适用于在程序早期初始化时创建已经确定需要加载的类型实例，比如项目的数据库实例。 Go 语言实现时，借助 Go 的`init`函数来实现特别方便，虽然 `init()` 函数在每个 package 中都被执行了多次，但是在同一个 package 中全局变量只会被初始化一次。

```go
package dao
// 注意定义非导出类型
type databaseConn struct{
    ...
}

var dbConn *databaseConn

func init(){
    dbConn = &databaseConn{}
}

func GetInstance() *databaseConn{
    return dbConn
}
```



C++实现方式：

```c++
```







### 懒汉模式

该模式就是延迟加载，要考虑并发环境下，你的判断实例是否已经创建时，是不是用的当前读。

```go
package singleton
import (
    "sync"
)

type singleton struct{}
var instance *singleton
var once sync.Once

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
}
```





## 策略模式

​		定义一类算法族，将每个算法分别封装起来，让他们可以互相替换，此模式让算法的变化独立于使用算法的客户端。



例子：用户支付，客户端使用微信支付、或者是其他三方在线支付。

- PayBehavior：抽象策略，对支付任务进行接口抽象
- WxPay 和 ThirdPay ：是具体的策略实现
- PaxCtx：上下文对象在这里有两个作用，第一是协调自己持有的 PayBehavior 具体实现，完成支付的任务，第二是维护发起支付需要的支付参数--即图中的私有属性`payParams`。



```go
// 定义PayBehavior 策略的接口
type PayBehavior interface {
     OrderPay(px *PayCtx)
}
```

有了接口后，我们来定义两个策略的实现：

```go
// 具体支付策略实现
// 微信支付
type WxPay struct{}

func (*WxPay) OrderPay(px *PayCtx) {
	fmt.Printf("Wx支付加工支付请求 %v\n", px.payParams)
	fmt.Println("正在使用Wx支付进行支付")
}

// 三方支付
type ThirdPay struct{}

func (*ThirdPay) OrderPay(px *PayCtx) {
	fmt.Printf("三方支付加工支付请求 %v\n", px.payParams)
	fmt.Println("正在使用三方支付进行支付")
}
```

有了策略的实现后，还得有个上下文来协调它们，以及持有完成这个任务所必需的那些入参`payParams`:

```go
type PayCtx struct {
	// 提供支付能力的接口实现
	payBehavior PayBehavior
	// 支付参数
	payParams map[string]interface{}
}

func (px *PayCtx) setPayBehavior(p PayBehavior) {
	px.payBehavior = p
}

func (px *PayCtx) Pay() {
	px.payBehavior.OrderPay(px)
}

func NewPayCtx(p PayBehavior) *PayCtx {
	// 支付参数，Mock数据
	params := map[string]interface{}{
		"appId": "234fdfdngj4",
		"mchId": 123456,
	}
	return &PayCtx{
		payBehavior: p,
		payParams:   params,
	}
}
```

调用过程：

```go

func main() {
	wxPay := &WxPay{}
	px := NewPayCtx(wxPay)
	px.Pay()
	// 假设现在发现微信支付没钱，改用三方支付进行支付
	thPay := &ThirdPay{}
	px.setPayBehavior(thPay)
	px.Pay()
}
```





## 责任链模式

它是一种行为型设计模式。使用这个模式，我们能为请求创建一条由多个处理器组成的链路，每个处理器各自负责自己的职责，相互之间没有耦合，完成自己任务后请求对象即传递到链路的下一个处理器进行处理。

责任链在很多流行框架里都有被用到，像中间件、拦截器等框架组件都是应用的这种设计模式。

### 背景

上面我们说了职责链在项目公共组件中的一些应用，让我们能在核心逻辑的前置和后置流程中增加一些基础的通用功能。但其实在一些核心的业务中，**应用职责链模式能够让我们无痛地扩展业务流程的步骤**。

比如淘宝在刚刚创立的时候购物生成订单处理流程起初可能是这样的。

![淘宝下单简单流程图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E6%B7%98%E5%AE%9D%E4%B8%8B%E5%8D%95%E7%AE%80%E5%8D%95%E6%B5%81%E7%A8%8B%E5%9B%BE.png)



整个流程比较干净**"用户参数校验--购物车数据校验--商品库存校验--运费计算--扣库存—生成订单"**，我们姑且把它称为清纯版的购物下单流程，这通常都是在产品从0到1的时候，流程比较清纯，在线购物你能实现在线选品、下单、支付这些就行了。

责任链模式抽象成 UML 类图如下：

![责任链模块UML图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%9D%97UML%E5%9B%BE.png)



### 具体实现

下单接口结构如下：

```go
type OrderHandler interface {
	SetNext(OrderHandler) OrderHandler
	Execute(*order) error
	Do() error
}
```


- 成员方法

- - `SetNext`: 把下一个对象的实例绑定到当前对象的`nextHandler`属性上；
  - `Do`: 当前对象业务逻辑入口，他是每个处理对象实现自己逻辑的地方；
  - `Execute`: 负责职责链上请求的处理和传递；它会调用当前对象的`Do`，`nextHandler`不为空则调用`nextHandler.Do`；



Next类充当抽象类型，实现公共方法（SetNext和Execute），抽象方法（Do）不实现留给实现类自己实现

```go
type Next struct {
	nextHandler OrderHandler	// 下一个等待被调用的对象实例
}

// SetNext 把下一个对象的实例绑定到当前对象的nextHandler属性上，并返回下一个对象的实例
func (n *Next) SetNext(handler OrderHandler) OrderHandler {
	n.nextHandler = handler
	return handler
}

func (n *Next) Execute(o *order) (err error) {
	if n.nextHandler != nil {
		if err = n.nextHandler.Do(); err != nil { // 执行流程里的内容
			return
		}
		return n.nextHandler.Execute(o) // 调用下一个流程
	}
	return
}
```

- 成员属性

  - `nextHandler`: 下一个等待被调用的对象实例



具体流程实现结构体如下：

```go
// DataCheck 数据校验管理器；该结构体实现了OrderHandler接口
type DataCheck struct {
	Next
}

func (m *DataCheck) Do() (err error) {
	fmt.Println("doing UserDataCheck")
	return
}

// FreightCalc 运费计算；该结构体实现了OrderHandler接口
type FreightCalc struct {
	Next
}

func (m *FreightCalc) Do() (err error) {
	fmt.Println("doing FreightCalc")
	return
}

// DecreaseRepe 扣库存；该结构体实现了OrderHandler接口
type DecreaseRepe struct {
	Next
}

func (m *DecreaseRepe) Do() (err error) {
	fmt.Println("doing DecreaseRepe")
	return
}

// StartHandler 不做操作，作为第一个Handler向下转发请求
// Go 语法限制，抽象公共逻辑到通用Handler后，并不能跟继承一样让公共方法调用不通子类的实现
type StartHandler struct {
	Next
}

// Do 空Handler的Do
func (h *StartHandler) Do(c *order) (err error) {
	// 空Handler 这里什么也不做 只是载体 do nothing...
	return
}

func main() {
	startHandler := &StartHandler{}
	// 把下单流程链路串起来
    startHandler.SetNext(&DataCheck{}).	// DataCheck{}对象隐式转换成了OrderHandler接口
		SetNext(&FreightCalc{}).
		SetNext(&DecreaseRepe{})

	o := &order{name: "abc"}
	startHandler.Execute(o)
	fmt.Println("end")
}

type order struct {
	name string
}
```





## 代理模式

**代理模式**是一种结构型设计模式。 其中代理控制着对于原对象的访问， 并允许在将请求提交给原对象的前后进行一些处理，从而增强原对象的逻辑处理。

进行逻辑处理的原对象通常被称作服务对象，**代理要跟服务对象实现相同的接口**，才能让客户端傻傻分不清自己使用的到底是代理还是真正的服务对象，这样一来代理就能在客户端察觉不到的情况下对服务对象的处理逻辑进行增强。

UML图如下：

![代理模式UML图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8FUML%E5%9B%BE.png)





### 具体实现

假设有一个代表小汽车的 Car 类型，小汽车要的主要行为就是可以让人驾驶，所以 Car 需要实现一个代表驾驶行为的接口（interface）Vehicle，该接口只有一个方法`Drive()`

```go
// 接口
type Vehicle interface {
	Drive()
}

// Car结构体实现了Vehicle接口
type Car struct{}

func (c *Car) Drive() {
	fmt.Println("Car is being driven")
}
```

果在开车时要加一个驾驶员的年龄限制，我们该怎么办呢？ 通常的做法是，加一个表示驾驶员的类型 `Driver`。

```go
type Driver struct {
	Age int
}
```

然后再来一个包装 Driver 和 Vehicle 类型的包装类型，也就是所谓的代理类。

```go
// CarProxy结构体实现了Vehicle接口
type CarProxy struct {
	vehicle    Vehicle	// 接口对象
	driver *Driver
}

// 构造带有Car的CarProxy对象
func NewCarProxy(driver *Driver) *CarProxy {
	return &CarProxy{&Car{}, driver}
}

func (c *CarProxy) Drive() {
	if c.driver.Age >= 16 {
		c.vehicle.Drive()
	} else {
		fmt.Println("Driver too young!")
	}
}
```

调用：

```go
func main() {
 car := NewCarProxy(&Driver{12})
 car.Drive() // 输出 Driver too young!
 car2 := NewCarProxy(&Driver{22})
 car2.Drive() // 输出 Car is being driven
}
```

