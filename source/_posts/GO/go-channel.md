---
title: go-channel
tags: [go]      #添加的标签
categories: 
  - GO
description: 本文介绍关于关于channel和select的特性。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/01204-2118699694.png
---

通过一掉题目，抛砖引玉，了解一下`channel`和`select`的作用：

```go
func main() {
  c := make(chan int, 1)
  for done := false; !done; {
    select {
    default:
      print(1)
      done = true
    case <-c:
      print(2)
      c = nil
    case c <- 1:
      print(3)
    }
  }
}
```

结果输出为`321`



## channel收发数据的机制

1. 对于无缓冲区的`channel`，往`channel`发送数据和从`channel`接收数据都会阻塞。

2. 对于`nil channel`和有缓冲区的`channel`，收发数据的机制如下表所示：

    | channel为以下四种状态时的操作 | nil   | 空的     | 非空非满 | 满了     |
    | ----------------------------- | ----- | -------- | -------- | -------- |
    | 往channel发送数据             | 阻塞  | 发送成功 | 发送成功 | 阻塞     |
    | 从channel接收数据             | 阻塞  | 阻塞     | 接收成功 | 接收成功 |
    | 关闭channel                   | panic | 关闭成功 | 关闭成功 | 关闭成功 |

3. `channel`被关闭后：

   - 往被关闭的`channel`发送数据会触发panic。

   - 从被关闭的`channel`接收数据，会先读完`channel`里的数据。如果数据读完了，继续从`channel`读数据会拿到`channel`里存储的元素类型的零值。

     ```go
     data, ok := <- c 
     ```

     对于上面的代码，如果channel `c`关闭了，继续从`c`里读数据，当`c`里还有数据时，`data`就是对应读到的值，`ok`的值是`true`。如果`c`的数据已经读完了，那`data`就是零值，`ok`的值是`false`。

   - `channel`被关闭后，如果再次关闭，会引发panic。



## `select`的运行机制

- 选取一个可执行不阻塞的`case`分支，如果多个`case`分支都不阻塞，会随机算一个`case`分支执行，和`case`分支在代码里写的顺序没关系。
- 如果所有`case`分支都阻塞，会进入`default`分支执行。
- 如果没有`default`分支，那`select`会阻塞，直到有一个`case`分支不阻塞。





## channel底层

`channel`源码如下：

```go
func makechan(t *chantype, size int) *hchan

// channel结构体
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

// lock protects all fields in hchan, as well as several
// fields in sudogs blocked on this channel.
//
// Do not change another G's status while holding this lock
// (in particular, do not ready a G), as this can deadlock
// with stack shrinking.
    lock mutex
}
```

通过`make`函数来创建`channel`时，Go会调用运行时的`makechan`函数。

从上面的代码可以看出`makechan`返回的是指向`channel`的指针。

因此`channel`作为函数参数时，实参`channel`和形参`channel`都指向同一个`channel`结构体的内存空间，所以在函数内部对`channel`形参的修改对外部`channel`实参是可见的，反之亦然。