---
title: c++标准库锁的使用
tags: [c++]      #添加的标签
categories: 
  - c++
description: C++11中锁的浅谈笔记
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00747-1007550032.png
---

​		C++11中，std::unique_lock, std::lock_guard, std::recursive_mutex可以简单理解为对std::mutex的封装，且对互斥量的unlock是在对象（比如std::unique_lock对象)销毁时执行。

（1）区域锁lock_guard使用起来比较简单，除了构造函数外没有其他member function，在整个区域都有效。

```c++
用法：std::lock_guard<std::mutex>或std::lock_guard<std::recursice_mutex>
```

（2）区域锁unique_lock**除了lock_guard的功能外**，提供了更多的member_function，相对来说更灵活一些。unique_lock的最有用的一组函数为：lock()、try_lock()、try_lock_for()、try_lock_until()、unlock()。

```c++
std::unique_lock<std::mutex>或std::unique_lock<std::recursice_mutex>
```

通过上面的函数，可以通过lock/unlock可以比较灵活的控制锁的范围，减小锁的粒度。
通过try_lock_for/try_lock_until则可以控制加锁的等待时间，此时这种锁为乐观锁。

（3）一个递归互斥量(recursive mutex)是一个可锁的对象，只是它允许同一个线程对互斥量对象获取多级所有权。如此，它允许已进行了锁操作的线程，再次lock（或try-lock）互斥量对象，获取该互斥量对象一个新所有权：互斥量对象一直为拥有线程锁住，在调用unlock的次数和lock次数相同前。

