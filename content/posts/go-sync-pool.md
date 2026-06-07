+++
title = 'go sync.pool 学习笔记 '
date = 2025-11-09T18:35:38+08:00
categories = ["go"]
tags = ["go", "对象池"]
author = 'xhy'
+++

# 概述

`sync.pool`  对象池可以用来复用临时对象，减少内存压力，降低 GC 压力。

# 示例

## 基本用法

``` go
type Worker struct{}  
  
func (w *Worker) Name() string {  
    return "worker"  
}  
  
func main() {  
    workerPool := sync.Pool{New: func() interface{} {  
       return Worker{}  
    }}  
  
    worker := workerPool.Get().(Worker)  
    defer workerPool.Put(worker)  
  
    name := worker.Name()  
    fmt.Println(name)  
}
```

`sync.pool` 是单对象池，不是多对象池。基本使用方法是 `Get` 和 `Put` 方法，`Get` 用来从对象池中取对象，`Put` 用来将不用的对象放回对象池中。

## 适用场景

`sync.pool` 中的对象可能会被运行时回收。有可能在需要使用时对象被回收而重新创建。因此，`sync.pool` 适合存储高频创建，作用时间短的对象。比如以下场景：
- JSON 处理：频繁分配的 []byte 切片；
- Web 服务：HTTP 请求处理的缓冲区；
- 数据库操作：连接池的辅助工具；

[Go sync.Pool 的陷阱与正确用法：从踩坑到最佳实践](https://juejin.cn/post/7480797795785195520) 这篇文章写的很好关于 `sync.pool` 的陷阱和正确用法，可以参考学习，这里就不赘述了。

## 性能测试

``` go
var globalBuf []byte  
  
func BenchmarkAllocateWithoutPool(b *testing.B) {  
    for i := 0; i < b.N; i++ {  
       buf := make([]byte, 1024)  
       globalBuf = buf     // 这里将 buf 赋值给 globalBuf，不然会内存逃逸
    }  
}  
  
func BenchmarkAllocateWithPool(b *testing.B) {  
    pool := sync.Pool{New: func() interface{} { return make([]byte, 1024) }}  
    b.ResetTimer()  
    for i := 0; i < b.N; i++ {  
       buf := pool.Get().([]byte)  
       pool.Put(buf)  
    }  
}
```

测试结果：
``` bash
 go test -bench . -benchmem
goos: darwin
goarch: arm64
pkg: go-by-example/sync/pool
cpu: Apple M3
BenchmarkAllocateWithoutPool-8           8672882               137.2 ns/op          1024 B/op          1 allocs/op
BenchmarkAllocateWithPool-8             36728509                31.91 ns/op           24 B/op          1 allocs/op
PASS
ok      go-by-example/sync/pool 3.047s
```

Go Benchmark的输出格式为：
``` bash
BenchmarkName-GOMAXPROCS Iterations TimePerOp(ns/op) BytesPerOp(B/op) AllocsPerOp(allocs/op) 
```

- **TimePerOp**：单次操作耗时（纳秒），越小越快。
- **AllocsPerOp**：单次操作的内存分配次数，越小对GC越友好。
- **BytesPerOp**：单次操作分配的总字节数，越小内存效率越高。

可以看出，使用 `sync.pool` 对象池相比于不使用 `sync.pool` 的性能对比：
- 单次操作耗时占比：31.91 / 137.2 = 23.2%
- 单次操作分配内存：24/1024 = 2.3%

## 并发

我们进一步看并发场景下对象复用是什么情况。

**非并发场景**

首先看非并发场景对象复用情况。示例如下：
``` go
type Worker struct{}  
  
func (w *Worker) Name() string {  
    return "worker"  
}

func main() {  
    runtime.GOMAXPROCS(4)
    
	var createWorkerTime int32  
	workerPool := sync.Pool{New: func() interface{} {  
	    atomic.AddInt32(&createWorkerTime, 1) 
	    return Worker{}  
	}}
	
	currencyCount := 1024 * 1
    for i := 0; i < currencyCount; i++ {  
       worker := workerPool.Get().(Worker)  
       time.Sleep(time.Millisecond * 1)  
       workerPool.Put(worker)  
    }  
  
    fmt.Println("create worker time: ", atomic.LoadInt32(&createWorkerTime))  
}
```

输出：
``` bash
create worker time:  1
```

这里对象只创建了一次。

需要注意的是，`sync.pool` 的 `Get` 和 `Put` 是并发安全的。但是创建对象并不是并发安全的，需要用户自己实现。因此，在 `sync.pool.New` 中使用 `atomic.AddInt32` 原子操作并发安全的更新 `createWorkerTime` 变量。

**并发场景**

示例如下：
``` go
func main() {  
    runtime.GOMAXPROCS(4)  
  
    var createWorkerTime int32  
    workerPool := sync.Pool{New: func() interface{} {  
       atomic.AddInt32(&createWorkerTime, 1)  
       return Worker{}  
    }}  
  
    currencyCount := 1024 * 1  
    var wg sync.WaitGroup  
    for i := 0; i < currencyCount; i++ {  
       wg.Add(1)  
       go func(i int) {  
          defer wg.Done()  
          worker := workerPool.Get().(Worker)  
          defer workerPool.Put(worker)
	    }(i)  
	}  
  
	wg.Wait()  
	fmt.Println("create worker time: ", atomic.LoadInt32(&createWorkerTime))
}
```

输出：
``` bash
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  4
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  4
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  4
```

我们仅调整 `runtime.GOMAXPROCS` 为 8，运行程序输出：
``` bash
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  5
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  6
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  7
```

当调整 `runtime.GOMAXPROCS` 时，对象创建次数不固定。要解释其中发什么了什么，需要从 GPM 入手，`runtime.GOMAXPROCS` 设置 P 的个数，P 会调度 G 到线程 M 上运行，而对象是 P 私有的，如果 G 上的 P 没有对象，则会创建对象。这也解释了，为什么 P 变多了会影响对象的复用次数。

继续构造示例如下：

``` go 
currencyCount := 1024 * 1  
var wg sync.WaitGroup  
for i := 0; i < currencyCount; i++ {  
    wg.Add(1)  
    go func(i int) {  
       defer wg.Done()  
       worker := workerPool.Get().(Worker)  
       defer workerPool.Put(worker)  
       time.Sleep(time.Millisecond * 100)  
    }(i)  
}
```

我们在协程内加了 `time.Sleep(time.Millisecond * 100)` 运行三次程序：
``` bash
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  1024
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  1024
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  1024
```

更新 `runtime.GOMAXPROCS` 在此运行三次程序，结果都是 `1024`。

这是为什么呢？

还是和 GPM 有关，对象是 P 私有的，P 调度 G 到协程 M 上运行，如果 P 有对象，则会将对象给 G，将私有的对象置为 nil，下次分配对象时如果没有对象，则会调用 `sync.pool.New` 创建对象。

这里 G 拿到 P 的私有对象后，在线程 M 上运行。由于设置了 `time.Sleep` G 陷入阻塞状态，M 会运行下一个 G，下一个 G 发现 P 的私有对象已经被阻塞的 G 拿掉了，又会调用 `sync.pool.New` 创建对象。如此重复，导致每次对象都在创建。

基于这样的逻辑，我们在构造示例如下：
``` go
currencyCount := 1024 * 1  
var wg sync.WaitGroup  
for i := 0; i < currencyCount; i++ {  
    wg.Add(1)  
    go func(i int) {  
       defer wg.Done()  
       worker := workerPool.Get().(Worker)  
       defer workerPool.Put(worker)  
       name := worker.Name()  
       fmt.Println("worker name: ", name, "currency id: ", i)  
    }(i)  
}
```

输出：
``` bash
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  980
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  930
~/project/go-by-example/sync/pool git:[main] go run main.go
create worker time:  884
```

这里没有用 `time.Sleep` 使 G 陷入阻塞，而是打印对象的名字。输出的对象创建次数并不固定。

这是因为在有些 P 上， 当前 G 执行完将对象 `Put` 归还给 P 了，下一个 G 会从 P 上拿到对象。而有些 P，当前 G 并未将对象归还给 P，而下一个 G 又找 P 要对象，触发创建对象逻辑，导致每次运行创建对象的次数都不一样。

# 小结

本文介绍了 `sync.pool` 的使用，性能分析及并发场景下的对象复用情况，对于 `sync.pool` 的原理级了解还是要从源码层面入手。

# 参考资料

- [Go sync.Pool 的陷阱与正确用法：从踩坑到最佳实践](https://juejin.cn/post/7480797795785195520)
- [Go sync.Pool](https://geektutu.com/post/hpg-sync-pool.html)

