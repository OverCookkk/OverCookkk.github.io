---
title: go常用方法
tags: [go]      #添加的标签
categories: 
  - GO
description:
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00735-3981599422.png
---



## 删除切片中的重复元素

假设我们需要写一个函数实现用户ID切片元素去重。



### 方法1：Go安全方法

该方法在保持原切片不变的情况下很有用。创建一个新的切片只保存不同的元素。我们使用map的键来判别重复元素，**map的值为空结构体不占内存**。

```go
func RemoveDuplicates(userIDs []int64) []int64 {
    // 只需要检查key，因此value使用空结构体，不占内存
    processed := make(map[int64]struct{})	//使用map[int64]bool也可以，但是bool会占内存

    uniqUserIDs := make([]int64, 0)
    for _, uid := range userIDs {
        // 如果用户ID已经处理过，就跳过
        if _, ok := processed[uid]; ok {
            continue
        }

        // 将唯一的用户ID加到切片中
        uniqUserIDs = append(uniqUserIDs, uid)

        // 将用户ID标记为已存在
        processed[uid] = struct{}{}
    }
    return uniqUserIDs
}
```



### 方法2: 在原切片中移动元素

该方法能满足您的性能要求，并且可以修改原切片。首先对切片排序，然后通过两个迭代器：元素唯一迭代器和常规迭代器。通过一次遍历切片，返回一个包含不同元素的子切片。

```go
import (
    "sort"
)

func RemoveDuplicatesInPlace(userIDs []int64) []int64 {
    // 如果有0或1个元素，则返回切片本身。
    if len(userIDs) < 2 {
        return userIDs
    }

    //  使切片升序排序
    sort.SliceStable(userIDs, func(i, j int) bool { return userIDs[i] < userIDs[j] })

    uniqPointer := 0

    for i := 1; i < len(userIDs); i++ {
        // 比较当前元素和唯一指针指向的元素
        //  如果它们不相同，则将项写入唯一指针的右侧。
        if userIDs[uniqPointer] != userIDs[i] {
            uniqPointer++
            userIDs[uniqPointer] = userIDs[i]
        }
    }
    return userIDs[:uniqPointer+1]
}
```



## 反转切片元素

```go
func main() {
  a := []int{1, 2, 3, 4, 5, 6} // input int array
  reverseArray := ReverseSlice(a)
  fmt.Println("Reverted array : ", reverseArray) // print output
}

// 双数：先反转中间两个数，再由内向外反转； 单书：中间数不变，由内向外反转
func ReverseSlice(a []int) []int {
  for i := len(a)/2 - 1; i >= 0; i-- {
   pos := len(a) - 1 - i
   a[i], a[pos] = a[pos], a[i]
}
 return a
}
```



## 将 slice 转换为字符分隔的字符串

```go
func main() {
   result := ConvertSliceToString([]int{10, 20, 30, 40})
   fmt.Println("Slice converted string : ", result)
}


func ConvertSliceToString(input []int) string {
   var output []string
   for _, i := range input {
      output = append(output, strconv.Itoa(i))
   }
   return strings.Join(output, ",")
}
```

输出结果如下：

```text
output:
Slice converted string :  10,20,30,40
```

除此之外，strings.Join方法比普通的"str"+"str2"这种形式的字符串拼接效率更高，这是因为string本身就是一个常量，那拼接成一个新字符串，就必须要销毁原string对象，然后使当前引用指向新的字符串对象，这样做的开销是非常大的，而strings.Join则不用。



## 对map中的数据进行排序

map的底层是哈希表结构，类似C++的unordered_map，它里面的内容是无序的；需要借助切片来对map中的数据进行排序。

```go
map1 := make(map[int]int)
map1[10] = 100
map1[1] = 13
map1[4] = 110
map1[8] = 90

keys := make([]int, 0)
for k := range map1 {
    keys = append(keys, k)
}
fmt.Println(keys)
sort.Ints(keys) //使用sort库对切片的int进行排序
for _, v := range keys {
    fmt.Printf("after sort key:%d, value:%d\n", v, map1[v])
}
```



## 可变长形参

`Go`语言允许一个函数把任意数量的值作为参数，`Go`语言内置了`...`**操作符，在函数的最后一个形参才能使用**`...`操作符，使用它必须注意如下事项：

- 可变长参数必须在函数列表的最后一个；
- 把可变长参数当切片来解析，可变长参数没有值时就是`nil`切片
- 可变长参数的类型必须相同

```go
func test(a int, b ...int){
    return
}
```

在传参的时候也可以传递切片使用`...`进行解包转换为参数列表，`append`方法就是最好的例子：

```go
var s1 []int
s1 = append(s1, 1)
s1 = append(s1, s1...)
// func append(slice []Type, elems ...Type) []Type
```



## 声明不定长数组

使用`...`操作符声明数组时，只管填充元素值，其他的交给编译器自己去计算长度。

```go
a := [...]int{1, 2, 3}
```

