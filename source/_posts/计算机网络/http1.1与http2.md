---
title: http1.1与http2
#tags: [http]      #添加的标签
categories: 
  - 计算机网络
  #- http
description: http/1.1与http/2的区别与原理
#cover: 
---



## 什么是HTTP/2 ？

2015年，互联网工程任务组（IETF）发布了HTTP / 2，这是最有用的互联网协议HTTP的第二个主要版本。 它源自较早的实验SPDY协议。



## HTTP/2与HTTP/1.1的区别

1. **请求多路复用**

HTTP / 2可以通过单个TCP连接并行发送多个数据请求。 这是HTTP / 2协议的最高级功能，因为它允许您从一台服务器异步下载Web文件。 大多数现代浏览器将TCP连接限制为一台服务器。

![http1_and_http2_tcp](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/http1_and_http2_tcp.jpg)



2. **标头压缩**

HTTP / 1.1的标头没有压缩，HTTP / 2压缩大量冗余头帧。它使用HPACK规范作为标头压缩的简单安全方法。 客户端和服务器都维护在先前的客户端-服务器请求中使用的标头列表。



3. **二进制协议**

HTTP / 1.1使用文本数据，这通常在网络上效率较低。而HTTP / 2是二进制数据。



4. **服务端推送**

HTTP/2会在用户的浏览器和服务器在建立连接后，服务器主动将一些资源推送给浏览器并缓存起来的机制。有了缓存，当浏览器想要访问已缓存的资源的时候就可以直接从缓存中读取了。

