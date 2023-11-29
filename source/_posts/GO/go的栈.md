---
title: go中的栈
tags: [go]      #添加的标签
categories: 
  - GO
description: 本文分析go中栈内存的结构
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00744-3734460254.png
---

## go的协程栈

### 作用
协程栈程序内部的示意图，也就是下面这个样子。整体区域就是go中的栈区（RAM stack），里面是放go的栈内存的，中间的小块是放go一个协程的协程栈，一个协程栈的第一个方法是goexit() ，它是为了退出之后重新进行调度用户方法的，后面的就是用户的一个一个方法了。

![go的协程栈示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E7%9A%84%E5%8D%8F%E7%A8%8B%E6%A0%88%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

```
每个协程第一个栈帧为 goexit()

每次调用其他函数会插入一个栈帧

用户的main方法首先会开辟一个main.main的栈帧

栈帧首先记录栈基址（就是指从哪个方法调用进来的）方便返回的时候知道返回地址在哪

开辟调用方法的返回值，return就是将返回值写回上一个栈帧预留的空间
```

作用：

- 记录协程的执行路径（do1() → do2()）
- 存储局部变量（方法内部声明的变量会记录在协程栈中）
- 存储函数传参（方法间的参数传递，例如do2()需要一个入参，do1()是通过栈内存把参数传递给do2()）
- 存储函数返回值（do2()有返回值给do1()，用的也是栈内存传递）



### 位置

Go 语言中，协程（goroutine）的栈是动态地分配在堆上的。每个 goroutine 开始时会在堆上分配一小块栈空间，而这个栈空间会根据需要动态地增长或缩减。

1. 初始栈空间分配：当一个 goroutine 被创建时，它会在堆上获得一小块初始栈空间。这个空间不是分配在传统意义上的栈（即函数调用的栈），而是在堆上。

2. 动态栈大小调整：如果一个 goroutine 在执行过程中需要更多的栈空间（例如，由于深层函数调用或大的局部变量），Go 运行时会自动检测这种情况并在堆上为这个 goroutine 分配更大的栈空间。相反，如果栈空间的使用减少，Go 运行时也可以减少这个 goroutine 的栈空间。

3. 栈和堆的区别：在许多编程语言中，"栈"通常用于存储函数调用的上下文，包括局部变量等，而"堆"用于存储动态分配的内存（如通过 new、malloc 等分配的内存）。在 Go 中，这个界限因为 goroutine 的动态栈特性而变得模糊。

虽然 goroutine 的栈在概念上类似于传统的函数调用栈，但它是动态分配在堆上的，这是 Go 语言高效处理大量 goroutines 的关键之一。



### 结构

```go
package main

func sum(a, b int) int {
  s := 0
  s = a + b
  return s
}

func main()  {
  a := 3
  b := 5
  print(sum(a, b))
}
```

![go程序执行的过程步骤](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C%E7%9A%84%E8%BF%87%E7%A8%8B%E6%AD%A5%E9%AA%A4.png)

1. 局部变量、函数参数等入栈
2. 第二个图是sum从main.main栈参数并计算出`s`的结果
3. `s`的结果写回main预留的内存中
4. 回收sum函数的栈帧

> 往后就是清理sum函数返回值、sum函数参数...，再给print开栈帧



## 协程栈不够大怎么办呢？

在go中协程栈只有2k-4k，协程栈不够大会发生逃逸分析。

### 什么是逃逸分析呢？

Go编译器解析源代码，决定哪些变量分配在stack内存空间，哪些变量分配在heap内存空间的过程就叫做逃逸分析，属于Go代码编译过程中的一个分析环节。

通过逃逸分析，编译器会尽可能把能分配在栈上的对象分配在栈上，避免堆内存频繁GC垃圾回收带来的系统开销，影响程序性能(只有heap内存空间才会发生GC)。



### 指针逃逸

局部变量以指针的方式从方法中传出或被全局变量引用，这种现象被称为指针逃逸（Escape）。我们来分析几种情况：

- 情况一（最基本）：**在某个函数中new或字面量创建出的变量，将其指针作为函数返回值，则该变量一定发生逃逸。**

```go
func main() {
	c := call2()
}
func call2() *int {
	x := 2
	return &x
}
```

```text
go run -gcflags "-m -l" main.go
# command-line-arguments
./main.go:25:2: moved to heap: x	// x发生逃逸
./main.go:22:2: c declared and not used
```



- 情况二：指针作为函数调用参数，**则该变量如果没有被逃逸的变量的或者全局变量引用，指针不会逃逸。**

```text
func main() {
	a := make([]int, 5)
	call(&a)
}

func call(a *[]int) {
	(*a)[0] = 1
}

```

变量a未发生逃逸



- 情况三：**仅仅在函数内对变量做取址操作，而未将指针传出，指针不会逃逸**

```text
package main

func main() {
	a := 2
	b := &a
}
```

变量a未发生逃逸



- 情况四：**该变量如被逃逸的变量的或者全局变量引用，指针会逃逸。**

```text
package main

var g *int

func main() {
	a := 2
	g = &a
}
```

```text
go run -gcflags "-m -l" main.go
\# command-line-arguments
./main.go:6:2: moved to heap: a
```



情况五：**被指针类型的slice、map和chan引用的指针一定发生逃逸**

```go
func main() {
	a := make([]*int, 1)
	b := 12
	a[0] = &b

	c := make(map[string]*int)
	d := 14
	c["aaa"] = &d

	e := make(chan *int, 1)
	f := 15
	e <- &f
}
```

```text
go run -gcflags "-m -l" main.go
\# command-line-arguments
./main.go:5:2: moved to heap: b
./main.go:9:2: moved to heap: d
./main.go:13:2: moved to heap: f
```



- 情况六：**闭包内引用的外部变量会发生逃逸**

```go
func Increase() func() int {
	n := 0
	return func() int {
		n++
		return n
	}
}

func main() {
	in := Increase()
	fmt.Println(in()) // 1
	fmt.Println(in()) // 2
}
```

```text
$ go build -gcflags=-m main_closure.go 
# command-line-arguments
./main_closure.go:6:2: moved to heap: n
```

闭包=引用+环境，闭包让你可以在一个内层函数中访问到其外层函数的作用域。



### 空接口逃逸

> 如果函数的参数为 interface{}，函数的实参很可能会逃逸
> 因为 interface{} 类型的函数往往会使用反射（反射要求对象是在堆上），未使用反射则不会逃逸

```go
package main

import "fmt"

func b() {
  i := 0 // 因为下面的 fmt.Println() 接收的是 interface{}，i会逃逸到堆上
  fmt.Println(i) // func Println(a ...interface{}) (n int, err error) {...}
}

func main() {
  b()
}
```



### 大对象逃逸

过大的变量会导致栈空间不足，在64位机器中，一般超过64KB的变量会逃逸。

```go
func main() {
	a := make([]int, 10000)
	b := make([]int, 1000)
}	
```

```text
go run -gcflags "-m -l" main.go (-m打印逃逸分析信息，-l禁止内联编译)
\# command-line-arguments
./main.go:22:11: make([]int, 10000) escapes to heap //无法被一个执行栈装下，即便没有返回，也会直接在堆上分配；

./main.go:23:11: main make([]int, 1000) does not escape //对象能够被一个执行栈装下，变量没有返回到栈外，进而没有发生逃逸。
```



### 栈帧太多

> 栈空间是从堆中申请的，可以多申请

- Go 栈的初始空间为2KB
- 在函数调用前判断栈空间（morestack），必要时堆栈进行扩容
- 早期使用分段栈，后期使用连续栈

#### 分段栈

分段栈的情况是： 假如第一个栈帧空间不够，直接使用图中箭头指向的空间

可以想象一下，当一个栈里面出现大量栈帧空间不够用时，使用这种方法会在不连续的空间来回跳转

优点： 没有空间浪费
缺点： 栈指针会再不连续的空间跳转（当两块空间中有返回值时）

> 1.13之前使用

![go的栈扩容（分段栈）](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E7%9A%84%E6%A0%88%E6%89%A9%E5%AE%B9%EF%BC%88%E5%88%86%E6%AE%B5%E6%A0%88%EF%BC%89.png)

#### 连续栈

连续栈： 直接将原先栈空间不够的拷贝到新开辟的栈空间

优点： 空间一直连续
缺点： 伸缩时的开销大

原理： 当空间不足时扩容，变为原来的2倍（老的栈空间不足时，会找一块2倍大的栈空间并拷贝过去）；当空间使用率不足1/4时缩容，变为原来的1/2。

![go的栈扩容（连续栈）](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E7%9A%84%E6%A0%88%E6%89%A9%E5%AE%B9%EF%BC%88%E8%BF%9E%E7%BB%AD%E6%A0%88%EF%BC%89.png)



## 如何利用逃逸分析提升性能

传值会拷贝整个对象，而传指针只会拷贝指针地址，指向的对象是同一个。传指针可以减少值的拷贝，但是会导致内存分配逃逸到堆中，增加垃圾回收(GC)的负担。在对象频繁创建和删除的场景下，传递指针导致的 GC 开销可能会严重影响性能。

一般情况下，**对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的性能。**
