---
title: Redis唯快不破的秘密
tags: [Redis]      #添加的标签
categories: #添加的分类
  - 中间件
  - Redis
description: 
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00723-77064873.png
---



## 前言

根据官方数据，Redis 的 QPS 可以达到约 100000（每秒请求数），有兴趣的可以参考官方的基准程序测试《How fast is Redis？》，地址：https://redis.io/topics/benchmarks



## 完全基于内存实现

内存直接由 CPU 控制，也就是 CPU 内部集成的内存控制器，所以说内存是直接与 CPU 对接，享受与 CPU 通信的最优带宽。**Redis 将数据存储在内存中，读写操作不会因为磁盘的 IO 速度限制**。



## 高效的数据结构

MySQL 为了提高检索速度使用了 B+ Tree 数据结构，所以 Redis 速度快应该也跟数据结构有关。Redis 提供给我们使用的 5 种数据类型：String、List、Hash、Set、SortedSet。

在 Redis 中，常用的 5 种数据类型和应用场景如下：

- **String：** 缓存、计数器、分布式锁等。
- **List：** 链表、队列、微博关注人时间轴列表等。
- **Hash：** 用户信息、Hash 表等。
- **Set：** 去重、赞、踩、共同好友等。
- **Zset：** 访问量排行榜、点击量排行榜等。

当然是为了追求速度，不同数据类型使用不同的数据结构速度才得以提升。每种数据类型都有一种或者多种数据结构来支撑，底层数据结构有 6 种。

![Redis数据类型与底层数据结构关系](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E4%B8%8E%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%85%B3%E7%B3%BB.png)

### Redis hash 字典

Redis 整体就是一个哈希表来保存所有的键值对，无论数据类型是 5 种的任意一种。哈希表，本质就是一个数组，每个元素被叫做哈希桶，不管什么数据类型，每个桶里面的 entry 保存着实际具体值的指针。

![Redis全局哈希表](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/Redis%E5%85%A8%E5%B1%80%E5%93%88%E5%B8%8C%E8%A1%A8.png)

整个数据库就是一个**全局哈希表**，而哈希表的时间复杂度是 O(1)，只需要计算每个键的哈希值，便知道对应的哈希桶位置，定位桶里面的 entry 找到对应数据，这个也是 Redis 快的原因之一。

随着写入 Redis 的数据越来越多的时候，哈希冲突不可避免，会出现不同的 key 计算出一样的哈希值。

Redis 通过**链式哈希**解决冲突：**也就是同一个 桶里面的元素使用链表保存**。但是当链表过长就会导致查找性能变差可能，所以 Redis 为了追求快，使用了两个全局哈希表。用于 rehash 操作，增加现有的哈希桶数量，减少哈希冲突。

开始默认使用 hash 表 1 保存键值对数据，哈希表 2 此刻没有分配空间。当数据越来多触发 rehash 操作，则执行以下操作：

1. 给 hash 表 2 分配更大的空间；
2. 将 hash 表 1 的数据重新映射拷贝到 hash 表 2 中；
3. 释放 hash 表 1 的空间。

**值得注意的是，将 hash 表 1 的数据重新映射到 hash 表 2 的过程中并不是一次性的，这样会造成 Redis 阻塞，无法提供服务。**

而是采用了**渐进式 rehash**，每次处理客户端请求的时候，先从 hash 表 1 中第一个索引开始，将这个位置的 所有数据拷贝到 hash 表 2 中，就这样将 rehash 分散到多次请求过程中，避免耗时阻塞。



### SDS 简单动态字符

字符串结构使用最广泛，**通常我们用于缓存登陆后的用户信息**，key = userId，value = 用户信息 JSON 序列化成字符串。

C 语言中字符串的获取 「MageByte」的长度，要从头开始遍历，直到 「\0」为止，对于redis来说，这样是不能容忍的。

C 语言字符串结构与 SDS 字符串结构对比图如下所示：

![C 语言字符串与SDS](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/C%20%E8%AF%AD%E8%A8%80%E5%AD%97%E7%AC%A6%E4%B8%B2%E4%B8%8ESDS.png)

两者的区别：

1. SDS有着**O(1) 时间复杂度获取字符串长度**
2. **空间预分配**：SDS 被修改后，程序不仅会为 SDS 分配所需要的必须空间，还会分配额外的未使用空间。分配规则如下：如果对 SDS 修改后，len 的长度小于 1M，那么程序将分配和 len 相同长度的未使用空间。如果对 SDS 修改后 len 长度大于 1M，那么程序将分配 1M 的未使用空间。
3. **惰性空间释放**：当对 SDS 进行缩短操作时，程序并不会回收多余的内存空间，而是使用 free 字段将这些字节数量记录下来不释放，后面如果需要 append 操作，则直接使用 free 中未使用的空间，减少了内存的分配。
4. **二进制安全**：在 Redis 中不仅可以存储 String 类型的数据，也可能存储一些二进制数据。二进制数据并不是规则的字符串格式，其中会包含一些特殊的字符如 '\0'，在 C 中遇到 '\0' 则表示字符串的结束，但在 SDS 中，标志字符串结束的是 len 属性。



### zipList 压缩列表

当一个列表只有少量数据的时候，那么 Redis 就会使用压缩列表来做列表键的底层实现。

ziplist 是由一系列特殊编码的连续内存块组成的顺序型的数据结构，ziplist 中可以包含多个 entry 节点，每个节点可以存放整数或者字符串。

```c
struct ziplist<T> {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}
```

![redis的ziplist](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/redis%E7%9A%84ziplist.png)

如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N)



### linkedlis双端列表

Redis List 数据类型通常被用于队列、微博关注人时间轴列表等场景。不管是先进先出的队列，还是先进后出的栈，双端列表都很好的支持这些特性。

后续版本对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist。

**quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。**

![redis的quicklist](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/redis%E7%9A%84quicklist.png)



### skipList 跳跃表

sorted set 类型的排序功能便是通过「跳跃列表」数据结构来实现。

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到*快速访问节点*的目的。

跳跃表支持平均 O（logN）、最坏 O（N）复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

跳表在链表的基础上，增加了多层级索引，通过索引位置的几个跳转，实现数据的快速定位，如下图所示：

![跳跃表](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E8%B7%B3%E8%B7%83%E8%A1%A8.png)

当需要查找 40 这个元素需要经历三次查找。



## 单线程模型

**我们要明确的是：Redis 的单线程指的是 Redis 的网络 IO 以及键值对指令读写是由一个线程来执行的。** 对于 Redis 的持久化、集群数据同步、异步删除等都是其他线程执行。

在运行每个任务之前，CPU 需要知道任务在何处加载并开始运行。也就是说，系统需要帮助它预先设置 CPU 寄存器和程序计数器，这称为 CPU 上下文。

这些保存的上下文存储在系统内核中，并在重新计划任务时再次加载。这样，任务的原始状态将不会受到影响，并且该任务将看起来正在连续运行。

**切换上下文时，我们需要完成一系列工作，这是非常消耗资源的操作。**

### 单线程又什么好处？

1. 不会因为线程创建导致的性能消耗；
2. 避免上下文切换引起的 CPU 消耗，没有多线程切换的开销；
3. 避免了线程之间的竞争问题，比如添加锁、释放锁、死锁等，不需要考虑各种锁问题。
4. 代码更清晰，处理逻辑简单。

因为 Redis 是基于内存的操作，CPU 不是 Redis 的瓶颈，Redis 的瓶颈最**有可能是机器内存的大小或者网络带宽**。



## I/O 多路复用模型

Redis 采用 I/O 多路复用技术，并发处理连接。采用了 epoll + 自己实现的简单的事件框架。epoll 中的读、写、关闭、连接都转化成了事件，然后利用 epoll 的多路复用特性，绝不在 IO 上浪费一点时间。