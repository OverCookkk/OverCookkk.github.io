---
title: go-channel
tags: [go]      #添加的标签
categories: 
  - GO
description: 本文介绍关于channel和select的特性。
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

channel收发数据的机制如下表所示：

| channel为以下四种状态时的操作 | 无缓冲区 | 已关闭                                     | nil   | 空的     | 非空非满 | 满了     |
| ----------------------------- | -------- | ------------------------------------------ | ----- | -------- | -------- | -------- |
| 往channel发数据               | 阻塞     | panic                                      | 阻塞  | 发送成功 | 发送成功 | 阻塞     |
| 从channel读数据               | 阻塞     | 先读完原有数据，再读到存储的元素类型的零值 | 阻塞  | 阻塞     | 接收成功 | 接收成功 |
| 关闭channel                   | 关闭成功 | painc                                      | panic | 关闭成功 | 关闭成功 | 关闭成功 |

-  管道没有缓冲区，从管道读数据会阻塞，直到有协程向管道中写入数据。同样，向管道写入数据也会阻塞，直到有协程从管道读取数据



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
    qcount   uint           // 当前 channel 中存在多少个元素；
    dataqsiz uint           // 当前 channel 能存放的元素容量；
    buf      unsafe.Pointer // channel 中用于存放元素的环形缓冲区（环形数组）；
    elemsize uint16	// channel 元素类型的大小；
    closed   uint32	// 标识 channel 是否关闭；
    elemtype *_type // channel 元素类型；
    sendx    uint   // 发送元素进入环形缓冲区的 index；
    recvx    uint   // 接收元素所处的环形缓冲区的 index；
    recvq    waitq  // 因接收而陷入阻塞的协程队列；
    sendq    waitq  // 因发送而陷入阻塞的协程队列；

// lock protects all fields in hchan, as well as several
// fields in sudogs blocked on this channel.
//
// Do not change another G's status while holding this lock
// (in particular, do not ready a G), as this can deadlock
// with stack shrinking.
    lock mutex
}

// 阻塞的协程队列
type waitq struct {
    first *sudog	// 队列头部
    last  *sudog	// 队列尾部
}

// 用于包装协程的节点
type sudog struct {
    g *g	// 协程

    next *sudog // 队列中的下一个节点
    prev *sudog	// 队列中的前一个节点
    elem unsafe.Pointer // data element (may point to stack) ，读取/写入 channel 的数据的容器
    // ...
    c        *hchan // 标识与当前 sudog 交互的 chan
}
```

通过`make`函数来创建`channel`时，Go会调用运行时的`makechan`函数。

从上面的代码可以看出`makechan`返回的是指向`channel`的指针。

因此`channel`作为函数参数时，实参`channel`和形参`channel`都指向同一个`channel`结构体的内存空间，所以在函数内部对`channel`形参的修改对外部`channel`实参是可见的，反之亦然。

![go-channel底层结构图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go-channel%E5%BA%95%E5%B1%82%E7%BB%93%E6%9E%84%E5%9B%BE.png)

**向 channel 写数据的流程：** 如果等待接收队列 recvq 不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从 recvq 取出 G,并把数据写入，最后把该 G 唤醒，结束发送过程； 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程； 如果缓冲区中没有空余位置，将待发送数据写入 G，将当前 G 加入 sendq，进入睡眠，等待被读 goroutine 唤醒；

**向 channel 读数据的流程：** 如果等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G，把 G 中数据读出，最后把 G 唤醒，结束读取过程； 如果等待发送队列 sendq 不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把 G 中数据写入缓冲区尾部，把 G 唤醒，结束读取过程； 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；将当前 goroutine 加入 recvq，进入睡眠，等待被写 goroutine 唤醒；

**读写流程总结：先从缓冲区buf中读写数据，再考虑是否从recvq或者sendq队列读写数据**



**使用场景：** 消息传递、消息过滤，信号广播，事件订阅与广播，请求、响应转发，任务分发，结果汇总，并发控制，限流，同步与异步