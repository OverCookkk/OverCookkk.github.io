---
title: go-interface
tags: [go]      #添加的标签
categories: 
  - GO
description: 本文介绍关于interface底层结构与原理。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/01207-1220238211.png
---

## 介绍

本文介绍的是1.17.10版本的interface相关的源码实现部分。

不同于int, struct等具体类型，接口类型是一种抽象类型，它只是定义了规则，**是一种约定**。通过接口类型的变量或对象，仅仅知道其能提供哪些方法而已。

![go-interface示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go-interface%E7%A4%BA%E6%84%8F%E5%9B%BE.png)



我们需要了解`interface`的内部结构，才能理解这个题目的含义。

interface在使用的过程中，共有两种表现形式

一种为**空接口(empty interface)**，定义如下：

```go
var MyInterface interface{}
```

另一种为**非空接口(non-empty interface)**, 定义如下：

```go
type MyInterface interface {
		function()
}
```





## 空interface类型

相对于非空interface，空interface比较简单，它因为没有约定任何函数，所以任何对象都满足这个约定，都可以转换成空interface。先看看是定义，比较简单：

```go
// empty interface
type eface struct {      //空接口
    _type *_type         //类型信息
    data  unsafe.Pointer //指向数据的指针(go语言中特殊的指针类型unsafe.Pointer类似于c语言中的void*)
}
```

**_type属性**：是GO语言中所有类型的公共描述，Go语言几乎所有的数据结构都可以抽象成 _type，是所有类型的公共描述，**type负责决定data应该如何解释和操作**，type的结构代码如下:

```go
type _type struct {
    size       uintptr  //类型大小
    ptrdata    uintptr  //前缀持有所有指针的内存大小
    hash       uint32   //数据hash值
    tflag      tflag
    align      uint8    //对齐
    fieldalign uint8    //嵌入结构体时的对齐
    kind       uint8    //kind 有些枚举值kind等于0是无效的
    alg        *typeAlg //函数指针数组，类型实现的所有方法
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```

**data属性:** 表示指向具体的实例数据的指针，他是一个`unsafe.Pointer`类型，相当于一个C的万能指针`void*`。



因为是空的约定，所以相比较于非空interface iface中，关于interface的数据结构都没了，保存实现约定的函数集func也不需要了，只保留了具体类型的type和data

![go的空interface类型eface示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E7%9A%84%E7%A9%BAinterface%E7%B1%BB%E5%9E%8Beface%E7%A4%BA%E6%84%8F%E5%9B%BE.png)



## 非空interface类型

iface 表示 non-empty interface 的数据结构，非空接口初始化的过程就是初始化一个iface类型的结构.

下面代码中的IPeople即是非空接口类型，而People因为有GetName 和 GetAge这两个函数，即是说People实现了IPeople。

```go
type IPeople interface {
	GetName() string
	GetAge() int
}

type People struct {
	name string
	age  int
}

func (p People) GetName() string {
	return p.name
}

func (p People) GetAge() int {
	return p.age
}

func (p People) GetHeight() int {
	return 170
}

func test(p IPeople) {
	fmt.Printf("Name: %s, Age: %d \n", p.GetName(), p.GetAge())
}

func main() {
	var p People = People{name: "timefly32", age: 18}
	test(p) // 会先调用runtime.convT2I(SB)，把People转为IPeople
}
```

下面是非空interface数据结构相关的源码，对应上面的应用代码。

```go
type iface struct {
	tab  *itab 
	data unsafe.Pointer //指向具体类型People的数据部分
}

//这个struc很重要，包含接口类型和具体类型的"元数据"
type itab struct {
	inter *interfacetype //接口类型相关，即IPeople相关部分
	_type *_type //具体类型的类型，即People的_type；与eface的type相同
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr //具体类型People实现的方法集，即实现的约定；虽然声明时是固定大小为1，但在使用时会直接通过fun指针获取其中的数据，并且不会检查数组的边界，它是个动态数组
}

type interfacetype struct {
	typ     _type //IPeople的_type
	pkgpath name
	mhdr    []imethod //IPeople定义的函数集，即约定
}
```

![go-interface的iface相关数据结构](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go-interface%E7%9A%84iface%E7%9B%B8%E5%85%B3%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

通过上面的代码和图片可以看到，interface在源码角度上，也是个struct，只不过能因为它是一种约定，一种抽象，所以源码设计上，作者通过iface这个结构来表示非空interface类型的对象，itab(**很重要**)把原interface类型(IPeople)和具体类型(IPeople)的"元数据"都附带上了，其中包含了原interface类型定义的函数集合(**约定**)，和具体类型实现的函数集合(**实现了约定**)，**即约定和实现了约定**。当然，在编译或者runtime创建iface/itab的时候，会根据interface定义函数规则，查找具体类型的方法集，**当任何一个函数没有实现的话，约定即未实现，转换失败**。



## 题目分析

通过题目，复习interface的内部结构：

1. 非空interface类型题目

```go
type People interface {
	Show()
}

type Student struct{}

func (stu *Student) Show() {

}

func live() People {
	var stu *Student
	return stu
}

func main() {
	if live() == nil {
		fmt.Println("AAAAAAA")
	} else {
		fmt.Println("BBBBBBB")
	}
}
```

**结果**

```
BBBBBBB
```

**解析**

People拥有一个Show方法的，属于非空接口，People的内部定义应该是一个`iface`结构体，stu是一个指向nil的空指针，但是最后`return stu` 会触发`匿名变量 People = stu`值拷贝动作，所以最后`live()`返回给上层的是一个`People insterface{}`类型，也就是一个`iface struct{}`类型。 stu为nil，只是`iface`中的data 为nil而已。 但是`iface struct{}`本身并不为nil。

![go-interface的iface题目解析](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go-interface%E7%9A%84iface%E9%A2%98%E7%9B%AE%E8%A7%A3%E6%9E%90.png)



2. 空interface类型题目

```go
func Foo(x interface{}) {
	if x == nil {
		fmt.Println("empty interface")
		return
	}
	fmt.Println("non-empty interface")
}
func main() {
	var p *int = nil
	Foo(p)
}
```

**结果**

```
non-empty interface
```



**分析**

不难看出，`Foo()`的形参`x interface{}`是一个空接口类型`eface struct{}`。

当执行`Foo(p)`时，指针p复制拷贝给空interface类型的x，而x的结构如下：

```go
type eface struct {      //
    _type *_type         // _type为*int
    data  unsafe.Pointer // data = p = nil
}
```

所以x结构体本身不为nil，而是data指针指向的p为nil。



3. inteface{}与*interface{}

ABCD中哪一行存在错误？

```go
type S struct {
}

func f(x interface{}) {
}

func g(x *interface{}) {
}

func main() {
	s := S{}
	p := &s
	f(s) //A
	g(s) //B
	f(p) //C
	g(p) //D
}
```

**结果**

```
B、D两行错误
```

**分析**

因为Golang是强类型语言，interface是所有golang类型的父类 函数中`func f(x interface{})`的`interface{}`可以支持传入golang的任何类型，包括指针，但是函数`func g(x *interface{})`只能接受`*interface{}`