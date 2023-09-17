---
title: go中数组与切片对比
tags: [go]      #添加的标签
categories: 
  - GO
description: Go 数组比切片好在哪？
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00745-1935579311.png
---



具体解释：

[go数组比切片好在哪？]: https://mp.weixin.qq.com/s/zp1vdhGukEYKpzAdPt--Mw



## 总结

如下：

- 数组是值对象，可以进行比较，可以将数组用作 map 的映射键。而这些，切片都不可以，不能比较，无法作为 map 的映射键。
- 数组有编译安全的检查，可以在早起就避免越界行为。切片是在运行时会出现越界的 panic，阶段不同。
- 数组可以更好地控制内存布局，若拿切片替换，会发现不能直接在带有切片的结构中分配空间，数组可以。
- 数组在访问单个元素时，性能比切片好。
- 数组的长度，是类型的一部分。在特定场景下具有一定的意义。
- 数组是切片的基础，每个数组都可以是一个切片，但并非每个切片都可以是一个数组。如果值是固定大小，可以通过使用数组来获得较小的性能提升（至少节省 slice 头占用的空间）。