---
layout: post
title: golang在mac下编译linux二进制文件
tags:
  - golang
excerpt_separator: <!--more-->
---

一般来说加了这俩env就够了：

```
GOOS=linux GOARCH=amd64 go build
```

然而有些第三方依赖，比如sqlite3使用了cgo，那边编译时往往需要打开 `CGO_ENABLED`

```
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build
```

如果仍然编译错误，比如我这里的报错部分如下：

```shell
/usr/local/Cellar/go/1.14/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: warning: ignoring file /var/folders/fz/sqyf7ggd3819rb8z18s_ywsc0000gn/T/go-link-911751888/go.o, building for macOS-x86_64 but attempting to link with file built for unknown-unsupported file format ( 0x7F 0x45 0x4C 0x46 0x02 0x01 0x01 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 )
Undefined symbols for architecture x86_64:
  "__cgo_topofstack", referenced from:
      __cgo_26061493d47f_C2func_getnameinfo in 000002.o
      __cgo_26061493d47f_Cfunc_getnameinfo in 000002.o
      __cgo_26061493d47f_C2func_getaddrinfo in 000004.o
      __cgo_26061493d47f_Cfunc_gai_strerror in 000004.o
      __cgo_26061493d47f_Cfunc_getaddrinfo in 000004.o
      __cgo_532c566fb20a_Cfunc_ZSTD_getErrorName in 000018.o
      __cgo_532c566fb20a_Cfunc_ZSTD_isError in 000018.o
      ...
  "_main", referenced from:
     implicit entry/start for main executable
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```
 <!--more-->
看下基本情况：

```shell
$ go version         
go version go1.14 darwin/amd64
$ go env #摘了点儿cgo相关的env
GOARCH="amd64"
GOOS="darwin"
GCCGO="gccgo"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"

$ gcc --version
Configured with: --prefix=/Library/Developer/CommandLineTools/usr --with-gxx-include-dir=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/4.2.1
Apple clang version 11.0.0 (clang-1100.0.33.8)
Target: x86_64-apple-darwin19.5.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
```

默认`CGO_ENABLED=1`时，是用gcc的Target的`x86_64-apple-darwin19.5.0`去编译的。

那么我们需要重新下载gcc。下面官方下载链接[^3]，我没法正常打开。反而是一个爬虫网站[^4]可以……

下载好后，在`~/.bash_profile`文件里加入：

```
export GCC_LINUX64=/usr/local/gcc-4.8.1-for-linux64
export GCC_LINUX32=/usr/local/gcc-4.8.1-for-linux32

export PATH=$PATH:$GCC_LINUX32/bin:$GCC_LINUX64/bin
```

接下来在编译时加上env的`CC=x86_64-pc-linux-gcc`即可[^1]。

```
CC=x86_64-pc-linux-gcc CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build
```

#### Reference
1. [go交叉编译出错(Mac编译Linux)](https://blog.csdn.net/ITqingliang/article/details/103999318)
2. 有人推荐可以用[xgo](https://github.com/karalabe/xgo)编译
[^1]: [How to cross compile Go with CGO programs for a different OS/Arch](https://gist.github.com/steeve/6905542)
[^3]: 下载GCC: [Cross GCC on Mac OS X](http://crossgcc.rts-software.org/doku.php?id=compiling_for_linux)
[^4]: [Mac OSX下编译Linux Kernel Module](http://blog.chinaunix.net/uid-24386829-id-4509506.html)
