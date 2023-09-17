---
title: go切片
tags: [go]      #添加的标签
categories: 
  - GO
description: 本文主要介绍切片的基础。
toc: true
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00737-2230708892.png
---

零值切片（用`var`声明的切片）可立即使用，无需调用`make()`创建。



## 底层数据结构

切片的本质就是对底层数组的封装，它包含了三个信息：底层数组的指针、切片的长度（len）和切片的容量（cap）。

```go
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```

举个例子，现在有一个数组`a := [8]int{0, 1, 2, 3, 4, 5, 6, 7}`，切片`s1 := a[:5]`，相应示意图如下。

![go_切片底层数据结构](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go_%E5%88%87%E7%89%87%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)



## 切片传参（值拷贝）

1. 使用append对实体参数切片添加内容，并不会改变原切片

   ```go
   func sliceTest(resp []int) {
   	resp = append(resp, 100)
   }
   
   func main() {
   	testSlice1 := []int{1, 3, 5, 1, 6}
   	fmt.Println(testSlice1)	// {1, 3, 5, 1, 6}
   	sliceTest(testSlice1)
   	fmt.Println(testSlice1)	// {1, 3, 5, 1, 6}
   }
   ```
   将一个切片作为函数参数传递给函数时，其实采用的是==**值传递**==，因为`Data`是一个指向数组的指针，所以对该指针进行值拷贝（浅拷贝）时，得到的指针仍指向相同的数组，所以通过拷贝的指针对底层数组进行修改时，原切片的值也会发生相应变化。
   
   **但是**，我们以值传递的方式传递切片结构体的时候，同时也是传递了`Len`和`Cap`的值拷贝，因为这两个成员并不是指针，因此，当函数返回时，指针指向的内容改变了，原切片结构体的`Len`和`Cap`并没有改变，testSlice1的值还是原来的值。
   
   

2. 使用append对指针参数切片添加内容，才能改变原切片

   ```go
   func sliceTest(resp *[]int) {
   	testSlice1 := []int{9, 8, 7, 8, 5}
   	*resp = testSlice1
   	*resp = append(*resp, 100)
   }
   
   func main() {
   	testSlice1 := []int{1, 3, 5, 1, 6}
   	fmt.Println(testSlice1)	// {1, 3, 5, 1, 6}
   	sliceTest(&testSlice1)
   	fmt.Println(testSlice1)	// {9, 8, 7, 8, 5, 100}
   }
   ```

   resp这个指针也是一个原始指针的副本，改变这个指针的内容，原切片也会变化。



3.使用append后，底层数组改变，但是形参的slice的len还是0

```go
func main() {
 sl := make([]int, 0, 10)
 var appenFunc = func(s []int) {
  s = append(s, 10, 20, 30)
 }
 appenFunc(sl)
 fmt.Println(sl)		// 虽然底层数组赋值了10,20,30，但是由于sl的len是0，所以打印出来的是[]
    fmt.Println(sl[:10])	// sl[0:10]，把底层数组的值打印出来了 [10 20 30 0 0 0 0 0 0 0]
}
```





## 切片的赋值拷贝

切片拷贝前后两个变量共享底层数组，对一个切片的修改会影响另一个切片的内容。

以下代码会造成内存泄露，

```go
var a []int

func f(b []int) []int {
 a = b[:2]
 return a
}
```

泄露的点，就在于虽然切片 b 已经在函数内结束了他的使命了，不再使用了。但切片 a 还在使用，切片 a 和 切片 b 引用的是同一块底层数组（共享内存块)。
![go_切片拷贝](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go_%E5%88%87%E7%89%87%E6%8B%B7%E8%B4%9D.png)

虽然切片 a 只有底层数组中 0 和 1 两个索引位正在被使用，其余未使用的底层数组空间毫无作用。**但由于正在被引用，他们也不会被 GC，因此造成了泄露**。



解决方法：

1、利用切片的特性。当切片的容量空间不足时，会**重新申请一个新的底层数组来存储，让两者彻底分手**。

```go
var a []int
func f(b []int) []int{
    a.append(a, b[:2]...)
}
```

a容量为 0。此时将期望的数据，追加过去。自然而然他就会遇到容量空间不足的情况，也就能实现申请新底层数据。



2、使用go的内置函数`copy()`将一个切片的数据赋值到另外一个切片空间汇总，`copy()`函数的使用格式如下：

```go
copy(destSlice, srcSlice []T)
```

```go
func main(){
    a := []int{1,2}
    b := make([]int, 5)
    copy(b, a)	//使用copy()函数将切片a中的元素复制到切片b，深拷贝
}
```



## 切片的扩容问题

当底层数组不能容纳新增的元素时，切片就会自动按照一定的策略进行“扩容”，**此时该切片指向的底层数组就会更换**。切片的容量按照1，2，4，8，16这样的规则自动进行扩容，每次扩容后都是扩容前的**2倍**，当容量达到 2048 时，会采取新的策略，避免申请内存过大，导致浪费。Go 语言源代码 [runtime/slice.go](https://golang.org/src/runtime/slice.go) 中是这么实现的，不同版本可能有所差异。

由于扩容的因素，以下程序会导致无法修改切片的值，

```go

func appendVal(testSlice []string, val string){
   fmt.Printf("testSlice:%p\n", testSlice)
   testSlice = append(testSlice, "addCap") //触发了扩容机制
   fmt.Printf("after append testSlice:%p\n", testSlice)
   testSlice[0] = val
}

func main() {
   var testSlice []string
   testSlice = make([]string, 5)
   testSlice[0] = "foo"
   appendVal(testSlice,"bar")
   fmt.Println(testSlice[0]) //此时打印出的值为foo
}
```

此时因为扩容的影响导致原切片和传递后的切片不再有关联，因此打印值回到了最初的原数据foo。



## nil切片与空切片的区别

我们看下在Go源码中的builtin中的定义：

> nil is a predeclared identifier representing the zero value for a pointer, channel, func, interface, map, or slice type.

翻译成中文的大致含义是：**nil是为pointer、channel、func、interface、map或slice类型预定义的标识符，代表这些类型的零值。**

可见，在Go中，nil代表的是上述类型的零值，切片类型的默认零值是nil。

它们的区别如下：

- nil切片的长度和容量都是0，空切片的长度为0，容量由指向的底层数组决定
- 空切片 != nil切片
- **nil切片的底层ptr指针是nil，而空切片的底层ptr指针指向底层数组的地址（指向了具体地址）**

```go
var s1 []string			 // s1是nil切片
s2 := []string{}		 // s2是空切片
s3 ：= make([]string, 0)	// s3是空切片
```

