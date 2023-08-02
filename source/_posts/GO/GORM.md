---
title: GORM
#tags: [hexo建站]      #添加的标签
categories: 
  - GO
description: 
#cover: 
---



## 什么是ORM

```text
Object Relational Mapping
 对象	    关系		映射
```

把程序中的对象实例或者对象，映射成数据库中的一条数据。

数据表<---->结构体

数据行<---->结构体实例

字段结<---->构体字段

```go
//相当于数据表
type UserInfo struct{
    ID uint	//相当于字段
    Name string
    Gender string
}
//相当于数据行
u1 := UserInfo{
    ID : 1,
    Name : "lihua",
    Gender: "男"，
}
```



## ORM优缺点

优点：

- 提高开发效率

缺点：

- 牺牲执行性能
- 牺牲灵活性
- 弱化SQL能力



