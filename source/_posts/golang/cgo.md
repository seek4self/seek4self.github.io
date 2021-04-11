---
title: cgo 的魔法
top: false
cover: false
toc: true
mathjax: true
summary: c 和 go 之间的交互
categories: 编程语言
tags:
  - Golang
  - cgo
  - unsafe
date: 2021-04-10 10:45:02
password:
---

## cgo 访问 c 标准库

当我们需要用 go 调用 c 语言写的接口时，就需要用 `cgo` 进行连接，`cgo` 属于 go 语言的一个特殊工具，直接通过一个例子来看

```go
package main

/*         
#include <stdio.h>
#include <stdlib.h>
*/
import "C"     
// 上面注释部分是导入 c 的标准库头文件， 必须用注释包裹在内且在 import "C" 上面一行，
// go编译器读到上面的注释时，会自行跳转至 gcc 编译, 所以注释中不能出现除 c 语言以外的语法
// "C" 并不是 go 的基础包，相当于一个虚拟包，import 相当于一个启用开关，该句表示该程序已经启用 cgo 的特性，并且只能单独一行，不能和其他包一块导入

import "unsafe"    
// unsafe 包如其名，并不安全，该包常用于 c 和 go 之间的指针转换和地址运算         

func main() {
  // CString() 是 go 内部的接口，可以转换 go 的字符串为 c 语言字符串格式 char*
  cs := C.CString("Hello, World\n")
  // 调用c接口方式为 C.api(), 
  // 因为 CString 使用 c 的 malloc 申请空间，go GC 不会清理，用完后需要手动 free
  // unsafe.Pointer 相当于 c 的 void*，这里做了强制转换
  defer C.free(unsafe.Pointer(cs))
  C.puts(cs)
}
```

想要看具体的实现，可以通过命令行 `go tool cgo [filename.go]` 查看编译过程中间生成的文件，看到具体的类型转换，更多 `cgo` 命令用法见[官方cgo](https://golang.org/cmd/cgo/#hdr-Using_cgo_directly)

## cgo 类型转换

go 传给 c 的参数都必须使用 `C.cType` 转换成 c 的类型，这里着重谈一下 `string` 的转换，更多类型转换请阅读 [Go语言高级编程](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-03-cgo-types.html)

因为 c 并没有 `string` 类型，只有 `char[]`, 所以 go 内置了 `string` 的转换函数，如下所示，可以根据需要自行选择

```go
// Go string to C string
// The C string is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be
// freed, such as by calling C.free (be sure to include stdlib.h
// if C.free is needed).
func C.CString(string) *C.char

// Go []byte slice to C array
// The C array is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be
// freed, such as by calling C.free (be sure to include stdlib.h
// if C.free is needed).
func C.CBytes([]byte) unsafe.Pointer

// C string to Go string
func C.GoString(*C.char) string

// C data with explicit length to Go string
func C.GoStringN(*C.char, C.int) string

// C data with explicit length to Go []byte
func C.GoBytes(unsafe.Pointer, C.int) []byte
```

上述的函数，均是通过拷贝的方式进行数据转换，将 go `string`转为 c `char*`时，使用 `malloc` 申请内存空间并拷贝数据到对应的地址，通过手动调用 `free` 释放内存；将 c `char*` 转换为 `string` 时，go 自行申请内存空间，并将 c 的数据拷贝出来，内存交由 `GC` 处理。通过拷贝 可以使 go 的内存和 c 的内存互不干扰，简化了内存管理和接口调用，但是内存申请和拷贝也增加了性能的损耗。  

除了上述字符串、字节数组以外的类型转换，都是 c 和 go 共用地址，取决于内存是谁申请的，如下面代码所示：

```go
package main

/*
int arr[10];
void add(int *a, int n) {
  for (int i = 0; i< n; i++) {
    printf("%d ",a[i]);
    a[i]++;
  }
}
*/
import "C"
import (
  "fmt"
  "unsafe"
)

func main() {
  arr1 := (*[10]int32)(unsafe.Pointer(&C.arr[0]))[:10:10] // 将 C.arr 转换成[10]int32 数组，然后从数组中截取长度和容量均为10切片，
  arr1[1] = 123           // 修改会直接修改 c 的内存
  fmt.Printf("%+v %[1]T %v\n", arr1, C.arr) // [0 123 0 0 0 0 0 0 0 0]   []int32 [0 123 0 0 0 0 0 0 0 0]
  C.arr[2] = C.int(456)
  fmt.Printf("%+v %[1]T %v\n", arr1, C.arr) // [0 123 456 0 0 0 0 0 0 0] []int32 [0 123 456 0 0 0 0 0 0 0]
  arr1 = append(arr1, 1)  // 这里 arr1 切片已经扩容，重新分配内存，脱离了 C.arr
  arr1[3] = 1
  C.arr[3] = C.int(789)
  fmt.Printf("%+v %v len(arr1):%d cap(arr1):%d\n", arr1, C.arr, len(arr1), cap(arr1)) // [0 123 456 1 0 0 0 0 0 0 1] [0 123 456 789 0 0 0 0 0 0] len(arr1):11 cap(arr1):20

  a := []int32{1, 2, 3}
  C.add((*C.int)(unsafe.Pointer(&a[0])), C.int(len(a)))
  fmt.Println(a)  // [2 3 4]
}
```

所以需要额外注意, go 传给 c 的参数若是像 `slice`、`map` 等内存可能发生改变的类型，尽可能不要发生扩容现象，或者申请内存时，直接用 `C.malloc()` 申请固定大小 c 的地址，确保内存不会发生改变,

## unsafe.Pointer

`unsafe` 包是 go 提供的针对类型安全操作，具体接口查看[官网doc](https://pkg.go.dev/unsafe)，这里主要谈一下 `unsafe.Pointer` 类型

在使用 cgo 过程中不可避免要传递指针参数，因为 go 的指针类型不能像 c 的指针类型可以隐式转换，go 需要用到类似于 `void*` 的 `unsafe.Pointer` 类型强制转换成 c 可以用的指针，而且可以转换任意类型的指针，而且可以绕过类型检测读写内存，所以，使用时需要格外小心。 `Pointer` 有如下四种特殊操作：

```markdown
- 任何类型的指针值都可以转换为 Pointer。 
- Pointer 可以转换为任何类型的指针值。 
- uintptr 可以转换为指针。 
- 指针可以转换为 uintptr。 
```

### Pointer转换实例

> 下面内容翻译自 go [unsafe](https://pkg.go.dev/unsafe)

以下涉及 `Pointer` 的模式是有效的。不使用这些模式的代码今天可能无效，或者将来可能无效。甚至下面的有效模式也带有重要的警告。

运行 `go vet` 可以帮助查找不符合这些模式的 `Pointer` 用法，但是运行 `go vet` 没警告不能保证这些代码是有效。

#### 1. 将 `*T1` 转换为指向 `*T2` 的指针

假设T2不大于T1，并且两个共享相同的内存布局，则此转换允许将一种类型的数据重新解释为另一种类型的数据。一个示例是`math.Float64bits`的实现：

```go
func Float64bits(f float64) uint64 {
  return *(*uint64)(unsafe.Pointer(&f))
}
```

#### 2. 将`Pointer`转换为`uintptr`（但不转换回`Pointer`）

将 `Pointer` 转换为 `uintptr` 会生成所指向的值的内存地址（整数）。这种 `uintptr` 的通常用法是打印它。通常，将 `uintptr` 转换回 `Pointer` s是无效的。  
`uintptr`是整数，而不是引用。 将 `Pointer` 转换为 `uintptr` 会创建一个没有指针语义的整数值。 即使`uintptr`保留了某个对象的地址，垃圾回收器也不会在对象移动时更新该`uintptr`的值，该`uintptr`也不会使该对象被回收。

其余的模式枚举了从`uintptr`到`Pointer`的唯一有效转换。

#### 3. 用算术将`Pointer`转换为`uintptr`并返回

如果p指向已分配的对象，则可以通过转换为 `uintptr`，添加偏移量并将其转换回 `Pointer` 的方式将其推进对象。

```go
p = unsafe.Pointer(uintptr(p) + offset)
```

此模式最常见的用法是访问数组的结构或元素中的字段：

```go
// equivalent to f := unsafe.Pointer(&s.f)
f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))

// equivalent to e := unsafe.Pointer(&x[i])
e := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i*unsafe.Sizeof(x[0]))
```

以这种方式从指针添加和减去偏移量都是有效的。通常使用 `＆^` 舍入指针（通常用于对齐）也是有效的。在所有情况下，结果都必须继续指向原始分配的对象。  
与C语言不同，将指针移到其原始分配的末尾是无效的：

```go
// INVALID: end points outside allocated space.
var s thing
end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))

// INVALID: end points outside allocated space.
b := make([]byte, n)
end = unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(n))
```

**注意**：`uintptr` 和 `Pointer` 两个转换必须出现在相同的表达式中，并且它们之间只有中间的算术运算：

```go
// INVALID: uintptr cannot be stored in variable
// before conversion back to Pointer.
u := uintptr(p)
p = unsafe.Pointer(u + offset)
```

**注意**：待转换的指针必须指向已分配的对象，不能为nil。

```go
// INVALID: conversion of nil pointer
u := unsafe.Pointer(nil)
p := unsafe.Pointer(uintptr(u) + offset)
```

#### 4. 调用 `syscall.Syscall` 时将指针转换为 `uintptr`

`syscall` 包中的 `Syscall` 函数将其 `uintptr` 参数直接传递给操作系统，然后，操作系统可以根据调用的详细信息将其中一些参数重新解释为指针。 也就是说，系统调用实现正在将某些参数从 `uintptr` 隐式转换回指针。  
如果必须将指针参数转换为 `uintptr` 用作参数，则该转换必须出现在调用表达式本身中：

```go
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
```

编译器通过安排保留引用的分配对象（如果有的话），直到调用完成（即使从类型本身而言）也不会移动，来处理在汇编中实现的函数的调用的参数列表中转换为`uintptr`的 `Pointer`。 似乎在调用过程中不再需要该对象。  
为了使编译器能够识别这种模式，转换必须出现在参数列表中：

```go
// INVALID: uintptr cannot be stored in variable
// before implicit conversion back to Pointer during system call.
u := uintptr(unsafe.Pointer(p))
syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))
```

#### 5. 将 `reflect.Value.Pointer` 或 `reflect.Value.UnsafeAddr` 的结果从 `uintptr` 转换为 `Pointer`

包反射的名为 `Pointer` 和 `UnsafeAddr` 的 `Value` 方法返回类型 `uintptr` 而不是 `unsafe.Pointer`，以防止调用者在不首先导入 `"unsafe"` 的情况下将结果更改为任意类型。 但是，这意味着结果很脆弱，必须在调用后立即使用相同的表达式将其转换为Pointer：

```go
p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
```

与上述情况一样，在转换之前存储结果是无效的：

```go
// INVALID: uintptr cannot be stored in variable
// before conversion back to Pointer.
u := reflect.ValueOf(new(int)).Pointer()
p := (*int)(unsafe.Pointer(u))
```

#### 6. 将一个 `reflect.SliceHeader` 或 `reflect.StringHeader` 数据字段与指针进行转换

与前面的情况一样，反射数据结构 `SliceHeader` 和 `StringHeader` 将字段 `Data` 声明为 `uintptr`，以防止调用者在不首先导入 `"unsafe"` 的情况下将结果更改为任意类型。 但是，这意味着 `SliceHeader` 和 `StringHeader` 仅在解释实际切片或字符串值的内容时才有效。

```go
var s string
hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // case 1
hdr.Data = uintptr(unsafe.Pointer(p))              // case 6 (this case)
hdr.Len = n
```

在这种用法中，`hdr.Data` 实际上是在字符串标题中引用基础指针的另一种方法，而不是 `uintptr` 变量本身。  
通常，`reflect.SliceHeader` 和 `reflect.StringHeader` 只能用作指向实际切片或字符串的 `*reflect.SliceHeader` 和 `*reflect.StringHeader`，而不能用作纯结构。 程序不应声明或分配这些结构类型的变量。

```go
// INVALID: a directly-declared header will not hold Data as a reference.
var hdr reflect.StringHeader
hdr.Data = uintptr(unsafe.Pointer(p))
hdr.Len = n
s := *(*string)(unsafe.Pointer(&hdr)) // p possibly already lost
```
