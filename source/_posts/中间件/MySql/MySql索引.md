---
title: MySql索引
tags: [MySql]      #添加的标签
categories: MySql
description: 
#cover: 
---



## 什么是索引

索引（在MySQL中也叫“键key”）是存储引擎快速找到记录的一种数据结构。在查询语句select前加上explain命令，可以获取索引执行情况列表。



## 索引类型

使用`show index from 表名;`命令可以查看索引详情。

1. 主键索引primary key：它是一种特殊的唯一索引，不允许有空值。一般是在建表的时候使用语句`PRIMARY KEY (id) `同时创建主键索引。注意：一个表只能有一个主键。

   ```mysql
   CREATE TABLE `phpcolor_ad` (  
   `id` mediumint(8) NOT NULL AUTO_INCREMENT,  
   `name` varchar(30) NOT NULL,
   PRIMARY KEY (`id`),  
   );
   ```

![MySql主键索引](https://gitee.com/hu-zhihong/picbed/raw/master/MySql%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95.png)

2. 唯一索引 UNIQUE：唯一索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一（每个表可以有多个 UNIQUE 约束，但是每个表只能有一个 PRIMARY KEY 约束）。可以通过`ALTER TABLE 表名 ADD UNIQUE (列名);`创建唯一索引；可以通过`ALTER TABLE 表名 ADD UNIQUE (列名1,列名2);`；也可以在创建表的时候使用语句`KEY type (type)`创建索引。

   ```mysql
   CREATE TABLE `phpcolor_ad` (  
   `id` mediumint(8) NOT NULL AUTO_INCREMENT,  
   `name` varchar(30) NOT NULL,  
   `type` mediumint(1) NOT NULL,  
   PRIMARY KEY (`id`),  
   KEY `type` (`type`)  
   );
   ```

创建组合索引结果如下：

![MySql唯一索引](https://gitee.com/hu-zhihong/picbed/raw/master/MySql%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95.png)



3. 普通索引 INDEX：这是最基本的索引，它没有任何限制，它与唯一索引在查询能力上是没差别的，主要考虑的是对更新性能的影响。**注意，如果索引是用B+树结构，则B+树的每个叶子节点都是主键值，因为这样可以节省存储空间以及保证一致性**。可以通过`ALTER TABLE 表名 ADD INDEX index_name (列名);`创建普通索引：

![MySql普通索引](https://gitee.com/hu-zhihong/picbed/raw/master/MySql%E6%99%AE%E9%80%9A%E7%B4%A2%E5%BC%95.png)



4. 组合索引 INDEX：即一个索引包含多个列，多用于避免回表查询。可以通过`ALTER TABLE 表名 ADD INDEX index_name(column1,column2,column3);`创建组合索引，它可以支持最左前缀原则，所以联合索引需要合理安排字段的前后顺序。

   ![（name，age）索引示意图](https://gitee.com/hu-zhihong/picbed/raw/master/%EF%BC%88name%EF%BC%8Cage%EF%BC%89%E7%B4%A2%E5%BC%95%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

   

5. 全文索引 FULLTEXT：也称全文检索，是目前搜索引擎使用的一种关键技术。可以通过`ALTER TABLE 表名 ADD FULLTEXT (列名);`创建全文索引。



索引一经创建不能修改，如果要修改索引，只能删除重建。可以使用
DROP INDEX index_name ON table_name;删除索引。



## 索引设计的原则

1）适合索引的列是出现在where子句中的列，或者连接子句中指定的列；
2）基数较小的类，索引效果较差，没有必要在此列建立索引；
3）使用短索引，如果对长字符串列进行索引，应该指定一个前缀长度，这样能够节省大量索引空间；
4）不要过度索引。索引需要额外的磁盘空间，并降低写操作的性能。在修改表内容的时候，索引会进行更新甚至重构，索引列越多，这个时间就会越长。所以只保持需要的索引有利于查询即可。



## 索引优化

有些时候虽然数据库有索引，但是并不被优化器选择使用。我们可以通过SHOW STATUS LIKE 'Handler_read%';查看索引的使用情况：

![MySql索引使用情况](https://gitee.com/hu-zhihong/picbed/raw/master/MySql%E7%B4%A2%E5%BC%95%E4%BD%BF%E7%94%A8%E6%83%85%E5%86%B5.png)

**Handler_read_key**：如果索引正在工作，Handler_read_key的值将很高。
**Handler_read_rnd_next**：数据文件中读取下一行的请求数，如果正在进行大量的表扫描，值将较高，则说明索引利用不理想。

索引优化规则：
1）如果MySQL估计使用索引比全表扫描还慢，则不会使用索引。
返回数据的比例是重要的指标，比例越低越容易命中索引。记住这个范围值——30%，后面所讲的内容都是建立在返回数据的比例在30%以内的基础上。

2）前导模糊查询不能命中索引。
假设name列创建了一个普通索引，前导模糊查询不能命中索引：
`EXPLAIN SELECT * FROM user WHERE name LIKE '%s%';`

**非前导模糊查询则可以使用索引**，可优化为使用非前导模糊查询：
`EXPLAIN SELECT * FROM user WHERE name LIKE 's%';`

3）复合索引的情况下，查询条件不包含索引列最左边部分（不满足最左原则），不会命中符合索引。
假设name,age,status列创建了复合索引，根据最左原则，可以命中复合索引index_name：
`SELECT * FROM user WHERE name='swj' AND status=1;`。

**注意，最左原则并不是说是查询条件的顺序，而是查询条件中是否包含索引最左列字段（如此例子中索引最左列字段是name）。**

4）union、in、or都能够命中索引，建议使用in，查询的CPU消耗：or>in>union。

5）用or分割开的条件，如果or前的条件中列有索引，而后面的列中没有索引，那么涉及到的索引都不会被用到。

因为or后面的条件列中没有索引，那么后面的查询肯定要走全表扫描，在存在全表扫描的情况下，就没有必要多一次索引扫描增加IO访问。

6）负向条件查询不能使用索引，可以优化为in查询。
负向条件有：!=、<>、not in、not exists、not like等。

7）范围条件查询可以命中索引。范围条件有：<、<=、>、>=、between等。

9）建立索引的列，不允许为null。