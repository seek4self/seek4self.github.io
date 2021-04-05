---
title: Golang 数组和切片的区别
date: 2021-04-03 20:36:36
top: false
cover: false
password:
toc: true
mathjax: true
summary:
categories: 编程语言
tags: 
  - Golang
  - Slice
---
golang 集合类型包含数组和切片两种，存储相同类型的序列数据

## 初始化

- 数组：初始化后长度固定不变，不可赋值为其他长度的类型，可以通过索引修改内部元素

```go
// 数组 type: [len]Type
arr := [1]int{}
fmt.Println(arr, len(arr), cap(arr)) // [0] 1 1
arr[0] = 1
arr := [...]int{1, 2, 3} // 编译器根据内容推导类型： [3]int
```

- 切片：初始化后长度动态可变，可增长可收缩，底层数据是基于数组指针的结构体，

```go
// 切片 type: []Type
sli := make([]int, 2) // 切片初始化建议使用 make 指定长度，
fmt.Println(sli, len(sli), cap(sli)) // [0 0] 2 2
sli[0] = 1
sli = append(sli, 2) // 切片使用 append 进行追加元素并在容量不足时进行扩容
fmt.Println(sli, len(sli), cap(sli)) // [1 0 2] 3 4
```

## 访问和赋值

- 数组和切片：均通过 `[n]` 获取或修改对应位置元素

```go
arr := [...]int{1, 2, 3, 4, 5}
arr[1] = -1
arr[2] += arr[4] // 8
sli := arr[1:4]  // 通过 : 符号初始化切片
fmt.Printf("%T: %[1]v len: %d cap: %d\n", sli, len(sli), cap(sli)) // []int: [-1 8 4] len: 3 cap: 4
sli = append(sli, sli...) // 复制并合并切片
sli1 := sli[2:] // 截取部分切片
sli1[2] = 0     // 切片修改 sli 底层数据
fmt.Println(sli, len(sli), cap(sli))    // [-1 8 4 -1 0 4] 6 8
fmt.Println(sli1, len(sli1), cap(sli1)) //      [4 -1 0 4] 4 6
sli1 = append(sli1, 5, 6)
fmt.Printf("ptr: %p %[1]v %d %d\n", sli1, len(sli1), cap(sli1)) // ptr: 0xc000020090 [4 -1 0 4] 4 6
sli1 = append(sli1, []int{5, 6, 7}...)  // 容量不够，进行扩容，内存地址发生变动
sli1[1] = 2 // 此时修改不会影响 sli
fmt.Printf("ptr: %p %[1]v %d %d\n", sli1, len(sli1), cap(sli1)) // ptr: 0xc00007c060 [4 2 0 4 5 6 7] 7 12
```

> `fmt` 占位符小知识  
> `%T`: 打印变量类型  
> `%[n]v`: 使用 `[n]` 访问第n个参数，不用重复传入参数
> `%p`: 打印变量指针

### 切片结构

```go
type SliceHeader struct {
    Data uintptr // 底层数组的指针
    Len  int     // 切片的长度
    Cap  int     // 截取底层数据的容量  >= Len
}
```

由于结构体中引用了底层数组的指针，所以slice传值是引用传递，修改形参内部元素会影响实参的数据

```go
slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
s1 := slice[2:5]
s2 := s1[2:6:7]  // 截取 s1 从索引 [2,6) 且容量为 7以前的部分
fmt.Println(slice, len(slice), cap(slice))  // [0 1 2 3 4 5 6 7 8 9] 10 10
fmt.Println(s1, len(s1), cap(s1))           //     [2 3 4]           3 8
fmt.Println(s2, len(s2), cap(s2))           //         [4 5 6 7]     4 5
s2 = append(s2, 100) // 容量(5)等于长度(5)，修改原底层数组
s2 = append(s2, 200) // 长度(6)超出容量(5)，重新分配内存
s1[2] = 20           // 修改原底层数组，不影响 s2
fmt.Println(slice)   // [0 1 2 3 20 5 6 7 100 9]
fmt.Println(s1)      //     [2 3 20]
fmt.Println(s2)      //          [4 5 6 7 100 200]
```

因此多个切片可以共用同一个底层数组，实际内容却不相同，想了解更多深层内容，请移步[Go语言设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/#321-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)

### 切片扩容

- 一般情况下：
  - 若切片当前长度小于1024时，容量会翻倍增长
  - 若切片当前长度大于1024时，容量会每次增长1/4
- **特殊情况**：当数组中元素所占的字节大小为 1、8 或者 2 的倍数时，会发生内存对齐

具体源码解析可参见下面两文：

- [Go语言问题集](https://www.bookstack.cn/read/qcrao-Go-Questions/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87-%E5%88%87%E7%89%87%E7%9A%84%E5%AE%B9%E9%87%8F%E6%98%AF%E6%80%8E%E6%A0%B7%E5%A2%9E%E9%95%BF%E7%9A%84.md)
- [Go语言设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/#324-%E8%BF%BD%E5%8A%A0%E5%92%8C%E6%89%A9%E5%AE%B9)

## 可比较性

- 数组：当长度和内容相同时可以视为相等，可以用 `==` 比较
- 切片：因为底层数组可能变化，不可以直接用 `==` 比较，**特别的**：空切片 `[]T(nil)` 与 `nil` 是相等的

```go
arr1 := [2]int{1, 2}
arr2 := [2]int{1, 2}
arr3 := [2]int{2, 3}
fmt.Println(arr1==arr2, arr1==arr3) // true false

var slice []int
fmt.Println(slice == nil, []int{} == nil, []int(nil) == nil) // true false true
// 一般切片不直接和 nil 比较，检测长度是否为零进行判断
fmt.Println(len(slice)==0) // true
```

## 小结

- 相同点：
  - 都用于存储单一类型序列元素
  - 底层均基于数组结构
  - 均通过索引下标访问元素
- 区别：

|   -    |                          数组                          |                              切片                               |
| :----: | :----------------------------------------------------: | :-------------------------------------------------------------: |
| 初始化 |                   长度固定，不可修改                   | 长度、容量动态增长<br> 追加元素超出容量，进行扩容，内存重新分配 |
|  赋值  | 新数组与原数组是两个变量，修改新数组对原数组不产生影响 |     新切片与原切片指向同一个底层数组，修改内容会影响原切片      |
| 可比较 |                       可相互比较                       |                        仅可与 `nil` 比较                        |
