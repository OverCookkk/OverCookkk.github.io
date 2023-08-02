---
title: go-map
tags: [map]      #添加的标签
categories: 
  - GO
description:
#toc: true
#cover: 
---



## map定义

Go语言中提供的映射关系容器为`map`，其内部使用`散列表（hash）`实现。map是一种**无序**的基于`key-value`的数据结构，Go语言中的map是**引用类型**，必须初始化才能使用。

map类型的变量默认初始值为nil，需要使用make()函数来分配内存。语法为：

```go
make(map[KeyType]ValueType, [cap])
```

其中cap表示map的容量，该参数虽然不是必须的，但是我们应该在初始化map的时候就为其指定一个合适的容量。



map作为函数参数使用时，由于是它是引用类型，所以函数里改变map值，函数外也会跟着改变。



## map基本使用

map中的数据都是成对出现的，map的插入元素示例代码如下：

```go
func main() {
	scoreMap := make(map[string]int, 8)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
}
```

map也支持在声明的时候填充元素，例如：

```go
func main() {
	userInfo := map[string]string{
		"username": "xxx",
		"password": "123456",
	}
}
```

Go语言中有个判断map中键是否存在的优雅写法，格式如下:

```go
if value, ok := map[key]; ok {
}
```

Go语言中使用`for range`遍历map：

```go
for k, v := range scoreMap {
    fmt.Println(k, v)
}
```

**注意：** 遍历map时的元素顺序与添加键值对的顺序无关。



## map底层概述

- `map` 只是一个哈希表。数据被排列成一组 `bucket`。
- 每个 `bucket` 最多包含 `8` 个 `键/值` 对。
- **一个元素的哈希值的低位字节位用于选择 `bucket`，最高的 8 bit存放在桶的开头，用于后续产生哈希冲突时优先对比高8bit，再相同时才对比key。**
- 每个 `bucket` 包含每个哈希的几个高位字节位(`tophash`)，以区分单个桶中的条目。
- 如果超过 `8` 个 `key` 哈希到同一个桶，我们将额外的桶以链表的方式起来。（解决哈希冲突，链表法）
- 当哈希表扩容时，我们会分配一个两倍大的新 `bucket` 数组。然后 `bucket` 从旧 `bucket` 数组增量复制到新 `bucket` 数组。
- `map` 迭代器遍历 `bucket` 数组，并按遍历顺序返回键（遍历完普通桶之后，遍历溢出桶）。
- 为了保持迭代语义，我们永远不会在它们的桶中移动键（`bucket` 内键的顺序在扩容的时候不变。如果改变了桶内键的相对顺序，键可能会返回 0 或 2 次）。
- 在扩容哈希表时，迭代器仍在旧的桶中迭代，并且必须检查新桶，检查正在迭代的 `bucket` 是否已经被迁移到新桶。



## go map底层具体原理

https://juejin.cn/post/7177582930313609273#heading-12

https://zhuanlan.zhihu.com/p/421607260





## 并发安全的sync.Map

如果要保证并发的安全，最朴素的想法就是使用锁，但是这意味着要把一些并发的操作强制串行化，性能自然就会下降。

### 核心思想

事实上，除了使用锁，还有一个办法，也可以达到类似并发安全的目的，就是原子操作（atomic）。sync.Map的设计非常巧妙，充分利用了atmoic的尽量使用原子操作和mutex的配合，最大程度上减少了锁的使用。

```go
type Map struct {
	mu Mutex
	read atomic.Value // readOnly
    dirty map[interface{}]*entry
	misses int
}

type readOnly struct {
	m       map[interface{}]*entry
	amended bool //是否修改， true if the dirty map contains some key not in m.
}

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	p unsafe.Pointer // *interface{}
}
```

使用了两个原生的map作为存储介质，分别是read map和dirty map（只读字典和脏字典）。

- 只读字典使用atomic.Value来承载，保证原子性和高性能；脏字典则需要用互斥锁来保护，保证了互斥。

- 只读字典和脏字典中的键值对集合并不是实时同步的，它们在某些时间段内可能会有不同。

- 无论是read还是dirty，本质上都是map[interface{}]*entry类型，这里的entry其实就是Map的value的容器。

- entry的本质，是一层封装，可以表示具体值的指针，也可以表示key已删除的状态（即逻辑假删除）



### 架构设计图

![sysc.Map底层数据结构原理图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/sysc.Map%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%8E%9F%E7%90%86%E5%9B%BE.png)

- read map由于是原子包托管，主要负责高性能，但是无法保证拥有全量的key（因为对于新增key，会首先加到dirty中），所以read某种程度上，类似于一个key的快照。

- **dirty map拥有全量的key**，当Store操作要新增一个之前不存在的key的时候，预先是增加自dirty中的。

- 在查找指定的key的时候，总会先去只读字典中寻找，并不需要锁定互斥锁。只有当read中没有，但dirty中可能会有这个key的时候，才会在锁的保护下去访问dirty。

- 在存储键值对的时候，只要read中已存有这个key，并且该键值对未被标记为“expunged”，就会把新值存到里面并直接返回，这种情况下也不需要用到锁。

- expunged和nil，都表示标记删除，但是它们是有区别的，简单说expunged是read独有的，而nil则是read和dirty共有的。

- read和map的关系，是一直在动态变化的，可能存在重叠，也可能是某一方为空；重叠的公共部分，由分为两种情况，nil和normal。

- read和dirty之间是会互相转换的，在dirty中查找key对次数足够多的时候，sync.Map会把dirty直接作为read，即触发dirty=>read升级。同时在某些情况，也会出现read=>dirty的重塑



更多read和dirty转换的细节参考https://view.inews.qq.com/a/20220613A08UDG00



### sync.Map基本使用

1. Load：会从read和dirty两个中拿，如果read有且dirty中没有修改过，判断该条记录的状态如果是nil或者expunged（dirty中被删掉）则返回获取不到，其他状态返回read中的值；如果read中没有且dirty中修改，就加锁并从dirty中拿，并把missed字段+1,同步dirty到read,dirty=nil.

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        // 可能锁等待的时候read中已经有该key了，所以做二次检查
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // Regardless of whether the entry was present, record a miss: this key
            // will take the slow path until the dirty map is promoted to the read
            // map.
            m.missLocked() //misses计数+1，且如果misses小于dirty的长度，dirty同步到read，dirty=nil
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load() 
}

func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	m.read.Store(readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}
```

2. Store：如果read中存在key且尝试更新成功（p不是expaunged状态，即在dirty中被删掉）,返回即可 否则加锁，并再次检查read中有没有该key,有可能加锁的时候read中已经被同步了该key了。 如果read中有key而之前没有，说明从dirty中同步过来一波，如果p是expunged,就把p设置成nil并且dirty中加入entry 如果read中没有dirty中有，直接存储 如果read中没有dirty中也没有，如果read和dirty中是一样的没有修改过，如果dirty为nil，同步read到dirty,存储到dirty,并且修改read的属性字段amend设置成修改过true。
3. Read：如果read中不存在key且read和dirty中不一致，上锁，再次检查read,如果还是如此，删除dirty中的key 如果read中存在key,如果entry.p为nil或者expunged，返回false,如果不是，entry.p设置成nil，标记删除。



### 总结

- sync.Map中有read与dirty,操作dirty需要锁

- 每次判断完read有无key之后进行进行加锁操作后还需要再次判断read有无key，防止在此时间内进行了从dirty同步到read操作，然后操作dirty

- load场景中主要从read读，没有再从dirty读，同时会计数，如果数量超过一定量，数据从dirty同步到read

- 其中有两处read和dirty互相同步的地方，在store场景主要存到dirty, 如果dirty中有直接更新，当dirty为nil的时候会把read中不是expunged状态的同步到dirty；dirty什么时候为nil呢，在load的时候从dirty中读的次数太多的时候会把dirty同步到read，并且dirty=nil

- 删除为标记删除，把entry.p=nil来标记。