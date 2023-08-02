---
title: 位图BitMap
tags: [BitMap]      #添加的标签
categories: 数据结构
description: 
#cover: 
---



## 场景

给40亿个不重复的无符号整数，没排过序。给一个无符号整数，如何快速判断一个数是否在这40亿个数中。



## 位图BitMap

位图BitMap：位图是一个**数组**的**每一个数据**的**每一个二进制位**表示一个数据，0表示数据不存在，1表示数据存在。

![BitMap示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/BitMap%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

图中，第一行数值表示一个uint类型（4个字节），136存放在第四个uint类型里，并在该uint类型的第25bit位置上。

BitMap实现代码如下：

```c++
#include <iostream>
#include <vector>
using namespace std;

class BitMap
{
public:
    BitMap(size_t range)
    {
        //此时多开辟一个空间
        _bits.resize(range >> 5 + 1);	//range >> 5等价于range / 32
    }
    void Set(size_t x)
    {
        int index = x / 32;//确定哪个数据（区间）
        int temp = x % 32;//确定哪个Bit位		x&7==x%8.该Byte里第几个 
        _bits[index] |= (1 << temp);//位操作即可
    }
    void Reset(size_t x)
    {
        int index = x / 32;
        int temp = x % 32;
        _bits[index] &= ~(1 << temp);//取反
    }
    bool Get(size_t x)
    {
        int index = x / 32;
        int temp = x % 32;
        if (_bits[index]&(1<<temp))
            return 1;
        else
            return 0;
    }

private:
    vector<int> _bits;	//有(n/32)个int，每个int有32个bit位置
    //此外也可以使用char* _bits; int gsize; 此时有(n/8)个char，每个char有8个bit位置
};
```

假设我们需要查找1000范围内的数值，则需要`BitMap bm(1000);`内部的`_bits`生成了32个int就可以保存0~1000的值。（一个int，有32个bit，32个int就有1024个bit，每个bit就可以保存一个数值）



## 解决方法

40亿QQ号码，用位图中的1个bit存储一个QQ号码，那么8个bit等于1个字节，40亿/8/1024/1024=476M，只需要不到512MB就可以存储完40亿QQ号码。

使用上面的BitMap类，使用Set方法，把40亿个QQ号码存到位图中，然后只需要调用Get方法，判断该bit位置的值是否为1就可以知道QQ号码是否存在。



## 拓展

### 问题一：用BitMap进行排序

文件中有40亿个互不相同的QQ号码，请设计算法对QQ号码进行排序，内存限制1G. 

直接用bitmap存40亿个QQ号码，然后从小到大遍历正整数，当bitmapFlag的值为1时，就输出该值，输出后的正整数序列就是排序后的结果。



### 问题二：用BitMap求Top-K问题

文件中有40亿个互不相同的QQ号码，求这些QQ号码的top-K，内存限制1G. 

除了小顶堆或者文件切割方法外，可以直接用bitmap排序，解决top-K问题。
