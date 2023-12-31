---
title: c++智能指针
tags: [c++]      #添加的标签
categories: 
  - c++
description: 
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00751-3020921006.png
---



## 前言

C++ 中有四种智能指针：auto_pt、unique_ptr、shared_ptr、weak_ptr ，包含在头文件\<memory\>中，其中后三个是 C++11 支持，第一个已经被 C++11 弃用且被 unique_prt 代替，不推荐使用。下文将对其逐个说明。



## auto_ptr

构造方法：

```c++
//初始化方式1
std::auto_ptr<int> ap1(new int(8));
//初始化方式2
std::auto_ptr<int> ap2;
ap2.reset(new int(8));
```

当复制一个 std::auto_ptr 对象时（拷贝复制或 operator= 复制），原对象所持有的堆内存对象也会转移给复制出来的对象，原对象会变为空，如果继续对原对象操作会产生错误，所以auto_ptr被unique_ptr替代。



## unique_ptr

std::unique_ptr 对其持有的堆内存具有唯一拥有权，也就是 std::unique_ptr 不可以拷贝或赋值给其他对象，其拥有的堆内存仅自己独占，std::unique_ptr 对象销毁时会释放其持有的堆内存。

构造方法：

```c++
//初始化方式1
std::unique_ptr<int> up1(new int(123));
//初始化方式2
std::unique_ptr<int> up2;
up2.reset(new int(123));
//初始化方式3 (-std=c++14)
std::unique_ptr<int> up3 = std::make_unique<int>(123);
```

应该尽量使用初始化方式 3 的方式去创建一个 std::unique_ptr 而不是方式 1 和 2，因为形式 3 更安全。



## shared_ptr 

std::shared_ptr 持有的资源可以在多个 std::shared_ptr 之间共享，每多一个std::shared_ptr 对资源的引用，资源引用计数将增加 1，每一个指向该资源的 std::shared_ptr 对象析构时，资源引用计数减 1，最后一个 std::shared_ptr 对象析构时，发现资源计数为 0，将释放其持有的资源。
构造方法与unique_ptr一致，另外，std::shared_ptr 有几个常用函数如下：

- void swap (unique_ptr& x)：将 shared_ptr 对象的内容与对象 x 进行交换，在它们两者之间转移管理指针的所有权而不破坏或改变二者的引用计数。
- void reset()、void reset (ponit p)：没有参数时，先将管理的计数器引用计数减一并将管理的指针和计数器置清零。有参数 p 时，先做面前没有参数的操作，再管理 p 的所有权和设置计数器。
- element_type* get()：得到其管理的指针。
- long int use_count()：返回与当前智能指针对象在同一指针上共享所有权的 shared_ptr 对象的数量，如果这是一个空的 shared_ptr，则该函数返回 0。如果要用来检查 use_count 是否为 1，可以改用成员函数 unique 会更快。
- bool unique()：返回当前 shared_ptr 对象是否不和其他智能指针对象共享指针的所有权，如果这是一个空的 shared_ptr，则该函数返回 false。
- element_type& operator\*()：重载指针的 * 运算符，返回管理的指针指向的地址的引用。
- element_type* operator->()：重载指针的 -> 运算符，返回管理的指针，可以访问其成员。
- explicit operator bool()：返回存储的指针是否已经是空指针，返回的结果与 get() != 0 相同。



## weak_ptr

std::weak_ptr 是一个不控制资源生命周期的智能指针，是对对象的一种弱引用，只是提供了对其管理的资源的一个访问手段，引入它的目的为协助 std::shared_ptr 工作。std::weak_ptr 可以从一个 std::shared_ptr 或另一个 std::weak_ptr 对象构造，它的构造和析构不会引起引用计数的增加或减少。

**std::weak_ptr 的正确使用场景是那些资源如果可能就使用，如果不可使用则不用的场景，它不参与资源的生命周期管理**



## make_shared存在的必要性

### 引用计数原理

C++11中的shared_ptr的定义我们截取一部分大概是这样的：

```c++
template<class T>
class shared_ptr{
private:
    T *px;                  // 指向所引用对象的指针
    unsigned long* pn;      // 指向引用计数的指针
};
```

用px来记录引用的对象的指针，使用pn来记录有多少个shared_ptr引用了相同对象（引用计数），当pn指向的引用计数为0时，`delete px`；



### shared_ptr和make_shared

C++11直接使用 `shared_ptr<T>` 和 `make_shared<T>` 都可以创建智能指针。

1. 使用shared_ptr直接创建智能指针：`auto p = shared_ptr<int>(new int(100));`

会有下面的两个过程：
（1）`new int`申请内存，并把指针传给shared_ptr中的px
（2）在shared_ptr中，会另外申请一块内存，初始化引用计数为1，并把指针赋值给pn

这样把创建一个智能指针需要分两步申请内存，会存在下面两个问题：
（1）当 `new int` 申请内存成功，但引用计数内存申请失败时，很可能造成内存泄漏。
（2）内存分配是一个消耗性能的过程，分两次分配内存，意味着性能会下降。

2. 使用make_shared创建智能指针：`auto p = make_shared<int>(100);`，make_shared只会申请一次内存，这块内存会大于int所占用的内存，多出的部分被用于智能指针引用计数。这样就避免了直接使用shared_ptr带来的问题。
