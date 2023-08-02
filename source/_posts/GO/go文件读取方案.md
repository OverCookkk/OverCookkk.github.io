---
title: go文件读取方案
tags: [go]      #添加的标签
categories: 
  - GO
description: 通过以实际不同大小的文件为例，来具体比较下不同读取文件方式的差异。
#cover: 
---



## 整个文件加载

Go 提供了可一次性读取文件内容的方法：os.ReadFile 与 ioutil.ReadFile。在 Go 1.16 开始，ioutil.ReadFile 就等价于 os.ReadFile。

```go
func BenchmarkOsReadFile4KB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  _, err := os.ReadFile("./4KB.txt")
  if err != nil {
   b.Fatal(err)
  }
 }
}

func BenchmarkOsReadFile4MB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  _, err := os.ReadFile("./4MB.txt")
  if err != nil {
   b.Fatal(err)
  }
 }
}

func BenchmarkOsReadFile4GB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  _, err := os.ReadFile("./4GB.txt")
  if err != nil {
   b.Fatal(err)
  }
 }
}

func BenchmarkOsReadFile16GB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  _, err := os.ReadFile("./16GB.txt")
  if err != nil {
   b.Fatal(err)
  }
 }
}
```

一次性加载文件的优缺点非常明显，它能减少 IO 次数，但它会将文件内容都加载至内存中，对于大文件，存在内存撑爆的风险。



## 逐行读取

Go 中 bufio.Reader 对象提供了一个 ReadLine() 方法，但其实我们更多地是使用 ReadBytes('\n') 或者 ReadString('\n') 代替。

```go
func ReadLines(filename string) {
 fi, err := os.Open(filename)
 if err != nil{
  panic(err)
 }
 defer fi.Close()
 reader := bufio.NewReader(fi)
 for {
  _, err = reader.ReadString('\n')
  if err != nil {
   if err == io.EOF {
    break
   }
   panic(err)
  }
 }
}

func BenchmarkReadLines4KB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadLines("./4KB.txt")
 }
}

func BenchmarkReadLines4MB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadLines("./4MB.txt")
 }
}

func BenchmarkReadLines4GB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadLines("./4GB.txt")
 }
}

func BenchmarkReadLines16GB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadLines("./16GB.txt")
 }
}
```



## 块读取

块读取也称为分片读取，这也很好理解，我们可以将内容分成一块块的，每次读取指定大小的块内容。这里，我们将块大小设置为 4KB。

```go
func ReadChunk(filename string) {
 f, err := os.Open(filename)
 if err != nil {
  panic(err)
 }
 defer f.Close()
 buf := make([]byte, 4*1024)
 r := bufio.NewReader(f)
 for {
  _, err = r.Read(buf)
  if err != nil {
   if err == io.EOF {
    break
   }
   panic(err)
  }
 }
}

func BenchmarkReadChunk4KB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadChunk("./4KB.txt")
 }
}

func BenchmarkReadChunk4MB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadChunk("./4MB.txt")
 }
}

func BenchmarkReadChunk4GB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadChunk("./4GB.txt")
 }
}

func BenchmarkReadChunk16GB(b *testing.B) {
 for i := 0; i < b.N; i++ {
  ReadChunk("./16GB.txt")
 }
}
```

汇总结果

```text
BenchmarkOsReadFile4KB-8           92877             12491 ns/op
BenchmarkOsReadFile4MB-8            1620            744460 ns/op
BenchmarkOsReadFile4GB-8               1        7518057733 ns/op
signal: killed

BenchmarkReadLines4KB-8            90846             13184 ns/op
BenchmarkReadLines4MB-8              493           2338170 ns/op
BenchmarkReadLines4GB-8                1        3072629047 ns/op
BenchmarkReadLines16GB-8               1        12472749187 ns/op

BenchmarkReadChunk4KB-8            99848             12262 ns/op
BenchmarkReadChunk4MB-8              913           1233216 ns/op
BenchmarkReadChunk4GB-8                1        2095515009 ns/op
BenchmarkReadChunk16GB-8               1        8547054349 ns/op
```

在本文的测试条件下（每行数据 1KB），对于小对象 4KB 的读取，三种方式差距并不大；在 MB 级别的读取中，直接加载最快，但块读取也慢不了多少；上了 GB 后，块读取方式会最快。

且有一点可以注意到的是，在整个文件加载的方式中，对于 16 GB 的文件数据（测试机器运行内存为 8GB），会内存耗尽出错，没法执行。



## 总结

不管是什么大小的文件，均不推荐整个文件加载的方式，因为它在小文件时的速度优势并没有那么大，相较于安全隐患，不值得选择它。

块读取是优先选择，尤其对于一些没有换行符的文件，例如音视频等。通过设定合适的块读取大小，能让速度和内存得到很好的平衡。且在读取过程中，往往伴随着处理内容的逻辑。每块内容可以赋给一个工作 goroutine 来处理，能更好地并发。