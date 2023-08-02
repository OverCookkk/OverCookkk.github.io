---
title: Redis高可用篇：哨兵集群原理
tags: [Redis,高可用,哨兵]      #添加的标签
categories: Redis                           #添加的分类
description: 哨兵用于监控redis主从库的情况。
#cover: 
---



## 概要

主从复制是高可用的基石，从库宕机依然可以将请求发送给主库或者其他从库，但是 Master 宕机，只能响应读操作，写请求无法再执行。所以主从复制架构面临一个严峻问题，**主库挂了，无法执行「写操作」，无法自动选择一个 Slave 切换为 Master**，也就是无法**故障自动切换**。为此，Redis 官方提供一个高可用方案——**哨兵（Sentinel）**。



## 哨兵的任务

搭建实例采用三个哨兵形成集群，三个数据节点（一主两从）方式搭建，如下图所示：

![Redis哨兵集群](https://gitee.com/hu-zhihong/picbed/raw/master/Redis%E5%93%A8%E5%85%B5%E9%9B%86%E7%BE%A4.png)

哨兵是 Redis 的一种运行模式，它专注于**对 Redis 实例（主节点、从节点）运行状态的监控，并能够在主节点发生故障时通过一系列的机制实现选主及主从切换，实现故障转移，确保整个 Redis 系统的可用性**。Redis 哨兵具备的能力有如下几个：

- **监控**：持续监控 master 、slave 是否处于预期工作状态。
- **自动切换主库**：当 Master 运行故障，哨兵启动自动故障恢复流程：从 slave 中选择一台作为新 master。
- **通知**：让 slave 执行 replicaof（复制），与新的 master 同步；并且通知客户端与新 master 建立连接。

![redis哨兵执行任务与目标](https://gitee.com/hu-zhihong/picbed/raw/master/redis%E5%93%A8%E5%85%B5%E6%89%A7%E8%A1%8C%E4%BB%BB%E5%8A%A1%E4%B8%8E%E7%9B%AE%E6%A0%87.png)

### 监控

Sentinel 通过以每秒一次的频率向所有 Master、Slave、其他 Sentinel 发送 PING 命令，如果 slave 没有在在规定时间内响应「哨兵」的 PING 命令，「哨兵」就认为它挂掉了，就会将他记录为「下线状态」；假如 master 没有在规定时间响应 「哨兵」的 PING 命令，哨兵就判定master下线，开始执行「自动切换 master 」的流程。

PING 命令的回复有两种情况：

1. 有效回复：返回 +PONG、-LOADING、-MASTERDOWN 任何一种；
2. 无效回复：有效回复之外的回复，或者指定时间内返回任何回复。

为了防止误判master挂掉，「哨兵」设计了「主观下线」和「客观下线」两种暗号。

1. 主观下线：如果回复是无效回复，哨兵会把master标记为主观下线，如果检测的是slave，哨兵直接标记为主观下线，因为主库还在，从库挂掉影响不大。
2. 客观下线：**判断 master 是否下线不能只有一个「哨兵」说了算，只有一定数量（通过配置文件配置）的哨兵判断 master 已经「主观下线」，这时候才能将 master 标记为「客观下线」**。

只有 master 被判定为「客观下线」，才会进一步触发哨兵开始主从切换流程。

![redis哨兵的客观下线](https://gitee.com/hu-zhihong/picbed/raw/master/redis%E5%93%A8%E5%85%B5%E7%9A%84%E5%AE%A2%E8%A7%82%E4%B8%8B%E7%BA%BF.png)



### 自动切换主库

「哨兵」的第二个任务，选择新 master 。需要从slave中按照一定规则选择一个合适的从库作为主库。按照一定的 **「筛选条件」 + 「打分」** 策略，选出master。

![redis新master选择](https://gitee.com/hu-zhihong/picbed/raw/master/redis%E6%96%B0master%E9%80%89%E6%8B%A9.png)



### 通知

最后一个任务，「哨兵」将新 「master 」的连接信息发送给其他 slave，并且让 slave 执行 replacaof 命令，和新「master 」建立连接，并进行数据复制。

除此之外，「哨兵」还需要将新master的连接信息通知客户端，使得让所有将读写请求转移到新 master。





## 哨兵集群工作原理

哨兵部门并不是一个人，多个人共同组成一个哨兵集群，哨兵之间可以相互通信，主要归功于 Redis 的 `pub/sub` 发布/订阅机制。

master 有一个 `__sentinel__:hello` 的专用通道，用于哨兵之间发布和订阅消息。当多个哨兵实例都在主库上做了发布和订阅操作后，它们之间就能知道彼此的 IP 地址和端口，从而相互发现建立连接。

如下图所示，哨兵 2 向 Master 发送 `INFO` 命令，Master 就把 slave 列表返回给哨兵 2，哨兵 2 便根据 slave 列表连接信息与每一个 slave 建立连接，并基于此连接实现持续监控。剩下的哨兵也同理基于此实现监控。

![redis哨兵与master以及slave连接图](https://gitee.com/hu-zhihong/picbed/raw/master/redis%E5%93%A8%E5%85%B5%E4%B8%8Emaster%E4%BB%A5%E5%8F%8Aslave%E8%BF%9E%E6%8E%A5%E5%9B%BE.png)