---
title: http与https
#tags: [http,https]      #添加的标签
categories: 
  - 计算机网络
  #- http
description: http与https的区别与原理
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00699-1913874295.png
---

## http和https的基本概念

- http：超文本传输协议HTTP协议被用于在Web浏览器和网站服务器之间传递信息，HTTP协议以明文方式发送内容，不提供任何方式的数据加密，

- https在http的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。

![http and https](https://gitee.com/hu-zhihong/picbed/raw/master/http%20and%20https.png)

SSL/TLS协议的基本过程是这样的：

1. 客户端向服务器端索要并验证公钥。
2. 双方协商生成“对话密钥”。
3. 双方采用“对话密钥”进行加密通信。



## http和https的区别

HTTPS和HTTP的区别主要如下：

1. https协议需要到ca申请证书，一般免费证书较少，因而需要一定费用。
2. http是超文本传输协议，信息是明文传输，https则是具有安全性的ssl加密传输协议。
3. http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。
4. http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全。





## https的工作原理

客户端在使用HTTPS方式与Web服务器通信时有以下几个步骤，如图所示。

1. 客户使用https的URL访问Web服务器，要求与Web服务器建立SSL连接。

2. Web服务器收到客户端请求后，会将网站的证书信息（证书中包含公钥）传送一份给客户端。

3. 客户端的浏览器与Web服务器开始协商SSL连接的安全等级，也就是信息加密的等级。

4. 客户端的浏览器根据双方同意的安全等级，建立会话密钥，然后利用网站的公钥将会话密钥加密，并传送给网站。

5. Web服务器利用自己的私钥解密出会话密钥。

6. Web服务器利用会话密钥加密与客户端之间的通信。

![https principle](https://gitee.com/hu-zhihong/picbed/raw/master/https%20principle.gif)
