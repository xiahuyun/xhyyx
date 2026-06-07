+++
title = 'Go runtime 调度器精讲（一）：Go 程序初始化'
date = 2024-09-11T11:43:38+08:00
categories = ["go"]
tags = ["go", "runtime", "scheduler"]
author = 'xhy'
+++

# 前言

本系列将介绍 Go runtime 调度器。要学好 Go 语言，runtime 运行时是绕不过去的，它相当于一层“操作系统”对我们的程序做“各种类型”的处理。其中，调度器作为运行时的核心，是必须要了解的内容。本系列会结合 Go plan9 汇编，深入到 runtime 调度器的源码层面去看程序运行时，goroutine 协程创建等各种场景下 runtime 调度器是如何工作的。

本系列会运用到 Go plan9 汇编相关的知识，不熟悉的同学可先看看 [这里](https://www.cnblogs.com/xingzheanan/p/18390537) 了解下。

# Go 程序初始化

首先，从一个经典的 `Hello World` 程序入手，查看程序的启动，以及启动该程序时调度器做了什么。

```go
package main

func main() {
	println("Hello World")
}
```

## 准备

程序启动经过编译和链接两个阶段，我们可以通过 `go build -x hello.go` 查看构建程序的过程：
```bash
# go build -x hello.go 
...
// compile 编译 hello.go
/usr/local/go/pkg/tool/linux_amd64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -p main -complete -buildid uHBjeIlqt1oQO9TLC5SE/uHBjeIlqt1oQO9TLC5SE -goversion go1.21.0 -c=3 -nolocalimports -importcfg $WORK/b001/importcfg -pack ./hello.go
...
// link 链接库文件生成可执行文件
/usr/local/go/pkg/tool/linux_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=27kmwBgRtsWy6cL5ofDV/uHBjeIlqt1oQO9TLC5SE/Ye3W7EEwzML-FanTsWbe/27kmwBgRtsWy6cL5ofDV -extld=gcc $WORK/b001/_pkg_.a
```

这里省略了不相关的输出，经过编译，链接过程之后得到可执行文件 `hello`：
```bash
# ls
go.mod  hello  hello.go
# ./hello 
Hello World
```

## 1.2 进入程序

上一节生成了可执行程序 `hello`。接下来进入本文的主题，通过 `dlv` 进入 `hello` 程序，查看在执行 Go 程序时，运行时做了什么。

我们可以通过 `readelf` 查看可执行程序的入口：
```bash
# readelf -h ./hello
ELF Header:
  ...
  Entry point address:               0x455e40
```

省略了不相关的信息，重点看 `Entry point address`，它是进入 `hello` 程序的入口点。通过 `dlv` 进入该入口点：
```bash
# dlv exec ./hello
Type 'help' for list of commands.
(dlv) b *0x455e40
Breakpoint 1 set at 0x455e40 for _rt0_amd64_linux() /usr/local/go/src/runtime/rt0_linux_amd64.s:8
```

可以看到入口点指向的是 `/go/src/runtime/rt0_linux_amd64.s` 中的 `_rt0_amd64_linux()` 函数。

接下来，进入该函数查看启动 Go 程序时，运行时做了什么。

```bash
// c 命令执行到入口点位置
(dlv) c
> _rt0_amd64_linux() /usr/local/go/src/runtime/rt0_linux_amd64.s:8 (hits total:1) (PC: 0x455e40)
Warning: debugging optimized function
     3: // license that can be found in the LICENSE file.
     4:
     5: #include "textflag.h"
     6:
     7: TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
=>   8:         JMP     _rt0_amd64(SB)      // 跳转到 _rt0_amd64

// si 单步执行指令
(dlv) si
> _rt0_amd64() /usr/local/go/src/runtime/asm_amd64.s:16 (PC: 0x454200)
Warning: debugging optimized function
TEXT _rt0_amd64(SB) /usr/local/go/src/runtime/asm_amd64.s
=>      asm_amd64.s:16  0x454200        488b3c24        mov rdi, qword ptr [rsp]
        asm_amd64.s:17  0x454204        488d742408      lea rsi, ptr [rsp+0x8]
        asm_amd64.s:18  0x454209        e912000000      jmp $runtime.rt0_go     // 这里跳转到 runtime 的 rt0_go

// 进入 rt0_go
(dlv) si
> runtime.rt0_go() /usr/local/go/src/runtime/asm_amd64.s:161 (PC: 0x454220)
Warning: debugging optimized function
TEXT runtime.rt0_go(SB) /usr/local/go/src/runtime/asm_amd64.s
=>      asm_amd64.s:161 0x454220        4889f8          mov rax, rdi
        asm_amd64.s:162 0x454223        4889f3          mov rbx, rsi
        asm_amd64.s:163 0x454226        4883ec28        sub rsp, 0x28
        asm_amd64.s:164 0x45422a        4883e4f0        and rsp, -0x10
        asm_amd64.s:165 0x45422e        4889442418      mov qword ptr [rsp+0x18], rax
        asm_amd64.s:166 0x454233        48895c2420      mov qword ptr [rsp+0x20], rbx
```

`rt0_go` 是 runtime 执行 Go 程序的入口。  


```bash
需要补充说明的是：我们使用的 si 显示的是 CPU 单步执行的指令，是 CPU 真正执行的指令。而 Go plan9 汇编是“优化”了的汇编指令，所以会发现 si 显示的输出和 asm_amd64.s 中定义的不一样。在实际分析的时候可以结合两者一起分析。
```

结合着 [asm_amd64.s/rt0_go](https://github.com/golang/go/blob/master/src/runtime/asm_amd64.s) 分析 si 输出的 CPU 指令：
```bash
=>      asm_amd64.s:161 0x454220        4889f8          mov rax, rdi                    // 将 rdi 寄存器中的 argc 移到 rax 寄存器：rax = argc
        asm_amd64.s:162 0x454223        4889f3          mov rbx, rsi                    // 将 rsi 寄存器中的 argv 移到 rbx 寄存器：rbx = argv
        asm_amd64.s:163 0x454226        4883ec28        sub rsp, 0x28                   // 开辟栈空间
        asm_amd64.s:164 0x45422a        4883e4f0        and rsp, -0x10                  // 对齐栈空间为 16 字节的整数倍（因为 CPU 的一组 SSE 指令需要内存地址必须是 16 字节的倍数）
        asm_amd64.s:165 0x45422e        4889442418      mov qword ptr [rsp+0x18], rax   // 将 argc 移到栈空间 [rsp+0x18]
        asm_amd64.s:166 0x454233        48895c2420      mov qword ptr [rsp+0x20], rbx   // 将 argv 移到栈空间 [rsp+0x20]
```

画出栈空间如下图：

![rt0_go stack](/images/go-rt0-stack.jpg)

继续分析:
```bash
(dlv) si
> runtime.rt0_go() /usr/local/go/src/runtime/asm_amd64.s:170 (PC: 0x454238)
Warning: debugging optimized function
        asm_amd64.s:166 0x454233        48895c2420              mov qword ptr [rsp+0x20], rbx
=>      asm_amd64.s:170 0x454238        488d3d815b0700          lea rdi, ptr [runtime.g0]           // 将 runtime.g0 的地址移到 rdi 寄存器中，rdi = &g0
        asm_amd64.s:171 0x45423f        488d9c240000ffff        lea rbx, ptr [rsp+0xffff0000]       // 将 [rsp+0xffff0000] 地址的值移到 rbx 中，后面会讲
        asm_amd64.s:172 0x454247        48895f10                mov qword ptr [rdi+0x10], rbx       // 将 rbx 中的地址，移到 [rdi+0x10]，实际是移到 g0.stackguard0
        asm_amd64.s:173 0x45424b        48895f18                mov qword ptr [rdi+0x18], rbx       // 将 rbx 中的地址，移到 [rdi+0x18]，实际是移到 g0.stackguard1
        asm_amd64.s:174 0x45424f        48891f                  mov qword ptr [rdi], rbx            // 将 rbx 中的地址，移到 [rdi]，实际是移到 g0.stack.lo
        asm_amd64.s:175 0x454252        48896708                mov qword ptr [rdi+0x8], rsp        // 将 rsp 中的地址，移到 [rdi+0x8]，实际是移到 g0.stack.hi
```

指令中 [runtime.g0](https://github.com/golang/go/blob/master/src/runtime/runtime2.go) 为运行时主线程提供运行的执行环境，它并不是执行用户代码的 goroutine。

使用 `regs` 查看寄存器 `rbx` 存储的是什么：
```bash
(dlv) regs
    Rip = 0x000000000045423f
    Rsp = 0x00007ffec8d155f0
    Rbx = 0x00007ffec8d15628

(dlv) si
> runtime.rt0_go() /usr/local/go/src/runtime/asm_amd64.s:172 (PC: 0x454247)
Warning: debugging optimized function
        asm_amd64.s:171 0x45423f        488d9c240000ffff        lea rbx, ptr [rsp+0xffff0000]
=>      asm_amd64.s:172 0x454247        48895f10                mov qword ptr [rdi+0x10], rbx

(dlv) regs
    Rip = 0x0000000000454247
    Rsp = 0x00007ffec8d155f0
    Rbx = 0x00007ffec8d055f0
```

可以看到，这段指令实际指向的是一段栈空间，`rsp:0x00007ffec8d155f0` 指向的是栈底，`rbx:0x00007ffec8d055f0` 指向的是栈顶，它们的内存空间是 64KB。

根据上述分析，画出栈空间布局如下图：

![g0 stack](/images/go-g0-stack.jpg)

继续往下分析，省略一些不相关的汇编代码。直接从 `asm_amd64.s/runtime·rt0_go:258` 开始看：
```bash
258	    LEAQ	runtime·m0+m_tls(SB), DI
259	    CALL	runtime·settls(SB)
260
261	    // store through it, to make sure it works
262	    get_tls(BX)
263	    MOVQ	$0x123, g(BX)
264	    MOVQ	runtime·m0+m_tls(SB), AX
265	    CMPQ	AX, $0x123
266	    JEQ 2(PC)
267	    CALL	runtime·abort(SB)
```

`dlv` 打断点，进入 258 行汇编指令处：
```bash
(dlv) b /usr/local/go/src/runtime/asm_amd64.s:258
Breakpoint 2 set at 0x4542cb for runtime.rt0_go() /usr/local/go/src/runtime/asm_amd64.s:258
(dlv) c
(dlv) si
> runtime.rt0_go() /usr/local/go/src/runtime/asm_amd64.s:259 (PC: 0x4542d2)
Warning: debugging optimized function
        // 将 [runtime.m0+136] 地址移到 rdi，rdi = &runtime.m0.tls
        asm_amd64.s:258 0x4542cb*       488d3d565f0700                  lea rdi, ptr [runtime.m0+136]
        // 调用 runtime.settls 设置线程本地存储
=>      asm_amd64.s:259 0x4542d2        e809240000                      call $runtime.settls
        // 将 0x123 移到 fs:[0xfffffff8]
        asm_amd64.s:263 0x4542d7        6448c70425f8ffffff23010000      mov qword ptr fs:[0xfffffff8], 0x123
        // 将 [runtime.m0+136] 的值移到 rax 寄存器中
        asm_amd64.s:264 0x4542e4        488b053d5f0700                  mov rax, qword ptr [runtime.m0+136]
        // 比较 rax 寄存器的值是否等于 0x123，如果不等于则执行 call $runtime.abort
        asm_amd64.s:265 0x4542eb        483d23010000                    cmp rax, 0x123
        asm_amd64.s:266 0x4542f1        7405                            jz 0x4542f8
        asm_amd64.s:267 0x4542f3        e808040000                      call $runtime.abort
```

这段指令涉及到线程本地存储的知识。线程本地存储（TLS）是一种机制，允许每个线程有自己独立的一组变量，即使这些变量在多个线程之间共享相同的代码。在 Go runtime 中，每个操作系统线程（M）都需要知道自己当前正在执行哪个 goroutine（G）。为了高效地访问这些信息，Go runtime 使用 TLS 来存储 G 的指针。这样每个线程都可以通过 TLS 快速找到自己当前运行的 G。m0 是 Go 程序启动时的第一个操作系统线程，并且负责初始化整个 Go runtime。在其他线程通过 Go runtime 的调度器创建时，调度器会自动为它们设置 TLS，并将 G 的指针写入 TLS。但 m0 是一个特殊的线程，它直接由操作系统创建，而没有经过 Go 调度器，因此需要通过汇编指令设置 TLS。

这段指令的逻辑是将 `runtime.m0.tls` 的地址送到 `rdi` 寄存器中，接着调用 [runtime.settls](https://github.com/golang/go/blob/master/src/runtime/sys_linux_amd64.s#L637) 设置 fs 段基址寄存器的值，使得通过段基址和偏移量就能访问到 `m0.tls`。最后验证设置的 `[段基址：偏移量]` 能否正确的访问到 `m0.tls`，将 `0x123` 传到 `[段基址：偏移量]`，这时如果访问正确，应该传给的是 `m0.tls[0] = 0x123`，然后将 `[runtime.m0+136]` 的内容，即 `m0.tls[0]` 拿出来移到 `rax` 寄存器做比较，如果一样，则说明通过 `[段基址：偏移量]` 可以正确访问到 `m0.tls`，否则调用 `runtime.abort` 退出 `runtime`。

每个线程都有自己的一组 CPU 寄存器值，不同的线程通过不同的段 fs 基址寄存器私有的存储全局变量。更详细的信息请参考 [Go语言调度器源代码情景分析之十：线程本地存储](https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483755&idx=1&sn=50d8d1f4df9ce4a8a0f366ac0b7a6fc6&scene=19#wechat_redirect)。


```bash
为加深这块理解，我们从汇编角度看具体是怎么设置的。

asm_amd64.s:258 0x4542cb*       488d3d565f0700                  lea rdi, ptr [runtime.m0+136]
=> rdi = &runtime.m0.tls = 0x00000000004ca228

asm_amd64.s:259 0x4542d2        e809240000                      call $runtime.settls
=> 设置的是 Fs_base 段基址寄存器的值，regs 查看 Fs_base=0x00000000004ca230

asm_amd64.s:263 0x4542d7        6448c70425f8ffffff23010000      mov qword ptr fs:[0xfffffff8], 0x123
=> fs:[0xfffffff8]，fs 是段基址，实际是 Fs_base 段基址寄存器的值，[0xfffffff8] 是偏移量。fs:[0xfffffff8] = 0x00000000004ca230:[0xfffffff8] = 0x00000000004ca228
=> 实际通过段基址寄存器 fs:[0xfffffff8] 访问的内存地址就是 m0.tls 的地址 0x00000000004ca228
```

继续往下执行：
```bash
=>      asm_amd64.s:271 0x4542f8        488d0dc15a0700                  lea rcx, ptr [runtime.g0]               // 将 runtime.g0 的地址移到 rcx，rcx = &runtime.g0
        asm_amd64.s:272 0x4542ff        6448890c25f8ffffff              mov qword ptr fs:[0xfffffff8], rcx      // 将 rcx 移到 m0.tls，实际是 m0.tls[0] = &runtime.g0
        asm_amd64.s:273 0x454308        488d05915e0700                  lea rax, ptr [runtime.m0]               // 将 runtime.m0 的地址移到 rax，rax = &runtime.m0
        asm_amd64.s:276 0x45430f        488908                          mov qword ptr [rax], rcx                // 将 runtime.g0 的地址移到 runtime.m0，实际是 runtime.m0.g0 = &runtime.g0
        asm_amd64.s:278 0x454312        48894130                        mov qword ptr [rcx+0x30], rax           // 将 runtime.m0 的地址移到 runtime.g0.m，实际是 runtime.g0.m = &runtime.m0
```

上述指令做的是关联主线程 `m0` 和 `g0`，这样 `m0` 就有了运行时执行环境。画出内存布局如下图：

![init stack](/images/go-init-stack.jpg)

# 小结

至此，我们的程序初始化部分就告一段落了，下一篇将正式进入调度器的部分。
