---
title: go使用注意点
tags: [go]      #添加的标签
categories: 
  - GO
#description: 
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00740-2959395182.png
---

1、需要注意`sync.WaitGroup`是一个结构体，进行参数传递的时候要传递指针。

2、Go语言中的`defer`语句会将其后面跟随的语句进行延迟处理。在`defer`归属的函数即将返回时，将延迟处理的语句按`defer`定义的**逆序**进行执行，也就是说，先被`defer`的语句最后被执行，最后被`defer`的语句，最先被执行。

在Go语言的函数中`return`语句在底层并不是原子操作，它分为给返回值赋值和RET指令两步。而`defer`语句执行的时机就在返回值赋值操作后，RET指令执行前。

defer经典案例

阅读下面的代码，写出最后的打印结果。

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
	fmt.Println(f1())
	fmt.Println(f2())
	fmt.Println(f3())
	fmt.Println(f4())
}
```

defer面试题

```go
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
	x := 1
	y := 2
    defer calc("AA", x, calc("A", x, y))	//先算calc("A", x, y)，得到值付给calc("AA",x,   )，执行defer压栈，此时压栈传入的x是1，不是下面的10
	x = 10
	defer calc("BB", x, calc("B", x, y))
	y = 20
}
```

```go
type temp struct{}

func (t *temp) Add(elem int) *temp {
    fmt.Println(elem)
    return &temp{}
}

func main() {
    tt := &temp{}
    defer tt.Add(1).Add(5).Add(2)	// 从左到右会先计算Add(1).Add(5)，最后的Add(2)才会压栈
    tt.Add(3)
}
```

```go
func main() {
 var whatever [6]struct{}
 for i := range whatever {
  defer func() {
   fmt.Println(i)
  }()
 }
}
// 这段程序的输出结果是：5,5,5,5,5
// 闭包=函数+引用环境，在 for 循环结束后，局部变量 i 的值已经是 5 了，并且 defer的闭包是直接引用变量的 i。
```

```go
func main() {
 var whatever [6]struct{}
 for i := range whatever {
  defer func(i int) {
   fmt.Println(i)
  }(i)
 }
}
// 输出：5,4,3,2,1,0
// 在 for 循环时，局部变量 i 已经传入进 defer func 中 ，属于值传递。其值在 defer 语句声明时的时候就已经确定下来了。
```





3、

在做对外接口的数据对接和转换时，我们经常需要对 JSON 数据进行处理。

如下代码：

```go
type T struct {
 name string
 age  int
}

func main() {
 p := T{"煎鱼", 18}
 jsonData, _ := json.Marshal(p)
 fmt.Println(string(jsonData))
}
```

输出结果是空，原因是 JSON 的输出，只会输出公开（导出）字段，也就是结构体T里面的字段name和age首字母必须为大写。



4、

程序运行期间`funcB`中引发了`panic`导致程序崩溃，异常退出了。这个时候我们就可以通过`recover`将程序恢复回来，继续往后执行。

```go
func funcA() {
	fmt.Println("func A")
}

func funcB() {
	defer func() {
		err := recover()
		//如果程序出出现了panic错误,可以通过recover恢复过来
		if err != nil {
			fmt.Println("recover in B")
		}
	}()
	panic("panic in B")
}

func funcC() {
	fmt.Println("func C")
}
func main() {
	funcA()
	funcB()
	funcC()
}
```

**注意：**

1. `recover()`必须搭配`defer`使用。

2. `defer`一定要在可能引发`panic`的语句之前定义。

3. panic 只会对当前 Goroutine 的 defer 有效

   ```go
   func main() {
    defer func() {
     if err := recover(); err !=nil{
      fmt.Println(err)
     }
    }()
    go func() {
     fmt.Println("======begin work======")
     panic("nil pointer exception")
    }()
    time.Sleep(time.Second*100)
    fmt.Println("======after work======")
   }
   ```
   以上的panic是在另一个goroutine中定义的，所以对main中的defer无法cover住这个异常。









5、

在 Go 中，循环迭代器变量是一个单一的变量，在每个循环迭代中取不同的值。这如果使用不当，可能会导致非预期的行为。

如下代码：

```go
func main() {
 var out []*int
 for i := 0; i < 3; i++ {
  out = append(out, &i)
 }
 fmt.Println("Values:", *out[0], *out[1], *out[2])
 fmt.Println("Addresses:", out[0], out[1], out[2])
}
```

输出结果：

```text
Values: 3 3 3
Addresses: 0x40e020 0x40e020 0x40e020
```

原因是：在每次迭代中，我们将 i 的地址追加到 out 切片中，但由于它是同一个变量，我们实际上追加的是相同的地址，该地址最终包含分配给 i 的最后一个值。



6、for...range...拷贝副本

```go
func main() {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}

	for k, v := range m {
		fmt.Printf("k:%s v:%v", k, v)
	}
}
```

输出结果如下：

```text
k:zhou v:&{wang 22}
k:li v:&{wang 22}
k:wang v:&{wang 22}
```

原因：for range每次产生的k，v都是一个值拷贝，不是stus值对应的引用。

详解：for中，第一次循环，变量stu被赋值，指向"zhou"，然后放到map中保存，第二次循环，变量stu的被重新赋值，改为指向“li”，这时map中的指向也跟着改变。

解决方法：

```go
for _, stu := range stus{
    a := stu	//每次重新定义一个新的变量a
    m[stu.name] = &a
}
```

或者map存储student的实体，而不是指针，如map[string]student，m[stu.Name] = stu，把stu复制了一份到map中。



例子2：

在循环迭代器变量上使用 goroutine

```go
values := []int{1, 2, 3, 4, 5}
for _, val := range values {
 go func() {
  fmt.Println(val)
 }()
}

time.Sleep(time.Second)
```

输出结果如下：

```text
5
5
4
5
5
```

原因：在闭包执行的匿名函数中，val的值由于在匿名函数中找不到，就去到函数外面找，由于外层的val值是切片values的一个临时变量，在for...range...执行时，不断被赋予新的值，所以，匿名函数中的val也会跟着改变。

解决方法：

```go
values := []int{1, 2, 3, 4, 5}
 for _, val := range values {
  go func(val int) {	//通过把val传递进去，相当于拷贝了一个新的val给匿名函数使用
   fmt.Println(val)
  }(val)
 }
```



7、常量

`iota`是go语言的常量计数器，只能在常量的表达式中使用。

**`iota`在const关键字出现时将被重置为0。const中每新增一行常量声明将使`iota`计数一次(iota可理解为const语句块中的行索引)。 **

```go
const (
	a1 = iota //0
	a2        //1
	a3        //2
)

//使用_跳过某些值
const (
	b1 = iota //0
	b2        //1
	_         //该值被丢弃
	b4        //3
)

//iota声明中间插队
const (
	c1 = iota //0
	c2 = 100  //100
	c3 = iota //2
	c4        //3
)
const n5 = iota //0

//多个iota定义在一行
const (
	d1, d2 = iota + 1, iota + 2 //1,2
	d3, d4 = iota + 1, iota + 2 //2,3
)

const (
 bit0, mask0 = 1 << iota, 1<<iota - 1	//1,0
 bit1, mask1                            //2,1
 _, _                                   
 bit3, mask3                            //8,7
)
```



8、float不支持按位或操作

```go
func main() {
 var a, b = 1.0, 2.0
 fmt.Println(a | b)
}
```

float不支持按位或操作，编译不通过



9、

```go
var nums1 []interface{}
nums2 := []int{1, 2, 3}
nums3 := append(nums1, nums2)
fmt.Printf("%T\n", nums3)
fmt.Println(len(nums3))
```

打印结果：

```text
[]interface {}
1
```

原因：使用append，会把nums2当成一个空接口类型加到空接口切片nums1中。



拓展：把apend这句改成一下

```go
nums3 := append(nums3, nums2...)
```

会报错：

```text
cannot use nums2 (variable of type []int) as type []interface{} in argument to append
```

原因：`nums3 := append(nums3, nums2...)`本质是把`[]int`类型转换成`[]interface{}`类型，go不支持这样的转换。



10、

```text
A. var x = nil
B. var x interface{} = nil
C. var x string = nil
D. var x error = nil
```

nil 只能赋值给指针、chan、func、interface、map 或 slice 类型的变量



11、

```go
 m := [...]int{
  'a': 1,
  'b': 2,
  'c': 3,
 }
 m['a'] = 3
 fmt.Println(len(m))
```

输出：

```text
100
```

原因：定义并赋值一个int数组，使用`...`会根据初始值的个数自行推断数组的长度，而字符a、b、c对应的ASCII的值为97,98,99，所以数组m的最大值为m['c']，即m[99]。



12、

1. 没有支持前缀自增自减的运算语句，也就是不允许 ++a。
2. 运算符 ++ 和 -- 只能作为一个语句来使用，不可以作为表达式被赋值给其它的变量使用。例如：`b := a++`是不允许的。



13、

go中所有的函数传参都是值传递，对于基础值类型在传参中使用深拷贝，实际上对于**指针、slice(切片)、map(映射)、channel(管道)、函数、接口**，它们都是引用类型（如切片底层也是一个指针），使用的是浅拷贝，即指针拷贝了一个副本，指向的内存地址依然是原数据。



14、使用channel实现互斥锁

```go
type Mutex struct {
	ch chan struct{}
}

func NewMutex() *Mutex {
	return &Mutex{make(chan struct{}, 1)}
}

func (m *Mutex) lock() {
	m.ch <- struct{}{}
}

func (m *Mutex) unlock() {
	<-m.ch
}

func (m *Mutex) tryLock() bool {
	select {
	case m.ch <- struct{}{}:
		return true
	default:
	}
	return false
}

func (m *Mutex) lockTimeOut(timeout time.Duration) bool {
	timer := time.NewTimer(timeout)
	select {
	case m.ch <- struct{}{}:
		return true
	case <-timer.C: // 定时时间走完，会从通道C输出内容
		// default:
	}
	return false
}
```



15、

go中使用`map[string]struct{}`来模拟一个C++的`set`



16、

**全局变量的作用域是整个包，局部变量的作用域是该变量所在的花括号内**

```go
var t int
func init() {
    t, err := strconv.Atoi("2")
    if err != nil {
        log.Fatalln(err)
    }
    fmt.Println("init:", t)
}
func main() {
    fmt.Println("main:", t)
}
```

输出：

```text
init: 2
main: 0
```

原因：init中的t是用:=生成的，所以t是局部变量，在init函数中覆盖了全局变量t。全局变量t并没有被赋值，它还是原来的0值。

修改为：

```go
var t int
func init() {
    var err error
    t, err = strconv.Atoi("2")
    if err != nil {
        log.Fatalln(err)
    }
    fmt.Println("init:", t)
}
func main() {
    fmt.Println("main:", t)
}
```



17、

实现接口时有下面的约束：

- 如果定义的是 (Type)Method，则该类型会隐式的声明一个 (*Type)Method；
- 如果定义的是 (*Type)Method ，则不会隐式什么一个 (Type)Method。

https://mp.weixin.qq.com/s/_duDs0oHc_z_p--3OoIfVw



18、

在使用copy函数时，只要注意两个切片中的最小长度就行了。copy函数只会将min(len(dstslice), len(srcslice))个元素拷贝到dstslice中，有时候会是0个，所以这个要特别注意的。



19、

```go
var y = 5.2
const z = 2
fmt.Println(y / z) // 常量a转换为浮点型，输出2.6
```

```go
const a = 7.0
var b = 2
fmt.Println(a / b) // 常量a转换为整型，输出3
```

```go
const (
    x = 5.0
    y = 4
)
fmt.Println(x / y) // y转换为浮点型，输出1.25
```

```go
a := 40
f := float64(a/100.0)
fmt.Println(f) // 输出0
```

- 对于变量而言，如果没有显示指定数据类型，编译器会根据赋值自动推导出确定的数据类型，**变量不会根据上下文转换其类型**。整数的默认类型是`int`，浮点数的默认类型是`float64`

- 对于常量而言，**如果没有显示指定数据类型**，编译器同样会推导出一个数据类型，**但是没有显示指定数据类型的常量在代码上下文里可以根据需要隐式转化为需要的数据类型进行计算**。

- Go不允许不同的数据类型做运算。当变量和**没有显示指定数据类型的常量**混合在一起运算时，如果常量转化成变量的类型不会损失精度，那常量会自动转化为变量的数据类型参与运算。如果常量转化成变量的类型会损失精度，那就会编译报错。