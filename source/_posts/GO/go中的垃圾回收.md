---
title: go中的垃圾回收
tags: [go]      #添加的标签
categories: 
  - GO
description: 
cover: 
---

## 常见的几种算GC法

这里垃圾回收的思路，有下面三种方式：

1. 标记-清除
2. 标记-整理
3. 标记-复制

### 标记-清除（golang使用的方式）

这种是最简单粗暴的方式

![go中标记-清除的GC方式](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E6%A0%87%E8%AE%B0-%E6%B8%85%E9%99%A4%E7%9A%84GC%E6%96%B9%E5%BC%8F.png)

> 会有碎片的问题，但是golang使用了分级策略，所以影响不大



### 标记-整理

初始状态：标记了需要删除的对象

标记-整理：把后面的对象整理过来让整个堆内存的碎片化情况大大改善但是开销很大，会造成GC卡顿（老版本的java用的这种，因为之前的数据量不大，影响相对小一点）

![go中标记-整理的GC方式](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E6%A0%87%E8%AE%B0-%E6%95%B4%E7%90%86%E7%9A%84GC%E6%96%B9%E5%BC%8F.png)



### 标记-复制

会将当前有用的内存复制到新的区域中，再把旧的存储空间清理，该方法会导致内存浪费，但解决了碎片化的问题

![go中标记-复制的GC方式](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%AD%E6%A0%87%E8%AE%B0-%E5%A4%8D%E5%88%B6%E7%9A%84GC%E6%96%B9%E5%BC%8F.png)



## 如何查找需要回收的对象

在程序中，有一些对象是不能被清除的，下面是具体例子

1. 被栈上的指针引用

2. 被全局变量指针引用

3. 被寄存器中的指针引用

上述变量被称为 Root Set （GC Root）

### 可达性分析标记法，串行GC

这种方法go1.3及之前使用，下面中格子是系统中一个一个栈空间，字母表示的不同的对象，字母之间边的情况是不同对象之间的关系。

该方法的垃圾回收是串行的，在扫描整个进程中的对象时，需要暂停其他所有协程，为什么，因为其他协程在运行时会影响这个垃圾回收机制的判断。
具体步骤是下面几步：

1. Stop The World， 暂停所有其他协程
2. 开始进行BFS（广度优先搜索]）进行搜索
3. 下一步搜索，先标记，后清除（这里就是清除对象G和H）
4. 释放对内存
5. 恢复所有其他协程

![go中GC的可达性分析标记法](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%ADGC%E7%9A%84%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%90%E6%A0%87%E8%AE%B0%E6%B3%95.png)



### 三色标记法

在go1.5及之后使用三色并发标记清除进行垃圾回收，以下图片节点颜色作用如下：

- 黑色：有用，已经分析扫描（已完成对其引用的遍历的对象）
- 灰色：有用，还未分析扫描（尚未完成对其引用的遍历的对象）
- 白色：暂时无用，最后需要清除的对象

1. 把所有的对象标记为白色（放入白色待处理队列中）。
    ![go三色标记法GC过程1](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E8%BF%87%E7%A8%8B1.png)



2. 然后并发地遍历待处理队列中的对象，把程序根结点集合RootSet里的对象标记为灰色（放入灰色的队列中）
   ![go三色标记法GC过程2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E8%BF%87%E7%A8%8B2.png)



3. BFS遍历标记为灰色的对象，找到关联的对象并标记为灰色，同时把自己标记为黑色
    ![go三色标记法GC过程3](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E8%BF%87%E7%A8%8B3.png)



4. 重复此步骤，直到只剩下 黑色 和 白色，灰色队列中无任何对象
    ![go三色标记法GC过程4](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E8%BF%87%E7%A8%8B4.png)

当我们全部的可达对象都遍历完后，灰色队列将不再存在灰色对象，目前全部内存的数据只有两种颜色，黑色和白色。那么黑色对象就是我们程序逻辑可达（需要的）对象，这些数据是目前支撑程序正常业务运行的，是合法的有用数据，不可删除，白色的对象是全部不可达对象，目前程序逻辑并不依赖他们，那么白色对象就是内存中目前的垃圾数据，需要被清除。



### 屏障机制

为了解决并发的GC问题，使用了屏障机制，屏障机制分为两种：插入屏障和删除屏障。



#### 插入屏障

场景：当已经遍历过某个对象后（该对象置为黑了），让该对象**指向一个插入的新对象**，这时新对象是白色的，就变成待GC的对象了。为了解决这个问题，就要使用插入屏障方法了。

具体操作：**在A对象引用B对象（新插入的对象）的时候，B对象被标记为灰色。**

注意：黑色对象的内存槽有两种位置, `栈`和`堆`. 栈空间的特点是容量小，但是要求相应速度快，因为函数调用弹出频繁使用，所以“插入屏障”机制，在**栈空间的对象操作中不使用**。 而仅仅使用在堆空间对象的操作中。

步骤：

1. 遍历灰色队列，将可达的对象，从白色标记为灰色，遍历之后的灰色，标记为黑色。

    ![go三色标记法GC插入屏障1](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E6%8F%92%E5%85%A5%E5%B1%8F%E9%9A%9C1.png)

    

2. 由于并发特性，此时别的协程向对象4添加对象8、对象1添加对象9，对象4在堆区，即将触发插入屏障机制（黑色对象添加白色，将白色改为灰色），对象8变成灰色；对象1不触发屏障机制，依然为白色。
    ![go三色标记法GC插入屏障2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E6%8F%92%E5%85%A5%E5%B1%8F%E9%9A%9C2.png)
    
    
    
3. 继续循环上述流程进行三色标记，直到没有灰色节点。
    ![go三色标记法GC插入屏障3](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E6%8F%92%E5%85%A5%E5%B1%8F%E9%9A%9C3.png)

    
    
4. 由于栈不执行屏障机制，所以为了解决栈并发GC问题，还需要栈重新进行三色标记扫描，但这次为了对象不丢失，要对本次标记扫描启动STW（暂停所有协程）暂停。直到栈空间的三色标记结束。
    ![go三色标记法GC插入屏障4](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E6%8F%92%E5%85%A5%E5%B1%8F%E9%9A%9C4.png)

    
    
5. 在STW中，将栈中的对象进行三色标记，直到没有灰色节点。
    ![go三色标记法GC插入屏障5](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E6%8F%92%E5%85%A5%E5%B1%8F%E9%9A%9C5.png)



 最后将栈和堆空间扫描剩余的全部白色节点清除。



#### 删除屏障

场景：当一个对象还没遍历到它时，就被它的父节点断开连接了，但是该节点还需要使用。为了解决这个问题，就要使用插入屏障方法了。

具体操作：**被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。**

步骤：

1. 灰色对象1删除对象5，如果不触发删除屏障，5-2-3路径与主链路断开，最后均会被清除。

![go三色标记法GC删除屏障1](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E5%88%A0%E9%99%A4%E5%B1%8F%E9%9A%9C1.png)

2. 触发删除屏障，被删除的对象5，被标记为灰色，继续执行三色标记过程，直到没有灰色节点。

![go三色标记法GC删除屏障2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/go%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%E6%B3%95GC%E5%88%A0%E9%99%A4%E5%B1%8F%E9%9A%9C2.png)





#### 混合屏障机制

