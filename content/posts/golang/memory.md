---
title: 'Go 内存分配器'
date: 2023-12-02T22:39:00+08:00
# weight: 1
# aliases: ["/alias"]
tags: ["golang","内存分配器"]
author: "sweetpear"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
summary: "介绍了go内存分配的三级模型，还讨论了分配器的种类和设计原则，以及内存布局的组件和对象分配的流程。整体而言，Go的内存分配器通过多级缓存和分级分配的思想，提高了内存管理的效率和空间利用率。"
# canonicalURL: "https://canonical.url/to/page"
---
# 内存分配
## 参考
[draveness 内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)

[Go语言原本 内存分配](https://golang.design/under-the-hood/zh-cn/part2runtime/ch07alloc/)

## 例子
golang的内存分配模型可以与现实中的 零售店-经销商-工厂 模式进行类比，目的都在于可以合理的分配资源以及保证高效率

具体而言，零售店（线程缓存mcache）拥有很多种类的商品（可以看做是空闲内存空间或者内存页），每种商品由一个售货员（mspan）进行管理（记录售出状态）和负责销售，零售店的优势就在于

离顾客（goroutine）家很近，买东西比较便捷。当顾客需要买数量较少的商品时（微对象小对象）直接去零售店就好，如果要大批量订货（大对象）则需要直接去工厂，零售店的库存肯定不够。

当零售店库存不足时，会向经销商（mcentral）进货。经销商拥有所有种类的商品。

当经销商也没有足够的库存时，就要向工厂（mheap）进货。工厂仓库（arena）中存着所有已经生产完的商品，当仓库中所有的货都被预定了没有可售商品时，就会找原料供应商（操作系统）进货。

获得原料（page）后，就会进行加工并且规划一些新仓库存放（堆扩容）。

供货链采用三个层级的策略，减少了顾客需要亲自去工厂交易的繁琐，也保证了工厂不会将大量时间花在处理小订单上边，提高了商品流通的效率。

## 设计原则
### 分配器
#### 线性分配器
依靠指针的增量来确定下一个可用的内存位置，需要搭配具有拷贝特性的垃圾回收算法（标记压缩、复制回收、分代回收等）整理内存碎片，JAVA采用的分配方法，对于C/C++等直接对外暴露指针的语言不适用。

优点：复杂度较低

缺点：无法重用回收的内存（整理碎片之前）

#### 空闲链表分配器
将内存块通过指针连成链表，空闲链表分配器可以选择不同的策略在链表中的内存块中进行选择。

优点：可重用回收的内存

缺点：分配时需要遍历链表

* 四种内存分配策略
    * 首次适应（First-Fit）— 从链表头开始遍历，选择第一个大小大于申请内存的内存块；
    * 循环首次适应（Next-Fit）— 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块；
    * 最优适应（Best-Fit）— 从链表头遍历整个链表，选择最合适的内存块；
    * **隔离适应（Segregated-Fit）**— 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；

其中，**隔离适应**与GO使用的策略相似：
![隔离适应](https://img.draveness.me/2020-02-29-15829868066452-segregated-list.png)

### 分配思想
#### 分级分配
它将对象划分为多个不同的大小级别（go中为微对象、小对象、大对象），并使用不同的分配器来管理每个级别的内存分配。通过分级分配，可以更好地适应不同大小对象的内存需求，提高整体的内存管理效率和空间利用率。

* 注：分配器和分配思想是两个不同层面的概念，分级分配是在宏观层面为新对象选择合适的缓存存储级别，而分配器则面向底层内存分配的具体实现。

### 内存布局
![布局](https://img.draveness.me/2020-02-29-15829868066479-go-memory-layout.png)

mspan是内存管理单元，负责管理内存页和分配的对象

mcache作用相当于缓存，它缓存了一些可以直接拿来用的内存管理单元，直接满足当前线程的内存需求，不用加锁步骤简单速度很快。

mcentral像是一个二级缓存，按种类不同每类保存一些可以拿来用的内存管理单元，方便mcache层快速获取想要的种类。

mheap由一个个的arena构成，arena作为虚拟内存的**索引**可以快速定位内存中的几页。

* 注：不是一个mcache对应一个mcentral哦

go1.11以前堆内存空间是连续的，1.11中改为使用稀疏内存，`runtime.heapArena` 管理 `pagesPerArena` 个页（不同平台可能不同，但*页*都是8KB）

* 注：go语言的页（8KB）由运行时自己定义的，与操作系统的页（大部分4KB）不同。更大的页可以减少内存管理开销（小对象可以挤一挤）、提高缓存命中率（大对象不用分太碎）以及满足对齐需求。

## 核心组件
### mspan
`runtime.mspan` 是GO语言内存管理的**基本单元**，每个mspan管理npages个GO页并持有管理的页的占用情况。每个mspan作为**相同spanClass大小等级**的双向链表中的一个节点相互引用。

* 最小分配单位是对象（object）而不是页，object的大小是由mspan的跨度类（spanClass）决定的

对象间通过 `FreeList` 链表形式连接。

* Freelist是一种特殊的链表形式，通过节点值的高8字节确定下个节点的内存地址，而不是传统的保存一个Next指针字段。可以节省内存。

#### spanClass
`runtime.spanClass` 是mspan的跨度类，它决定了内存管理单元中存储的对象大小和个数。

源码中预存储了*67*个跨度类尺寸 (size class)，运行时中还包含*1*个id为0的特殊跨度类，它能够管理大于 32KB 的特殊对象。

跨度类一共有 68 * 2 = 136 个：68个尺寸（size class），每种尺寸有两个类型（ scan 和 noscan ），noscan表示对象不包含指针所以不会被GC扫描，可以提高GC效率。

一个spanClass由一个 `uint8` 表示，其中高7位是id，最后一位表示 scan 或 noscan。

在运行时分配对象时会根据对象大小选择适合的跨度类。也正是与上述[隔离适应](####空闲链表分配器)策略的相似之处。

### mcache
线程缓存，会与处理器 P 一一绑定，主要用来缓存微小对象。每一个线程缓存都在 `mcache.alloc` 数组中持有 68 * 2 个 `runtime.mspan`，每一个数组元素都是某一个特定大小的 mspan 的链表头指针。

拥有微对象分配器，专门管理16B以下的**非指针类型**的对象，

优势：由于与P绑定，所以不会有并发问题，访问时不用加锁，效率更高。（为什么不与M绑定：M可能陷入休眠状态，期间无法复用缓存）

#### 初始化
在初始化处理器P时会调用 `runtime.allomcache` 在系统栈中使用 mheap 中的线程缓存分配器进行初始化。

### mcentral
中心缓存，每个中心缓存管理**一种跨度类**对应的内存管理单元（mspan），总共136个由mheap保存。这些mspan由四个 `spanSet` 存储，分别是：

* 包含空闲对象的 partial set:   [swept,unswept]

* 不包含空闲对象 full set:      [swept,unswept]

    * 垃圾回收器完成对某个内存跨度的扫描和标记操作后，该内存跨度就被认为是 "swept"

访问中心缓存中的内存管理单元**需要使用互斥锁**：

### mheap
页堆，以全局变量形式 `runtime.mheap_` 存储，包含了长度为136（scan & noscan）的mcentral数组，持有的mspan列表以及所有的 `heapArena`（由二维数组存储）。

#### heapArena
GO堆在逻辑上划分为多个 `heapArena` ，可以把 heapArena 看作“堆块”， 每“块”里有 `heapArenaBytes / pageSize` 个页， 分别由 mspan 进行管理。

## 对象分配
入口: `runtime.newobject`，`new`关键字的实现

在分配时，会占有m线程，并根据对象大小和是否为指针类型（scan）确定分配逻辑：
```
if size <= 32KB {
    if noscan && size < 16B {
        分配微对象
    } else {
        分配小对象
    }
} else {
    分配大对象
}
```

### 大对象
对于size>32KB的对象，直接在堆上进行分配。

#### 流程
1. 计算需要的pages，然后进入系统栈尝试获取新的mspan

2. 在分配之前先回收一部分内存以避免内存大量占用

3. 【mheap加锁】内存分配、mspan初始化 【mheap解锁】

    1. 从堆上分配新的内存页和内存管理单元mspan（id为0的特殊mspan），如果堆空间不足还要考虑扩容（向操作系统申请空间），如果p0上的mspan缓存 `runtime.p.mspancache` 不足还要用 `runtime.fixalloc` 分配器分配

    * 页内存分配使用的是全局的页分配器 `runtime.pageAlloc`（Radix tree）

    2. 初始化刚刚获得的mspan，并且建立mheap和mspan的关系（加入heapArena.spans）

    3. 清理新分配的内存，进行归零操作

4. 返回mspan中的base地址 `mspan.startAddr`

### 小对象
对于16B<=size<=32KB的对象和size<16但包含指针B的对象，，先后尝试从线程缓存、中心缓存、堆中获取mspan，如果没找到可用的则从操作系统分配新页

#### 流程
1. 根据对象大小找到对应的跨度类，尝试从mcache的mspan数组缓存（`mcache.alloc`）中的获取对应的mspan，通过 `mspan.allocCache`（保存对象空闲状态的位图，原理：[德布鲁因序列](https://halfrost.com/go_s2_de_bruijn/)）寻找能够容纳当前对象的空闲对象内存空间，如果有则完成分配

2. 如果上一步没有找到分配空间，则需要调用 `runtime.refill` 向上级组件获取mspan替换已经不存在可用对象的当前mspan
    1. 将当前的mspan（已满）放入 mcentral 的 full set

    2. 尝试从 mcentral的 partial swept set获取一个mspan
    
    3. 尝试从 mcentral的 partial unswept set获取一个mspan，并且sweep它

    4. 尝试从 mcentral的 full unswept set获取一个mspan，并且sweep它， 如果sweep后发现没有空闲空间，则放到 full swept set中

    5. 如果尝试100次获取后（go1.21）依旧没有找到可用的mspan， 则继续向堆申请
        1. 向堆申请流程与[大对象分配流程](####流程)一致（只有span size不同）
        2. 初始化mspan字段以及初始化heapArena中该mspan的位图信息等字段
    
    6. 至此，寻找能用的mspan的流程终于结束了

3. 将新获取到的mspan保存在（`mcache.alloc`）中，并且返回新mspan中的空闲内存地址（freeIndex）

### 微对象
对于size<16且不包含指针（noscan）B的对象，使用mcache中的微分配器（mcache.tiny等字段）分配。**主要目标**是短字符串和**逃逸的临时变量**

一个tiny内存块（由maxTinySize决定，16B）中可以存多个微对象，大家挤一挤从而节省了空间。而释放内存时，要等块中**所有的对象**都被标记为垃圾时才会进行。

* 逃逸变量：函数内部定义的局部变量，但在函数结束后仍然被其他对象或函数引用的变量（指针作为返回值等）

* 为什么必须要是noscan：为了保证潜在浪费的内存量是有限的（`the amount of potentially wasted memory is bounded.`）

#### 流程
1. 先根据对象的size进行一些内存对齐的操作，调整 `mcache.tinyoffset` 的内存地址

* 为什么要内存对齐：减少 CPU 访问内存的次数。因为 CPU 是以**字长**为单位访问内存的而不是字节。

2. 看看当前tiny内存块中是否装得下（`tinyoffset+size <= maxTinySize`），装得下则直接分配，返回对象内存地址

3. 否则，同[小对象分配](#e6b581e7a88b-1)流程一样获取一个mspan替换旧的tiny块，唯二不同是span size和不用对新分配的内存进行清零（why？）

4. 返回分配到的内存地址

## 总结
GO中内存布局利用了多级缓存以及隔离适应策略（划分多个size class），优化了内存分配的效率，提高了空间利用率，并且由运行时全权管理堆区内存，方便了开发者。这也正是golang内存占用小、运行速度快、使用简便的原因之一。
