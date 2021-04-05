---
title: 对 golang map 的认识
top: false
cover: false
toc: true
mathjax: true
date: 2021-04-05 10:31:49
password:
summary:
categories: 编程语言
tags:
  - Golang
  - Map
---

谈一谈对 golang map 的认识，map 是属于哈希结构数据类型，用于存储键值对，键值唯一，存储没有先后顺序。和切片类似，都是引用传值。

## 初始化及基本操作

map 一般情况下用 `make` 函数初始化, 也可以使用字面量直接初始化一些初值

```go
m := make(map[string]int)
m["a"] = 2 // 若 'a' 不存在，新建键值对，若存在，则修改
fate := map[string]string{
    "fsn": "fate stay night",
    "fz": "fate zero",
    "fgo": "fate grand order",  // 每行结尾需要逗号，若与花括号写在同一行，则最后一个元素不需要
}
fmt.Println(fate["fsn"]) // 通过 key 访问 value， 元素不存在返回类型的零值
delete(fate, "fgo")      // 使用 delete 函数删除键值对，无论元素是否存在都会执行，且没有返回值
if val, ok := fate["fsn-ubw"]; ok {
    // 可以使用第二返回值 ok 检测键值对是否存在
    // 并且返回的 val 不能进行取地址操作，当 map 扩容后，该键值的地址会发生改变，
    // fmt.Println(&val) // 编译不会通过
}
for k, v := range fate {
    // 使用 range 对键值对进行遍历，每次返回的顺序是随机的，是因为要考虑到 map 扩容带来的内存变动，故意设定每次随机返回
    fmt.Println(k, v)
}
```

## 线程安全

map 在单个 `goroutine` 内是安全的，因为存取都是顺序操作。  
当有多个 `goroutine` 同时对 map 进行读写操作时，会发生 `panic`，因为 map 内部有写状态检测标志 `flag`，当 `flag==1` 时，表示正在写操作，此时再有 `goroutine` 的写操作进来，就会直接抛出异常。  

解决方法也非常简单，一种方法是将 map 放在结构体中，并添加锁 `sync.RWMutex`，当然这会减缓服务器的速度，所以为了效率，尽量确保多个 `goroutine` 对 map 都是读操作。

```go
var counter = struct{
    sync.RWMutex
    m map[string]int
}{m: make(map[string]int)}
// 读取操作
counter.RLock()
n := counter.m["some_key"]
counter.RUnlock()
fmt.Println("some_key:", n)
// 写操作
counter.Lock()
counter.m["some_key"]++
counter.Unlock()
```

另一种方法是使用 golang 官方的 `sync` 包中的 [Map](https://golang.org/pkg/sync/#Map)，可以实现安全读写操作。

## 底层结构

关于 map 的底层结构，我没有深入研究，有兴趣可以研读下面的文章：

- [Go 语言问题集(Go Questions)](https://www.bookstack.cn/read/qcrao-Go-Questions/map-map%20%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E6%98%AF%E4%BB%80%E4%B9%88.md)
- [Go 语言设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#332-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)

这里简单讲一下扩容的过程

### 扩容

map 为了提高哈希查找速率，将键值对数据存储在多个容量只有 8 的桶内，也就是一桶内最多存储8对数据，桶之间通过链表连接。当 map 到达扩容条件时，创建新桶并进行数据迁移。

了解扩容前，需要知道一个概念，go 定义了一个变量 `装载因子`，如下所示

```go
// 装载因子  =  元素个数 / 桶的数量
loadFactor := count / (2^B)
```

扩容有两个条件可以触发：

1. 装载因子超过 6.5 时
2. 溢出桶(overflow buckets)过多

溢出桶简单理解就是已经存满了的桶额外添加的桶，

针对不同的条件，有不同的扩容策略

- 对于条件1，go 会添加新桶，**容量翻倍**，并将旧桶中的数据迁移到新桶中
- 对于条件2，go 开辟新的内存区，**容量和之前相等**，并将之前旧桶中的数据全部迁移过去，通过 GC 机制释放之前的内存区，达到整理内存，提高访问效率的目的

扩容完成后就需要迁移数据，但是迁移并不是一次性完成的，而是在每次进行写操作之后会迁移部分数据，直到旧桶中的数据迁移完成
