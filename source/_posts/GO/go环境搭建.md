---
title: go环境搭建
#tags: [hexo建站]      #添加的标签
categories: 
  - GO
description: 
#cover: 
---



## go中gRPC的环境搭建

首先需要安装 gRPC golang版本的软件包。 安装官方安装命令：

```text
go get google.golang.org/grpc
```

是会报错误的，正确的安装方式：

```text
# 下载grpc-go
git clone https://github.com/grpc/grpc-go.git $GOPATH/src/google.golang.org/grpc
# 下载golang/net
git clone https://github.com/golang/net.git $GOPATH/src/golang.org/x/net
# 下载golang/text
git clone https://github.com/golang/text.git $GOPATH/src/golang.org/x/text
# 下载sys
git clone https://github.com/golang/sys.git $GOPATH/src/golang.org/x/sys
# 下载go-genproto
git clone https://github.com/google/go-genproto.git $GOPATH/src/google.golang.org/genproto
# 下载go-protobuf
git clone https://github.com/protocolbuffers/protobuf-go.git  $GOPATH/src/google.golang.org/protobuf

# 下载完以上的内容后，还需下载下面说的Go Plugins（protoc-gen-go），才能开始安装grpc
cd $GOPATH/src/
go install google.golang.org/grpc
```



### 安装protoc编译器

protoc 用于编译 protocolbuf (.proto文件) 和 protobuf 运行时。

### 下载

安装编译器最简单的方式是去[protobuf仓库地址](https://link.zhihu.com/?target=https%3A//github.com/protocolbuffers/protobuf/releases)下载预编译好的 protoc 二进制文件，仓库中可以找到每个平台对应的编译器二进制文件。

解压后进入bin文件夹中，window下，找到可执行文件：protoc.exe 。然后将它放入到$GOROOT的bin中。

linux下，

新增环境变量

```text
$ vim /etc/profile 
```

增加以下内容

```text
#记得改成自己的路径
export PATH=$PATH:/data/jbchen5/src/protoc-3.18.0-linux-x86_64/bin
```

使其生效

```text
$ source /etc/profile
```

最后查看是否成功

```text
$ protoc --version
libprotoc 3.18.0
```



### 安装Go Plugins（protoc-gen-go）

除了安装 protoc 之外还需要安装各个语言对应的编译插件，这里我们用的Go 语言，所以还需要安装一个 Go 语言的编译插件。

```text
git clone https://github.com/golang/protobuf.git  $GOPATH/src/github.com/golang/protobuf
cd $GOPATH/src/
go install github.com/golang/protobuf/protoc-gen-go/
```

> 编译器插件 protoc-gen-go 将安装在默认位于GOPATH/bin下面，请务必将protoc-gen-go放到$GOROOT的bin中。





protoc -I . --go_out=plugins=grpc:. ./QueryUser.proto

go mod init 名字    该名字必须与文件夹名字一样，不然包含PB会包含不到





## go常用小工具



### goland环境搭建插件配置

#### 格式化protobuf工具LLVM

1. 下载工具格式化工具，地址如下：http://releases.[llvm](https://so.csdn.net/so/search?q=llvm&spm=1001.2101.3001.7020).org/download.html

2. 配置clang-format：

   在设置->工具->File Watcher里，新建一个File Watcher，然后选择
   
   文件类型：Protocol Buffer
   
   程序：选择LLVS里bin目录下的clang-format.exe
   
   参数：如下
   
   ```text
   -style="{BasedOnStyle: Google, IndentWidth: 4, ColumnLimit: 0, AlignConsecutiveAssignments: true, AlignConsecutiveAssignments: true}"
   -i
   $FilePath$
   ```
   注意：如果你是Windows，clang-format要选择clang-format.exe



### go结构体字节对齐工具

go官方提供了fieldalignment 检测和修复工具，可以快速的帮我们解决内存对齐扣字节的问题。

- 安装fieldalignment：
   `go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest`
   
- 直接输入`fieldalignment`会输出帮助信息。字节对齐命令如下：
   `fieldalignment -fix xxx`
   说明：参数 -fix 会自动帮我们做到内存对齐，不需要我们做额外的操作；
   参数 xxx 是我们的需要fix的文件的路径。
   注意：进行字节对齐后，原有的struct中的注释会消失，需要手动恢复。
   
- 字节对齐后会输出如下信息：

   ```text
   D:\golang\src\rerank_service\model\RerankInfo.go:14:24: struct with 104 pointer bytes could be 72
   D:\golang\src\rerank_service\model\RerankInfo.go:33:15: struct of size 72 could be 64
   ```

   





