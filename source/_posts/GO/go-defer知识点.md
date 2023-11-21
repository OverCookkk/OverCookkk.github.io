---
title: go-defer知识点
tags: [go]      #添加的标签
categories: 
  - GO
description: 在用 Golang 开发的时候，defer 这个语法也是必备的知识，但是我们除了知道他是在一个函数退出之前执行，对于 defer 是否还有其他地方需要注意的呢。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/391071.jpeg
---

本文整理的 `defer` 的全场景使用情况。

##  defer 的执行顺序

Go语言中的`defer`语句会将其后面跟随的语句进行延迟处理。在`defer`归属的函数即将返回时，将延迟处理的语句按`defer`定义的**逆序**进行执行，也就是说“栈的关系”，先被`defer`的语句最后被执行，最后被`defer`的语句，最先被执行。



## defer 与 return 谁先谁后

在Go语言的函数中`return`语句在底层并不是原子操作，它分为给返回值赋值和RET指令两步。而`defer`语句执行的时机就在返回值赋值操作后，RET指令执行前。

例子：

```go
func f1() int {
	x := 5
	defer func() {
		x++
	}()
	return x
}

func f2() (x int) {
	defer func() {
		x++
	}()
	return 5
}

func f3() (y int) {
	x := 5
	defer func() {
		x++
	}()
	return x
}
func f4() (x int) {
	defer func(x int) {
		x++	//改变的是函数中的x的副本
	}(x)
	return 5
}
func main() {
	fmt.Println(f1()) // 5
	fmt.Println(f2()) // 6
	fmt.Println(f3()) // 5
	fmt.Println(f4()) // 5
}
```



## defer 遇见 panic

遇到 panic 时，会触发defer出栈执行 defer。在执行 defer 过程中：遇到 recover 则停止 panic，返回 recover 处继续往下执行。如果没有遇到 recover，执行完本协程的所有 defer 后，向 stderr 抛出 panic 信息。

### defer 遇见 panic，但是并不捕获异常的情况：

```go
func main() {
    defer_func()

    fmt.Println("main 正常结束")
}

func defer_func() {
    defer func() { fmt.Println("defer1") }()
    defer func() { fmt.Println("defer2") }()

    panic("异常信息")  //触发defer出栈

    defer func() { fmt.Println("defer3: 在panic之后，永远执行不到") }()
}
```

**结果**

```text
defer2
defer1
panic: 异常信息
//... 异常堆栈信息
```



### defer 遇见 panic，并捕获异常

```go
func main() {
    defer_func()

    fmt.Println("main 正常结束")
}

func defer_func() {

    defer func() {
        fmt.Println("defer1:捕获异常")
        if err := recover(); err != nil {
            fmt.Println(err)
        }
    }()

    defer func() { fmt.Println("defer2") }()

    panic("异常信息")  //触发defer出栈

    defer func() { fmt.Println("defer3: panic 之后, 永远执行不到") }()
}
```

**结果**

```text
defer2
defer1:捕获异常
异常信息
main 正常结束
```



## defer 中包含 panic

**panic 仅有最后一个可以被 revover 捕获**。

```go
func main()  {

    defer func() {
       if err := recover(); err != nil{
           fmt.Println(err)
       }else {
           fmt.Println("fatal")
       }
    }()

    defer func() {
        panic("defer panic2")
    }()

    panic("panic1")
}
```

**结果**

```text
defer panic2
```

当执行到`panic("panic1")`语句后，触发第一个panic，然后defer 顺序出栈执行，先执行`panic("defer panic2")`，这panic会覆盖的第一个panic，最后执行第二个defer，所以捕获了第二个panic。



## defer 下的函数参数包含子函数

**defer入栈时，需要把函数地址、函数形参一同压进栈**

1. 例子1：

```go
func function(index int, value int) int {

    fmt.Println(index)

    return index
}

func main() {
    defer function(1, function(3, 0))
    defer function(2, function(4, 0))
}
```

结果

```text
3
4
2
1
```

defer 一共会压栈两次，先进栈 1，后进栈 2。 那么在压栈`function(1, function(3, 0))`的时候，需要把**函数地址**、**函数形参**一同压进栈，那么为了得到第二个参数的结果，所以就需要先执行 `function(3, 0)`，将第二个参数算出，所以第一个结果是3；同理，`function(2, function(4, 0))`也一样。



2. **闭包引用外部环境变量（创建新的副本）**

```go
func DeferFunc() (t int) {
    defer func(i int) {
        fmt.Println(i)
        fmt.Println(t)
    }(t)
    t = 1
    return 2
}

func main() {
    DeferFunc()
}
```

结果

```text
0
2
```

（1）初始化返回值 t 为零值 0

（2）defer入栈，同时需要把它对应的函数参数也入栈，因为`t=0`，所以`i=0`，所以第一个println会打印0

（3）t被赋值为1，然后return，t被赋值为2，最后第二个println打印2



3. **闭包引用外部环境变量**

```go
func main() {
 var whatever [6]struct{}
 for i := range whatever {
  defer func() {
   fmt.Println(i)
  }()
 }
}
```

结果：

```text
5,5,5,5,5
```

闭包=函数+引用环境，在 for 循环结束后，局部变量 i 的值已经是 5 了，并且defer的闭包是直接引用变量的 i。



4. **defer链式调用，优先从左到右计算，只对最后的函数入栈**

```go
type temp struct{}

func (t *temp) Add(elem int) *temp {
    fmt.Println(elem)
    return &temp{}
}

func main() {
    tt := &temp{}
    defer tt.Add(1).Add(5).Add(2)
    tt.Add(3)
}
```

结果

```text
1
5
3
2
```

defer压栈只会压最后一个Add，前两个Add会在压栈前优先从左往右计算。


## defer遇见exit

```go
func test1() {
	fmt.Println("test")
}

func main() {
	fmt.Println("main start")
	defer test1()
	fmt.Println("main end")
	os.Exit(0)
}
```

结果

```text
main start
main end
```

如果在函数里是因为执行了os.Exit而退出，而不是正常return退出或者panic退出，那程序会立即停止，被defer的函数调用不会执行。



## 被defer的函数或方法的参数的值在执行到defer语句的时候就被确定下来了

```go
func a() {
    i := 0
    defer fmt.Println(i) // 最终打印0
    i++
    return
}
```

因为执行defer时候，对i的值进行入栈操作，此时i是0，所以是0入栈