+++
title = 'go 源码编译流程'
date = 2025-07-27T22:18:38+08:00
categories = ["go"]
tags = ["go"]
author = 'xhy'
+++

# Go 源码编译流程

Go 编译源代码需要经过编译，链接过程。通过以下示例看 Go (Go 版本为 v1.24.1)语言是如何编译源码的。

首先，代码目录结构如下：
``` bash
➜  demo1 git:(main) ✗ tree ./
./
├── cmd
│   └── app1
│       └── main.go
└── pkg
    └── pkg1
        └── pkg1.go

// main.go
package main

import "github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1"

func main() {
	pkg1.Func1()
}

// pkg1.go
package pkg1

import "fmt"

func Func1() {
	fmt.Println("pkg1.Func1 invoked")
}
```

编译 app1 main.go 为可执行文件：
``` bash
➜  demo1 git:(main) ✗ go build -x -v ./cmd/app1
WORK=/var/folders/xl/5zdz2b514tg_fmsn0k9h0b200000gn/T/go-build2927962297

...
cat >/var/folders/xl/5zdz2b514tg_fmsn0k9h0b200000gn/T/go-build2927962297/b001/importcfg.link << 'EOF' # internal
packagefile github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1=/Users/hxia/Library/Caches/go-build/4c/4c4a8646f05704995bdb9ecacc59745b0c87e54e0b3f134ebc787ffa86349e24-d
...
modinfo "0w\xaf\f\x92t\b\x02A\xe1\xc1\a\xe6\xd6\x18\xe6path\tgithub.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1\nmod\tgithub.com/TroyXia/effective-go-book\tv0.0.0-20250726040104-4c9ceff7dcec+dirty\t\nbuild\t-buildmode=exe\nbuild\t-compiler=gc\nbuild\tCGO_ENABLED=1\nbuild\tCGO_CFLAGS=\nbuild\tCGO_CPPFLAGS=\nbuild\tCGO_CXXFLAGS=\nbuild\tCGO_LDFLAGS=\nbuild\tGOARCH=arm64\nbuild\tGOOS=darwin\nbuild\tGOARM64=v8.0\nbuild\tvcs=git\nbuild\tvcs.revision=4c9ceff7dcecb5af5c6e76869f02e15c5a71d2b2\nbuild\tvcs.time=2025-07-26T04:01:04Z\nbuild\tvcs.modified=true\n\xf92C1\x86\x18 r\x00\x82B\x10A\x16\xd8\xf2"
EOF

mkdir -p $WORK/b001/exe/
cd .
GOROOT='/Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64' /Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/pkg/tool/darwin_arm64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=pie -buildid=CoYhGGCXfSwQX5HjxsS7/ocfceuFgMPhkI7fMX0t9/gBNCu5T0ndxhcTWp_FbS/CoYhGGCXfSwQX5HjxsS7 -extld=clang /Users/hxia/Library/Caches/go-build/4c/4c4a8646f05704995bdb9ecacc59745b0c87e54e0b3f134ebc787ffa86349e24-d
/Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/pkg/tool/darwin_arm64/buildid -w $WORK/b001/exe/a.out # internal
mv $WORK/b001/exe/a.out app1
rm -rf $WORK/b001/
```

`go build` 命令对应用/包进行编译。其中，`-x` 选项打印编译的执行命令，`-v` 选项打印编译时的包名。

从 `go build` 输出信息可以看出，编译主要分为以下几步：
1. 构建工作空间 `WORK` 目录，该目录在编译完会删除。`-work` 选项可以保留工作空间（不过临时的编译文件会被删除）。
2. 构造编译的链接包信息 `importcfg.link`。
3. 链接器 `link` 链接 `importcfg.link` 中的包目标文件（.a 文件）为可执行文件 `a.out`。

上述流程中少了编译过程，那么编译是在哪里发生的呢？

在 `go build` 中添加 `-a` 选项，`-a` 选项会重新编译所依赖的包（部分内容和上述编译输出类似，为减少冗余，这里重点关注编译部分）：
``` bash
cd /Users/hxia/project/effective-go-book/chapter3/demo1
/Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/pkg/tool/darwin_arm64/compile -o $WORK/b013/_pkg_.a -trimpath "$WORK/b013=>" -p internal/byteorder -lang=go1.24 -std -complete -buildid c6rtfNe70CDveX5CsxT3/c6rtfNe70CDveX5CsxT3 -goversion go1.24.4 -c=4 -shared -nolocalimports -importcfg $WORK/b013/importcfg -pack /Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/src/internal/byteorder/byteorder.go
/Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/pkg/tool/darwin_arm64/compile -o $WORK/b015/_pkg_.a -trimpath "$WORK/b015=>" -p internal/coverage/rtcov -lang=go1.24 -std -complete -buildid w3eNR5jLvNw3wS6UVtCI/w3eNR5jLvNw3wS6UVtCI -goversion go1.24.4 -c=4 -shared -nolocalimports -importcfg $WORK/b015/importcfg -pack /Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/src/internal/coverage/rtcov/rtcov.go
/Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/pkg/tool/darwin_arm64/compile -o $WORK/b016/_pkg_.a -trimpath "$WORK/b016=>" -p internal/godebugs -lang=go1.24 -std -complete -buildid n8Me6R799Ql_sSkrNPi9/n8Me6R799Ql_sSkrNPi9 -goversion go1.24.4 -c=4 -shared -nolocalimports -importcfg $WORK/b016/importcfg -pack /Users/hxia/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.24.4.darwin-arm64/src/internal/godebugs/table.go
```

可以看到 `compile` 编译包为 `_pkg_.a` 目标文件。该文件会作为缓存存在于 go env 缓存目录：
``` bash
➜  demo1 git:(main) ✗ go env | grep cache -i
GOCACHE='/Users/hxia/Library/Caches/go-build'

// 进入缓存目录，缓存目录中存放的是编译包的目标文件（静态库，以 -a 结尾）和依赖描述文件（以 -d 结尾，记录包之间的依赖关系和编译参数）
➜  go-build pwd
/Users/hxia/Library/Caches/go-build
➜  go-build cd 04 
➜  04 ls
040220f430a0f504e501643d86f6973881ff5c1ddac97cda7419062ffebf2c9a-d 0466cee1e7f91da1b64696743011bf7315f38d07f501651dd54095fc0f4f25a2-a
```

看到这里大致理解了，之所以刚开始没有出现编译是因为依赖包已经编译好加载到缓存中了，其引用的是缓存中的依赖描述文件 `/Users/hxia/Library/Caches/go-build/4c/4c4a8646f05704995bdb9ecacc59745b0c87e54e0b3f134ebc787ffa86349e24-d`。

## Go 1.9 源码编译流程

如上节所示，在 Go 1.11 版本之后的静态库都是加载到缓存中。这里以 Go 1.9 版本为例更直观的查看编译过程。

首先通过 `go install` 安装包，`go install` 会编译包为静态库。如下：
``` bash
# ./go install -x github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1
...
/root/hxia/go/go/pkg/tool/linux_amd64/compile -o $WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1.a -trimpath $WORK -goversion go1.9.7 -p github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1 -complete -buildid 31988b53e81eee1d2b33fbaf9b29322314015d4a -D _/root/go/src/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1 -I $WORK -pack ./pkg1.go
mkdir -p /root/go/pkg/linux_amd64/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/
cp $WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1.a /root/go/pkg/linux_amd64/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1.a
```

`go install` 调用 `compile` 编译包 `github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1` 为静态库 `$WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1.a`，接着 `cp` 拷贝该静态库到 `/root/go/pkg/linux_amd64/github.com/TroyXia/effective-go-book/chapter3/demo1/pkg/pkg1.a`。

继续编 go 应用查看应用是如何加载 pkg1 静态库的：
``` bash
# ./go build -x -v github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1
WORK=/tmp/go-build326903613
...
/root/hxia/go/go/pkg/tool/linux_amd64/compile -o $WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1.a -trimpath $WORK -goversion go1.9.7 -p main -complete -buildid 4994d4113529a2a372aab06efa535b1ffff36829 -D _/root/go/src/github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1 -I $WORK -I /root/go/pkg/linux_amd64 -pack ./main.go
cd .

/root/hxia/go/go/pkg/tool/linux_amd64/link -o $WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1/_obj/exe/a.out -L $WORK -L /root/go/pkg/linux_amd64 -extld=gcc -buildmode=exe -buildid=4994d4113529a2a372aab06efa535b1ffff36829 $WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1.a
cp $WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1/_obj/exe/a.out app1
```

流程比较清晰，`compile` 编译 `app` 包到静态库 `$WORK/github.com/TroyXia/effective-go-book/chapter3/demo1/cmd/app1.a`(其中包括了应用依赖的 pkg1.a 静态库)。最终通过 `link` 链接静态库成可执行文件 `a.out`。

这里有一个问题是 `app` 依赖的静态库是 `go install` 提前编译好的 `pkg1.a` 还是 `$WORK` 目录下的静态库呢？

我们更新 `pkg1.go` 的内容，查看内容变化后静态库链接的是旧的还是更新后的 `pkg1.a`。
```bash
package pkg

import "fmt"

func Func1() {
	fmt.Println("pkg1.Func1 invoked already")
}
```

编译后运行应用输出 `pkg1.Func1 invoked already`，`go install` 提前编译的静态库并未被链接。

因为在 `link` 链接这里指定的链接目录顺序为 `-L $WORK -L /root/go/pkg/linux_amd64`。链接器现在 `$WORK` 搜索链接的静态库，如果没有找到再到 `/root/go/pkg/linux_amd64` 目录下搜索，而 `go install` 编译的静态库是在 `/root/go/pkg/linux_amd64` 目录下。

调换链接器的链接库目录顺序，手动执行，输出 `pkg1.Func1 invoked`。代码编译用的是老的静态库。

# 静态可执行文件

继续看静态可执行文件。在 macOS 系统编译应用为可执行文件。如下：
``` bash
➜  demo1 git:(main) ✗ go build -o app cmd/app1/main.go 
➜  demo1 git:(main) ✗ ls -la
total 4600
drwxr-xr-x@ 5 hxia  staff      160 Jul 27 21:52 .
drwxr-xr-x@ 3 hxia  staff       96 Jul 26 11:51 ..
-rwxr-xr-x@ 1 hxia  staff  2351842 Jul 27 21:52 app
drwxr-xr-x@ 3 hxia  staff       96 Jul 26 11:52 cmd
drwxr-xr-x@ 3 hxia  staff       96 Jul 26 11:52 pkg
➜  demo1 git:(main) ✗ otool -L app 
app:
        /usr/lib/libSystem.B.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libresolv.9.dylib (compatibility version 0.0.0, current version 0.0.0)
```

`otool` 显示静态可执行文件还需要调用动态库 `libSystem.B.dylib` 和 `libresolv.9.dylib`，这个静态可执行文件看起来不静态啊。

这是由于 macOS 操作系统的缘故，编译的可执行文件并不是完整的静态可执行。

通过 `go build` 编译成 `Linux amd64` 系统的可执行文件 app：
```bash
GOOS=linux GOARCH=amd64 go build -o app cmd/app1/main.go 

// 拷贝 app 到 Linux amd64 机器上

// ldd 查看 app 是否依赖动态库
# ldd app
        不是动态可执行文件
```

在 Linux amd64 机器上可以看到编译出的是完整的静态可执行文件，可以直接运行。

# 参考资料

- [GO笔记之详解GO的编译执行流程](https://zhuanlan.zhihu.com/p/62922404)
- [Go语言精进之路](https://book.douban.com/subject/35720729/)
