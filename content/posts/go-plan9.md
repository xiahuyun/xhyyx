+++
title = 'Go plan9 汇编：内存对齐和递归'
date = 2024-09-02T10:29:38+08:00
categories = ["go"]
tags = ["go", "plan9", "汇编"]
author = 'xhy'
+++

# 前言

在 Go plan9 汇编系列文章中，介绍了函数和函数栈的调用。这里继续看内存对齐和递归调用方面的内容。

# 内存对齐

直接上示例：  
```go
type temp struct {
	a bool
	b int16
	c []string
}

func main() {
	var t = temp{a: true, b: 1, c: []string{}}
	fmt.Println(unsafe.Sizeof(t))
}
```

输出：  
```bash
32
```

改写 temp 结构体成员变量位置：
```go
type temp struct {
	a bool
    c []string
	b int16
}

func main() {
	var t = temp{a: true, b: 1, c: []string{}}
	fmt.Println(unsafe.Sizeof(t))
}
```

输出：
```bash
40
```

为什么移动下结构体成员的位置会对结构体在内存中的大小有影响呢？


打印示例中结构体成员变量地址如下：  
```go
# 示例 1
func main() {
	var t = temp{a: true, b: 1, c: []string{}}

	fmt.Println(unsafe.Sizeof(t.a), unsafe.Sizeof(t.b), unsafe.Sizeof(t.c))
	fmt.Printf("%p %p %p %p\n", &t, &t.a, &t.b, &t.c)

	fmt.Println(unsafe.Sizeof(t))
}

# go run ex10.go 
1 2 24
0xc0000a4000 0xc0000a4000 0xc0000a4002 0xc0000a4008
32

# 示例 2
func main() {
	var t = temp{a: true, b: 1, c: []string{}}

	fmt.Println(unsafe.Sizeof(t.a), unsafe.Sizeof(t.c), unsafe.Sizeof(t.b))
	fmt.Printf("%p %p %p %p\n", &t, &t.a, &t.c, &t.b)

	fmt.Println(unsafe.Sizeof(t))
}

# go run ex10.go 
1 24 2
0xc00006e090 0xc00006e090 0xc00006e098 0xc00006e0b0
40
```

可以看到，在为结构体分配内存时是要遵循内存对齐的，内存对齐是为了简化寻址，CPU 可一次找到变量的位置。因为内存对齐的存在，这里示例 2 中虽然变量 a 只占 1 个字节，但却独占了 8 个字节，这对于写代码来说是一种内存消耗，应当避免的。

# 递归

我们看一个递归的示例：
```go
func main() {
	println(sum(1000))
}

//go:nosplit
func sum(n int) int {
	if n > 0 {
		return n + sum(n-1)
	} else {
		return 0
	}
}
```

输出：
```bash
# go run ex7.go 
# command-line-arguments
main.sum: nosplit stack over 792 byte limit
main.sum<1>
    grows 24 bytes, calls main.sum<1>
    infinite cycle
```

这里我们在 `sum` 函数前加 `//go:nosplit` 是要声明这个函数是不可栈分裂的函数。意味着当函数栈满的时候，（内存分配器）不会为它开辟新的空间。

Go 为 goroutine 分配的初始栈空间大小为 2K，如果  main 栈加上 nosplit 的 sum 栈超过 2K，将导致爆栈。

将 `//go:nosplit` 拿掉，重新执行：
```go
func main() {
	println(sum(100000))
}

func sum(n int) int {
	if n > 0 {
		return n + sum(n-1)
	} else {
		return 0
	}
}
```

输出：
```bash
5000050000
```

那么 `sum` 是否可以无限递归呢？我们给 `sum` 一个大数 10000000000000，接着重新执行：
```bash
runtime: goroutine stack exceeds 1000000000-byte limit
runtime: sp=0xc0200f8398 stack=[0xc0200f8000, 0xc0400f8000]
fatal error: stack overflow
```

输出 `stack overflow`，main 协程的栈是从 0xc0200f8000 到 0xc0400f8000，这里递归所用的栈超过了 goroutine 栈的最大限制 `1000000000-byte`（超过的意思是 main 栈加上 sum 递归调用的栈超过了最大限制），也就是 1G。

