---
title: Redis高可用篇：主从同步原理
tags: [Redis,高可用]      #添加的标签
categories: #添加的分类
  - 中间件
  - Redis
description: 主从同步实现redis的高可用性。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00712-2369818868.png
---



## 背景

Redis具有高可用行，高可用有两层含义：一是数据尽量少丢失，二是服务尽量少中断。AOF和RDB保证了前者，对于后者，redis的做法就是增加副本冗余量，将一份数据同时保存在多个实例上，即使有一个实例出现了故障，需要一段时间才能恢复，其他的实例也可以对外提供服务，不会影响业务使用。



## 原理

redis提供了主从库模式，以保证数据副本的一致，主从库之间采用的就是**读写分离**的方式。

- 读操作：主库、从库都可以接收；
- 写操作：首先到主库执行，然后，主库将写操作同步给从库。

![redis主从复制的读写分离](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/redis%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E7%9A%84%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB.png)

如果主从库都能接收客户端的写操作，就会导致主从库的数据不一致，除非使用加锁等操作，但是加锁会带来巨额的开销。