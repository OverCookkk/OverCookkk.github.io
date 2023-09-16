---
title: 用Redis的数据结构实现亿级数据统计
tags: [Redis,Bitmap]      #添加的标签
categories: #添加的分类
  - 中间件
  - Redis
description: redis不同数据结构的应用场景不同，本文将介绍面对不同的业务场景，选择合适的数据结构使得统计数据更加的高效。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00709-2393217881.png
---



## 业务场景

在移动应用的业务场景中，我们需要保存这样的信息：一个 key 关联了一个数据集合。

常见的场景如下：

- 给一个 userId ，判断用户登陆状态；
- 显示用户某个月的签到次数和首次签到时间；
- 两亿用户最近 7 天的签到情况，统计 7 天内连续签到的用户总数；
- 统计每天的新增与第二天的留存用户数；
- 统计网站的对访客（Unique Visitor，UV）量
- 最新评论列表
- 根据播放量音乐榜单

通常情况下，我们需要统计的数据过大，通常达到百万，甚至过亿，所以必须选择能高效地统计大量数据的集合类型。

**如何选择合适的数据集合，我们首先要了解常用的统计模式，并运用合理的数据类型来解决实际问题。**四种统计类型如下：

1. 二值状态统计；
2. 聚合统计；
3. 排序统计；
4. 基数统计。



## 二值状态统计

基于上述的业务场景，统计的数据，比如用户登录状态，只有两种状态（登录和非登录），分别对应的是1和0，所以适合使用Bitmap数据结构实现二值状态统计。比如登陆状态我们用一个 bit 位表示，一亿个用户也只占用 一亿 个 bit 位内存 ≈ （100000000 / 8/ 1024/1024）12 MB。

### Bitmap底层结构

Bitmap 的底层数据结构用的是 String 类型的 SDS 数据结构来保存位数组，Redis 把每个字节数组的 8 个 bit 位利用起来，每个 bit 位 表示一个元素的二值状态（不是 0 就是 1）。

可以将 Bitmap 看成是一个 bit 为单位的数组，数组的每个单元只能存储 0 或者 1，数组的下标在 Bitmap 中叫做 offset 偏移量。

![Bitmap底层结构图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/Bitmap%E5%BA%95%E5%B1%82%E7%BB%93%E6%9E%84%E5%9B%BE.jpg)



### 业务场景一：判断用户登录状态

Bitmap 提供了 `GETBIT、SETBIT` 操作，通过一个偏移值 offset 对 bit 数组的 offset 位置的 bit 位进行读写操作，需要注意的是 offset 从 0 开始。

```redis
setbit <key> <offset> <value>	//设置或者清空 key 的 value 在 offset 处的 bit 值（只能是 0 或者 1）。
getbit <key> <offset>	//获取 key 的 value 在 offset 处的 bit 位的值，当 key 不存在时，返回 0。
```

login_status作为登录的key，用户ID作为offset，登录状态用value表示，登录则调用`setbit login_status 10086 1`把用户10086的value设为1，检查该用户是否登录则调用`getbit login_status 10086`，判断返回值为多少。



### 业务场景二：用户每个月的签到情况

一个月最多有31天，最多只需要31个bit位，可以把Bitmap的key设计成`uid:sign:{userId}:{yyyyMM}`，月份的每一天的值 - 1可以作为offset（因为offset从0开始）。

如统计用户10086在2021年6月份的打卡情况，执行步骤如下：

第一步，执行`setbit uid:sign:10086:202106 16 1`，记录用户10086在2021年6月16日的打卡情况；

第二步，统计该用户在 6 月份的打卡次数，使用 `BITCOUNT` 指令。该指令用于统计给定的 bit 数组中，值 = 1 的 bit 位的数量，`bitcount uid:sign:10086:202106`。



再例如统计这个月首次打卡时间：

此外，Redis 提供了 `BITPOS key bitValue [start] [end]`指令，返回数据表示 Bitmap 中第一个值为 `bitValue` 的 offset 位置。在默认情况下， 命令将检测整个位图， 用户可以通过可选的 `start` 参数和 `end` 参数指定要检测的范围。

执行`bitpos uid:sign:10086:2021056 1`，得到用户10086在2021年6月首次打卡的时间，注意，这个事件需要加上1，因为offset是从0开始的。



### 业务场景三：连续签到用户总数

例如在记录了一个亿的用户连续 7 天的打卡数据，如何统计出这连续 7 天连续打卡用户总数呢？

思路分析：把每一天的打卡数据都设计成一个key，userid作为offset，key 对应的集合的每个 bit 位的数据则是一个用户在该日期的打卡记录。连续7天则表示，即有7个类型为Bitmap的key，同样的 UserID  offset 都是一样的，当一个 userID 在 7 个 Bitmap 对应的 offset 位置的 bit = 1 就说明该用户 7 天连续打卡。

Redis 提供了 `bittop operation destkey key [key ...]`这个指令用于对一个或者多个 键 = key 的 Bitmap 进行位元操作。`opration` 可以是 `and`、`OR`、`NOT`、`XOR`。当 BITOP 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 `0` 。空的 `key` 也被看作是包含 `0` 的字符串序列。`destkey`是进行位元操作后新生成的key。

第一步：执行`bittop and destkey 20210615 20210616 20210617 ....`把7个key进行按位与操作；

第二步：使用命令`bitcount destkey`统计destkey中1的个数，即为连续7天签到的用户数量。



## 聚合统计

指的就是统计多个集合元素的聚合结果，比如说：

- 统计多个元素的共有数据（交集）；
- 统计两个集合其中的一个独有元素（差集统计）；
- 统计多个集合的所有元素（并集统计）。

Redis 的 Set 类型支持集合内的增删改查，底层使用了 Hash 数据结构，无论是 add、remove 都是 O(1) 时间复杂度。并且支持多个集合间的交集、并集、差集操作，利用这些集合操作，解决上边提到的统计问题。

### 业务场景一：共同好友（交集）

比如 QQ 中的共同好友正是聚合统计中的交集。我们将账号作为 Key，该账号的好友作为 Set 集合的 value。

执行步骤如下：

```redis
sadd user1 A B C
sadd user2 B C D
sinterstore user:共同好友 user1 user2	//统计user1与user2的共同好友，结果是B和C
```



### 业务场景二：每日新增的数量（差集）

比如，统计某个 App 每日新增注册用户量，只需要对近两天的总注册用户量集合取差集即可。

比如，2021-06-01 的总用户量存放在 `key = user:20210601` set 集合中，2021-06-02 的总用户量存放在 `key = user:20210602` 的集合中。



###  业务场景二：总共新增好友（并集）

还是差集的例子，统计 2021/06/01 和 2021/06/02 两天总共新增的用户量，只需要对两个集合执行并集。

使用命令：`SUNIONSTORE userid:new user:20210602 user:20210601`，此时新的集合 userid:new 则是两日新增的好友。

### 总结

Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致 Redis 实例阻塞。

所以，可以专门部署一个集群用于统计，让它专门负责聚合计算，或者是把数据读取到客户端，在客户端来完成聚合统计，这样就可以规避由于阻塞导致其他服务无法响应。



## 排序统计

Redis 的 4 个集合类型中（List、Set、Hash、Sorted Set），List 和 Sorted Set 就是有序的，Sorted Set 类型占用的内存容量是 List 类型的数倍之多，所以尽量使用list，对于列表数量不多的情况，可以用 Sorted Set 类型来实现。

- List：按照元素插入 List 的顺序排序，使用场景通常可以作为 消息队列、最新列表、排行榜；
- Sorted Set：根据元素的 score 权重排序，我们可以自己决定每个元素的权重值。使用场景（排行榜，比如按照播放量、点赞数）。

### 业务场景一：最新评论列表

每当一个用户评论，则利用 `LPUSH key value [value ...]` 插入到 List 队头。接着再用 `LRANGE key star stop` 获取列表指定区间内的元素。但是如果是频繁更新的列表，list类型的分页可能导致列表元素重复或者漏掉。

只有不需要分页（比如每次都只取列表的前 5 个元素）或者更新频率低（比如每天凌晨统计更新一次）的列表才适合用 List 类型实现。对于需要分页并且会频繁更新的列表，需用使用有序集合 Sorted Set 类型实现。



### 业务场景二：排行榜

比如要一周音乐榜单，我们需要实时更新播放量，并且需要分页展示。除此以外，排序是根据**播放量**来决定的，这个时候 List 就无法满足了。

实现步骤如下：

第一步，首先通过命令`zadd musicTop 100000000 青花瓷 8999999 花田错`将歌曲以及它对应的播放量添加到set里。

第二步：歌曲每播放一次，就通过命令`zincrby musicTop 1 青花瓷`将歌曲对应的分数+1。

第三步：最后我们需要获取 musicTop **前十**播放量音乐榜单，目前最大播放量是 N ，可通过如下指令获取：`zrangebyscore musicTop N-9 N WITHSCORES`。N获取方式是通过`zrevrange key start stop [WITHSCORES]`指令。其中元素的排序按 `score` 值递减(从大到小)来排列，如`ZREVRANGE musicTop 0 0 WITHSCORES`



## 基数统计

基数统计：统计一个集合中不重复元素的个数，常见于计算独立用户数（UV）。

Redis 提供了 `HyperLogLog` 数据结构就是用来解决种种场景的统计问题。

`HyperLogLog` 是一种不精确的去重基数方案，它的统计规则是基于概率实现的，标准误差 0.81%，这样的精度足以满足 UV 统计需求了。

关于 HyperLogLog 的原理过于复杂，如果想要了解的请移步：

- https://www.zhihu.com/question/53416615
- https://en.wikipedia.org/wiki/HyperLogLog

### 业务场景：统计网站的UV

方案一：采用集合（[Set](http://mp.weixin.qq.com/s?__biz=MzU3NDkwMjAyOQ==&mid=2247485665&idx=1&sn=3cf8e45aaa071fa26975bca34b8878e4&chksm=fd2a1283ca5d9b95889dbc3ec5949f784f6b8e6b469e49675731a8a544321fc4633058e27c41&scene=21#wechat_redirect)）这种数据结构，set中不允许有重复的value，网站名作为key，用户作为value，当插入（sadd）新的元素时，set中的个数增加，当插入重复元素时，set中的个数不变，最后通过`scard`命令统计元素个数，但是当访问量巨大，就需要一个超大的Set集合，将会浪费大量空间。

方案二：还可以利用 Hash 类型实现，将用户 ID 作为 Hash 集合的 key，访问页面则执行 HSET 命令将 value 设置成 1。

方案三：使用Redis 提供的 `HyperLogLog` 高级数据结构，每个 `HyperLogLog` 最多只需要花费 12KB 内存就可以计算 2 的 64 次方个元素的基数。Redis 对 `HyperLogLog` 的存储进行了优化，在计数比较小的时候，存储空间采用稀疏矩阵，占用空间很小。只有在计数很大，稀疏矩阵占用的空间超过了阈值才会转变成稠密矩阵，占用 12KB 空间。

使用步骤如下：

```redis
PFADD Redis数据 user1 user2 user3	//往HyperLogLog中添加信息
PFADD MySQL数据 user1 user2 user4
PFMERGE 数据库 Redis数据 MySQL数据	//合并两个HyperLogLog对象
PFCOUNT 数据库 // 返回值 = 4	//统计新的HyperLogLog对象的个数
```

