---
title: 实存(RES)与虚存(VIRT)分析
tags: [linux]      #添加的标签
categories: Linux
description: 在linux中使用top命令会看到实存(RES)与虚存(VIRT)等信息，他们分别代表着什么意思呢？
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/391060.jpeg
---



## 概念

![linux_top](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/linux_top.png)

VIRT：

```text
1、进程“需要的”虚拟内存大小，包括进程使用的库、代码、数据，以及malloc、new分配的堆空间和分配的栈空间等；
2、假如进程新申请10MB的内存，但实际只使用了1MB，那么它会增长10MB，而不是实际的1MB使用量。
3、VIRT = SWAP + RES
```



RES:

```text
1、进程当前使用的内存大小，包括使用中的malloc、new分配的堆空间和分配的栈空间，但不包括swap out量；
2、包含其他进程的共享；
3、如果申请10MB的内存，实际使用1MB，它只增长1MB，与VIRT相反；
4、关于库占用内存的情况，它只统计加载的库文件所占内存大小。
5、RES = CODE + DATA
```



SHR:

```text
1、除了自身进程的共享内存，也包括其他进程的共享内存；
2、虽然进程只使用了几个共享库的函数，但它包含了整个共享库的大小；
3、计算某个进程所占的物理内存大小公式：RES – SHR；
4、swap out后，它将会降下来。
```



## 总结

1. 堆、栈分配的内存，如果没有使用是不会占用实存的，只会记录到虚存。
2. 如果程序占用实存比较多，说明程序申请内存多，实际使用的空间也多。
3. 如果程序占用虚存比较多，说明程序申请来很多空间，但是没有使用。
