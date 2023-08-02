---
title: Sql语句优化的小技巧
tags: [MySql]      #添加的标签
categories: MySql
description: 
#cover: 
---

## 避免使用select*



## 用union all代替union



## 小表驱动达标

小表驱动大表，也就是说用小表的数据集驱动大表的数据集。

假如有order和user两张表，其中order表有10000条数据，而user表有100条数据。

这时如果想查一下，所有有效的用户下过的订单列表。

可以使用`in`关键字实现：

```mysql
select * from order
where user_id in (select id from user where status=1)
```

此业务需求用in关键字去实现更加合适，因为如果sql语句中包含了in关键字，则它会优先执行in里面的`子查询语句`，然后再执行in外面的语句。如果in里面的数据量很少，作为条件查询速度更快。



## 批量操作

如果你有一批数据经过业务处理之后，需要插入数据，该怎么办？

如果在循环中逐条插入数据，

```c++
for(Order order: list)
{
   orderMapper.insert(order):
}
//insert into order(id,code,user_id) values(123,'001',100);
```

该操作需要多次请求数据库，才能完成这批数据的插入。

这时，提供一个批量插入数据的方法。

```c++
orderMapper.insertBatch(list);
//insert into order(id,code,user_id) values(123,'001',100),(124,'002',100),(125,'003',101);
```

这样只需要远程请求一次数据库，sql性能会得到提升，数据量越多，提升越大。

**但需要注意的是，不建议一次批量操作太多的数据，如果数据太多数据库响应也会很慢。批量操作需要把握一个度，建议每批数据尽量控制在500以内。如果数据多于500，则分多批次处理。**



## 多用limit

有时候，我们需要查询某些数据中的第一条，比如：查询某个用户下的第一个订单，想看看他第一次的首单时间。

```mysql
select id, create_date 
from order 
where user_id=123 
order by create_date asc 
limit 1;
```

使用`limit 1`，只返回该用户下单时间最小的那一条数据即可。



## in中值太多

对于批量查询接口，我们通常会使用`in`关键字过滤出数据。比如：想通过指定的一些id，批量查询出用户信息。

```mysql
select id,name from category where id in (1,2,3...100000000);
```

如果我们不做任何限制，该查询语句一次性可能会查询出非常多的数据，很容易导致接口超时。

所以建议使用limit限制次数，最多500条。



## 高效的分页

有时候，列表页在查询数据时，为了避免一次性返回过多的数据影响接口性能，我们一般会对查询接口做分页处理。

在mysql中分页一般用的`limit`关键字：

```mysql
select id,name,age 
from user limit 10,20;
```

如果表中数据量少，用limit关键字做分页，没啥问题。但如果表中数据量很多，用它就会出现性能问题，如：

```mysql
select id,name,age 
from user limit 1000000,20;
```

mysql会查到1000020条数据，然后丢弃前面的1000000条，只查后面的20条数据，这个是非常浪费资源的。

海量数据分页应该先找到上次分页最大的id，然后利用id上的索引查询。不过该方案，要求id是连续的，并且有序的，如下：

```mysql
select id,name,age 
from user where id > 1000000 limit 20;
```

还能使用`between`优化分页。

```mysql
select id,name,age 
from user where id between 1000000 and 1000020;
```

需要注意的是between要在唯一索引上分页，不然会出现每页大小不一致的问题。



## 用连接查询代替子查询

mysql中如果需要从两张以上的表中查询出数据的话，一般有两种实现方式：`子查询` 和 `连接查询`。

子查询的例子如下：

```mysql
select * from order
where user_id in (select id from user where status=1)
```

子查询语句可以通过`in`关键字实现，一个查询语句的条件落在另一个select语句的查询结果中。程序先运行在嵌套在最内层的语句，再运行外层的语句。

子查询语句的优点是简单，结构化，如果涉及的表数量不多的话。

但缺点是**mysql执行子查询时，需要创建临时表，查询完毕后，需要再删除这些临时表，有一些额外的性能消耗**。

这时可以改成连接查询。具体例子如下：

```mysql
select o.* from order o
inner join user u on o.user_id = u.id
where u.status=1
```



## join的表不宜过多

根据阿里巴巴开发者手册的规定，join表的数量不应该超过`3`个。

如果join太多，mysql在选择索引的时候会非常复杂，很容易选错索引。



## 控制索引的数量

众所周知，索引能够显著的提升查询sql的性能，但索引数量并非越多越好。

因为表中新增数据时，需要同时为它创建索引，而索引是需要额外的存储空间的，而且还会有一定的性能消耗。

阿里巴巴的开发者手册中规定，单表的索引数量应该尽量控制在`5`个以内，并且单个索引中的字段数不超过`5`个。

mysql使用的B+树的结构来保存索引的，在insert、update和delete操作时，需要更新B+树索引。如果索引过多，会消耗很多额外的性能。

那么，问题来了，如果表中的索引太多，超过了5个该怎么办？

这个问题要辩证的看，如果你的系统并发量不高，表中的数据量也不多，其实超过5个也可以，只要不要超过太多就行。

但对于一些高并发的系统，请务必遵守单表索引数量不要超过5的限制。

那么，高并发系统如何优化索引数量？

能够建联合索引，就别建单个索引，可以删除无用的单个索引。

将部分查询功能迁移到其他类型的数据库中，比如：Elastic Seach、HBase等，在业务表中只需要建几个关键索引即可。



## 选择合理的字段类型

我们在选择字段类型时，应该遵循这样的原则：

1. 能用数字类型，就不用字符串，因为字符的处理往往比数字要慢。
2. 尽可能使用小的类型，比如：用bit存布尔值，用tinyint存枚举值等。
3. 长度固定的字符串字段，用char类型。
4. 长度可变的字符串字段，用varchar类型。
5. 金额字段用decimal，避免精度丢失问题。



## 提升group by的效率

```mysql
select user_id,user_name from order
group by user_id
having user_id <= 200;
```

这种写法性能不好，它先把所有的订单根据用户id分组之后，再去过滤用户id大于等于200的用户。

分组是一个相对耗时的操作，为什么我们不先缩小数据的范围之后，再分组呢？

```mysql
select user_id,user_name from order
where user_id <= 200
group by user_id
```

使用where条件在分组前，就把多余的数据过滤掉了，这样分组时效率就会更高一些。

> 其实这是一种思路，不仅限于group by的优化。我们的sql语句在做一些耗时的操作之前，应尽可能缩小数据范围，这样能提升sql整体的性能。



## 索引优化
