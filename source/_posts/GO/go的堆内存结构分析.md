---
title: go的堆内存结构分析
tags: [go]      #添加的标签
categories: 
  - GO
description: 本文分析go中堆内存的结构
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/391559.png
---

## 操作系统虚拟内存

- × 不是win的“虚拟内存”（内存不够的时候拿硬盘做虚拟内存）
- √ 操作系统给应用提供的虚拟内存空间
  - 系统会给每个进程一个虚拟的内存空间，而不是直接的物理内存，操作系统管理这些虚拟内存空间映射到物理内存空间
  - 背后是物理内存，也有可能有磁盘
- Linux获取虚拟内存：mmap、madvice

### Linux（64位）

下面是以一台64位物理机，64GB内存，展示了 进程和物理内存之间隔着一个虚拟内存

> 若虚拟内存超过物理内存(64GB)就是内存溢出（OOM），操作系统会杀掉进程

![进程和物理内存的示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E8%BF%9B%E7%A8%8B%E5%92%8C%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E7%9A%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)



### go中虚拟内存是怎么获取的？

有上图可以知道每个进程有独立的虚拟内存，那么现在在go程序中虚拟内存是怎么获取的？
是通过一个一个变量获取还是一批批获取的，答案是一批批获取的，在go中有个这样一个结构体 `heapArena`



### heapArena结构体

- 在64位操作系统中(win除外) Go 每次申请的虚拟内存单元为64MB（以heapArena为单元申请，一次64MB，释放也是一次64MB）
- 最多有4,194,304个虚拟内存单元（2^20，刚好可以占满256TB）
- 所有的heapArena组成了mheap（Go堆内存）
- 当heapArena空间不足时，向操作系统申请新的heapArena

mheap 与 heapArena 关系示意图：

![mheap 与 heapArena 关系示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/mheap%20%E4%B8%8E%20heapArena%20%E5%85%B3%E7%B3%BB%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

> go heap 会按照arena的大小增长，每次预留arena大小整数倍的虚拟地址空间。arena的大小与平台相关，除了windows，其他系统64位的平台下arena的大小都是64M。在32位的平台中，为了使go heap比较连续，没有碎片，当程序启动的时候就会先预留一大块虚拟地址空间，如果这些空间都被用完了，才会每次按照arena大小整数倍去预留虚拟地址空间。

```
       Platform  Addr bits  Arena size  L1 entries   L2 entries
 --------------  ---------  ----------  ----------  -----------
       */64-bit         48        64MB           1    4M (32MB)
 windows/64-bit         48         4MB          64    1M  (8MB)
       */32-bit         32         4MB           1  1024  (4KB)
     */mips(le)         31         4MB           1   512  (2KB)
```

./src/runtime/mheap.go:229

```go
// 62行，mheap
type mheap struct { // 这个就是golang的堆内存
    // ...
    // ↓ ↓ ↓ ↓ 157行 ↓ ↓ ↓ ↓
    arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena // 记录向操作系统申请的所有内存单元
    // ...
}

// 229行，这个结构体描述了一个64MB的内存单元（不是一个结构体64MB），记录向操作系统申请64MB虚拟内存的信息
// bitmap、pageMarks、pageSpecials都与GC有关
type heapArena struct {
    bitmap [heapArenaBitmapBytes]byte // 用于记录这个arena中有哪些位置有指针
    spans [pagesPerArena]*mspan  // 内存管理单元
    pageInUse [pagesPerArena / 8]uint8
    pageMarks [pagesPerArena / 8]uint8
    pageSpecials [pagesPerArena / 8]uint8
    checkmarks *checkmarksMap
    zeroedBase uintptr
}
```

### 内存管理单元

在go中虚拟内存是以heapArena 结构体方式 对应，那么每一块这样的内存在go中是如何使用的呢，这里有三种方式

- 线性分配
- 链表分配
- 分级分配

#### 分级分配

线性分配或者链表分配容易出现空间碎片

分级分配的思想是，有如下几步

1. 先把大的内存拿过来后分成很多小块，相同大小的块属于一个组，叫做mspan
2. 将对象放入能放进去的最小箱子
3. 回收对象后，下一次有对象来了，就直接放入空闲的空间里面

![go中内存分级分配示意图2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E5%86%85%E5%AD%98%E5%88%86%E7%BA%A7%E5%88%86%E9%85%8D%E7%A4%BA%E6%84%8F%E5%9B%BE2.png)

![go中内存分级分配示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E5%86%85%E5%AD%98%E5%88%86%E7%BA%A7%E5%88%86%E9%85%8D%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

#### 分级的思想中的级（内存管理单元 mspan）

``上面所述的“级”就是 “内存管理单元 mspan”``

- 根据隔离适应策略，使用内存时的最小单位为mspan
- 每个mspan为N个大小相同的“格子”
- Go中一共有67种mspan，根据需求创建不同级别的mspan

> class 0 比较特别，没有固定大小
> 源码详情：./src/runtime/sizeclasses.go

```
   级别    对象大小  格子的大小   对象数  页面尾部浪费   最大浪费
 class  bytes/obj  bytes/span  objects  tail waste  max waste
     1          8        8192     1024           0     87.50%
     2         16        8192      512           0     43.75%
     3         32        8192      256           0     46.88%
    ...
    10        128        8192       64           0     11.72%
    11        144        8192       56         128     11.82%
    ...
    37       1792       16384        9         256     15.57%
    38       2048        8192        4           0     12.45%
    39       2304       16384        7         256     12.46%
    ...
    66      28672       57344        2           0      4.91%
    67      32768       32768        1           0     12.50%
```

> 因为mspan管理内存的最小单位是页面，而页面的大小不一定是size class大小的倍数，这会导致一些内存被浪费
>
> 例如下图中一个mspan划分成若干个slot用于分配，但是mspan占用页面的大小不能被slot的大小整除，所以有一个tail waste

![go中内存mspan分配示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E5%86%85%E5%AD%98mspan%E5%88%86%E9%85%8D%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

> mspan：./src/runtime/mheap.go:384

```go
// 很明显是一个链表
type mspan struct {
  next *mspan // next span in list, or nil if none
  prev *mspan // previous span in list, or nil if none
  list *mSpanList // For debugging. TODO: Remove.
}
```

``每个heapArena中的mspan都不确定，如何快速找到所需的mspan级别？``

### 中心索引 mcentral

- 136个mcentral结构体
  - 68个组需要GC扫描的mspan（堆中的对象）
  - 68个组不需要GC扫描的mspan（常量）
- mcentral 就是个链表头，保存了同样级别的所有mspan

![go中内存管理中心索引关系图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B8%AD%E5%BF%83%E7%B4%A2%E5%BC%95%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

#### 代码

> ./src/runtime/mheap.go:207

```go
type mheap struct {
    // ...
    // ↓ ↓ ↓ ↓ 207行 ↓ ↓ ↓ ↓
    central [numSpanClasses]struct { // numSpanClasses = 68 << 1 = 136
        mcentral mcentral
        pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
    }
    // ...
}
```

> ./src/runtime/mcentral.go:20

```go
type mcentral struct {
    spanclass spanClass // uint8 隔离级别
    partial [2]spanSet // 空闲的   A spanSet is a set of *mspans.
    full    [2]spanSet // 已满的
}
```

### 线程缓存 mcache

- mcentral 实际是中心索引，使用互斥锁保护
  - mcentral.go:119的cacheSpan()方法里调用了tryAcquire(s)方法，底层是通过atomic加锁实现
  - 在高并并发场景下，锁竞争问题严重？
- 参考协程GMP模型（P的本地队列），增加线程本地缓存
  - 不需要全局的协程列表获取线程，在本地就可以获取
  - 线程缓存mcache
    - 每个P拥有一个mcache
    - 一个mcache拥有136个mspan
      - mcentral中每种（级别、GC扫描类型）span取一个组成mcache分配给线程
      - 68个需要GC扫描的mspan
      - 68个不需要GC扫描的mspan

![go中线程缓存和架构之间的关系](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E7%BA%BF%E7%A8%8B%E7%BC%93%E5%AD%98%E5%92%8C%E6%9E%B6%E6%9E%84%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png)

> ./src/runtime/runtime2.go:614行可以看到P中的mcache

```go
type p struct {
    // ...
    mcache      *mcache // 在线程执行的时候需分配内存（变量、常量等）就直接往这里写，写满后会进行全局交换
    // ...
}
```

> ./src/runtime/mcache.go:44 可以看到mcache中有136个span

```go
type mcache struct {
  nextSample uintptr
  scanAlloc  uintptr
  tiny       uintptr
  tinyoffset uintptr
  tinyAllocs uintptr
  alloc [numSpanClasses]*mspan  // numSpanClasses = 68 << 1 = 136
  stackcache [_NumStackOrders]stackfreelist
  flushGen uint32
}
```

### 总结

- Go模仿TCmalloc，建立了自己的堆内存架构（c++用的,google开发go的时候直接拿过来了）
- 使用heapArena向操作系统申请内存
- 使用heapArena时，以mspan为单位（有一堆），防止碎片化
- mcentral是mspan们的中心索引（不用遍历heapArena，遍历mcentral即可，都分好类了，但是会有锁的并发问题）
- mcache记录了分配给每个P的本地mspan

## 堆内存分配

### 对象级别

> 微、小对象分配至普通 mspan（class 1~67）
> 大对象量身定制 mspan （class 0 无固定大小）

go分配堆内存前，会按照对象的大小进行不同的分配，那么对象的大小是如何定义的呢，下面是针对对象大小的一个定义：

- Tiny微对象 (0,16B) 无指针
- Small小对象 \[16B,32KB]
- Large 大对象 (32KB,+∞)
- 微小对象（32KB以下的）分配 到普通mspan （1-67级span）
- 大对象 量身定做mspan（0级span）（0级span 是没有固定大小的）



### 微对象分配

1. 从mcache 拿到2级mspan
2. 将多个微对象合并一个16Byte 存入

![go中堆分配（微对象分配）](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E5%A0%86%E5%88%86%E9%85%8D%EF%BC%88%E5%BE%AE%E5%AF%B9%E8%B1%A1%E5%88%86%E9%85%8D%EF%BC%89.png)

---

#### 代码

> ./src/runtime/malloc.go:903

``可以推论 class 1 的 span 在当前Go版本用不到``

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    // ...
    // ↓ ↓ ↓ ↓ ↓ 991行 ↓ ↓ ↓ ↓ ↓
    if size <= maxSmallSize { // 先判断是否是微、小对象（小于32KB）
        if noscan && size < maxTinySize { // 判断是否是微对象（小于16B）
            // 注释1001行注释说明是组合成一个16B (bytes)
            off := c.tinyoffset
            if size&7 == 0 {
                off = alignUp(off, 8)
            } else if sys.PtrSize == 4 && size == 12 {
                off = alignUp(off, 8)
            } else if size&3 == 0 {
                off = alignUp(off, 4)
            } else if size&1 == 0 {
                off = alignUp(off, 2)
            }
            if off+size <= maxTinySize && c.tiny != 0 {
                x = unsafe.Pointer(c.tiny + off)
                c.tinyoffset = off + size
                c.tinyAllocs++
                mp.mallocing = 0
                releasem(mp)
                return x
            }
            span = c.alloc[tinySpanClass] // 这里拿的是 class 2 的 span
            v := nextFreeFast(span)
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(tinySpanClass)
            }
            x = unsafe.Pointer(v)
            (*[2]uint64)(x)[0] = 0
            (*[2]uint64)(x)[1] = 0
            if !raceenabled && (size < c.tinyoffset || c.tiny == 0) {
                c.tiny = uintptr(x)
                c.tinyoffset = size
            }
            size = maxTinySize
        } else { // 这里是小对象（16B~32KB）
            var sizeclass uint8
            // 通过查表确定使用几级的span
            if size <= smallSizeMax-8 {
                sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
            } else {
                sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
            }
            size = uintptr(class_to_size[sizeclass])
            spc := makeSpanClass(sizeclass, noscan)
            span = c.alloc[spc]
            v := nextFreeFast(span) // 找到没被占用的span中的小格子（obj）
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(spc) // 若没找到，则进行mcache替换
            }
            x = unsafe.Pointer(v)
            if needzero && span.needzero != 0 {
                memclrNoHeapPointers(unsafe.Pointer(v), size)
            }
        }
    } else {
        shouldhelpgc = true
        span, isZeroed = c.allocLarge(size, needzero && !noscan, noscan)
        span.freeindex = 1
        span.allocCount = 1
        x = unsafe.Pointer(span.base())
        size = span.elemsize
    }
    // ...
}
```

---

> ./src/runtime/malloc.go:876

```go
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
    s = c.alloc[spc]
    shouldhelpgc = false
    freeIndex := s.nextFreeIndex()
    if freeIndex == s.nelems {
        // The span is full.
        if uintptr(s.allocCount) != s.nelems {
            println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
            throw("s.allocCount != s.nelems && freeIndex == s.nelems")
        }
        c.refill(spc) // 在这里进行mcache替换
        shouldhelpgc = true
        s = c.alloc[spc]

        freeIndex = s.nextFreeIndex()
    }

    if freeIndex >= s.nelems {
        throw("freeIndex is not valid")
    }

    v = gclinkptr(freeIndex*s.elemsize + s.base())
    s.allocCount++
    if uintptr(s.allocCount) > s.nelems {
        println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
        throw("s.allocCount > s.nelems")
    }
    return
}
```

---

> ./src/runtime/mcache.go:146

```go
func (c *mcache) refill(spc spanClass) {
    s := c.alloc[spc]
    if uintptr(s.allocCount) != s.nelems {
        throw("refill of span with free space remaining")
    }
    if s != &emptymspan {
    // Mark this span as no longer cached.
        if s.sweepgen != mheap_.sweepgen+3 {
            throw("bad sweepgen in refill")
        }
        mheap_.central[spc].mcentral.uncacheSpan(s) // 卸载mcache
    }
    // Get a new cached span from the central lists.
    s = mheap_.central[spc].mcentral.cacheSpan() // 从中心索引装载mcache
    if s == nil {
        throw("out of memory")
    }
    // ...
}
```

#### mchache的替换

- mcache中，每个级别的mspan（根据隔离级别表格，有不同的对象(格子)数）只有一个
- 当mspan满了之后，会中mcentral中换一个新的
- 若mcentral中所有的span都满了，会进行扩容
  - mcentral中，只有有限数量的mspan
  - 当mspan缺少时，会从虚拟内存中申请更多(最多2^20)的heapArena（64MB）开辟新的mspan

### 大对象分配

- 直接从heapArena开辟0级mspan
- 0级的mspan为大对象定制（可大可小）
  - 67级最大的格子大小是32KB，

#### 代码

> ./src/runtime/malloc.go:903

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    // ...
    if size <= maxSmallSize {
        // ...
        // ↓ ↓ ↓ ↓ ↓ 1065行 ↓ ↓ ↓ ↓ ↓
    } else { // 这里是大对象
        shouldhelpgc = true
        span, isZeroed = c.allocLarge(size, needzero && !noscan, noscan) // 在这里定制
        span.freeindex = 1
        span.allocCount = 1
        x = unsafe.Pointer(span.base())
        size = span.elemsize
    }
    // ...
}
```

### 总结

- Go将对象分为3种，微(0,16B)、小\[16B,32KB]、大(32KB,+∞)
- 微、小对象使用mcache
  - mcache中的mspan装满后，与mcentral交换新的mcache（这里才有中心索引的锁竞争）
  - mcentral不足时，在heapArena开辟新的mspan
- 大对象直接在heapArena开辟新的mspan