---
title: UPX 压缩 go build 可执行文件
top: false
cover: false
toc: true
mathjax: false
date: 2021-09-02 10:46:55
password:
summary: Go 编译优化可执行文件
categories: 编程语言
tags:
- Golang
- build
---

Go 是静态编译语言，`go build`正常编译时会将依赖以及调试信息等都打包进去，不需要运行环境支持，但是这样一来，可执行文件体积就比较大，怎么解决呢，通过查询资料，有两种方法可以进一步减小可执行文件的体积。

## 设置编译参数

`go build` 命令提供有可选编译参数，其中 `-ldflags` 选项将 `go tool link` 的参数传递给 `go build`，其中有下面两个参数，可以减少编译后的体积

```bash
$ go tool link --help
usage: link [options] main.o
  ...
  -s    disable symbol table      # 去掉符号表信息
  -w    disable DWARF generation  # 去掉 DWARF 调试信息，不能进行 gdb 调试 
  ...
```

实际测试下：

```bash
# 不添加编译参数
$ go build
# 查看大小
$ ll sql2md
-rwxrwxr-x 1 mazhuang mazhuang 9.8M Sep  2 15:00 sql2md

# 添加编译参数 -w
$ go build -ldflags="-w"
$ ll sql2md             
-rwxrwxr-x 1 mazhuang mazhuang 8.1M Sep  2 15:08 sql2md

# 添加编译参数 -s
$ go build -ldflags="-s"
$ ll sql2md
-rwxrwxr-x 1 mazhuang mazhuang 7.3M Sep  2 15:08 sql2md

# 添加编译参数 -w -s
$ go build -ldflags="-w -s" 
$ ll sql2md
-rwxrwxr-x 1 mazhuang mazhuang 7.3M Sep  2 15:02 sql2md
```

可以看到体积有了缩减，但是并不是很明显，其中 `-s` 和 `-w -s` 选项编译后的体积的是一样的，且体积最小

## 使用 UPX 压缩

[UPX(The Ultimate Packer for eXecutables)](https://github.com/upx/upx) 是开源的可执行文件压缩软件，UPX 通常会将程序和 DLL 的文件大小减少约 50%-70%，从而 减少磁盘空间、网络加载时间、下载时间和其他分销和存储成本。

UPX 压缩的程序和库是完全独立的并完全像以前一样运行，大多数情况下没有运行时或内存损失支持的格式。

### 安装

```bash
# For CentOS/Fedora/RHEL
$ sudo yum install upx
# For Ubuntu
$ sudo apt install upx
```

### 压缩测试

不使用编译参数，编译后压缩

```bash
$ go build && upx sql2md 
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2017
UPX 3.94        Markus Oberhumer, Laszlo Molnar & John Reiser   May 12th 2017

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
  10190160 -$   5405412   53.05%   linux/amd64   sql2md                        

Packed 1 file.

$ ll sql2md
-rwxrwxr-x 1 mazhuang mazhuang 5.2M Sep  2 17:08 sql2md
```

重新编译携带编译参数

```bash
$ go build -ldflags="-s" && upx sql2md
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2017
UPX 3.94        Markus Oberhumer, Laszlo Molnar & John Reiser   May 12th 2017

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   7620264 ->   2833612   37.19%   linux/amd64   sql2md                        

Packed 1 file.

$ ll sql2md
-rwxrwxr-x 1 mazhuang mazhuang 2.8M Sep  2 17:19 sql2md
```

带参数编译后再压缩，体积从 9.8 缩小到 2.8, 减少了约 71.4%，有了很明显的改善
