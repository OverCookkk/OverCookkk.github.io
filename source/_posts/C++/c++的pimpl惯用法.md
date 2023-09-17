---
title: c++的pimpl 惯用法
tags: [c++]      #添加的标签
categories: 
  - c++
description: Pimpl是“pointer to implementation”的缩写， 该技巧可以避免在头文件中暴露私有细节。
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00748-1007550031.png
---



## 使用场景

现在这里有一个名为 **CSocketClient** 的网络通信类，定义如下：

```c++
/**
 * 网络通信的基础类, SocketClient.h
 */
class CSocketClient
{
public:
    CSocketClient();
    ~CSocketClient();
 
public:  
    void SetProxyWnd(HWND hProxyWnd);

    bool    Init(CNetProxy* pNetProxy);
    bool    Uninit();
    
    int Register(const char* pszUser, const char* pszPassword); 
    void GuestLogin();  
.........

private:
    void LoadConfig();
    static UINT CALLBACK SendDataThreadProc(LPVOID lpParam);
    static UINT CALLBACK RecvDataThreadProc(LPVOID lpParam);
    bool Send();
    bool Recv();
    bool CheckReceivedData();
    void SendHeartbeatPackage();

private:
    SOCKET                          m_hSocket;
    short                           m_nPort;
    char                            m_szServer[64];
    long                            m_nLastDataTime;        //最近一次收发数据的时间
    long                            m_nHeartbeatInterval;   //心跳包时间间隔，单位秒
..............
};
```

**CSocketClient** 类的 **public** 方法提供对外接口供第三方使用，每个函数的具体实现在 **SocketClient.cpp** 中，对第三方使用者不可见。在 Windows 系统上作为提供给第三方使用的库，一般需要提供给使用者 .h、.lib和 .dll文件，在 Linux 系统上需要提供 .h、.a 或 .so文件。



## 使用pimpl惯用法

**pimpl** 惯用法，即 **Pointer to Implementation**，为了能保持对外的接口不变，又能尽量不暴露一些关键性的成员变量和私有函数的实现，把关键性的成员变量和私有函数放到另个一个Impl类中,

```c++
class Impl
{
public:
	Impl()
	{
        //TODO: 你可以在这里对成员变量做一些初始化工作
	}
	
	~Impl()
	{
        //TODO: 你可以在这里做一些清理工作
	}
	
public:
	SOCKET                          m_hSocket;
    short                           m_nPort;
    char                            m_szServer[64];
    long                            m_nLastDataTime;        //最近一次收发数据的时间
    long                            m_nHeartbeatInterval;   //心跳包时间间隔，单位秒
..........
};
```

然后在**CSocketClient**类中定义一个**Impl** 类的指针，通过该指针调用**Impl** 类里面的方法，

```c++
/**
 * 网络通信的基础类, SocketClient.h
 */
class CSocketClient
{
public:
    CSocketClient();
    ~CSocketClient();

 //重复的代码省略...

private:
	class   Impl;
    Impl*	m_pImpl;
};
```



## 优点

这个方法的优点：

- 核心数据成员被隐藏；

  核心数据成员被隐藏，不必暴露在头文件中，对使用者透明，提高了安全性。

- 降低编译依赖，提高编译速度；

  由于原来的头文件的一些私有成员变量可能是非指针非引用类型的自定义类型，需要在当前类的头文件中包含这些类型的头文件，使用了 **pimpl** 惯用法以后，这些私有成员变量被移动到当前类的 cpp 文件中，因此头文件不再需要包含这些成员变量的类型头文件，当前头文件变“干净”，这样其他文件在引用这个头文件时，依赖的类型变少，加快了编译速度。

- 接口与实现分离。

  使用了 **pimpl** 惯用法之后，即使 **CSocketClient** 或者 **Impl** 类的实现细节发生了变化，对使用者都是透明的，对外的 **CSocketClient** 类声明仍然可以保持不变。例如我们可以增删改 Impl 的成员变量和成员方法而保持 **SocketClient.h** 文件内容不变；如果不使用 **pimpl** 惯用法，我们做不到不改变 **SocketClient.h** 文件而增删改 **CSocketClient** 类的成员。
