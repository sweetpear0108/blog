---
title: 'Go 垃圾收集器'
date: 2023-12-22T16:48:00+08:00
# weight: 1
# aliases: ["/alias"]
tags: ["golang","垃圾收集器"]
author: "sweetpear"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
summary: "该文章介绍了Go语言中的垃圾回收机制，包括算法、策略和优化技术。"
# canonicalURL: "https://canonical.url/to/page"
---
# Collector
## 参考
[draveness垃圾收集器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#72-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8)

[go语言原本-垃圾回收](https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/)

[go程序员面试笔试宝典-垃圾回收器20问](https://golang.design/go-questions/memgc/principal/)

[渐进式垃圾回收-CSDN](https://blog.csdn.net/puss0/article/details/126357922)

[奇伢云存储-golang混合写屏障](https://liqingqiya.github.io/golang/gc/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/%E5%86%99%E5%B1%8F%E9%9A%9C/2020/07/24/gc5.html)

## 设计原则

### 垃圾收集算法
* 追踪式：从根对象出发，一步步扫描回收对象
* 引用计数式：对象自身包含一个被引用的计数器

    * 根对象是什么：全局变量、执行栈、寄存器

#### 分代式
根据对象的存活时间进行分类。（JAVA：年轻代、老年代、永久代）
* 优点：相比传统标记清除 STW 更小，分代选取不同垃圾收集策略，效率高。
* 缺点：额外开销，内存占用，且golang中编译器会通过逃逸分析将大部分新生对象存储在栈上（栈直接被回收），因此没必要。

#### 标记清除
从根对象出发，将确定存活的对象进行标记，并清扫可以回收的对象。
* 优点：简单
* 缺点：需要长时间 STW

#### 三色标记清除
优化的标记清除算法，将对象分为**三类**：黑、白、灰，保证与用户程序**并发执行**时程序的正确性（对象不会被错误回收）
* 优点：利用并发的优势，大幅减小 STW 时间
* 缺点：复杂，需要配合**屏障技术**，否则会出现悬挂指针问题（一个还在生命周期的对象被错误回收了）

### 收集策略
并发和增量策略都需要配合[屏障技术](###屏障技术)保证回收的正确性

#### 增量
增量式收集是将一个完整的 GC 过程切成多个更小的 GC 时间片，穿插在用户程序之间执行，在执行时仍需要 STW。
目的：减少了每次 STW 用户程序需要等待的时间，将总等待时间分散到多个时间片中。
![increment](https://img.draveness.me/2020-03-16-15843705141864-incremental-collector.png)

#### 并发
利用多核优势在部分阶段与用户程序并行执行
目的：减少了总的 STW 时间
![concurrent](https://img.draveness.me/2020-03-16-15843705141871-concurrent-collector.png)

### 三色抽象
#### 概念
三类对象：
* 白色对象：未被回收器扫描到的对象。回收开始阶段全部对象都为白色，扫描结束后存在的白色对象为不可达对象。
* 灰色对象：已经被回收器扫描但存在指针指向一些还没被扫描白色对象。（**波面**）
* 黑色对象：对象本身以及其所有指针指向的对象都被扫描过，确定存活的对象。

#### 三色不变性
想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性（Tri-color invariant）中的一种：
* 强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；（黑色不能指向白色）
* 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径；（黑色可以指向白色，前提是必须要有另外的灰色对象直接或间接指向这个白色）

#### 为什么要三个色？
为了配合屏障技术，保证在[并发或者增量](#收集策略)执行时不会出现**悬挂指针**（本来不应该被回收的对象却被回收了）。

具体来说，就是要保证达成至少一种三色不变性。

### 屏障技术
是一种同步机制，使用户程序在进行指针写操作时，能够“通知”回收器，进而不会破坏三色不变性，保证在增量或并发回收时程序的正确性

#### Dijkstra 插入写屏障
在修改引用时，将**新**指向的对象置灰。换句话说，目的是将有存活可能的对象都标记成灰色
```go
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```
slot是代码中的接收位置（destination），ptr是被指向的对象，ptr需要进入slot中（goes into the slot）

![dijkstra](https://img.draveness.me/2020-03-16-15843705141840-dijkstra-insert-write-barrier.png)

如图所示，用户程序修改A的指针，将其指向C，这时写屏障触发将C标记为灰色，在收集器启动时，将从B C 两个对象出发进行扫描。

* 保证了**强**三色不变性。
* 是一种**保守**的屏障技术。
* 可能会遗留不再存活的对象，如图中的B，或者当图中第二三步间重新改变A指向B后的C。
* 栈上对象也需要保证回收正确性，因此需要为栈对象添加写屏障或STW并重新扫描栈上对象，GO1.7版本以前采用了后者。

#### Yuasa 删除写屏障
在删除引用时，如果**原来**指向的对象为白色，则将其置灰。目的是间接保留原有的引用关系使得扫描可以进行下去。

```
writePointer(slot, ptr)
    shade(*slot)
    *slot = ptr
```

![yuasa](https://img.draveness.me/2021-01-02-16095599123266-yuasa-delete-write-barrier.png)

第二步由于B本来就是灰色的，所以不用改；第三步将B指向C的引用删除，这时如果不将C染成灰色，后续收集器就不会扫描白色的C和D，于是CD被错误的回收了。

```
如果第二步 A 指向了另外一个独立的白色对象 E 岂不是出错了？
这种情况不可能出现，原因见下边第三点。
```

* 保证了**弱**三色不变性。
* 同样会遗留不再存活的对象。
* 【very important！】**指针更新一定是从一个活动对象指向另一个活动对象，因为非活动对象是没有任何指针引用的，用户程序不可能再引用到它。换句话说，不可能新增一个指向孤立的白色对象的引用**

#### 混合写屏障
插入和删除写屏障都存在的问题：对栈对象加屏障严重拖累性能，尤其对于go来说（goroutine可以很多）

如果不对栈对象加屏障，那么会导致的问题：
* 只用插入屏障：一个黑色栈对象指向一个白色堆对象，并且原来灰色对象指向该白对象的指针被删除了，的会导致悬挂指针问题，需要标记后 STW 重新扫描所有栈对象。
* 只用删除屏障：标记结束后不需要重扫栈，但是图1->图2过程会导致悬挂指针。

![1](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-07-25-gc5/39D8A3CD-C763-4893-996E-27DE39AE3CFB.png)
![2](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-07-25-gc5/A1873F49-31DE-47AD-835C-36007D34FBFB.png)

因此，选择混合写屏障（插入+删除）可以避免指针悬挂问题，且不用标记结束后重新扫栈，提高了效率。
```
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

此外，要将创建的所有**新**对象都标记成黑色，防止新分配的对象被错误地回收。

## 垃圾回收的触发
### 触发阈值
通过环境变量 `GOGC` 设置，默认为100，即增长 100% 的堆内存才会触发 GC。

不同于进入 STW 以后进行垃圾收集的方法，并发收集器并不能等到堆内存达到触发阈值时才开始运行，因为在 GC 期间还会有用户程序在分配新的堆内存。

所以使用 Pacing 算法计算触发 GC 的最佳时间，使得收集（标记扫描）结束时堆内存近似达到阈值

### 触发时机
可以分为三种情况：
1. 后台定时检查，当一定时间内没有触发（默认2min），就会触发新的 GC 循环。
2. 用户程序手动触发，如果当前没有开启 GC ，则触发新的循环。
3. 申请内存时根据堆大小触发，当堆内存达到阈值时，则触发新的循环。

#### 后台触发
运行时在应用程序启动时创建的一个专门用于触发垃圾收集的goroutine，大多数时间是在休眠的，由 sysmon 在满足触发条件的时候唤醒。

#### 手动触发
用户程序通过 `runtime.GC` 函数主动通知运行时执行

#### 分配内存
在分配微对象和小对象的内存时，如果mcache中找不到空闲内存单元，则会向下层申请mspan并触发 GC。

在分配大对象之前，一定会先触发 GC 回收内存。

## GC 实现
核心函数 `runtime.gcStart`，开启一个 GC 循环。

### 垃圾回收三状态
垃圾回收器通过 _GCoff、_GCMark 和 _GCMarktermination 三个标记来确定写屏障状态
```
_GCoff  ---->  _GCmark  ---->  _GCmarktermination
   /\                                  |
   |___________________________________|

```
* _GCoff：清理状态; 写屏障**关闭**

* _GCmark：标记状态；写屏障**开启**

* _GCmarktermination：标记终止状态；写屏障**开启**

### 垃圾回收四阶段
清理终止阶段（STW）--> 标记阶段 --> 标记终止阶段（STW）--> 清理阶段

1. 清理终止阶段

状态 `_GCoff`；所有处理器在这时进入安全点；对上个垃圾回收阶段进行一些收尾工作（例如清理缓存池、清理已经被标记的内存单元、清理标记等等），
以及完成进入标记阶段前的一些准备工作（[启动后台标记任务](#后台标记模式)、更新 GC 组件中相关变量），**进入 STW 状态**。

2. 标记阶段

状态切换至 `_GCmark`；扫描栈上、全局变量等根对象并将它们加入队列，将所有P的微分配器中的对象置灰，以及**开启写屏障**，**退出 STW 状态**，程序恢复执行，后台标记任务开始恢复进行。

3. 标记终止阶段

当所有的标记任务都完成后，**进入 STW 状态**，状态切换至 `_GCmarktermination`；以及完成标记的收尾工作（统计数据、重制状态等），**关闭写屏障**。

4. 清理阶段

状态切换至 `_GCoff`；**退出 STW 状态**，后台并发清理所有的内存管理单元

* 总之，在访问公共资源时需要进入 STW 状态，进行扫描标记时要开启写屏障

#### 如何暂停和恢复程序
暂停：抢占所有的处理器P，将他们更新至 `_Pgcstop` 状态

恢复：依次唤醒所有的处理器

#### 后台标记模式
在步骤1（标记阶段开始前），运行时会为全局**每个处理器**创建一个用于后台标记任务的 Goroutine（遍历 `runtime.allp`），这些 Goroutine 都会与P绑定成为 node结构体 被加入 `runtime.gcBgMarkWorkerPool` **线程池**并**陷入休眠**等待调度器的唤醒；程序恢复后在调度循环中寻找可运行的G `runtime.findrunnable` 的时候被从池中取出调度到P上运行。

这些后台标记任务被称为 `gcMarkWorker`

worker有三种模式：
1. gcMarkWorkerDedicatedMode：处理器专门负责标记对象，**不会被调度器抢占**
2. gcMarkWorkerFractionalMode：当垃圾收集的后台**CPU 使用率**达不到预期时（默认为 25%），启动该类型的工作协程帮助垃圾收集达到利用率的目标
3. gcMarkWorkerIdleMode：当处理器**没有**可以执行的 Goroutine 时，它会运行垃圾收集的标记任务**直到被抢占**

在调度循环中会根据全局处理器的个数以及垃圾收集的 CPU 利用率确定worker的工作模式

三种模式相互协同保证了标记的速率，在到达[堆内存触发阈值](#触发阈值)前完成标记任务

#### 辅助标记
在并发标记阶段期间，当 Goroutine 调用 `runtime.mallocgc` 分配新对象时，该函数会检查申请内存的 Goroutine 是否处于入不敷出的状态：它遵循一条非常简单并且朴实的原则，分配多少内存就需要完成多少标记任务。

* 假设如果有个Goroutine一直大量的分配内存，这样有可能永远都扫描不完所有对象，或者当扫描结束时堆内存已经到了一个很夸张的大小。所以辅助标记就是为了避免这种情况。

* 后台标记模式和辅助标记区别：

后台标记是通过**并发**加快标记的速度，以保证到达触发阈值时完成标记任务；

辅助标记是在后台并发标记**期间**，**当前** goroutine **分配内存**时，通过控制分配与标记的“动态平衡”，保证到达触发阈值时完成标记任务；


## 观察程序GC的四种方法
### GODEBUG
```shell
GODEBUG=gctrace=1 ./main
```
输出
```shell
gc 1 @0.000s 2%: 0.009+0.23+0.004 ms clock, 0.11+0.083/0.019/0.14+0.049 ms cpu, 4->6->2 MB, 5 MB goal, 12 P
scvg: 8 KB released
scvg: inuse: 3, idle: 60, sys: 63, released: 57, consumed: 6 (MB)
```
其中，gc开头的是用户代码向运行时申请内存产生的垃圾回收，scvg开头的是运行时向操作系统申请内存（归还内存）产生的垃圾回收

### trace
```go
func main() {
	f, _ := os.Create("trace.out")
	defer f.Close()
	trace.Start(f)
	defer trace.Stop()
	(...)
}
```
通过在代码内调用trace API 来在网页内进行可视化观察

### debug.ReadGCStats
```go
func printGCStats() {
	s := debug.GCStats{}
	debug.ReadGCStats(&s)
	fmt.Printf("gc %d last@%v, PauseTotal %v\n", s.NumGC, s.LastGC, s.PauseTotal)
}
```
可以搭配计时器实现对指定指标的定时监控

### runtime.ReadMemStats
```go
func printMemStats() {
	s := runtime.MemStats{}
    runtime.ReadMemStats(&s)
	fmt.Printf("gc %d last@%v, next_heap_size@%vMB\n", s.NumGC, time.Unix(int64(time.Duration(s.LastGC).Seconds()), 0), s.NextGC/(1<<20))
}
```
和上边方法一样，只不过这里直接调用了运行时的结构体

## GC调优
调优思想：优化内存的申请速度（控制），尽可能的少申请内存（减少），复用已申请的内存（复用），减少GC触发频率（频率）

控制：提高赋值器CPU使用率，减少花费在调度器的等待上的时间（使用 `sync.WaitGroup` 等并发策略避免系统中存在过多的goroutine，因为每个goroutine都会占用内存）。

减少：合理使用数据结构

复用：sync.Pool 等池化技术

频率：调整GOGC上限（```GOGC=1000 ./main```）