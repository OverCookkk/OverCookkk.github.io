---
title: go中的堆、栈以及逃逸分析
tags: [go]      #添加的标签
categories: 
  - GO
description: 
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00744-3734460254.png
---



## 堆和栈

调用堆栈是一个LIFO后进先出的堆栈数据结构，用于存储参数、局部变量以及线程执行函数时跟踪的其他数据。每个函数调用都会向栈添加（push）一个新的栈帧（frame），每个返回的函数都会从堆栈中删除（pop）。

当最近的栈帧被弹出时，我们必须能够安全地释放它的内存。因此，**我们不能在栈上存储任何后来需要在其他地方被引用的东西。**

![函数调用时的栈](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E6%97%B6%E7%9A%84%E6%A0%88.png)

由于线程是由操作系统管理的，线程堆栈的可用内存量通常是固定的，例如，在许多Linux环境中默认为8MB。



## Go栈和堆

由操作系统管理的线程被Go运行时完全抽象化了，我们转而使用一种新的抽象（协程）：`Goroutines`。`goroutines`在概念上与线程非常相似，但它们存在于用户空间。这意味着运行时设定了堆栈的行为规则，而不是操作系统。

![go中栈的分布](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E6%A0%88%E7%9A%84%E5%88%86%E5%B8%83.png)

goroutine堆栈不是由操作系统设定的硬性限制，而是以少量内存（目前为2KB）开始。在每个函数调用被执行之前，在函数序言中执行一个检查，以验证堆栈溢出不会发生。在下图中，`convert()`函数可以在当前堆栈大小的限制范围内执行（没有SP溢出堆栈`guard0`）。

![goroutine调用栈的特写](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/goroutine%E8%B0%83%E7%94%A8%E6%A0%88%E7%9A%84%E7%89%B9%E5%86%99.png)

这意味着Go中的堆栈是动态大小的，只要有足够的内存可供使用，堆栈通常可以不断地增长。



Go堆在概念上又与上述的线程模型相似。所有的`goroutine`共享一个公共的堆，任何不能存储在堆上的东西最终都会被放在那里。垃圾回收器的工作是随后释放不再被引用的堆变量。



### 如何知道变量分配到栈中还是堆中呢？

Golang 中的变量只要被引用就一直会存活，**存储在堆上还是栈上由内部实现决定而和具体的语法没有关系。**

通常情况下：

- 分配到栈上：**函数调用的参数**、**返回值**以及**小类型局部变量**大都会被分配到栈上，这部分内存会由编译器进行管理。 无需 GC 的标记。

- 分配到堆上：**大对象**、**逃逸的变量**会被分配到堆上，分配到堆上的对象。Go 的运行时 GC 就会在 后台将对应的内存进行标记从而能够在垃圾回收的时候将对应的内存回收，进而增加了开销。



#### 大对象逃逸

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



#### 指针逃逸

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



## 如何利用逃逸分析提升性能

传值会拷贝整个对象，而传指针只会拷贝指针地址，指向的对象是同一个。传指针可以减少值的拷贝，但是会导致内存分配逃逸到堆中，增加垃圾回收(GC)的负担。在对象频繁创建和删除的场景下，传递指针导致的 GC 开销可能会严重影响性能。

一般情况下，**对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的性能。**
