---
title: go-runtime
tags: [go]      #添加的标签
categories: 
  - GO
description: golang 为什么效率高，goroutine 的是怎么执行的？golang runtime 是什么？本文将会对 golang runtime 进行简析。
cover: 
---

golang 为什么效率高，goroutine 的是怎么执行的？golang runtime 是什么？本文将会对 golang runtime 进行简析。



## runtime包的作用

golang 的 runtime 在 golang 中的地位类似于 Java 的虚拟机，不过 go runtime 不是虚拟机。golang 程序生成可执行文件在指定平台上即可运行，效率很高， 它和 c/c++ 一样编译出来的是二进制可执行文件。我们知道运行 golang 的程序并不需要主机安装有类似 Java 虚拟机之类的东西，那是因为在编译时，golang 会将 runtime 部分代码链接进去。

golang 的 runtime 核心功能包括以下内容：

1. **协程(goroutine)调度(并发调度模型)**
2. **垃圾回收(GC)**
3. **内存分配**
4. 使得 golang 可以支持如 pprof、trace、race 的检测
5. 支持 golang 的内置类型 channel、map、slice、string等的实现
6. 等等

下图是 golang 程序、runtime、可执行文件与操作系统之间的关系。区别于 Java 需要安装虚拟机，go 语言的可执行文件已经包含了 golang 的 runtime，它为用户的 go 程序提供协程调度、内存分配、垃圾回收等功能。此外还会与系统内核进行交互，从而真正的利用好 CPU 等资源。

![golang 程序、runtime、可执行文件与操作系统之间的关系](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/golang%20%E7%A8%8B%E5%BA%8F%E3%80%81runtime%E3%80%81%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%E4%B8%8E%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png)